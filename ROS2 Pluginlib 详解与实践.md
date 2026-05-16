在ROS 2中，`pluginlib` 是一个非常强大且核心的C++库，它的主要作用是**在运行时动态加载和卸载插件（类的实例）**。

简单来说，`pluginlib` 允许你的程序在不需要修改源代码、也不需要重新编译基础程序的情况下，通过加载外部的动态链接库（`.so` 文件）来扩展或修改功能。

以下是对 ROS 2 `pluginlib` 的详细介绍和使用指南：

---

### 一、 核心优势与应用场景

#### 1. 为什么需要 pluginlib？

传统的C++开发中，如果你想使用某个类，通常需要包含它的头文件（`.h`/`.hpp`）并在编译时链接它的库文件。这意味着主程序和子模块之间是**强耦合**的。
而 `pluginlib` 实现了一种**松耦合**的架构：主程序只需要知道“基类”（接口）的定义，插件开发者只需继承这个基类去实现具体功能。主程序在运行时会自动去寻找并加载这些实现了基类接口的插件。

#### 2. 典型应用场景

在 ROS 2 生态中，最著名的应用就是 **Nav2（Navigation 2）** 导航框架。

* Nav2 提供了规划器（Planner）、控制器（Controller）、恢复行为（Recovery）的**基类**。
* 开发者可以编写各种算法（如 A*算法插件、Dijkstra算法插件、TEB控制器插件）。
* 用户只需在参数文件（`.yaml`）中修改插件名称，Nav2 就能在运行时动态加载对应的算法，完全不需要重新编译 Nav2 本身。

---

### 二、 核心概念

要理解 `pluginlib`，需要掌握以下四个核心要素：

1. **基类 (Base Class)**：定义了插件必须实现的标准化接口（通常是包含纯虚函数的抽象类）。
2. **插件/派生类 (Plugin / Derived Class)**：继承自基类并实现了具体功能的类。
3. **插件描述文件 (Plugin Description XML)**：一个 XML 文件，用于告诉机器这个插件所在的库路径、插件名称、基类名称等元数据。
4. **类加载器 (ClassLoader)**：主程序用来在运行时根据字符串（插件名称）寻找并实例化对应类的工具。

---

### 三、 pluginlib 开发全流程（以多边形计算为例）

假设我们有一个基础包 `polygon_base`（定义多边形接口），和一个插件包 `polygon_plugins`（实现三角形和正方形）。

#### 1. 定义基类 (Base Class)

创建一个纯虚函数类，作为接口。这通常放在一个单独的包中。

```cpp
// polygon_base/include/polygon_base/regular_polygon.hpp
namespace polygon_base
{
  class RegularPolygon
  {
    public:
      virtual void initialize(double side_length) = 0;
      virtual double area() = 0;
      virtual ~RegularPolygon(){}
    protected:
      double side_length_;
  };
}

```

#### 2. 编写插件类 (Derived Class)

在插件包中继承基类并实现它。

```cpp
// polygon_plugins/include/polygon_plugins/triangle.hpp
#include "polygon_base/regular_polygon.hpp"
#include <cmath>

namespace polygon_plugins
{
  class Triangle : public polygon_base::RegularPolygon
  {
    public:
      void initialize(double side_length) override { side_length_ = side_length; }
      double area() override { return 0.5 * side_length_ * (sqrt(3) / 2) * side_length_; }
  };
}

```

#### 3. 注册插件 (导出类)

**这是最关键的一步！** 你必须在 C++ 文件（`.cpp`）中使用 `pluginlib` 提供的宏来注册你的类，这样它才能被构建成可以被动态加载的共享库。

```cpp
// polygon_plugins/src/triangle.cpp
#include "polygon_plugins/triangle.hpp"
#include <pluginlib/class_list_macros.hpp> // 必须包含此头文件

// 宏参数：派生类, 基类
PLUGINLIB_EXPORT_CLASS(polygon_plugins::Triangle, polygon_base::RegularPolygon)

```

#### 4. 创建插件描述文件 (plugins.xml)

在插件包的根目录下创建一个 XML 文件，告诉 ROS 2 系统你的插件长什么样。

```xml
<!-- polygon_plugins/plugins.xml -->
<library path="polygon_plugins"> <!-- 库的名称，CMake生成的目标名 -->
  <class type="polygon_plugins::Triangle" base_class_type="polygon_base::RegularPolygon">
    <description>This is a triangle plugin.</description>
  </class>
</library>

```

