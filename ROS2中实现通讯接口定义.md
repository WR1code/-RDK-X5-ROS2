为了满足您的需求，我们将采用**纯 C++ (ament_cmake)** 的方式来构建整个 ROS 2 工作空间。这样可以完全避免使用 `setup.py` 和混合编译带来的麻烦。图形界面将使用 **Qt5** 和 C++ 实现。

整体流程分为以下几步：
1. 创建自定义消息文件（`.msg`）。
2. 编写 C++ 发布者节点（读取系统 /proc 和 sysinfo 数据）。
3. 编写 C++ Qt5 订阅者节点（显示 GUI 界面）。
4. 配置纯 C++ 的 `CMakeLists.txt`。
5. 编译运行与局域网通信配置。

以下是完整的操作流程和代码：

### 第一步：创建工作空间和功能包

打开终端，创建一个新的工作空间并生成一个纯 C++ 的包：

```bash
mkdir -p ~/sys_monitor_ws/src
cd ~/sys_monitor_ws/src
# 创建纯 C++ 包，依赖 rclcpp 和消息生成相关库
ros2 pkg create --build-type ament_cmake sys_monitor_pkg --dependencies rclcpp std_msgs rosidl_default_generators
```

### 第二步：定义自定义消息接口

在包目录下创建 `msg` 文件夹并编写接口文件。

```bash
cd ~/sys_monitor_ws/src/sys_monitor_pkg
mkdir msg
```

创建 `msg/SysStatus.msg` 文件，内容如下：

```text
# msg/SysStatus.msg
string timestamp       # 记录信息的时间
string hostname        # 主机名称
float32 cpu_usage      # CPU使用率 (%)
float32 mem_usage      # 内存使用率 (%)
float32 mem_total      # 内存总大小 (MB)
float32 mem_free       # 剩余内存 (MB)
uint64 net_rx          # 网络接收数据量 (Bytes)
uint64 net_tx          # 网络发送数据量 (Bytes)
```

### 第三步：编写 C++ 发布者 (系统状态采集)

在 `src` 目录下创建 `sys_pub.cpp`。这个节点将利用 Linux 底层 API (`sysinfo`, `/proc/stat`, `/proc/net/dev`) 获取实时数据。

```cpp
// src/sys_pub.cpp
#include <rclcpp/rclcpp.hpp>
#include "sys_monitor_pkg/msg/sys_status.hpp"
#include <sys/sysinfo.h>
#include <unistd.h>
#include <fstream>
#include <sstream>
#include <iomanip>

class SysMonitorPub : public rclcpp::Node {
public:
    SysMonitorPub() : Node("sys_monitor_pub"), last_total_cpu(0), last_idle_cpu(0) {
        publisher_ = this->create_publisher<sys_monitor_pkg::msg::SysStatus>("sys_status", 10);
        timer_ = this->create_wall_timer(
            std::chrono::seconds(1),
            std::bind(&SysMonitorPub::timer_callback, this));
    }

private:
    void timer_callback() {
        auto msg = sys_monitor_pkg::msg::SysStatus();

        // 1. 获取时间
        auto now = std::chrono::system_clock::now();
        std::time_t now_c = std::chrono::system_clock::to_time_t(now);
        std::stringstream ss;
        ss << std::put_time(std::localtime(&now_c), "%Y-%m-%d %H:%M:%S");
        msg.timestamp = ss.str();

        // 2. 获取主机名
        char hostname[256];
        gethostname(hostname, sizeof(hostname));
        msg.hostname = std::string(hostname);

        // 3. 获取内存信息
        struct sysinfo memInfo;
        sysinfo(&memInfo);
        long long totalPhysMem = memInfo.totalram * memInfo.mem_unit;
        long long physMemUsed = totalPhysMem - (memInfo.freeram * memInfo.mem_unit);
        msg.mem_total = totalPhysMem / (1024.0 * 1024.0);
        msg.mem_free = (memInfo.freeram * memInfo.mem_unit) / (1024.0 * 1024.0);
        msg.mem_usage = (physMemUsed / (double)totalPhysMem) * 100.0;

        // 4. 获取CPU使用率 (解析 /proc/stat)
        std::ifstream stat_file("/proc/stat");
        std::string line;
        std::getline(stat_file, line);
        std::istringstream iss(line);
        std::string cpu_label;
        long user, nice, system, idle, iowait, irq, softirq;
        iss >> cpu_label >> user >> nice >> system >> idle >> iowait >> irq >> softirq;
        long total_cpu = user + nice + system + idle + iowait + irq + softirq;
        long current_idle = idle + iowait;
        
        if (last_total_cpu != 0) {
            long total_diff = total_cpu - last_total_cpu;
            long idle_diff = current_idle - last_idle_cpu;
            msg.cpu_usage = (1.0 - (double)idle_diff / total_diff) * 100.0;
        } else {
            msg.cpu_usage = 0.0;
        }
        last_total_cpu = total_cpu;
        last_idle_cpu = current_idle;

        // 5. 获取网络收发数据 (解析 /proc/net/dev，汇总非lo网卡)
        std::ifstream net_file("/proc/net/dev");
        uint64_t rx_bytes = 0, tx_bytes = 0;
        while (std::getline(net_file, line)) {
            if (line.find("lo:") == std::string::npos && line.find(":") != std::string::npos) {
                size_t colon_pos = line.find(":");
                std::istringstream net_iss(line.substr(colon_pos + 1));
                uint64_t rx, tx;
                long dummy;
                // rx_bytes 是第1个，tx_bytes 是第9个
                net_iss >> rx >> dummy >> dummy >> dummy >> dummy >> dummy >> dummy >> dummy >> tx;
                rx_bytes += rx;
                tx_bytes += tx;
            }
        }
        msg.net_rx = rx_bytes;
        msg.net_tx = tx_bytes;

        publisher_->publish(msg);
    }

    rclcpp::Publisher<sys_monitor_pkg::msg::SysStatus>::SharedPtr publisher_;
    rclcpp::TimerBase::SharedPtr timer_;
    long last_total_cpu;
    long last_idle_cpu;
};

int main(int argc, char * argv[]) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<SysMonitorPub>());
    rclcpp::shutdown();
    return 0;
}
```

