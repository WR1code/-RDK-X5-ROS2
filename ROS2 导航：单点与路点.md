在机器人导航的语境中（比如您刚刚问到的 ROS 2 / Nav2），**单点（Single Goal/Pose）** 和 **路点（Waypoint）** 是两种最基础、也是最核心的任务目标形式。

简单来说，它们的区别就像是“打车直达”**和**“公交车停靠站”的区别。

以下是详细说明：

### 一、 什么是单点 (Single Point / Goal Pose)？

**单点**，顾名思义，就是**一个单一的最终目的地**。
在导航系统中，它通常不仅包含一个二维平面的坐标 $(x, y)$，还包含机器人在该点应当具备的**朝向（Orientation / Yaw）**。

* **行为模式：** 机器人从当前的起点出发，由导航算法（全局规划器）计算出一条最优或最短的路径，避开途中的障碍物，最终到达这唯一的目的地，并调整好指定的姿态停下。
* **生活中的类比：** 就像你用打车软件叫车，你只输入了一个终点（比如“机场航站楼的某个特定门口”）。司机（导航系统）会自己决定走哪条路，你只关心最终准确到达那里。
* **适用场景：**
* “去充电桩充电”（目标是一个确切的对接点）。
* “去拿取货物”（到达指定货架位置）。
* 一键召唤机器人到用户身边。



### 二、 什么是路点 (Waypoint / Via-point)？

**路点**，也叫航点或途经点，是**一系列按顺序排列的坐标点集合**。机器人需要依次经过或到达这些点。

在执行路点导航时，机器人实际上是在执行一系列连续的导航任务（A $\rightarrow$ B $\rightarrow$ C $\rightarrow$ D）。

* **行为模式：** 导航系统会将整个大任务拆解。首先规划从当前位置到第一个路点（Point 1）的路径；到达后，再规划从 Point 1 到 Point 2 的路径，依此类推，直到走完最后一个点。
* **生活中的类比：**
* **情况 A（停靠）：** 就像公交车的路线，必须在设定的每一个站点（路点）停靠一下，上下客，然后再前往下一站。
* **情况 B（不停靠，仅穿过）：** 就像马拉松比赛路线上的打卡点，运动员只需要经过那个区域即可，不需要停下脚步。


* **适用场景：**
* **安防巡逻：** 机器人需要在办公楼的各个走廊尽头、机房门口依次“打卡”检查。
* **农业喷洒：** 无人机或农业机器人按照设定的“S型”轨迹在农田的各个路点之间穿梭作业。
* **餐厅送餐：** 机器人一次性带着三桌客人的菜，先去1号桌（路点1），再去3号桌（路点2），最后回厨房（终点）。



---

### 三、 核心区别对比

| 对比维度 | 单点导航 (Single Goal) | 路点导航 (Waypoints) |
| --- | --- | --- |
| **目标数量** | 仅 1 个 | 多个（2个或以上） |
| **轨迹复杂度** | 简单，通常由算法自动算出一条最短路径 | 复杂，由用户/程序强制规定了机器人必须经过的关键区域 |
| **Nav2 动作接口** | `NavigateToPose` | `FollowWaypoints` 或 `NavigateThroughPoses` |
| **执行逻辑** | 规划一次 $\rightarrow$ 移动 $\rightarrow$ 结束 | 规划 $\rightarrow$ 移动 $\rightarrow$ 检查序列 $\rightarrow$ 规划 $\rightarrow$ 移动... |

**总结：**
单点告诉机器人“最终要去哪里”**（Result-oriented）；而路点不仅告诉机器人要去哪里，还在很大程度上限制了机器人**“大概要怎么走”（Process-oriented）。






在ROS 2中进行导航，主要依赖于 **Nav2 (Navigation 2)** 框架。 Nav2 是一个高度模块化和可配置的导航栈，支持从简单的单点移动到复杂的多航点（Waypoint）跟随。

