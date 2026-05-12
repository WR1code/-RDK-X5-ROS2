<img width="576" height="640" alt="7709844c-b956-4987-9d78-c5a48377b053" src="https://github.com/user-attachments/assets/5c0ab28f-8e8f-4f97-b14b-f879e0821b6b" />



在 ROS 2 中，单点导航主要依赖于 **Nav2 (Navigation 2)** 框架。与简单的传感器数据发布（使用 Topic）或瞬间完成的计算（使用 Service）不同，导航是一个**长时间运行的任务**。因此，ROS 2 中调用导航接口的核心机制是 **Action（动作）**。

具体来说，单点导航使用的核心 Action 接口是 `nav2_msgs/action/NavigateToPose`。

以下是调用该接口进行单点导航的详细讲解：

---

### 1. 核心概念：Action 通信机制

Action 是一种基于客户端-服务器（Client-Server）模型的通信机制，专门用于长时间任务。它由三个主要部分组成：

* **Goal（目标）**：客户端发送给服务器的指令（例如：“去坐标 [X: 2.0, Y: 1.0]”）。
* **Feedback（反馈）**：服务器在执行任务期间持续发回的状态（例如：“当前距离目标还有 1.5 米”）。
* **Result（结果）**：任务结束时返回的最终状态（例如：“成功到达”或“被障碍物卡住，导航失败”）。

在单点导航中：

* **Action Server**：由 Nav2 的 `bt_navigator`（行为树导航器）节点提供，它负责接收目标并控制机器人的移动。
* **Action Client**：由你自己编写的节点充当，负责发送目标坐标点。

---

### 2. NavigateToPose 接口定义

了解如何调用之前，我们需要知道 `NavigateToPose` 这个 Action 的数据结构。它的定义大致如下：

**Goal:**

* `pose` (`geometry_msgs/PoseStamped`): 包含目标位置（x, y, z）和姿态/朝向（四元数 x, y, z, w），以及坐标系（通常是 `map`）。
* `behavior_tree` (string): 可选，指定使用的行为树文件路径。

**Result:**

* `result` (`std_msgs/Empty`): 为空，主要通过 Action 的状态码（SUCCEEDED, ABORTED 等）来判断是否成功。

**Feedback:**

* `current_pose` (`geometry_msgs/PoseStamped`): 机器人的当前位置。
* `navigation_time` (`builtin_interfaces/Duration`): 已用导航时间。
* `distance_remaining` (float): 距离目标的剩余直线距离。

---

### 3. 如何通过命令行调用 (快速测试)

在编写代码之前，你可以使用 ROS 2 的 CLI 工具直接发送 Action Goal，这非常适合测试 Nav2 是否正常运行：

```bash
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 2.0, y: 0.0, z: 0.0}, orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}}}}"

```

* `/navigate_to_pose` 是 Action 的名称。
* `nav2_msgs/action/NavigateToPose` 是 Action 的类型。
* 后面的 JSON 字符串是我们传递的 Goal 参数（前往 map 坐标系的 [2.0, 0.0] 点，朝向为正前方 `w:1.0`）。

---

### 4. 如何通过代码调用 (Python API)

在实际开发中，我们通常使用 `rclpy` (Python) 或 `rclcpp` (C++) 来编写 Action Client。以下是一个完整的 Python 示例：

#### 代码示例：单点导航客户端

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient
from nav2_msgs.action import NavigateToPose
from geometry_msgs.msg import PoseStamped

class SingleWaypointNavigator(Node):
    def __init__(self):
        super().__init__('single_waypoint_navigator')
        # 1. 创建 Action Client，连接到 '/navigate_to_pose'
        self._action_client = ActionClient(self, NavigateToPose, 'navigate_to_pose')

    def send_goal(self, x, y, w):
        self.get_logger().info('等待 Action Server 启动...')
        # 2. 等待 Nav2 导航服务器准备就绪
        self._action_client.wait_for_server()

        # 3. 构建 Goal 消息
        goal_msg = NavigateToPose.Goal()
        goal_msg.pose.header.frame_id = 'map'  # 目标点所在的坐标系
        goal_msg.pose.header.stamp = self.get_clock().now().to_msg()
        
        # 设置位置 (Position)
        goal_msg.pose.pose.position.x = float(x)
        goal_msg.pose.pose.position.y = float(y)
        goal_msg.pose.pose.position.z = 0.0
        
        # 设置朝向 (Orientation - 四元数)
        goal_msg.pose.pose.orientation.x = 0.0
        goal_msg.pose.pose.orientation.y = 0.0
        goal_msg.pose.pose.orientation.z = 0.0
        goal_msg.pose.pose.orientation.w = float(w)

        self.get_logger().info(f'发送目标点: x={x}, y={y}')

        # 4. 异步发送 Goal，并绑定反馈和结果的回调函数
        self._send_goal_future = self._action_client.send_goal_async(
            goal_msg, 
            feedback_callback=self.feedback_callback
        )
        self._send_goal_future.add_done_callback(self.goal_response_callback)

    # 5. 处理服务器对 Goal 的接受/拒绝响应
    def goal_response_callback(self, future):
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().error('目标被服务器拒绝！')
            return
        
        self.get_logger().info('目标已被接受，开始导航...')
        # 如果接受，继续等待最终结果
        self._get_result_future = goal_handle.get_result_async()
        self._get_result_future.add_done_callback(self.get_result_callback)

    # 6. 处理导航过程中的实时反馈
    def feedback_callback(self, feedback_msg):
        feedback = feedback_msg.feedback
        self.get_logger().info(f'导航中... 距离目标还剩: {feedback.distance_remaining:.2f} 米')

    # 7. 处理最终的导航结果
    def get_result_callback(self, future):
        result_status = future.result().status
        
        # rclpy.action.GoalStatus.STATUS_SUCCEEDED 值为 4
        if result_status == 4: 
            self.get_logger().info('✅ 成功到达目标点！')
        else:
            self.get_logger().info(f'❌ 导航失败，状态码: {result_status}')
            
        rclpy.shutdown() # 任务完成后关闭节点

def main(args=None):
    rclpy.init(args=args)
    navigator = SingleWaypointNavigator()
    
    # 发送一个目标点: X=2.0, Y=1.0, 朝向没有旋转(w=1.0)
    navigator.send_goal(x=2.0, y=1.0, w=1.0)
    
    # 保持节点运行以接收回调
    rclpy.spin(navigator)

if __name__ == '__main__':
    main()

```

### 5. 调用流程总结

无论你是用 C++ 还是 Python，调用 `NavigateToPose` 接口的核心逻辑都是这五步：

1. **实例化 Action Client** 并绑定 `navigate_to_pose` 名称。
2. **`wait_for_server()`**：确保 Nav2 的系统已经完全启动，防止指令发送到虚空。
3. **构造 `geometry_msgs/PoseStamped` 消息**：设定 `frame_id`（极重要，通常是 `map` 或 `odom`）和目标的三维坐标与四元数。
4. **发送 Goal 并处理 Feedback**：在回调函数中读取 `distance_remaining`，可用于在 UI 上显示进度条。
5. **处理 Result**：监听最终状态，判断机器人是成功到达目的地，还是因为路径规划失败（如四周被障碍物完全封死）而放弃了任务。
