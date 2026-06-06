# vLLM 使用指南 02：基础篇 - 使用vLLM运行单个模型

## 一、LLM 类详细配置

### 1.1 基础配置参数

```python
from vllm import LLM

llm = LLM(
    # 模型配置
    model="meta-llama/Llama-2-7b-hf",
    tokenizer="meta-llama/Llama-2-7b-hf",
    tokenizer_mode="auto",
    
    # 计算配置
    tensor_parallel_size=1,
    pipeline_parallel_size=1,
    
    # 内存配置
    gpu_memory_utilization=0.9,
    swap_space=4,
    cpu_offload_gb=0,
    
    # 模型长度配置
    max_model_len=4096,
    
    # 其他配置
    seed=42,
    trust_remote_code=True,
    enforce_eager=False
)
```

### 1.2 参数详解

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `model` | 模型名称或本地路径 | 必填 |
| `tokenizer` | tokenizer 名称或路径（默认同 model） | `None` |
| `tensor_parallel_size` | 张量并行大小（多卡） | `1` |
| `gpu_memory_utilization` | GPU 内存利用率 | `0.9` |
| `swap_space` | CPU swap 空间大小（GB） | `4` |
| `max_model_len` | 最大模型序列长度 | `None`（自动推断） |
| `trust_remote_code` | 是否信任远程代码 | `False` |
| `seed` | 随机种子 | `0` |

---

## 二、SamplingParams 详细配置

### 2.1 完整参数配置

```python
from vllm import SamplingParams

sampling_params = SamplingParams(
    # 生成长度
    max_tokens=100,
    min_tokens=0,
    
    # 采样策略
    temperature=0.7,
    top_p=0.9,
    top_k=-1,
    repetition_penalty=1.0,
    frequency_penalty=0.0,
    presence_penalty=0.0,
    
    # 特殊控制
    stop=["</s>", "END"],
    stop_token_ids=[2],
    include_stop_str_in_output=False,
    ignore_eos=False,
    
    # 多候选
    n=1,
    best_of=1,
    use_beam_search=False,
    
    # Logits 处理
    logits_processors=None,
    
    # 输出选项
    prompt_logprobs=None,
    logprobs=None,
    skip_special_tokens=True,
    
    # 其他
    seed=None
)
```

### 2.2 采样参数说明

#### 温度参数（temperature）
```python
# 温度越高，随机性越大
temp_high = SamplingParams(temperature=1.5)  # 高随机性
temp_medium = SamplingParams(temperature=0.7)  # 平衡
temp_low = SamplingParams(temperature=0.1)   # 低随机性
temp_zero = SamplingParams(temperature=0)    # 确定性（贪婪采样）
```

#### Top-k 和 Top-p 采样
```python
# Top-k 采样：从概率最高的 k 个 token 中采样
sampling_topk = SamplingParams(top_k=50)

# Top-p（nucleus）采样：从累积概率达到 p 的 token 集合中采样
sampling_topp = SamplingParams(top_p=0.9)

# 组合使用
sampling_both = SamplingParams(top_k=50, top_p=0.9)
```

#### 惩罚参数
```python
sampling_penalty = SamplingParams(
    frequency_penalty=0.5,  # 降低频繁出现 token 的概率
    presence_penalty=0.3,   # 降低已出现 token 的概率
    repetition_penalty=1.1  # 重复惩罚（>1 减少重复）
)
```

---

## 三、实际应用示例

### 3.1 聊天模型应用

#### Llama 2 Chat 格式

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="meta-llama/Llama-2-7b-chat-hf",
    trust_remote_code=True
)

def format_llama2_prompt(messages):
    prompt = ""
    for message in messages:
        if message["role"] == "system":
            prompt += f"<<SYS>>{message['content']}<</SYS>>\n"
        elif message["role"] == "user":
            prompt += f"[INST] {message['content']} [/INST]"
        elif message["role"] == "assistant":
            prompt += f" {message['content']}</s><s>"
    return prompt

# 准备对话
messages = [
    {"role": "system", "content": "你是一个乐于助人的助手。"},
    {"role": "user", "content": "请解释一下什么是人工智能？"}
]

prompt = format_llama2_prompt(messages)

# 生成回复
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=512,
    stop=["</s>"]
)

output = llm.generate(prompt, sampling_params)[0]
print("AI回复:", output.outputs[0].text)
```

#### Qwen 聊天格式

```python
def format_qwen_prompt(messages):
    prompt = ""
    for message in messages:
        if message["role"] == "user":
            prompt += f"<|im_start|>user\n{message['content']}<|im_end|>\n"
        elif message["role"] == "assistant":
            prompt += f"<|im_start|>assistant\n{message['content']}<|im_end|>\n"
        elif message["role"] == "system":
            prompt += f"<|im_start|>system\n{message['content']}<|im_end|>\n"
    prompt += "<|im_start|>assistant\n"
    return prompt
```

### 3.2 代码生成示例

```python
from vllm import LLM, SamplingParams

llm = LLM(model="codellama/CodeLlama-7b-hf")

