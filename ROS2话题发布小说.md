
### 第一阶段：代码保姆级注释

#### 1. 发布者代码 (`novel_publisher.py`) - 广播站

```python
# 导入 ROS 2 的 Python 接口库，这是写所有 ROS 2 Python 节点的基础
import rclpy 
# 从 rclpy 库中导入 Node 类。我们写的节点必须要“继承”这个基础类
from rclpy.node import Node 
# 导入标准的消息类型 String（字符串）。因为我们要发文字，所以用这个格式
from std_msgs.msg import String 
import os

# 定义一个名为 NovelPublisher 的类，它继承自 Node
class NovelPublisher(Node):
    
    # 这是类的“初始化”函数，当这个程序一启动，首先就会执行这里面的代码
    def __init__(self):
        # 给这个节点起个正式的名字叫 'novel_publisher'，在 ROS 2 系统里这就是它的身份证名
        super().__init__('novel_publisher')
        
        # 【核心步骤1：创建发布者】
        # self.create_publisher 需要三个参数：
        # 1. String: 规定发送的数据类型是字符串
        # 2. 'novel_channel': 规定发送的频道名字（话题名），订阅者必须听这个频道才能收到
        # 3. 10: 消息队列长度。可以理解为“缓存区”，如果网卡了，最多把10条消息排队，多了就丢弃旧的
        self.publisher_ = self.create_publisher(String, 'novel_channel', 10)
        
        # 【核心步骤2：创建定时器】
        # 设定一个时间间隔，单位是秒。这里设为 2.0 秒
        timer_period = 2.0  
        # 创建一个定时器，每隔 2.0 秒，就会自动去调用一次 self.timer_callback 这个函数
        self.timer = self.create_timer(timer_period, self.timer_callback)

        # 告诉程序，小说文件的名字叫什么
        self.file_path = 'novel.txt'

        # 尝试打开这个文件（防止文件不存在导致程序直接崩溃）
        try:
            # 以 'r' (只读) 模式打开文件，并指定编码为 utf-8（防止中文乱码）
            self.file = open(self.file_path, 'r', encoding='utf-8')
            # 使用 ROS 2 自带的日志打印功能（比 print 更好用，会带上时间戳和节点名）
            self.get_logger().info(f'✅ 成功打开小说文件: {self.file_path}，开始广播...')
        except FileNotFoundError:
            # 如果找不到文件，就报错提醒
            self.get_logger().error(f'❌ 找不到文件: {self.file_path}。请确保文件与运行脚本在同一目录！')
            self.file = None

    # 这是被定时器调用的回调函数（每 2 秒执行一次）
    def timer_callback(self):
        # 如果文件根本没打开（比如一开始就报错了），那直接退出这个函数，啥也不干
        if not self.file:
            return

        # 从文件里读取一行文字
        line = self.file.readline()
        
        # 如果 line 里面有内容（说明没读到文件结尾）
        if line:
            # .strip() 是 Python 的小技巧，用来去掉文字开头和结尾的回车换行符和空格，让文字更干净
            clean_line = line.strip()
            
            # 如果这行文字清理完之后不是空的（跳过空白行）
            if clean_line:
                # 创建一个空的 String 消息包
                msg = String()
                # 把清理好的文字塞进消息包的 data 属性里
                msg.data = clean_line
                # 【核心步骤3：发布消息】把消息包扔到频道里发出去！
                self.publisher_.publish(msg)
                # 在自己的屏幕上打印一下发了什么，方便调试
                self.get_logger().info(f'📡 发送: "{msg.data}"')
        else:
            # 如果 readline() 没读到东西，说明书读完了
            self.get_logger().info('🎉 全文连载结束！')
            # 关闭文件，养成好习惯
            self.file.close()
            # 把自变量设为 None，这样下次定时器再触发时，就会直接 return 退出，不再尝试读取
            self.file = None 

# 整个程序的入口
def main(args=None):
    # 1. 初始化 ROS 2 的底层通信系统
    rclpy.init(args=args)
    # 2. 实例化我们刚才写的那个节点
    novel_publisher = NovelPublisher()
    
    # 3. spin 的意思是“旋转、持续运行”。这行代码会让程序一直卡在这里不死掉，
    # 并且在后台一直监听定时器，直到你按下键盘的 Ctrl+C 强行关闭它
    rclpy.spin(novel_publisher)
    
    # 4. 如果你按了 Ctrl+C，程序走出了 spin，就会执行下面这两步清理后事
    novel_publisher.destroy_node() # 销毁节点
    rclpy.shutdown() # 关闭 ROS 2 通信系统

# Python 脚本的标准写法，确保这个文件被直接运行时才会执行 main() 函数
if __name__ == '__main__':
    main()
```

