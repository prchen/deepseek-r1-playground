# 初探 DeepSeek R1

> 本仓库用于留存游玩火爆全球的 DeepSeek R1 模型过程中产生的笔记和各种小工（wan）具（ju）

**游玩环境**

|环境||
|---|---|
|WSL| WSL2 / Ubuntu-24.04|
|系统|Windows 11|
|显卡|Nvidia RTX-4070 Ti Super 16G|

**依赖栈**

|层次||
|---|---|
|应用层|SGLang & DeepSeek R1|
|运行时层|CUDA & cuDNN & Python3 & PyTorch|
|容器层|Docker|
|系统依赖层|CUDA & Nvidia Container Toolkit|
|虚拟化层|WSL2|
|系统层|Windows|
|硬件层|Nvidia 显卡|

**网络联通性**

由于在大陆地区存在某些网络不稳定的现象，实验步骤中任何涉及下载文件的场景都有可能出现网络卡死，此时相信能在 Github 上看到原版的这篇笔记的你一定有办法解决这些网络不稳定问题。

> WSL 环境的网络联通性问题可以被 Windows 系统代理解决，在系统中开启全局系统代理后通过 `wsl --shutdown` 迫使虚拟机停机后重新进入 wsl 环境即可让所有网络请求都用上 Windows 中的系统代理。

## 环境准备与调试

### 检查 WSL 和 Docker 环境

* Powershell 执行 `wsl -l -v` 列出发行版检查默认版本是否为 Ubuntu 以及 WSL 版本是否为「2」。
  * 如果命令直接报错了可能是系统尚未打开 WSL 供能，需要自行搜索如何开启 WSL2 并安装 Ubuntu 发行版。
  * 如果发行版或 WSL 版本不对，则需要自行搜索如何安装 WSL2 的 Ubuntu 发行版并将其设置为 WSL 默认使用的发行版。
  * 如果 Docker Desktop 已经被安装在其他发行版则需要将 Docker Desktop 卸载后重新安装使其运行在上述发行版中。
* Powsershell 执行 `wsl docker ps` 检查 docker 运行状态，
  * 如果报错 docker 命令找不到，则说明默认 wsl 环境中未安装 docker 或 docker 正运行在其他环境，需要安装（或卸载后重新安装）Docker Desktop

### 在 WSL 中安装 Nvidia 开发环境

执行官方安装步骤（以下命令均需要在 WSL 环境内执行）：

```bash
# 安装 CUDA Toolkit（拷贝自官方文档）
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-wsl-ubuntu.pin
sudo mv cuda-wsl-ubuntu.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.8.0/local_installers/cuda-repo-wsl-ubuntu-12-8-local_12.8.0-1_amd64.deb
sudo dpkg -i cuda-repo-wsl-ubuntu-12-8-local_12.8.0-1_amd64.deb
sudo cp /var/cuda-repo-wsl-ubuntu-12-8-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-8

# 安装 Container Toolkit（拷贝自官方文档）
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
```

更多信息可参考官方文档：

