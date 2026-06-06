# vLLM 使用指南 04：批量推理篇 - 高效批量推理

## 一、批量推理基础

### 1.1 什么是批量推理

批量推理是指同时处理多个输入提示词，充分利用 GPU 并行计算能力，大幅提高推理吞吐量。vLLM 通过 PagedAttention 技术和连续批处理（Continuous Batching）实现了卓越的批量处理性能。

### 1.2 基础批量推理示例

```python
from vllm import LLM, SamplingParams
import time

# 初始化模型
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    tensor_parallel_size=1,
    gpu_memory_utilization=0.9
)

# 准备批量输入
prompts = [
    "请写一首关于秋天的诗",
    "什么是机器学习？请简单解释",
    "如何学习编程？",
    "推荐几本好书",
    "写一个 Python 函数计算阶乘"
] * 10  # 复制多次以展示批量处理

# 配置采样参数
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=200
)

# 执行批量推理
start_time = time.time()
outputs = llm.generate(prompts, sampling_params)
elapsed_time = time.time() - start_time

# 统计结果
total_tokens = sum(len(output.outputs[0].token_ids) for output in outputs)
throughput = total_tokens / elapsed_time

print(f"处理 {len(prompts)} 个请求")
print(f"总耗时: {elapsed_time:.2f}秒")
print(f"总 token 数: {total_tokens}")
print(f"吞吐量: {throughput:.2f} tokens/s")
```

---

## 二、批量推理高级配置

### 2.1 LLM 批量相关参数

```python
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    # 批量处理相关配置
    max_num_batched_tokens=8192,     # 单个批次的最大 token 数
    max_num_seqs=256,                # 单个批次的最大序列数
    max_model_len=4096,              # 单个序列的最大长度
    # GPU 内存配置
    gpu_memory_utilization=0.9,
    swap_space=4,
    # 并行配置
    tensor_parallel_size=1
)
```

### 2.2 性能优化参数详解

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `max_num_batched_tokens` | 每个批次的总 token 上限 | GPU 显存大小决定 |
| `max_num_seqs` | 每个批次的最大序列数 | 128-512 |
| `gpu_memory_utilization` | GPU 内存利用率 | 0.85-0.95 |
| `swap_space` | CPU swap 空间（GB） | 4-8 |

---

## 三、实际应用场景

### 3.1 批量文本分类

```python
from vllm import LLM, SamplingParams
import json

llm = LLM(model="meta-llama/Llama-2-7b-chat-hf")

# 准备分类任务数据
texts = [
    "这款手机拍照很好看",
    "这家餐厅的食物很好吃",
    "今天的天气真糟糕",
    "这部电影非常精彩",
    "这个产品质量很差"
]

def build_classification_prompt(text):
    return f"""请分析以下文本的情感，只能回答"正面"或"负面"：
文本：{text}
情感："""

prompts = [build_classification_prompt(text) for text in texts]

sampling_params = SamplingParams(
    temperature=0,  # 确定性输出
    max_tokens=10,
    stop=["\n"]
)

# 批量分类
outputs = llm.generate(prompts, sampling_params)

# 解析结果
results = []
for text, output in zip(texts, outputs):
    sentiment = output.outputs[0].text.strip()
    results.append({
        "text": text,
        "sentiment": sentiment
    })

print(json.dumps(results, ensure_ascii=False, indent=2))
```

### 3.2 批量翻译

```python
from vllm import LLM, SamplingParams

llm = LLM(model="Qwen/Qwen-7B-Chat")

# 待翻译文本
chinese_texts = [
    "人工智能正在改变世界",
    "学习编程需要持续练习",
    "今天天气很好，适合出去走走"
]

def build_translation_prompt(text):
    return f"将以下中文翻译成英文：{text}\n英文翻译："

prompts = [build_translation_prompt(text) for text in chinese_texts]

sampling_params = SamplingParams(
    temperature=0.3,
    max_tokens=100
)

# 批量翻译
outputs = llm.generate(prompts, sampling_params)

for zh, output in zip(chinese_texts, outputs):
    en = output.outputs[0].text.strip()
    print(f"中文: {zh}")
    print(f"英文: {en}")
    print("-" * 50)
```

