# LangGraph 中断与持久化 — 生产级深度剖析

> **定位**：回答"审批中断在生产环境到底靠不靠谱"这一整类追问，覆盖 interrupt 本质、checkpoint 原理、崩溃恢复、架构对比
> **前置依赖**：[Agent开发.md](./Agent开发.md) 的 §4.2-4.5（StateGraph 基础）
> **相关**：[面试问答.md](./面试问答.md) Q2-4/Q2-5 · [独立产品交付.md](./独立产品交付.md) §3 项目1
> [← 返回学习框架](./学习框架.md)

---

## 一、四个核心问题速答

面试官追问时，先给结论再展开：

**Q1：中断是在内存里一直等吗？**

> "不是。interrupt 触发后 `invoke()` 调用立即返回，线程完全释放。'等待'不是进程 hang 住，而是 checkpointer 在 PostgreSQL 里存了一条记录标记'这个工作流停在审批节点'。后续飞书回调带着 `thread_id` 过来，LangGraph 从数据库加载状态、恢复执行。全程不占用服务端内存和线程。"

**Q2：拆成多个 workflow + HTTP 请求启动第二个，可行吗？**

> "完全可行，而且是更解耦的生产方案。核心区别在于状态管理——LangGraph interrupt 是框架帮你管状态，拆 workflow 需要你自己在业务表里做状态的序列化和交接。两者都能过生产，取舍点是'状态机由框架管还是自己管'。"

**Q3：StateGraph 怎么天然支持状态持久化？**

> "通过 checkpointer 机制。编译 graph 时传入 `PostgresSaver`，每个 super-step 结束后 LangGraph 自动把 State 快照 + 控制流位置 + 中断信息写入 PostgreSQL。同一个 `thread_id` 后续 invoke 时，框架自动从最新 checkpoint 恢复。对开发者来说几乎是透明的。"

**Q4：进程崩溃后怎么恢复？**

> "checkpoint 已经持久化在 PostgreSQL 里了，不在进程内存里。重启后用同一个 `thread_id` 和 `Command(resume=...)` 调用 `app.invoke()`，LangGraph 从数据库加载最后 checkpoint，自动从中断点继续。加上回调幂等检查，飞书重复投递也不怕。"

---

## 二、interrupt 机制解剖

### 2.1 两种 interrupt 写法

现有文档用的是编译期 `interrupt_before`（在节点**之前**暂停）：

```python
# 方式一：编译期 interrupt_before —— 在 severity_pricing 节点执行前暂停
app = workflow.compile(
    checkpointer=PostgresSaver(conn),
    interrupt_before=["severity_pricing"]  # 暂停在节点入口
)
# invoke 执行到 severity_pricing 前自动返回，节点函数没有被调用
result = app.invoke(input_data, config)
# 此时 result 是 severity_pricing 之前最后一个节点(poc_verification)的输出
```

更灵活的写法是节点内部 `interrupt()`（在节点**之中**暂停，可以携带上下文数据）：

```python
from langgraph.types import interrupt

def severity_pricing(state: AgentState) -> dict:
    """漏洞定级定价节点 —— 内部中断等人工审批"""
    # 先做 RAG 检索和定价分析
    rag_result = retrieve_similar_cases(state["report_content"])
    suggested_level = analyze_severity(rag_result)
    suggested_bounty = calculate_bounty(suggested_level, rag_result)

    # 触发中断：把分析结果传给审批人，等待决定
    decision = interrupt({
        "type": "approval_required",
        "report_id": state["report_id"],
        "suggested_severity": suggested_level,
        "suggested_bounty": suggested_bounty,
        "similar_cases": rag_result[:3],   # 供审批人参考
    })

    # --- 审批人操作后，从这里继续执行 ---
    # decision 的值就是 Command(resume=...) 传入的值
    return {
        "severity_level": suggested_level,
        "bounty_amount": suggested_bounty,
        "approval_status": decision["result"],  # "approved" | "rejected"
    }
```

**两种方式的区别**：

