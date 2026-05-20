在ROS 2中，**URDF (Unified Robot Description Format)** 是一种基于XML的规范，用于描述机器人的物理结构、视觉外观和碰撞属性。构建一个小车模型，本质上就是用代码“搭积木”，将小车的各个底盘、车轮和传感器通过关节连接起来。

以下是通过URDF构建小车模型的详细方法和步骤：

### 1. URDF 的核心概念

在编写代码之前，需要理解URDF的两个最核心元素：

* **`<link>` (连杆/部件)：** 描述机器人的刚体部分（例如：车身底盘、左轮、右轮、雷达）。它包含三个主要属性：
* `<visual>`：外观属性（形状、颜色、尺寸），用于在RViz中显示。
* `<collision>`：碰撞属性，用于物理仿真（如Gazebo）和避障计算。
* `<inertial>`：惯性属性（质量、惯性张量），用于物理引擎计算力学。


* **`<joint>` (关节)：** 描述两个 `<link>` 之间的连接关系和运动方式。
* **类型 (`type`)：** 常见的有 `fixed`（固定，如雷达和车身）、`continuous`（连续旋转，如车轮）、`revolute`（有角度限制的旋转，如舵机）。
* **父子关系：** 定义 `<parent>` 和 `<child>`，例如车身是父，车轮是子。



---

### 2. 构建小车模型的标准步骤

假设我们要构建一个**两轮差速小车（带一个万向轮）**，下面是具体的实施步骤。

#### 第一步：创建 ROS 2 功能包

在你的工作空间（如 `~/ros2_ws/src`）中创建一个专门用于存放机器人描述文件的功能包：

```bash
ros2 pkg create --build-type ament_cmake my_car_description
cd my_car_description
mkdir urdf launch rviz

```

#### 第二步：编写 URDF 文件

在 `urdf` 目录下新建一个文件 `my_car.urdf`。我们将分块构建这个小车。

**1. 定义机器人和主车身 (Base Link)**
我们将车身定义为一个蓝色的长方体。

```xml
<?xml version="1.0"?>
<robot name="my_simple_car">

  <!-- 定义材料颜色 -->
  <material name="blue">
    <color rgba="0 0 0.8 1"/>
  </material>
  <material name="black">
    <color rgba="0 0 0 1"/>
  </material>

  <!-- 车身 (Base Link) -->
  <link name="base_link">
    <visual>
      <geometry>
        <!-- 车身尺寸：长0.4m, 宽0.2m, 高0.1m -->
        <box size="0.4 0.2 0.1"/>
      </geometry>
      <material name="blue"/>
    </visual>
    <collision>
      <geometry>
        <box size="0.4 0.2 0.1"/>
      </geometry>
    </collision>
    <!-- 简化的惯性参数 -->
    <inertial>
      <mass value="5.0"/>
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01"/>
    </inertial>
  </link>

</robot>

```

**2. 添加驱动车轮 (Wheels)**
接下来定义左轮，并通过 `continuous` 类型的关节连接到 `base_link` 上。轮子一般用圆柱体 (`cylinder`) 表示。由于圆柱体默认是朝向Z轴的，我们需要将其翻转绕X或Y轴旋转（使用 `<origin rpy="..." />`）使其变成车轮的姿态。

在 `<robot>` 标签内继续添加：

```xml
  <!-- 左轮 -->
  <link name="left_wheel">
    <visual>
      <!-- 圆柱体默认立着，需要绕X轴旋转90度 (1.5708弧度) -->
      <origin xyz="0 0 0" rpy="1.5708 0 0"/>
      <geometry>
        <!-- 半径0.1m, 宽度0.05m -->
        <cylinder radius="0.1" length="0.05"/>
      </geometry>
      <material name="black"/>
    </visual>
  </link>

  <!-- 左轮关节 -->
  <joint name="left_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="left_wheel"/>
    <!-- 关节位置：相对于车身中心向后靠，向左偏移 -->
    <origin xyz="-0.1 0.125 0" rpy="0 0 0"/>
    <!-- 旋转轴：绕Y轴旋转 -->
    <axis xyz="0 1 0"/>
  </joint>

  <!-- 右轮 (与左轮对称) -->
  <link name="right_wheel">
    <visual>
      <origin xyz="0 0 0" rpy="1.5708 0 0"/>
      <geometry>
        <cylinder radius="0.1" length="0.05"/>
      </geometry>
      <material name="black"/>
    </visual>
  </link>

  <joint name="right_wheel_joint" type="continuous">
    <parent link="base_link"/>
    <child link="right_wheel"/>
    <!-- 向右偏移 -->
    <origin xyz="-0.1 -0.125 0" rpy="0 0 0"/>
    <axis xyz="0 1 0"/>
  </joint>

```

