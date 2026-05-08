和传感器类似，**URDF 原生标准其实只定义了“关节（Joint）”的物理运动学属性**，并没有真正的“执行器（Actuator）”或“电机”的概念。

在 ROS2 中，想要在 URDF 中构建执行器，我们需要将 **URDF 的 `<joint>` 标签** 与 **`ros2_control` 框架（或 Gazebo 插件）** 结合起来。通过定义关节的类型（怎么动）以及控制接口（怎么控），我们就能模拟出现实世界中所有的主流电机和气动/液压装置。

在 ROS2 的 URDF 环境下，我们能构建的执行器可以从**运动形式**和**控制接口**两个维度来分类：

---

### 一、 按“运动形式”分类（基于 `<joint>` 类型）

这是执行器的物理表现形式，由 URDF 的 `<joint type="...">` 决定。能够被驱动（Actuated）的关节主要有以下三种：

#### 1. 伺服舵机 / 机械臂关节 (Revolute Joint)

* **物理表现:** 绕着某一个轴旋转，**有严格的角度限制**（例如只能在 -90° 到 +90° 之间转动）。
* **对应现实硬件:** 机械臂的关节电机、RC伺服舵机、云台的俯仰轴。
* **URDF 核心属性:** `type="revolute"`，并且必须包含 `<limit>` 标签来定义位置（角度）、速度和力矩的上限。

#### 2. 连续旋转电机 (Continuous Joint)

* **物理表现:** 绕着某一个轴旋转，**没有角度限制**，可以无限期地转动下去。
* **对应现实硬件:** 机器人的驱动轮（如无刷直流电机、步进电机）、螺旋桨桨叶。
* **URDF 核心属性:** `type="continuous"`，只需要定义速度和力矩的上限，不需要位置限制。

#### 3. 直线推杆 / 气缸 (Prismatic Joint)

* **物理表现:** 沿着某一个轴进行直线滑动（平移），**有距离限制**。
* **对应现实硬件:** 直线电机、液压杆、气缸、机械夹爪的平行开合轨道、升降台。
* **URDF 核心属性:** `type="prismatic"`，必须包含 `<limit>` 标签来定义移动的最远/最近距离、最大线速度和推力。

*(注：URDF中还有 `fixed` (固定)、`floating` (浮动)、`planar` (平面) 关节，但它们通常作为被动连接或基座约束，不作为执行器来驱动。)*

---

### 二、 按“控制接口”分类（基于 `ros2_control`）

这是执行器的大脑，决定了你给电机发送什么类型的指令。在 ROS2 中，这是通过在 URDF 中嵌入 `<ros2_control>` 标签来实现的。

#### 1. 位置控制器 (Position Control)

* **说明:** 你发送一个目标位置（角度或距离），执行器自动运动到该位置并保持。
* **典型应用:** 机械臂的轨迹规划（MoveIt2 通常输出位置指令）、舵机转向。
* **ROS2 接口声明:** `<command_interface name="position"/>`

#### 2. 速度控制器 (Velocity Control)

* **说明:** 你发送一个目标速度（角速度或线速度），执行器以此速度持续运动。
* **典型应用:** 轮式移动机器人的底盘驱动（Nav2 或键盘遥控发送 `cmd_vel`）、传送带。
* **ROS2 接口声明:** `<command_interface name="velocity"/>`

#### 3. 力矩/力控制器 (Effort / Torque Control)

* **说明:** 你发送一个力矩（N·m）或推力（N）大小，这类似于控制电机的电流。
* **典型应用:** 四足机器狗的腿部柔性控制、阻抗控制、机械臂的拖动示教、抓取易碎物品的力控夹爪。
* **ROS2 接口声明:** `<command_interface name="effort"/>`

---

### 三、 传动装置 (Transmission)

真实的电机通常带有减速箱（Gearbox）。在 URDF 中，可以使用 `<transmission>` 标签来描述电机（Actuator）与关节（Joint）之间的减速比。

* **简单减速传动 (Simple Transmission):** 最常用，例如电机转 50 圈，关节转 1 圈（减速比 50:1）。
* 虽然在现代 `ros2_control` 中，减速比的计算有时会被直接放入硬件驱动层（Hardware Component）处理，但标准的 URDF 解析器仍然支持 `<transmission>` 标签以保留物理建模的严谨性。

---

### 四、 完整示例：在 ROS2 中声明一个“驱动轮”

要让一个轮子成为真正的“执行器”，你需要在 URDF 中写出它的关节，并声明它的 `ros2_control` 属性（这里以速度控制的连续旋转电机为例，结合 Gazebo 仿真）：

```xml
<!-- 1. 定义物理运动学：连续旋转的轮子关节 -->
<joint name="left_wheel_joint" type="continuous">
  <parent link="base_link"/>
  <child link="left_wheel_link"/>
  <origin xyz="0 0.15 0" rpy="-1.5708 0 0"/>
  <axis xyz="0 0 1"/> <!-- 绕 Z 轴转动 -->
  <limit effort="10.0" velocity="5.0"/> <!-- 限制最大力矩和速度 -->
</joint>

<!-- 2. 定义 ROS2 控制接口：赋予它“生命” -->
<ros2_control name="GazeboSystem" type="system">
  <!-- 指定使用的硬件接口插件（此处为 Gazebo 仿真插件） -->
  <hardware>
    <plugin>gazebo_ros2_control/GazeboSystem</plugin>
  </hardware>
  
  <!-- 将上面定义的 joint 变成执行器 -->
  <joint name="left_wheel_joint">
    <!-- 指令接口：我们打算给这个电机发送“速度”指令 -->
    <command_interface name="velocity"/>
    <!-- 状态接口：我们需要读取这个电机的“位置”和“速度”反馈 -->
    <state_interface name="position"/>
    <state_interface name="velocity"/>
  </joint>
</ros2_control>

<!-- 3. 加载 Gazebo ros2_control 控制器解析插件 -->
<gazebo>
  <plugin filename="libgazebo_ros2_control.so" name="gazebo_ros2_control">
    <parameters>$(find my_robot)/config/my_controllers.yaml</parameters>
  </plugin>
</gazebo>

```

### 总结

在 ROS2 的 URDF 中，**并没有预设某个标签叫 `<motor>**`。我们能构建的所有执行器，本质上都是通过组合 **`Revolute / Continuous / Prismatic` (运动学限制)** 加上 **`Position / Velocity / Effort` (控制逻辑)** 来拼装出来的。掌握了这种积木式的组合逻辑，你就能在 ROS2 中描述出任何种类的机器人致动器。
