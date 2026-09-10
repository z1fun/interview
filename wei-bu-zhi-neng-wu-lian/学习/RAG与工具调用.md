# RAG 与 Function Call 工具调用

---

## 一、通用 RAG vs 个性化 RAG

| 维度 | 通用 RAG | 个性化 RAG（数字人场景） |
|------|---------|----------------------|
| 检索范围 | 全局知识库 | 单个用户的知识库 |
| 检索目标 | 事实准确性 | 个性化相关性 + 情感贴合 |
| 排序策略 | 语义相似度 | 语义 + 时间衰减 + 重要性 + 情感 |
| 隔离需求 | 无 | **多租户严格隔离** |
| 更新频率 | 批量更新 | 每次对话后异步更新 |
| 遗忘机制 | 通常不需要 | **必需**（用户请求 / 合规 / 时间衰减） |

### 通用 RAG 流程

```
文档 → Chunk 分割 → Embedding → 向量数据库
查询 → Embedding → 语义搜索 → Top-K 召回 → 注入 LLM 上下文
```

### 个性化 RAG 流程（数字人）

```
用户对话 → 信息提取 (LLM) → 分类 (事实/偏好/事件/情感)
        → Embedding → 写入用户分区 (pgvector)
        
新对话 → 查询 Embedding → 语义搜索 + 时间衰减 + 重要性加权
      → Top-K 召回 → 注入上下文（与 System Prompt + 对话历史合并）
```

---

## 二、关键技术细节

### 2.1 多租户隔离

```sql
-- 分区键隔离：每个用户的数据物理独立
CREATE TABLE user_memories (
    id UUID PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,    -- 分区键
    content TEXT NOT NULL,
    embedding VECTOR(1536) NOT NULL,
    category VARCHAR(32),            -- fact / preference / event / emotion
    importance FLOAT DEFAULT 0.5,
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP
);

-- 索引按用户过滤
CREATE INDEX ON user_memories 
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
    
-- 查询时强制用户隔离
SELECT content, 1 - (embedding <=> $1) AS similarity
FROM user_memories
WHERE user_id = $2                         -- 必须按用户过滤
ORDER BY embedding <=> $1
LIMIT $3;
```

### 2.2 时间衰减 + 重要性加权

```python
def score_memory(memory, query_embedding, current_time):
    """
    score = 语义相似度 × 时间衰减 × 重要性
    """
    similarity = cosine_similarity(query_embedding, memory.embedding)
    
    days_elapsed = (current_time - memory.created_at).days
    decay = math.exp(-0.01 * days_elapsed)  # 衰减系数 0.01
    
    importance = memory.importance  # 0.0 ~ 1.0
    
    return similarity * decay * importance
```

### 2.3 情感化重排序

数字人场景特有：不仅检索"相关的"，还要检索"合适的"。

```python
def emotional_rerank(memories, user_current_emotion):
    """
    根据用户当前情绪调整排序
    - 用户难过 → 优先展示过去的温暖互动
    - 用户开心 → 优先展示共同庆祝的记忆
    """
    for mem in memories:
        if user_current_emotion == "负面":
            if mem.emotion_tag == "温暖":
                mem.score *= 1.5  # 提升温暖记忆
            elif mem.emotion_tag == "负面":
                mem.score *= 0.5  # 降低负面记忆（不要雪上加霜）
    return sorted(memories, key=lambda m: m.score, reverse=True)
```

### 2.4 检索失败的降级策略

```
检索结果 > 0 条 → 正常注入
检索结果 = 0 条 → 降级策略：
  1. 扩大检索范围（减少过滤条件）
  2. 返回通用上下文（用户画像摘要）
  3. 不注入记忆，仅依赖 System Prompt + 对话历史
```

---

## 三、Function Call（工具调用）

