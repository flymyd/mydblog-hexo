# ComfyUI老照片修复

## 一、部署ComfyUI及ComfyUI Manager

```bash
conda create -n comfyui python==3.11 -y

conda activate comfyui

pip install torch torchvision torchaudio

git clone https://github.com/comfyanonymous/ComfyUI.git

cd ComfyUI

pip install -r requirements.txt

cd custom_nodes

git clone https://github.com/ltdrdata/ComfyUI-Manager comfyui-manager

cd comfyui-manager

pip install -r requirements.txt
```

## 二、下载所需模型

- 模型文件
  - juggernautXL_v9Rdphoto2Lightning.safetensors ：[下载地址](https://hf-mirror.com/martintomov/comfy/resolve/46fede7a4c0bd8f3bee95aa2e5d090bd11d87ff5/juggernautXL_v9Rdphoto2Lightning.safetensors)。
  - SUPIR-v0Q.ckpt：[下载地址](https://huggingface.co/camenduru/SUPIR/blob/main/SUPIR-v0Q.ckpt)。
  - diffusers_xl_depth_small.safetensors：[下载地址](https://huggingface.co/lllyasviel/sd_control_collection/blob/d1b278d0d1103a3a7c4f7c2c327d236b082a75b1/diffusers_xl_depth_small.safetensors)。
  - t2i-adapter_xl_openpose.safetensors：[下载地址](https://huggingface.co/lllyasviel/sd_control_collection/blob/main/t2i-adapter_xl_openpose.safetensors)。
  - t2i-adapter_diffusers_xl_lineart.safetensors：[下载地址](https://huggingface.co/lllyasviel/sd_control_collection/blob/7cf256327b341fedc82e00b0d7fb5481ad693210/t2i-adapter_diffusers_xl_lineart.safetensors)。
  - 4x_foolhardy_Remacri.pth：[下载地址](https://huggingface.co/FacehugmanIII/4x_foolhardy_Remacri/blob/main/4x_foolhardy_Remacri.pth)。
  - control-lora-recolor-rank256.safetensors：[下载地址](https://huggingface.co/licyk/control-lora/blob/efa33a28d9b50b8492aa59ca44b6ee217a37497a/control-lora-recolor-rank256.safetensors)。

```bash
vi ~/comfy_models.sh


#!/bin/bash

TARGET_DIR="/root/ComfyUI/models"
mkdir -p "$TARGET_DIR"

if ! command -v aria2c &> /dev/null; then
    echo "Installing aria2..."
    apt update && apt install -y aria2
fi

urls=(
"https://hf-mirror.com/martintomov/comfy/resolve/46fede7a4c0bd8f3bee95aa2e5d090bd11d87ff5/juggernautXL_v9Rdphoto2Lightning.safetensors"
"https://hf-mirror.com/camenduru/SUPIR/resolve/main/SUPIR-v0Q.ckpt"
"https://hf-mirror.com/lllyasviel/sd_control_collection/resolve/d1b278d0d1103a3a7c4f7c2c327d236b082a75b1/diffusers_xl_depth_small.safetensors"
"https://hf-mirror.com/lllyasviel/sd_control_collection/resolve/main/t2i-adapter_xl_openpose.safetensors"
"https://hf-mirror.com/lllyasviel/sd_control_collection/resolve/7cf256327b341fedc82e00b0d7fb5481ad693210/t2i-adapter_diffusers_xl_lineart.safetensors"
"https://hf-mirror.com/FacehugmanIII/4x_foolhardy_Remacri/resolve/main/4x_foolhardy_Remacri.pth"
)

for url in "${urls[@]}"; do
    filename=$(basename "$url")
    echo "尝试下载：$filename"
    aria2c -x 16 -s 16 -o "$filename" \
        --check-certificate=false \
        --header="User-Agent: Mozilla/5.0" \
        --header="Referer: https://hf-mirror.com" \
        -d "$TARGET_DIR" "$url"
done

echo "✅ 所有模型下载完成，保存在：$TARGET_DIR"
```

以下是一个自动带放置的脚本：

```bash
#!/bin/bash

# --- 配置 ---
# ComfyUI 的 models 目录的绝对路径
# 请根据您的实际环境修改此路径
BASE_DIR="/root/ComfyUI/models"

# --- 脚本开始 ---

echo "脚本将把模型下载到: $BASE_DIR"
echo "请确保该路径正确。"

# 检查并安装 aria2c (如果需要)
if ! command -v aria2c &> /dev/null; then
    echo "aria2c 未找到，正在尝试安装..."
    # 适用于 Debian/Ubuntu 系统
    apt-get update && apt-get install -y aria2
    if [ $? -ne 0 ]; then
        echo "❌ aria2 安装失败。请手动安装后再运行此脚本。"
        exit 1
    fi
fi

# 使用 here-document 定义模型列表
# 格式: <目标子目录> <模型下载URL>
while read -r dest_subdir url; do
    # 拼接成完整的目标路径
    full_dest_path="$BASE_DIR/$dest_subdir"

    # 获取文件名
    filename=$(basename "$url")

    # 创建目标子目录 (如果不存在)
    mkdir -p "$full_dest_path"

    # 检查文件是否已存在
    if [ -f "$full_dest_path/$filename" ]; then
        echo "✅ 文件已存在，跳过下载: $filename"
    else
        echo "==> 开始下载: $filename"
        echo "    目标路径: $full_dest_path"

        aria2c \
            -x 16 \
            -s 16 \
            --console-log-level=warn \
            --check-certificate=false \
            --header="User-Agent: Mozilla/5.0" \
            --header="Referer: https://hf-mirror.com/" \
            -d "$full_dest_path" \
            "$url"

        if [ $? -eq 0 ]; then
            echo "✅ 下载成功: $filename"
        else
            echo "❌ 下载失败: $filename"
        fi
    fi
    echo "--------------------------------------------------"
done <<'EOF'
checkpoints/sdxl https://hf-mirror.com/martintomov/comfy/resolve/46fede7a4c0bd8f3bee95aa2e5d090bd11d87ff5/juggernautXL_v9Rdphoto2Lightning.safetensors
checkpoints https://hf-mirror.com/camenduru/SUPIR/resolve/main/SUPIR-v0Q.ckpt
controlnet https://hf-mirror.com/lllyasviel/sd_control_collection/resolve/d1b278d0d1103a3a7c4f7c2c327d236b082a75b1/diffusers_xl_depth_small.safetensors
controlnet https://hf-mirror.com/lllyasviel/sd_control_collection/resolve/main/t2i-adapter_xl_openpose.safetensors
controlnet https://hf-mirror.com/lllyasviel/sd_control_collection/resolve/7cf256327b341fedc82e00b0d7fb5481ad693210/t2i-adapter_diffusers_xl_lineart.safetensors
upscale_models https://hf-mirror.com/FacehugmanIII/4x_foolhardy_Remacri/resolve/main/4x_foolhardy_Remacri.pth
EOF

echo "🎉 所有指定的模型已处理完毕。"
```

## 三、启动服务

```bash
CUDA_VISIBLE_DEVICES=2 HTTPS_PROXY=http://192.168.192.252:7890 python main.py
```

## 四、修复节点

点击`Manager`，选择`Install Missing Custom Nodes`，全部勾中并等待安装完成。

## 五、节点说明

* 加载图片（Load Image）在这里上传或选择待修复的老照片

* 加载模型（Load Checkpoint）`sdxl\juggernautXL_v9Rdphoto2Lightning.safetensors` ，这是合并了 **4 Steps LoRA** 的 Juggernaut V9 模型

* 加载 SUPIR-v0Q.ckpt 模型

* 加载控制网络（Load ControlNet Model）control-lora-recolor-rank256.safetensors 用于给黑白色的老照片重新上色

* 加载高级控制网络（Load Advanced ControlNet Model） diffusers_xl_depth_small.safetensors

* 加载高级控制网络（Load Advanced ControlNet Model） t2i-adapter_xl_openpose.safetensors

* 加载高级控制网络（Load Advanced ControlNet Model） t2i-adapter_diffusers_xl_lineart.safetensors

* 图像放大到8K（8K by Upscale）加载 4x_foolhardy_Remacri.pth 模型并放大图像至 8K 分辨率

本工作流程可以输出 2K，上色，放大2K与最终放大对比，8K调色，8K调色前后对比，原图与8K对比等图片。按照自己的需求保存使用。