**3. 添加支撑万向轮 (Caster Wheel)**
两轮小车需要一个支撑轮来保持平衡。通常用一个较小的球体 (`sphere`) 表示，使用固定的关节。

```xml
  <!-- 前置万向轮 -->
  <link name="caster_wheel">
    <visual>
      <geometry>
        <sphere radius="0.05"/>
      </geometry>
      <material name="black"/>
    </visual>
  </link>

  <joint name="caster_wheel_joint" type="fixed">
    <parent link="base_link"/>
    <child link="caster_wheel"/>
    <!-- 位于车身前方，高度为了与后轮齐平通常需要下沉 -->
    <origin xyz="0.15 0 -0.05" rpy="0 0 0"/>
  </joint>

```

---

### 3. 在 ROS 2 中可视化模型 (RViz2)

写好URDF后，我们需要启动 `robot_state_publisher` 节点来读取URDF，发布坐标系变换 (TF)，并在 RViz2 中查看。

在 `launch` 目录下新建 `display.launch.py`：

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node

def generate_launch_description():
    pkg_name = 'my_car_description'
    urdf_file = os.path.join(
        get_package_share_directory(pkg_name),
        'urdf',
        'my_car.urdf'
    )

    with open(urdf_file, 'r') as infp:
        robot_desc = infp.read()

    return LaunchDescription([
        # 发布机器人状态 (TF)
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            name='robot_state_publisher',
            output='screen',
            parameters=[{'robot_description': robot_desc}]
        ),
        # 启动关节状态发布器 (允许使用滑块控制车轮转动)
        Node(
            package='joint_state_publisher_gui',
            executable='joint_state_publisher_gui',
            name='joint_state_publisher_gui'
        ),
        # 启动 RViz2
        Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2',
            output='screen'
        )
    ])

```

*(注意：记得在 `CMakeLists.txt` 中安装 `urdf`, `launch` 和 `rviz` 目录，并编译工作空间。)*

在RViz2打开后，你需要：

1. 将 **Fixed Frame** 设置为 `base_link`。
2. 点击左下角 **Add**，选择 **RobotModel**。
3. 你就能看到你构建的蓝黑配色小车了，并且可以通过弹出的 GUI 界面拖动滑块，看到左右轮转动。

---

### 4. 进阶建议：使用 Xacro

写纯 URDF 会遇到一个痛点：代码冗余度极高。比如左轮和右轮除了Y轴位置不同，其他所有代码完全一样。

在实际的 ROS 2 开发中，**强烈建议使用 Xacro (XML Macros)** 来替代纯 URDF。Xacro 允许你：

* **使用变量：** 比如定义 `wheel_radius = 0.1`，方便全局修改。
* **数学运算：** 比如 `origin xyz="${body_length/2} 0 0"`。
* **定义宏 (Macros)：** 像写函数一样定义一个“生成车轮”的模板，只需调用两次就能生成左右轮，极大精简代码。

可以通过运行 `xacro my_car.urdf.xacro > my_car.urdf` 将其转换回标准的 URDF 供系统使用（在 ROS 2 launch 文件中通常会利用 `Command` 自动解析 Xacro）。



**Xacro (XML Macros)** 是 ROS 中用来简化 URDF 编写的神器。如果你把 URDF 比作原生的 HTML，那么 Xacro 就相当于带了变量、函数和数学运算的模板引擎。

使用 Xacro，你可以告别大量重复的代码（比如写四个一模一样的车轮），并且只需修改一个变量就能全局调整模型尺寸。

以下是 Xacro 的核心用法和实战指南。

---

### 1. 基础准备：声明 Xacro 命名空间

要让系统识别 `.xacro` 文件，必须在 `<robot>` 标签中添加 Xacro 的命名空间。

```xml
<?xml version="1.0"?>
<robot name="my_car" xmlns:xacro="http://www.ros.org/wiki/xacro">
    <!-- 所有的 xacro 代码写在这里面 -->
</robot>

```

---

### 2. Xacro 的四大核心功能

#### ① 属性与变量 (`Property`)

你可以把常用的尺寸、质量等定义为变量，方便统一修改。

* **定义：** `<xacro:property name="变量名" value="数值" />`
* **调用：** `${变量名}`

```xml
<!-- 定义车轮的半径和宽度 -->
<xacro:property name="wheel_radius" value="0.1" />
<xacro:property name="wheel_length" value="0.05" />

<!-- 调用变量 -->
<cylinder radius="${wheel_radius}" length="${wheel_length}"/>

