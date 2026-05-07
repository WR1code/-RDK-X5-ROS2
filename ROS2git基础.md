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




在 ROS 2 开发中，由于往往需要同时管理多个功能包（Packages）和依赖项，项目结构通常是一个包含多个独立 Git 仓库的**工作空间（Workspace）**。因此，纯粹依靠基础的 `git clone`、`git commit` 往往不够用。

以下是 ROS 2 开发中 Git 的详细进阶用法，涵盖了多仓库管理、分支策略、排错溯源以及社区规范。

---

### 1. 核心进阶：多仓库管理神器 `vcstool`

在 ROS 2 中，最进阶的 Git 用法其实是**不要单独使用 Git**，而是结合官方推荐的 `vcstool` 工具。它可以让你用一条指令同时对工作空间下的数十个 Git 仓库进行批量操作。

**安装 vcstool：**
```bash
sudo apt install python3-vcstool
```

#### A. 导出当前工作空间的状态
当你配置好了一个可以完美编译的 ROS 2 工作空间，你需要记录下所有仓库的当前状态（甚至是具体的 commit Hash），以便团队成员复现。
```bash
# 进入工作空间的 src 目录
cd ~/ros2_ws/src

# 导出所有仓库的 URL 和版本信息到 .repos 文件
vcs export --type repos > my_workspace.repos

# 如果想要精确到当前的 commit hash（锁定版本，防止上游更新导致编译失败）
vcs export --exact --type repos > my_workspace_exact.repos
```

#### B. 一键拉取/恢复多仓库
新同事拿到你的 `.repos` 文件后，只需要一行命令就能克隆所有依赖：
```bash
mkdir -p ~/ros2_ws/src && cd ~/ros2_ws
vcs import src < src/my_workspace.repos
```

#### C. 批量 Git 操作
`vcs` 可以将 Git 命令广播到所有的子目录中：
```bash
# 检查所有仓库的状态（看哪个包有未提交的代码）
vcs custom --args status

# 批量拉取所有仓库的最新代码
vcs pull

# 将所有仓库切换到 humble 分支（如果存在的话）
vcs custom --args checkout humble
```

---

### 2. ROS 2 特色的分支管理策略

ROS 2 的版本迭代非常快（如 Foxy, Humble, Iron, Rolling），因此在编写开源包或公司内部组件时，Git 分支的管理有特定的约定。

*   **发行版分支 (Distribution Branches):** 通常你的仓库应该包含以 ROS 2 发行版命名的分支。例如 `humble` 分支只支持 ROS 2 Humble，`rolling` 分支存放最新特性。
*   **Backport（向后移植）:** 如果你在 `rolling` 分支修复了一个 bug，需要同步到 `humble`，此时要用到 `cherry-pick`。

**实战用法（Bug 向后移植）：**
```bash
# 假设你在 rolling 分支提交了修复，commit hash 为 a1b2c3d
git checkout humble
# 将该修复应用到 humble 分支
git cherry-pick a1b2c3d
# 如果有冲突，解决冲突后运行：
git cherry-pick --continue
```

---

### 3. 调试与排错：使用 `git bisect` 抓虫

在机器人开发中，经常会遇到“昨天机器狗还能跑，今天拉取了最新代码突然报 Segfault 崩溃了”的情况。如果期间有几十个提交，手动排查会非常痛苦。`git bisect` 是通过二分查找定位错误 Commit 的神器。

**实战用法：**
```bash
cd ~/ros2_ws/src/broken_package

# 开始二分查找
git bisect start

# 标记当前版本是有问题的 (Bad)
git bisect bad

# 告诉 Git 昨天那个正常工作的 commit hash 是没问题的 (Good)
git bisect good e4f5g6h

# 此时 Git 会自动切换到中间的一个 commit，你需要重新编译并测试节点
colcon build --packages-select broken_package
source install/setup.bash
ros2 run broken_package some_node

# 测试后告诉 Git 结果：
# 如果测试通过，运行：git bisect good
# 如果依然崩溃，运行：git bisect bad

# Git 会继续二分，直到精准指出是哪一行代码引入了 Bug。
# 结束排查，回到主分支
git bisect reset
```

---

### 4. 保持历史整洁：交互式 Rebase (`git rebase -i`)

ROS 2 官方社区（以及大多数正规机器人公司）对 Pull Request (PR) 的要求非常严格。他们不喜欢看到 `Update typo`, `Fix bug again`, `Try again` 这样琐碎的提交历史。在提交 PR 前，你需要使用 `rebase` 压缩你的提交。

**实战用法：**
假设你最近做了 4 个提交，但你想把它们合并成一个整洁的提交：
```bash
# 交互式变基最近的 4 个提交
git rebase -i HEAD~4
```
在弹出的编辑器中，将除了第一个 `pick` 之外的都改为 `squash` 或 `s`：
```text
pick 1a2b3c4 Add new sensor fusion algorithm
s 5d6e7f8 Fix typo in variable name
s 9g0h1i2 Fix compile error
s 3j4k5l6 Tune parameters for testing
```
保存并退出后，Git 会让你重新编写合并后的 Commit Message。这样你推送到 GitHub 的历史就会非常干净。

---

### 5. Git Submodule vs vcstool

有时你会看到一些 ROS 项目使用 `git submodule` 来管理依赖，但在 ROS 2 中，**强烈建议优先使用 `vcstool` 而不是 Submodule**。

*   **为什么不用 Submodule？**
    *   Submodule 会将依赖包硬编码绑定到特定的 commit。
    *   在 ROS 2 工作空间中，你通常需要跨多个包进行修改并同时测试，Submodule 的 "Detached HEAD" 状态会让开发变得非常繁琐。
*   **什么时候可以用 Submodule？**
    *   当你引入的是一个**非 ROS 的第三方 C++ 纯依赖库**（例如某个传感器厂商提供的纯驱动库），并且你不打算修改它时，可以将其作为 Submodule 放在你的 package 目录下，用 CMake 直接引入。

---

### 总结建议工作流

一个成熟的 ROS 2 开发者日常 Git 工作流通常是这样的：
1. 使用 `vcs import` 拉取整个工作环境。
2. 在特定的 package 内切出 `feature` 分支进行开发。
3. 如果上游库有更新，使用 `vcs pull` 同步整个工作空间。
4. 遇到诡异 Bug 使用 `git bisect` 快速定位。
5. 开发完成后，使用 `git rebase -i` 整理提交历史。
6. 使用 `vcs export` 更新 `.repos` 文件，锁定依赖版本。
