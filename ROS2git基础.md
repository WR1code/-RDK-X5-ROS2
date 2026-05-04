在ROS 2开发中，Git不仅是代码备份工具，更是管理复杂机器人项目（涉及多个节点、传感器驱动和算法模块）的绝对核心。特别是在使用 `colcon` 编译多个功能包时，良好的版本控制习惯能帮你省去无数的麻烦。

这是一份专为ROS 2开发者定制的Git基础与进阶指令教程。

---

### 1. 初始配置 (Git Config)
在你的Ubuntu终端中，首先需要告诉Git你是谁。这会在你每次提交代码时留下记录。

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"
```
*提示：使用 `--global` 意味着这台电脑上的所有Git仓库都会默认使用这个身份。*

### 2. ROS 2 专属避坑指南：`.gitignore`
**这是最重要的一步！** 在ROS 2工作空间中，你绝对不能把编译生成的文件提交到Git仓库中，否则仓库会变得无比臃肿，且在其他电脑上拉取后会导致编译冲突。

在你的代码仓库根目录（通常是某个 package 的目录，或者如果你把整个 `src` 作为仓库的话，就是 `src` 目录）下，创建一个 `.gitignore` 文件，至少包含以下内容：

```text
# ROS 2 编译生成目录 (如果在工作空间根目录建仓)
build/
install/
log/

# Python 缓存
__pycache__/
*.pyc

# C/C++ 编译中间文件
*.o
*.so
*.a

# IDE 配置文件 (如 VSCode, CLion)
.vscode/
.idea/
```

### 3. 本地核心工作流 (Add & Commit)
假设你刚用 `ros2 pkg create` 创建了一个新的功能包，或者修改了某个C++或Python节点。

*   **初始化仓库 (如果还没创建的话):**
    ```bash
    cd ~/ros2_ws/src/your_package_name
    git init
    
```
*   **查看当前状态 (高频指令):**
    用于检查你修改了哪些文件，比如是否动了 `CMakeLists.txt` 或 `package.xml`。
    ```bash
    git status
    ```
*   **将修改添加到暂存区:**
    ```bash
    git add .                  # 添加当前目录下的所有更改 (最常用)
    git add src/my_node.cpp    # 仅添加特定文件
    ```
*   **提交更改到本地历史:**
    提交信息要清晰，说明你做了什么修改（例如：修复了YOLO检测节点的内存泄漏）。
    ```bash
    git commit -m "feat: 新增了图像发布的ROS 2节点"
    ```

### 4. 远程仓库操作 (Clone, Push, Pull)
在团队协作（或在树莓派/RDK等边缘计算板上部署代码）时，你需要与远程仓库（如 GitHub, Gitee, GitLab）交互。

*   **克隆现有的ROS 2包:**
    **注意：** 通常你应该把代码克隆到工作空间的 `src` 目录下，然后再回到工作空间根目录进行 `colcon build`。
    ```bash
    cd ~/ros2_ws/src
    git clone https://github.com/user/awesome_ros2_package.git
    ```
*   **将本地代码推送到远程服务器:**
    ```bash
    git push origin main  # 将代码推送到主分支 (旧版Git可能叫 master)
    ```
*   **从远程拉取最新代码:**
    在团队成员更新了导航算法或模型文件后，你需要同步。
    ```bash
    git pull origin main
    ```

### 5. 分支管理 (Branching)
在机器人开发中，千万不要直接在 `main` 分支上测试不稳定的新算法（比如正在调参的导航代价地图配置）。新建一个分支进行开发是最佳实践。

*   **创建并切换到新分支:**
    ```bash
    git checkout -b test_new_yolo_model
    ```
*   **查看所有分支:**
    ```bash
    git branch
    ```
*   **合并分支:**
    当新模型在本地通过了ROS 2的测试，表现稳定后，切回主分支并将其合并。
    ```bash
    git checkout main
    git merge test_new_yolo_model
    ```

### 6. 🌟 ROS 2 进阶技巧：多仓库管理 (`vcstool`)
在复杂的ROS 2项目中，你的 `src` 目录下往往包含十几个甚至几十个来自不同作者的Git仓库（比如底盘驱动、雷达驱动、Nav2、自定义节点等）。使用原生的Git指令一个一个 `clone` 会非常低效。

ROS 2 社区的标准做法是使用 `vcstool`：

1.  把所有依赖的仓库链接写进一个 `xxx.repos` 文件中。
2.  使用一条指令自动拉取所有Git仓库：
    ```bash
    vcs import src < my_project.repos
    ```
3.  一键查看 `src` 下所有Git仓库的状态：
    ```bash
    vcs custom --args status
    ```
*(如果你的电脑上还没有这个工具，可以通过 `sudo apt install python3-vcstool` 安装)*

**开发建议：** 在写代码前，先 `git pull` 确保代码是最新的；在跑真机测试前，确保当前的修改已经 `git commit`，这样一旦机器狗/小车跑出问题或者代码崩溃，你可以随时回退到上一个安全版本。
```
