# vLLM 使用指南 05：多模型篇 - 同时运行多个模型

## 一、多模型运行概述

### 1.1 为什么需要多模型

在实际应用中，我们经常需要：
- 不同模型处理不同任务（如：通用对话、代码生成、翻译等）
- 根据任务动态选择最合适的模型
- A/B 测试不同模型的效果
- 模型 ensemble 提升效果

### 1.2 vLLM 多模型支持方式

vLLM 提供了多种多模型部署方式：
1. **多个独立 vLLM 进程** - 最稳定但资源消耗大
2. **API 服务多模型加载** - 通过命令行参数加载多个模型
3. **使用 vllm serve** - 新版本的服务模式

---

## 二、多模型 API 服务部署

### 2.1 命令行启动多模型服务

```bash
# 方式一：通过 --model 和 --served-model-name 参数
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --served-model-name llama-2-7b \
    --model Qwen/Qwen-7B-Chat \
    --served-model-name qwen-7b \
    --model mistralai/Mistral-7B-Instruct-v0.2 \
    --served-model-name mistral-7b \
    --host 0.0.0.0 \
    --port 8000
```

### 2.2 使用配置文件

创建 `models_config.yaml`：
```yaml
models:
  - name: llama-2-7b
    path: meta-llama/Llama-2-7b-chat-hf
    tensor_parallel_size: 1
    gpu_memory_utilization: 0.4
    
  - name: qwen-7b
    path: Qwen/Qwen-7B-Chat
    tensor_parallel_size: 1
    gpu_memory_utilization: 0.4
```

### 2.3 客户端调用多模型

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

# 调用 Llama 2
response1 = client.chat.completions.create(
    model="llama-2-7b",
    messages=[{"role": "user", "content": "Hello!"}]
)
print("Llama 2 回复:", response1.choices[0].message.content)

# 调用 Qwen
response2 = client.chat.completions.create(
    model="qwen-7b",
    messages=[{"role": "user", "content": "你好！"}]
)
print("Qwen 回复:", response2.choices[0].message.content)
```

---

## 三、多进程部署方案

### 3.1 使用不同端口部署多个服务

```bash
# 终端 1 - 启动 Llama 2 服务
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --host 0.0.0.0 \
    --port 8001

# 终端 2 - 启动 Qwen 服务
python -m vllm.entrypoints.openai.api_server \
    --model Qwen/Qwen-7B-Chat \
    --host 0.0.0.0 \
    --port 8002
```

### 3.2 使用 Nginx 做反向代理

```nginx
upstream vllm_backends {
    server 127.0.0.1:8001;  # Llama 2
    server 127.0.0.1:8002;  # Qwen
}

server {
    listen 8000;
    
    location /llama/ {
        rewrite ^/llama/(.*) /$1 break;
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
    }
    
    location /qwen/ {
        rewrite ^/qwen/(.*) /$1 break;
        proxy_pass http://127.0.0.1:8002;
        proxy_set_header Host $host;
    }
}
```

### 3.3 使用 Docker Compose 管理

创建 `docker-compose.yml`：
```yaml
version: '3.8'

services:
  llama-server:
    image: vllm/vllm-openai:latest
    ports:
      - "8001:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    command: --model meta-llama/Llama-2-7b-chat-hf --host 0.0.0.0 --port 8000

  qwen-server:
    image: vllm/vllm-openai:latest
    ports:
      - "8002:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    command: --model Qwen/Qwen-7B-Chat --host 0.0.0.0 --port 8000
```

启动服务：
```bash
docker-compose up -d
```

---

## 四、模型路由与调度

### 4.1 智能路由服务

```python
from flask import Flask, request, jsonify
from openai import OpenAI

app = Flask(__name__)

# 配置不同模型的客户端
MODEL_CLIENTS = {
    "llama": OpenAI(base_url="http://localhost:8001/v1", api_key="none"),
    "qwen": OpenAI(base_url="http://localhost:8002/v1", api_key="none"),
    "code": OpenAI(base_url="http://localhost:8003/v1", api_key="none")
}

def select_model(prompt):
    """根据输入选择合适的模型"""
    prompt_lower = prompt.lower()
    
    # 代码相关
    if any(keyword in prompt_lower for keyword in ["代码", "编程", "函数", "python", "code"]):
        return "code"
    
    # 中文任务
    if any("\u4e00" <= char <= "\u9fff" for char in prompt):
        return "qwen"
    
    # 默认使用 Llama
    return "llama"