### 3.3 批量数据标注

```python
from vllm import LLM, SamplingParams
import csv

llm = LLM(model="meta-llama/Llama-2-7b-hf")

# 待标注数据
data = [
    "苹果", "香蕉", "汽车", "自行车", "桌子",
    "椅子", "老虎", "熊猫", "电脑", "手机"
]

def build_annotation_prompt(item):
    return f"""请将以下物品分类为"水果"、"交通工具"、"家具"或"动物"：
物品：{item}
分类："""

prompts = [build_annotation_prompt(item) for item in data]

sampling_params = SamplingParams(
    temperature=0,
    max_tokens=20,
    stop=["\n"]
)

# 批量标注
outputs = llm.generate(prompts, sampling_params)

# 保存结果
annotated_data = []
for item, output in zip(data, outputs):
    category = output.outputs[0].text.strip()
    annotated_data.append([item, category])

with open("annotated_data.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["物品", "分类"])
    writer.writerows(annotated_data)

print("标注完成，已保存到 annotated_data.csv")
```

---

## 四、大规模批量处理

### 4.1 分块处理大型数据集

```python
from vllm import LLM, SamplingParams
import time
from tqdm import tqdm

llm = LLM(model="meta-llama/Llama-2-7b-hf")

# 生成大型数据集（示例）
large_dataset = [f"这是第 {i} 个测试提示词" for i in range(1000)]

def process_in_chunks(dataset, chunk_size=100):
    """分块处理大型数据集"""
    results = []
    
    sampling_params = SamplingParams(
        temperature=0.7,
        max_tokens=50
    )
    
    # 分批处理
    for i in tqdm(range(0, len(dataset), chunk_size), desc="处理进度"):
        chunk = dataset[i:i+chunk_size]
        outputs = llm.generate(chunk, sampling_params)
        results.extend([output.outputs[0].text for output in outputs])
    
    return results

# 执行处理
start_time = time.time()
all_results = process_in_chunks(large_dataset, chunk_size=50)
total_time = time.time() - start_time

print(f"处理完成！共 {len(large_dataset)} 条")
print(f"总耗时: {total_time:.2f}秒")
print(f"平均每条: {total_time/len(large_dataset):.3f}秒")
```

### 4.2 异步批量处理

```python
from vllm import LLM, SamplingParams
import asyncio
from concurrent.futures import ThreadPoolExecutor

llm = LLM(model="meta-llama/Llama-2-7b-hf")
executor = ThreadPoolExecutor(max_workers=1)

async def process_batch_async(prompts):
    """异步处理批次"""
    sampling_params = SamplingParams(temperature=0.7, max_tokens=100)
    
    # 在单独的线程中运行推理
    loop = asyncio.get_event_loop()
    outputs = await loop.run_in_executor(
        executor,
        llm.generate,
        prompts,
        sampling_params
    )
    return outputs

async def main():
    # 准备多个批次
    batches = [
        [f"批次1 - 提示{i}" for i in range(10)],
        [f"批次2 - 提示{i}" for i in range(10)],
        [f"批次3 - 提示{i}" for i in range(10)]
    ]
    
    # 并发处理（注意：vLLM LLM 实例本身不是线程安全的，
    # 实际使用时需要注意同步机制）
    tasks = [process_batch_async(batch) for batch in batches]
    all_outputs = await asyncio.gather(*tasks)
    
    for i, outputs in enumerate(all_outputs):
        print(f"批次 {i+1} 处理完成，共 {len(outputs)} 条")

asyncio.run(main())
```

---

## 五、性能基准测试

### 5.1 批量推理性能测试脚本

