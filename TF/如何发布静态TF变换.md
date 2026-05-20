在 ROS 2 中，静态 TF 变换（Static Transforms）用于表示系统中相对位置永远不会改变的坐标系。例如，机器人底盘（`base_link`）和固定在其上的激光雷达（`lidar_link`）或摄像头之间的相对位置。

因为静态 TF 变换不会随时间改变，ROS 2 的 `tf2` 系统对其进行了优化：它只需要发布一次（或偶尔发布），所有监听者都会将其缓存下来，这比高频发布的动态 TF 节省了大量系统带宽。

在 ROS 2 中，发布静态 TF 变换主要有四种方法：**命令行工具**、**Launch 文件**、**Python 节点** 和 **C++ 节点**。

---

### 方法 1：使用命令行工具（最简单，适合测试）

ROS 2 自带了 `tf2_ros` 包，其中包含一个可执行文件 `static_transform_publisher`，可以直接在终端中运行。

**语法 1（使用欧拉角 - X Y Z Yaw Pitch Roll）：**
```bash
ros2 run tf2_ros static_transform_publisher x y z yaw pitch roll frame_id child_frame_id
```

**语法 2（使用明确的参数标志 - 推荐，更清晰）：**
```bash
ros2 run tf2_ros static_transform_publisher --x x --y y --z z --yaw yaw --pitch pitch --roll roll --frame-id frame_id --child-frame-id child_frame_id
```

**示例：**
假设你要发布一个激光雷达 (`laser_link`) 相对于机器人中心 (`base_link`) 在 X轴前方 0.5米，Z轴上方 0.2米的位置，且没有旋转：
```bash
ros2 run tf2_ros static_transform_publisher --x 0.5 --y 0.0 --z 0.2 --yaw 0 --pitch 0 --roll 0 --frame-id base_link --child-frame-id laser_link
```

---

### 方法 2：在 Launch 文件中启动（推荐用于实际项目）

在实际项目中，我们通常会在启动机器人时，通过 Launch 文件将静态 TF 节点与其他节点一起启动。

创建一个 Python 格式的 launch 文件（例如 `static_tf_launch.py`）：

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='tf2_ros',
            executable='static_transform_publisher',
            name='static_transform_publisher_laser',
            arguments=[
                '--x', '0.5', 
                '--y', '0.0', 
                '--z', '0.2', 
                '--yaw', '0.0', 
                '--pitch', '0.0', 
                '--roll', '0.0', 
                '--frame-id', 'base_link', 
                '--child-frame-id', 'laser_link'
            ]
        )
    ])
```
运行此 launch 文件即可自动发布静态变换。

---

### 方法 3：使用 Python 编写节点发布

如果你需要在代码逻辑中计算并发布静态坐标，可以使用 Python 的 `tf2_ros.StaticTransformBroadcaster`。

```python
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
from tf2_ros.static_transform_broadcaster import StaticTransformBroadcaster

class StaticTfPublisher(Node):
    def __init__(self):
        super().__init__('static_tf_publisher_node')
        
        # 1. 创建静态 TF 广播器
        self.tf_broadcaster = StaticTransformBroadcaster(self)
        
        # 2. 构造 TransformStamped 消息
        t = TransformStamped()
        
        # 填写时间戳和坐标系名称
        t.header.stamp = self.get_clock().now().to_msg()
        t.header.frame_id = 'base_link'
        t.child_frame_id = 'camera_link'
        
        # 填写平移关系 (X, Y, Z)
        t.transform.translation.x = 0.3
        t.transform.translation.y = 0.0
        t.transform.translation.z = 0.5
        
        # 填写旋转关系 (四元数: qx, qy, qz, qw)
        # 这里以无旋转 (单位四元数) 为例
        t.transform.rotation.x = 0.0
        t.transform.rotation.y = 0.0
        t.transform.rotation.z = 0.0
        t.transform.rotation.w = 1.0
        
        # 3. 发布静态 TF 变换
        self.tf_broadcaster.sendTransform(t)
        self.get_logger().info('成功发布静态 TF 变换: base_link -> camera_link')

def main(args=None):
    rclpy.init(args=args)
    node = StaticTfPublisher()
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
*注意：静态 TF 只需要调用一次 `sendTransform()`，底层会自动将其锁存（latch）在 `/tf_static` 话题上，不需要使用定时器循环发布。*

---

### 方法 4：使用 C++ 编写节点发布

对于性能要求更高的场景，可以使用 C++ 的 `tf2_ros::StaticTransformBroadcaster`。

```cpp
#include <memory>
#include "rclcpp/rclcpp.hpp"
#include "geometry_msgs/msg/transform_stamped.hpp"
#include "tf2_ros/static_transform_broadcaster.h"
#include "tf2/LinearMath/Quaternion.h"

class StaticFramePublisher : public rclcpp::Node
{
public:
  explicit StaticFramePublisher() : Node("static_tf_publisher_cpp")
  {
    // 1. 初始化广播器
    tf_broadcaster_ = std::make_shared<tf2_ros::StaticTransformBroadcaster>(this);

    // 2. 构造消息
    geometry_msgs::msg::TransformStamped t;
    t.header.stamp = this->get_clock()->now();
    t.header.frame_id = "base_link";
    t.child_frame_id = "imu_link";

    // 填写平移
    t.transform.translation.x = 0.0;
    t.transform.translation.y = 0.0;
    t.transform.translation.z = 0.1;

    // 填写旋转 (将欧拉角转换为四元数)
    tf2::Quaternion q;
    q.setRPY(0.0, 0.0, 0.0); // Roll, Pitch, Yaw
    t.transform.rotation.x = q.x();
    t.transform.rotation.y = q.y();
    t.transform.rotation.z = q.z();
    t.transform.rotation.w = q.w();

    // 3. 发送变换
    tf_broadcaster_->sendTransform(t);
    RCLCPP_INFO(this->get_logger(), "成功发布静态 TF 变换: base_link -> imu_link");
  }

private:
  std::shared_ptr<tf2_ros::StaticTransformBroadcaster> tf_broadcaster_;
};

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<StaticFramePublisher>());
  rclcpp::shutdown();
  return 0;
}
```

---

### 如何验证你的 TF 变换是否发布成功？

无论你使用哪种方法发布了静态 TF，都可以使用以下工具进行验证：

1. **终端打印 TF 数据：**
   ```bash
   ros2 run tf2_ros tf2_echo base_link laser_link
   ```
   如果发布成功，终端会持续打印两者之间的位移和旋转关系。

2. **生成 TF 树状图 (PDF格式)：**
   ```bash
   ros2 run tf2_tools view_frames
   ```
   这会在当前目录下生成一个 `frames.pdf` 文件，打开它可以直观地看到所有坐标系的连接关系。

3. **使用 RViz2 可视化：**
   * 运行 `rviz2`
   * 将 `Fixed Frame` 设为 `base_link`
   * 点击左下角 `Add` -> 选择 `TF`
   * 你将在 3D 视图中看到代表 `base_link` 和子坐标系的坐标轴。
```
