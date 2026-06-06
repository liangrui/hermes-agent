# vLLM 使用指南 01：入门篇 - 什么是vLLM及快速安装

## 一、什么是vLLM

### 1.1 vLLM简介

vLLM 是一个高性能的大语言模型推理和服务库，由加州大学伯克利分校开发。它以**极致的推理速度**和**高效的显存利用**著称，通过创新的 PagedAttention 技术实现了比传统推理框架高出数倍的吞吐量。

### 1.2 核心特性

- **PagedAttention 技术**：将虚拟内存管理思想引入注意力机制，高效处理变长序列
- **高吞吐量**：支持连续批处理（continuous batching），吞吐量比 HuggingFace Transformers 高 10-100 倍
- **灵活的部署方式**：支持 Python API、OpenAI 兼容 API、命令行等多种使用方式
- **丰富的模型支持**：支持 Llama、Qwen、Baichuan、ChatGLM、Phi 等主流模型
- **量化支持**：支持 AWQ、GPTQ、SqueezeLLM 等量化技术
- **分布式推理**：支持 Tensor Parallelism 进行多卡分布式推理

### 1.3 适用场景

- **高并发 API 服务**：为多个用户同时提供 LLM 服务
- **批量推理**：处理大量文本生成任务
- **资源受限环境**：在有限显存下运行大模型

---

## 二、环境准备

### 2.1 系统要求

- **操作系统**：Linux（推荐）、macOS、Windows（WSL2）
- **Python**：3.8 - 3.11
- **CUDA**：11.8+（如需 GPU 加速）
- **GPU**：NVIDIA GPU，显存 8GB+（推荐 24GB+）

### 2.2 检查环境

```bash
# 检查 Python 版本
python --version

# 检查 CUDA 版本（如有 GPU）
nvidia-smi
```

---

## 三、安装vLLM

### 3.1 使用 pip 安装（推荐）

#### 方式一：稳定版安装

```bash
# 安装最新稳定版
pip install vllm

# 或指定版本
pip install vllm==0.4.2
```

#### 方式二：从源码安装（如需最新特性）

```bash
# 克隆代码库
git clone https://github.com/vllm-project/vllm.git
cd vllm

# 安装
pip install -e .
```

#### 方式三：使用 Docker

```bash
# 拉取官方镜像
docker pull vllm/vllm-openai:latest

# 运行容器
docker run --gpus all -p 8000:8000 vllm/vllm-openai:latest --model meta-llama/Llama-2-7b-hf
```

### 3.2 可选依赖安装

```bash
# 安装开发依赖
pip install vllm[dev]

# 安装特定后端
pip install vllm[flash-attn]  # FlashAttention 支持
pip install vllm[ray]        # Ray 分布式支持
```

### 3.3 验证安装

```python
import vllm
print(f"vLLM 版本: {vllm.__version__}")
```

---

## 四、快速开始

### 4.1 第一个简单示例

创建 `simple_demo.py`：

```python
from vllm import LLM, SamplingParams

# 1. 初始化模型
llm = LLM(model="facebook/opt-125m")

# 2. 准备提示词
prompts = [
    "Hello, my name is",
    "The capital of France is",
    "Once upon a time,"
]

# 3. 设置采样参数
sampling_params = SamplingParams(
    temperature=0.8,
    top_p=0.95,
    max_tokens=50
)

# 4. 生成文本
outputs = llm.generate(prompts, sampling_params)

# 5. 打印结果
for output in outputs:
    prompt = output.prompt
    generated_text = output.outputs[0].text
    print(f"提示词: {prompt!r}")
    print(f"生成结果: {generated_text!r}")
    print("-" * 50)
```

运行示例：

```bash
python simple_demo.py
```

### 4.2 核心组件介绍

#### LLM 类
```python
from vllm import LLM

# 基本初始化
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",  # 模型名称或路径
    tensor_parallel_size=1,            # 张量并行大小（多卡）
    gpu_memory_utilization=0.9,        # GPU 内存利用率
    trust_remote_code=True             # 信任远程代码（用于自定义模型）
)
```

#### SamplingParams 类
```python
from vllm import SamplingParams

# 配置采样参数
sampling_params = SamplingParams(
    max_tokens=100,          # 最大生成 token 数
    temperature=0.7,         # 温度参数（0=确定性）
    top_p=0.9,               # nucleus sampling
    top_k=50,                # top-k sampling
    frequency_penalty=0.0,   # 频率惩罚
    presence_penalty=0.0,    # 存在惩罚
    stop=["\n", "END"],      # 停止词
    n=1                      # 生成多少个候选
)
```

---

## 五、使用 HuggingFace 模型

### 5.1 使用本地模型

```python
llm = LLM(model="/path/to/your/local/model")
```

### 5.2 使用 HuggingFace Hub 模型

```python
# 自动从 HuggingFace Hub 下载
llm = LLM(model="Qwen/Qwen-7B-Chat")
```

### 5.3 使用自定义模型

如果模型需要特殊加载代码，设置 `trust_remote_code=True`：

```python
llm = LLM(
    model="your-custom/model",
    trust_remote_code=True
)
```

---

## 六、常见问题排查

### 6.1 显存不足

```python
# 降低 GPU 内存利用率
llm = LLM(
    model="your-model",
    gpu_memory_utilization=0.7,  # 降低到 70%
    max_model_len=2048           # 限制最大序列长度
)
```

### 6.2 CUDA 版本不匹配

确保安装的 PyTorch 和 vLLM 与系统 CUDA 版本匹配：

```bash
# 检查 PyTorch CUDA 版本
python -c "import torch; print(torch.version.cuda)"
```

### 6.3 模型下载问题

配置 HuggingFace 镜像：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

---

## 七、总结

本篇我们学习了：
1. vLLM 的核心概念和优势
2. 如何安装和配置 vLLM
3. 快速上手的第一个示例
4. 核心组件的基本用法
5. 常见问题的排查方法

下一篇我们将深入学习如何使用 vLLM 运行单个模型，掌握更多细节配置。
