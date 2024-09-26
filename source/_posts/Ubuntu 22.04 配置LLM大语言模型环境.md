---
title: Ubuntu 22.04 配置LLM大语言模型环境
toc: true
date: 2023-12-22 00:00:00
tags: [Ubuntu, AI, LLM]
categories:
  - [AI研究, 环境搭建]
---
本文介绍了清洁安装的Ubuntu Server 22.04 LTS安装NVIDIA显卡驱动、CUDA 12.1、cuDNN的方法及ChatGLM3、百川2、FastChat等大语言模型的部署使用方法。

## 安装NVIDIA驱动

禁用nouveau

```shell
sudo vi /etc/modprobe.d/blacklist.conf
```

尾部追加一行

```shell
blacklist nouveau
```

执行并重启系统

```shell
sudo update-initramfs -u
reboot
```

检查nouveau是否关闭成功，应当无输出

```shell
lsmod | grep nouveau
```

安装535驱动

```shell
sudo apt update
sudo apt install nvidia-driver-535 -y
```

当然，也可以查询推荐驱动，选择recommended的版本安装：

```shell
ubuntu-drivers devices
```

装完后执行`nvidia-smi`验证，应该有正确的输出。

## 安装CUDA12.1

下载并执行：

```shell
wget https://developer.download.nvidia.com/compute/cuda/12.1.0/local_installers/cuda_12.1.0_530.30.02_linux.run
chmod +x cuda_12.1.0_530.30.02_linux.run
sh cuda_12.1.0_530.30.02_linux.run
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
export PATH=/usr/local/cuda-12.1/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.1/lib64:$LD_LIBRARY_PATH
执行：
source ~/.bashrc 
```

验证：

```shell
nvcc --version
systemctl status nvidia-persistenced
```

均应输出有效信息。

## 安装NVIDIA-Docker（可选）

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

* 执行以下命令验证

```shell
docker run --gpus all -it --name nvtest nvidia/cuda:12.3.1-base-ubuntu22.04 /bin/sh
nvidia-smi
```

附：

* 使用所有GPU

  `docker run --gpus all nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi`

* 使用两个GPU

  `docker run --gpus 2 nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi`

* 指定GPU运行

  `docker run --gpus '"device=1,2"' nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi`

  `docker run --gpus '"device=UUID-ABCDEF,1"' nvidia/cuda:12.3.1-base-ubuntu22.04 nvidia-smi`

## 安装cuDNN（可选）

官网文档链接：`https://docs.nvidia.com/deeplearning/cudnn/install-guide/index.html#installlinux-deb`

记得安装zlib：

```shell
sudo apt-get install zlib1g
```

## 安装Miniconda

* 下载并安装miniconda：

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
sh Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
```

* conda和pip换源

```shell
conda config --set show_channel_urls yes
vi ~/.condarc

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

pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/
```

## 使用ModelScope下载模型到本地

* 新建一个下载模型的脚本download_model.py

```python
import sys
from modelscope import snapshot_download

if len(sys.argv) < 2:
    print("Usage: python download_model.py <model_name>")
    sys.exit(1)

model_name = sys.argv[1]
model_dir = snapshot_download(model_name, revision="master")
print(model_dir)
```

* 再新建一个shell脚本dl_model.sh用于拉起模型下载脚本

```shell
#!/bin/bash

if [ "$#" -ne 1 ]; then
    echo "Usage: <model_name>"
    exit 1
fi

python download_model.py "$1"
```

## 设置代理脚本（加速github等）

* 设置代理setproxy

```shell
#!/bin/sh

# for terminal
export proxyserveraddr=192.168.114.114
export proxyserverport=7890
export HTTP_PROXY="http://$proxyserveraddr:$proxyserverport/"
export HTTPS_PROXY="https://$proxyserveraddr:$proxyserverport/"
export FTP_PROXY="ftp://$proxyserveraddr:$proxyserverport/"
export SOCKS_PROXY="socks://$proxyserveraddr:$proxyserverport/"
export NO_PROXY="localhost,127.0.0.1,localaddress,.localdomain.com,可以新增你想过滤的ip段;"
export http_proxy="http://$proxyserveraddr:$proxyserverport/"
export https_proxy="https://$proxyserveraddr:$proxyserverport/"
export ftp_proxy="ftp://$proxyserveraddr:$proxyserverport/"
export socks_proxy="socks://$proxyserveraddr:$proxyserverport/"
export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com,可以新增你想过滤的ip段;"

