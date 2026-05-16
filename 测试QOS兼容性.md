在开发像“社区智能治理机器人”这样需要依靠无线网络（Wi-Fi/5G）进行图传和导航的系统时，QoS测试是极其关键的一环。因为在理想的本地网络下，无论你怎么设置QoS，效果看起来都一样；只有在网络拥堵或弱网环境下，QoS的威力才会显现。

完整的QoS测试通常分为三个阶段：**基准验证**、**弱网模拟**和**代码级事件监控**。以下是在Ubuntu环境下进行ROS 2 QoS测试的详细流程：

### 第一阶段：使用自带工具进行基准验证

在写代码之前，先用ROS 2自带的 `demo_nodes` 来快速验证不同QoS策略组合的表象。

**1. 启动带特定QoS参数的节点**
你可以通过命令行参数直接修改官方demo节点的QoS。打开两个终端：

**终端 1 (发布者 - 模拟Best Effort):**

```bash
ros2 run demo_nodes_cpp talker --ros-args -p qos_reliability:=best_effort

```

**终端 2 (订阅者 - 模拟Reliable):**

```bash
ros2 run demo_nodes_cpp listener --ros-args -p qos_reliability:=reliable

```

*现象：* 你会发现Listener什么也收不到。此时使用 `ros2 topic info /chatter --verbose` 查看，就能直观地看到QoS不兼容的状态。

**2. 测试 Durability (持久性)**
测试 `Transient Local` (发给晚加入的节点)。

* 先启动发布者（比如发布一个模拟的全局代价地图），让它发几条消息后停止（或者保持运行）。
* 再启动订阅者，如果QoS是 `Volatile`，订阅者收不到历史数据；如果是 `Transient Local`，订阅者一启动就会立刻打印出之前缓存的最新消息。

---

### 第二阶段：弱网环境模拟（最核心的测试）

如前所述，不丢包的网络测不出 `Reliable` 和 `Best Effort` 的区别。在Ubuntu中，我们可以使用强大的 **`tc` (Traffic Control)** 和 **`netem`** 模块来人为制造恶劣网络。

> **⚠️ 注意：** 以下命令会修改你系统的网络行为，测试完成后务必记得恢复！这里以本地回环网卡 `lo` 为例。

**1. 制造丢包 (Packet Loss)**
假设你的YOLOv11正在通过图像话题高频发图：

```bash
# 在本地回环网卡上增加 15% 的随机丢包率
sudo tc qdisc add dev lo root netem loss 15%

```

* **测试 Reliable：** 消息序列号（Sequence）不会断，但你会感觉到明显的**延迟和卡顿**（因为底层DDS在疯狂重传）。
* **测试 Best Effort：** 画面保持**低延迟和实时**，但如果你看话题的序列号，会发现中间跳号了（比如1, 2, 4, 7... 丢掉的帧直接被放弃了）。

**2. 制造高延迟 (Latency)**
测试导航控制指令在网络延迟下的表现：

```bash
# 增加 200ms 的网络延迟，波动正负 50ms
sudo tc qdisc change dev lo root netem delay 200ms 50ms

```

**3. 测试完毕，恢复网络（非常重要！）**

```bash
# 清除本地回环网卡上的所有限制
sudo tc qdisc del dev lo root

```

---

### 第三阶段：代码级的 QoS 事件监控 (C++)

在实际的机器人工程开发中，我们不能一直靠盯着终端看。ROS 2 允许我们在代码中绑定 **QoS Event Callbacks（QoS事件回调）**，让机器人在通信出问题时自动报警或触发安全机制。

这在参加大型比赛或实际巡逻时非常有用。例如，当底盘控制器超过200ms没收到主控的 `cmd_vel` 时，必须立刻刹车。

**C++ 监控 Deadline Missed (错过期限) 的示例思路：**

在创建Publisher或Subscriber时，可以配置 `QoS Deadline` 并绑定事件回调。

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class QosMonitorNode : public rclcpp::Node {
public:
  QosMonitorNode() : Node("qos_monitor_node") {
    
    // 1. 设置 QoS 策略，要求每 100ms 至少收到一次消息
    rclcpp::QoS qos_profile(10);
    qos_profile.deadline(100ms); 

    // 2. 配置 QoS 事件选项
    rclcpp::SubscriptionOptions options;
    options.event_callbacks.deadline_callback = 
      [this](rclcpp::QoSDeadlineRequestedInfo & info) {
        RCLCPP_WARN(this->get_logger(), 
          "警告: 发生网络卡顿或节点宕机！已连续 %d 次错过 100ms 的最后期限！", 
          info.total_count);
        // 在这里可以调用底盘急停的函数
      };

    // 3. 创建订阅者
    sub_ = this->create_subscription<std_msgs::msg::String>(
      "important_control_cmd", qos_profile, 
      [this](const std_msgs::msg::String::SharedPtr msg) {
        // 正常收到消息的处理逻辑
      }, 
      options);
  }

private:
  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr sub_;
};