#### 2. 订阅者代码 (`novel_subscriber.py`) - 收音机

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class NovelSubscriber(Node):
    
    def __init__(self):
        # 给自己起个名叫 'novel_subscriber'
        super().__init__('novel_subscriber')
        
        # 【核心步骤：创建订阅者】
        # 参数1：接收的数据类型是 String
        # 参数2：监听的频道名叫 'novel_channel' (必须和发布者一模一样！)
        # 参数3：一旦听到消息，就立刻去执行 self.listener_callback 这个函数去处理
        # 参数4：10 是消息队列长度
        self.subscription = self.create_subscription(
            String,
            'novel_channel',
            self.listener_callback,
            10)
        
        # 这行代码主要是为了防止 Python 的垃圾回收机制把 subscription 不小心清理掉
        self.subscription  
        
        self.get_logger().info('📻 收音机已打开，正在监听小说频道...')

    # 这是处理消息的回调函数，只要一收到消息，ROS 2 就会自动把消息作为 `msg` 参数传给这个函数
    def listener_callback(self, msg):
        # 把收到的消息里的文字（msg.data）打印到屏幕上
        self.get_logger().info(f'📖 听到: "{msg.data}"')

def main(args=None):
    rclpy.init(args=args)
    novel_subscriber = NovelSubscriber()
    # 同样，使用 spin 让收音机一直保持开机状态，等待接收消息
    rclpy.spin(novel_subscriber)
    
    novel_subscriber.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

### 第二阶段：从零开始运行的完整步骤

这里我们采用**最简单快捷的 Python 直接运行法**（不需要复杂的 `colcon build` 编译，非常适合小白验证代码逻辑）。假设你使用的是装有 ROS 2 的 Ubuntu 系统。

#### 第 1 步：打开终端并准备文件夹
按键盘快捷键 `Ctrl + Alt + T` 打开一个新的命令行终端。

我们要建一个专属的文件夹来放这三个文件：
```bash
# 创建一个名为 ros2_novel_test 的文件夹
mkdir ~/ros2_novel_test

# 进入这个文件夹
cd ~/ros2_novel_test
```

#### 第 2 步：创建三个文件
在终端中，我们用简单的文本编辑器 `nano` 来创建代码文件（如果你习惯用 VS Code，可以直接在 VS Code 里在这个文件夹下建文件，把前面的代码复制进去保存即可）。

1.  **创建小说文件：**
    ```bash
    nano novel.txt
    ```
    把小说内容复制进去（例如《西游记》开头），然后按 `Ctrl+O`（保存），敲回车确认，再按 `Ctrl+X`（退出）。

2.  **创建发布者文件：**
    ```bash
    nano novel_publisher.py
    ```
    把上面带注释的**发布者代码**全部复制进去。保存并退出（`Ctrl+O`, 回车, `Ctrl+X`）。

3.  **创建订阅者文件：**
    ```bash
    nano novel_subscriber.py
    ```
    把上面带注释的**订阅者代码**全部复制进去。保存并退出。

此时，你可以输入 `ls` 命令，应该能看到这三个文件静静地躺在文件夹里。

#### 第 3 步：激活 ROS 2 环境（非常重要！）
每次新打开一个终端，你都必须告诉这个终端：“我要用 ROS 2 啦！”。
*假设你安装的是 ROS 2 Humble 版本（其他版本如 foxy, iron 把 humble 替换掉即可）：*
```bash
source /opt/ros/humble/setup.bash
```

#### 第 4 步：运行“收音机”（订阅者）
在刚才那个终端里，确保你还在 `~/ros2_novel_test` 文件夹下，运行：
```bash
python3 novel_subscriber.py
```
**屏幕会显示：** `📻 收音机已打开，正在监听小说频道...`
（这时候它会卡在这里等消息，不要关掉它）。

#### 第 5 步：运行“广播站”（发布者）
按 `Ctrl + Alt + T` **再打开一个新的终端窗口**。

同样地，进入文件夹并激活环境：
```bash
cd ~/ros2_novel_test
source /opt/ros/humble/setup.bash
```

然后运行发布者：
```bash
python3 novel_publisher.py
```
**屏幕会显示：** `✅ 成功打开小说文件...`，然后每隔 2 秒显示 `📡 发送: "xxx"`。

#### 第 6 步：见证奇迹的时刻！
现在，把你电脑屏幕上的这两个终端窗口**并排放在一起**。
你会看到：
*   **右边终端（发布者）**每隔 2 秒发出一句台词。
*   **左边终端（订阅者）**立刻同步接收到这句台词，并显示 `📖 听到: "xxx"`。

如果想停止它们，在终端里按下 `Ctrl + C` 即可强制退出程序。
```
