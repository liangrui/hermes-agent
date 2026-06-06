# vLLM 使用指南 08：量化篇 - 使用量化模型加速推理

## 一、量化技术概述

### 1.1 什么是量化

量化是将模型权重从高精度（如 FP16/FP32）转换为低精度（如 INT8/INT4）的技术，优点是：
- 显著降低显存占用
- 提高推理速度
- 降低硬件要求
- 保持较高的模型质量

### 1.2 vLLM 支持的量化方法

| 方法 | 精度 | 显存节省 | 速度提升 | 支持模型 |
|------|------|----------|----------|----------|
| AWQ | INT4 | ~75% | ~2-4x | LLaMA, Qwen, Mistral 等 |
| GPTQ | INT4 | ~75% | ~2-3x | LLaMA, Qwen, Mistral 等 |
| SqueezeLLM | INT4 | ~75% | ~2x | LLaMA 等 |

---

## 二、使用 AWQ 量化模型

### 2.1 使用预量化模型

```python
from vllm import LLM, SamplingParams

# 使用 AWQ 量化模型
llm = LLM(
    model="TheBloke/Llama-2-7B-Chat-AWQ",
    quantization="awq",
    dtype="float16",
    trust_remote_code=True
)

# 推理
sampling_params = SamplingParams(
    max_tokens=200,
    temperature=0.7
)

outputs = llm.generate("解释什么是机器学习:", sampling_params)
print(outputs[0].outputs[0].text)
```

### 2.2 命令行启动 API 服务

```bash
python -m vllm.entrypoints.openai.api_server \
    --model TheBloke/Llama-2-7B-Chat-AWQ \
    --quantization awq \
    --dtype float16 \
    --host 0.0.0.0 \
    --port 8000
```

### 2.3 AWQ 高级配置

```python
llm = LLM(
    model="TheBloke/Llama-2-7B-Chat-AWQ",
    quantization="awq",
    # AWQ 特定配置
    awq_quantization_config={
        "zero_point": True,
        "group_size": 128,
    },
    # 其他优化
    gpu_memory_utilization=0.85,
    max_model_len=4096,
)
```

---

## 三、使用 GPTQ 量化模型

### 3.1 GPTQ 基础使用

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="TheBloke/Llama-2-7B-Chat-GPTQ",
    quantization="gptq",
    dtype="float16",
    trust_remote_code=True
)

# 推理
sampling_params = SamplingParams(
    max_tokens=150,
    temperature=0.6
)

outputs = llm.generate("介绍一下人工智能的历史:", sampling_params)
print(outputs[0].outputs[0].text)
```

### 3.2 GPTQ 配置选项

```python
llm = LLM(
    model="TheBloke/Llama-2-7B-Chat-GPTQ",
    quantization="gptq",
    # GPTQ 特定配置
    gptq_quantization_config={
        "bits": 4,
        "group_size": 128,
        "desc_act": True,
    },
    # 加载量化权重
    # quantized_weights_path="/path/to/weights.safetensors",
)
```

---

## 四、自己量化模型（可选）

### 4.1 使用 AutoAWQ 量化

```python
# 安装 AutoAWQ
# pip install autoawq

from awq import AutoAWQForCausalLM
from transformers import AutoTokenizer

def quantize_with_awq(model_path, output_path):
    """使用 AWQ 量化模型"""
    
    # 配置
    quant_config = {
        "zero_point": True,
        "q_group_size": 128,
        "w_bit": 4,
        "version": "GEMM"
    }
    
    # 加载模型
    model = AutoAWQForCausalLM.from_pretrained(model_path)
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    
    # 量化
    model.quantize(tokenizer, quant_config=quant_config)
    
    # 保存
    model.save_quantized(output_path)
    tokenizer.save_pretrained(output_path)
    
    print(f"量化完成，模型保存在: {output_path}")
    return output_path

# 使用
quantized_model_path = quantize_with_awq(
    "meta-llama/Llama-2-7b-chat-hf",
    "./llama-2-7b-awq"
)
```

### 4.2 使用 AutoGPTQ 量化

```python
# 安装 AutoGPTQ
# pip install auto-gptq

from transformers import AutoTokenizer, GPTQConfig
from auto_gptq import AutoGPTQForCausalLM

def quantize_with_gptq(model_path, output_path):
    """使用 GPTQ 量化模型"""
    
    # 配置
    gptq_config = GPTQConfig(
        bits=4,
        group_size=128,
        desc_act=True,
        dataset="wikitext2",
    )
    
    # 加载并量化
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    model = AutoGPTQForCausalLM.from_pretrained(
        model_path,
        gptq_config=gptq_config
    )
    
    # 保存
    model.save_quantized(output_path)
    tokenizer.save_pretrained(output_path)
    
    print(f"量化完成，模型保存在: {output_path}")
    return output_path
```

---

## 五、量化模型性能对比

### 5.1 基准测试脚本

```python
import time
import torch
from vllm import LLM, SamplingParams

