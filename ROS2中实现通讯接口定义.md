没问题！为了让你能够最顺畅地完成开发，我把所有内容重新梳理了一遍。

这是一个**从零开始的完整纯净版流程**。我们使用 `ament_cmake` 作为构建系统，将自定义消息和 Python 业务节点分装在两个包中，并将 Python 脚本的注册统一放在 `CMakeLists.txt` 里。

---

### 准备工作：安装依赖
确保你的系统中安装了获取系统信息所需的 Python 库：
```bash
pip3 install psutil
```

---

### 第一阶段：创建自定义消息包 (`sys_info_interfaces`)

**1. 创建包目录**
打开终端，运行：
```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake sys_info_interfaces
mkdir -p sys_info_interfaces/msg
```

**2. 编写消息文件 `SystemInfo.msg`**
创建文件 `~/ros2_ws/src/sys_info_interfaces/msg/SystemInfo.msg`，填入：
```text
string timestamp
string hostname
float32 cpu_usage
float32 mem_usage
float32 mem_total
float32 mem_available
float32 net_recv
float32 net_sent
```

**3. 修改 `CMakeLists.txt`**
打开 `~/ros2_ws/src/sys_info_interfaces/CMakeLists.txt`，替换为以下完整内容：
```cmake
cmake_minimum_required(VERSION 3.8)
project(sys_info_interfaces)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SystemInfo.msg"
)

ament_package()
```