| | interrupt_before | interrupt() |
|---|---|---|
| 暂停位置 | 节点执行**前** | 节点执行**中** |
| 节点是否执行 | 不执行，等 resume 后才进入 | 执行到 `interrupt()` 行暂停，resume 后从下一行继续 |
| 是否携带数据 | 只能传 State 当前值 | 可以携带分析结果、参考数据等上下文 |
| 适用场景 | 简单审批（只看上游结果） | 复杂审批（审批人需要看 AI 分析结论） |

### 2.2 执行语义：不是阻塞等待

关键理解：`app.invoke()` 在遇到 interrupt 时**会返回**，不会阻塞线程。

```python
# FastAPI 端点：启动审批流程
@app.post("/vuln-report/assess")
async def assess_report(report: ReportInput):
    config = {"configurable": {"thread_id": f"report-{report.report_id}"}}

    # invoke 遇到 interrupt 后立即返回，不阻塞
    # 返回的 state 包含中断信息
    result = app.invoke(
        {"report_content": report.content, "report_id": report.report_id},
        config
    )

    # 此时可以检查是否有 pending interrupt
    state = app.get_state(config)
    if state.interrupts:
        # 发送飞书审批卡片（异步，不阻塞）
        await send_feishu_approval_card(
            thread_id=f"report-{report.report_id}",
            payload=state.interrupts[0].value  # interrupt 携带的数据
        )

    return {"status": "pending_approval", "thread_id": f"report-{report.report_id}"}
```

"等待审批"的本质：PostgreSQL 的 `checkpoints` 表里有一条记录，`interrupts` 字段非空。**不占内存，不占线程，不占 HTTP 连接。**

并发 100 个审批挂起？只是 100 条数据库记录。

### 2.3 为什么必须配 checkpointer

```python
# ❌ 错误：不配 checkpointer 使用 interrupt 会直接报错
app = workflow.compile(interrupt_before=["severity_pricing"])
app.invoke(data)  # RuntimeError: Interrupts require a checkpointer

# ⚠️ 能用但脆弱：MemorySaver —— 进程重启即丢失所有状态
from langgraph.checkpoint.memory import MemorySaver
app = workflow.compile(
    checkpointer=MemorySaver(),
    interrupt_before=["severity_pricing"]
)
# 只适合本地开发调试

# ✅ 生产：PostgresSaver —— 状态持久化到 PostgreSQL
from langgraph.checkpoint.postgres import PostgresSaver

with PostgresSaver.from_conn_string(DB_URL) as checkpointer:
    checkpointer.setup()  # 自动建表
    app = workflow.compile(
        checkpointer=checkpointer,
        interrupt_before=["severity_pricing"]
    )
```

| checkpointer | 存储位置 | 进程重启 | 多实例共享 | 适用场景 |
|---|---|---|---|---|
| MemorySaver | 进程内存 dict | 丢失 | 不支持 | 本地开发、单元测试 |
| SqliteSaver | 本地 SQLite 文件 | 保留 | 不支持（文件锁） | 单机小规模生产 |
| PostgresSaver | PostgreSQL | 保留 | 支持 | 生产环境 |
| AsyncPostgresSaver | PostgreSQL（异步） | 保留 | 支持 | 异步 FastAPI 生产 |

---

## 三、checkpoint 到底存了什么

### 3.1 数据结构

每个 super-step 结束后，LangGraph 自动写入一条 checkpoint 记录：

