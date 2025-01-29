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

**准备临时容器**

接下来先运行起来一个临时容器在其中安装 PyTorch 环境。

```bash
# 启动一个不会退出的 daemon 容器
docker run --name cuda-devel --restart always --gpus all -dt nvidia/cuda:12.6.3-cudnn-devel-ubuntu24.04
# 登入到容器 shell
docker exec -it cuda-devel /bin/bash
```

在容器中安装运行时依赖，官方文档说是 DeepSeek R1 参考 DeepSeek V3 的步骤，先试着来不过步骤和官网有一定差异。

```bash
# 通过包管理安装一些依赖
apt install -y git python3-full python3-pip

# 准备 venv 环境
mkdir -p /opt/DeepSeek-R1-Playground
python3 -m venv /opt/DeepSeek-R1-Playground/.venv

# 将 DeepSeek-V3 仓库克隆到 /opt 下
git -C /opt clone https://github.com/deepseek-ai/DeepSeek-V3.git

# 安装 DeepSeek-V3 的依赖
/opt/DeepSeek-R1-Playground/.venv/bin/pip3 install -r /opt/DeepSeek-V3/inference/requirements.txt
```

**下载模型文件**

由于模型非常非常大在容器内下载的话后续不方便重复使用，需要退出容器环境回到 wsl 中进行下载。

```bash
# 安装 git lfs
sudo apt install git-lfs
git lfs install

# 克隆这里需要先添加 ssh 公钥到自己的 hugging face 账户下并克隆到一个记得住的位置，回头整理笔记时再加文档，然后文件发现下不下来得分头行动也后面再补充文档吧
GIT_LFS_SKIP_SMUDGE=1 git clone git@hf.co:deepseek-ai/DeepSeek-V3
```

然后去 Hugging Face 手动把模型的数据文件下下来，有 163 个 4G 左右的切片，得下好久：[模型仓库](https://huggingface.co/deepseek-ai/DeepSeek-R1/tree/main)

未完待续
