docker pull openexplorer/ai_toolchain_ubuntu_20_x5_gpu:v1.2.8

docker run -it --rm \
  --gpus all \
  --shm-size=15g \
  -v /你的/本地/OE包实际路径:/open_explorer \
  -v /你的/本地/数据集实际路径:/data/horizon_x5/data \
  openexplorer/ai_toolchain_ubuntu_20_x5_gpu:v1.2.8

https://developer.d-robotics.cc/rdk_doc/Application_case/line_follower?_highlight=open&_highlight=exploer#d-robotics-rdk%E5%B9%B3%E5%8F%B0

算法工具链OE包获取方式 wget -c ftp://vrftp.horizon.ai/Open_Explorer_gcc_9.3.0/2.3.3/horizon_xj3_open_explorer_v2.3.3_20220727.tar.gz
