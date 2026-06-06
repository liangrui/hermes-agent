# vLLM 使用指南 03：API 篇 - OpenAI 兼容 API 服务

## 一、启动 API 服务器

### 1.1 命令行启动

#### 基础启动
```bash
# 使用 Python 模块启动
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --host 0.0.0.0 \
    --port 8000
```

#### 完整配置启动
```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --tokenizer meta-llama/Llama-2-7b-chat-hf \
    --host 0.0.0.0 \
    --port 8000 \
    --tensor-parallel-size 1 \
    --gpu-memory-utilization 0.9 \
    --max-model-len 4096 \
    --dtype auto \
    --quantization awq \
    --trust-remote-code
```

### 1.2 使用 Docker 启动

```bash
docker run --gpus all \
    -p 8000:8000 \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    vllm/vllm-openai:latest \
    --model meta-llama/Llama-2-7b-chat-hf \
    --host 0.0.0.0 \
    --port 8000
```

### 1.3 命令行参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--model` | 模型名称或路径 | 必填 |
| `--host` | 服务主机地址 | `0.0.0.0` |
| `--port` | 服务端口 | `8000` |
| `--tensor-parallel-size` | 张量并行大小 | `1` |
| `--gpu-memory-utilization` | GPU 内存利用率 | `0.9` |
| `--max-model-len` | 最大序列长度 | `None` |
| `--dtype` | 数据类型（auto/float16/bfloat16/float32） | `auto` |
| `--quantization` | 量化类型（awq/gptq/squeezellm） | `None` |
| `--trust-remote-code` | 信任远程代码 | `False` |
| `--api-key` | API 密钥 | `None` |

---

## 二、使用 OpenAI SDK 调用

### 2.1 基础调用

```python
from openai import OpenAI

# 初始化客户端
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"  # vLLM 默认不需要 API key
)

# 文本补全
completion = client.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    prompt="Hello, my name is",
    max_tokens=100,
    temperature=0.7
)

print(completion.choices[0].text)
```

### 2.2 聊天补全（Chat Completions）

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

response = client.chat.completions.create(
    model="meta-llama/Llama-2-7b-chat-hf",
    messages=[
        {"role": "system", "content": "你是一个乐于助人的助手。"},
        {"role": "user", "content": "请解释一下什么是机器学习？"}
    ],
    temperature=0.7,
    max_tokens=512
)

print(response.choices[0].message.content)
```

### 2.3 流式输出

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

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

---

## 三、使用 HTTP 请求直接调用

### 3.1 文本补全 API

```python
import requests

url = "http://localhost:8000/v1/completions"

payload = {
    "model": "meta-llama/Llama-2-7b-chat-hf",
    "prompt": "Hello, my name is",
    "max_tokens": 100,
    "temperature": 0.7,
    "top_p": 0.9,
    "n": 1
}

response = requests.post(url, json=payload)
result = response.json()

print(result["choices"][0]["text"])
```

### 3.2 聊天补全 API

```python
import requests

url = "http://localhost:8000/v1/chat/completions"

payload = {
    "model": "meta-llama/Llama-2-7b-chat-hf",
    "messages": [
        {"role": "system", "content": "你是一个代码助手。"},
        {"role": "user", "content": "写一个 Python 快速排序函数"}
    ],
    "temperature": 0.5,
    "max_tokens": 500
}

response = requests.post(url, json=payload)
result = response.json()

print(result["choices"][0]["message"]["content"])
```

### 3.3 流式 HTTP 请求

```python
import requests
import json

url = "http://localhost:8000/v1/chat/completions"

payload = {
    "model": "meta-llama/Llama-2-7b-chat-hf",
    "messages": [{"role": "user", "content": "讲故事"}],
    "stream": True,
    "temperature": 0.8,
    "max_tokens": 300
}

response = requests.post(url, json=payload, stream=True)

for line in response.iter_lines():
    if line:
        line = line.decode("utf-8")
        if line.startswith("data: "):
            data = line[6:]
            if data != "[DONE]":
                try:
                    chunk = json.loads(data)
                    if "choices" in chunk and len(chunk["choices"]) > 0:
                        delta = chunk["choices"][0].get("delta", {})
                        if "content" in delta:
                            print(delta["content"], end="", flush=True)
                except json.JSONDecodeError:
                    pass
```

---

## 四、多模型服务

### 4.1 同时加载多个模型

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --served-model-name llama-2-7b-chat \
    --model Qwen/Qwen-7B-Chat \
    --served-model-name qwen-7b-chat \
    --host 0.0.0.0 \
    --port 8000
```

### 4.2 调用指定模型

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

# 调用 Llama 2
response1 = client.chat.completions.create(
    model="llama-2-7b-chat",
    messages=[{"role": "user", "content": "Hello"}]
)