**4. 修改 `package.xml`**
打开 `~/ros2_ws/src/sys_info_interfaces/package.xml`，确保包含以下依赖声明：
```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>sys_info_interfaces</name>
  <version>0.0.0</version>
  <description>System info custom messages</description>
  <maintainer email="user@todo.todo">user</maintainer>
  <license>TODO: License declaration</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  
  <build_depend>rosidl_default_generators</build_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

---

### 第二阶段：创建核心功能包 (`sys_monitor_app`)

为了代码结构清晰，我们在该包下创建一个 `scripts` 文件夹存放 Python 代码。

**1. 创建包目录**
```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake sys_monitor_app
mkdir -p sys_monitor_app/scripts
```

**2. 编写信息发布节点 `sys_pub.py`**
创建文件 `~/ros2_ws/src/sys_monitor_app/scripts/sys_pub.py`，填入：
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import rclpy
from rclpy.node import Node
from sys_info_interfaces.msg import SystemInfo
import psutil
import socket
from datetime import datetime

class SysInfoPublisher(Node):
    def __init__(self):
        super().__init__('sys_info_publisher')
        self.publisher_ = self.create_publisher(SystemInfo, 'system_info_topic', 10)
        self.timer = self.create_timer(1.0, self.timer_callback)
        self.hostname = socket.gethostname()

    def timer_callback(self):
        msg = SystemInfo()
        
        msg.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        msg.hostname = self.hostname
        
        msg.cpu_usage = float(psutil.cpu_percent(interval=None))
        mem = psutil.virtual_memory()
        msg.mem_usage = float(mem.percent)
        msg.mem_total = float(mem.total / (1024 * 1024))
        msg.mem_available = float(mem.available / (1024 * 1024))
        
        net = psutil.net_io_counters()
        msg.net_recv = float(net.bytes_recv / (1024 * 1024))
        msg.net_sent = float(net.bytes_sent / (1024 * 1024))

        self.publisher_.publish(msg)
        self.get_logger().info(f'发布状态: CPU {msg.cpu_usage}%, 内存 {msg.mem_usage}%')

def main(args=None):
    rclpy.init(args=args)
    node = SysInfoPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

**3. 编写界面订阅节点 `gui_sub.py`**
创建文件 `~/ros2_ws/src/sys_monitor_app/scripts/gui_sub.py`，填入：
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import rclpy
from rclpy.node import Node
from sys_info_interfaces.msg import SystemInfo
import tkinter as tk

class SysInfoGUI(Node):
    def __init__(self):
        super().__init__('sys_info_gui')
        self.subscription = self.create_subscription(
            SystemInfo,
            'system_info_topic',
            self.listener_callback,
            10)
        
        self.root = tk.Tk()
        self.root.title("ROS2 系统监控")
        self.root.geometry("400x350")
        
        self.labels = {}
        fields = [
            ("timestamp", "记录时间"), ("hostname", "主机名称"),
            ("cpu_usage", "CPU使用率 (%)"), ("mem_usage", "内存使用率 (%)"),
            ("mem_total", "内存总大小 (MB)"), ("mem_available", "剩余内存 (MB)"),
            ("net_recv", "网络接收量 (MB)"), ("net_sent", "网络发送量 (MB)")
        ]
        
        for i, (key, text) in enumerate(fields):
            tk.Label(self.root, text=text + ":", font=("Arial", 12, "bold")).grid(row=i, column=0, padx=20, pady=10, sticky="e")
            self.labels[key] = tk.Label(self.root, text="等待数据...", font=("Arial", 12))
            self.labels[key].grid(row=i, column=1, sticky="w")

        self.spin_ros()

    def listener_callback(self, msg):
        self.labels["timestamp"].config(text=msg.timestamp)
        self.labels["hostname"].config(text=msg.hostname)
        self.labels["cpu_usage"].config(text=f"{msg.cpu_usage:.1f}")
        self.labels["mem_usage"].config(text=f"{msg.mem_usage:.1f}")
        self.labels["mem_total"].config(text=f"{msg.mem_total:.1f}")
        self.labels["mem_available"].config(text=f"{msg.mem_available:.1f}")
        self.labels["net_recv"].config(text=f"{msg.net_recv:.1f}")
        self.labels["net_sent"].config(text=f"{msg.net_sent:.1f}")

    def spin_ros(self):
        rclpy.spin_once(self, timeout_sec=0.01)
        self.root.after(50, self.spin_ros)

def main(args=None):
    rclpy.init(args=args)
    gui_node = SysInfoGUI()
    gui_node.root.mainloop()
    gui_node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

**4. 赋予执行权限（关键）**
```bash
chmod +x ~/ros2_ws/src/sys_monitor_app/scripts/*.py
```

**5. 修改 `CMakeLists.txt`（注册节点）**
打开 `~/ros2_ws/src/sys_monitor_app/CMakeLists.txt`，替换为：
```cmake
cmake_minimum_required(VERSION 3.8)
project(sys_monitor_app)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclpy REQUIRED)
find_package(sys_info_interfaces REQUIRED)

# 安装并重命名 Python 脚本，使其可通过 ros2 run 直接调用
install(PROGRAMS scripts/sys_pub.py
  DESTINATION lib/${PROJECT_NAME}
  RENAME sys_pub
)

install(PROGRAMS scripts/gui_sub.py
  DESTINATION lib/${PROJECT_NAME}
  RENAME gui_sub
)

ament_package()
```

**6. 修改 `package.xml`**
打开 `~/ros2_ws/src/sys_monitor_app/package.xml`，添加 `rclpy` 和自定义消息的依赖：
```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>sys_monitor_app</name>
  <version>0.0.0</version>
  <description>System monitor app using CMake</description>
  <maintainer email="user@todo.todo">user</maintainer>
  <license>TODO: License declaration</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <depend>rclpy</depend>
  <depend>sys_info_interfaces</depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

---

### 第三阶段：编译与运行测试

**1. 彻底清理并编译整个工作空间**
如果你之前编译过旧的 `ament_python` 版本，必须先清理：
```bash
cd ~/ros2_ws
rm -rf build/ install/ log/
colcon build
```

**2. 运行发布端（在终端 A）**
```bash
source ~/ros2_ws/install/setup.bash
ros2 run sys_monitor_app sys_pub
```

**3. 运行接收端与界面（在终端 B）**
```bash
source ~/ros2_ws/install/setup.bash
ros2 run sys_monitor_app gui_sub
```

局域网注意事项（如果其他主机收不到数据）：

确保两台电脑关闭了防火墙（如 Ubuntu 执行 sudo ufw disable）。

确保两台电脑位于同一网段，且环境变量 ROS_DOMAIN_ID 保持一致（如未设置，默认一致）。如果有冲突，可以在两台电脑的 ~/.bashrc 中同时添加：export ROS_DOMAIN_ID=10（数字可以自定义）。
