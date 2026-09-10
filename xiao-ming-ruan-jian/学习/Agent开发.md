# Agent 智能体开发 — 系统学习

> **定位**：把你的LangGraph实战经验升华为系统化理论，补齐LangChain/LlamaIndex广度
> **前置依赖**：AI漏洞研判系统（LangGraph 5-Agent） + AI助理（Dify）
> **学习目标**：能讲清Agent vs Workflow选型、能解释LangGraph核心机制、能设计多Agent协作方案
> [← 返回学习框架](./学习框架.md) | 相关: [面试问答.md](./面试问答.md) · [RAG.md](./RAG.md) · [独立产品交付.md](./独立产品交付.md)

---

## 一、Agent核心概念

### 1.1 什么是Agent？

Agent（智能体）是一个能**自主感知、推理、决策、执行**的AI系统。

```
┌──────────────────────────────────────────────┐
│                 Agent 循环                     │
│                                              │
│  感知(Observe) → 思考(Think) → 行动(Act)      │
│       ↑                            ↓          │
│       └────── 观察结果 ←────────────┘          │
│                                              │
│  直到目标达成或达到终止条件                        │
└──────────────────────────────────────────────┘
```

### 1.2 Agent vs Workflow（工作流）⭐必考题

| 维度 | Workflow（工作流） | Agent（智能体） |
|------|-------------------|----------------|
| 控制方式 | 预定义的确定性流程 | 模型自主决策 |
| 执行路径 | 固定（A→B→C） | 动态（根据情况选下一步） |
| 灵活性 | 低（改流程要改代码） | 高（能处理未预见的情况） |
| 可预测性 | 高（每次执行路径相同） | 低（同样输入可能走不同路径） |
| 成本 | 低 | 高（多次推理+工具调用） |
| 适用场景 | 标准化流程、批量任务 | 复杂度不确定、需要自主判断 |

> **你的口述话术**："工作流像流水线——每一步做什么是提前定好的；Agent像员工——它自己判断什么时候该做什么。实际上最佳实践是混合模式：主体用工作流保证确定性，关键决策点交给Agent做自主判断。我的AI漏洞研判系统就是这个思路——整体流程是确定的5步，但每个节点的判断是Agent自主完成的。"

### 1.3 ReAct模式（Reasoning + Acting）

ReAct是当前Agent的主流设计模式：

```
循环：
  Thought: 我现在需要什么信息？应该怎么做？
  Action: 调用工具获取信息
  Observation: 工具返回了什么？
  (回到 Thought，直到有足够信息回答)
```

**举例**：用户问"深圳今天适合户外运动吗？"
```
Thought: 需要知道深圳今天的天气
Action: call get_weather(city="深圳")
Observation: 晴，25-32度，风力2级
Thought: 晴天、温度适宜、风力小，非常适合户外运动
Answer: 今天深圳天气晴朗，温度25-32度，风力不大，非常适合户外运动
```

---

## 二、模块1：工具调用（Function Calling）

### 2.1 Function Calling原理

```
用户问题 → LLM判断需要工具 → 输出结构化调用请求
  → 代码执行工具 → 结果回填上下文 → LLM基于结果继续
```

### 2.2 工具Schema设计最佳实践

```python
# 好的工具定义
tool_schema = {
    "name": "search_products",
    "description": "搜索商品信息。用于用户询问商品价格、库存、规格时调用。",
    "parameters": {
        "type": "object",
        "properties": {
            "keyword": {
                "type": "string",
                "description": "商品名称关键词，如'iPhone 15'、'茅台'"
            },
            "category": {
                "type": "string",
                "enum": ["食品", "电子产品", "服装", "其他"],
                "description": "商品类别"
            }
        },
        "required": ["keyword"]
    }
}
```

**设计要点**：
1. **description写清楚**：这是模型判断要不要调这个工具的唯一依据
2. **参数约束明确**：enum限制范围、description说明含义
3. **责任单一**：一个工具只做一件事

### 2.3 工具失败处理

| 失败类型 | 处理方式 |
|---------|---------|
| 网络超时 | 自动重试（指数退避，最多3次） |
| API限流 | 等待后重试，或降级用缓存 |
| 参数错误 | 把错误信息回填给Agent，让它重新构造参数 |
| 工具不可用 | 用备选工具替代，或标记为需要人工处理 |

### 2.4 MCP协议（Model Context Protocol）

MCP是Anthropic提出的工具调用标准化协议。它定义了统一的接口规范：
- 工具发现：列出所有可用工具及其能力
- 工具调用：标准化的请求/响应格式
- 资源访问：统一的文件/数据库访问方式

**价值**：以前每个AI应用都要自己写工具集成代码。MCP标准化后，实现了MCP的工具可以被任何支持MCP的AI应用直接调用——像USB外设一样即插即用。

