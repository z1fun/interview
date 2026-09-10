# Go + Python 后端巩固

> **定位**：Go巩固（你的优势） + Python补齐（AI栈日常语言），双语言协同
> **前置依赖**：5年Go经验 + FastAPI实战
> **学习目标**：Go高频面试题对答如流、Python能讲清asyncio/FastAPI/GIL
> [← 返回学习框架](./学习框架.md) | 相关: [面试问答.md](./面试问答.md) · [AI工程化.md](./AI工程化.md)

---

## 一、双语言定位

### 你的差异化叙事

```
JD说："熟练Python/Go后端开发，会基础的go语言"

你的回应策略：
- Go不是"会基础"，而是5年主力语言（这是稀缺性！）
- Python在AI栈上够用（FastAPI + LangGraph + Pydantic 实战经验）
- 双语言协同："AI推理链路用Python（生态），高并发业务层用Go（性能）"
```

### 什么场景用什么语言？

| 场景 | 语言 | 理由 |
|------|------|------|
| LLM调用、RAG链路、Agent编排 | Python | AI生态（LangChain/LangGraph/LlamaIndex都是Python） |
| 高并发API网关 | Go | goroutine + 低延迟 |
| 推理服务网关/限流/负载均衡 | Go | 并发性能 + 部署简单（单二进制） |
| 数据管道/ETL | 看情况 | Go快但Python库多 |
| 微服务业务层 | Go | go-zero/gRPC生态成熟 |
| 快速原型/Dify工作流 | Python | FastAPI开发效率高 |

---

## 二、Go巩固（快刷，重点关注高频题）

### 2.1 GMP并发模型 ⭐⭐⭐

```
G (Goroutine) — 用户态轻量级协程，一个程序可创建成千上万个
M (Machine)   — 操作系统线程，GOMAXPROCS决定M的数量
P (Processor) — 逻辑处理器，维护本地G队列

调度流程：
1. 每个P有一个本地G队列
2. M绑定P来执行G
3. M执行完当前G后从P的本地队列取下一个
4. 本地队列空 → 去全局队列或别的P偷一半（work stealing）
5. G阻塞（系统调用）→ M和P解绑，P找空闲M继续执行
```

> **口述要点**：强调"work stealing做自动负载均衡"和"上下文切换在用户态、成本低"

### 2.2 Channel核心

```go
// 无缓冲 — 同步
ch := make(chan int)
go func() { ch <- 42 }()  // 阻塞直到有人收
val := <-ch                // 阻塞直到有人发

// 有缓冲 — 异步
ch := make(chan int, 10)  // 10个缓冲位
ch <- 1                     // 不阻塞（缓冲没满）
ch <- 2
close(ch)                   // 关闭后不能再发但可以继续收
for v := range ch { ... }  // 收完自动退出

// select — 多路监听
select {
case msg := <-ch1:
    handle(msg)
case ch2 <- data:
    // 发送成功
case <-time.After(3 * time.Second):
    // 超时
case <-ctx.Done():
    // context取消
default:
    // 非阻塞（所有channel都不就绪时执行）
}
```

### 2.3 Context最佳实践

```go
// 传递取消信号和超时
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// 在goroutine间传递请求级别的值（traceID、userID）
ctx = context.WithValue(ctx, "traceID", "abc-123")

// 原则：
// 1. context作为函数的第一个参数
// 2. 不要存到结构体里
// 3. 不要传nil，用context.TODO()
// 4. 不要把可变对象放到value里
```

### 2.4 内存与GC

```
三色标记+并发清扫：
- 白色：潜在垃圾
- 灰色：正在扫描
- 黑色：确认存活

流程：根对象标记灰色 → 扫描灰色引用的白色标灰 → 灰标黑 → 重复 → 剩余白色回收

GOGC参数控制GC触发时机：默认100（内存翻倍时触发GC）
GC STW通常在100微秒以内
```

### 2.5 逃逸分析

变量本可以分配在栈上（函数返回自动回收），但以下情况会"逃逸"到堆（需要GC）：
- 返回局部变量的指针
- 变量大小编译时不确定（如make([]int, n)的n是变量）
- 被闭包引用
- interface类型变量
- 往channel发送指针

> 用 `go build -gcflags='-m'` 查看逃逸分析。不需要刻意避免——正常代码的逃逸是可接受的。

### 2.6 常见高频速查

