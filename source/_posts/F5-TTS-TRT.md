# F5-TTS-TRT

克隆项目

```shell
git clone https://github.com/SWivid/F5-TTS.git
```

切换工作目录

```shell
cd F5-TTS/src/f5_tts/runtime/triton_trtllm
```

新建脚本：`vi build_F5_tts_assets.sh`

```shell
#!/bin/bash
set -e

MODEL_ID=${1:-F5TTS_Base}

echo "Starting F5-TTS asset build for model: $MODEL_ID"
echo "Current directory: $(pwd)" 

F5_TTS_HF_DOWNLOAD_PATH="./F5-TTS_HF_MODELS" 
F5_TTS_TRT_LLM_CHECKPOINT_PATH="./trtllm_ckpt"
F5_TTS_TRT_LLM_ENGINE_PATH="./f5_trt_llm_engine"
VOCODER_TRT_ENGINE_PATH="./vocos_vocoder.plan"
MODEL_REPO_PATH="./model_repo" 

# 国内使用镜像站
export HF_MIRROR=https://hf-mirror.com
export HF_ENDPOINT=https://hf-mirror.com

echo "Stage 0: Downloading F5 TTS model ($MODEL_ID) from Hugging Face..."
mkdir -p "$F5_TTS_HF_DOWNLOAD_PATH"
huggingface-cli download SWivid/F5-TTS --repo-type model --include "$MODEL_ID/*" --local-dir "$F5_TTS_HF_DOWNLOAD_PATH" --local-dir-use-symlinks False

echo "Stage 1: Converting checkpoint and building TRT-LLM engine..."
python3 ./scripts/convert_checkpoint.py \
    --timm_ckpt "$F5_TTS_HF_DOWNLOAD_PATH/$MODEL_ID/model_1200000.pt" \
    --output_dir "$F5_TTS_TRT_LLM_CHECKPOINT_PATH" --model_name "$MODEL_ID"

TRTLLM_MODELS_PATH=$(python3 -c "import tensorrt_llm, os; print(os.path.join(os.path.dirname(tensorrt_llm.__file__), 'models'))")
echo "Patching TensorRT-LLM models at: $TRTLLM_MODELS_PATH"
cp -r ./patch/* "$TRTLLM_MODELS_PATH/"

trtllm-build --checkpoint_dir "$F5_TTS_TRT_LLM_CHECKPOINT_PATH" \
  --max_batch_size 8 \
  --output_dir "$F5_TTS_TRT_LLM_ENGINE_PATH" --remove_input_padding disable

echo "Stage 2: Exporting vocos vocoder to ONNX and TensorRT engine..."
ONNX_VOCODER_PATH="./vocos_vocoder.onnx"
python3 scripts/export_vocoder_to_onnx.py --vocoder vocos --output-path "$ONNX_VOCODER_PATH"
bash scripts/export_vocos_trt.sh "$ONNX_VOCODER_PATH" "$VOCODER_TRT_ENGINE_PATH"

echo "Stage 3: Building Triton server model repository..."
rm -rf "$MODEL_REPO_PATH"
cp -r ./model_repo_f5_tts "$MODEL_REPO_PATH"


python3 scripts/fill_template.py -i "$MODEL_REPO_PATH/f5_tts/config.pbtxt" \
    vocab:"$F5_TTS_HF_DOWNLOAD_PATH/$MODEL_ID/vocab.txt",model:"$F5_TTS_HF_DOWNLOAD_PATH/$MODEL_ID/model_1200000.pt",trtllm:"$F5_TTS_TRT_LLM_ENGINE_PATH",vocoder:vocos

mkdir -p "$MODEL_REPO_PATH/vocoder/1/"
cp "$VOCODER_TRT_ENGINE_PATH" "$MODEL_REPO_PATH/vocoder/1/vocoder.plan"

echo "Cleaning up intermediate files to reduce image size..."
rm -rf "$F5_TTS_TRT_LLM_CHECKPOINT_PATH" 
rm -f "$ONNX_VOCODER_PATH" 

echo "F5-TTS asset build completed for model: $MODEL_ID"
```

新建Dockerfile：`vi Dockerfile`

```dockerfile
FROM nvcr.io/nvidia/tritonserver:24.12-py3

ARG MODEL_ID=F5TTS_Base
ENV MODEL_ID=${MODEL_ID}
ENV PYTHONIOENCODING=utf-8
ENV DEBIAN_FRONTEND=noninteractive
ENV HF_MIRROR=https://hf-mirror.com
ENV HF_ENDPOINT=https://hf-mirror.com

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace

COPY requirements-pytorch.txt /workspace/requirements-pytorch.txt

RUN pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/ && \
    pip install --no-cache-dir -r /workspace/requirements-pytorch.txt && \
    pip install --no-cache-dir \
    tritonclient[grpc] \
    tensorrt-llm==0.16.0 \
    torchaudio==2.5.1 \
    jieba \
    pypinyin \
    librosa \
    vocos \
    huggingface_hub[cli]

# 这里需要ghproxy镜像
RUN git clone https://ghfast.top/https://github.com/SWivid/F5-TTS.git

COPY build_F5_tts_assets.sh /workspace/F5-TTS/src/f5_tts/runtime/triton_trtllm/build_F5_tts_assets.sh
RUN chmod +x /workspace/F5-TTS/src/f5_tts/runtime/triton_trtllm/build_F5_tts_assets.sh

WORKDIR /workspace/F5-TTS/src/f5_tts/runtime/triton_trtllm/
RUN ./build_F5_tts_assets.sh ${MODEL_ID}

EXPOSE 8000 8001 8002

CMD ["tritonserver", "--model-repository=/workspace/F5-TTS/src/f5_tts/runtime/triton_trtllm/model_repo", "--strict-model-config=false", "--log-verbose=1"]
```

修改Compose：`vi docker-compose.yml`

```yaml
services:
  tts:
    image: flymyd114-triton-f5-tts:${MODEL_ID:-F5TTS_Base}-prebuilt 
    build:
      context: .
      dockerfile: Dockerfile
      args:
        MODEL_ID: ${MODEL_ID:-F5TTS_Base}
    shm_size: '2gb'
    ports:
      - "8000:8000"
      - "8001:8001"
      - "8002:8002"
    environment:
      - PYTHONIOENCODING=utf-8
      - MODEL_ID=${MODEL_ID:-F5TTS_Base} 
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [gpu]
```

构建镜像

```shell
cd F5-TTS/src/f5_tts/runtime/triton_trtllm/
export MODEL_ID=F5TTS_Base
docker build --build-arg MODEL_ID=${MODEL_ID} -t flymyd114-triton-f5-tts:${MODEL_ID}-prebuilt .
```

构建后启动镜像

```shell
export MODEL_ID=F5TTS_Base
docker compose up
```
