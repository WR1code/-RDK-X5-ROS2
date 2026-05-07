在 ROS 2 (Robot Operating System 2) 中，**`ros2 bag`** 是一个极其核心且强大的命令行工具。它的主要功能是**记录（Record）**和**回放（Play）** ROS 2 系统中的主题（Topic）数据。

无论是进行算法调试、机器学习数据采集，还是复现机器人运行时的真实场景，`ros2 bag` 都是不可或缺的。与 ROS 1 不同，ROS 2 默认使用 SQLite3 数据库格式来存储数据，并且在较新的版本（如 Iron, Jazzy）中默认或广泛支持性能更佳的 **MCAP** 格式。

以下是 `ros2 bag` 的详细使用指南：

---

### 1. 基础准备
在使用之前，请确保你已经启动了 ROS 2 的环境：
```bash
source /opt/ros/<你的ROS2版本>/setup.bash
# 例如：source /opt/ros/humble/setup.bash
```

---

### 2. 记录数据 (`ros2 bag record`)

你可以将当前 ROS 2 网络中正在发布的主题数据录制下来，保存为一个文件夹（包含数据库文件和元数据文件）。

*   **录制单个或多个指定主题：**
    ```bash
    ros2 bag record /topic1 /topic2
    ```
    *系统会自动生成一个以时间戳命名的文件夹（例如：`rosbag2_2023_10_26-15_30_00`）来保存数据。*

*   **自定义保存的文件夹名称 (`-o` / `--output`)：**
    ```bash
    ros2 bag record -o my_custom_bag /topic_name
    ```

*   **录制所有活动主题 (`-a` / `--all`)：**
    ```bash
    ros2 bag record -a
    ```
    *⚠️ 注意：录制所有主题可能会产生非常庞大的文件，尤其是包含高频摄像头图像或点云数据时，建议谨慎使用。*

*   **指定存储格式 (`-s` / `--storage`)：**
    ROS 2 支持不同的存储插件。如果你想使用更高性能的 MCAP 格式（推荐用于大型数据）：
    ```bash
    ros2 bag record -s mcap -a
    ```

*   **录制特定类型的主题 (`-e` / `--regex`)：**
    支持使用正则表达式来匹配要录制的主题：
    ```bash
    ros2 bag record -e "/sensor/.*"
    ```

---

### 3. 查看 Bag 文件信息 (`ros2 bag info`)

录制完成后，你可以不播放数据，直接查看 bag 文件的元数据（包含多少条消息、时长、包含哪些主题及其数据类型等）。
```bash
ros2 bag info my_custom_bag
```

**输出示例说明：**
你可以看到 `Files` (文件路径), `Duration` (录制时长), `Messages` (总消息数), 以及 `Topic information` (各个主题的详细统计)。这对于确认数据是否成功录制非常有用。

---

### 4. 回放数据 (`ros2 bag play`)

将录制好的数据按照原有的时间戳重新发布到 ROS 2 网络中，这使得其他节点感觉就像在和真实的硬件交互一样。

*   **基础回放：**
    ```bash
    ros2 bag play my_custom_bag
    ```

*   **循环回放 (`-l` / `--loop`)：**
    数据播放完毕后自动从头开始，非常适合持续调试某个节点：
    ```bash
    ros2 bag play -l my_custom_bag
    ```

*   **调整回放速率 (`-r` / `--rate`)：**
    可以加速或减速播放数据。例如，以 2 倍速播放：
    ```bash
    ros2 bag play -r 2.0 my_custom_bag
    ```
    以 0.5 倍速（慢动作）播放：
    
```bash
    ros2 bag play -r 0.5 my_custom_bag
    ```

*   **只回放特定的主题：**
    如果你录制了很多主题，但在调试时只想回放其中几个：
    ```bash
    ros2 bag play my_custom_bag --topics /topic1 /topic2
    ```

*   **从特定的时间点开始回放 (`--start-offset`)：**
    跳过前面的无用数据，从录制开始后的第 N 秒播放（例如跳过前 10 秒）：
    ```bash
    ros2 bag play my_custom_bag --start-offset 10
    ```

---

### 5. 高级技巧与常见问题

*   **QoS (服务质量) 不匹配问题：**
    ROS 2 引入了 QoS 机制。如果在回放 bag 文件时，你的接收节点收不到数据，很可能是因为你的节点使用了 `Best Effort` 的可靠性，而 rosbag 默认以 `Reliable` 发布（或者反过来）。
    如果遇到这种情况，可以使用 `--qos-profile-overrides-path` 传入一个 YAML 文件来覆盖回放时的 QoS 设置，或者在你的代码中调整订阅者的 QoS 策略。
*   **压缩 Bag 文件：**
    由于未经压缩的数据包可能非常大，在 ROS 2 中可以在记录时进行压缩（目前通常需要安装额外的包，如 `ros-humble-rosbag2-compression-zstd`）：
    ```bash
    ros2 bag record --compression-mode file --compression-format zstd -a
    ```
*   **存储格式转换：**
    由于 ROS 1 ( `.bag` ) 和 ROS 2 (文件夹/SQLite3/MCAP) 格式不同，如果需要互通数据，通常需要使用 `rostopic` 桥接（`ros1_bridge`）或者专用的格式转换工具（如 `mcap` 官方提供的 CLI 工具）。

掌握 `ros2 bag` 的这些命令，能够极大地提升你在机器人软件开发和测试过程中的效率。
```
