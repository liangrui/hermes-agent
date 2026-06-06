# vLLM 使用指南 10：精通篇 - 最佳实践与故障排除

## 一、最佳实践总览

### 1.1 性能优化最佳实践

```python
# 性能优化配置模板
from vllm import LLM, SamplingParams

def create_optimized_llm(
    model_path: str,
    quantization: str = "awq",
    tensor_parallel_size: int = 1,
    max_model_len: int = 4096
) -> LLM:
    """创建优化配置的 LLM 实例"""
    return LLM(
        model=model_path,
        quantization=quantization,
        tensor_parallel_size=tensor_parallel_size,
        dtype="auto",
        gpu_memory_utilization=0.9,
        max_model_len=max_model_len,
        max_num_batched_tokens=8192,
        max_num_seqs=256,
        enforce_eager=False,
        trust_remote_code=True
    )

# 使用示例
llm = create_optimized_llm(
    model_path="TheBloke/Llama-2-7B-Chat-AWQ",
    quantization="awq",
    tensor_parallel_size=1
)
```

### 1.2 显存使用优化

| 优化方法 | 显存节省 | 效果说明 |
|---------|---------|---------|
| AWQ 量化 | ~75% | INT4 量化，推荐 |
| GPTQ 量化 | ~75% | INT4 量化 |
| 降低 max_model_len | 30-50% | 减少 KV cache 预留 |
| 降低 gpu_memory_utilization | 可变 | 给系统留更多显存 |
| 批量推理 | 可变 | 提高 GPU 利用率 |

---

## 二、常见问题与故障排除

### 2.1 显存相关问题

**问题 1：CUDA out of memory**

```python
# 解决方案：优化配置
llm = LLM(
    model="your-model",
    # 1. 降低显存占用率
    gpu_memory_utilization=0.7,  # 从 0.9 降低到 0.7
    # 2. 减少最大序列长度
    max_model_len=2048,  # 从 4096 降低到 2048
    # 3. 使用量化
    quantization="awq",
    # 4. CPU offload（如果可用）
    cpu_offload_gb=4,
    swap_space=8
)

# 如果还是不行，尝试用更小的模型
# 或者使用量化版本的模型
```

**问题 2：模型加载缓慢**

```python
import time
from vllm import LLM

def measure_model_load_time(model_path, **kwargs):
    """测量模型加载时间"""
    start = time.time()
    llm = LLM(model=model_path, **kwargs)
    load_time = time.time() - start
    print(f"模型加载时间: {load_time:.2f}秒")
    return llm

# 加速加载的方法：
# 1. 使用本地模型路径
# 2. 确保模型已经缓存
# 3. 禁用不必要的功能
llm = measure_model_load_time(
    model_path="/path/to/local/model",
    trust_remote_code=True,
    # 避免在加载时做额外工作
)
```

### 2.2 生成质量问题

**问题：生成结果重复或质量低**

```python
from vllm import SamplingParams

# 配置采样参数提高质量
sampling_params = SamplingParams(
    temperature=0.7,        # 平衡随机性和一致性
    top_p=0.9,             # nucleus 采样
    top_k=50,              # top-k 采样
    repetition_penalty=1.1, # 轻微重复惩罚
    frequency_penalty=0.1,  # 频率惩罚
    presence_penalty=0.1,   # 存在惩罚
    max_tokens=512,        # 足够的生成长度
    n=1,                   # 单个结果
    seed=42                 # 固定种子（可选，用于调试）
)
```

### 2.3 API 服务问题

**问题 1：API 服务启动失败**

```bash
# 检查 GPU 可用性
nvidia-smi

# 检查 CUDA 版本
nvcc --version

# 检查 vLLM 依赖
pip list | grep vllm
pip list | grep torch

# 尝试降低显存使用
python -m vllm.entrypoints.openai.api_server \
    --model your-model \
    --gpu-memory-utilization 0.7 \
    --max-model-len 2048 \
    --port 8000
```

**问题 2：请求超时或速度慢**

```python
# 1. 使用批量推理
prompts = ["prompt1", "prompt2", "prompt3"]
results = llm.generate(prompts, sampling_params)

# 2. 优化采样参数（减少 max_tokens）
fast_sampling = SamplingParams(
    temperature=0.7,
    max_tokens=100  # 减少到必要的长度
)

# 3. 使用更小的模型或量化版本
```

### 2.4 兼容性问题

**问题：模型加载报错**

