wget http://fishros.com/install -O fishros && bash fishros

ros2 run turtlesim turtlesim_node

ros2 run turtlesim turtle_teleop_key

根据您上传的图片，您已经掌握了一些最基础的文件操作和编辑命令（如 `cd`, `mkdir`, `nano`, `cat`, `rm` 等）。

为了帮助您更高效地使用 Ubuntu 系统，我为您整理了一份日常开发和运维中最常用的指令清单，并按功能进行了分类：

### 1. 文件和目录操作

这是日常使用中最频繁的指令，用于管理系统中的文件和文件夹。

| 指令 | 作用 | 常用参数与示例 |
| :--- | :--- | :--- |
| **`pwd`** | 显示当前工作路径 | `pwd` (打印完整路径) |
| **`ls`** | 列出目录下的内容 | `ls -l` (显示详细信息), `ls -a` (显示包含隐藏文件的所有内容) |
| **`cd`** | 切换当前目录 | `cd ~` (进入用户主目录), `cd ..` (返回上一级目录) |
| **`mkdir`** | 创建新文件夹 | `mkdir project` (创建名为 project 的文件夹) |
| **`touch`** | 创建空白文件 | `touch index.html` (创建或更新该文件的时间戳) |
| **`rm`** | 删除文件或目录 | `rm file.txt` (删除文件), `rm -r folder` (递归删除整个文件夹) |
| **`cp`** | 复制文件或目录 | `cp file.txt copy.txt` (复制文件), `cp -r dir1 dir2` (复制文件夹) |
| **`mv`** | 移动文件或重命名 | `mv old.txt new.txt` (重命名), `mv file.txt /home/user/` (移动文件) |

---

### 2. 文件查看与编辑

当您不需要打开重量级的图形化编辑器时，可以在终端内直接查看和修改代码或配置文件。

| 指令 | 作用 | 常用参数与示例 |
| :--- | :--- | :--- |
| **`cat`** | 查看文件全部内容 | `cat hello.txt` (直接在终端输出文件内容) |
| **`less`** | 分页查看长文件 | `less syslog.log` (按空格键翻页，按 `q` 退出) |
| **`grep`** | 在文件中搜索文本 | `grep "error" log.txt` (在 log.txt 中查找包含 error 的行) |
| **`nano`** | 简单易用的文本编辑器 | `nano config.ini` (图片中展示的编辑器，适合新手) |
| **`vim`** | 功能强大的文本编辑器 | `vim script.sh` (需学习快捷键，但效率极高) |
还可以用gedit来图形化修改文本文件，长相酷似文本编辑器

---

### 3. 软件与包管理 (APT)

Ubuntu 使用 `apt` 包管理器来安装和卸载软件。执行这些命令通常需要管理员权限（在命令前加 `sudo`）。

| 指令 | 作用 | 常用参数与示例 |
| :--- | :--- | :--- |
| **`sudo apt update`** | 更新软件源列表 | 每次安装新软件前建议先执行，获取最新软件版本信息 |
| **`sudo apt upgrade`** | 升级已安装的软件 | 将系统中所有软件升级到最新版本 |
| **`sudo apt install`** | 安装新软件 | `sudo apt install python3` (安装 Python 3) |
| **`sudo apt remove`** | 卸载软件 | `sudo apt remove vim` (卸载 vim 软件) |

---

### 4. 系统状态与监控

用于查看计算机的资源占用情况。

| 指令 | 作用 | 常用参数与示例 |
| :--- | :--- | :--- |
| **`top`** / **`htop`** | 实时监控系统进程和资源 | `top` (按 `q` 退出，建议安装界面更友好的 `htop`) |
| **`df`** | 查看磁盘空间使用情况 | `df -h` (以 GB/MB 等易读格式显示磁盘余量) |
| **`free`** | 查看内存使用情况 | `free -m` (以 MB 为单位显示内存和交换空间) |
| **`ps`** | 查看当前运行的进程 | `ps aux` (列出系统中所有的进程状态) |
| **`kill`** | 强制结束进程 | `kill 1234` (结束 PID 为 1234 的进程) |

---

### 5. 网络与权限管理

| 指令 | 作用 | 常用参数与示例 |
| :--- | :--- | :--- |
| **`ping`** | 测试网络连通性 | `ping google.com` (按 `Ctrl+C` 停止) |
| **`ip a`** | 查看本机 IP 地址和网卡 | 取代了旧版的 `ifconfig` 指令 |
| **`sudo`** | 以管理员身份执行命令 | `sudo [任何命令]` (需要输入当前用户的密码) |
| **`chmod`** | 修改文件或目录的权限 | `chmod +x script.sh` (赋予文件可执行权限) |

