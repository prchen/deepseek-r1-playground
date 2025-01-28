# 浅玩 DeepSeek R1

> 本仓库用于留存游玩火爆全球的 DeepSeek R1 模型过程中产生的笔记和各种小工（wan）具（ju）

# Windows 环境部署

## 依赖栈

后面画个图吧没偷懒的话，文字版：

* DeepSeek R1（应用层）
* Python3 & CUDA & cuDNN & PyTorch（运行时层）
* Docker（容器化层）
* CUDA & Nvidia Container Toolkit（容器依赖层）
* WSL - Ubuntu（虚拟化层）
* Windows（系统层）
* Nvidia 显卡（硬件层）

## 环境安装

**检查 WSL 发行版**

略，大致就是：

* 检查是不是 Ubuntu 不是的话装上并设置为 wsl 默认发行版
* 如果 Docker Desktop 已经基于奇怪的发行版运行需要卸掉 Docker 重装

**在 WSL 中安装 CUDA 和 Container Toolkit**

参考（需要注意文档中所有命令需要运行在 wsl 环境内而非 powershell）：

* [CUDA 安装官方文档](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=WSL-Ubuntu&target_version=2.0&target_type=deb_local)
* [Container Toolkit 安装官方文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

**验证 CUDA 版本**

有两个命令，需要记录下来它们输出的较低 CUDA 版本号是多少，后面要用。

```bash
# 命令1
nvidia-smi
# 命令2
/usr/local/cuda/bin/nvcc --version
```

**拉取 CUDA + cuDNN 镜像**

这里需要找到合适的版本，上述步骤中我的环境中两处 CUDA 版本一个为 12.7 一个为 12.8，拉镜像时需要使用版本 <= 12.7 的 tag。

全部 tag 参考：[官方镜像](https://hub.docker.com/r/nvidia/cuda/tags?name=ubuntu)

```bash
docker pull nvidia/cuda:12.6.3-cudnn-devel-ubuntu24.04
```

**验证 CUDA + cuDNN 容器环境**

运行以下命令，没报错的话就说明 CUDA + cuDNN 环境准备好了。

```bash
docker run --rm --gpus all -it nvidia/cuda:12.6.3-cudnn-devel-ubuntu24.04 nvidia-smi
```

未完待续
