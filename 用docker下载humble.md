# 1. 添加 NVIDIA Container Toolkit 的官方软件源
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# 2. 更新软件源列表
sudo apt update

# 3. 这时候就能成功安装了
sudo apt install -y nvidia-container-toolkit

# 1. 自动配置 Docker 运行时以支持 NVIDIA 显卡
sudo nvidia-container-toolkit runtime configure --runtime=docker

# 2. 解决你遇到的 Warning 提示，重载系统服务配置文件
sudo systemctl daemon-reload

# 3. 重启 Docker 使所有配置生效
sudo systemctl restart docker

到这一步，显卡直通环境就彻底建好了。在启动大体积的 ROS 容器前，我们可以用一个非常小的测试镜像来验证显卡是否握手成功：

Bash
docker run --rm --gpus all ubuntu:22.04 nvidia-smi
🎉 成功的标志： 如果终端里打印出了和你宿主机一模一样的 NVIDIA 显卡表格（能看到你的 RTX 显卡型号），说明显卡直通彻底成功！

第二步：启动适配 OriginBot 的 Docker 容器
我们可以直接使用 OSRF（官方）提供的带有桌面和 Gazebo 的 Humble 镜像。


  sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
EOF



sudo systemctl daemon-reload
sudo systemctl restart docker



# 1. 清理可能残留的重名容器
docker rm -f originbot_env 2>/dev/null

# 2. 再次运行容器
docker run -it \
  --name originbot_env \
  --net=host \
  --ipc=host \
  --privileged \
  --runtime=nvidia \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v /home/w/dev_ws:/workspace/dev_ws \
  osrf/ros:humble-desktop-full