```

### 💡 实战测试建议清单

针对你的社区巡逻机器人项目，建议进行以下三个维度的针对性测试：

1. **雷达/视觉里程计 (高频并发测试):** 将大流量的 YOLO 检测框话题和雷达点云均设为 `Best Effort`，使用 `tc` 施加 10% 丢包，运行系统，观察 CPU 占用率是否比 `Reliable` 时显著下降（减少了无用的重传开销）。
2. **建图保存 (持久性测试):** 将建图节点的地图发布话题设为 `Transient Local`。让机器人跑一圈建好图，**杀掉**并重新启动 Rviz2，观察地图是否能瞬间重新加载出来。
3. **急停安全验证 (Deadline 测试):** 结合上面的 C++ 代码，用 `tc` 突然施加 500ms 的巨大延迟，测试你的底盘节点是否能准确触发 Deadline 回调并抱死电机。







在 ROS 2 的实际开发中，最让人头疼的往往是“节点都在运行，话题也能看到，但就是收不到数据”的幽灵Bug。这绝大多数是因为发布者和订阅者的 QoS 不兼容导致的默默失败（Silent Failure）。

为了进行专业的 QoS 测试，我们不仅要设置 QoS，更重要的是**利用代码级“QoS 事件回调（QoS Event Callbacks）”来捕获这些异常**。

下面我为你分别用 **Python** 和 **C++** 编写一套极其典型的 QoS 测试代码。

**测试场景设计：**
我们将**故意制造一个 QoS 冲突**。

* **发布者 (Publisher)：** 提供 `Best Effort`（尽力而为）的低质量服务。
* **订阅者 (Subscriber)：** 要求 `Reliable`（绝对可靠）的高质量服务。
* **预期结果：** 通信建立失败。但因为我们写了监控代码，程序会自动报警，告诉你到底是哪一项 QoS 导致了不兼容。

---

### 一、 Python 版本 (rclpy) 测试代码

Python 开发效率高，适合快速验证。在 ROS 2 Python 中，需要导入 `rclpy.qos` 和 `rclpy.qos_event`。

#### 1. Python 发布者 (qos_test_pub.py)

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

# 导入 QoS 相关的配置类
from rclpy.qos import QoSProfile, QoSReliabilityPolicy, QoSHistoryPolicy
# 导入 QoS 事件回调类
from rclpy.qos_event import PublisherEventCallbacks

class QosTestPublisher(Node):
    def __init__(self):
        super().__init__('qos_test_pub')
        
        # 1. 定义 QoS 策略：尽力而为 (Best Effort)
        qos_profile = QoSProfile(
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=10,
            reliability=QoSReliabilityPolicy.BEST_EFFORT # <--- 故意设置为低要求
        )

        # 2. 绑定 QoS 异常事件回调
        event_callbacks = PublisherEventCallbacks(
            incompatible_qos=self.on_incompatible_qos
        )

        # 3. 创建发布者，传入 qos_profile 和 event_callbacks
        self.publisher_ = self.create_publisher(
            String, 
            'test_topic', 
            qos_profile, 
            event_callbacks=event_callbacks
        )
        
        self.timer = self.create_timer(1.0, self.timer_callback)
        self.count = 0
        self.get_logger().info('发布者已启动 (QoS: Best Effort)，等待订阅者...')

    def timer_callback(self):
        msg = String()
        msg.data = f'Hello ROS 2: {self.count}'
        self.publisher_.publish(msg)
        self.count += 1

    # 4. QoS 不兼容时的触发函数
    def on_incompatible_qos(self, event):
        self.get_logger().error(
            f'🚨 警告：发现不兼容的订阅者！'
            f'冲突的 QoS 策略 ID: {event.last_policy_kind}'
        )

def main(args=None):
    rclpy.init(args=args)
    node = QosTestPublisher()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()

```

#### 2. Python 订阅者 (qos_test_sub.py)

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

from rclpy.qos import QoSProfile, QoSReliabilityPolicy, QoSHistoryPolicy
from rclpy.qos_event import SubscriptionEventCallbacks

class QosTestSubscriber(Node):
    def __init__(self):
        super().__init__('qos_test_sub')
        
        # 1. 定义 QoS 策略：必须可靠 (Reliable)
        qos_profile = QoSProfile(
            history=QoSHistoryPolicy.KEEP_LAST,
            depth=10,
            reliability=QoSReliabilityPolicy.RELIABLE # <--- 故意设置为高要求
        )

        # 2. 绑定 QoS 异常事件回调
        event_callbacks = SubscriptionEventCallbacks(
            incompatible_qos=self.on_incompatible_qos
        )

        # 3. 创建订阅者
        self.subscription = self.create_subscription(
            String,
            'test_topic',
            self.listener_callback,
            qos_profile,
            event_callbacks=event_callbacks
        )
        self.get_logger().info('订阅者已启动 (QoS: Reliable)，正在监听...')

    def listener_callback(self, msg):
        self.get_logger().info(f'收到消息: "{msg.data}"')

    # 4. QoS 不兼容时的触发函数
    def on_incompatible_qos(self, event):
        self.get_logger().error(
            f'🚨 错误：发布者的 QoS 无法满足我的要求！'
            f'冲突的 QoS 策略 ID: {event.last_policy_kind}'
        )