### 第四步：编写 C++ Qt5 订阅者 (GUI 界面)

在 `src` 目录下创建 `sys_gui.cpp`。这里使用 `QTimer` 来安全地在 Qt 的主线程中调用 `rclcpp::spin_some`，完美融合 ROS 2 和 Qt 的事件循环。

```cpp
// src/sys_gui.cpp
#include <rclcpp/rclcpp.hpp>
#include "sys_monitor_pkg/msg/sys_status.hpp"

#include <QApplication>
#include <QWidget>
#include <QVBoxLayout>
#include <QLabel>
#include <QTimer>
#include <QString>

class SysGuiNode : public rclcpp::Node {
public:
    SysGuiNode(QLabel* label) : Node("sys_gui_node"), label_(label) {
        subscription_ = this->create_subscription<sys_monitor_pkg::msg::SysStatus>(
            "sys_status", 10, std::bind(&SysGuiNode::topic_callback, this, std::placeholders::_1));
    }

private:
    void topic_callback(const sys_monitor_pkg::msg::SysStatus::SharedPtr msg) const {
        QString text = QString(
            "<b>=== 系统实时状态 ===</b><br><br>"
            "时间: %1<br>"
            "主机名称: %2<br>"
            "CPU 使用率: %3 %<br>"
            "内存 使用率: %4 %<br>"
            "内存 总大小: %5 MB<br>"
            "剩余 内存: %6 MB<br>"
            "网络 接收数据: %7 Bytes<br>"
            "网络 发送数据: %8 Bytes"
        ).arg(msg->timestamp.c_str())
         .arg(msg->hostname.c_str())
         .arg(msg->cpu_usage, 0, 'f', 2)
         .arg(msg->mem_usage, 0, 'f', 2)
         .arg(msg->mem_total, 0, 'f', 2)
         .arg(msg->mem_free, 0, 'f', 2)
         .arg(msg->net_rx)
         .arg(msg->net_tx);
         
        label_->setText(text);
    }

    rclcpp::Subscription<sys_monitor_pkg::msg::SysStatus>::SharedPtr subscription_;
    QLabel* label_;
};

int main(int argc, char * argv[]) {
    // 1. 初始化 ROS 2
    rclcpp::init(argc, argv);

    // 2. 初始化 Qt 应用
    QApplication app(argc, argv);
    QWidget window;
    window.setWindowTitle("ROS 2 系统监控大屏");
    window.resize(350, 250);

    QVBoxLayout *layout = new QVBoxLayout(&window);
    QLabel *info_label = new QLabel("等待接收系统数据...", &window);
    info_label->setStyleSheet("font-size: 14px;");
    layout->addWidget(info_label);
    window.setLayout(layout);

    // 3. 创建 ROS 2 节点
    auto node = std::make_shared<SysGuiNode>(info_label);

    // 4. 使用 QTimer 触发 ROS 2 的事件循环
    QTimer *ros_timer = new QTimer(&window);
    QObject::connect(ros_timer, &QTimer::timeout, [&node]() {
        rclcpp::spin_some(node); // 非阻塞式处理 ROS 回调
    });
    ros_timer->start(10); // 每 10ms 触发一次

    window.show();

    // 5. 启动 Qt 事件循环
    int result = app.exec();

    // 6. 清理退出
    rclcpp::shutdown();
    return result;
}
```