prompts = [
    "def fibonacci(n):\n    # 计算斐波那契数列",
    "# Python 函数来计算列表的平均值"
]

sampling_params = SamplingParams(
    temperature=0.3,
    top_p=0.95,
    max_tokens=200,
    stop=["\n\n"]
)

outputs = llm.generate(prompts, sampling_params)

for i, output in enumerate(outputs):
    print(f"示例 {i+1}:")
    print(output.outputs[0].text)
    print("-" * 50)
```

### 3.3 文本总结示例

```python
from vllm import LLM, SamplingParams

llm = LLM(model="facebook/opt-1.3b")

def summarize_text(text):
    prompt = f"请总结以下内容，不超过50个字：\n{text}\n\n总结："
    return prompt

long_text = """
人工智能（AI）是计算机科学的一个分支，它企图了解智能的实质，
并生产出一种新的能以人类智能相似的方式做出反应的智能机器。
该领域的研究包括机器人、语言识别、图像识别、自然语言处理和专家系统等。
"""

prompts = [summarize_text(long_text)]

sampling_params = SamplingParams(
    temperature=0.5,
    max_tokens=100
)

outputs = llm.generate(prompts, sampling_params)
print("总结结果:", outputs[0].outputs[0].text)
```

---

## 四、处理生成结果

### 4.1 输出对象结构

```python
outputs = llm.generate(prompts, sampling_params)

# 遍历输出
for output in outputs:
    print(f"原始提示词: {output.prompt}")
    print(f"提示词 token 数: {len(output.prompt_token_ids)}")
    
    # 每个 prompt 可能有多个生成结果
    for completion in output.outputs:
        print(f"生成文本: {completion.text}")
        print(f"生成 token: {completion.token_ids}")
        print(f"完成原因: {completion.finish_reason}")
        
        # 如果启用了 logprobs
        if completion.logprobs:
            print(f"Logprobs: {completion.logprobs}")
```

### 4.2 获取多个候选结果

```python
sampling_params = SamplingParams(
    n=3,  # 生成3个候选
    temperature=0.8,
    max_tokens=100
)

output = llm.generate(prompt, sampling_params)[0]

for i, completion in enumerate(output.outputs):
    print(f"候选 {i+1}: {completion.text}")
```

### 4.3 获取 logprobs

```python
sampling_params = SamplingParams(
    logprobs=5,  # 返回 top 5 token 的 logprob
    prompt_logprobs=5,  # 同时返回 prompt 的 logprob
    max_tokens=50
)

output = llm.generate(prompt, sampling_params)[0]

# 打印每个生成 token 的 top 5 候选
for i, logprob in enumerate(output.outputs[0].logprobs):
    print(f"位置 {i}:")
    for token_id, prob in logprob.items():
        token = llm.get_tokenizer().decode(token_id)
        print(f"  '{token}': {prob:.4f}")
```

---

## 五、进阶技巧

### 5.1 使用自定义 tokenizer

```python
from transformers import AutoTokenizer
from vllm import LLM

# 加载自定义 tokenizer
tokenizer = AutoTokenizer.from_pretrained(
    "your-custom-tokenizer",
    trust_remote_code=True
)

# 传递给 LLM
llm = LLM(
    model="your-model",
    tokenizer=tokenizer
)
```

### 5.2 设置生成限制

```python
sampling_params = SamplingParams(
    max_tokens=512,
    min_tokens=50,  # 至少生成 50 个 token
    stop=["\n\n", "###"],
    ignore_eos=False
)
```

### 5.3 固定种子可复现

```python
# 设置全局种子
llm = LLM(
    model="your-model",
    seed=42
)

# 或在采样时设置
sampling_params = SamplingParams(seed=42)
```

### 5.4 跳过特殊 token

```python
sampling_params = SamplingParams(
    skip_special_tokens=True,  # 跳过特殊 token 如 <s>, </s>
    spaces_between_special_tokens=False
)
```

---

## 六、性能监控

### 6.1 监控生成速度

```python
import time

start_time = time.time()
outputs = llm.generate(prompts, sampling_params)
elapsed_time = time.time() - start_time

total_tokens = sum(len(o.outputs[0].token_ids) for o in outputs)
tokens_per_second = total_tokens / elapsed_time

print(f"生成速度: {tokens_per_second:.2f} tokens/s")
print(f"总耗时: {elapsed_time:.2f}s")
print(f"总 token 数: {total_tokens}")
```

### 6.2 监控显存使用

```python
import torch

# 检查显存使用
print(f"已分配: {torch.cuda.memory_allocated() / 1024**3:.2f} GB")
print(f"已缓存: {torch.cuda.memory_reserved() / 1024**3:.2f} GB")
```

---

## 七、总结

本篇我们学习了：
1. LLM 类的详细配置参数
2. SamplingParams 的完整使用方法
3. 不同模型的提示词格式化
4. 生成结果的解析和处理
5. 实用的进阶技巧
6. 性能监控方法

下一篇我们将学习如何使用 vLLM 提供 OpenAI 兼容的 API 服务。
