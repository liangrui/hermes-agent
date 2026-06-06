# vLLM 使用指南 09：部署篇 - 生产环境部署指南

## 一、生产环境部署概述

### 1.1 生产环境要求

生产环境部署需要考虑：
- 高可用性
- 可扩展性
- 安全性
- 监控与告警
- 日志记录
- 负载均衡

### 1.2 部署架构选择

| 场景 | 推荐架构 |
|------|----------|
| 小型应用 | 单机 + API 服务 |
| 中型应用 | 单机 + 负载均衡 + 监控 |
| 大型应用 | 集群 + 多模型 + 监控 + K8s |

---

## 二、Docker 容器化部署

### 2.1 基础 Dockerfile

```dockerfile
# 使用 NVIDIA PyTorch 镜像
FROM nvidia/cuda:12.1.0-cudnn8-runtime-ubuntu22.04

# 设置工作目录
WORKDIR /app

# 安装 Python
RUN apt-get update && apt-get install -y \
    python3.10 python3-pip -y && \
    ln -s /usr/bin/python3.10 /usr/bin/python && \
    rm -rf /var/lib/apt/lists/*

# 安装 vLLM 和依赖
RUN pip install --no-cache-dir \
    vllm==0.4.2 \
    openai \
    fastapi \
    uvicorn[standard] \
    python-multipart \
    prometheus-client \
    python-json-logger

# 设置环境变量
ENV HF_HOME=/app/models
ENV HF_HUB_CACHE=/app/models

# 创建非 root 用户
RUN useradd -m -u 1000 vllm && \
    chown -R vllm:vllm /app
USER vllm

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 启动命令
CMD ["python", "-m", "vllm.entrypoints.openai.api_server"]
```

### 2.2 docker-compose.yml 配置

```yaml
version: '3.8'

services:
  vllm-server:
    build: .
    container_name: vllm-server
    ports:
      - "8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - HF_HOME=/app/models
      - HF_HUB_ENABLE_HF_TRANSFER=1
      - VLLM_USE_MODELS_CACHE=/app/models
    volumes:
      - ./models:/app/models
      - ./logs:/app/logs
    command:
      - --model
      - TheBloke/Llama-2-7B-Chat-AWQ
      - --quantization
      - awq
      - --host
      - 0.0.0.0
      - --port
      - "8000"
      - --tensor-parallel-size
      - "1"
      - --gpu-memory-utilization
      - "0.9"
      - --max-model-len
      - "4096"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
```

---

## 三、Kubernetes 部署

### 3.1 Deployment 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-deployment
  labels:
    app: vllm
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm
  template:
    metadata:
      labels:
        app: vllm
    spec:
      containers:
      - name: vllm
        image: my-vllm-server:latest
        ports:
        - containerPort: 8000
          name: http
        resources:
          limits:
            nvidia.com/gpu: 1
          memory: "32Gi"
            cpu: "16"
          requests:
            nvidia.com/gpu: 1
            memory: "16Gi"
            cpu: "8"
        env:
        - name: HF_HOME
          value: "/models
        - name: HF_HUB_ENABLE_HF_TRANSFER
          value: "1"
        volumeMounts:
        - name: model-cache
          mountPath: /models
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: vllm-model-cache
      nodeSelector:
        gpu-type: a10g

---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
spec:
  selector:
    app: vllm
  ports:
  - protocol: TCP
    port: 8000
    targetPort: http
  type: ClusterIP

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-deployment
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-model-cache
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### 3.2 Ingress 配置

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vllm-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "100m
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  tls:
  - hosts:
    - vllm.example.com
    secretName: vllm-tls
  rules:
  - host: vllm.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vllm-service
            port:
              number: 8000
```

---

## 四、安全性配置

### 4.1 API 密钥认证

```python
# 带认证的 vLLM API 代理服务
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import APIKeyHeader
from vllm.entrypoints.openai.api_server import app as vllm_app
import uvicorn

app = FastAPI()