# 调用 Qwen
response2 = client.chat.completions.create(
    model="qwen-7b-chat",
    messages=[{"role": "user", "content": "你好"}]
)
```

---

## 五、高级配置

### 5.1 设置 API 密钥认证

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --api-key your-secret-key-here \
    --host 0.0.0.0 \
    --port 8000
```

```python
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="your-secret-key-here"
)
```

### 5.2 使用 SSL/TLS

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --ssl-keyfile /path/to/key.pem \
    --ssl-certfile /path/to/cert.pem \
    --host 0.0.0.0 \
    --port 8000
```

```python
client = OpenAI(
    base_url="https://localhost:8000/v1",
    api_key="not-needed"
)
```

### 5.3 配置请求限制

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-2-7b-chat-hf \
    --max-num-batched-tokens 8192 \
    --max-num-seqs 256 \
    --host 0.0.0.0 \
    --port 8000
```

---

## 六、构建 Web 应用示例

### 6.1 Flask 代理服务

```python
from flask import Flask, request, jsonify, Response
from openai import OpenAI
import json

app = Flask(__name__)

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

@app.route("/api/chat", methods=["POST"])
def chat():
    data = request.json
    messages = data.get("messages", [])
    stream = data.get("stream", False)
    
    def generate():
        stream_response = client.chat.completions.create(
            model="meta-llama/Llama-2-7b-chat-hf",
            messages=messages,
            stream=True,
            temperature=0.7,
            max_tokens=512
        )
        for chunk in stream_response:
            if chunk.choices[0].delta.content:
                yield f"data: {json.dumps({'content': chunk.choices[0].delta.content})}\n\n"
    
    if stream:
        return Response(generate(), mimetype="text/event-stream")
    else:
        response = client.chat.completions.create(
            model="meta-llama/Llama-2-7b-chat-hf",
            messages=messages,
            temperature=0.7,
            max_tokens=512
        )
        return jsonify({
            "content": response.choices[0].message.content
        })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### 6.2 简单的 HTML 聊天界面

```html
<!DOCTYPE html>
<html>
<head>
    <title>vLLM 聊天助手</title>
    <style>
        .chat-container { max-width: 800px; margin: 0 auto; padding: 20px; }
        .message { padding: 10px; margin: 10px 0; border-radius: 8px; }
        .user { background-color: #e3f2fd; text-align: right; }
        .assistant { background-color: #f5f5f5; }
        #input { width: 100%; padding: 10px; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="chat-container">
        <div id="chat"></div>
        <input type="text" id="input" placeholder="输入消息..." onkeypress="handleKeyPress(event)">
    </div>
    <script>
        const chatDiv = document.getElementById('chat');
        const input = document.getElementById('input');
        let messages = [];

        function addMessage(role, content) {
            const div = document.createElement('div');
            div.className = `message ${role}`;
            div.textContent = content;
            chatDiv.appendChild(div);
            chatDiv.scrollTop = chatDiv.scrollHeight;
        }

        async function sendMessage() {
            const content = input.value.trim();
            if (!content) return;
            
            input.value = '';
            addMessage('user', content);
            messages.push({ role: 'user', content });
            
            const responseDiv = document.createElement('div');
            responseDiv.className = 'message assistant';
            chatDiv.appendChild(responseDiv);
            
            const response = await fetch('/api/chat', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ messages, stream: true })
            });
            
            const reader = response.body.getReader();
            const decoder = new TextDecoder();
            let aiContent = '';
            
            while (true) {
                const { done, value } = await reader.read();
                if (done) break;
                const chunk = decoder.decode(value);
                const lines = chunk.split('\n');
                for (const line of lines) {
                    if (line.startsWith('data: ')) {
                        const data = JSON.parse(line.slice(6));
                        if (data.content) {
                            aiContent += data.content;
                            responseDiv.textContent = aiContent;
                        }
                    }
                }
            }
            
            messages.push({ role: 'assistant', content: aiContent });
        }

        function handleKeyPress(event) {
            if (event.key === 'Enter') sendMessage();
        }
    </script>
</body>
</html>
```

---

## 七、API 参考

### 7.1 模型列表 API

```bash
curl http://localhost:8000/v1/models
```

### 7.2 健康检查

```bash
curl http://localhost:8000/health
```

---

## 八、总结

本篇我们学习了：
1. 如何启动 vLLM 的 OpenAI 兼容 API 服务
2. 使用 OpenAI SDK 进行调用
3. 使用 HTTP 请求直接调用 API
4. 多模型服务配置
5. 高级配置（API 密钥、SSL 等）
6. 构建 Web 应用示例

下一篇我们将学习 vLLM 的高效批量推理技巧。