```python
# checkpoint 的逻辑结构（实际在 PostgresSaver 中拆分到多张表）
{
    "v": 1,                              # checkpoint 协议版本
    "id": "1ef7e8a2-...",               # checkpoint 唯一 ID
    "ts": "2024-08-05T10:30:00Z",       # 创建时间戳
    "parent_checkpoint_id": "1ef7e8a1-...",  # 上一个 checkpoint，形成历史链
    "channel_values": {                  # 当前 State 的所有字段值
        "report_id": "SRC-2024-0815",
        "report_content": "...",
        "completeness_result": {"is_complete": True},
        "poc_code": "import requests\n...",
        "verify_result": {"status": "vulnerable", "confidence": 0.92},
        "severity_level": "",            # 尚未填充（在 interrupt 点）
        "bounty_amount": 0,
        "approval_status": "pending",
        "error_info": ""
    },
    "channel_versions": {                # 每个 channel 的版本号（用于合并冲突检测）
        "report_content": 5,
        "completeness_result": 5,
        ...
    },
    "versions_seen": {                   # 记录各节点对 channel 版本的感知
        "completeness_check": {"report_content": 1},
        "poc_risk_assessment": {"completeness_result": 3},
        ...
    },
    "next": ("severity_pricing",),       # 下一步要执行的节点
    "interrupts": [                      # 活跃的中断列表（为空表示没有中断）
        {
            "id": "interrupt-uuid",
            "node": "severity_pricing",
            "value": {                   # interrupt() 调用时传入的 payload
                "type": "approval_required",
                "report_id": "SRC-2024-0815",
                "suggested_severity": "high",
                "suggested_bounty": 3000
            }
        }
    ]
}
```

### 3.2 存储时机

LangGraph 在**每个 super-step 结束后**写入 checkpoint。

一个 super-step 的定义：从当前节点开始执行，直到遇到以下任一情况为止：
- 到达一个需要外部输入的节点（`interrupt_before` 目标 / 调用了 `interrupt()` 的节点）
- 到达 `END`
- 经过的所有中间节点都执行完毕

```
执行示例：
  completeness_check → poc_risk_assessment → poc_verification
  ↑                                               ↑
  入口                                        checkpoint 1
  （三个节点在一个 super-step 中顺序执行）

  → interrupt 触发
    ↑
  checkpoint 2（含 interrupts）

  → resume → severity_pricing → auto_retest → END
                ↑
          checkpoint 3（interrupts 清空，approval_status 更新）
```

### 3.3 节点内中断的"断点回放"原理

当节点内调用了 `interrupt()` 时，有一个重要细节：**pending writes**。

```python
def severity_pricing(state: AgentState) -> dict:
    # 假设前面执行了一些操作，修改了 state
    # 然后在这里 interrupt
    decision = interrupt({...})
    # resume 后继续
    return {"approval_status": decision["result"]}
```

在 `interrupt()` 被调用时：
1. 节点函数的执行被"冻结"
2. 当前 super-step 结束，checkpoint 写入（`next` 指向 `severity_pricing`，`interrupts` 非空）
3. resume 时：恢复 `severity_pricing` 节点的执行，但不是从头开始——从 `interrupt()` 调用的下一行继续
4. `interrupt()` 返回 `Command(resume=...)` 中传入的值

这就是 checkpoint 的"断点精确恢复"能力。

### 3.4 状态管理 API

```python
# 查看当前状态（包括中断信息）
state = app.get_state(config)
print(state.interrupts)     # 活跃中断列表，为空 = 无中断
print(state.values)         # 当前 state 值
print(state.next)           # 下一步执行的节点

# 查看完整历史（所有 checkpoint 的链）
history = list(app.get_state_history(config))
for snapshot in history:
    print(f"checkpoint {snapshot.config['configurable']['checkpoint_id']}")
    print(f"  next: {snapshot.next}")
    print(f"  interrupts: {snapshot.interrupts}")

# 直接修改状态（绕过节点执行 — 超时兜底的关键）
app.update_state(
    config,
    {"approval_status": "rejected", "error_info": "审批超时自动驳回"},
    as_node="severity_pricing"  # 声称这个更新来自 severity_pricing 节点
)
```

---

## 四、崩溃恢复与生产加固

### 4.1 崩溃场景分类

