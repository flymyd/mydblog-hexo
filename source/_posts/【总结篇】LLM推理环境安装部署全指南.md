---
title: 【总结篇】LLM推理环境安装部署全指南
toc: true
date: 2025-2-25 11:44:10
tags: [Ubuntu, AI, LLM]
categories:
  - [AI研究, 环境搭建]
---

本文介绍了安装WSL2、Ubuntu、NVIDIA显卡驱动、CUDA 12.4.1、cuDNN的方法及Xinference、vLLM、SGLang等推理框架的部署使用方法。

本文写于2025年2月25日。AI技术日新月异，其中的许多内容可能很快过时。

本文适用于NVIDIA显卡用户，建议GPU架构为Turing或更新（对应RTX 20系或以上）。

## 系统安装及基本配置

### 纯净的物理机：Ubuntu Server 22.04.5 LTS

下载地址：https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso

烧录后正常安装即可。

### Windows用户：Ubuntu 22.04.5 LTS on WSL2 

* 确保你的系统不是家庭版，系统版本为 1903 或更高版本，内部版本为 18362.1049 或更高版本
* 关闭电脑上的所有代理软件
* 在开始菜单中找到`控制面板`，右上角搜索`启用或关闭`，选择`启用或关闭Windows功能`
* 勾选`适用于Linux的Windows子系统`、`虚拟机平台`（Win11可能没有部分选项）
* 重启

（对于Windows 10用户）需要额外进行的步骤：

* 下载并安装：https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi
* 打开Powershell管理员，执行以下命令：`wsl --set-default-version 2`

继续：

* 在开始菜单中找到`Microsoft Store`，搜索`ubuntu 22.04.5`，下载第一个。
* 下载完成后会弹出一个自动安装的窗体，关掉。
* 开启代理，打开Powershell管理员。
* 执行以下命令以通过代理快速安装（以监听7890端口的代理为例）：

```powershell
$env:HTTPS_PROXY="127.0.0.1:7897"
wsl --install --web-download
```

* 然后设置用户名和密码即可。

### Windows用户：配置WSL2网络以在宿主机快速访问

* 按下WIN+R，输入`C:\Windows\System32\drivers\etc`，回车
* 找到hosts文件，右键选择“属性” ，选择“安全”选项卡
* 点击“编辑”，找到当前用户组（一般是Users）
* 勾选"完全控制"，在弹出的对话框中确认，点击确定
* 在开始菜单中找到Ubuntu，运行后安装依赖并创建脚本：

```shell
sudo apt update
sudo apt install net-tools -y
vi /opt/win_wsl_domain.sh
```

* 复制如下脚本后，按`i`进入编辑模式，按Shift+Ins粘贴，然后按`ESC`，输入`:wq`回车以保存。

```shell
#!/bin/bash
win_hosts_path="/mnt/c/Windows/System32/drivers/etc/hosts"
wsl_domain="wslubuntu"
# 获取 wsl 的 ip
wsl_ip=$(ifconfig eth0 | grep -w inet | awk '{print $2}')
# 判断是否已存在 win 的域名，如果存在则修改，否则追加
if grep -wq "$wsl_domain" $win_hosts_path; then
    # 此处因为权限问题没有直接用 sed 修改 hosts 文件
    win_hosts=$(sed -s "s/.* $wsl_domain/$wsl_ip $wsl_domain/g" $win_hosts_path)
    echo "$win_hosts" > $win_hosts_path
    echo "$(date '+%Y-%m-%d %H:%M:%S') 修改win-hosts[$wsl_domain] ==> $wsl_ip $wsl_domain"
else
    echo "$wsl_ip $wsl_domain" >> $win_hosts_path
    echo "$(date '+%Y-%m-%d %H:%M:%S') 新增win-hosts[$wsl_domain] ==> $wsl_ip $wsl_domain"
fi
echo "host change ok!"
```

* 赋权限并执行

```shell
chmod +x /opt/win_wsl_domain.sh
/opt/win_wsl_domain.sh
```

* （可选）附加到bash启动项

```shell
vi ~/.bashrc
# 在文件末尾添加
/opt/win_wsl_domain.sh
```

## 驱动安装

### Ubuntu Server用户

* 禁用nouveau

```shell
sudo vi /etc/modprobe.d/blacklist.conf
```

* 尾部追加一行

```shell
blacklist nouveau
```

* 执行并重启系统

```shell
sudo update-initramfs -u
reboot
```

* 检查nouveau是否关闭成功，应当无输出

```shell
lsmod | grep nouveau
```

* 安装550驱动

```shell
sudo apt update
sudo apt install nvidia-driver-550 -y
```

* 当然，也可以查询推荐驱动，选择recommended的版本安装：

```shell
ubuntu-drivers devices
```

装完后执行`nvidia-smi`验证，应该有正确的输出。

### WSL2用户

* 在Windows中安装NVIDIA驱动（如为魔改CMP系列卡或Tesla用户，建议安装`雨糖科技 552.13 CloudGaming驱动`。仅魔改显存如2080Ti 22G等用户安装普通驱动即可）

## CUDA安装

