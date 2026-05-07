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


掌握了 ROS 2 Launch 的基础用法后，在实际的大型机器人工程（如自动驾驶、复杂机械臂控制）中，我们往往还需要更高级的动态控制和逻辑编排能力。由于 ROS 2 的 Launch 是纯 Python 编写的，这使得它具备了极高的灵活性。

以下是几个在企业级和进阶 ROS 2 开发中非常实用的高级用法：

### 一、 事件处理器 (Event Handlers)：处理节点崩溃与连带启动

在复杂的系统中，节点的启动往往有前后依赖关系，或者某个核心节点崩溃时需要拉着整个系统一起重启/退出。这时候需要用到 `RegisterEventHandler`。

**场景**：当一个短生命周期的节点（例如地图加载器）执行完毕退出后，再去启动主程序；或者当核心节点意外崩溃时，关闭整个 Launch。

```python
from launch import LaunchDescription
from launch.actions import RegisterEventHandler, LogInfo, EmitEvent
from launch.events.process import ProcessExited
from launch.event_handlers import OnProcessExit
from launch_ros.actions import Node
from launch.actions import ExecuteProcess

def generate_launch_description():
    # 模拟一个会自己退出的任务（比如初始化脚本）
    init_process = ExecuteProcess(
        cmd=['sleep', '3'],
        output='screen'
    )

    # 核心节点
    main_node = Node(
        package='demo_nodes_cpp',
        executable='talker',
        name='main_talker'
    )

    # 事件监听：当 init_process 退出时，打印日志并启动 main_node
    event_handler = RegisterEventHandler(
        event_handler=OnProcessExit(
            target_action=init_process,
            on_exit=[
                LogInfo(msg="初始化任务完成，开始启动主节点..."),
                main_node
            ]
        )
    )

    return LaunchDescription([
        init_process,
        event_handler
    ])
```

### 二、 动态命令执行 (Command Substitution)：Xacro 转 URDF 的利器

在加载机器人模型时，我们通常写的是 `.xacro` 文件，但 `robot_state_publisher` 需要的是 `.urdf` 字符串。在 ROS 2 中，我们不需要提前编译，可以在 Launch 中直接使用 `Command` 动态解析。

```python
import os
from launch import LaunchDescription
from launch.substitutions import Command, FindExecutable, PathJoinSubstitution
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare
from launch_ros.parameter_descriptions import ParameterValue

def generate_launch_description():
    # 定位 xacro 文件
    xacro_file = PathJoinSubstitution([
        FindPackageShare('my_robot_description'),
        'urdf',
        'robot.xacro'
    ])

    # 动态执行 xacro 命令，将其转换为字符串。
    # ParameterValue 配合 value_type=str 极其重要，否则可能会被解析成 yaml 字典
    robot_description_content = ParameterValue(
        Command(['xacro ', xacro_file, ' is_sim:=true']), 
        value_type=str
    )

    robot_state_publisher_node = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        parameters=[{'robot_description': robot_description_content}]
    )

    return LaunchDescription([robot_state_publisher_node])
```

### 三、 组动作与命名空间推送 (GroupAction & PushRosNamespace)

当你通过 `IncludeLaunchDescription` 引入了别人写好的、包含几十个节点的庞大 Launch 文件，且想把它们全部塞进某个特定的命名空间（Namespace）或统一增加某个重映射时，如果去修改原文件就太蠢了。此时应当使用 `GroupAction`。

```python
from launch import LaunchDescription
from launch.actions import GroupAction
from launch_ros.actions import Node, PushRosNamespace

def generate_launch_description():
    # 创建一个组
    sensor_group = GroupAction(
        actions=[
            # 为组内的所有节点强制添加统一的命名空间
            PushRosNamespace('robot_1/sensors'),
            
            # 这里的节点最终名称将变成 /robot_1/sensors/lidar_node
            Node(package='urg_node', executable='urg_node_driver', name='lidar_node'),
            Node(package='camera_ros', executable='camera_node', name='camera_node')
        ]
    )

    return LaunchDescription([sensor_group])
```

### 四、 终极杀器：OpaqueFunction (不透明函数)

**这是高级 Launch 开发中最常遇到的痛点**。`LaunchConfiguration` 获取的参数是“替换变量”（Substitutions），它们在 Python 执行时是没有字符串值的，只有在 Launch 引擎真正启动那一刻才被解析。

如果你想在 Python 代码中**使用传入的参数进行字符串拼接、条件判断或读取外部文件**，你必须进入 Launch 上下文，这就是 `OpaqueFunction` 的作用。

**场景**：根据终端传入的参数 `robot_type`，去 Python 里读取对应的 YAML 文件并提取某些键值。

```python
import yaml
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument, OpaqueFunction
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node

def setup_nodes(context, *args, **kwargs):
    # 1. 这里通过 context.perform() 将 LaunchConfiguration 真正转化为纯 Python 字符串
    rob_type_str = LaunchConfiguration('robot_type').perform(context)
    
    # 2. 现在你可以用纯 Python 逻辑了！比如根据型号读取特定配置
    config_file = f"/tmp/config/{rob_type_str}_config.yaml"
    # with open(config_file, 'r') as f:
    #     params = yaml.safe_load(f)
    
    print(f"正在为 {rob_type_str} 型号机器人生成节点...")

    # 3. 返回你想要启动的 Action 列表
    return [
        Node(
            package='my_pkg',
            executable='my_exec',
            name=f'{rob_type_str}_controller'
        )
    ]

def generate_launch_description():
    return LaunchDescription([
        DeclareLaunchArgument('robot_type', default_value='agv'),
        # 使用 OpaqueFunction 将执行逻辑推迟到上下文建立之后
        OpaqueFunction(function=setup_nodes)
    ])
```

### 五、 管理生命周期节点 (Lifecycle Nodes)

ROS 2 引入了生命周期（Lifecycle）的概念（Unconfigured -> Inactive -> Active）。Launch 可以编排这些节点的启动序列，确保底盘底层驱动激活后，再激活上层导航算法。

```python
from launch import LaunchDescription
from launch_ros.actions import LifecycleNode
from launch_ros.events.lifecycle import ChangeState
from launch.actions import EmitEvent
from launch.events import matches_action
from lifecycle_msgs.msg import Transition

def generate_launch_description():
    # 声明一个生命周期节点
    driver_node = LifecycleNode(
        package='my_driver_pkg',
        executable='driver_node',
        name='hardware_driver',
        namespace=''
    )

    # 触发事件：请求节点进入配置状态 (Configure)
    emit_configure_event = EmitEvent(
        event=ChangeState(
            lifecycle_node_matcher=matches_action(driver_node),
            transition_id=Transition.TRANSITION_CONFIGURE,
        )
    )

    return LaunchDescription([
        driver_node,
        emit_configure_event
    ])
```

### 总结建议

*   **简单项目**：使用 `Node`、`DeclareLaunchArgument` 和 `IncludeLaunchDescription` 足够应对 90% 的场景。
*   **复杂集成/多机协同**：大量使用 `GroupAction` + `PushRosNamespace` 来隔离不同模块。
*   **需要极高定制化逻辑**：果断使用 `OpaqueFunction`，它能让你重新获得写纯 Python 脚本处理数据的自由度。