在Nav2中，导航的核心机制是 **Action (动作)**。单点导航通常调用 `/navigate_to_pose` 动作，而路点导航则调用 `/follow_waypoints` 或 `/navigate_through_poses` 动作。

以下是三种最常用的实现方式：**RViz2图形界面**、**Python API (nav2_simple_commander)** 以及 **命令行 (CLI)**。

---

### 方法一：使用 RViz2 界面（最直观）

在启动了你的机器人和 Nav2 栈（例如 `ros2 launch nav2_bringup tb3_simulation_launch.py`）之后，RViz2 会打开。

**1. 单点导航 (Single-point Navigation)**

* 在 RViz2 顶部工具栏中，找到 **"2D Goal Pose"** 按钮。
* 点击该按钮后，在地图上你想要机器人去的地方点击并拖动（拖动方向代表机器人的最终朝向）。
* 松开鼠标后，Nav2 会自动规划路径并控制机器人移动。

**2. 路点导航 (Waypoint Navigation)**

* 在 RViz2 左侧的 "Displays" 面板中，找到 **"Nav2 Waypoint"** 插件（如果没有，可以点击 Add -> Nav2/Waypoint 加上）。
* 点击顶部工具栏的 **"Nav2 Goal"** 按钮，在地图上依次点击你想要设置的多个路点。
* 每点击一个点，面板中就会增加一个路点。设置完毕后，点击面板底部的 **"Start Navigation"**，机器人会依次到达这些点。

---

### 方法二：使用 Python API（开发者最常用）

在 ROS 2 中，编写原生 Action Client 比较繁琐。因此，Nav2 官方提供了一个非常强大的 Python 库：`nav2_simple_commander`。它封装了底层的 Action 调用，使得代码非常简洁。

首先确保安装了该包（以 Humble 为例）：

```bash
sudo apt install ros-humble-nav2-simple-commander

```

#### 1. Python 实现：单点导航

以下是一个让机器人导航到指定坐标 `(x: 2.0, y: 1.0)` 的完整 Python 示例：

```python
import rclpy
from geometry_msgs.msg import PoseStamped
from nav2_simple_commander.robot_navigator import BasicNavigator, TaskResult

def main():
    rclpy.init()
    
    # 实例化导航器
    navigator = BasicNavigator()

    # 设置机器人的初始位置 (如果之前在RViz里设置过了可以注释掉这一步)
    initial_pose = PoseStamped()
    initial_pose.header.frame_id = 'map'
    initial_pose.header.stamp = navigator.get_clock().now().to_msg()
    initial_pose.pose.position.x = 0.0
    initial_pose.pose.position.y = 0.0
    initial_pose.pose.orientation.w = 1.0
    navigator.setInitialPose(initial_pose)

    # 等待 Nav2 启动完成
    navigator.waitUntilNav2Active()

    # 1. 设置目标点
    goal_pose = PoseStamped()
    goal_pose.header.frame_id = 'map'
    goal_pose.header.stamp = navigator.get_clock().now().to_msg()
    goal_pose.pose.position.x = 2.0
    goal_pose.pose.position.y = 1.0
    goal_pose.pose.orientation.w = 1.0 # 面向前方

    # 2. 发送单点导航请求
    navigator.goToPose(goal_pose)

    # 3. 循环检查任务状态
    while not navigator.isTaskComplete():
        # 这里可以获取反馈信息，例如距离目标的剩余距离
        feedback = navigator.getFeedback()
        if feedback:
            print(f'距离目标还有: {feedback.distance_remaining:.2f} 米')

    # 4. 获取最终结果
    result = navigator.getResult()
    if result == TaskResult.SUCCEEDED:
        print('成功到达目标点！')
    elif result == TaskResult.CANCELED:
        print('导航被取消！')
    elif result == TaskResult.FAILED:
        print('导航失败！')

    # 关闭 ROS 2 节点
    rclpy.shutdown()

if __name__ == '__main__':
    main()

```