| 考点 | 一句话答案 |
|------|-----------|
| defer执行顺序 | LIFO（后进先出），参数在声明时求值 |
| nil slice vs empty slice | nil的底层数组指针为nil（JSON序列化null），empty的指针不为nil（JSON序列化[]） |
| map遍历无序 | 底层哈希表+刻意随机起始bucket |
| interface判nil | 类型和值两个指针都为nil才算nil |
| slice扩容 | Go 1.18+: cap<256时翻倍，大于256时渐变增长约1.25倍 |
| GORM事务 | db.Transaction(func(tx *gorm.DB) error {...})自动提交/回滚 |
| go-zero | 集API+RPC+ETCD注册发现+限流熔断的微服务框架 |
| pprof排障 | import _ "net/http/pprof"，/debug/pprof/heap看内存，/goroutine看goroutine泄漏 |

### 2.7 手写题预案

```go
// 扇出扇入模式 — 并发处理+聚合结果
func fanOutFanIn(inputs []string, worker func(string) int) []int {
    results := make(chan int, len(inputs))
    for _, input := range inputs {
        go func(in string) {
            results <- worker(in)
        }(input)
    }
    var output []int
    for i := 0; i < len(inputs); i++ {
        output = append(output, <-results)
    }
    return output
}

// 超时控制
func doWithTimeout(ctx context.Context, fn func() error) error {
    errCh := make(chan error, 1)
    go func() { errCh <- fn() }()
    select {
    case err := <-errCh:
        return err
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

---

## 三、Python补齐（AI栈重点）

### 3.1 GIL与并发策略 ⭐

**GIL（全局解释器锁）**：同一时刻只有一个线程执行Python字节码。

```
CPU密集型 → 多线程无效（GIL竞争）→ 用多进程(multiprocessing)或C扩展(numpy)
I/O密集型 → 多线程有用（GIL在I/O时释放）→ 或用asyncio协程
高并发I/O → asyncio（单线程事件循环，切换成本极低）
```

**选型决策表**：

| 场景 | 方案 | 说明 |
|------|------|------|
| FastAPI Web服务 | asyncio | 处理成千上万并发连接 |
| 调用外部API | asyncio + httpx/aiohttp | 异步HTTP请求 |
| 阻塞库（如某些老ORM） | run_in_executor | 把阻塞调用扔到线程池 |
| 并行跑多个模型推理 | multiprocessing | 每个进程独立GIL |
| LLM调用 | asyncio | API调用是I/O密集 |

### 3.2 asyncio核心

```python
import asyncio

# 协程函数
async def fetch_data(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)  # await = 挂起当前协程，让事件循环跑别的
        return response.json()

# 并发执行多个协程
async def fetch_all(urls: list[str]) -> list[dict]:
    tasks = [fetch_data(url) for url in urls]
    return await asyncio.gather(*tasks)  # 所有协程并发跑，全部完成再返回

# 事件循环在FastAPI里是隐式的——你写async def路由，框架自动调度
```

> **关键理解**：async不等于多线程——所有协程在同一个线程里。阻塞操作会卡死整个事件循环。你的async函数里不能有time.sleep()，要用await asyncio.sleep()。

### 3.3 FastAPI深度

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# Pydantic模型 = 自动校验 + 自动文档
class ChatRequest(BaseModel):
    message: str
    conversation_id: Optional[str] = None
    stream: bool = False

class ChatResponse(BaseModel):
    answer: str
    sources: list[dict] = []

# 依赖注入 — 复用认证/数据库连接/配置
async def get_db():
    db = await create_db_connection()
    try:
        yield db
    finally:
        await db.close()

@app.post("/chat", response_model=ChatResponse)
async def chat(
    request: ChatRequest,
    db = Depends(get_db)  # 自动注入
):
    if not request.message.strip():
        raise HTTPException(status_code=400, detail="消息不能为空")
    # ... RAG链路
    return ChatResponse(answer="...", sources=[...])
```

**FastAPI核心优势**：
1. 自动OpenAPI文档（/docs）
2. Pydantic数据校验
3. 原生async/await
4. 依赖注入系统
5. 性能接近Node.js（基于Starlette）

### 3.4 FastAPI vs Gin对比

| 维度 | FastAPI | Gin |
|------|---------|-----|
| 语言 | Python | Go |
| 性能 | 接近Node.js（够用） | 高（Go原生并发） |
| 开发效率 | 高（动态语言+自动文档） | 中（编译语言+写法啰嗦） |
| 类型安全 | Pydantic运行时校验 | 编译时类型检查 |
| AI生态 | 极好（LangChain/LlamaIndex都是Python） | 差 |
| 部署 | uvicorn+gunicorn | 单二进制 |
| 适用场景 | AI服务、快速原型 | 高并发业务、微服务 |

### 3.5 Python工程化