def benchmark_model(model_name, quantization=None, num_prompts=32):
    """测试模型性能"""
    
    print(f"\n{'='*50}")
    print(f"测试模型: {model_name}")
    print(f"量化方法: {quantization or '无'}")
    print(f"{'='*50}")
    
    # 初始化模型
    llm = LLM(
        model=model_name,
        quantization=quantization,
        dtype="float16",
        trust_remote_code=True,
    )
    
    # 准备提示词
    prompts = ["解释什么是深度学习?"] * num_prompts
    
    # 采样参数
    sampling_params = SamplingParams(
        max_tokens=100,
        temperature=0.7
    )
    
    # 预热
    print("预热中...")
    llm.generate(prompts[:2], sampling_params)
    
    # 正式测试
    print("开始测试...")
    torch.cuda.reset_peak_memory_stats()
    
    start_time = time.time()
    outputs = llm.generate(prompts, sampling_params)
    elapsed_time = time.time() - start_time
    
    # 统计指标
    total_tokens = sum(len(o.outputs[0].token_ids) for o in outputs)
    throughput = total_tokens / elapsed_time
    peak_memory = torch.cuda.max_memory_allocated() / 1024**3
    
    print(f"总耗时: {elapsed_time:.2f}s")
    print(f"吞吐量: {throughput:.2f} tokens/s")
    print(f"峰值显存: {peak_memory:.2f} GB")
    print(f"总 token 数: {total_tokens}")
    
    # 清理
    del llm
    torch.cuda.empty_cache()
    
    return {
        "throughput": throughput,
        "memory": peak_memory,
        "time": elapsed_time
    }

# 对比测试
print("开始对比测试...")

# 原始模型（FP16）
results_fp16 = benchmark_model("meta-llama/Llama-2-7b-chat-hf")

# AWQ 量化模型
results_awq = benchmark_model("TheBloke/Llama-2-7B-Chat-AWQ", quantization="awq")

# GPTQ 量化模型
results_gptq = benchmark_model("TheBloke/Llama-2-7B-Chat-GPTQ", quantization="gptq")

# 打印对比
print("\n" + "="*50)
print("性能对比总结")
print("="*50)
print(f"FP16 - 吞吐量: {results_fp16['throughput']:.1f} tokens/s, 显存: {results_fp16['memory']:.1f} GB")
print(f"AWQ  - 吞吐量: {results_awq['throughput']:.1f} tokens/s, 显存: {results_awq['memory']:.1f} GB")
print(f"GPTQ - 吞吐量: {results_gptq['throughput']:.1f} tokens/s, 显存: {results_gptq['memory']:.1f} GB")
print(f"AWQ 显存节省: {((results_fp16['memory'] - results_awq['memory']) / results_fp16['memory'] * 100):.1f}%")
print(f"AWQ 加速比: {results_awq['throughput'] / results_fp16['throughput']:.1f}x")
```

---

## 六、量化最佳实践

### 6.1 如何选择量化方法

```python
def recommend_quantization(use_case):
    """根据使用场景推荐量化方法"""
    
    recommendations = {
        "最大速度": "AWQ",
        "最高质量": "GPTQ",
        "平衡": "AWQ",
        "低配置硬件": "AWQ",
    }
    
    return recommendations.get(use_case, "AWQ")
```

### 6.2 量化模型配置建议

```python
# 小模型（7B）
llm_7b_awq = LLM(
    model="TheBloke/Llama-2-7B-Chat-AWQ",
    quantization="awq",
    gpu_memory_utilization=0.85,
    max_model_len=4096,
    max_num_batched_tokens=8192,
)

# 中型模型（13B）
llm_13b_awq = LLM(
    model="TheBloke/Llama-2-13B-Chat-AWQ",
    quantization="awq",
    tensor_parallel_size=1,  # 单卡可运行
    gpu_memory_utilization=0.8,
    max_model_len=4096,
)

# 大模型（70B）- 需要多卡
llm_70b_awq = LLM(
    model="TheBloke/Llama-2-70B-Chat-AWQ",
    quantization="awq",
    tensor_parallel_size=4,  # 4 张 GPU
    gpu_memory_utilization=0.85,
)
```

### 6.3 质量评估建议

```python
def evaluate_quantization_quality(original_model, quantized_model, test_prompts):
    """评估量化质量"""
    from difflib import SequenceMatcher
    
    llm_orig = LLM(model=original_model, dtype="float16")
    llm_quant = LLM(model=quantized_model, quantization="awq")
    
    sampling_params = SamplingParams(max_tokens=100, temperature=0)
    
    similarity_scores = []
    
    for prompt in test_prompts:
        orig_output = llm_orig.generate([prompt], sampling_params)[0].outputs[0].text
        quant_output = llm_quant.generate([prompt], sampling_params)[0].outputs[0].text
        
        similarity = SequenceMatcher(None, orig_output, quant_output).ratio()
        similarity_scores.append(similarity)
    
    avg_similarity = sum(similarity_scores) / len(similarity_scores)
    print(f"平均输出相似度: {avg_similarity:.3f}")
    
    return avg_similarity
```

---

## 七、常见问题与解决方案

### 7.1 量化模型加载失败

```python
# 问题：找不到量化配置
# 解决：显式指定量化参数
llm = LLM(
    model="path/to/local/quantized/model",
    quantization="awq",
    trust_remote_code=True,
    # 如果需要，手动指定配置
    # awq_quantization_config={"group_size": 128, "zero_point": True}
)

# 问题：CUDA 内核错误
# 解决：确保使用正确的 PyTorch 和 CUDA 版本
# 检查：torch.__version__
```

### 7.2 量化后质量下降

```python
# 提高量化质量的配置
llm = LLM(
    model="quantized-model",
    quantization="awq",
    # 使用更大的 group_size（以更多显存为代价）
    # awq_quantization_config={"group_size": 64}  # 更好的质量
)

# 或者使用 GPTQ，通常在某些任务上质量更好
llm_gptq = LLM(
    model="gptq-model",
    quantization="gptq"
)
```

---

## 八、总结

本篇我们学习了：
1. 量化技术的基本概念和优势
2. AWQ 和 GPTQ 量化模型的使用方法
3. 如何自己量化模型（可选）
4. 量化模型的性能对比和基准测试
5. 量化的最佳实践和常见问题

下一篇我们将学习生产环境部署指南。