```python
# 1. 启用 trust_remote_code
llm = LLM(
    model="your-model",
    trust_remote_code=True  # 重要！
)

# 2. 检查模型格式是否支持
# vLLM 支持: Llama, Mistral, Qwen, Baichuan, ChatGLM, Gemma 等

# 3. 尝试使用 transformers 先测试一下
from transformers import AutoModelForCausalLM, AutoTokenizer
model = AutoModelForCausalLM.from_pretrained("your-model")
tokenizer = AutoTokenizer.from_pretrained("your-model")
# 如果 transformers 也报错，说明模型本身有问题
```

---

## 三、调试技巧

### 3.1 开启详细日志

```python
import logging
import sys

# 配置详细日志
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    stream=sys.stdout
)

# 设置 vLLM 的日志级别
logging.getLogger("vllm").setLevel(logging.DEBUG)

# 现在运行 LLM 会有详细日志
llm = LLM(model="your-model")
```

### 3.2 检查 GPU 状态

```python
import torch
import psutil

def print_gpu_status():
    """打印 GPU 状态信息"""
    print("=" * 50)
    print("GPU 状态:")
    print("=" * 50)
    
    if torch.cuda.is_available():
        print(f"GPU 数量: {torch.cuda.device_count()}")
        
        for i in range(torch.cuda.device_count()):
            print(f"\nGPU {i}:")
            print(f"  名称: {torch.cuda.get_device_name(i)}")
            print(f"  已分配: {torch.cuda.memory_allocated(i)/1024**3:.2f} GB")
            print(f"  已缓存: {torch.cuda.memory_reserved(i)/1024**3:.2f} GB")
            print(f"  总计: {torch.cuda.get_device_properties(i).total_memory/1024**3:.2f} GB")
    else:
        print("未检测到 GPU")

# 使用示例
print_gpu_status()
```

### 3.3 性能分析

```python
import time
from contextlib import contextmanager

@contextmanager
def profile_context(name):
    """性能分析上下文管理器"""
    start = time.time()
    yield
    elapsed = time.time() - start
    print(f"[{name}] 耗时: {elapsed:.3f}秒")

# 使用示例
with profile_context("模型加载"):
    llm = LLM(model="your-model")

with profile_context("生成推理"):
    outputs = llm.generate(["Hello"], sampling_params)
```

---

## 四、生产环境最佳实践检查清单

### 4.1 部署前检查清单

- [ ] 选择合适大小的模型（7B, 13B, 70B）
- [ ] 决定是否使用量化（AWQ 或 GPTQ）
- [ ] 确定 GPU 配置（单卡或多卡 tensor parallel）
- [ ] 配置 API 认证（API Key）
- [ ] 启用 HTTPS/TLS
- [ ] 配置健康检查端点
- [ ] 设置日志轮转
- [ ] 配置监控（Prometheus + Grafana）
- [ ] 设置告警规则
- [ ] 准备回滚方案
- [ ] 进行压力测试

### 4.2 运行时监控检查

```python
# 监控脚本示例
import psutil
import torch
import time

def monitor_resources():
    """监控系统资源使用"""
    print("\n" + "="*50)
    print("资源监控")
    print("="*50)
    
    # CPU
    print(f"CPU 使用率: {psutil.cpu_percent()}%")
    print(f"内存使用: {psutil.virtual_memory().percent}%")
    
    # GPU
    if torch.cuda.is_available():
        for i in range(torch.cuda.device_count()):
            allocated = torch.cuda.memory_allocated(i) / 1024**3
            reserved = torch.cuda.memory_reserved(i) / 1024**3
            print(f"GPU {i} 已分配: {allocated:.2f} GB, 已缓存: {reserved:.2f} GB")

# 定期监控
while True:
    monitor_resources()
    time.sleep(30)  # 每30秒检查一次
```

---

## 五、高级技巧

### 5.1 模型并行推理

```python
from vllm import LLM, SamplingParams

# 使用多张 GPU 张量并行
llm = LLM(
    model="TheBloke/Llama-2-70B-Chat-AWQ",
    tensor_parallel_size=4,  # 使用 4 张 GPU
    quantization="awq",
    gpu_memory_utilization=0.9
)

# 大型模型推理
sampling_params = SamplingParams(
    max_tokens=1024,
    temperature=0.7
)

output = llm.generate("Explain quantum physics", sampling_params)[0]
print(output.outputs[0].text)
```

### 5.2 连续批处理优化