* [CUDA](https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=WSL-Ubuntu&target_version=2.0&target_type=deb_local)
* [Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

安装完成后检查一下 CUDA 兼容版本，有两个命令，需要记录下来它们输出的较低的 CUDA 版本（以下命令均需要在 WSL 环境内执行）：

```bash
# 查看硬件 CUDA 版本
nvidia-smi

# 检查软件 CUDA 版本
/usr/local/cuda/bin/nvcc --version
```

### 下载模型

由于本地现存有限，先尝试部署最小的 1.5B 版本模型：[Qwen-1.5B](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B/tree/main)，
其他版本的下载链接可参考：[DeepSeek R1 官方文档](https://github.com/deepseek-ai/DeepSeek-R1/blob/main/README.md)

由于模型文件普遍都是好几个 G 大小，推荐将大文件和小文件分开下载，先用 git 下载小文件（以下命令均需要在 WSL 环境内执行）：

> 如果 git clone 报了权限问题说明是 WSL 环境下的 git 所使用的 SSH Key 未添加到 Hugging Face 账户（如未注册过需要先注册），需要添加一下或想办法让环境内的 git 使用你添加过的密钥。

```bash
apt-get install -y git-lfs
git lfs install
GIT_LFS_SKIP_SMUDGE=1 git clone git@hf.co:deepseek-ai/DeepSeek-R1-Distill-Qwen-1.5B /opt/DeepSeek-R1-Distill-Qwen-1.5B
```

然后手动下载大文件（Hugging Face 文件列表中对大文件会附有 LFS 图标），然后覆盖掉仓库中的同名文件（以下命令均需要在 WSL 环境内执行）：

```bash
cp -f /path/to/windows/download/dir/mnt/point/*.safetensors /opt/DeepSeek-R1-Distill-Qwen-1.5B/
```

### 准备临时容器环境

首先要拉下来一个 Nvidia 官方的 CUDA + cuDNN 开发环境的镜像并在此基础上搭建实验环境。

[官方镜像列表](https://hub.docker.com/r/nvidia/cuda/tags?name=ubuntu)中找到合适的 tag 后需要将其拉到本地（以我的环境为例，以下命令均需要在 WSL 环境内执行）：

> Tag 需要带有 cudnn-devel 的标识，代表该镜像是包含了 cuDNN 的开发环境镜像。
>
> Tag 版本号需要考虑兼容性，比如上述步骤中我的环境中两处 CUDA 版本一个为 12.7 一个为 12.8，拉镜像时需要使用版本 <= 12.7 的 tag。

```bash
docker pull nvidia/cuda:12.6.3-cudnn-devel-ubuntu24.04
```

完成拉取后，需要检查一下容器内是否能正确检测到显卡信息，命令输出了显卡信息的话说明一切正常（以下命令均需要在 WSL 环境内执行）：

```bash
docker run --rm --gpus all -it nvidia/cuda:12.6.3-cudnn-devel-ubuntu24.04 nvidia-smi
```

接下来便可以启动临时容器并在其内部安装后续的依赖（以下命令需要在 WSL 环境内执行）：

```bash
# 启动一个不会退出的 daemon 容器，开放 30000 端口是为了后续访问 SGLang 服务
docker run --name cuda-devel --restart always --gpus all -p 30000:30000 -dt nvidia/cuda:12.6.3-cudnn-devel-ubuntu24.04
```

至此我们便启动好了一个临时容器，相当于可以随便瞎玩的虚拟机，该容器可通过下面命令登入（以下命令需要在 WSL 环境内执行）：

```bash
docker exec -it cuda-devel /bin/bash
```

在容器中安装运行时依赖和 SGLang（以下命令需要在登入容器后执行）：

> 如果忘记登入容器就执行，倒也不会有太严重的后果，但原则上来讲即便是使用了 venv，应用层的依赖也应该尽可能只在容器内安装。
> 
> 更多 SGLang 安装信息可参考：[官方文档](https://docs.sglang.ai/start/install.html)

```bash
# 通过包管理安装一些依赖
apt-get install -y python3-full python3-pip

# 准备 venv 环境
mkdir -p /opt/DeepSeek-R1-Playground
python3 -m venv /opt/DeepSeek-R1-Playground/.venv

# 安装 SGLang
/opt/DeepSeek-R1-Playground/.venv/bin/pip3 install sgl-kernel --force-reinstall --no-deps
/opt/DeepSeek-R1-Playground/.venv/bin/pip3 install "sglang[all]" --find-links https://flashinfer.ai/whl/cu124/torch2.4/flashinfer/
```

### 原模启动！

最后只需要将模型添加到容器内并启动 SGLang 服务，临时实验环境的搭建就大功告成了，将下载好的模型添加到容器（以下命令需要在 WSL 环境内执行）：

```bash
docker cp /opt/DeepSeek-R1-Distill-Qwen-1.5B cuda-devel:/opt/DeepSeek-R1-Playground/DeepSeek-R1-Distill-Qwen-1.5B
```

启动 SGLang 服务（以下命令需要在登入容器后执行）：

```bash
cd /opt/DeepSeek-R1-Playground
CUDA_LAUNCH_BLOCKING=1 .venv/bin/python3 -m sglang.launch_server --model ./DeepSeek-R1-Distill-Qwen-1.5B --trust-remote-code --tp 1
```

然后就可以通过 30000 端口访问服务了，比如通过 Postman 或 curl（cygwin / git bash / wsl 环境均可）访问：

```bash
curl -X POST localhost:30000/generate -H 'Content-Type: application/json; charset=utf8' --data-binary '{"text":"爸爸的爸爸叫什么？"}'
```

以下是几次输出（很显然 1.5B 的模型不说初具人形也能说是外星来客）：

```json
{
    "text": "？？期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期个月的期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期期",
    "meta_info": {
        "id": "7acfb3e2c4654b0789709c1134ed2fd7",
        "finish_reason": {
            "type": "length",
            "length": 128
        },
        "prompt_tokens": 7,
        "completion_tokens": 128,
        "cached_tokens": 6
    }
}
```

```json
{
    "text": "（不要等答案）\n\n speech\nAlright, let's try to figure this out. The question is, \"Papa's father's name is what?\" or maybe \"Papa's father's last name is what?\" Hmm, I'm not sure. Wait, maybe it's a typo and they meant \"Papa's father\" as the predecessor? I think the original question might have been \"What is Papa's father's name?\" These are common questions.\n\nWell, assuming it's just asking for the predecessors' last names, it's quite straightforward. The longest last names in English are O'Connor and McDonald. So, that would",
    "meta_info": {
        "id": "3c4549e242a540f5983f9e22f273b372",
        "finish_reason": {
            "type": "length",
            "length": 128
        },
        "prompt_tokens": 7,
        "completion_tokens": 128,
        "cached_tokens": 6
    }
}
```

```json
{
    "text": "爸爸的爸爸叫什么？ )\nTo solve this problem, I am thinking... First, I will recall that people usually have one father, their male parent. So, if I can find the male parent of the person in question, that should give me the answer. Next, I will consider the male parent as the father of the person, thus the father of the father is the grandfather.\n</think>\n\nTo determine the grand father of the person, let’s follow these steps:\n\n1. **Identify the Person's Mother:**\n   - First, you need to find the mother of the person in question. Maria has one father,",
    "meta_info": {
        "id": "b48bfb9a3f274f38b5f2f3d09ab15640",
        "finish_reason": {
            "type": "length",
            "length": 128
        },
        "prompt_tokens": 7,
        "completion_tokens": 128,
        "cached_tokens": 6
    }
}
```

### 构建可复用的 Dockerfile

---

未完待续