| 场景 | 发生了什么 | checkpoint 状态 | 恢复方式 |
|------|-----------|----------------|---------|
| **节点执行中崩溃** | `severity_pricing` 执行到一半进程被 kill | 上一个节点的 checkpoint 完好 | 重新 invoke，节点重新执行（需保证节点幂等） |
| **等待审批时进程重启** | interrupt 已触发，飞书卡片已发，进程重启 | 最新 checkpoint 含 interrupts 记录，在 PostgreSQL 中完好 | 飞书回调到达后 `app.invoke(Command(resume=...), config)`，从 checkpoint 恢复 |
| **飞书回调丢失** | 审批人点了通过但网络问题回调没到达 | checkpoint 仍标记为 interrupt 状态 | 定时任务扫描超时未审批的记录，发提醒或自动升级 |
| **飞书回调重复投递** | 同一个审批结果飞书发了两次回调 | 第一次恢复后 interrupts 清空 | 第二次回调时 `get_state` 检查 interrupts 为空 → 幂等返回 |

### 4.2 正常流程 vs 崩溃恢复（时序对比）

```
正常流程：
  Node A 完成 → [cp1] → Node B 完成 → [cp2] → interrupt → [cp3+interrupts]
  → 飞书卡片 → 审批人点击 → 回调 /callback → Command(resume) → [cp4] → Node C

崩溃恢复流程：
  Node A 完成 → [cp1] → Node B 完成 → [cp2] → interrupt → [cp3+interrupts]
  → 飞书卡片 → 💥 进程崩溃/重启
  → 审批人点击 → 飞书回调 /callback
  → app.invoke(Command(resume=...), config)
  → 从 PostgreSQL 加载 [cp3] → 恢复 severity_pricing 执行 → [cp4] → Node C
```

### 4.3 飞书回调端点：幂等、可重试

```python
from langgraph.types import Command

@app.post("/feishu/callback")
async def feishu_approval_callback(request: Request):
    """飞书审批卡片回调 — 崩溃安全、幂等"""
    # 1. 飞书签名校验
    body = await request.json()
    if not verify_feishu_signature(request.headers, body):
        raise HTTPException(401, "Invalid signature")

    # 2. 解析审批结果
    action = body.get("action")  # "approved" | "rejected" | "transferred"
    thread_id = body.get("action_value", {}).get("thread_id")
    config = {"configurable": {"thread_id": thread_id}}

    # 3. 幂等检查：是否已经处理过
    state = app.get_state(config)
    if not state.interrupts:
        # interrupts 为空 = 已经 resume 过了
        return {"status": "already_processed"}

    # 4. 权限校验：审批人是否匹配
    expected_approver = state.values.get("approver_id")
    actual_user = body.get("user_id")
    if expected_approver and actual_user != expected_approver:
        return {"status": "not_authorized"}

    # 5. 恢复执行
    try:
        result = app.invoke(
            Command(resume={"result": action, "approver": actual_user}),
            config
        )
    except Exception as e:
        # 节点执行失败时 checkpoint 不会前进，可重试
        logger.error(f"Resume failed for {thread_id}: {e}")
        raise HTTPException(500, "Resume failed, retryable")

    return {"status": "ok", "next_node": result.get("next")}
```

### 4.4 审批超时处理

```python
# 定时任务（可用 APScheduler 或 Celery Beat，每 10 分钟执行一次）
async def scan_timeout_approvals():
    """扫描超时的审批，自动提醒或驳回"""
    conn = await get_db_connection()

    # 从 PostgreSQL checkpoints 表查询 pending 中断
    # 注意：不同版本的 PostgresSaver 表结构可能不同，这里用 LangGraph API
    rows = await conn.fetch("""
        SELECT cp.thread_id, cp.checkpoint, cp.created_at
        FROM checkpoints cp
        WHERE cp.checkpoint->'interrupts' IS NOT NULL
          AND cp.checkpoint->'interrupts' != '[]'
          AND cp.parent_checkpoint_id IS NOT NULL
    """)

    now = datetime.utcnow()
    for row in rows:
        elapsed = now - row["created_at"]

        if elapsed > timedelta(hours=48):
            # 超过 48 小时：自动驳回
            config = {"configurable": {"thread_id": row["thread_id"]}}
            app.update_state(
                config,
                {
                    "approval_status": "auto_rejected",
                    "error_info": f"审批超时({elapsed.total_seconds()/3600:.0f}h)，自动驳回"
                },
                as_node="severity_pricing"
            )
            logger.info(f"Auto-rejected timeout approval: {row['thread_id']}")

        elif elapsed > timedelta(hours=24):
            # 超过 24 小时未响应：发提醒
            await send_feishu_reminder(row["thread_id"])
            logger.info(f"Reminder sent for: {row['thread_id']}")
```