```python
from vllm import LLM, SamplingParams
import time

def process_batch(prompts, chunk_size=32):
    """分批处理大量请求"""
    llm = LLM(
        model="your-model",
        max_num_batched_tokens=8192,
        max_num_seqs=chunk_size
    )
    
    sampling_params = SamplingParams(
        max_tokens=100,
        temperature=0.7
    )
    
    all_results = []
    
    for i in range(0, len(prompts), chunk_size):
        chunk = prompts[i:i + chunk_size]
        outputs = llm.generate(chunk, sampling_params)
        all_results.extend(outputs)
    
    return all_results

# 使用示例
large_prompt_list = [f"Prompt {i}" for i in range(1000)]
results = process_batch(large_prompt_list, chunk_size=32)
```

---

## 六、性能基准测试与对比

### 6.1 vLLM vs 其他推理框架

| 框架 | 吞吐量 (tokens/s) | 显存占用 | 推荐使用场景 |
|------|------------------|---------|-------------|
| vLLM | 高 (✓✓✓✓✓) | 中 | 生产环境，高并发 |
| HuggingFace | 低 (✓✓) | 高 | 开发调试，小批量 |
| Text Generation Inference | 中 (✓✓✓) | 中 | 生产环境 |
| ExLlamaV2 | 高 (✓✓✓✓) | 低 | 量化模型，单用户 |

### 6.2 性能测试脚本

```python
import time
from vllm import LLM, SamplingParams

def benchmark_vllm(
    model_path: str,
    num_prompts: int = 64,
    prompt_length: int = 50,
    gen_tokens: int = 100
):
    """vLLM 基准测试"""
    print(f"测试模型: {model_path}")
    
    # 初始化模型
    llm = LLM(model=model_path)
    
    # 准备测试数据
    base_prompt = "Hello, my name is "
    prompts = [base_prompt * (prompt_length // len(base_prompt))] * num_prompts
    
    # 采样参数
    sampling_params = SamplingParams(
        max_tokens=gen_tokens,
        temperature=0.7
    )
    
    # 预热
    print("预热中...")
    llm.generate(prompts[:2], sampling_params)
    
    # 正式测试
    print("开始基准测试...")
    start = time.time()
    outputs = llm.generate(prompts, sampling_params)
    elapsed = time.time() - start
    
    # 计算指标
    total_tokens = sum(len(output.outputs[0].token_ids) for output in outputs)
    throughput = total_tokens / elapsed
    avg_latency = elapsed / num_prompts
    
    print(f"\n结果:")
    print(f"  总耗时: {elapsed:.2f} 秒")
    print(f"  总 token: {total_tokens}")
    print(f"  吞吐量: {throughput:.2f} tokens/s")
    print(f"  平均延迟: {avg_latency:.3f} s/req")
    
    return throughput, avg_latency

# 运行基准测试
benchmark_vllm("TheBloke/Llama-2-7B-Chat-AWQ")
```

---

## 七、总结与进阶方向

### 7.1 完整学习路径回顾

1. ✅ **入门篇** - vLLM 基础与安装
2. ✅ **基础篇** - 单模型推理
3. ✅ **API 篇** - OpenAI 兼容服务
4. ✅ **批量推理篇** - 高效批量处理
5. ✅ **多模型篇** - 多模型部署
6. ✅ **优化篇** - 性能调优
7. ✅ **高级篇** - 自定义采样与流输出
8. ✅ **量化篇** - 量化模型使用
9. ✅ **部署篇** - 生产环境部署
10. ✅ **精通篇** - 最佳实践与故障排除

### 7.2 进阶学习方向

- 深入了解 vLLM 的 PagedAttention 技术原理
- 研究多模态模型（如 Llava）的 vLLM 支持
- 探索 Speculative Decoding 推理加速
- 学习使用 Ray 进行分布式部署
- 贡献代码到 vLLM 项目

### 7.3 资源链接

- [vLLM 官方文档](https://docs.vllm.ai)
- [vLLM GitHub 仓库](https://github.com/vllm-project/vllm)
- [AWQ 量化工具](https://github.com/mit-han-lab/llm-awq)
- [GPTQ 量化工具](https://github.com/AutoGPTQ/AutoGPTQ)
- [vLLM Discord 社区](https://discord.gg/jw8WnZ4X)

---

**恭喜！你已经完成了全部 10 篇 vLLM 从入门到精通的学习指南！现在你可以自信地在生产环境中使用 vLLM 部署和优化大语言模型了。**