whereis查询文件地址

> **学习小技巧：** 如果遇到不认识的指令，或者不知道某个指令怎么用，可以在终端输入 `man [指令名]` (例如 `man ls`) 或者 `[指令名] --help` 来查看官方的详细使用说明书。

某个指令加上--help可以查询提示/n

开始用vscode编程之前，先用mdkir catkin_ws创造工作空间，然后mdkir src作为主目录

<img width="1362" height="617" alt="image" src="https://github.com/user-attachments/assets/ddb8647b-ae47-4156-8a92-0c49e99a3347" />

# 🐧 Ubuntu 22.04 `echo` 命令实战指南

在 Ubuntu (默认 Bash 终端) 中，`echo` 既可以用来做简单的屏幕打印，也是编写 Shell 脚本时进行文件操作和变量调试的神器。

## 📊 核心用法速查表

| 功能场景 | 命令示例 | 运行结果 / 作用说明 |
| :--- | :--- | :--- |
| **基础打印** | `echo "Hello Ubuntu"` | 在屏幕上输出 `Hello Ubuntu` 并自动换行。 |
| **输出变量** | `echo $USER` | 提取并输出系统的环境变量（如当前登录的用户名）。 |
| **取消换行** | `echo -n "Loading..."` | 输出内容，但结尾 **不会自动换行**。 |
| **开启转义** | `echo -e "Line1\nLine2"` | `-e` 允许解析特殊字符，`\n` 代表换行，分两行打印。 |
| **插入制表符**| `echo -e "Name\tAge"` | `\t` 代表插入一个 Tab（制表符）空格，常用于对齐。 |
| **覆盖写入** | `echo "123" > config.txt`| 创建并写入 `123`；**警告：原文件内容会被清空覆盖。** |
| **追加写入** | `echo "456" >> config.txt`| 将 `456` 追加写入到文件的 **最末尾**，保留原有内容。 |
| **组合命令** | `echo "Time: $(date)"` | 利用 `$()` 执行内部命令，将时间拼接到 `echo` 中输出。 |

---

## 💻 终端演示代码块

你可以直接在终端中复制运行以下代码来查看效果：

### 1. 基础打印与变量
```bash
# 打印普通文本
echo "欢迎使用 Ubuntu 22.04！"

# 查看当前用户的家目录路径
echo "我的家目录是: $HOME"
```

### 2. 文件操作 (重定向)
```bash
# 将一段配置写入文件（如果 config.conf 不存在会自动创建）
echo "PORT=8080" > config.conf

# 继续向该文件追加内容，而不破坏上面的 PORT 变量
echo "HOST=127.0.0.1" >> config.conf

# 查看刚才写入的内容
cat config.conf
```

### 3. 高阶玩法：输出彩色提示文字
在编写脚本时，彩色文字能让日志更清晰：
```bash
# 打印红色报错信息
echo -e "\e[31m[ERROR] 配置文件加载失败！\e[0m"

# 打印绿色成功信息
echo -e "\e[32m[SUCCESS] 服务启动完成！\e[0m"

# 打印黄色警告信息
echo -e "\e[33m[WARNING] 磁盘空间不足！\e[0m"
```

---

## 💡 避坑指南：单引号 vs 双引号

在 Shell 脚本中，包裹文字的符号决定了 `echo` 会如何处理变量：

* **双引号 `"`（解析变量，推荐）：**
  ```bash
  echo "Hello, $USER"
  # 输出: Hello, root (假设当前是 root 用户)
  ```

* **单引号 `'`（强输出，原样打印）：**
  ```bash
  echo 'Hello, $USER'
  # 输出: Hello, $USER (变量不会被解析)
  ```
：printenv | grep AMENT
这其实是两个命令通过一个“管道”组合在一起使用的：

printenv：意思是“打印环境变量 (Print Environment)”。如果单独运行它，它会把系统当前所有的环境变量全部列出来，通常会有眼花缭乱的好几十行。

| (管道符)：它的作用是把左边命令（printenv）输出的那一大堆结果，直接当成输入材料“喂”给右边的命令。

grep AMENT：grep 是 Linux 里非常强大的文本搜索工具。它的作用是从刚刚传过来的一堆文本里，专门揪出包含 AMENT 这个关键词的文本行。


.开头的一般是隐藏的文件，例如gedit ~/.bashrc，在这个文件里面可以配置source环境变量

AMENT_PREFIX_PATH 是 ROS 2 (Robot Operating System 2) 中的一个核心环境变量。

它的主要作用是告诉 ROS 2 系统及其构建工具（如 colcon）去哪里寻找已经安装的 ROS 2 软件包（packages）。
