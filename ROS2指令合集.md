这是一份结构化的 **ROS2 实用指令合集（Cheat Sheet）**。无论你是刚接触 ROS2 还是需要日常开发备忘，这些命令都能涵盖绝大多数高频使用场景。

---

### 1. 基础环境与编译 (Environment & Build)
在运行任何 ROS2 命令前，确保你已经 source 了基础环境和当前工作空间。

*   **配置基础环境：**
    ```bash
    source /opt/ros/<你的版本号>/setup.bash  # 例如: source /opt/ros/humble/setup.bash
    ```
*   **配置当前工作空间：**
    
```bash
    source install/setup.bash
    ```
*   **编译整个工作空间：**
    ```bash
    colcon build
    ```
*   **使用软链接编译（推荐用于 Python 或资源文件开发，免去重复编译）：**
    ```bash
    colcon build --symlink-install
    ```
*   **仅编译特定包：**
    ```bash
    colcon build --packages-select <package_name>
    ```

---

### 2. 节点运行与管理 (Nodes)
节点是 ROS2 中执行计算的独立进程。

*   **运行一个节点：**
    
```bash
    ros2 run <package_name> <executable_name>
    ```
*   **列出当前运行的所有节点：**
    ```bash
    ros2 node list
    ```
*   **查看特定节点的详细信息（订阅、发布、服务等）：**
    ```bash
    ros2 node info /<node_name>
    ```

---

### 3. 话题通信 (Topics)
话题是节点之间持续发布和订阅数据的主要方式。

*   **列出所有活动话题：**
    ```bash
    ros2 topic list
    ```
*   **列出包含消息类型的话题：**
    ```bash
    ros2 topic list -t
    ```
*   **实时打印话题数据（监听）：**
    ```bash
    ros2 topic echo /<topic_name>
    ```
*   **查看话题的详细信息（发布者和订阅者数量）：**
    ```bash
    ros2 topic info /<topic_name>
    ```
*   **查看话题的发布频率：**
    ```bash
    ros2 topic hz /<topic_name>
    ```
*   **手动向话题发布数据（按频率）：**
    ```bash
    ros2 topic pub --rate 1 /<topic_name> <msg_type> "{data: 'value'}"
    # 例如：ros2 topic pub --rate 1 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 2.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 1.8}}"
    ```

---

### 4. 服务调用 (Services)
服务用于节点间的同步/异步“请求-响应”通信。

*   **列出所有活动的服务：**
    ```bash
    ros2 service list
    ```
*   **查看服务的类型：**
    ```bash
    ros2 service type /<service_name>
    ```
*   **手动调用服务：**
    ```bash
    ros2 service call /<service_name> <service_type> "{request_data: 'value'}"
    # 例如清空小海龟的轨迹：ros2 service call /clear std_srvs/srv/Empty
    ```

---

### 5. 参数配置 (Parameters)
参数是节点的配置值（类似全局或局部变量）。

*   **列出所有节点的参数：**
    ```bash
    ros2 param list
    ```
*   **获取特定节点的某个参数值：**
    ```bash
    ros2 param get <node_name> <parameter_name>
    ```
*   **动态修改特定节点的某个参数值：**
    ```bash
    ros2 param set <node_name> <parameter_name> <value>
    ```
*   **将节点的所有参数导出为 YAML 文件：**
    ```bash
    ros2 param dump <node_name>
    ```

---

### 6. 动作管理 (Actions)
动作适用于长时间运行的任务，允许中途取消并能提供实时反馈（如导航到目标点）。

*   **列出所有活动的动作：**
    ```bash
    ros2 action list
    ```
*   **查看动作详细信息：**
    
```bash
    ros2 action info /<action_name>
    ```
*   **手动发送动作目标：**
    ```bash
    ros2 action send_goal /<action_name> <action_type> "{goal_data: 'value'}" --feedback
    ```

---

### 7. 启动文件与包管理 (Launch & Packages)

*   **启动 Launch 文件（一键启动多个节点和配置）：**
    ```bash
    ros2 launch <package_name> <launch_file_name.py>
    ```
*   **列出系统中所有的 ROS2 包：**
    ```bash
    ros2 pkg list
    ```
*   **创建一个新的 ROS2 包 (C++ / Python)：**
    ```bash
    ros2 pkg create --build-type ament_cmake <package_name>
    ros2 pkg create --build-type ament_python <package_name>
    ```
*   **列出包内的所有可执行文件：**
    ```bash
    ros2 pkg executables <package_name>
    ```
在 ROS2 中，`ros2 pkg create` 也就是创建功能包的命令，包含了非常丰富的参数。掌握这些参数可以帮你省去大量手动修改 `package.xml` 和 `CMakeLists.txt`（或 `setup.py`）的时间。

