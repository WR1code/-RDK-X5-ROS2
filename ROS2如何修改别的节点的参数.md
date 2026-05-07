在 ROS 2 中，参数（Parameters）的底层机制是基于服务（Services）实现的。每一个支持参数的节点都会自动创建一系列与参数相关的服务（如 `describe_parameters`, `get_parameters`, `set_parameters` 等）。

因此，修改其他节点的参数，本质上就是跨节点调用该节点的 `set_parameters` 服务。根据你的使用场景，可以通过以下四种主要方式来实现：

---

### 1. 使用命令行工具 (CLI) - 适合调试和快速测试

如果你在终端中临时需要修改某个运行中节点的参数，CLI 是最便捷的方式。

*   **查看所有节点及其参数：**
    ```bash
    ros2 param list
    ```
*   **查看某个参数的当前值：**
    ```bash
    ros2 param get <node_name> <parameter_name>
    ```
*   **修改某个参数的值：**
    ```bash
    ros2 param set <node_name> <parameter_name> <value>
    
```
    *示例：将 `camera_node` 的 `framerate` 参数修改为 30*
    ```bash
    ros2 param set /camera_node framerate 30
    ```
    *(注意：终端会自动推断你输入的 `30` 是整数类型。如果是浮点数可以输入 `30.0`，布尔值为 `true/false`。)*

---

### 2. 使用 Launch 文件 - 适合系统启动时配置

如果你想在启动其他节点时，或者在启动由别人编写的节点（如 Navigation2 或 MoveIt2 的节点）时修改其默认参数，可以通过 Python Launch 文件来实现。
```python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(
            package='target_package_name',
            executable='target_executable_name',
            name='target_node',
            # 在这里覆盖目标节点的参数
            parameters=[
                {'my_parameter_name': 'new_string_value'},
                {'enable_feature_x': True}
            ]
        )
    ])
```

---

### 3. 使用 Python API (rclpy) - 适合编写控制脚本