@app.route("/chat", methods=["POST"])
def chat():
    data = request.json
    messages = data.get("messages", [])
    user_input = messages[-1]["content"] if messages else ""
    
    # 智能选择模型
    selected_model = select_model(user_input)
    client = MODEL_CLIENTS[selected_model]
    
    print(f"使用模型: {selected_model}")
    
    response = client.chat.completions.create(
        model=selected_model,
        messages=messages,
        temperature=0.7,
        max_tokens=512
    )
    
    return jsonify({
        "content": response.choices[0].message.content,
        "model_used": selected_model
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

### 4.2 模型负载均衡

```python
import random
from openai import OpenAI

class ModelLoadBalancer:
    def __init__(self):
        self.model_instances = {
            "llama": [
                OpenAI(base_url="http://localhost:8001/v1", api_key="none"),
                OpenAI(base_url="http://localhost:8011/v1", api_key="none")
            ],
            "qwen": [
                OpenAI(base_url="http://localhost:8002/v1", api_key="none")
            ]
        }
        self.request_counts = {model: [0] * len(instances) 
                              for model, instances in self.model_instances.items()}
    
    def get_client(self, model_name):
        """使用最少连接策略选择实例"""
        instances = self.model_instances.get(model_name)
        if not instances:
            return None
        
        # 选择请求最少的实例
        counts = self.request_counts[model_name]
        min_idx = counts.index(min(counts))
        
        # 更新计数
        self.request_counts[model_name][min_idx] += 1
        
        return instances[min_idx]
    
    def release_client(self, model_name, client_idx):
        """释放请求计数"""
        if model_name in self.request_counts and client_idx < len(self.request_counts[model_name]):
            self.request_counts[model_name][client_idx] -= 1
```

---

## 五、多模型 Ensemble

### 5.1 模型投票集成

```python
from openai import OpenAI
from collections import Counter

class ModelEnsemble:
    def __init__(self):
        self.clients = [
            OpenAI(base_url="http://localhost:8001/v1", api_key="none"),
            OpenAI(base_url="http://localhost:8002/v1", api_key="none"),
            OpenAI(base_url="http://localhost:8003/v1", api_key="none")
        ]
        self.model_names = ["llama", "qwen", "mistral"]
    
    def generate(self, prompt, temperature=0.7, max_tokens=100):
        """使用多个模型生成并选择最佳结果"""
        results = []
        
        for client, model_name in zip(self.clients, self.model_names):
            try:
                response = client.completions.create(
                    model=model_name,
                    prompt=prompt,
                    temperature=temperature,
                    max_tokens=max_tokens
                )
                results.append({
                    "model": model_name,
                    "text": response.choices[0].text
                })
            except Exception as e:
                print(f"模型 {model_name} 出错: {e}")
        
        return results
    
    def classification_vote(self, prompt, labels):
        """分类任务的投票集成"""
        predictions = []
        
        for client, model_name in zip(self.clients, self.model_names):
            try:
                formatted_prompt = f"{prompt}\n答案只能是以下之一: {', '.join(labels)}\n答案:"
                response = client.completions.create(
                    model=model_name,
                    prompt=formatted_prompt,
                    temperature=0,
                    max_tokens=10
                )
                pred = response.choices[0].text.strip()
                # 找到最匹配的标签
                matched = min(labels, key=lambda x: abs(len(x) - len(pred)))
                predictions.append(matched)
            except Exception as e:
                print(f"模型 {model_name} 出错: {e}")
        
        # 投票
        if predictions:
            vote_result = Counter(predictions).most_common(1)[0][0]
            return vote_result, predictions
        
        return None, []
```

---

## 六、GPU 资源管理

### 6.1 使用多 GPU 分别部署

```bash
# GPU 0 部署 Llama 2
CUDA_VISIBLE_DEVICES=0 python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --port 8001

# GPU 1 部署 Qwen
CUDA_VISIBLE_DEVICES=1 python -m vllm.entrypoints.openai.api_server \
    --model Qwen/Qwen-7B-Chat \
    --port 8002

# GPU 2-3 使用张量并行部署大模型
CUDA_VISIBLE_DEVICES=2,3 python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-70b-chat-hf \
    --tensor-parallel-size 2 \
    --port 8003
```

### 6.2 内存预算分配

```python
def calculate_model_memory(model_size_billion, dtype="float16"):
    """估算模型显存占用（粗略）"""
    bytes_per_param = {
        "float32": 4,
        "float16": 2,
        "bfloat16": 2,
        "int8": 1,
        "int4": 0.5
    }
    # 模型参数 + KV Cache 预留
    model_memory = model_size_billion * bytes_per_param[dtype] / 1024  # GB
    kv_cache_memory = 4  # 预留
    return model_memory + kv_cache_memory

# 示例：24GB GPU 部署两个 7B 模型
llama_memory = calculate_model_memory(7, "float16")  # ~18GB
qwen_memory = calculate_model_memory(7, "float16")   # ~18GB

print(f"注意: 单个 24GB GPU 可能无法同时加载两个 7B 模型")
print("建议: 使用量化或多 GPU 部署")
```

---

## 七、总结

本篇我们学习了：
1. 多模型部署的场景和方式
2. 使用 vLLM API 服务加载多个模型
3. 多进程部署与反向代理配置
4. 模型路由与负载均衡
5. 多模型集成方法
6. GPU 资源管理策略

下一篇我们将学习 vLLM 的性能调优技巧。