#### 5. 配置包和构建系统

你需要让 ROS 2 工具链知道 XML 文件的存在。

**在 `package.xml` 中导出：**
注意 `<polygon_base>` 是你的**基类所在的包名**。

```xml
<export>
  <polygon_base plugin="${prefix}/plugins.xml" />
</export>

```

**在 `CMakeLists.txt` 中安装：**
将插件编译为动态库，并安装库文件和 XML 文件。

```cmake
# 编译插件库
add_library(polygon_plugins SHARED src/triangle.cpp)
target_include_directories(polygon_plugins PUBLIC include)
ament_target_dependencies(polygon_plugins polygon_base pluginlib)

# 安装库
install(TARGETS polygon_plugins
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

# 安装 XML 描述文件
install(FILES plugins.xml DESTINATION share/${PROJECT_NAME})

```

#### 6. 在主程序中加载和使用插件

使用 `pluginlib::ClassLoader` 来实例化插件对象。

```cpp
#include <pluginlib/class_loader.hpp>
#include "polygon_base/regular_polygon.hpp"

int main(int argc, char** argv)
{
  // 创建类加载器
  // 参数1：基类所在的包名
  // 参数2：基类的完全限定名
  pluginlib::ClassLoader<polygon_base::RegularPolygon> poly_loader("polygon_base", "polygon_base::RegularPolygon");

  try
  {
    // 创建共享指针实例
    std::shared_ptr<polygon_base::RegularPolygon> triangle = poly_loader.createSharedInstance("polygon_plugins::Triangle");
    
    triangle->initialize(10.0);
    printf("Triangle area: %.2f\n", triangle->area());
  }
  catch(pluginlib::PluginlibException& ex)
  {
    printf("The plugin failed to load for some reason. Error: %s\n", ex.what());
  }

  return 0;
}

```

---

### 四、 ROS 2 与 ROS 1 中 pluginlib 的主要区别

如果你之前用过 ROS 1 的 `pluginlib`，过渡到 ROS 2 非常简单，核心思想完全一致，仅有几处小变化：

1. **头文件变动**：ROS 2 中头文件后缀通常变为了 `.hpp`（如 `<pluginlib/class_list_macros.hpp>`）。
2. **包管理/构建系统**：从 `catkin` 变为了 `ament_cmake`，因此在 `CMakeLists.txt` 中使用的宏变成了 `ament_target_dependencies` 等。
3. **智能指针**：ROS 2 的 `ClassLoader` 原生支持返回 `std::shared_ptr` 和 `std::unique_ptr`，这比以前更加现代和安全。

### 五、 总结

`pluginlib` 是 ROS 2 高阶开发中必不可少的工具。它强制开发者采用**面向接口编程**的思想，极大地提高了代码的可复用性、可测试性和系统的可扩展性。掌握它之后，你不仅能更灵活地组织自己的机器人代码结构，也能更顺畅地阅读和修改像 Nav2、MoveIt 2 这样大型开源项目的源码。



这四个组件共同构成了一个优雅的“动态解耦”**架构。它们将程序分为两个世界：**编译时的静态世界**和**运行时的动态世界。

为了让你更直观地理解它们之间的关系，我们可以先看一个通俗的比喻， 然后再深入技术细节。

### 🔌 一个通俗的比喻：电脑与USB设备

* **基类 (Base Class)** = **USB 接口标准**。它规定了接口的形状、引脚和通信协议（纯虚函数）。电脑（主程序）只认这个标准，不关心插进来的是什么。
* **插件 (Plugin)** = **具体的USB设备**（如鼠标、键盘、U盘）。它们内部结构各异，但对外必须长着一个符合标准的 USB 插头（继承基类并实现接口）。
* **XML 描述文件** = **设备的驱动说明书/铭牌**。上面写着“我是罗技鼠标，我的接口类型是USB，我的驱动文件在系统XX目录下”。
* **类加载器 (ClassLoader)** = **操作系统的即插即用管理器**。当你在软件中输入“使用罗技鼠标”时，管理器去翻找说明书（XML），找到对应的驱动文件（.so库），加载它，并让电脑可以通过 USB 标准接口（基类）来控制这个鼠标。

---

### ⚙️ 详细的技术关系脉络