### 4.5 多实例部署的注意事项

如果 FastAPI 部署了多个实例（如 Kubernetes replicas > 1），需要注意：

- **回调可能到达任意实例**：因为状态在共享的 PostgreSQL 中，任意实例都能恢复，没问题
- **定时扫描只需一个实例执行**：用分布式锁（如 Redis `SETNX` 或 PostgreSQL advisory lock）确保扫描任务只有一个实例运行
- **checkpoint 写入的并发安全**：PostgresSaver 使用 PostgreSQL 的行级锁和 MVCC 保证同一 `thread_id` 的 checkpoint 写入是串行的

---

## 五、两种方案架构对比

这是本文最重要的章节，直接回答"拆成两个 workflow 行不行"。

### 5.1 方案 A：单 workflow + LangGraph interrupt（现状）

```
POST /vuln-report/assess
  → app.invoke(initial_state, config)
    → completeness_check
    → poc_risk_assessment
    → poc_verification
    → severity_pricing  ← interrupt_before / interrupt()
    → [checkpoint 写入 PG，interrupts 非空]
    → invoke 返回
  ← {"status": "pending_approval", "thread_id": "report-xxx"}

飞书卡片发送（在 API handler 中异步执行）

审批人点击卡片
  → 飞书 POST /feishu/callback
  → app.invoke(Command(resume={"result": "approved"}), config)
    → 从 PG 加载 checkpoint
    → 恢复 severity_pricing 执行
    → auto_retest
    → END
  ← {"status": "ok"}
```

**优点**：
- 图定义集中在一个地方：从入口到 END 完整可见
- LangGraph 自动管理状态：不需要手动序列化/反序列化
- resume 语义精确：`interrupt()` 后的代码直接从断点继续
- 配合 LangSmith 可以追踪完整链路

**缺点**：
- 图的定义耦合了"执行"和"等待审批"两个语义
- 如果审批前后需要不同的部署（如审批后的 auto_retest 需要 GPU），不够灵活
- 依赖 PostgresSaver 的数据库连接

### 5.2 方案 B：拆两个 workflow + HTTP 回调

```
Workflow A：审批前链路
  POST /workflow/assess
    → completeness_check
    → poc_risk_assessment
    → poc_verification
    → 计算定价建议 → 写入业务表（状态: pending_approval）
    → END
  ← {"status": "pending_approval", "report_id": "xxx"}

审批人点击卡片
  → 飞书 POST /workflow/approve
  → 从业务表加载状态
  → 构造第二个 workflow 的输入
  → 调用 Workflow B

Workflow B：审批后链路
  → severity_pricing（应用审批结果）
  → auto_retest
  → END
```

```python
# 方案 B 代码骨架
@app.post("/workflow/assess")
async def workflow_a(report: ReportInput):
    report_id = generate_report_id()
    result = assess_graph.invoke({"report_content": report.content, "report_id": report_id})

    # 手动保存状态到业务表
    await db.execute("""
        INSERT INTO workflow_state (report_id, state_json, status)
        VALUES ($1, $2, 'pending_approval')
    """, report_id, json.dumps(result))

    await send_feishu_card(report_id, result["pricing_suggestion"])
    return {"status": "pending_approval", "report_id": report_id}


@app.post("/workflow/approve")
async def workflow_b(callback: FeishuCallback):
    report_id = callback.action_value["report_id"]

    # 关键：幂等检查
    state_row = await db.fetchrow(
        "SELECT * FROM workflow_state WHERE report_id = $1 AND status = 'pending_approval'",
        report_id
    )
    if not state_row:
        return {"status": "already_processed"}  # 幂等

    # 手动从业务表加载状态
    previous_state = json.loads(state_row["state_json"])
    previous_state["approval_result"] = callback.action

    # 调用第二个 workflow
    result = approve_graph.invoke(previous_state)

    # 更新状态
    await db.execute(
        "UPDATE workflow_state SET status = 'completed', state_json = $2 WHERE report_id = $1",
        report_id, json.dumps(result)
    )
    return {"status": "ok"}
```