```

#### ② 数学运算 (`Math`)

在 `${}` 内部，不仅可以调用变量，还可以直接进行基本的数学运算（加减乘除、甚至调用 Python 的 math 库，如 `pi`）。

```xml
<xacro:property name="body_length" value="0.4" />

<!-- 把车轮位置设置在车身长度的四分之一处 -->
<origin xyz="${body_length / 4.0} 0 0" rpy="0 ${pi/2} 0"/>

```

#### ③ 宏定义/函数 (`Macro`) —— **最强大的功能**

你可以把重复的模块（比如车轮）写成一个带参数的“函数”，然后多次调用。

* **定义宏：** `<xacro:macro name="宏名称" params="参数1 参数2">`
* **调用宏：** `<xacro:宏名称 参数1="值" 参数2="值"/>`

**示例：定义一个生成车轮的宏**

```xml
<!-- 定义名为 wheel 的宏，接收 prefix(前缀,如left/right) 和 y_offset(Y轴偏移量) 两个参数 -->
<xacro:macro name="wheel" params="prefix y_offset">
    
    <link name="${prefix}_wheel">
        <visual>
            <geometry>
                <cylinder radius="0.1" length="0.05"/>
            </geometry>
            <material name="black"/>
        </visual>
    </link>

    <joint name="${prefix}_wheel_joint" type="continuous">
        <parent link="base_link"/>
        <child link="${prefix}_wheel"/>
        <!-- 使用 y_offset 参数控制左右位置 -->
        <origin xyz="-0.1 ${y_offset} 0" rpy="${pi/2} 0 0"/>
        <axis xyz="0 0 1"/>
    </joint>

</xacro:macro>

<!-- ======================================= -->
<!-- 一键调用！瞬间生成左右两个车轮和对应的关节 -->
<xacro:wheel prefix="left" y_offset="0.125" />
<xacro:wheel prefix="right" y_offset="-0.125" />

```

#### ④ 文件包含 (`Include`)

当机器人变得复杂时（比如加入了机械臂、各种传感器），把所有代码写在一个文件里会非常臃肿。你可以把不同部分写在不同的 `.xacro` 文件中，然后像 C++ 的 `#include` 一样组合起来。

```xml
<!-- 引入其他 xacro 文件 -->
<xacro:include filename="lidar.xacro" />
<xacro:include filename="camera.xacro" />

```

---

### 3. 如何在 ROS 2 中运行 Xacro？

Xacro 文件不能直接被 RViz2 或 Gazebo 识别，系统最终需要的是纯净的 URDF。在 ROS 2 中，有两种常见的使用方式：

#### 方法 A：命令行手动转换 (用于调试)

你可以使用命令行工具将 `.xacro` 文件直接编译成 `.urdf` 文件，检查生成的代码对不对。

```bash
# 安装 xacro 工具（如果还没安装的话）
sudo apt install ros-<your_distro>-xacro

# 将 xacro 转换为 urdf 并输出到终端查看
xacro my_car.urdf.xacro

# 将转换后的结果保存为标准的 urdf 文件
xacro my_car.urdf.xacro > my_car.urdf

```

#### 方法 B：在 Launch 文件中动态解析 (官方推荐)

在实际的 ROS 2 开发中，我们几乎不会手动去生成 `.urdf` 文件。我们会在 `launch.py` 文件中，利用 `Command` 动作，在节点启动的瞬间动态解析 Xacro 文件。

修改上一篇提到的 `display.launch.py`，将读取文件的部分替换为执行 Xacro 命令：

```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node
# 引入 Command 用于执行 shell 命令
from launch.substitutions import Command 

def generate_launch_description():
    pkg_name = 'my_car_description'
    xacro_file = os.path.join(
        get_package_share_directory(pkg_name),
        'urdf',
        'my_car.urdf.xacro' # 注意后缀改成了 xacro
    )

    # 动态执行 xacro 命令，将结果作为字符串存入 robot_description
    robot_description = Command(['xacro ', xacro_file])

    return LaunchDescription([
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            # 直接将动态解析后的模型文本传给参数
            parameters=[{'robot_description': robot_description}] 
        ),
        # ... (启动 rviz2 等其他节点)
    ])

```

---

掌握了 Xacro 的 `Property`、`Macro` 和 `Include`，你就具备了构建复杂机器人的基础。

接下来，你想了解**如何用 Xacro 模块化地给这辆小车添加一个激光雷达（LiDAR）模型**，还是想看看**如何为模型添加 Gazebo 仿真标签使其能在物理引擎中跑起来**？
