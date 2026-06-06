# vLLM 使用指南 07：高级篇 - 自定义采样和流输出

## 一、自定义 Logits 处理器

### 1.1 基础概念

vLLM 支持通过 `logits_processors` 自定义 token 采样逻辑，用于实现：
- 自定义输出限制
- 特定 token 的概率调整
- 分类器引导的生成
- 安全内容过滤

### 1.2 简单的 Logits 处理器

```python
import torch
from vllm import LLM, SamplingParams
from typing import List, Optional

def ban_specific_tokens(logits: torch.Tensor, 
                        input_ids: Optional[List[torch.Tensor]] = None) -> torch.Tensor:
    """禁止特定 token 的生成"""
    # 假设我们要禁止 token 100 和 200
    banned_tokens = [100, 200]
    logits[:, banned_tokens] = -float('inf')
    return logits

def temperature_scaling(logits: torch.Tensor, 
                       input_ids: Optional[List[torch.Tensor]] = None) -> torch.Tensor:
    """自定义温度缩放"""
    temperature = 0.5
    logits = logits / temperature
    return logits

# 使用自定义处理器
llm = LLM(model="meta-llama/Llama-2-7b-hf")

sampling_params = SamplingParams(
    max_tokens=100,
    logits_processors=[ban_specific_tokens, temperature_scaling]
)

outputs = llm.generate("Hello, world!", sampling_params)
print(outputs[0].outputs[0].text)
```

### 1.3 高级处理器 - 重复惩罚增强

```python
class AdvancedRepetitionPenalty:
    def __init__(self, penalty: float = 1.1, decay_factor: float = 0.99):
        self.penalty = penalty
        self.decay_factor = decay_factor
        self.token_counts = {}
    
    def __call__(self, logits: torch.Tensor, 
                 input_ids: Optional[List[torch.Tensor]] = None) -> torch.Tensor:
        if input_ids is not None:
            # 统计出现过的 token
            for ids in input_ids:
                for token_id in ids.tolist():
                    self.token_counts[token_id] = self.token_counts.get(token_id, 0) + 1
            
            # 应用惩罚
            for token_id, count in self.token_counts.items():
                # 随着时间衰减惩罚
                current_penalty = self.penalty * (self.decay_factor ** (count - 1))
                logits[:, token_id] /= current_penalty
        
        return logits

# 使用
sampling_params = SamplingParams(
    max_tokens=200,
    logits_processors=[AdvancedRepetitionPenalty(penalty=1.2)]
)
```

### 1.4 基于规则的内容过滤

```python
class ContentSafetyFilter:
    def __init__(self, unsafe_token_ids: List[int]):
        self.unsafe_token_ids = set(unsafe_token_ids)
    
    def __call__(self, logits: torch.Tensor, 
                 input_ids: Optional[List[torch.Tensor]] = None) -> torch.Tensor:
        # 将不安全 token 的概率设为 0
        for token_id in self.unsafe_token_ids:
            logits[:, token_id] = -float('inf')
        return logits

# 获取 tokenizer 来识别不安全词汇
llm = LLM(model="meta-llama/Llama-2-7b-hf")
tokenizer = llm.get_tokenizer()

# 示例：假设某些敏感词的 token ids
unsafe_tokens = []  # 需要根据实际情况填充
for word in ["unsafe_word1", "unsafe_word2"]:
    token_ids = tokenizer.encode(word, add_special_tokens=False)
    unsafe_tokens.extend(token_ids)

sampling_params = SamplingParams(
    max_tokens=100,
    logits_processors=[ContentSafetyFilter(unsafe_tokens)]
)
```

---

## 二、流式输出（Streaming）

### 2.1 Python API 流式生成

```python
from vllm import LLM, SamplingParams

llm = LLM(model="meta-llama/Llama-2-7b-hf")

# 普通（非流式）生成
sampling_params = SamplingParams(max_tokens=100)
outputs = llm.generate("Hello, my name is", sampling_params)
print("完整输出:", outputs[0].outputs[0].text)

# 流式生成 - 使用 AsyncLLMEngine
from vllm import AsyncLLMEngine
from vllm.engine.arg_utils import AsyncEngineArgs
import asyncio

async def stream_generate():
    engine_args = AsyncEngineArgs(model="meta-llama/Llama-2-7b-hf")
    engine = AsyncLLMEngine.from_engine_args(engine_args)
    
    sampling_params = SamplingParams(max_tokens=100, temperature=0.7)
    
    # 生成请求
    request_id = "streaming-demo"
    results_generator = engine.generate(
        "Tell me a story about space exploration.",
        sampling_params,
        request_id
    )
    
    # 流式处理结果
    print("开始生成:")
    generated_text = ""
    async for result in results_generator:
        output = result.outputs[0]
        # 获取新生成的 token
        new_text = output.text[len(generated_text):]
        if new_text:
            print(new_text, end="", flush=True)
            generated_text = output.text
    
    print("\n生成完成!")
    return generated_text

# 运行
asyncio.run(stream_generate())
```

