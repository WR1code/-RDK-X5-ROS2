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
import rclpy
from rclpy.node import Node
from rclpy.executors import MultiThreadedExecutor
from rclpy.callback_groups import MutuallyExclusiveCallbackGroup
from chapt4_interfaces.srv import FaceDetect
from chapt4_interfaces.msg import FacePosition
from cv_bridge import CvBridge
import face_recognition
import time

class FaceDetectServer(Node):
    def __init__(self):
        super().__init__('face_detect_server')
        self.cb_group = MutuallyExclusiveCallbackGroup()
        self.srv = self.create_service(
            FaceDetect, 
            'face_detect', 
            self.detect_callback,
            callback_group=self.cb_group
        )
        self.bridge = CvBridge()
        self.get_logger().info('✅ 人脸检测服务已启动 (多线程并发模式)，等待请求...')

    def detect_callback(self, request, response):
        self.get_logger().info('收到图像请求，正在独立线程中处理...')
        start_time = time.time()

        try:
            cv_image = self.bridge.imgmsg_to_cv2(request.image, "rgb8")
        except Exception as e:
            self.get_logger().error(f'图像转换失败: {e}')
            response.success = False
            response.message = f'图像解析失败: {str(e)}'
            response.number = 0
            return response

        face_locations = face_recognition.face_locations(cv_image)

        response.success = True
        response.message = '检测成功'
        response.use_time = float(time.time() - start_time)
        response.number = len(face_locations)
        
        response.faces = []
        for loc in face_locations:
            face_msg = FacePosition()
            face_msg.top = loc[0]
            face_msg.right = loc[1]
            face_msg.bottom = loc[2]
            face_msg.left = loc[3]
            response.faces.append(face_msg)

        self.get_logger().info(f'完成检测：发现 {response.number} 个人脸, 耗时 {response.use_time:.4f} 秒')
        return response

def main(args=None):
    rclpy.init(args=args)
    node = FaceDetectServer()
    executor = MultiThreadedExecutor(num_threads=4)
    executor.add_node(node)
    
    try:
        executor.spin()
    except KeyboardInterrupt:
        pass
    finally:
        executor.shutdown()
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

**2. 客户端：`face_detect_client.py`**
```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from chapt4_interfaces.srv import FaceDetect
from cv_bridge import CvBridge
import cv2
import os
from ament_index_python.packages import get_package_share_directory

class FaceDetectClient(Node):
    def __init__(self):
        super().__init__('face_detect_client')
        self.client = self.create_client(FaceDetect, 'face_detect')
        self.bridge = CvBridge()
        self.connect_to_server()

    def connect_to_server(self):
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('人脸检测服务未上线，等待中...')
        self.send_request_async()

    def send_request_async(self):
        pkg_share = get_package_share_directory('demo_python_service')
        image_path = os.path.join(pkg_share, 'resource', 'default.jpg')

        cv_image = cv2.imread(image_path)
        if cv_image is None:
            self.get_logger().error(f'无法读取图像: {image_path}')
            return

        request = FaceDetect.Request()
        request.image = self.bridge.cv2_to_imgmsg(cv_image, encoding="bgr8")

        self.get_logger().info('📤 发送图片进行检测 (异步非阻塞模式)...')
        self.future = self.client.call_async(request)
        self.future.add_done_callback(self.response_callback)

    def response_callback(self, future):
        try:
            response = future.result()
            
            if not response.success:
                self.get_logger().error(f'❌ 服务端返回错误: {response.message}')
                return

            self.get_logger().info('✅ 【检测结果】')
            self.get_logger().info(f'状态: {response.message}')
            self.get_logger().info(f'人脸个数: {response.number}')
            self.get_logger().info(f'识别耗时: {response.use_time:.4f} 秒')
            
            for i, face in enumerate(response.faces):
                self.get_logger().info(
                    f'人脸 {i+1} 位置 -> '
                    f'Top: {face.top}, Right: {face.right}, '
                    f'Bottom: {face.bottom}, Left: {face.left}'
                )
                
        except Exception as e:
            self.get_logger().error(f'服务调用失败或发生异常: {e}')
        finally:
            rclpy.shutdown()

def main(args=None):
    rclpy.init(args=args)
    node = FaceDetectClient()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    except Exception:
        pass

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
