<img width="1499" height="735" alt="6012ee4f-e846-47ea-bf92-977df99128a1" src="https://github.com/user-attachments/assets/9f20d9b4-d7b3-4d54-9af8-c2b030ddeae8" />

---

### 第一步：环境与依赖准备

在终端中安装人脸识别和图像转换所需的 Python 库：

```bash
# 安装 face_recognition
pip3 install face_recognition -i https://pypi.tuna.tsinghua.edu.cn/simple

# 安装 opencv
pip3 install opencv-python -i https://pypi.tuna.tsinghua.edu.cn/simple

# 安装 cv_bridge
sudo apt install ros-$ROS_DISTRO-cv-bridge
```

---



### 第一步：创建并编译接口包 (`chapt4_interfaces`)

我们需要自定义一个结构化的 Message 数组用于存放多个人脸的坐标，以及一个 Service 文件用于通信。

**1. 创建包与目录**
```bash
cd ~/ros2_ws/src
ros2 pkg create chapt4_interfaces --build-type ament_cmake --dependencies sensor_msgs
mkdir -p ~/ros2_ws/src/chapt4_interfaces/msg
mkdir -p ~/ros2_ws/src/chapt4_interfaces/srv
```

**2. 编写消息与服务文件**
创建 `~/ros2_ws/src/chapt4_interfaces/msg/FacePosition.msg`：
```text
# FacePosition.msg
# 描述单个人脸的边界框坐标
int32 top
int32 right
int32 bottom
int32 left
```

创建 `~/ros2_ws/src/chapt4_interfaces/srv/FaceDetect.srv`：
```text
# FaceDetect.srv
sensor_msgs/Image image  # 原始图像
---
bool success             # 服务端处理是否成功
string message           # 状态说明或错误信息
int16 number             # 人脸个数
float32 use_time         # 识别耗时
chapt4_interfaces/FacePosition[] faces  # 结构化的人脸位置数组
```

**3. 在 CMakeLists.txt 中注册接口**
修改 `chapt4_interfaces/CMakeLists.txt`：
```cmake
cmake_minimum_required(VERSION 3.8)
project(chapt4_interfaces)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(rosidl_default_generators REQUIRED)

# 注册并生成自定义 Message 和 Service
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/FacePosition.msg"
  "srv/FaceDetect.srv"
  DEPENDENCIES sensor_msgs
)

ament_package()
```

**4. 修改 `package.xml`**
在 `chapt4_interfaces/package.xml` 中添加：
```xml
  <build_depend>rosidl_default_generators</build_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>
```

**5. 编译接口包**
```bash
cd ~/ros2_ws
colcon build --packages-select chapt4_interfaces
source install/setup.bash
```

---

### 第二步：创建服务包并配置 CMakeLists (`demo_python_service`)

**1. 创建基于 CMake 的包与目录**
```bash
cd ~/ros2_ws/src
ros2 pkg create demo_python_service --build-type ament_cmake --dependencies rclpy chapt4_interfaces --license Apache-2.0
mkdir -p ~/ros2_ws/src/demo_python_service/resource
mkdir -p ~/ros2_ws/src/demo_python_service/scripts
```

**2. 下载测试图片**
```bash
wget https://github.com/ultralytics/yolov5/raw/master/data/images/zidane.jpg -O ~/ros2_ws/src/demo_python_service/resource/default.jpg
```

**3. 在 CMakeLists.txt 中注册 Python 文件（核心步骤）**
打开 `demo_python_service/CMakeLists.txt`，重点关注 `install(PROGRAMS ...)` 这一块，这就是向 ROS 2 注册 Python 节点的地方：

