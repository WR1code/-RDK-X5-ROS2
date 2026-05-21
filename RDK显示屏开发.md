2 Display Options > D1 Dsiplay Choice > 2 HDMI可以修改

可以使用srpi-config工具来切换回HDMI输出

默认分辨率可以通过修改/etc/X11/xorg.conf.d/1-resolution.conf文件来实现：

Section "Screen"
    Identifier "Screen0"
    Device "Device0"
    Monitor "Monitor0"
    DefaultDepth 24
    SubSection "Display"
        Depth 24
        Modes "1280x720"
    EndSubSection
EndSection

<img width="1356" height="806" alt="image" src="https://github.com/user-attachments/assets/bf9374a0-7b3b-4fc4-94ec-5c5aa4c4b8c1" />
使用srpi-config工具来选择2.8inch DSI LCD，重启后生效。

2 Display Options > D3 MIPI LCD Choice > 2.8inch DSI LCD

https://developer.d-robotics.cc/rdk_doc/Quick_start/display_use/display_rdkx5
