第一步：安装中文基础字体
有些 RDK X5 的系统镜像默认缺少中文字体，如果不先安装字体，中文输入法打出来的字可能会变成“方块”或“乱码”。

Bash
sudo apt update
sudo apt install fonts-wqy-microhei fonts-wqy-zenhei -y
第二步：安装 Fcitx 5 框架和拼音输入法
运行以下命令安装 Fcitx 5 的核心组件和智能拼音插件：

Bash
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-config-qt -y
第三步：配置环境变量（关键）
为了让系统各应用软件（尤其是 Qt 和终端）能够完美唤起 Fcitx 5 输入法，需要配置一下环境变量：

运行以下命令打开配置文件：

Bash
sudo nano /etc/environment
在文件末尾添加以下几行内容（注意大小写）：

Plaintext
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
按 Ctrl + O 保存，按 Enter 确认，再按 Ctrl + X 退出。

第四步：设置 Fcitx 5 为默认输入法并重启
在桌面打开终端，输入 im-config。

弹出的图形界面中点击 OK 确认，然后点击 Yes 允许修改。

在选择列表中找到并勾选 fcitx5，点击 OK 保存。

立即重启 RDK X5 让所有配置生效：

Bash
sudo reboot
第五步：添加拼音并使用
重启进入桌面后，在应用程序菜单中找到 Fcitx 5 Configuration（输入法配置）。

<img width="874" height="609" alt="image" src="https://github.com/user-attachments/assets/cde7b39c-8cd8-428b-9fd1-54b55b34c5eb" />

系统检测到你当前是 XFCE 桌面环境。

当前系统默认的自动选择是 ibus（Current automatic choice: 'ibus'）。

由于你系统的语言环境可能设置为了英文（Locale='en_US'），系统没有触发中文环境下的自动切换规则，所以它依然卡在 ibus 上，导致你的 Fcitx 5 无法生效。

接下来怎么操作？
点击右下角的 OK 按钮。

随后系统会弹出一个新的窗口，询问你：“Do you want to explicitly select the configuration?”（你想要显式选择配置吗？）

毫不犹豫地选择 Yes。

紧接着会弹出一个输入法列表（包含 ibus、fcitx5 等选项）。在列表中找到并勾选 fcitx5，然后点击 OK。

配置完成后，为了让这个修改在整个 XFCE 图形界面彻底生效，请在终端执行重启：