```python
# 项目结构
project/
├── api/           # FastAPI路由层
│   ├── __init__.py
│   └── routes/
├── services/      # 业务逻辑层（Agent编排、RAG链路）
├── models/        # Pydantic数据模型
├── db/            # 数据库操作
├── tools/         # Agent工具函数
├── config.py      # Pydantic Settings配置管理
└── main.py        # 入口
```

**配置管理**：
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    llm_api_key: str
    llm_model: str = "deepseek-chat"
    max_tokens: int = 4096

    class Config:
        env_file = ".env"

settings = Settings()  # 自动从环境变量/.env读取并做类型校验
```

**日志**：用structlog做结构化日志（每行JSON格式），方便后续对接日志系统。

---

## 四、数据库与中间件速查

### 4.1 MySQL vs PostgreSQL

| 维度 | MySQL | PostgreSQL |
|------|-------|-----------|
| 复杂查询 | 一般 | 优秀（CTE/窗口函数/递归查询） |
| JSON支持 | 一般 | 优秀（JSONB + GIN索引） |
| 扩展能力 | 弱 | 强（pgvector/pg_cron/PostGIS等） |
| 简单读写 | 快 | 差不多 |
| AI项目适配 | 弱（没有向量扩展） | 强（pgvector原生向量检索） |

> AI项目普遍选PG——pgvector让业务库同时处理向量检索，运维简单。

### 4.2 Redis常见模式

```
String → 缓存对象、计数器、分布式锁(SETNX)
Hash   → 对象缓存（部分更新，不用整体序列化）
List   → 消息队列(LPUSH/RPOP)、最新N条
Set    → 标签、去重、共同好友
ZSet   → 排行榜、延迟队列(score=执行时间戳)
```

### 4.3 RabbitMQ回顾

你的经验：
- POS数据中台：异步上传（解耦）、手动ACK保证不丢
- 商云前台：Topic交换机按门店路由、双向通信

---

## 五、系统设计答题框架

### 四步法

```
1. 场景分析 → 这个系统解决什么问题？用户是谁？核心功能是什么？
2. 约束明确 → QPS多少？延迟要求？数据量？一致性要求？
3. 架构设计 → 画架构图（前端→API网关→服务→存储），技术选型并解释why
4. 扩展讨论 → 性能瓶颈在哪？怎么扩容？监控怎么做？有哪些trade-off？
```

### 演练：设计一个API限流系统

1. 场景：保护后端服务不被流量冲垮
2. 约束：单用户级+全局级、实时性要求高
3. 架构：网关层(Nginx限流) → Redis存储计数 → 令牌桶算法 → 429+Retry-After
4. 扩展：降级用小模型、排队机制、监控仪表盘

---

## 六、面试高频题 + 自查清单

### Go高频题

1. GMP模型怎么工作的？（见§2.1）
2. Channel底层原理？（见[面试问答.md](./面试问答.md) Q10-2）
3. GC三色标记？（见§2.4）
4. 逃逸分析？（见§2.5）
5. defer执行顺序和坑？（见[面试问答.md](./面试问答.md) Q10-5）
6. slice扩容机制？（见[面试问答.md](./面试问答.md) Q10-7）
7. Context的作用？（见§2.3）
8. go-zero微服务特点？（见[面试问答.md](./面试问答.md) Q10-12）
9. GORM事务？（见[面试问答.md](./面试问答.md) Q10-11）
10. pprof排查？（见[面试问答.md](./面试问答.md) Q10-14）

### Python高频题

1. GIL是什么？（见§3.1）
2. asyncio vs 多线程 vs 多进程？（见§3.1）
3. 装饰器？（见[面试问答.md](./面试问答.md) Q11-3）
4. FastAPI依赖注入？（见§3.3）
5. Pydantic怎么用？（见[面试问答.md](./面试问答.md) Q11-6）
6. FastAPI vs Gin？（见§3.4）
7. Python类型注解？（见[面试问答.md](./面试问答.md) Q11-12）

### 自查清单

Go：
- [ ] 能画出GMP调度模型图
- [ ] 能解释work stealing机制
- [ ] 能写出带超时的select模式
- [ ] 能讲清逃逸分析的常见场景
- [ ] 能讲出go-zero微服务架构的核心组件

Python：
- [ ] 能讲清GIL的影响和在什么场景不受影响
- [ ] 能对比asyncio/多线程/多进程的适用场景
- [ ] 能手写一个简单的装饰器
- [ ] 能写出FastAPI SSE流式输出的代码骨架
- [ ] 能用一句话讲清FastAPI和Gin的选型

---

> [← 返回学习框架](./学习框架.md)