### 5.3 对比表

| 维度 | 方案 A：单 workflow + interrupt | 方案 B：拆 workflow + HTTP |
|------|-------------------------------|--------------------------|
| **状态管理** | LangGraph checkpointer 自动存储和恢复 | 需要自己序列化到业务表，手动加载 |
| **resume 语义** | 精确断点恢复（`interrupt()` 后下一行继续） | 无原生 resume — 第二个 workflow 从头执行 |
| **代码集中度** | 一个图定义全流程，可读性高 | 两个独立入口 + 状态交接代码 |
| **崩溃恢复** | checkpointer 自动处理 | 需要自己扫描业务表的待处理记录 |
| **部署灵活性** | 审批前后在同一服务 | 可部署在不同服务、不同环境 |
| **超时处理** | `update_state` + 定时扫描 | 定时扫描业务表 |
| **监控可观测性** | LangSmith 追踪全链路 | 需手动跨 workflow 关联 trace |
| **复杂度** | 低（框架原生能力） | 中（需处理幂等、补偿、状态交接） |
| **对框架依赖** | 强依赖 LangGraph checkpointer | 不绑定 LangGraph，甚至不用 LangGraph 也行 |

### 5.4 选型建议

| 场景 | 推荐方案 |
|------|---------|
| 审批链路短（分钟到小时），同一服务能处理全程 | 方案 A（单 workflow + interrupt） |
| 审批可能持续数天，审批前后服务环境不同 | 方案 B（拆 workflow + HTTP） |
| 审批前后的计算资源需求差异大（如审批后需要 GPU 执行 auto_retest） | 方案 B |
| 团队不想深度依赖 LangGraph 的状态持久化能力 | 方案 B |
| 想用 LangSmith 追踪完整审批链路做分析 | 方案 A |
| 审批节点后有复杂的分支路由 | 方案 A（条件边天然适合） |

**实际推荐**：对于漏洞研判系统的场景（审批通常在几分钟到几小时内完成，全流程在同一套技术栈内），**方案 A 是更优选择**。面试时如果被问到为什么不用拆分方案，可以说：

> "两种方案都能过生产。我选单 workflow + PostgresSaver 是因为：第一，审批通常在小时内完成，不是跨天级的异步任务；第二，全流程在同一套 Python 服务里，没必要拆分部署；第三，LangGraph 的条件边天然适合'审批通过→复测 / 驳回→重评'的路由逻辑。如果有一天审批环节需要独立出来做一个专门的审批微服务，再拆也不迟。"

### 5.5 补充：方案 C — DB 信号 + 轮询

还有一种方案是不用 LangGraph interrupt，审批节点通过查询数据库中的审批状态表来决定行为：

```python
def severity_pricing(state: AgentState) -> dict:
    # 先存草稿 + 发飞书卡片
    save_draft(state["report_id"], suggested_pricing)
    send_feishu_card(state["report_id"])

    # 不中断，不 resume——轮询等审批结果
    for _ in range(max_retries):
        time.sleep(30)  # 每 30 秒查一次
        approval = get_approval_status(state["report_id"])
        if approval != "pending":
            break
    # 问题：这是同步阻塞！不适合 FastAPI 的异步模型
```

方案 C 的问题是：它把异步的"人等审批"变成了同步的"代码等审批"，阻塞线程、浪费连接。不推荐在生产环境使用，仅在此作为反面参考。

---

## 六、面试官追问预案

