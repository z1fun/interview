# AI工程化 — 从部署到运维

> **定位**：补齐AI工程化知识（加分项），能讲清模型部署/量化/API封装/并发/MLOps的基本概念
> **前置依赖**：Docker基础、后端工程经验
> **学习目标**：能回答"大模型怎么部署""怎么优化并发""怎么做模型量化"
> [← 返回学习框架](./学习框架.md) | 相关: [面试问答.md](./面试问答.md) · [Go-Python后端.md](./Go-Python后端.md) · [RAG.md](./RAG.md)

---

## 一、AI工程化全景

```
┌──────────────────────────────────────────────────────────────┐
│                    AI 工程化闭环                               │
│                                                              │
│  开发 → 部署 → 运营 → 迭代 → 回到开发                           │
│                                                              │
│  开发层：Prompt管理、模型选型、RAG链路、Agent编排                │
│  部署层：推理服务、API封装、容器化、配置管理                      │
│  运营层：并发优化、限流降级、日志监控、成本控制                    │
│  迭代层：效果评估、数据回流、模型升级、知识库更新                   │
└──────────────────────────────────────────────────────────────┘
```

### AI工程化 vs 传统后端工程化的差异

| 维度 | 传统后端 | AI工程化 |
|------|---------|---------|
| 硬件资源 | CPU + 内存 | GPU + 显存（稀缺且昂贵） |
| 部署产物 | 代码 | 代码 + 模型权重（GB级别） |
| 性能指标 | QPS、延迟 | 额外关注：TTFT、TPOT、TPS |
| 成本结构 | 服务器 | 服务器 + GPU + API token费 |
| 版本管理 | Git管代码 | Git管代码 + 模型版本 + Prompt版本 + 知识库版本 |
| 可观测性 | 日志、指标、trace | 额外关注：token消耗、无答案率、检索质量 |

---

## 二、模块1：模型部署

### 2.1 推理框架对比

| 框架 | 类型 | 核心优化 | 适用场景 |
|------|------|---------|---------|
| **vLLM** | 生产级推理引擎 | PagedAttention + Continuous Batching | 高并发生产环境 |
| **Ollama** | 本地轻量方案 | 一键启动、自动下载模型 | 开发测试、本地运行 |
| **llama.cpp** | CPU推理 | C++实现、GGUF量化 | 无GPU场景 |
| **TGI** (Text Generation Inference) | 生产级推理引擎 | HuggingFace生态 | HF生态用户 |
| **SGLang** | 新兴推理框架 | RadixAttention、structured output | 追求极致性能 |

### 2.2 vLLM核心优化

**PagedAttention**：
- 传统方式为每个请求预分配连续的KV Cache内存 → 碎片化浪费
- PagedAttention把KV Cache分成固定大小的page（类似操作系统内存分页）
- 请求结束释放page → 内存利用率大幅提升
- 吞吐量提升2-4倍
- KV Cache（Key-Value Cache）是 Transformer 模型在**自回归生成（autoregressive generation）**过程中用来缓存中间计算结果的显存区域。

**Continuous Batching**：
- Batching 是指把多个用户的推理请求合并在一起、同时放到 GPU 上计算，以此提高 GPU 利用率和吞吐量。
- 传统batching：等一批请求全部完成才能换新的一批
- Continuous batching：有请求完成就立刻加入新请求 → GPU利用率更高

### 2.3 显存估算公式

```
显存需求 ≈ 参数量 × 精度字节数 + KV Cache + 框架开销

例子：
7B 模型 FP16: 7B × 2 bytes = 14GB + KV Cache(约2-4GB) ≈ 16-20GB
7B 模型 INT4:  7B × 0.5 bytes = 3.5GB + KV Cache ≈ 6-8GB
13B 模型 FP16: 13B × 2 bytes = 26GB + KV Cache ≈ 30-35GB
70B 模型 INT4: 70B × 0.5 bytes = 35GB + KV Cache ≈ 40-45GB
```

**GPU选型参考**：
- A10 (24GB)：7B FP16 或 13B INT4
- A100-40G (40GB)：13B FP16 或 70B INT4（勉强）
- A100-80G (80GB)：70B INT4（宽裕）或 70B FP16（需要多卡）
- 3090/4090 (24GB)：开发测试用7B模型足够

### 2.4 云API vs 私有化部署