---

## 三、模块2：LangChain 基础

### 3.1 LangChain核心抽象

| 抽象 | 作用 | 示例 |
|------|------|------|
| **Model** | 封装LLM调用 | ChatOpenAI, ChatDeepSeek |
| **Message** | 对话消息 | HumanMessage, AIMessage, SystemMessage |
| **Prompt** | Prompt模板 | ChatPromptTemplate |
| **Chain** | 串联多个组件 | LLMChain, SequentialChain |
| **Tool** | 工具封装 | @tool装饰器定义函数为工具 |
| **Agent** | 自主决策执行 | AgentExecutor |
| **Memory** | 对话记忆 | ConversationBufferMemory |

### 3.2 LCEL（LangChain Expression Language）

```python
from langchain_core.runnables import RunnablePassthrough

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
# 用 | 管道符串联，数据从左流到右
```

### 3.3 LangChain的局限

- Chain是线性的——做不了复杂分支和循环
- AgentExecutor的黑盒程度比较高，不容易自定义控制流
- Memory管理比较简单，不适合复杂的状态维护

**这就是为什么要用LangGraph**：

---

## 四、模块3：LangGraph深度 ⭐你的王牌

### 4.1 为什么LangGraph比LangChain更适合复杂Agent？

| 维度 | LangChain | LangGraph |
|------|-----------|-----------|
| 流程模型 | 线性Chain | 有向图（节点+边） |
| 分支逻辑 | 有限（RouterChain） | 原生支持条件边 |
| 循环 | 不支持 | 原生支持 |
| 状态管理 | 简单Memory | StateGraph + Checkpointer |
| 人工介入 | 不支持 | 原生支持interrupt/resume |
| 持久化 | 需要自己实现 | 内置Checkpointer |

### 4.2 StateGraph核心概念

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

# 1. 定义State（节点间共享的数据结构）
class AgentState(TypedDict):
    report_content: str
    completeness_result: dict
    poc_code: str
    verify_result: dict
    severity_level: str
    bounty_amount: int
    approval_status: str
    error_info: str

# 2. 创建StateGraph
workflow = StateGraph(AgentState)

# 3. 添加节点（每个节点是一个处理函数）
def completeness_check(state: AgentState) -> dict:
    """信息完整性评估节点"""
    # 分析report_content，判断信息是否完整
    result = analyze_completeness(state["report_content"])
    return {"completeness_result": result}

workflow.add_node("completeness_check", completeness_check)
workflow.add_node("poc_risk_assessment", poc_risk_assessment)
workflow.add_node("poc_verification", poc_verification)
workflow.add_node("severity_pricing", severity_pricing)
workflow.add_node("auto_retest", auto_retest)

# 4. 设置入口
workflow.set_entry_point("completeness_check")

# 5. 添加条件边（根据state决定走哪个分支）
def route_after_completeness(state: AgentState) -> str:
    if state["completeness_result"]["is_complete"]:
        return "poc_risk_assessment"
    else:
        return END  # 信息不完整直接结束，驳回报告

workflow.add_conditional_edges(
    "completeness_check",
    route_after_completeness,
    {
        "poc_risk_assessment": "poc_risk_assessment",
        END: END
    }
)

# 6. 添加普通边
workflow.add_edge("poc_risk_assessment", "poc_verification")
workflow.add_edge("poc_verification", "severity_pricing")
workflow.add_edge("severity_pricing", "auto_retest")
workflow.add_edge("auto_retest", END)

# 7. 编译（可选checkpointer做持久化，interrupt做人工介入）
app = workflow.compile(
    checkpointer=PostgresSaver(conn),  # PostgreSQL持久化状态
    interrupt_before=["severity_pricing"]  # 在定价前暂停等人工审批
)
```

### 4.3 你的5-Agent系统对应讲解

```
┌──────────────────┐
│  信息完整性评估    │ ← 入口节点
│  判断报告是否完整   │
└──────┬───────────┘
       │ is_complete? 
       ├──YES──→ ┌──────────────────┐
       │         │ POC构造与风险评估  │
       │         │ 生成验证脚本       │
       │         └──────┬───────────┘
       │                ↓
       │         ┌──────────────────┐
       │         │ POC自动验证       │
       │         │ Docker沙箱执行    │
       │         └──────┬───────────┘
       │                ↓
       │         ┌──────────────────┐
       │         │ 漏洞定级定价       │ ← interrupt_before：暂停等飞书审批
       │         │ RAG检索+定价决策   │
       │         └──────┬───────────┘
       │                ↓ (审批通过)
       │         ┌──────────────────┐
       │         │ 自动复测          │
       │         │ 修复后重新验证     │
       │         └──────┬───────────┘
       │                ↓
       └──NO──→  END     END
              (驳回)