def main(args=None):
    rclpy.init(args=args)
    node = QosTestSubscriber()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()

```

---

### 二、 C++ 版本 (rclcpp) 测试代码

在你的社区巡逻机器人项目中，如果使用 C++ 编写底盘驱动或视觉推理（YOLO 等），使用 C++ 进行 QoS 控制能获得更好的性能。

#### 1. C++ 发布者 (qos_test_pub.cpp)

```cpp
#include <chrono>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class QosTestPublisher : public rclcpp::Node {
public:
    QosTestPublisher() : Node("qos_test_pub"), count_(0) {
        
        // 1. 设置 QoS 为 Best Effort
        rclcpp::QoS qos_profile(10);
        qos_profile.best_effort(); // <--- 低要求

        // 2. 配置发布者选项，绑定不兼容回调
        rclcpp::PublisherOptions pub_options;
        pub_options.event_callbacks.offered_incompatible_qos_callback =
            [this](rclcpp::QOSOfferedIncompatibleQoSInfo & info) {
                RCLCPP_ERROR(this->get_logger(), 
                    "🚨 警告：发现不兼容的订阅者！冲突策略枚举值: %d", 
                    info.last_policy_kind);
            };

        // 3. 创建发布者
        publisher_ = this->create_publisher<std_msgs::msg::String>(
            "test_topic", qos_profile, pub_options);

        timer_ = this->create_wall_timer(
            1000ms, std::bind(&QosTestPublisher::timer_callback, this));
            
        RCLCPP_INFO(this->get_logger(), "发布者启动 (Best Effort)...");
    }

private:
    void timer_callback() {
        auto msg = std_msgs::msg::String();
        msg.data = "Hello ROS 2: " + std::to_string(count_++);
        publisher_->publish(msg);
    }

    rclcpp::TimerBase::SharedPtr timer_;
    rclcpp::Publisher<std_msgs::msg::String>::SharedPtr publisher_;
    size_t count_;
};

int main(int argc, char * argv[]) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<QosTestPublisher>());
    rclcpp::shutdown();
    return 0;
}

```

#### 2. C++ 订阅者 (qos_test_sub.cpp)

```cpp
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

class QosTestSubscriber : public rclcpp::Node {
public:
    QosTestSubscriber() : Node("qos_test_sub") {
        
        // 1. 设置 QoS 为 Reliable
        rclcpp::QoS qos_profile(10);
        qos_profile.reliable(); // <--- 高要求

        // 2. 配置订阅者选项，绑定不兼容回调
        rclcpp::SubscriptionOptions sub_options;
        sub_options.event_callbacks.requested_incompatible_qos_callback =
            [this](rclcpp::QOSRequestedIncompatibleQoSInfo & info) {
                RCLCPP_ERROR(this->get_logger(), 
                    "🚨 错误：发布者 QoS 无法满足订阅要求！冲突策略枚举值: %d", 
                    info.last_policy_kind);
            };

        // 3. 创建订阅者
        subscription_ = this->create_subscription<std_msgs::msg::String>(
            "test_topic", qos_profile,
            [this](const std_msgs::msg::String::SharedPtr msg) {
                RCLCPP_INFO(this->get_logger(), "收到数据: '%s'", msg->data.c_str());
            }, 
            sub_options);
            
        RCLCPP_INFO(this->get_logger(), "订阅者启动 (Reliable)...");
    }

private:
    rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription_;
};

int main(int argc, char * argv[]) {
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<QosTestSubscriber>());
    rclcpp::shutdown();
    return 0;
}

```

---

### 三、 如何运行与观察结果

将代码编译（C++用 `colcon build`）或直接运行（Python）后：

1. 打开**终端 1** 运行发布者。
2. 打开**终端 2** 运行订阅者。

**你将看到的现象：**
订阅者的终端**不会**打印“收到数据”，而是会疯狂弹出红色报错：

```
[ERROR]: 🚨 错误：发布者的 QoS 无法满足我的要求！冲突的 QoS 策略 ID: 1

```

发布者的终端也会同步报错，提示发现了不兼容的订阅者。
*(注：策略 ID 1 在底层的枚举中代表的就是 `RELIABILITY` 冲突)*。

### 💡 实战建议

在你的社区治理机器人项目中，建议在底盘 STM32 通信节点、YOLO 图像订阅节点中，**强制加入这种 QoS 异常监控代码**。一旦因为配置错误或网络原因导致 QoS 不匹配，程序会第一时间把错误打在终端上，而不是让你对着黑屏怀疑人生，这能节约你大量的 Debug 时间！