| 维度 | 云API（DeepSeek/通义/OpenAI） | 私有化部署 |
|------|---------------------------|----------|
| 成本 | 按量付费，用量小时便宜 | 固定成本（GPU服务器），用量大时便宜 |
| 延迟 | 网络延迟不可控 | 本地推理延迟低 |
| 数据安全 | 数据出域 | 数据在本地 |
| 运维 | 零运维 | 需要运维GPU服务器 |
| 模型选择 | 只能用平台提供的模型 | 任意模型/微调版本 |

> **你的口述**："目前我们主要是用API调用——AI助理用DeepSeek API，漏洞系统也是调API。但如果业务对数据安全要求高（比如金融、政务），或者调用量大到API按量计费不划算，就会考虑私有化部署vLLM。这个决策主要看三个维度：成本、数据安全、延迟要求。"

---

## 三、模块2：模型量化

### 3.1 什么是量化？

把模型参数从高精度（FP16，16位浮点）转成低精度（INT8，8位整数 或 INT4，4位整数）。精度越低 → 显存占用越小 → 推理越快，但精度损失越大。

### 3.2 量化精度对比

| 精度 | 每个参数占用 | 7B模型显存 | 精度损失 | 使用场景 |
|------|-----------|-----------|---------|---------|
| FP32 | 4 bytes | 28 GB | 无损 | 训练 |
| FP16/BF16 | 2 bytes | 14 GB | 几乎无损 | 生产推理默认 |
| INT8 | 1 byte | 7 GB | 轻微 | 显存紧张时 |
| INT4 | 0.5 bytes | 3.5 GB | 明显 | 消费级显卡、边缘设备 |

### 3.3 量化方法

| 方法 | 原理 | 特点 |
|------|------|------|
| **GGUF** (llama.cpp) | 离线量化，多种精度等级可选 | CPU推理友好，生态最成熟 |
| **GPTQ** | 基于训练后校准数据的量化 | GPU推理，效果好 |
| **AWQ** | 基于激活值感知的量化 | 比GPTQ更快，精度类似 |
| **bitsandbytes** | 运行时量化 | 方便（HuggingFace集成），但性能略差 |

### 3.4 GGUF量化等级

```
Q8_0: 8位量化，几乎无损
Q6_K: 6位量化，质量高
Q5_K_M: 5位量化，推荐平衡点
Q4_K_M: 4位量化，推荐在消费级显卡上跑
Q3_K_M: 3位量化，极致压缩但质量下降明显
```

---

## 四、模块3：性能与并发

### 4.1 大模型服务的关键指标

| 指标 | 含义 | 重要性 |
|------|------|--------|
| **TTFT** (Time To First Token) | 首个token的响应时间 | 影响用户感知延迟 |
| **TPOT** (Time Per Output Token) | 每个输出token的生成时间 | 影响流式输出流畅度 |
| **TPS** (Tokens Per Second) | 每秒生成token数 | 衡量吞吐量 |
| **QPS** (Queries Per Second) | 每秒处理请求数 | 衡量并发能力 |

### 4.2 并发优化策略

- **异步处理**：FastAPI的async/await，不阻塞主线程
- **连接池**：复用HTTP连接，减少建连开销
- **任务队列**：高峰期请求排队，平滑流量，设置超时避免无限等待
- **Continuous Batching**（推理引擎层）：动态合并请求，提升GPU利用率

### 4.3 SSE流式输出实现

```python
# FastAPI SSE流式输出
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

@app.post("/chat/stream")
async def chat_stream(message: str):
    async def generate():
        async for token in llm.generate_stream(message):
            yield f"data: {json.dumps({'token': token})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # 禁用Nginx缓冲
        }
    )
```

```go
// Go SSE流式输出（如果你用Go做推理网关）
func chatStreamHandler(w http.ResponseWriter, r *http.Request) {
    flusher, _ := w.(http.Flusher)
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")

    for token := range llmStream {
        fmt.Fprintf(w, "data: {\"token\": \"%s\"}\n\n", token)
        flusher.Flush()
    }
    fmt.Fprintf(w, "data: [DONE]\n\n")
    flusher.Flush()
}
```

### 4.4 限流方案

```
┌─────────┐     ┌──────────────┐     ┌─────────┐
│  用户请求  │ →  │ 令牌桶/滑动窗口  │ →  │  LLM服务  │
└─────────┘     │  放行/排队/拒绝  │     └─────────┘
                └──────────────┘
```