```

**面试口述**：
> "五个节点分别对应安全运营审核流程的五个阶段。节点间通过State共享——每个节点读取上游的输出、写入自己的结果。关键设计有两个：一是在完整性评估后用条件边判断——信息不全直接驳回，不进后续节点，节省资源；二是在定级定价节点用interrupt_before暂停——发飞书审批卡片给运营人员，审批通过后才继续，保证敏感决策有人工把关。"

### 4.4 Checkpointer（状态持久化）

```python
# 设置 checkpointer -> 每个节点执行后自动保存State
app = workflow.compile(checkpointer=PostgresSaver(conn))

# 执行时指定thread_id
config = {"configurable": {"thread_id": "report-12345"}}
app.invoke({"report_content": "..."}, config)

# 随时可以恢复状态
state = app.get_state(config)
# 人工审批后恢复执行
app.invoke(None, config)  # None表示继续从上次中断处执行
```

**`invoke(input, config)` 两个参数详解**：

| 参数 | 含义 |
|---|---|
| `input`（第1个参数） | **dict** → 传入初始 State，工作流从第一个节点开始执行。**`None`** → 没有新输入，从上次中断的 checkpoint 恢复继续执行（配合 `interrupt_before/after` 使用） |
| `config`（第2个参数） | 运行时配置字典，核心字段：`"configurable":{"thread_id":"..."}` — **`thread_id` 是最关键的**，LangGraph 用它去 checkpointer 里查找该会话的 State 历史。同一个 `thread_id` = 同一条状态链，首次执行和恢复执行必须用同一个 `thread_id` 才能串起来。可选字段还包括：`checkpoint_ns`（子图命名空间）、`checkpoint_id`（回滚到指定版本）、`callbacks`（回调）、`tags`（追踪标签）、`metadata`（自定义元数据）、`recursion_limit`（最大递归次数，默认25）、`max_concurrency`（最大并发节点数） |

**Checkpointer 持久化机制总结**：你负责**选择存储后端并传入**（一行 `checkpointer=PostgresSaver(conn)`），框架负责**在每个节点执行后自动保存 State**（零侵入，节点代码完全无感）。这是编译时注入的横切关注点，不需要在任何节点里手动写 `save()`。

### 4.5 Interrupt（人工介入）

```python
# 在severity_pricing节点前暂停
app = workflow.compile(interrupt_before=["severity_pricing"])

# 执行到severity_pricing前自动暂停
result = app.invoke(input_data, config)

# 此时发送飞书审批卡片...
# 审批通过后：
app.invoke(None, config)  # 继续执行severity_pricing
```

> **深入阅读**：interrupt 的内存本质 vs 持久化、崩溃恢复、checkpoint 内部结构、单 workflow 与拆 workflow 架构对比 —— 详见 [LangGraph中断与持久化深度.md](./LangGraph中断与持久化深度.md)

---

## 五、模块4：LlamaIndex

### 5.1 LlamaIndex定位

LlamaIndex专注于**数据索引和检索**，核心场景是RAG。

| 维度 | LlamaIndex | LangChain |
|------|-----------|-----------|
| 核心定位 | 数据框架（索引+检索） | 通用LLM应用框架 |
| RAG能力 | 开箱即用、功能丰富 | 需要自己组合 |
| Agent能力 | 有但不强 | LangGraph更灵活 |
| 学习曲线 | 相对平缓 | 更陡 |

### 5.2 LlamaIndex的Agent

LlamaIndex也有Agent组件，支持FunctionTool和ReActAgent。但复杂编排场景不如LangGraph灵活。

### 5.3 选型建议

```
只做RAG → LlamaIndex（文档加载、索引、检索开箱即用）
只做Agent → LangChain + LangGraph（编排灵活）
RAG + Agent → 混用（LlamaIndex做检索，LangGraph做编排）
快速验证 → Dify/Coze（低代码平台）
```

---

## 六、模块5：多智能体协作

### 6.1 协作模式分类

| 模式 | 结构 | 通信方式 | 适用场景 |
|------|------|---------|---------|
| **顺序** | A→B→C | 共享State | 流水线任务（你的漏洞系统） |
| **并行** | A, B, C同时跑 | 最后合并 | 多维度同时分析 |
| **Supervisor** | 一个调度Agent分配任务给Worker | Supervisor统一调度 | 任务类型多样 |
| **辩论** | 多个Agent各自回答，互相评审 | 多轮对战 | 需要多视角决策 |
| **层级** | 上级Agent决定调用哪个下级 | 树形通信 | 复杂组织架构 |

### 6.2 你的系统属于哪种？

> "AI漏洞研判系统属于顺序协作模式——5个Agent按审计流程依次执行，通过共享State传递上下文。每个Agent职责单一、互相不直接通信，靠StateGraph的边来定义执行顺序。这种模式的好处是流程清晰、容易调试——出问题能快速定位到具体节点。"

### 6.3 多Agent的代价与取舍

**代价**：
- 成本高：每个Agent都要调LLM，5个Agent就是5次调用
- 延迟高：串行执行，用户要等所有Agent跑完
- 调试难：错误可能在一个节点的输出中产生但在后续节点才暴露
- 一致性风险：不同Agent之间的判断可能矛盾

**什么时候该用多Agent**：
- 任务天然分阶段且每阶段需要不同的专业判断 → 多Agent
- 能用一个Agent+多个工具解决 → 单Agent（更简单、更便宜、更可控）

---

## 七、模块6：Dify实战总结

### 7.1 Dify适合什么？

✅ 快速验证MVP
✅ 标准RAG问答
✅ 简单线性工作流
✅ 非技术团队也能用
✅ 需要知识库管理和对话界面开箱即用

### 7.2 Dify不适合什么？

❌ 复杂条件分支（if-else嵌套、循环）
❌ 多Agent协作（没有Agent间通信机制）
❌ 自定义检索策略（切片策略内置，无法精细控制）
❌ 深度系统集成（和其他系统API的复杂编排）

### 7.3 Dify vs LangGraph选型决策框架

```
你的需求是标准的RAG问答或简单工作流？
  ├── 是 → Dify（省时间、开箱即用）
  └── 否 → 继续判断
      你的需求有复杂分支、多Agent协作、需要精细控制？
        ├── 是 → LangGraph（灵活、可控）
        └── 否 → 回到Dify