### 3.1 JSON Schema 工具定义

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_user_memory",
            "description": "检索用户的长期记忆，用于个性化对话",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "用自然语言描述你想查找什么记忆"
                    },
                    "category": {
                        "type": "string",
                        "enum": ["fact", "preference", "event", "emotion"],
                        "description": "记忆类型过滤"
                    },
                    "limit": {
                        "type": "integer",
                        "default": 5,
                        "minimum": 1,
                        "maximum": 10
                    }
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的实时天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名"}
                },
                "required": ["city"]
            }
        }
    }
]
```

### 3.2 并发工具执行

当多个工具调用相互独立时，可以并发执行以减少延迟：

```python
# 并发：查天气 + 查记忆（互不依赖）
async def concurrent_tool_calls(tool_calls):
    tasks = []
    for tc in tool_calls:
        if tc.name == "search_user_memory":
            tasks.append(search_memory_async(tc.arguments))
        elif tc.name == "get_weather":
            tasks.append(get_weather_async(tc.arguments))
    
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results

# 串行：查天气的结果影响后续决策
def sequential_tool_calls(tool_calls):
    results = []
    for tc in tool_calls:
        result = execute_tool(tc)
        results.append(result)
        # 下一个工具调用可能依赖当前结果
    return results
```

### 3.3 参数校验 + 超时降级

```python
def execute_tool_with_safety(tool_call, timeout=10):
    """
    工具调用需要参数校验和超时保护
    """
    # 1. 参数校验
    try:
        args = validate_and_parse(tool_call.name, tool_call.arguments)
    except ValidationError as e:
        return {"error": f"参数校验失败: {e}"}
    
    # 2. 超时控制
    try:
        result = asyncio.wait_for(
            execute_tool(tool_call.name, args),
            timeout=timeout
        )
        return result
    except asyncio.TimeoutError:
        return {"error": f"工具 {tool_call.name} 执行超时 (> {timeout}s)"}
    except Exception as e:
        return {"error": f"工具执行异常: {str(e)}"}
```

### 3.4 工具调用在数字人对话中的位置

```
用户输入: "今天适合出门吗？"
         │
         ▼
LLM 推理: 用户想了解天气 + 可能记起了什么
         │
    ┌────┴────┐
    ▼         ▼
get_weather search_user_memory
  ("深圳")   ("用户喜欢户外活动")
    │         │
    └────┬────┘
         ▼
合并结果 → LLM 二次推理:
  "今天深圳 25°C 晴天，我记得你最喜欢爬山了，
   要不要去梧桐山走走？上次你说想去但一直没去成~"
```

---

## 四、RAG 在记忆系统中的定位

```
                    ┌─────────────────┐
                    │   新对话开始      │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  加载核心记忆     │ ← System Prompt（不变的）
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  加载短期记忆     │ ← Redis（当前会话摘要）
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  RAG 检索长期记忆 │ ← pgvector（个性化 RAG）
                    │  (语义+衰减+重要) │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  组装上下文       │
                    │  System Prompt   │
                    │  + RAG 结果      │
                    │  + 最近 N 轮对话  │
                    │  + 工具调用结果   │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │  LLM 推理 → 回复 │
                    └─────────────────┘
```

---

## 五、你的项目实践映射

**AI 漏洞研判系统中的 RAG**：

> "在漏洞研判系统中，我用 PostgreSQL + pgvector 搭建了历史漏洞知识库的 RAG 检索。当一个新漏洞提交进来，系统会检索历史上相似的漏洞——看它们的定级、定价、处理结论——作为当前漏洞的参考。技术上用了 pgvector 的 cosine 相似度搜索，并且加上了**时间衰减**（新漏洞参考价值更高）和**项目隔离**（不同项目的漏洞互不可见）。
>
> 这套 RAG 链路和数字人的长期记忆检索是同一套技术栈。区别在于：数字人场景需要额外做情感化重排序和更强的隐私隔离，但底层的 embedding + 向量检索 + 时间衰减 + 多租户隔离是完全一致的。"

---

## 今晚记忆要点

1. 个性化 RAG vs 通用 RAG：多了多租户隔离、时间衰减、重要性加权、情感化重排序、遗忘机制
2. 记忆检索排序公式：语义相似度 × e^(-λ×Δt) × 重要性
3. Function Call 三件套：JSON Schema 定义 + 并发执行 + 超时降级
4. 检索失败降级：扩大范围 → 通用画像 → 仅依赖对话历史
5. 情感化重排序：用户难过时提升温暖记忆、降低负面记忆
6. 你的映射：pgvector RAG = 长期记忆层；时间衰减 = 实际用过；项目隔离 = 多租户雏形