**分层限流**：
1. 全局限流：保护LLM服务不被冲垮（令牌桶，固定QPS）
2. 用户级限流：防止单用户滥用（每用户每分钟N次）
3. Token预算：每用户每天最多消耗X token
4. 降级策略：超过限流时用小模型替代或返回缓存结果

### 4.5 语义缓存

相似问题不重复调LLM，从缓存返回：
1. 把历史问答的问题做Embedding
2. 新问题来，先做向量相似度搜索
3. 相似度 > 阈值（如0.95）→ 直接返回缓存答案
4. 节省成本和延迟

---

## 五、模块4：API封装

### 5.1 API设计原则

- **OpenAI兼容协议**：大多数工具（LangChain、Dify等）默认兼容OpenAI的API格式，让服务更容易集成
- **统一模型网关**：前面一个网关接收请求，后面根据请求内容路由到不同模型（复杂任务→大模型，简单任务→小模型）
- **结构化输出**：用JSON mode或Function Calling确保输出可解析
- **密钥管理**：API密钥放环境变量/密钥管理系统，支持轮换

### 5.2 链路追踪

每个请求贯穿唯一的trace_id，串联：
```
用户请求(trace_id=xxx)
  → RAG检索（query改写→向量检索→重排）
  → LLM调用（model/deployment/tokens/duration）
  → 后处理（格式校验/引用标注）
  → 返回
```

便于排查问题——哪个环节慢了、错了、消耗了过多token。

---

## 六、模块5：MLOps基础

### 6.1 MLOps vs DevOps

| 维度 | DevOps | MLOps |
|------|--------|-------|
| 版本管理 | 代码 | 代码 + 数据 + 模型 + Prompt |
| 测试 | 单元测试/集成测试 | 额外：模型效果评估、数据质量检查 |
| 部署 | 代码部署 | 代码 + 模型部署 |
| 监控 | CPU/内存/QPS/错误率 | 额外：token消耗、模型效果、数据漂移 |
| 回滚 | 代码回滚 | 代码 + 模型 + 知识库回滚 |

### 6.2 LLM场景下的MLOps关注点

- **Prompt版本管理**：Git管理Prompt模板，提交时注明改动原因
- **知识库版本管理**：知识库更新要有版本号，可回溯
- **离线评估**：改Prompt/改检索策略后先在评估集上跑分，确认不退步再上线
- **线上监控**：无答案率、用户踩/赞比、响应延迟、token成本
- **数据回流**：用户反馈→标注→更新知识库→离线评估→上线

---

## 七、模块6：云原生部署

### 7.1 Docker部署AI服务

```dockerfile
# AI服务的Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/vulndb
      - LLM_API_KEY=${LLM_API_KEY}
    depends_on:
      - db
    # GPU支持
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: vulndb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### 7.2 K8s概念速览

| 概念 | 作用 | 类比 |
|------|------|------|
| Pod | 最小部署单元（一个或多个容器） | Docker容器 |
| Deployment | 管理Pod的副本数和更新策略 | docker-compose service |
| Service | 给Pod提供稳定的网络入口 | 负载均衡 |
| Ingress | 外部访问入口，路由到Service | Nginx反向代理 |

### 7.3 Serverless

云函数按需启动、按调用计费。适合不定时触发的AI任务（如定时报表生成），不适合需要常驻的在线推理服务（冷启动延迟高）。

---

## 八、面试高频题 + 自查清单

### 面试高频题（答案详见 [面试问答.md](./面试问答.md) §13）

1. 大模型私有化部署有哪些方案？
2. vLLM的PagedAttention是什么原理？
3. 模型量化是什么？FP16/INT8/INT4的区别？
4. 怎么估算模型需要多少显存？
5. 大模型API服务怎么做限流？
6. SSE和WebSocket怎么选？
7. Docker怎么部署带GPU的AI服务？
8. MLOps做什么？和DevOps有什么不同？
9. Prompt版本管理怎么做？
10. AI服务上线前要做哪些检查？

### 自查清单

- [ ] 能讲清vLLM/Ollama/llama.cpp的适用场景差异
- [ ] 能口算7B/13B/70B模型各需要多少显存
- [ ] 能写一个FastAPI SSE流式输出的代码骨架
- [ ] 能讲清限流的分层策略
- [ ] 能解释语义缓存的原理
- [ ] 能写一个docker-compose编排AI服务+向量数据库
- [ ] 能列举LLM场景下MLOps的四个关键关注点

---

> [← 返回学习框架](./学习框架.md)