这四者的关系可以按照软件的**开发阶段**和**运行阶段**来划分：

#### 1. 编译时关系：接口与实现 (Base Class ↔ Plugin)

**关系类型：C++ 继承与多态（强耦合）**

* **联系：** 插件类在 C++ 源码层面上**必须**直接继承基类，并且实现基类中定义的所有纯虚函数。
* **目的：** 保证主程序在不知道具体插件长什么样的情况下，可以通过基类的指针（如 `std::shared_ptr<BaseClass>`）安全地调用插件的功能。主程序只和基类头文件打交道，完全不需要包含插件的头文件。

#### 2. 桥梁关系：代码与元数据的映射 (Plugin ↔ XML)

**关系类型：信息注册（静态元数据绑定）**

* **联系：** 编译好的插件会被打包成一个黑盒（动态链接库 `.so` 文件）。XML 文件就像贴在这个黑盒外面的标签。XML 中明确写明了：
1. 插件的完整数据类型（如 `my_namespace::MyPlugin`）。
2. 它继承自哪个基类（如 `base_namespace::MyBase`）。
3. 包含这个插件的 `.so` 库文件的路径。


* **目的：** 在不运行 C++ 代码的情况下，让 ROS 2 系统知道目前有哪些插件可用，以及去哪里加载它们。

#### 3. 发现阶段：运行时寻找 (ClassLoader ↔ XML)

**关系类型：解析与寻址**

* **联系：** 主程序启动时，`ClassLoader` 甚至不知道有哪些插件存在。当主程序要求加载名为 "MyPlugin" 的插件时，`ClassLoader` 会去扫描整个 ROS 2 工作空间中的所有 `plugins.xml` 文件。
* **目的：** `ClassLoader` 通过读取 XML 文本，将主程序给定的“字符串（插件名）”转换为“物理路径（.so 库位置）”，从而知道下一步该去哪里读取内存。

#### 4. 实例化阶段：内存加载与类型转换 (ClassLoader ↔ Base Class ↔ Plugin)

**关系类型：动态加载与向上转型**

* **联系：** 这是整个流程的高潮。`ClassLoader` 找到 `.so` 库后，将其动态加载到内存中，并实例化那个具体的**插件对象 (Plugin)**。
* **关键点：** `ClassLoader` 在设计时就是与特定的**基类**绑定的（例如 `ClassLoader<RegularPolygon>`）。因此，它在创建完插件实例后，会自动将其**向上转型 (Upcasting)** 为基类的智能指针（如 `std::shared_ptr<RegularPolygon>`），然后交还给主程序。

---

### 🔄 总结：数据与控制流图

这四个角色的协作流程可以总结为一条单向的控制流：

1. **开发者写代码：** [**插件**] 继承 [**基类**] $\rightarrow$ 编译成 `.so`。
2. **开发者写配置：** 编写 [**XML**] $\rightarrow$ 记录 [**插件**] 和 [**基类**] 的对应关系及库路径。
3. **主程序要用插件：** 主程序告诉 [**ClassLoader**] 插件名字。
4. **加载器去干活：** [**ClassLoader**] 查阅 [**XML**] $\rightarrow$ 找到 `.so` $\rightarrow$ 在内存中创建 [**插件**] 实例。
5. **返回结果：** [**ClassLoader**] 将实例伪装成 [**基类**] 的指针，返回给主程序。
6. **主程序执行：** 主程序通过 [**基类**] 指针调用方法，实际执行的是 [**插件**] 里的代码。

通过这种精妙的关系，主程序和插件之间被完全隔离开来，唯一连接它们的纽带就是**基类**和**XML描述**，从而实现了极致的扩展性。





在 C++ 和插件化架构（如 ROS 2 的 `pluginlib`）中，“插件抽象类”（通常被称为基类 Base Class）扮演着“标准化接口”或“契约”的角色。

简单来说，**它是一个只定义了“应该有哪些功能”，但“不写具体怎么实现”的类。**

我们可以从以下三个方面来深入理解它：

### 1. 为什么叫“抽象类”？

在 C++ 中，如果一个类包含至少一个**纯虚函数（Pure Virtual Function）**，这个类就被称为“抽象类”。

* **纯虚函数的标志：** 在函数声明的末尾加上 `= 0;`。
* **抽象类的核心特性：** 你**绝对不能**直接实例化它（不能 `new` 一个抽象类的对象）。它生来就是为了被别人继承的。