#### 2. Python 实现：多路点导航 (Waypoints)

利用 `nav2_simple_commander`，路点导航只需构建一个包含多个 `PoseStamped` 的列表，并调用 `followWaypoints()` 即可。

```python
import rclpy
from geometry_msgs.msg import PoseStamped
from nav2_simple_commander.robot_navigator import BasicNavigator, TaskResult

def main():
    rclpy.init()
    navigator = BasicNavigator()
    navigator.waitUntilNav2Active()

    # 1. 创建多个路点
    waypoints = []
    
    # 路点 1
    pt1 = PoseStamped()
    pt1.header.frame_id = 'map'
    pt1.pose.position.x = 2.0
    pt1.pose.position.y = 0.0
    pt1.pose.orientation.w = 1.0
    waypoints.append(pt1)
    
    # 路点 2
    pt2 = PoseStamped()
    pt2.header.frame_id = 'map'
    pt2.pose.position.x = 2.0
    pt2.pose.position.y = 2.0
    pt2.pose.orientation.w = 1.0
    waypoints.append(pt2)

    # 路点 3
    pt3 = PoseStamped()
    pt3.header.frame_id = 'map'
    pt3.pose.position.x = 0.0
    pt3.pose.position.y = 2.0
    pt3.pose.orientation.w = 1.0
    waypoints.append(pt3)

    # 2. 发送多路点导航请求
    print(f"开始按照 {len(waypoints)} 个路点进行导航...")
    navigator.followWaypoints(waypoints)

    # 3. 循环检查任务状态
    while not navigator.isTaskComplete():
        feedback = navigator.getFeedback()
        if feedback:
            print(f'正在前往第 {feedback.current_waypoint} 个路点...')

    # 4. 获取结果
    result = navigator.getResult()
    if result == TaskResult.SUCCEEDED:
        print('成功完成所有路点巡航！')
    else:
        print('路点导航未成功完成。')

    rclpy.shutdown()

if __name__ == '__main__':
    main()

```

*注：在实际应用中，你需要经常更新 `header.stamp = navigator.get_clock().now().to_msg()` 以确保时间戳是最新的。*

---

### 方法三：使用命令行 (CLI) 工具

如果你只是想快速测试节点是否正常工作，可以直接通过终端发送 Action Goal。

**1. 命令行测试单点导航**
发送目标到 `/navigate_to_pose` action 服务器：

```bash
ros2 action send_goal /navigate_to_pose nav2_msgs/action/NavigateToPose "{pose: {header: {frame_id: 'map'}, pose: {position: {x: 2.0, y: 1.0, z: 0.0}, orientation: {x: 0.0, y: 0.0, z: 0.0, w: 1.0}}}}"

```

**2. 命令行测试多路点导航**
发送目标到 `/follow_waypoints` action 服务器（输入多个位姿）：

```bash
ros2 action send_goal /follow_waypoints nav2_msgs/action/FollowWaypoints "{poses: [
  {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 0.0, z: 0.0}, orientation: {w: 1.0}}},
  {header: {frame_id: 'map'}, pose: {position: {x: 1.0, y: 1.0, z: 0.0}, orientation: {w: 1.0}}}
]}"

```

### 核心概念总结

* **NavigateToPose:** 从A点规划一条全局路径到B点，并避障前往。
* **FollowWaypoints:** 给定 A, B, C 三个点，机器人会依次调用 A, B, C 的 `NavigateToPose`，在每个点处会停顿。
* **NavigateThroughPoses:** (这是另一个进阶动作) 给定 A, B, C，机器人会规划一条直接穿过 A、B，最终到达 C 的平滑轨迹，途经 A 和 B 时通常**不会**停下来。根据你的应用场景（如巡逻打卡 vs. 规定路径行驶）选择合适的动作。
