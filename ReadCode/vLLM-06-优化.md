# vLLM 使用指南 06：优化篇 - 性能调优技巧

## 一、性能优化概述

vLLM 的性能优化主要从以下几个方面入手：
1. **内存优化** - 显存利用率、KV Cache 管理
2. **计算优化** - 批量处理、并行计算
3. **调度优化** - 连续批处理、请求调度
4. **精度优化** - 混合精度、量化

---

## 二、内存优化配置

### 2.1 GPU 内存利用率调整

```python
from vllm import LLM

# 保守配置 - 降低显存使用
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    gpu_memory_utilization=0.7,  # 降低显存占用
    swap_space=8,  # 增加 CPU swap 空间
    max_model_len=2048,  # 限制最大序列长度
)

# 激进配置 - 最大化性能
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    gpu_memory_utilization=0.95,  # 尽可能使用显存
    max_model_len=4096,
)
```

### 2.2 KV Cache 优化

```python
llm = LLM(
    model="your-model",
    # KV Cache 相关参数
    max_num_batched_tokens=4096,  # 批次最大 token 数
    max_num_seqs=256,            # 批次最大序列数
    enable_prefix_caching=True,   # 启用前缀缓存
)
```

### 2.3 使用 CPU offload

```python
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    cpu_offload_gb=4,  # 4GB 权重 offload 到 CPU
    swap_space=16,     # 16GB swap 空间
)
```

---

## 三、并行计算优化

### 3.1 张量并行（Tensor Parallelism）

```python
# 单 GPU - 不使用并行
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    tensor_parallel_size=1,
)

# 2 张 GPU - 张量并行
llm = LLM(
    model="meta-llama/Llama-2-70b-hf",
    tensor_parallel_size=2,  # 使用 2 张 GPU
)

# 4 张 GPU - 更大模型
llm = LLM(
    model="meta-llama/Llama-2-70b-hf",
    tensor_parallel_size=4,
)
```

启动命令：
```bash
# 指定使用的 GPU
CUDA_VISIBLE_DEVICES=0,1 python your_script.py

# 或使用命令行参数
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-70b-hf \
    --tensor-parallel-size 2
```

### 3.2 流水线并行（Pipeline Parallelism）

```python
# 流水线并行（用于超大规模模型）
llm = LLM(
    model="very-large-model",
    tensor_parallel_size=2,
    pipeline_parallel_size=2,
)
```

---

## 四、批次处理优化

### 4.1 连续批处理配置

```python
llm = LLM(
    model="your-model",
    # 批次大小配置
    max_num_batched_tokens=8192,  # 增加批次 token 容量
    max_num_seqs=512,             # 增加序列数量
    # 调度策略
    disable_log_stats=False,      # 启用统计
)
```

### 4.2 动态批次调整

```python
class DynamicBatchManager:
    def __init__(self, max_tokens=8192, max_seqs=512):
        self.max_tokens = max_tokens
        self.max_seqs = max_seqs
        self.current_load = 0
    
    def get_optimal_batch_params(self, request_rate):
        """根据请求率动态调整批次参数"""
        if request_rate > 100:
            # 高请求率 - 增加批次大小
            return {
                "max_num_batched_tokens": min(self.max_tokens * 2, 16384),
                "max_num_seqs": min(self.max_seqs * 2, 1024)
            }
        elif request_rate < 10:
            # 低请求率 - 减小批次大小以降低延迟
            return {
                "max_num_batched_tokens": max(self.max_tokens // 4, 2048),
                "max_num_seqs": max(self.max_seqs // 4, 64)
            }
        else:
            return {
                "max_num_batched_tokens": self.max_tokens,
                "max_num_seqs": self.max_seqs
            }
```

---

## 五、精度优化

### 5.1 数据类型选择

```python
# FP16 - 平衡性能与精度
llm = LLM(
    model="your-model",
    dtype="float16",
)

# BF16 - 适合新 GPU（A100/H100等）
llm = LLM(
    model="your-model",
    dtype="bfloat16",
)

# FP32 - 最高精度但最慢
llm = LLM(
    model="your-model",
    dtype="float32",
)

# 自动选择（推荐）
llm = LLM(
    model="your-model",
    dtype="auto",
)
```

### 5.2 Flash Attention

```python
llm = LLM(
    model="your-model",
    enforce_eager=False,  # 启用 Flash Attention
)

# 命令行方式
python -m vllm.entrypoints.openai.api_server \
    --model your-model \
    --enforce-eager false
```

