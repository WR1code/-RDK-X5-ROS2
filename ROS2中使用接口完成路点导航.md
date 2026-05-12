在ROS 2中，完成路点导航（Waypoint Navigation）的核心在于理解和使用**通信接口（Interfaces）**。由于导航是一个耗时较长、需要实时状态反馈，且可能中途被取消或修改的任务，ROS 2 主要依赖 **动作（Action）** 接口来完成导航，辅以 **话题（Topic）** 和 **服务（Service）**。

在现代 ROS 2 生态中，通常使用 **Nav2 (Navigation 2)** 框架来实现。Nav2 提供了一个专门用于路点导航的接口：`NavigateThroughPoses`。

以下是使用接口完成路点导航的详细讲解：

---

### 1. 为什么使用 Action（动作）接口？

ROS 2 有三种主要接口：

* **话题 (Topic):** 适用于连续数据流（如雷达数据、实时速度控制 `/cmd_vel`）。
* **服务 (Service):** 适用于快速的请求/响应（如清除代价地图）。阻塞式调用。
* **动作 (Action):** **最适合导航。** 它专为长时间运行的任务设计，非阻塞，并且包含三个部分：
* **Goal（目标）:** 你想让机器人去哪里（一组路点）。
* **Feedback（反馈）:** 机器人执行过程中的实时状态（距离目标的剩余距离、当前位姿等）。
* **Result（结果）:** 任务最终是成功、失败还是被取消。



---

### 2. 核心接口：`NavigateThroughPoses.action`

在 Nav2 中，执行多路点导航的标准接口定义在 `nav2_msgs` 包中。它的内部结构大致如下：

```text
# Goal (目标：一组带有时间戳的位姿)
geometry_msgs/PoseStamped[] poses
---
# Result (结果：通常为空，或者包含一些最终的统计信息)
std_msgs/Empty result
---
# Feedback (反馈：实时状态)
geometry_msgs/PoseStamped current_pose  # 当前位姿
builtin_interfaces/Duration navigation_time # 已用时间
builtin_interfaces/Duration estimated_time_remaining # 预计剩余时间
int16 number_of_recoveries # 恢复行为触发的次数
float32 distance_remaining # 剩余距离

```

### 3. 实现路点导航的完整流程 (Python 示例)

要通过接口让机器人移动，你需要编写一个 **Action Client（动作客户端）** 节点，将路点发送给 Nav2 的 **Action Server（动作服务端）**。

以下是实现路点导航的核心步骤和代码结构：

#### 步骤一：导入必要的依赖和接口

你需要导入 `rclpy`、动作客户端模块，以及上面提到的消息类型。

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient
from nav2_msgs.action import NavigateThroughPoses
from geometry_msgs.msg import PoseStamped

```

#### 步骤二：创建 Action Client 节点

创建一个继承自 `Node` 的类，并初始化动作客户端，连接到 `/navigate_through_poses` 动作服务器。

```python
class WaypointNavigator(Node):
    def __init__(self):
        super().__init__('waypoint_navigator')
        
        # 创建动作客户端
        self.action_client = ActionClient(
            self, 
            NavigateThroughPoses, 
            'navigate_through_poses'
        )

```

#### 步骤三：构建路点 (Goal) 并发送请求

你需要将坐标（x, y, 姿态四元数）封装成 `PoseStamped` 对象的列表。

```python
    def send_waypoints(self, poses_list):
        # 等待动作服务器启动
        self.get_logger().info('等待导航服务器...')
        self.action_client.wait_for_server()

        # 创建 Goal 对象
        goal_msg = NavigateThroughPoses.Goal()
        
        # 构建路点列表
        waypoints = []
        for pose_data in poses_list:
            pose = PoseStamped()
            pose.header.frame_id = 'map' # 参考坐标系通常是 map
            pose.header.stamp = self.get_clock().now().to_msg()
            pose.pose.position.x = pose_data['x']
            pose.pose.position.y = pose_data['y']
            pose.pose.orientation.w = pose_data['w'] # 简化的姿态
            waypoints.append(pose)
            
        goal_msg.poses = waypoints

        self.get_logger().info('发送路点目标...')
        # 异步发送目标，并绑定反馈回调函数
        self.send_goal_future = self.action_client.send_goal_async(
            goal_msg,
            feedback_callback=self.feedback_callback
        )
        # 绑定目标接受状态的回调
        self.send_goal_future.add_done_callback(self.goal_response_callback)

```

#### 步骤四：处理反馈 (Feedback) 和结果 (Result)

接收 Nav2 实时发来的距离和时间反馈，并在到达终点时获取结果。

```python
    def feedback_callback(self, feedback_msg):
        # 提取反馈信息
        feedback = feedback_msg.feedback
        self.get_logger().info(f'剩余距离: {feedback.distance_remaining:.2f} 米')

    def goal_response_callback(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().info('目标被服务器拒绝！')
            return

        self.get_logger().info('目标已被接受，正在执行...')
        # 异步等待最终结果
        self.get_result_future = goal_handle.get_result_async()
        self.get_result_future.add_done_callback(self.get_result_callback)

    def get_result_callback(self, future):
        result = future.result().result
        status = future.result().status
        # status 4 表示 SUCCEEDED (成功)
        if status == 4:
            self.get_logger().info('成功到达所有路点！')
        else:
            self.get_logger().info(f'导航失败或被取消，状态码: {status}')
        
        rclpy.shutdown()

```

#### 步骤五：运行节点

最后，实例化节点并传入你的目标坐标。

```python
def main(args=None):
    rclpy.init(args=args)
    navigator = WaypointNavigator()
    
    # 定义你想去的路点 (x, y, w)
    my_waypoints = [
        {'x': 1.0, 'y': 0.0, 'w': 1.0},
        {'x': 2.0, 'y': 2.0, 'w': 1.0},
        {'x': 0.0, 'y': 2.0, 'w': 1.0}
    ]
    
    navigator.send_waypoints(my_waypoints)
    rclpy.spin(navigator)

if __name__ == '__main__':
    main()

```

---

### 4. 辅助接口的作用

除了 Action，在底层导航过程中，ROS 2 还在使用其他接口默默配合：

* **话题发布 (`/cmd_vel`)**: Nav2 的控制器（如 DWB 或 MPPI）计算出路径后，会通过 `/cmd_vel` 话题持续发布速度指令（线速度和角速度）给底盘驱动。
* **话题订阅 (`/odom`, `/scan`)**: 导航系统通过订阅里程计和激光雷达话题，实时更新自身在地图中的位置并规避动态障碍物。
* **服务调用 (`/global_costmap/clear_entirely_global_costmap`)**: 当机器人卡住（Recovery 阶段）时，导航行为树可能会调用服务接口来清除代价地图，尝试重新规划路线。

### 总结

在 ROS 2 中完成路点导航，本质上就是扮演一个 **Action Client** 的角色。你将一组坐标打包进 `NavigateThroughPoses.Goal` 中，通过网络发送给 Nav2 服务器。随后，利用回调函数监听 `Feedback` 以掌握进度，并在 `Result` 返回时确认任务完成。这种基于 Action 的异步通信机制，保证了机器人主程序的流畅运行和导航任务的高度可控性。