# API 密钥配置
API_KEYS = {"sk-your-secret-key-here",  # 生产环境应从环境变量读取

api_key_header = APIKeyHeader(name="Authorization", auto_error=False)

async def get_api_key(api_key_header: str = Depends(api_key_header)):
    if not api_key_header:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Missing API key"
        )
    if api_key_header not in [f"Bearer {k}" for k in API_KEYS]:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Invalid API key"
        )
    return api_key_header

# 挂载 vLLM 应用
app.mount("/v1", vllm_app)

@app.middleware("http")
async def auth_middleware(request, call_next):
    if request.url.path.startswith("/v1"):
        api_key = request.headers.get("Authorization")
        if not api_key or not any(api_key.startswith("Bearer "):
            return HTTPException(status_code=401, detail="Unauthorized")
        key = api_key[7:]
        if key not in API_KEYS:
            return HTTPException(status_code=403, detail="Forbidden")
    response = await call_next(request)
    return response

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 4.2 HTTPS/TLS 配置

```yaml
# 使用 cert-manager 证书配置
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: vllm-tls
spec:
  secretName: vllm-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - vllm.example.com
```

---

## 五、监控与日志

### 5.1 Prometheus 指标

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest
from fastapi import FastAPI, Response
import time

app = FastAPI()

# 指标定义
REQUEST_COUNT = Counter(
    'vllm_requests_total',
    'Total number of requests',
    ['model', 'endpoint']
)
REQUEST_LATENCY = Histogram(
    'vllm_request_duration_seconds',
    'Request latency',
    ['model', 'endpoint']
)
ACTIVE_CONNECTIONS = Gauge(
    'vllm_active_connections',
    'Number of active connections'
)
GPU_MEMORY_USAGE = Gauge(
    'vllm_gpu_memory_bytes',
    'GPU memory usage',
    ['gpu_id']
)
TOKENS_GENERATED = Counter(
    'vllm_tokens_generated_total',
    'Total tokens generated',
    ['model']
)

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(), media_type="text/plain")
```

### 5.2 Grafana 仪表板配置

```json
{
  "dashboard": {
    "panels": [
      {
        "title": "Requests per Second",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(vllm_requests_total[1m])"
          }
        ]
      },
      {
        "title": "Average Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(vllm_request_duration_seconds_bucket[5m])"
          }
        ]
      },
      {
        "title": "GPU Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "vllm_gpu_memory_bytes / 1024^3"
          }
        ]
      },
      {
        "title": "Tokens Generated",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(vllm_tokens_generated_total[1h])"
          }
        ]
      }
    ]
  }
}
```

### 5.3 结构化日志配置

```python
# logging_config.py
import logging
from pythonjsonlogger import jsonlogger

def setup_structured_logging():
    """配置结构化 JSON 日志"""
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    
    handler = logging.StreamHandler()
    formatter = jsonlogger.JsonFormatter(
        '%(asctime)s %(levelname)s %(name)s %(message)s %(module)s %(funcName)s %(lineno)d'
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    
    return logger

# 使用示例
logger = setup_structured_logging()
logger.info(
    "Request completed",
    extra={
        "model": "llama-2-7b",
        "tokens": 150,
        "latency_ms": 234.5,
        "user_id": "user-123"
    }
)
```

---

## 六、负载均衡

### 6.1 Nginx 反向代理配置

```nginx
upstream vllm_backends {
    server 127.0.0.1:8001 weight=1;
    server 127.0.0.1:8002 weight=1;
    keepalive 32;
}

server {
    listen 8000;
    server_name vllm.example.com;
    
    # 访问日志
    access_log /var/log/nginx/vllm-access.log;
    error_log /var/log/nginx/vllm-error.log;
    
    # 请求大小限制
    client_max_body_size 100M;
    
    # 超时设置
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    
    # 流式输出支持
    proxy_buffering off;
    proxy_cache off;
    
    location / {
        proxy_pass http://vllm_backends;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
    
    # 健康检查端点
    location /health {
        proxy_pass http://vllm_backends/health;
    }
}
```

---

## 七、总结

本篇我们学习了：
1. 生产环境部署架构选择
2. Docker 容器化部署
3. Kubernetes 部署配置
4. 安全认证与 HTTPS
5. Prometheus 监控与日志
6. 负载均衡配置

下一篇我们将学习最佳实践与故障排除。