### 第五步：配置纯 CMakeLists.txt 和 package.xml

这是最关键的一步，它将自定义消息、Qt5 和 C++ 节点关联在一起，完全不需要 `setup.py`。

修改 `sys_monitor_pkg/package.xml`，确保包含以下依赖：
```xml
  <buildtool_depend>ament_cmake</buildtool_depend>
  <depend>rclcpp</depend>
  <depend>std_msgs</depend>
  
  <!-- 消息生成依赖 -->
  <build_depend>rosidl_default_generators</build_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>
```

修改 `sys_monitor_pkg/CMakeLists.txt`，完整替换为以下内容：

```cmake
cmake_minimum_required(VERSION 3.8)
project(sys_monitor_pkg)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 1. 寻找 ROS 2 依赖
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(rosidl_default_generators REQUIRED)

# 2. 寻找 Qt5 依赖
find_package(Qt5 COMPONENTS Widgets REQUIRED)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

# 3. 声明和生成自定义消息
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/SysStatus.msg"
)

# 4. 编译发布者节点
add_executable(sys_pub src/sys_pub.cpp)
ament_target_dependencies(sys_pub rclcpp std_msgs)
# 将生成的消息头文件链接到发布者
rosidl_target_interfaces(sys_pub ${PROJECT_NAME} "rosidl_typesupport_cpp")

# 5. 编译 GUI 订阅者节点
add_executable(sys_gui src/sys_gui.cpp)
target_link_libraries(sys_gui Qt5::Widgets)
ament_target_dependencies(sys_gui rclcpp std_msgs)
# 将生成的消息头文件链接到订阅者
rosidl_target_interfaces(sys_gui ${PROJECT_NAME} "rosidl_typesupport_cpp")

# 6. 安装规则
install(TARGETS
  sys_pub
  sys_gui
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

### 第六步：编译与运行

由于您的系统是 Ubuntu，请确保已安装 Qt5 开发库：
```bash
sudo apt update
sudo apt install qtbase5-dev qt5-qmake
```

然后回到工作空间根目录进行编译：
```bash
cd ~/sys_monitor_ws
colcon build --packages-select sys_monitor_pkg
```

**运行测试：**

打开终端 1（运行数据采集节点）：
```bash
source ~/sys_monitor_ws/install/setup.bash
ros2 run sys_monitor_pkg sys_pub
```

打开终端 2（运行 Qt GUI 节点）：
```bash
source ~/sys_monitor_ws/install/setup.bash
ros2 run sys_monitor_pkg sys_gui
```

此时会弹出一个界面，实时刷新系统的各项占用指标。

### 关于“在局域网内其他主机上查看数据”的说明

由于 ROS 2 底层使用的是 DDS（Data Distribution Service），**局域网内的通信是原生自带的**。您完全不需要写 socket 或配置额外的路由，只需要满足以下两个条件：

1. **同一网段**：运行发布者节点（如树莓派/工控机）和运行 Qt GUI 界面节点（如您的笔记本）的设备连接在同一个路由器下（同一个 Wi-Fi 或有线局域网）。
2. **相同的 ROS_DOMAIN_ID**：在两台电脑的 `.bashrc` 中确保它们具有相同的域 ID（默认为 0，如果不冲突可以不设置）。
   ```bash
   # 在两台主机的终端中均执行，或者加入到 ~/.bashrc
   export ROS_DOMAIN_ID=10  
   ```

此时，在**主机 A** 上运行 `ros2 run sys_monitor_pkg sys_pub`，在**主机 B** 上运行 `ros2 run sys_monitor_pkg sys_gui`，Qt 界面就可以直接跨主机显示 A 的系统状态了。
```