```cmake
cmake_minimum_required(VERSION 3.8)
project(demo_python_service)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rclpy REQUIRED)
find_package(chapt4_interfaces REQUIRED)

# ==========================================
# 核心：在此处注册 Python 节点文件
# 将 scripts 目录下的 Python 脚本安装到 lib 目录下，
# 这样 ros2 run 就能识别到这两个可执行文件。
# ==========================================
install(PROGRAMS
  scripts/face_detect_server.py
  scripts/face_detect_client.py
  DESTINATION lib/${PROJECT_NAME}
)

# 注册并安装资源文件夹（图片）
install(DIRECTORY
  resource
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

---

### 第三步：编写 Python 节点代码

进入 `~/ros2_ws/src/demo_python_service/scripts/` 目录，创建以下两个文件。**注意：文件第一行必须保留 `#!/usr/bin/env python3`，这是让 CMake 知道如何执行它的关键。**

**1. 服务端：`face_detect_server.py`**
```python
#!/usr/bin/env python3
# 上面这行叫做 Shebang，告诉 Linux 系统使用 python3 解释器来运行这个脚本

# ================= 导入所需的依赖库 =================
import rclpy                                      # ROS 2 的 Python 客户端核心库
from rclpy.node import Node                       # ROS 2 节点基类，所有自定义节点都要继承它
from rclpy.executors import MultiThreadedExecutor # 多线程执行器，允许节点同时处理多个任务
from rclpy.callback_groups import MutuallyExclusiveCallbackGroup # 互斥回调组，用于控制并发行为

# 导入自定义的服务和消息接口 (你需要提前编译好 chapt4_interfaces 这个功能包)
from chapt4_interfaces.srv import FaceDetect      # 导入用于人脸检测的服务接口 (Service)
from chapt4_interfaces.msg import FacePosition    # 导入用于存放单个人脸位置信息的消息接口 (Message)

from cv_bridge import CvBridge                    # CvBridge 是一座桥梁，专门用于在 ROS 图像格式 和 OpenCV 图像格式之间转换
import face_recognition                           # 强大的开源人脸识别库 (底层基于 dlib)
import time                                       # Python 内置时间库，用来计算检测花了多长时间
# ===================================================

class FaceDetectServer(Node):
    """
    创建一个名为 FaceDetectServer 的节点类，继承自 rclpy.node.Node。
    它的作用是作为一个服务端 (Server)，等待别人发来图片，然后找出图片里的人脸并返回位置。
    """
    def __init__(self):
        # 1. 初始化父类，并给这个节点起个名字叫 'face_detect_server'
        super().__init__('face_detect_server')
        
        # 2. 创建一个互斥回调组
        # 简单来说，同一个互斥回调组里的任务，不能同时执行，必须排队。
        # 结合多线程执行器，可以确保当前服务在处理一张图片时，不会被同一服务的下一个请求强行打断产生数据混乱。
        self.cb_group = MutuallyExclusiveCallbackGroup()
        
        # 3. 创建服务 (Service)
        # 参数1: 服务的类型 FaceDetect
        # 参数2: 服务的名称 'face_detect' (客户端要调用这个名字)
        # 参数3: 收到请求时触发的回调函数 self.detect_callback
        # 参数4: 放入刚才创建的回调组中
        self.srv = self.create_service(
            FaceDetect, 
            'face_detect', 
            self.detect_callback,
            callback_group=self.cb_group
        )
        
        # 4. 初始化图像转换桥梁工具
        self.bridge = CvBridge()
        
        # 5. 在终端打印一句绿色的提示，告诉我们服务端已经成功启动啦！
        self.get_logger().info('✅ 人脸检测服务已启动 (多线程并发模式)，等待请求...')

    def detect_callback(self, request, response):
        """
        这是核心的回调函数！每当客户端发来一张照片请求检测时，就会自动运行这个函数。
        :param request: 客户端发来的请求数据 (包含我们要处理的 ROS 格式图片)
        :param response: 我们要返回给客户端的响应数据 (包含检测结果)
        """
        self.get_logger().info('收到图像请求，正在独立线程中处理...')
        
        # 记录一下开始时间，方便后面计算“耗时”
        start_time = time.time()

        # ============ 步骤1: 图像格式转换 ============
        try:
            # ROS 的图像消息格式 (request.image) 是不能直接用来做图像处理的。
            # 我们必须用 bridge 把它转换成 OpenCV 用的 numpy 数组格式。
            # "rgb8" 表示我们要把它转成 8位深度的 RGB 彩色图像。
            cv_image = self.bridge.imgmsg_to_cv2(request.image, "rgb8")
        except Exception as e:
            # 如果转换失败 (比如图片损坏)，就捕获异常，防止程序崩溃报错退出
            self.get_logger().error(f'图像转换失败: {e}')
            response.success = False                         # 告诉客户端：处理失败了
            response.message = f'图像解析失败: {str(e)}'     # 附带失败原因
            response.number = 0                              # 检测到的人脸数为 0
            return response                                  # 直接返回结果，结束本次处理

        # ============ 步骤2: 执行人脸检测 ============
        # 调用 face_recognition 库的核心方法。
        # 它的返回值是一个列表，里面存放了所有检测到的人脸的坐标框。
        # 每个框的格式是：(top, right, bottom, left) 即 (上边距，右边距，下边距，左边距)
        face_locations = face_recognition.face_locations(cv_image)

        # ============ 步骤3: 组装响应数据 ============
        # 既然代码能走到这里，说明检测顺利完成了，开始给 response 填入数据
        response.success = True
        response.message = '检测成功'
        
        # 计算检测花了多少秒：当前时间 - 开始时间
        response.use_time = float(time.time() - start_time) 
        
        # 列表里有几个元素，就说明找到了几张脸 (len() 函数用来获取列表长度)
        response.number = len(face_locations) 
        
        response.faces = [] # 准备一个空列表，用来装多个人脸的位置信息
        
        # 遍历检测到的每一个人脸坐标 (loc)
        for loc in face_locations:
            face_msg = FacePosition() # 实例化一个我们自定义的 FacePosition 消息对象
            
            # loc 里的四个值分别是 上、右、下、左 (对应 y1, x2, y2, x1)
            face_msg.top = loc[0]
            face_msg.right = loc[1]
            face_msg.bottom = loc[2]
            face_msg.left = loc[3]
            
            # 把装好坐标信息的 face_msg 塞进我们要返回给客户端的列表中
            response.faces.append(face_msg)

        # 在终端打印一下这次检测的最终结果
        self.get_logger().info(f'完成检测：发现 {response.number} 个人脸, 耗时 {response.use_time:.4f} 秒')
        
        # ============ 步骤4: 把结果送回给客户端 ============
        return response

# ================= 主程序入口 =================
def main(args=None):
    # 1. 初始化 ROS 2 客户端库，这是编写任何 ROS 2 节点的必须步骤
    rclpy.init(args=args)
    
    # 2. 实例化我们上面写的那个检测节点
    node = FaceDetectServer()
    
    # 3. 创建多线程执行器 (MultiThreadedExecutor)
    # num_threads=4 表示开启 4 个工作线程。
    # 这样如果有多个客户端同时发图片过来，程序可以把任务分发给不同线程去处理，而不会卡死。
    executor = MultiThreadedExecutor(num_threads=4)
    executor.add_node(node) # 把我们的节点交给这个多线程执行器来管理
    
    try:
        # 4. executor.spin() 是一个死循环，它的作用是让程序一直保持运行状态，
        # 竖起耳朵监听有没有客户端发来请求。直到我们在终端按下 Ctrl+C 才会停止。
        executor.spin()
    except KeyboardInterrupt:
        # 捕获用户按下的 Ctrl+C 终止信号，不做任何报错处理 (直接 pass)
        pass
    finally:
        # 5. 打扫战场：程序结束时，依次关闭执行器、销毁节点、彻底关闭 ROS 2 底层系统
        executor.shutdown()
        node.destroy_node()
        rclpy.shutdown()

# Python 的标准写法，判断这个脚本是不是被当作主程序直接运行的。
# 如果是，就执行 main() 函数。
if __name__ == '__main__':
    main()
```