```python
from vllm import LLM, SamplingParams
import time
import numpy as np

def benchmark_batch_inference(
    model_name,
    batch_sizes=[1, 8, 16, 32, 64],
    num_runs=3
):
    """基准测试不同批次大小的性能"""
    print(f"测试模型: {model_name}")
    
    llm = LLM(
        model=model_name,
        gpu_memory_utilization=0.9,
        max_num_batched_tokens=8192
    )
    
    # 生成测试提示词
    base_prompts = [
        "Hello, my name is",
        "The capital of France is",
        "Once upon a time"
    ]
    
    results = []
    
    for batch_size in batch_sizes:
        print(f"\n测试批次大小: {batch_size}")
        
        # 准备当前批次的提示词
        prompts = (base_prompts * (batch_size // 3 + 1))[:batch_size]
        
        sampling_params = SamplingParams(
            temperature=0.7,
            max_tokens=128
        )
        
        # 多次运行取平均
        times = []
        for i in range(num_runs):
            start = time.time()
            outputs = llm.generate(prompts, sampling_params)
            elapsed = time.time() - start
            times.append(elapsed)
            
            # 统计 token 数
            total_tokens = sum(len(o.outputs[0].token_ids) for o in outputs)
        
        avg_time = np.mean(times)
        std_time = np.std(times)
        throughput = total_tokens / avg_time
        
        results.append({
            "batch_size": batch_size,
            "avg_time": avg_time,
            "std_time": std_time,
            "throughput": throughput,
            "total_tokens": total_tokens
        })
        
        print(f"  平均耗时: {avg_time:.3f}s (±{std_time:.3f}s)")
        print(f"  吞吐量: {throughput:.2f} tokens/s")
    
    return results

# 运行基准测试
results = benchmark_batch_inference(
    "meta-llama/Llama-2-7b-hf",
    batch_sizes=[1, 4, 8, 16, 32]
)
```

### 5.2 性能对比可视化

```python
import matplotlib.pyplot as plt

def plot_benchmark_results(results):
    """可视化基准测试结果"""
    batch_sizes = [r["batch_size"] for r in results]
    throughputs = [r["throughput"] for r in results]
    avg_times = [r["avg_time"] for r in results]
    
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))
    
    # 吞吐量图
    ax1.plot(batch_sizes, throughputs, 'o-', linewidth=2, markersize=8)
    ax1.set_xlabel('批次大小')
    ax1.set_ylabel('吞吐量 (tokens/s)')
    ax1.set_title('批次大小 vs 吞吐量')
    ax1.grid(True)
    
    # 延迟图
    ax2.plot(batch_sizes, avg_times, 'o-', linewidth=2, markersize=8, color='orange')
    ax2.set_xlabel('批次大小')
    ax2.set_ylabel('平均耗时 (s)')
    ax2.set_title('批次大小 vs 延迟')
    ax2.grid(True)
    
    plt.tight_layout()
    plt.savefig('batch_performance.png')
    print("性能图已保存到 batch_performance.png")
```

---

## 六、最佳实践

### 6.1 批次大小选择建议

```python
def suggest_batch_size(gpu_memory_gb, model_size_billion):
    """根据 GPU 显存和模型大小建议批次大小"""
    # 粗略估算公式
    if model_size_billion <= 7:
        if gpu_memory_gb >= 24:
            return 64
        elif gpu_memory_gb >= 16:
            return 32
        else:
            return 16
    elif model_size_billion <= 13:
        if gpu_memory_gb >= 48:
            return 32
        else:
            return 16
    else:
        return 8  # 更大的模型需要更小的批次
```

### 6.2 内存管理技巧

```python
llm = LLM(
    model="your-model",
    gpu_memory_utilization=0.85,  # 稍微降低以留出余量
    swap_space=8,                 # 增加 swap 空间
    max_num_batched_tokens=4096,  # 限制总 token 数
    enable_prefix_caching=True    # 启用前缀缓存（如果支持）
)
```

---

## 七、总结

本篇我们学习了：
1. 批量推理的基础概念和优势
2. vLLM 批量推理的配置方法
3. 实际应用场景（分类、翻译、标注）
4. 大规模数据的分块处理
5. 性能基准测试方法
6. 最佳实践建议

下一篇我们将学习如何使用 vLLM 同时运行多个模型。
