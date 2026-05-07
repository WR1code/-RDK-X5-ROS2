在ROS 2中，Launch文件是用于同时启动多个节点、配置参数、重映射话题以及管理复杂机器人系统的核心工具。与ROS 1主要依赖XML不同，**ROS 2 官方首推且最强大的是使用 Python 编写 Launch 文件**，同时它也支持 XML 和 YAML 格式。

下面详细介绍 ROS 2 中 Python Launch 文件的书写规范、核心组件以及工程实践。

---

### 一、 基础文件规范

1. **命名规范**：通常以 `_launch.py` 结尾，例如 `robot_bringup_launch.py`。
2. **存放位置**：一般存放在功能包根目录下的 `launch/` 文件夹中。
3. **核心结构**：所有的 Python launch 文件都必须包含一个名为 `generate_launch_description()` 的函数，该函数必须返回一个 `LaunchDescription` 对象。

```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        # 在这里添加要启动的节点或执行的动作
    ])
```

---

### 二、 核心组件说明

在 `generate_launch_description()` 中，你可以组合多种“动作（Actions）”。以下是最常用的几个：

#### 1. 启动节点 (`Node`)
这是最基础的功能，用于配置并启动一个 ROS 2 节点。

```python
Node(
    package='demo_nodes_cpp',     # 功能包名称 (必须)
    executable='talker',          # 可执行文件名称 (必须)
    name='my_talker',             # 节点重命名 (可选)
    namespace='my_namespace',     # 命名空间 (可选)
    output='screen',              # 日志输出位置，通常设为 'screen' (可选)
    parameters=[{'my_param': 1}], # 传递参数字典或参数文件路径 (可选)
    remappings=[                  # 话题/服务重映射 (可选)
        ('/chatter', '/my_chatter')
    ]
)
```

#### 2. 声明启动参数 (`DeclareLaunchArgument`)
允许你在终端运行 `ros2 launch` 时动态传入参数。

```python
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration

# 声明参数
use_sim_time_arg = DeclareLaunchArgument(
    'use_sim_time',
    default_value='false',
    description='Use simulation (Gazebo) clock if true'
)

# 获取参数值，后续可以传给Node的parameters
use_sim_time = LaunchConfiguration('use_sim_time')
```

#### 3. 包含其他 Launch 文件 (`IncludeLaunchDescription`)
用于实现模块化，将大型系统的启动拆分为多个小的 launch 文件。

```python
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch_ros.substitutions import FindPackageShare
import os

# 包含另一个包的launch文件
include_other_launch = IncludeLaunchDescription(
    PythonLaunchDescriptionSource([
        os.path.join(
            FindPackageShare('other_package').find('other_package'),
            'launch',
            'other_launch.py'
        )
    ]),
    launch_arguments={'use_sim_time': 'true'}.items(), # 向被包含的launch传递参数
)
```

---

### 三、 完整 Launch 文件示例

以下是一个结合了参数声明、节点启动、话题重映射和条件判断的标准且健壮的 Launch 文件范例：

```python
import os
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, IncludeLaunchDescription
from launch.conditions import IfCondition
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    # 1. 定义 LaunchConfiguration 变量
    namespace = LaunchConfiguration('namespace')
    use_rviz = LaunchConfiguration('use_rviz')

    # 2. 声明 Launch 参数 (可以在终端通过 xxx:=yyy 覆盖)
    declare_namespace_cmd = DeclareLaunchArgument(
        'namespace',
        default_value='',
        description='Top-level namespace')

    declare_use_rviz_cmd = DeclareLaunchArgument(
        'use_rviz',
        default_value='True',
        description='Whether to start RVIZ')

    # 3. 定义要启动的节点
    talker_node = Node(
        package='demo_nodes_cpp',
        executable='talker',
        name='talker',
        namespace=namespace,
        output='screen',
        remappings=[
            ('chatter', 'my_chatter')
        ]
    )

    listener_node = Node(
        package='demo_nodes_cpp',
        executable='listener',
        name='listener',
        namespace=namespace,
        output='screen',
        remappings=[
            ('chatter', 'my_chatter')
        ]
    )

    # 4. 条件启动节点 (例如只有在 use_rviz=True 时才启动 rviz)
    # 注意：这里假设你有 rviz2 并且需要启动它
    rviz_node = Node(
        condition=IfCondition(use_rviz),
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen'
    )

    # 5. 创建 LaunchDescription 对象并填充
    ld = LaunchDescription()

    # 声明参数
    ld.add_action(declare_namespace_cmd)
    ld.add_action(declare_use_rviz_cmd)

    # 添加节点
    ld.add_action(talker_node)
    ld.add_action(listener_node)
    ld.add_action(rviz_node)

    return ld
```

---

### 四、 工程编译规范 (极易被忽略的步骤)

写完 Launch 文件后，**必须配置编译系统**，否则 `ros2 launch` 找不到你的文件。

#### 如果是 C++ (CMake) 功能包：
在 `CMakeLists.txt` 中添加安装指令：
```cmake
install(DIRECTORY
  launch
  DESTINATION share/${PROJECT_NAME}
)
```

#### 如果是 Python 功能包：
在 `setup.py` 的 `data_files` 列表中添加：
```python
import os
from glob import glob

setup(
    # ...
    data_files=[
        ('share/ament_index/resource_index/packages', ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        # 将 launch 文件夹下的所有 .py 文件安装到 share 目录下
        (os.path.join('share', package_name, 'launch'), glob('launch/*_launch.py')),
    ],
    # ...
)
```

### 五、 最佳实践总结

1. **分离配置与代码**：不要在 node 的参数列表里硬编码大量参数，应该将其写入 `.yaml` 文件，然后通过 `parameters=[LaunchConfiguration('params_file')]` 的方式加载。
2. **避免绝对路径**：永远使用 `FindPackageShare('pkg_name').find('pkg_name')` 结合 `os.path.join` 来定位包内的其他资源（如 rviz 配置文件、urdf 模型等），不要使用电脑上的绝对路径。
3. **充分利用条件判断**：使用 `IfCondition` 和 `UnlessCondition` 来控制是否开启仿真、调试工具等，使单个 launch 文件具备更强的通用性。 