| 追问 | 回答要点 |
|------|---------|
| "interrupt 的时候内存里挂着吗？" | 不会。invoke 返回，线程释放。'等待'是 PG 里的一条记录。并发 100 个审批挂起只是 100 条数据库行。 |
| "服务重启了审批怎么恢复？" | 状态在 PostgresSaver 里，不在进程内存。飞书回调带 thread_id 过来，invoke 时从 PG 加载 checkpoint 继续。 |
| "飞书回调重复了怎么办？" | 回调入口先 `get_state` 检查 interrupts——如果已经是空的，说明已处理过，直接返回成功。 |
| "24小时没人审批呢？" | 定时任务扫描 interrupts 非空且超过 24h 的 checkpoint——发飞书催办消息。 |
| "48小时还没人审批呢？" | 定时任务 `update_state` 强制写入驳回状态，自动关闭流程。同时发消息升级给上级。 |
| "为什么不用两个 workflow？" | 可以做，但没必要。LangGraph interrupt + PostgresSaver 已经解决了状态持久化和恢复，拆两个只会增加状态交接代码和幂等逻辑。除非审批前后需要不同的服务环境。 |
| "checkpoint 里存了什么？" | state 值快照 + 下一步执行节点 + 中断列表 + 版本信息。相当于给工作流在每个步骤后拍了"快照"。 |
| "并发 100 个审批挂起行不行？" | 完全没问题。每个挂起只是一条 PG 行。服务端零内存占用。真正执行时按 invoke 请求排队。 |
| "interrupt_before 和 interrupt() 什么区别？" | interrupt_before 在节点执行前暂停（编译期设置），interrupt() 在节点中间暂停（运行时调用）。后者更灵活——可以前面做完 RAG 分析再把结果带给审批人。 |
| "驳回后怎么回到前面节点？" | 条件边：`approval_status == "approved"` → auto_retest；`== "rejected"` → poc_risk_assessment 重新评估。StateGraph 支持条件边走向任意节点，天然支持这种回退路由。 |
| "多实例部署下怎么保证同一个 thread 只被一个实例处理？" | 状态在共享 PG 中，任意实例都能处理回调。checkpoint 写入靠 PG 行级锁保证串行。定时扫描只需一个实例执行（分布式锁）。 |
| "不用 checkpointer 能用 interrupt 吗？" | 不能。不配 checkpointer 时调用 interrupt 会直接报错 `RuntimeError`。interrupt 本质就是把状态存到 checkpoint 里然后返回。 |

---

## 七、加分项：上升到"持久化执行"概念

如果你想让面试官印象深刻，可以提一句这个概念：

> "LangGraph 的 checkpointer + interrupt 本质上是一种轻量级的'持久化执行'（durable execution）。这个概念在 Temporal、Azure Durable Functions 这类工作流引擎里是核心卖点——你的代码可以在任意点暂停，状态持久化，几小时甚至几天后从断点精确恢复。LangGraph 把这个能力带到了 AI Agent 编排领域，而且对开发者几乎是透明的——你写普通的 Python 函数，调 `interrupt()` 暂停，框架帮你搞定状态持久化和断点恢复。"

不需要展开，点到即止。展示你知道这个概念的存在，以及它在更广阔的技术生态里的位置。

---

## 八、自查清单

- [ ] 能一句话说清 interrupt 不是内存等待
- [ ] 能区分 MemorySaver 和 PostgresSaver 的场景
- [ ] 能画图解释 checkpoint 的存储内容（state + next + interrupts）
- [ ] 能写出飞书回调端点的幂等检查逻辑（`get_state` → 检查 interrupts）
- [ ] 能说清崩溃恢复的四类场景和各自处理方式
- [ ] 能画出方案 A（单 workflow）和方案 B（拆 workflow）的时序图
- [ ] 能从 5 个维度对比两种方案的优劣
- [ ] 能解释 interrupt_before 和 interrupt() 的区别
- [ ] 能讲 24h/48h 超时升级的实现方式（定时扫描 + update_state）
- [ ] 能一句话提到"持久化执行"概念展示知识面

---

> **关键提醒**：这部分是面试官区分"用过 LangGraph"和"理解 LangGraph"的分水岭。能回答本章的问题，说明你对工具的理解到了源码级，而不只是看了文档的 API 调用。
>
> [← 返回学习框架](./学习框架.md) | 相关: [Agent开发.md](./Agent开发.md) · [面试问答.md](./面试问答.md)