# for apt-get
cat <<-EOF| sudo tee /etc/apt/apt.conf
Acquire::http::proxy "http://$proxyserveraddr:$proxyserverport/";
Acquire::https::proxy "https://$proxyserveraddr:$proxyserverport/";
Acquire::ftp::proxy "ftp://$proxyserveraddr:$proxyserverport/";
Acquire::socks::proxy "socks://$proxyserveraddr:$proxyserverport/";
EOF
```

* 取消代理unsetproxy

```shell
#!/bin/sh
unset proxyserveraddr
unset proxyserverport
unset HTTP_PROXY
unset HTTPS_PROXY
unset FTP_PROXY
unset SOCKS_PROXY
unset NO_PROXY
unset http_proxy
unset https_proxy
unset ftp_proxy
unset socks_proxy
unset no_proxy
gsettings reset org.gnome.system.proxy ignore-hosts
echo -n ""|sudo tee /etc/apt/apt.conf
```

* 使用方法：

```shell
source setproxy
source unsetproxy
```

## 部署ChatGLM3

1. 创建虚拟环境

```shell
conda create -n chatglm python=3.11
```

2. 激活，开搞

```shell
conda activate chatglm
git clone https://github.com/THUDM/ChatGLM3.git
pip install modelscope
pip install -r requirements.txt
```

3. 安装pytorch

```shell
conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia
```

4. 下载模型

```shell
dl_model.sh ZhipuAI/chatglm3-6b
```

下载的模型路径为`/root/.cache/modelscope/hub/ZhipuAI/chatglm3-6b`

5. 编写composite_demo启动脚本（openai_api_demo等同理）

```shell
cd /root/ChatGLM3/composite_demo
vi start.sh
```

以下是start.sh的内容

```shell
export MODEL_PATH=/root/.cache/modelscope/hub/ZhipuAI/chatglm3-6b
streamlit run main.py
```

如果要使用composite_demo的code interpreter，首次启动前记得执行：

```shell
ipython kernel install --name chatglm3 --user
```

## 部署百川2

前面的虚拟环境准备、代码库拉取、模型下载等第1~4节和ChatGLM3差不多。

百川2的git仓库地址是`https://github.com/baichuan-inc/Baichuan2.git`

魔搭的模型名是`baichuan-inc/Baichuan2-7B-Chat`（或13B）

配置：修改启动脚本的init_model函数

```shell
vi Baichuan2/web_demo.py
```

web_demo.py的修改后函数如下：

```python
@st.cache_resource
def init_model():
    model = AutoModelForCausalLM.from_pretrained(
        "/root/.cache/modelscope/hub/baichuan-inc/Baichuan2-7B-Chat",
        torch_dtype=torch.float16,
        device_map="auto",
        trust_remote_code=True
    )
    model.generation_config = GenerationConfig.from_pretrained(
        "/root/.cache/modelscope/hub/baichuan-inc/Baichuan2-7B-Chat"
    )
    tokenizer = AutoTokenizer.from_pretrained(
        "/root/.cache/modelscope/hub/baichuan-inc/Baichuan2-7B-Chat",
        use_fast=False,
        trust_remote_code=True
    )
    return model, tokenizer
```

如果要以int8量化模式运行，在`torch_dtype=torch.float16,`后添加一行`load_in_8bit=True,`即可。

执行：`streamlit run web_demo.py`

## 搭建FastChat服务

* 搭建FastChat环境

```shell
conda create -n fastchat python=3.11
conda activate fastchat
git clone https://github.com/lm-sys/FastChat.git
cd FastChat
pip3 install --upgrade pip
pip3 install -e ".[model_worker,webui]"
```

* 写一组FastChat启动脚本：

  fastchat.sh（使用anaconda就把miniconda3换成anaconda3）

```shell
#!/bin/bash 
source ~/miniconda3/etc/profile.d/conda.sh
conda activate fastchat
if [ $? -ne 0 ]; then
    echo "Failed to activate Anaconda environment 'fastchat'."
    exit 1
fi
/root/fastchat-process/main.sh "$@"
```

* 创建相关文件和文件夹

```shell
mkdir ~/fastchat-process
touch /root/fastchat-process/controller.log
touch /root/fastchat-process/gradio_web_server.log
touch /root/fastchat-process/model_worker.log
touch /root/fastchat-process/openai_api_server.log
```

* 创建fastchat-process/main.sh

```shell
#!/bin/bash

source ~/anaconda3/etc/profile.d/conda.sh
conda activate fastchat

wait_for_startup() {
    while ! grep -q "Application startup complete" <(tail -n 10 -f "$1"); do
        sleep 1
    done
}

python3 -m fastchat.serve.controller &> "/root/fastchat-process/controller.log" &
echo "Controller is starting..."
wait_for_startup /root/fastchat-process/controller.log

python3 -m fastchat.serve.model_worker --model-path "$1" &> "/root/fastchat-process/model_worker.log" &
echo "Model worker is starting with model path $1..."
wait_for_startup /root/fastchat-process/model_worker.log

python3 -m fastchat.serve.openai_api_server --host localhost --port "$2" &> "/root/fastchat-process/openai_api_server.log" &
echo "OpenAI API server is starting on localhost:$2..."
wait_for_startup /root/fastchat-process/openai_api_server.log

python3 -m fastchat.serve.gradio_web_server &> "/root/fastchat-process/gradio_web_server.log" &
echo "Gradio web server is starting..."
wait_for_startup /root/fastchat-process/gradio_web_server.log

echo "All services have been started and are ready."
```

用法：`./fastchat.sh /root/Yi-6B-Chat 8000`

这将启动一个OPENAI_API端口为8000的服务和一个gradio web服务。具体日志可以在对应log中找到。