```cpp
// 这是一个抽象类（插件基类）
class Animal 
{
public:
    // 纯虚函数：只定义了“动物都会叫”，但没写具体怎么叫
    virtual void make_sound() = 0; 
    
    // 虚析构函数（基类必备，防止内存泄漏）
    virtual ~Animal() {} 
};

```

### 2. 它在插件系统中的作用是什么？

在主程序和外部插件之间，抽象类充当了“翻译官”**或**“标准协议”。

* **对主程序而言：** 主程序在编译时，根本不知道未来会有猫、狗还是鸭子插件。主程序只认识 `Animal` 这个抽象类。主程序写死的逻辑是：“只要加载进来一个 `Animal`，我就调用它的 `make_sound()` 方法”。
* **对插件开发者而言：** 如果你想写一个“猫”的插件，你必须继承 `Animal` 这个抽象类，并且**强制要求**你必须实现 `make_sound()` 这个方法，否则你的插件连编译都通不过。

```cpp
// 这是一个具体的插件类（派生类）
class CatPlugin : public Animal 
{
public:
    // 必须实现基类规定的动作
    void make_sound() override {
        printf("Meow! Meow!\n");
    }
};

```

### 3. 回顾上文的多边形例子

在上一篇的 `pluginlib` 介绍中，我们用到了下面这段代码：

```cpp
namespace polygon_base
{
  class RegularPolygon  // 这就是插件抽象类！
  {
    public:
      virtual void initialize(double side_length) = 0; // 纯虚函数
      virtual double area() = 0;                       // 纯虚函数
      virtual ~RegularPolygon(){}
  };
}

```

这里的 `RegularPolygon` 就是一个插件抽象类。它强行规定了：

1. 任何想成为多边形计算插件的代码，都必须有初始化边长的能力（`initialize`）。
2. 都必须有计算面积的能力（`area`）。

### 总结

**插件抽象类**就是一个**空壳子**。主程序拿着这个空壳子去指挥工作，而各种不同的插件负责把这个空壳子的内部逻辑填满。正是因为有了这种“只认抽象空壳，不认具体实现”的设计，才实现了主程序和插件之间的彻底解耦。






在ROS 2中，使用 `pluginlib` 构建运动控制节点是业界标准的高级实践。它的核心优势在于**解耦**和**多态**：你可以将运动控制算法（如PID、纯跟踪Pure Pursuit、LQR、MPC）写成独立的插件。主节点在运行时可以通过读取参数动态加载不同的算法，而无需重新编译主程序。这对于测试和维护来说非常完美。

以下是使用 `pluginlib` 构建一个完整、可维护的 ROS 2 运动控制节点的详细图文教程。

---

### 第一步：创建包结构

为了保持高可维护性，我们创建一个包含接口、插件实现和主节点的包：`motion_control_core`。

打开终端，进入你的工作空间 `src` 目录：

```bash
ros2 pkg create --build-type ament_cmake motion_control_core \
--dependencies rclcpp pluginlib geometry_msgs

```

---

### 第二步：定义基类（接口）

这是所有控制算法必须遵循的“契约”。将节点指针传递给插件非常重要，这样插件就可以自己声明和读取属于它的参数（例如PID的kp, ki, kd）。

创建文件 `include/motion_control_core/motion_controller_base.hpp`：

```cpp
#ifndef MOTION_CONTROL_CORE__MOTION_CONTROLLER_BASE_HPP_
#define MOTION_CONTROL_CORE__MOTION_CONTROLLER_BASE_HPP_

#include <string>
#include <rclcpp/rclcpp.hpp>
#include <geometry_msgs/msg/twist.hpp>

namespace motion_control_core
{

class MotionControllerBase
{
public:
  // 必须提供虚析构函数
  virtual ~MotionControllerBase() = default;

  /**
   * @brief 插件初始化函数
   * @param node 提供主节点的共享指针，用于获取参数或创建订阅/发布
   * @param plugin_name 插件的名称（命名空间前缀）
   */
  virtual void initialize(
    rclcpp::Node::SharedPtr node, 
    const std::string & plugin_name) = 0;

  /**
   * @brief 计算并输出控制指令
   * @return geometry_msgs::msg::Twist 速度指令
   */
  virtual geometry_msgs::msg::Twist compute_velocity() = 0;
};

}  // namespace motion_control_core

#endif  // MOTION_CONTROL_CORE__MOTION_CONTROLLER_BASE_HPP_

```

