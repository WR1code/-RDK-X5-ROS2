在 ROS 2 中，**动态 TF 变换（Dynamic Transforms）**用于表示系统中相对位置随时间不断变化的坐标系。

常见的应用场景包括：机器人移动时底盘（`base_link`）在世界坐标系（`odom` 或 `map`）中的位置、机械臂各个关节（`link1` 到 `link2`）在运动时的实时姿态等。

由于动态 TF 变换是实时变化的，它与静态 TF 有两个核心区别：
1. **发布频率**：动态 TF 需要在一个循环或定时器中高频持续发布（通常是 30Hz - 100Hz），而静态 TF 只需发布一次。
2. **底层机制**：动态 TF 发布在 `/tf` 话题上，缓冲区通常只保留最近几秒钟的历史数据；而静态 TF 发布在 `/tf_static` 话题上，会被接收端永久锁存。

在 ROS 2 中发布动态 TF，通常使用 **Python** 或 **C++** 编写节点。以下是详细的实现方法。

---

### 方法 1：使用 Python 编写动态 TF 广播器

我们需要使用 `tf2_ros.TransformBroadcaster`，并结合 ROS 2 的定时器（Timer）来持续更新和发布坐标变换。

下面的示例将模拟一个名为 `turtle_link` 的坐标系，它绕着中心坐标系 `world` 做圆周运动。

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
from tf2_ros import TransformBroadcaster
import math
import time

class DynamicTfPublisher(Node):
    def __init__(self):
        super().__init__('dynamic_tf_publisher')
        
        # 1. 创建动态 TF 广播器
        self.tf_broadcaster = TransformBroadcaster(self)
        
        # 2. 创建定时器，每 0.05 秒（20Hz）触发一次回调
        self.timer = self.create_timer(0.05, self.broadcast_timer_callback)
        self.start_time = time.time()

    def broadcast_timer_callback(self):
        # 计算当前运行的时间（秒）
        current_time = time.time() - self.start_time
        
        # 构造 TransformStamped 消息
        t = TransformStamped()
        
        # 填写时间戳（必须是最新时间）和坐标系
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'world'
        t.child_frame_id = 'turtle_link'
        
        # 模拟圆周运动的平移 (半径为 2.0)
        t.transform.translation.x = 2.0 * math.cos(current_time)
        t.transform.translation.y = 2.0 * math.sin(current_time)
        t.transform.translation.z = 0.0
        
        # 模拟随时间变化的旋转 (绕 Z 轴旋转)
        yaw = current_time  
        
        # 将欧拉角 (Yaw) 转换为四元数
        t.transform.rotation.x = 0.0
        t.transform.rotation.y = 0.0
        t.transform.rotation.z = math.sin(yaw / 2.0)
        t.transform.rotation.w = math.cos(yaw / 2.0)
        
        # 3. 发送变换
        self.tf_broadcaster.sendTransform(t)

def main(args=None):
    rclpy.init(args=args)
    node = DynamicTfPublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 方法 2：使用 C++ 编写动态 TF 广播器

对于自动驾驶或需要处理高频里程计数据的场景，C++ 是更好的选择。下面是实现同样圆周运动逻辑的 C++ 代码。

```cpp
#include <chrono>
#include <memory>
#include <cmath>

#include "rclcpp/rclcpp.hpp"
#include "geometry_msgs/msg/transform_stamped.hpp"
#include "tf2_ros/transform_broadcaster.h"
#include "tf2/LinearMath/Quaternion.h"

using namespace std::chrono_literals;

class DynamicFramePublisher : public rclcpp::Node
{
public:
  DynamicFramePublisher() : Node("dynamic_tf_publisher_cpp")
  {
    // 1. 初始化广播器
    tf_broadcaster_ = std::make_unique<tf2_ros::TransformBroadcaster>(*this);

    // 2. 创建定时器，以 30Hz 的频率发布 (约 33ms)
    timer_ = this->create_wall_timer(
      33ms, std::bind(&DynamicFramePublisher::publish_transform, this));
      
    start_time_ = this->now();
  }

private:
  void publish_transform()
  {
    // 计算经过的时间
    double dt = (this->now() - start_time_).seconds();

    geometry_msgs::msg::TransformStamped t;

    // 填写时间戳和坐标系
    t.header.stamp = this->get_clock()->now();
    t.header.frame_id = "world";
    t.child_frame_id = "robot_link";

    // 填写动态平移 (半径为 1.5 的圆周运动)
    t.transform.translation.x = 1.5 * cos(dt);
    t.transform.translation.y = 1.5 * sin(dt);
    t.transform.translation.z = 0.0;

    // 填写动态旋转 (使用 tf2 的四元数类)
    tf2::Quaternion q;
    q.setRPY(0, 0, dt); // Roll = 0, Pitch = 0, Yaw = dt
    t.transform.rotation.x = q.x();
    t.transform.rotation.y = q.y();
    t.transform.rotation.z = q.z();
    t.transform.rotation.w = q.w();

    // 3. 发送变换
    tf_broadcaster_->sendTransform(t);
  }

  rclcpp::TimerBase::SharedPtr timer_;
  std::unique_ptr<tf2_ros::TransformBroadcaster> tf_broadcaster_;
  rclcpp::Time start_time_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<DynamicFramePublisher>());
  rclcpp::shutdown();
  return 0;
}
```

---

### 开发动态 TF 时的核心注意事项

1. **时间戳（Stamp）至关重要**：动态 TF 的 `t.header.stamp` **必须**使用当前系统的最新时间（`this->get_clock()->now()`）。如果时间戳是旧的或者未来的，监听者在查询 TF 时会报错（`Extrapolation Exception`）。
2. **避免父子坐标系冲突**：在 TF 树中，一个坐标系（Frame）只能有一个父坐标系。如果两个节点同时试图将 `robot_link` 作为子坐标系发布不同的父节点连接，会导致 TF 树断裂或抖动。
3. **数据来源**：在实际项目中，动态 TF 中的计算通常不是使用定时器凭空生成的，而是订阅传感器数据（如里程计 `/odom`，或关节状态 `/joint_states`）的回调函数中进行计算并触发 `sendTransform`。

---

### 如何验证动态 TF

编译并运行你的代码后，可以使用以下工具验证 TF 树是否在动态更新：

**1. 实时监听 TF 数据变化**
```bash
ros2 run tf2_ros tf2_echo world turtle_link
```
终端会疯狂滚动输出两者之间不断变化的位移（Translation）和旋转（Rotation）数据。

**2. 使用 RViz2 进行 3D 可视化（最直观）**
1. 终端输入 `rviz2` 启动可视化界面。
2. 将全局坐标系 **Fixed Frame** 设置为你的父坐标系（例如 `world`）。
3. 点击左下角的 **Add** -> 选择 **TF**。
4. 展开 TF 菜单，勾选 `Show Names` 和 `Show Axes`。
5. 你将在 3D 界面中看到子坐标系（如 `turtle_link` 或 `robot_link`）绕着原点做动态移动。