以下是 `ros2 pkg create <package_name>` 后面可以跟的完整参数列表，为了方便理解，我将它们分为了三大类：**构建与依赖**、**代码生成**、**元数据配置**。

### 一、 构建与依赖参数 (核心功能)

这类参数决定了你的包用什么语言写，以及依赖哪些其他的库。

| 参数 / 简写 | 作用说明 | 常见选项 / 示例 |
| :--- | :--- | :--- |
| `--build-type` | **[必选/极重要]** 指定包的构建系统。如果不加，ROS2 默认使用 `ament_cmake` (即 C++ 环境)。 | `ament_cmake` (C++推荐)<br>`ament_python` (Python推荐)<br>`cmake` (纯CMake项目) |
| `--dependencies` / `-d` | **[常用]** 声明此包依赖的其他 ROS2 包。系统会自动将它们写入 `package.xml` 和构建脚本中。 | `-d rclcpp std_msgs sensor_msgs`<br>可以一次性跟多个包名，用空格隔开。 |
| `--destination-directory` | 指定功能包创建的路径。如果不填，默认就在你当前所处的终端目录下创建。 | `--destination-directory ~/ros2_ws/src` |

---

### 二、 代码自动生成参数 (提效利器)

如果你想在建包的同时，顺手把 Hello World 级别的节点或者库文件也建好，并自动配置好编译规则，用这两个参数：

| 参数 | 作用说明 | 适用场景 |
| :--- | :--- | :--- |
| `--node-name` | 自动在包内生成一个基础的节点源文件（`.cpp` 或 `.py`），并**自动帮你写好** `CMakeLists.txt` 或 `setup.py` 里的可执行文件（Executable）注册代码。 | `--node-name my_first_node`<br>非常适合快速验证想法，省去手动注册节点的麻烦。 |
| `--library-name` | 仅在 C++ (`ament_cmake`) 下有效。自动生成一个库文件（`.hpp` 和 `.cpp`）框架，并配置好导出规则。 | 开发底层算法或被其他包调用的共享库时使用。 |

---

### 三、 元数据配置参数 (工程规范)

这些参数主要用于填充 `package.xml` 中的信息。在做大型项目或者准备开源、打比赛提交代码时，规范填写这些信息是基本功。

| 参数 | 作用说明 | 示例 |
| :--- | :--- | :--- |
| `--license` | **[推荐]** 声明代码的开源许可证。 | `--license Apache-2.0` (ROS2默认)<br>`--license MIT`<br>`--license GPL-3.0` |
| `--maintainer-name` | 维护者姓名。如果不填，会自动抓取你当前 Ubuntu 系统的用户名。 | `--maintainer-name "Your Name"` |
| `--maintainer-email` | 维护者邮箱。 | `--maintainer-email "xxx@example.com"` |
| `--description` | 功能包的一句话简介。 | `--description "Object detection node using YOLO"` |

---

### 💡 黄金组合示例

在实际的机器人开发中，我们通常不会把所有参数都敲一遍，而是使用以下两种最典型的“黄金组合”模板：

**模板 1：创建一个标准的 C++ 节点包 (例如做视觉识别或底层控制)**
```bash
ros2 pkg create cv_detection_node --build-type ament_cmake --dependencies rclcpp sensor_msgs cv_bridge --node-name detector --license Apache-2.0
```
*执行后：你直接去 `src/detector.cpp` 里写 OpenCV 和 ROS2 图像转换的逻辑，然后 `colcon build` 就能直接运行，完全不用管 CMake 怎么写。*

**模板 2：创建一个标准的 Python 节点包 (例如做上层逻辑或深度学习推理)**
```bash
ros2 pkg create ai_decision_node --build-type ament_python --dependencies rclpy std_msgs geometry_msgs --node-name planner --license Apache-2.0
```
*执行后：直接打开生成的 `planner.py` 编写你的 Python 节点逻辑即可。*
---

### 8. 数据记录与回放 (rosbag2)
用于录制真实传感器数据并在后期回放仿真。

*   **录制所有话题：**
    ```bash
    ros2 bag record -a
    ```
*   **录制特定话题：**
    ```bash
    ros2 bag record /<topic_1> /<topic_2>
    ```
*   **回放录制的数据包：**
    ```bash
    ros2 bag play <bag_directory_or_file>
    ```
*   **查看数据包信息：**
    ```bash
    ros2 bag info <bag_directory_or_file>
    ```

> **💡 终极提示：**
> 如果你忘记了某个具体指令的用法，随时在命令后加上 `-h` 或 `--help`，例如：`ros2 topic pub --help`，ROS2 会提供非常详细的语法格式和示例。
```