```

> **你的故事**："AI助理用Dify是因为需求标准、要快速验证。漏洞系统用LangGraph是因为5-Agent复杂编排+自定义RAG+飞书深度集成，Dify做不了。两个选择在各自场景下都是对的——没有银弹，选最合适的工具。"

---

## 八、模块7：Agent评估与安全

### 8.1 Agent评估指标

| 指标 | 含义 | 如何衡量 |
|------|------|---------|
| 任务完成率 | Agent成功完成任务的百分比 | 人工标注结果 |
| 工具调用正确率 | 调用了正确的工具、传了正确的参数 | 日志分析 |
| 平均步数 | 完成任务花费的工具调用次数 | 日志统计 |
| Token消耗 | 单次任务的token成本 | API日志 |
| 幻觉率 | Agent编造了不存在的工具结果 | 工具调用日志验证 |

### 8.2 Agent安全实践

| 安全问题 | 防护措施 |
|---------|---------|
| 工具权限过大 | 最小权限原则——数据库工具用只读账号、文件工具限制目录 |
| 恶意参数注入 | 工具层做参数校验，过滤危险字符 |
| 代码执行风险 | 沙箱隔离（Docker容器+资源限制+超时） |
| 凭证泄露 | 密钥放环境变量/密钥管理系统，定期轮换 |
| 无限循环 | 设置最大步数和token预算上限 |
| 提示注入 | 分离系统指令和用户输入，工具层做权限校验 |

> **你的亮点**："我在漏洞系统里做了凭证自动失效检测——POC验证前先检查凭证状态，防止用过期凭证执行验证导致误报。Docker沙箱加超时和资源限制保护宿主机安全。这些都是在Agent系统里容易忽略但非常关键的安全设计。"

---

## 九、面试高频题 + 自查清单

### 面试高频题（答案详见 [面试问答.md](./面试问答.md) §8）

1. Agent和工作流有什么区别？怎么选？
2. LangChain和LangGraph有什么区别？
3. StateGraph的核心概念是什么？
4. 多Agent有哪几种协作模式？
5. Function Calling的工作原理？
6. Agent的安全问题有哪些？
7. Dify vs Coze vs 自己写代码怎么选？
8. ReAct模式是什么？
9. 什么是MCP协议？
10. Agent开发中最容易被忽视的问题是什么？
11. LlamaIndex和LangChain定位有什么区别？
12. Agent怎么控制成本？

### 自查清单

- [ ] 能用一句话讲清Agent和Workflow的区别
- [ ] 能画出LangGraph的StateGraph核心概念图（State/Node/Edge/条件边/Checkpointer/Interrupt）
- [ ] 能对着你的漏洞系统讲出5-Agent的源码级实现细节
- [ ] 能讲清楚Dify选型和LangGraph选型的决策依据
- [ ] 能列举至少三种多Agent协作模式并说明适用场景
- [ ] 能解释Function Calling的工作流程
- [ ] 能说出MCP协议的价值
- [ ] 能列举至少三个Agent安全实践

---

> [← 返回学习框架](./学习框架.md)