在 Python 节点中修改其他节点的参数，最直接的方法是创建一个针对目标节点 `set_parameters` 服务的客户端（Client）。
```python
# 导入 rclpy 核心库，它是 ROS 2 的 Python 客户端接口
import rclpy
# 导入 ROS 2 节点基类，所有自定义节点都应该继承它
from rclpy.node import Node

# 导入底层的参数服务接口。
# 在 ROS 2 中，每个节点启动时都会自动挂载一个名为 "~/<node_name>/set_parameters" 的服务。
# 该服务使用 rcl_interfaces/srv/SetParameters 服务类型。
from rcl_interfaces.srv import SetParameters
# 导入参数相关的消息类型。在向服务发送请求时，必须严格遵守这些数据结构。
from rcl_interfaces.msg import Parameter, ParameterType, ParameterValue

class ParamModifierNode(Node):
    def __init__(self):
        # 初始化节点，节点名为 'param_modifier_node'。
        # 这个名字在当前的 ROS 2 网络中必须是唯一的。
        super().__init__('param_modifier_node')
        
        # 定义我们需要修改参数的目标节点名称。
        # 注意：这里的斜杠 '/' 代表全局命名空间，确保精确匹配目标节点。
        target_node_name = '/target_node'
        
        # 【核心步骤 1：创建服务客户端】
        # 我们要调用目标节点的参数修改服务。
        # 服务的名称约定是：<目标节点名>/set_parameters
        # 服务的数据类型是：SetParameters
        self.client = self.create_client(SetParameters, f'{target_node_name}/set_parameters')
        
        # 【核心步骤 2：等待服务就绪】
        # 目标节点可能还没启动，或者网络存在延迟。
        # 使用 wait_for_service 进行非阻塞等待（每次检查超时设定为 1.0 秒）。
        while not self.client.wait_for_service(timeout_sec=1.0):
            # 如果等了 1 秒还没找到服务，打印提示信息并继续下一轮循环。
            self.get_logger().info(f'等待 {target_node_name} 的参数服务...')
            
        # 只要代码走到这里，说明目标节点已经在线，并且它的参数服务可以被调用了。
        # 立即执行参数修改操作。
        self.modify_remote_parameter()

    def modify_remote_parameter(self):
        # 【核心步骤 3：构造服务请求】
        # 创建一个 SetParameters 服务的请求对象。
        request = SetParameters.Request()
        
        # 构造参数的值 (ParameterValue)。
        # ROS 2 的参数是强类型的！我们必须显式告诉它这是一个整数 (PARAMETER_INTEGER)，
        # 并且把值赋给对应的字段 (integer_value=42)。
        # 如果是浮点数，类型就是 PARAMETER_DOUBLE，赋值给 double_value 字段。
        param_value = ParameterValue(type=ParameterType.PARAMETER_INTEGER, integer_value=42)
        
        # 构造完整的参数对象 (Parameter)，将参数名和上面定义好的参数值绑定在一起。
        # 假设目标节点里有一个名为 'my_target_param' 的参数。
        param = Parameter(name='my_target_param', value=param_value)
        
        # 将构造好的参数放入请求的参数列表中。
        # 注意：这里是一个列表，意味着你可以一次性在同一个请求中修改多个参数。
        request.parameters = [param]
        
        # 【核心步骤 4：发送异步请求】
        # call_async 会立即返回一个 Future 对象，而不会阻塞主线程。
        # 这在 ROS 2 编程中非常重要，避免节点卡死。
        future = self.client.call_async(request)
        
        # 为这个 Future 对象添加一个回调函数。
        # 当目标节点处理完请求并返回结果时，ROS 2 底层会自动触发 callback_global_param 函数。
        future.add_done_callback(self.callback_global_param)

    def callback_global_param(self, future):
        # 这个回调函数会在服务调用完成时被触发。
        try:
            # 获取服务端的响应结果。
            response = future.result()
            
            # response.results 是一个列表，其顺序和我们发送请求时的 request.parameters 顺序完全一致。
            # 遍历检查每一个参数的修改结果。
            for res in response.results:
                if res.successful:
                    # 如果 successful 为 True，说明目标节点接受了修改。
                    self.get_logger().info('参数修改成功！')
                else:
                    # 如果修改失败（例如目标节点没有声明这个参数，或者类型不匹配），
                    # res.reason 会包含目标节点返回的错误原因。
                    self.get_logger().warning(f'参数修改失败: {res.reason}')
        except Exception as e:
            # 捕获网络异常、服务崩溃等不可预见的错误。
            self.get_logger().error(f'服务调用异常: {e}')

def main(args=None):
    # 初始化 rclpy 环境，解析命令行参数
    rclpy.init(args=args)
    # 实例化我们定义的节点对象
    node = ParamModifierNode()
    # 阻塞式运行节点，保持节点存活，以便处理回调函数（例如 future 的 callback）
    rclpy.spin(node)
    # 收到中断信号（如 Ctrl+C）后，退出 spin，销毁节点释放资源
    node.destroy_node()
    # 清理 rclpy 环境
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 4. 使用 C++ API (rclcpp) - 适合高性能系统级开发

在 C++ 中，`rclcpp` 提供了一个非常方便的专门类 `rclcpp::AsyncParametersClient` 或者 `rclcpp::SyncParametersClient` 来处理外部节点的参数。
```cpp
// 引入 rclcpp 核心头文件，包含了 Node, Client 等基础类
#include <rclcpp/rclcpp.hpp>
// 引入时间相关的标准库，用于处理超时等时间逻辑
#include <chrono>

// 引入时间字面量命名空间，允许我们在代码中直接写 1s (1秒), 500ms (500毫秒) 等直观的时间表示
using namespace std::chrono_literals;