**2. 客户端：`face_detect_client.py`**
```python
#!/usr/bin/env python3
# 告诉 Linux 系统使用 python3 解释器来运行这个脚本

# ================= 导入所需的依赖库 =================
import rclpy                                      # ROS 2 的 Python 客户端核心库
from rclpy.node import Node                       # ROS 2 节点基类

# 导入自定义的服务接口 (需与服务端保持一致)
from chapt4_interfaces.srv import FaceDetect      

from cv_bridge import CvBridge                    # 用于在 ROS 图像和 OpenCV 图像之间转换的桥梁
import cv2                                        # OpenCV 图像处理库 (这里主要用来读取本地图片)
import os                                         # 操作系统接口，用于拼接文件路径
from ament_index_python.packages import get_package_share_directory # 用于获取 ROS 2 功能包的安装路径
# ===================================================

class FaceDetectClient(Node):
    """
    创建一个名为 FaceDetectClient 的节点类，继承自 Node。
    它的作用是作为一个客户端 (Client)，主动去找服务端，把图片发过去，并等待接收检测结果。
    """
    def __init__(self):
        # 1. 初始化父类，给节点起名叫 'face_detect_client'
        super().__init__('face_detect_client')
        
        # 2. 创建一个服务客户端 (Client)
        # 参数1: 服务类型 FaceDetect
        # 参数2: 要连接的服务名称 'face_detect' (必须和前面服务端创建的名字一模一样)
        self.client = self.create_client(FaceDetect, 'face_detect')
        
        # 3. 初始化图像转换桥梁工具
        self.bridge = CvBridge()
        
        # 4. 调用自定义的连接函数，准备发送请求
        self.connect_to_server()

    def connect_to_server(self):
        """
        连接服务端的方法。
        在发请求之前，我们要先确认服务端是不是已经启动了，不然发了也是白发。
        """
        # wait_for_service(timeout_sec=1.0) 意思是等 1 秒钟，看服务端在不在。
        # 如果不在 (返回 False)，就会一直循环打印提示，直到服务端上线。
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('人脸检测服务未上线，等待中...')
            
        # 只要代码跳出了上面的 while 循环，就说明服务端上线了！
        # 此时调用发送请求的函数
        self.send_request_async()

    def send_request_async(self):
        """
        读取本地图片，并异步发送给服务端的方法。
        异步的意思是：发完请求我就不管了，可以去干别的事，等服务端处理好了再通知我。
        """
        # 1. 找到你要检测的图片放在哪里
        # get_package_share_directory 可以找到编译后功能包的共享目录
        pkg_share = get_package_share_directory('demo_python_service')
        # 拼接出图片的完整绝对路径 (假设图片叫 default.jpg，放在 resource 文件夹下)
        image_path = os.path.join(pkg_share, 'resource', 'default.jpg')

        # 2. 使用 OpenCV 读取这张本地图片
        cv_image = cv2.imread(image_path)
        if cv_image is None:
            # 如果图片不存在或者路径写错了，打印错误并提前结束
            self.get_logger().error(f'无法读取图像: {image_path}')
            return

        # 3. 创建一个请求对象 (对应我们定义的 FaceDetect.srv 里的 Request 部分)
        request = FaceDetect.Request()
        
        # 4. 将 OpenCV 格式的图片 (BGR) 转换为 ROS 认识的图像消息格式，并塞进请求里
        request.image = self.bridge.cv2_to_imgmsg(cv_image, encoding="bgr8")

        self.get_logger().info('📤 发送图片进行检测 (异步非阻塞模式)...')
        
        # 5. 正式发送请求！
        # call_async() 就是异步调用的意思，它会立刻返回一个 future (未来凭证)。
        self.future = self.client.call_async(request)
        
        # 6. 给这个 future 绑定一个“回调函数” (response_callback)。
        # 意思是：等服务端把结果传回来之后，请自动执行 response_callback 这个函数。
        self.future.add_done_callback(self.response_callback)

    def response_callback(self, future):
        """
        处理服务端返回结果的回调函数。
        当服务端处理完图片并把 response 发回来时，这个函数会被自动触发。
        """
        try:
            # future.result() 可以把服务端返回的具体数据 (response) 提取出来
            response = future.result()
            
            # 判断服务端告诉我们的状态是不是成功
            if not response.success:
                self.get_logger().error(f'❌ 服务端返回错误: {response.message}')
                return

            # 如果成功，开心打印各项检测结果
            self.get_logger().info('✅ 【检测结果】')
            self.get_logger().info(f'状态: {response.message}')
            self.get_logger().info(f'人脸个数: {response.number}')
            self.get_logger().info(f'识别耗时: {response.use_time:.4f} 秒')
            
            # 遍历服务端发回来的多个人脸坐标框并打印
            # enumerate() 函数可以帮我们同时获取序号(i)和具体的坐标框(face)
            for i, face in enumerate(response.faces):
                self.get_logger().info(
                    f'人脸 {i+1} 位置 -> '
                    f'Top: {face.top}, Right: {face.right}, '
                    f'Bottom: {face.bottom}, Left: {face.left}'
                )
                
        except Exception as e:
            # 这里是你原来代码被截断的地方：如果在获取结果的过程中报错了，捕获并打印出来
            self.get_logger().error(f'处理服务端响应时发生异常: {e}')


# ================= 主程序入口 (这也是你原来缺失的部分) =================
def main(args=None):
    # 1. 初始化 ROS 2
    rclpy.init(args=args)
    
    # 2. 实例化客户端节点
    node = FaceDetectClient()
    
    try:
        # 3. 保持节点运行，监听各种事件 (比如等待服务端的回调响应)
        rclpy.spin(node)
    except KeyboardInterrupt:
        # 按下 Ctrl+C 退出时不报错
        pass
    finally:
        # 4. 打扫战场：销毁节点，关闭 ROS 2
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 第四步：赋权、编译与运行

由于我们在 CMake 中使用的是 `install(PROGRAMS ...)`，这要求源文件本身必须具备 Linux 的可执行权限。

**1. 赋予 Python 脚本可执行权限（极其重要）**
```bash
chmod +x ~/ros2_ws/src/demo_python_service/scripts/*.py
```
方法一：直接让 CMake 替你跑命令（最直接，推荐）
既然我们每次都要用 colcon build（它底层调用的是 CMake），我们就可以在 CMakeLists.txt 里加一条指令，让它在编译前自动帮你给源文件加执行权限。

打开你的 demo_python_service/CMakeLists.txt，在 install(PROGRAMS ...) 的上方，加入这段代码：

CMake

# ==========================================
# 自动化小技巧：在编译时自动为 Python 脚本赋予可执行权限
# 这样就再也不用手动运行 chmod +x 了
# ==========================================
execute_process(
  COMMAND chmod +x scripts/face_detect_server.py scripts/face_detect_client.py
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  RESULT_VARIABLE chmod_result
)

if(NOT chmod_result EQUAL 0)
  message(WARNING "自动赋予执行权限失败，请检查文件路径！")
endif()
**2. 整体编译工作空间**
```bash
cd ~/ros2_ws
colcon build --packages-select demo_python_service
source install/setup.bash
```

**3. 运行测试**
打开 **终端 1**，运行服务端（注意运行时的名称就是你的脚本文件名，带 `.py` 后缀）：
```bash
ros2 run demo_python_service face_detect_server.py
```

打开 **终端 2**，运行客户端：
```bash
source ~/ros2_ws/install/setup.bash
ros2 run demo_python_service face_detect_client.py
```
