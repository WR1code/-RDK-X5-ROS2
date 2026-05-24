docker pull openexplorer/ai_toolchain_ubuntu_20_x5_gpu:v1.2.8

docker run -it --rm \
  --gpus all \
  --shm-size=15g \
  -v /你的/本地/OE包实际路径:/open_explorer \
  -v /你的/本地/数据集实际路径:/data/horizon_x5/data \
  openexplorer/ai_toolchain_ubuntu_20_x5_gpu:v1.2.8