// 定义自定义节点类，继承自 rclcpp::Node
class ParamModifier : public rclcpp::Node
{
public:
    // 构造函数，初始化节点名为 "param_modifier"
    ParamModifier() : Node("param_modifier")
    {
        // 【核心步骤 1：创建异步参数客户端】
        // rclcpp 提供了专门用于操作外部参数的封装类 AsyncParametersClient。
        // 我们不需要手动拼凑 "/target_node/set_parameters" 这样的服务名，
        // 只需要传入当前的节点指针 (this) 和 目标节点的名称 ("target_node") 即可。
        parameters_client_ = std::make_shared<rclcpp::AsyncParametersClient>(this, "target_node");

        // 【核心步骤 2：等待参数服务就绪】
        // wait_for_service 会阻塞，直到连接上目标节点的服务。
        // 这里每次等待 1s。
        while (!parameters_client_->wait_for_service(1s)) {
            // 在等待期间，必须检查 rclcpp::ok()。
            // 这样如果用户按下了 Ctrl+C 强制退出，循环可以及时跳出，而不会变成死循环。
            if (!rclcpp::ok()) {
                RCLCPP_ERROR(this->get_logger(), "在等待服务时被中断。");
                return;
            }
            RCLCPP_INFO(this->get_logger(), "等待目标节点的参数服务...");
        }

        // 服务就绪后，调用我们自定义的函数开始修改参数
        set_remote_parameters();
    }

private:
    void set_remote_parameters()
    {
        // 【核心步骤 3：准备参数列表】
        // C++ 的 rclcpp::Parameter 相比 Python 更加智能，它支持自动类型推导。
        // 当我们传入 3.14，它自动识别为 PARAMETER_DOUBLE。
        // 当我们传入 "hello_ros2"，它自动识别为 PARAMETER_STRING。
        // 我们将它们放入一个初始化列表中。
        auto parameters_to_set = {
            rclcpp::Parameter("my_double_param", 3.14),
            rclcpp::Parameter("my_string_param", "hello_ros2")
        };

        // 【核心步骤 4：异步调用设置参数服务】
        // 调用 set_parameters 发送请求。
        // 第一个参数是我们要修改的参数列表。
        // 第二个参数是一个 Lambda 表达式回调函数，当服务端响应时会自动触发。
        auto future = parameters_client_->set_parameters(
            parameters_to_set,
            // 这是一个捕获了 this 指针的 Lambda 函数，接收一个 shared_future 对象作为参数
            [this](std::shared_future<std::vector<rcl_interfaces::msg::SetParametersResult>> future) {
                // future.get() 会提取服务端的返回结果（包含多个参数的修改结果的 Vector）
                auto results = future.get();
                
                // 遍历每一个参数的修改结果
                for (const auto & result : results) {
                    if (result.successful) {
                        // successful 为 true，说明目标节点接受了该修改
                        RCLCPP_INFO(this->get_logger(), "参数修改成功！");
                    } else {
                        // successful 为 false，打印失败原因 (reason 由目标节点提供)
                        // .c_str() 是将 C++ std::string 转换为 C 风格字符串，以匹配 RCLCPP_ERROR 的格式化输出
                        RCLCPP_ERROR(this->get_logger(), "参数修改失败: %s", result.reason.c_str());
                    }
                }
            }
        );
    }

    // 声明一个类成员变量，用于保存参数客户端的智能指针。
    // 保持它的生命周期与节点对象一致，否则客户端会被提前析构销毁。
    std::shared_ptr<rclcpp::AsyncParametersClient> parameters_client_;
};

int main(int argc, char ** argv)
{
    // 初始化 rclcpp 环境（解析命令行参数，设置内部上下文）
    rclcpp::init(argc, argv);
    
    // 创建我们自定义的 ParamModifier 节点实例，并将其交由 rclcpp::spin 接管。
    // spin 会阻塞当前线程，不断处理底层的回调（例如定时器、服务响应等）。
    rclcpp::spin(std::make_shared<ParamModifier>());
    
    // 当 spin 退出（通常是因为收到了 Ctrl+C 中断），清理 rclcpp 环境并释放资源。
    rclcpp::shutdown();
    
    return 0;
}
```

### ⚠️ 必须注意的关键点（避坑指南）

1. **必须先声明 (Declare)：** 在 ROS 2 中（Foxy 及更高版本），目标节点**必须**使用 `declare_parameter` 声明过该参数，你才能去修改它。如果目标节点没有声明，你尝试 `set_parameter` 会直接失败并报错（除非目标节点启动时将 `allow_undeclared_parameters` 设置为了 true）。
2. **类型必须匹配：** ROS 2 参数是强类型的（Integer, Double, String, Boolean, Arrays 等）。如果你尝试将一个已声明为整数的参数覆盖为字符串，修改操作会被拒绝。
3. **参数回调 (Parameter Callback)：** 即使你成功修改了目标节点的参数（服务返回 `successful=True`），目标节点是否会*实际根据新参数改变行为*，取决于目标节点是否注册了参数修改回调函数（`add_on_set_parameters_callback`）并在其中处理了业务逻辑更新。
```</ParamModifier>



要让这两份代码真正跑起来并看到效果，我们需要构建一个完整的测试环境。

由于 ROS 2 的安全机制，**目标节点必须先声明了参数，才能被外部修改**。为了方便测试，我们将使用 ROS 2 官方提供的一个万能测试节点 `parameter_blackboard`。这个节点的作用相当于一个“黑板”，允许我们随意向它写入任何参数。

以下是针对 Python 和 C++ 代码的具体实操步骤。假设你的 ROS 2 工作空间位于 `~/ros2_ws`。

---

### 第一部分：如何运行 Python 代码

#### 1. 创建 Python 功能包
打开终端（Terminal 1），创建一个名为 `my_py_param_pkg` 的 Python 包：
```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python my_py_param_pkg
```

