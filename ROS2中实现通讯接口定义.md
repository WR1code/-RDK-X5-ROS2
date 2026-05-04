这是一个非常典型的 ROS2 应用场景！根据你图片中的需求，我们需要完成以下几个核心任务：

1. **定义自定义消息接口 (`.msg`)**：包含时间、主机名、CPU、内存、网络等字段。
2. **编写信息采集与发布节点**：使用 Python 的 `psutil` 库获取系统真实数据，并通过 ROS2 发布。
3. **编写带 GUI 界面的订阅节点**：使用 Python 内置的 `tkinter` 库编写一个简单的图形界面来实时显示数据。
4. **局域网通信**：利用 ROS2 底层的 DDS 机制实现跨主机通信。

下面是完整的从零开始的操作流程和代码。

---

### 第一步：创建自定义消息接口包

ROS2 推荐将自定义消息（Interfaces）单独放在一个包中（使用 CMake 编译）。

1. **创建工作空间和消息包：**
   打开终端，运行以下命令：
   ```bash
   mkdir -p ~/ros2_ws/src
   cd ~/ros2_ws/src
   ros2 pkg create --build-type ament_cmake sys_info_interfaces
   cd sys_info_interfaces
   mkdir msg
   
```

2. **编写自定义消息文件：**
   在 `msg` 目录下新建文件 `SystemInfo.msg`，填入以下内容（对应图中的需求）：
   ```text
   string timestamp         # 记录信息的时间
   string hostname          # 主机名称
   float32 cpu_usage        # CPU使用率 (%)
   float32 mem_usage        # 内存使用率 (%)
   float32 mem_total        # 内存总大小 (MB)
   float32 mem_available    # 剩余内存 (MB)
   float32 net_recv         # 网络接收数据量 (MB)
   float32 net_sent         # 网络发送数据量 (MB)
   ```

3. **修改 `CMakeLists.txt`：**
   打开 `sys_info_interfaces/CMakeLists.txt`，在 `find_package(ament_cmake REQUIRED)` 后面添加以下内容：
   ```cmake
   find_package(rosidl_default_generators REQUIRED)

   rosidl_generate_interfaces(${PROJECT_NAME}
     "msg/SystemInfo.msg"
   )
   ```

4. **修改 `package.xml`：**
   打开 `sys_info_interfaces/package.xml`，添加以下依赖声明：
   ```xml
   <build_depend>rosidl_default_generators</build_depend>
   <exec_depend>rosidl_default_runtime</exec_depend>
   <member_of_group>rosidl_interface_packages</member_of_group>
   ```

5. **编译消息包：**
   ```bash
   cd ~/ros2_ws
   colcon build --packages-select sys_info_interfaces
   source install/setup.bash
   ```

---

### 第二步：创建功能包（采集节点与GUI节点）

我们将使用 Python 来编写业务逻辑。为了获取系统信息，你需要安装 `psutil` 库（如果还没有的话）：
```bash
pip3 install psutil
```

1. **创建 Python 功能包：**
   ```bash
   cd ~/ros2_ws/src
   ros2 pkg create --build-type ament_python sys_monitor_app --dependencies rclpy sys_info_interfaces
   ```

2. **编写信息采集与发布节点 (`publisher`)：**
   在 `sys_monitor_app/sys_monitor_app/` 目录下新建 `sys_pub.py`：

   ```python
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
           self.timer = self.create_timer(1.0, self.timer_callback) # 每秒发布一次
           self.hostname = socket.gethostname()

       def timer_callback(self):
           msg = SystemInfo()
           
           # 获取当前时间
           msg.timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
           msg.hostname = self.hostname
           
           # CPU 和 内存
           msg.cpu_usage = float(psutil.cpu_percent(interval=None))
           mem = psutil.virtual_memory()
           msg.mem_usage = float(mem.percent)
           msg.mem_total = float(mem.total / (1024 * 1024))         # 转换为 MB
           msg.mem_available = float(mem.available / (1024 * 1024)) # 转换为 MB
           
           # 网络 IO (累计值)
           net = psutil.net_io_counters()
           msg.net_recv = float(net.bytes_recv / (1024 * 1024))     # 转换为 MB
           msg.net_sent = float(net.bytes_sent / (1024 * 1024))     # 转换为 MB

           self.publisher_.publish(msg)
           self.get_logger().info(f'发布系统状态: CPU {msg.cpu_usage}%, 内存 {msg.mem_usage}%')

   def main(args=None):
       rclpy.init(args=args)
       node = SysInfoPublisher()
       rclpy.spin(node)
       node.destroy_node()
       rclpy.shutdown()

   if __name__ == '__main__':
       main()
   ```