---

### 第三步：实现具体插件（例如：PID控制器）

现在我们实现一个简单的插件。
创建头文件 `include/motion_control_core/plugins/pid_controller.hpp`：

```cpp
#ifndef MOTION_CONTROL_CORE__PLUGINS__PID_CONTROLLER_HPP_
#define MOTION_CONTROL_CORE__PLUGINS__PID_CONTROLLER_HPP_

#include "motion_control_core/motion_controller_base.hpp"

namespace motion_control_core
{
namespace plugins
{

class PidController : public motion_control_core::MotionControllerBase
{
public:
  PidController() = default;
  ~PidController() override = default;

  void initialize(rclcpp::Node::SharedPtr node, const std::string & plugin_name) override;
  geometry_msgs::msg::Twist compute_velocity() override;

private:
  rclcpp::Logger logger_{rclcpp::get_logger("PidController")};
  double kp_, ki_, kd_;
};

}  // namespace plugins
}  // namespace motion_control_core

#endif  // MOTION_CONTROL_CORE__PLUGINS__PID_CONTROLLER_HPP_

```

创建源文件 `src/plugins/pid_controller.cpp`。**这里最关键的是使用宏导出插件**：

```cpp
#include "motion_control_core/plugins/pid_controller.hpp"
// 必须包含这个头文件来导出类
#include <pluginlib/class_list_macros.hpp> 

namespace motion_control_core
{
namespace plugins
{

void PidController::initialize(rclcpp::Node::SharedPtr node, const std::string & plugin_name)
{
  logger_ = node->get_logger();
  
  // 声明插件自己的参数
  node->declare_parameter(plugin_name + ".kp", 1.0);
  node->declare_parameter(plugin_name + ".ki", 0.0);
  node->declare_parameter(plugin_name + ".kd", 0.1);

  node->get_parameter(plugin_name + ".kp", kp_);
  node->get_parameter(plugin_name + ".ki", ki_);
  node->get_parameter(plugin_name + ".kd", kd_);

  RCLCPP_INFO(logger_, "PID Controller Initialized with Kp: %.2f, Ki: %.2f, Kd: %.2f", kp_, ki_, kd_);
}

geometry_msgs::msg::Twist PidController::compute_velocity()
{
  geometry_msgs::msg::Twist cmd_vel;
  // 这里写具体的PID控制逻辑，为演示直接赋值
  cmd_vel.linear.x = 0.5 * kp_; 
  cmd_vel.angular.z = 0.1;
  RCLCPP_DEBUG(logger_, "Computing PID velocity");
  return cmd_vel;
}

}  // namespace plugins
}  // namespace motion_control_core

// 关键宏：将 PidController 注册为 MotionControllerBase 类型的插件
PLUGINLIB_EXPORT_CLASS(
  motion_control_core::plugins::PidController,
  motion_control_core::MotionControllerBase)

```

---

### 第四步：编写插件描述文件 (XML)

`pluginlib` 需要一个 XML 文件来知道去哪里找到这个类。
在包的根目录（和 `package.xml` 同级）创建 `motion_controller_plugins.xml`：

```xml
<library path="motion_control_plugins">
  <class type="motion_control_core::plugins::PidController" 
         base_class_type="motion_control_core::MotionControllerBase">
    <description>
      A standard PID motion controller plugin.
    </description>
  </class>
</library>

```

*注意：`path` 属性对应的是我们在 CMakeLists 里即将编译出的动态链接库的名字。*

---

### 第五步：开发主节点（Loader）

主节点负责管理生命周期、加载插件并通过定时器或回调调用插件的计算函数。
创建 `src/motion_control_node.cpp`：