#### 2. 写入代码
将上面的 Python 代码保存到 `~/ros2_ws/src/my_py_param_pkg/my_py_param_pkg/param_modifier.py` 文件中。

#### 3. 配置启动入口 (setup.py)
打开 `~/ros2_ws/src/my_py_param_pkg/setup.py`，找到 `entry_points` 字段，将其修改为：
```python
    entry_points={
        'console_scripts': [
            'py_modifier = my_py_param_pkg.param_modifier:main'
        ],
    },
```

#### 4. 编译工作空间
```bash
cd ~/ros2_ws
colcon build --packages-select my_py_param_pkg
source install/setup.bash
```

#### 5. 启动目标节点 (Terminal 2)
新开一个终端（Terminal 2），启动官方的“黑板”节点。
*关键点：我们通过 `--ros-args -r __node:=target_node` 将节点重命名为代码中期望的 `target_node`，并允许未声明的参数。*
```bash
source /opt/ros/humble/setup.bash  # 替换为你的ROS2版本，如 foxy/iron
ros2 run demo_nodes_cpp parameter_blackboard --ros-args -r __node:=target_node -p allow_undeclared_parameters:=true
```

#### 6. 运行你的 Python 修改器 (Terminal 1)
回到刚才编译包的终端（Terminal 1），运行代码：
```bash
ros2 run my_py_param_pkg py_modifier
```
**预期输出：** 你会看到终端打印出 `[INFO] [param_modifier_node]: 参数修改成功！`。

#### 7. 验证结果 (Terminal 3)
新开一个终端（Terminal 3），用命令行查看目标节点的参数是否真的被改了：
```bash
ros2 param get /target_node my_target_param
```
**预期输出：** `Integer value is: 42` (这正是我们在 Python 代码中设置的值)。

---

### 第二部分：如何运行 C++ 代码

#### 1. 创建 C++ 功能包
打开终端（Terminal 1），创建一个名为 `my_cpp_param_pkg` 的 C++ 包，并添加依赖：
```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake my_cpp_param_pkg --dependencies rclcpp rcl_interfaces
```

#### 2. 写入代码
将上面的 C++ 代码保存到 `~/ros2_ws/src/my_cpp_param_pkg/src/param_modifier.cpp` 文件中。

#### 3. 配置编译文件 (CMakeLists.txt)
打开 `~/ros2_ws/src/my_cpp_param_pkg/CMakeLists.txt`，在 `find_package(rcl_interfaces REQUIRED)` 之后添加以下几行内容，告诉编译器如何生成可执行文件：

```cmake
add_executable(cpp_modifier src/param_modifier.cpp)
ament_target_dependencies(cpp_modifier rclcpp rcl_interfaces)

install(TARGETS
  cpp_modifier
  DESTINATION lib/${PROJECT_NAME}
)
```

#### 4. 编译工作空间
```bash
cd ~/ros2_ws
colcon build --packages-select my_cpp_param_pkg
source install/setup.bash
```

#### 5. 启动目标节点 (Terminal 2)
如果你刚才的 Terminal 2 里的 `parameter_blackboard` 还在运行，**不用关掉**，继续用它即可。如果你关了，请重新运行：
```bash
ros2 run demo_nodes_cpp parameter_blackboard --ros-args -r __node:=target_node -p allow_undeclared_parameters:=true
```

#### 6. 运行你的 C++ 修改器 (Terminal 1)
回到 Terminal 1，运行 C++ 节点：
```bash
ros2 run my_cpp_param_pkg cpp_modifier
```
**预期输出：** 你会看到终端打印出 `[INFO] [param_modifier]: 参数修改成功！`（会打印两次，因为代码中同时修改了两个参数）。

#### 7. 验证结果 (Terminal 3)
在 Terminal 3 中，使用命令行检查这两个参数的值：
```bash
ros2 param get /target_node my_double_param
ros2 param get /target_node my_string_param
```
**预期输出：** 
分别显示 `Double value is: 3.14` 和 `String value is: hello_ros2`。

---

### 💡 核心逻辑总结

通过这套流程你可以看出，跨节点修改参数是一个典型的 **C/S（客户端/服务端）架构** 交互：
1. **Target Node**（如这里的 parameter_blackboard）是被动方，它启动了 Server，随时准备接收修改请求。
2. **Modifier Node**（你的 Python/C++ 代码）是主动方，它作为 Client 发起修改请求。
3. 两者依靠共同的通道名称 `/<node_name>/set_parameters` 进行握手和数据交换。