---

## 六、性能监控与分析

### 6.1 系统监控脚本

```python
import time
import torch
import psutil
from vllm import LLM, SamplingParams

class PerformanceMonitor:
    def __init__(self):
        self.metrics = []
    
    def measure_performance(self, llm, prompts, sampling_params, num_runs=5):
        """测量推理性能"""
        for i in range(num_runs):
            # 测量 GPU 内存
            torch.cuda.reset_peak_memory_stats()
            
            # 测量时间
            start_time = time.time()
            outputs = llm.generate(prompts, sampling_params)
            elapsed_time = time.time() - start_time
            
            # 收集指标
            peak_memory = torch.cuda.max_memory_allocated() / 1024**3
            total_tokens = sum(len(o.outputs[0].token_ids) for o in outputs)
            throughput = total_tokens / elapsed_time
            
            self.metrics.append({
                "run": i + 1,
                "time": elapsed_time,
                "throughput": throughput,
                "peak_memory_gb": peak_memory,
                "total_tokens": total_tokens
            })
            
            print(f"Run {i+1}: {throughput:.2f} tokens/s, {peak_memory:.2f} GB")
        
        return self.metrics
    
    def print_summary(self):
        """打印性能摘要"""
        if not self.metrics:
            return
        
        import numpy as np
        throughputs = [m["throughput"] for m in self.metrics]
        memories = [m["peak_memory_gb"] for m in self.metrics]
        
        print("\n=== 性能摘要 ===")
        print(f"平均吞吐量: {np.mean(throughputs):.2f} tokens/s (±{np.std(throughputs):.2f})")
        print(f"最大吞吐量: {np.max(throughputs):.2f} tokens/s")
        print(f"平均显存: {np.mean(memories):.2f} GB")
        print(f"峰值显存: {np.max(memories):.2f} GB")

# 使用示例
llm = LLM(model="meta-llama/Llama-2-7b-hf")
prompts = ["Hello, world!"] * 32
sampling_params = SamplingParams(max_tokens=100)

monitor = PerformanceMonitor()
monitor.measure_performance(llm, prompts, sampling_params)
monitor.print_summary()
```

### 6.2 性能对比测试

```python
def compare_configurations(model_name, configs):
    """对比不同配置的性能"""
    results = {}
    
    for config_name, config in configs.items():
        print(f"\n测试配置: {config_name}")
        
        llm = LLM(model=model_name, **config)
        
        prompts = ["Hello, world!"] * 64
        sampling_params = SamplingParams(max_tokens=50)
        
        monitor = PerformanceMonitor()
        metrics = monitor.measure_performance(llm, prompts, sampling_params, num_runs=3)
        
        avg_throughput = sum(m["throughput"] for m in metrics) / len(metrics)
        results[config_name] = avg_throughput
        
        # 清理显存
        del llm
        torch.cuda.empty_cache()
    
    print("\n=== 配置对比结果 ===")
    for name, throughput in results.items():
        print(f"{name}: {throughput:.2f} tokens/s")

# 配置对比
configs_to_test = {
    "baseline": {},
    "high_batch": {"max_num_batched_tokens": 8192, "max_num_seqs": 512},
    "fp16": {"dtype": "float16"},
    "tp2": {"tensor_parallel_size": 2},
}

compare_configurations("meta-llama/Llama-2-7b-hf", configs_to_test)
```

---

## 七、常见场景优化建议

### 7.1 高吞吐量场景

```python
llm = LLM(
    model="your-model",
    gpu_memory_utilization=0.9,
    max_num_batched_tokens=8192,
    max_num_seqs=512,
    dtype="float16",
    enforce_eager=False,
)
```

### 7.2 低延迟场景

```python
llm = LLM(
    model="your-model",
    gpu_memory_utilization=0.8,
    max_num_batched_tokens=1024,
    max_num_seqs=32,
    dtype="float16",
)
```

### 7.3 资源受限场景

```python
llm = LLM(
    model="your-model",
    gpu_memory_utilization=0.6,
    swap_space=8,
    cpu_offload_gb=4,
    max_model_len=1024,
    max_num_seqs=16,
    quantization="awq",  # 使用量化
)
```

---

## 八、总结

本篇我们学习了：
1. 显存优化配置
2. 多 GPU 并行策略
3. 批次处理优化
4. 精度与性能权衡
5. 性能监控与分析方法
6. 不同场景的优化建议

下一篇我们将学习自定义采样和流式输出等高级功能。