```cpp
#include <rclcpp/rclcpp.hpp>
#include <pluginlib/class_loader.hpp>
#include <geometry_msgs/msg/twist.hpp>
#include "motion_control_core/motion_controller_base.hpp"

class MotionControlNode : public rclcpp::Node
{
public:
  MotionControlNode() 
  : Node("motion_control_node"),
    // 初始化ClassLoader，指明包名和基类名
    plugin_loader_("motion_control_core", "motion_control_core::MotionControllerBase")
  {
    // 声明参数：决定加载哪个插件
    this->declare_parameter("controller_plugin", "motion_control_core::plugins::PidController");
    std::string plugin_type = this->get_parameter("controller_plugin").as_string();

    try {
      // 动态加载插件实例
      controller_ = plugin_loader_.createSharedInstance(plugin_type);
      
      // 初始化插件，传入当前节点指针和前缀名
      controller_->initialize(this->shared_from_this(), "my_controller");
      RCLCPP_INFO(this->get_logger(), "Successfully loaded plugin: %s", plugin_type.c_str());
    } 
    catch (pluginlib::PluginlibException & ex) {
      RCLCPP_ERROR(this->get_logger(), "Failed to load plugin. Error: %s", ex.what());
      return;
    }

    // 设置发布者和控制循环定时器
    cmd_vel_pub_ = this->create_publisher<geometry_msgs::msg::Twist>("cmd_vel", 10);
    timer_ = this->create_wall_timer(
      std::chrono::milliseconds(100), // 10Hz
      std::bind(&MotionControlNode::control_loop, this));
  }

private:
  void control_loop()
  {
    if (controller_) {
      auto cmd = controller_->compute_velocity();
      cmd_vel_pub_->publish(cmd);
    }
  }

  pluginlib::ClassLoader<motion_control_core::MotionControllerBase> plugin_loader_;
  std::shared_ptr<motion_control_core::MotionControllerBase> controller_;
  
  rclcpp::Publisher<geometry_msgs::msg::Twist>::SharedPtr cmd_vel_pub_;
  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char ** argv)
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<MotionControlNode>();
  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}

```

---

### 第六步：配置编译和导出依赖

这一步极其重要，很多新手配置不好会导致找不到插件。

修改 `package.xml`，导出你的 XML 描述文件，确保下游能发现它：

```xml
  <depend>rclcpp</depend>
  <depend>pluginlib</depend>
  <depend>geometry_msgs</depend>

  <!-- 关键导出部分 -->
  <export>
    <build_type>ament_cmake</build_type>
    <motion_control_core plugin="${prefix}/motion_controller_plugins.xml" />
  </export>

```

修改 `CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.8)
project(motion_control_core)

# 寻找依赖
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(pluginlib REQUIRED)
find_package(geometry_msgs REQUIRED)

# 1. 编译插件共享库
add_library(motion_control_plugins SHARED
  src/plugins/pid_controller.cpp
)
target_include_directories(motion_control_plugins PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)
ament_target_dependencies(motion_control_plugins rclcpp pluginlib geometry_msgs)

# 2. 编译主节点可执行文件
add_executable(motion_control_node src/motion_control_node.cpp)
target_include_directories(motion_control_node PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)
ament_target_dependencies(motion_control_node rclcpp pluginlib geometry_msgs)

# 3. 导出插件描述文件
pluginlib_export_plugin_description_file(motion_control_core motion_controller_plugins.xml)

# 4. 安装规则
install(TARGETS
  motion_control_plugins
  motion_control_node
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY include/
  DESTINATION include
)

install(FILES motion_controller_plugins.xml
  DESTINATION share/${PROJECT_NAME}
)

ament_package()

```

---

### 第七步：编译与运行测试

回到工作空间根目录进行编译：

```bash
colcon build --packages-select motion_control_core
source install/setup.bash

```

**运行主节点：**

```bash
ros2 run motion_control_core motion_control_node

```

你会在终端看到：

```text
[INFO] [PidController]: PID Controller Initialized with Kp: 1.00, Ki: 0.00, Kd: 0.10
[INFO] [motion_control_node]: Successfully loaded plugin: motion_control_core::plugins::PidController

```

**验证可维护性和灵活性（修改参数）：**
由于我们把参数读取交给了插件内部的 `initialize`，你可以在运行时覆盖它们：

```bash
ros2 run motion_control_core motion_control_node --ros-args -p my_controller.kp:=2.5 -p controller_plugin:="motion_control_core::plugins::PidController"

```

### 总结

这种架构让你拥有了极高的代码维护性。如果明天你需要开发一个 LQR 算法，你**不需要修改 `motion_control_node.cpp` 的任何代码**。你只需要新建一个 `lqr_controller.cpp`，继承 `MotionControllerBase`，并在 `XML` 和 `CMakeLists` 中注册它。运行时通过修改 `controller_plugin` 参数，你的机器人就能立刻切换大脑！