3. **编写带 GUI 的接收节点 (`subscriber`)：**
   在同目录下新建 `gui_sub.py`。这里使用 Python 自带的 `tkinter`，并使用非阻塞的方式让 ROS2 和 GUI 共存：

   ```python
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
           
           # 初始化 GUI
           self.root = tk.Tk()
           self.root.title("ROS2 系统状态监控器")
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

           # 启动 ROS2 节点的自旋刷新循环
           self.spin_ros()

       def listener_callback(self, msg):
           # 更新 GUI 上的标签
           self.labels["timestamp"].config(text=msg.timestamp)
           self.labels["hostname"].config(text=msg.hostname)
           self.labels["cpu_usage"].config(text=f"{msg.cpu_usage:.1f}")
           self.labels["mem_usage"].config(text=f"{msg.mem_usage:.1f}")
           self.labels["mem_total"].config(text=f"{msg.mem_total:.1f}")
           self.labels["mem_available"].config(text=f"{msg.mem_available:.1f}")
           self.labels["net_recv"].config(text=f"{msg.net_recv:.1f}")
           self.labels["net_sent"].config(text=f"{msg.net_sent:.1f}")

       def spin_ros(self):
           # 每次处理挂起的 ROS 事件（非阻塞）
           rclpy.spin_once(self, timeout_sec=0.01)
           # 利用 tkinter 的定时器循环调用自身
           self.root.after(50, self.spin_ros)

   def main(args=None):
       rclpy.init(args=args)
       gui_node = SysInfoGUI()
       gui_node.root.mainloop()  # 启动 GUI 主循环
       
       gui_node.destroy_node()
       rclpy.shutdown()

   if __name__ == '__main__':
       main()
   ```

4. **配置 `setup.py` 注册节点：**
   打开 `sys_monitor_app/setup.py`，找到 `entry_points`，添加启动入口：
   ```python
   entry_points={
       'console_scripts': [
           'sys_pub = sys_monitor_app.sys_pub:main',
           'gui_sub = sys_monitor_app.gui_sub:main',
       ],
   },
   ```

5. **编译功能包：**
   ```bash
   cd ~/ros2_ws
   colcon build --packages-select sys_monitor_app
   source install/setup.bash
   ```

---

### 第三步：运行与局域网通信测试（满足要求三）

**1. 在主机 A（被监控机器）上启动发送节点：**
打开终端，运行：
```bash
source ~/ros2_ws/install/setup.bash
ros2 run sys_monitor_app sys_pub
```
此时终端会每秒打印一次发送日志。

**2. 在本机或局域网内的其他机器 B 启动 GUI 节点：**
*注意：ROS2 默认基于 DDS 协议，同一局域网下只要 `ROS_DOMAIN_ID` 相同（默认均为0），不同主机之间会自动发现并通信，无需专门配置 IP！*

确保机器 B 也在同一个局域网（连着同一个WiFi或交换机），把编译好的工作空间拷贝过去（或用同样方法编译一遍），打开终端运行：
```bash
source ~/ros2_ws/install/setup.bash
ros2 run sys_monitor_app gui_sub
```
此时会弹出一个图形界面，实时跳动显示主机 A 的各项硬件数据。

**局域网注意事项（如果其他主机收不到数据）：**
1. 确保两台电脑关闭了防火墙（如 Ubuntu 执行 `sudo ufw disable`）。
2. 确保两台电脑位于同一网段，且环境变量 `ROS_DOMAIN_ID` 保持一致（如未设置，默认一致）。如果有冲突，可以在两台电脑的 `~/.bashrc` 中同时添加：`export ROS_DOMAIN_ID=10`（数字可以自定义）。
```