### 2.2 API 服务的流式输出

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

# 流式聊天补全
print("流式输出:")
stream = client.chat.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    messages=[
        {"role": "user", "content": "写一首关于春天的诗"}
    ],
    stream=True,
    temperature=0.8,
    max_tokens=300
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### 2.3 构建流式 Web 服务

```python
from fastapi import FastAPI, Response
from fastapi.responses import StreamingResponse
from vllm import AsyncLLMEngine
from vllm.engine.arg_utils import AsyncEngineArgs
from vllm import SamplingParams
import asyncio
import json

app = FastAPI()

# 全局引擎实例
engine = None

@app.on_event("startup")
async def startup_event():
    global engine
    engine_args = AsyncEngineArgs(
        model="meta-llama/Llama-2-7b-chat-hf",
        tensor_parallel_size=1
    )
    engine = AsyncLLMEngine.from_engine_args(engine_args)

@app.post("/stream/chat")
async def stream_chat(request: dict):
    user_message = request.get("message", "Hello")
    max_tokens = request.get("max_tokens", 200)
    temperature = request.get("temperature", 0.7)
    
    sampling_params = SamplingParams(
        max_tokens=max_tokens,
        temperature=temperature
    )
    
    async def generate():
        request_id = f"req-{id(asyncio.get_event_loop())}"
        results_generator = engine.generate(
            user_message,
            sampling_params,
            request_id
        )
        
        prev_text = ""
        async for result in results_generator:
            current_text = result.outputs[0].text
            new_text = current_text[len(prev_text):]
            
            if new_text:
                yield f"data: {json.dumps({'text': new_text}, ensure_ascii=False)}\n\n"
                prev_text = current_text
        
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

---

## 三、高级采样策略

### 3.1 束搜索（Beam Search）

```python
from vllm import LLM, SamplingParams

llm = LLM(model="meta-llama/Llama-2-7b-hf")

# 束搜索配置
sampling_params = SamplingParams(
    n=3,  # 生成 3 个候选
    best_of=5,  # 从 5 个候选中选最佳
    use_beam_search=True,
    temperature=0,  # 确定性采样
    max_tokens=100,
    length_penalty=1.0  # 长度惩罚
)

outputs = llm.generate("Explain quantum computing:", sampling_params)

# 打印所有候选结果
for i, output in enumerate(outputs):
    for j, completion in enumerate(output.outputs):
        print(f"\n候选 {j+1}:")
        print(completion.text)
```

### 3.2 约束采样（Constrained Sampling）

```python
class RegexConstrainedSampler:
    """使用正则表达式约束输出格式"""
    def __init__(self, pattern: str, tokenizer):
        import re
        self.pattern = re.compile(pattern)
        self.tokenizer = tokenizer
    
    def __call__(self, logits: torch.Tensor, 
                 input_ids: Optional[List[torch.Tensor]] = None) -> torch.Tensor:
        # 获取当前已生成的文本
        if input_ids is None:
            return logits
        
        # 这里简化实现，实际需要更复杂的状态管理
        # 完整实现可以结合 outlines 或 lm-format-enforcer 等库
        return logits

# 使用示例（需要额外库支持）
sampling_params = SamplingParams(
    max_tokens=200,
    # logits_processors=[RegexConstrainedSampler(r'^[A-Z].*\.$', tokenizer)]
)
```

---

## 四、输出分析与调试

### 4.1 获取 Logits 和概率

```python
sampling_params = SamplingParams(
    max_tokens=50,
    logprobs=5,  # 返回 top 5 token 的概率
    prompt_logprobs=5  # 同时返回提示词的概率
)

outputs = llm.generate("Hello, my name is", sampling_params)
output = outputs[0]

# 分析生成的每个 token
print("生成分析:")
for i, (token_id, logprob_dict) in enumerate(zip(
    output.outputs[0].token_ids,
    output.outputs[0].logprobs
)):
    token = llm.get_tokenizer().decode(token_id)
    print(f"\n位置 {i}: '{token}'")
    
    # 打印 top 5 候选
    sorted_logprobs = sorted(logprob_dict.items(), key=lambda x: -x[1])
    for j, (candidate_id, logprob) in enumerate(sorted_logprobs[:5]):
        candidate = llm.get_tokenizer().decode(candidate_id)
        prob = torch.exp(torch.tensor(logprob)).item()
        print(f"  {j+1}. '{candidate}': {prob:.4f}")
```

### 4.2 完成原因分析

```python
outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    for completion in output.outputs:
        print(f"\n完成原因: {completion.finish_reason}")
        print(f"生成长度: {len(completion.token_ids)}")
        
        if completion.finish_reason == "length":
            print("达到最大生成长度")
        elif completion.finish_reason == "stop":
            print("遇到停止词或 EOS token")
```

---

## 五、总结

本篇我们学习了：
1. 自定义 Logits 处理器的实现
2. 流式输出的使用方法
3. 高级采样策略（束搜索等）
4. 输出分析与调试技巧

下一篇我们将学习如何使用量化技术加速推理。