```shell
sudo apt-get install zlib1g -y
wget https://developer.download.nvidia.com/compute/cuda/12.4.1/local_installers/cuda_12.4.1_550.54.15_linux.run
sudo sh cuda_12.4.1_550.54.15_linux.run
```

执行后要等一会加载，然后进入交互式界面，按如下步骤操作：

* 提示已经存在驱动...选择continue
* 阅读并接受协议，输入accept回车
* 上下光标选中`- [X] Driver`列，按空格以取消勾选驱动，再选择Install回车
* 等待安装完成

编辑环境变量：

```shell
vi ~/.bashrc
尾部追加：
export PATH=/usr/local/cuda-12.4/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:/usr/local/cuda-12.4/extras/CUPTI/lib64:/usr/local/cuda-12.4/targets/x86_64-linux/lib:/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
export CUDA_HOME=/usr/local/cuda-12.4
执行：
source ~/.bashrc 
```

验证：

```shell
nvcc --version
systemctl status nvidia-persistenced
```

均应输出有效信息。

## （可选）安装Docker Engine

此处不再赘述，参考官网文档即可。
文档链接：https://docs.docker.com/engine/install/ubuntu/

## （可选）安装NVIDIA-Docker

官网文档链接：`https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html`

* 添加源并安装：

```shell
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit nvidia-container-runtime
```

* 在安装好Docker后，执行如下操作以配置Docker：

```shell
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

* 执行以下命令验证（镜像很大，酌情验证）

```shell
docker run --gpus all -it --name nvtest nvidia/cuda:12.3.1-base-ubuntu22.04 /bin/sh
nvidia-smi
```

## （可选）安装cuDNN

官网文档链接：`https://docs.nvidia.com/deeplearning/cudnn/installation/latest/linux.html#ubuntu-debian-network-installation`

## 安装Miniconda

* 下载并安装miniconda：

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
sh Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

* conda和pip换源

```shell
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
conda config --set show_channel_urls yes
rm ~/.condarc
vi ~/.condarc
```

* `.condarc`的内容，粘贴保存即可：

```shell
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch-lts: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  deepmodeling: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/
ssl_verify: false
```

## 推理集成：Xinference的安装

* 创建`conda`环境

```shell
conda create -n xinference python==3.11 -y
conda activate xinference
```

* 预修补工具链（防止`llama.cpp`构建报错）

```shell
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt update
sudo apt install gcc-11 g++-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 60 --slave /usr/bin/g++ g++ /usr/bin/g++-11
pip install --upgrade pip
pip install --upgrade setuptools wheel
sudo apt-get install build-essential
sudo apt-get install libgomp1
```

* 安装`Xinference`

```shell
pip install "xinference[all]"
```

* 创建运行脚本

```shell
vi xinference.sh
```

以下是脚本内容：

```shell
#!/bin/bash

set -e
CONDA_PATH="/root/miniconda3"
source "${CONDA_PATH}/etc/profile.d/conda.sh"
echo "Activate xinference..."
conda activate xinference
if [ $? -ne 0 ]; then
    echo "Xinference env activate failed！"
    exit 1
fi
echo "Starting xinference..."
xinference-local --host 0.0.0.0 --port 9997
```

记得给执行权限：

```shell
chmod +x xinference.sh
```

完成后执行即可。

## 推理后端：vLLM的安装

实际上，`Xinference`已经集成了vLLM、SGLang、llama.cpp等推理后端。但如果你喜欢更简单的环境，那么仅安装想要的推理后端也可以。

```shell
conda create -n vllm python=3.12 -y
conda activate vllm
pip install --upgrade pip
pip install vllm
```

安装完成后，执行`vllm serve --help`以查看推理参数。

以下是一个`R1-Distill`模型的运行命令示例：

```shell
vllm serve /root/models/FuseO1-DeepSeekR1-QwQ-SkyT1-32B-Preview-AWQ \
    --served-model-name FuseO1-32B \
    --enable-reasoning --reasoning-parser deepseek_r1 \
    --gpu-memory-utilization 0.95 \
    --max-model-len 32768 \
    --pipeline-parallel-size 2 \
    --quantization awq \
    --dtype float16 \
    --host 0.0.0.0 \
    --port 9997
```

具体参数的含义，请参阅文档：https://docs.vllm.ai/en/latest/serving/engine_args.html

如果推理`gguf`模型，请填写模型的具体路径，而非模型所在的目录名

`--served-model-name`为`OpenAI API`的注册ID，可以自己改个喜欢的

`--quantization`根据模型的量化方式来选择

如需张量并行，将`--pipeline-parallel-size`改为`--tensor-parallel-size`

如加载模型时爆显存，调小`--max-model-len`的值

如显卡为30系或更新，删除`--dtype float16`参数

如模型为传统（非思考链）的，删除`--enable-reasoning --reasoning-parser deepseek_r1`参数

## 推理后端：SGLang的安装

安装命令如下：

```shell
conda create -n sglang python=3.12 -y
conda activate sglang
pip install --upgrade pip
pip install "sglang[all]>=0.4.3.post2" --find-links https://flashinfer.ai/whl/cu124/torch2.5/flashinfer-python
```

