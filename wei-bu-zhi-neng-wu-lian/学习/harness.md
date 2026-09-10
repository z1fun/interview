# Harness 框架深度讲解 —— Agent 驾驭层工程

> **核心认知**：招聘信息中的 "Harness" ≠ Harness.io（CI/CD 平台）。它是 **Agent Harness（智能体驾驭层工程）**。

---

## 一、什么是 Harness？

### 1.1 核心公式

```
Agent = Model（引擎）+ Harness（底盘）
```

| 角色 | 职责 | 比喻 |
|------|------|------|
| **Model（模型）** | 推理：文本进 → 文本出 | 发动机 |
| **Harness（驾驭层）** | 工具调用、执行环境、编排、记忆、安全、CI/CD | 底盘 + 方向盘 + 刹车 |

> 业界共识：**模型只决定了 Agent 的上限，Harness 决定了 Agent 的下限**。一个没有 Harness 的 Agent 就是"裸奔的模型"——能说话但做不了事。

### 1.2 为什么"模型越强，Harness 越重要"？

- GPT-4 到 Claude 5，模型能力快速提升，**但非模型部分的工程价值占比已超过 70%**
- 模型只能输出文本 → 要变成"能办事的数字人"需要：工具（查数据库/调API）、记忆（记住用户）、编排（决定先做什么后做什么）、安全（不能乱说话）——这些都是 Harness
- Claude Code 的 15+ 工具、权限系统、Skill 懒加载、子代理上下文隔离、Plan 模式——全是 Harness，不是模型

**来源**：[深入浅出Agent: Harness最全调研 (CSDN)](https://blog.csdn.net/sjkflw121150/article/details/162765700)

---

## 二、Agent 工程的三个阶段演进

| 阶段 | 时代 | 核心问题 | 代表人物/工具 |
|------|------|---------|-------------|
| **Prompt Engineering** | 2023 | 怎么让模型输出我想要的？（控制输出） | Few-shot、CoT、ReAct |
| **Context Engineering** | 2024 | 怎么让模型看到该看的？（控制输入） | RAG、长上下文、上下文压缩 |
| **Harness Engineering** | 2025- | 怎么让模型稳定完成任务？（控制行为） | Claude Code、LangGraph、OpenHarness |

> 贵司招聘信息中对 Harness 的描述——"流程编排、任务管理、权限治理、流水线部署、环境配置、资源调度"——全部落在第三阶段 Harness Engineering。

---

## 三、Harness 八大抽象

这是 Harness 工程的核心概念体系。面试中问到"你理解的 Harness 包含哪些组件"，按这个框架回答。

| 抽象 | 定义 | 谁触发 | 上下文成本 | 持久性 | Claude Code 中的体现 |
|------|------|--------|-----------|--------|-------------------|
| **Tool Call** | 单次工具调用（函数执行） | 模型决策 | 工具定义 + 结果返回 | 无 | Read/Write/Bash/WebFetch 等 |
| **MCP** | 标准化外部工具/数据接入协议 | 模型决策 | 工具注册元数据 | 无 | `claude mcp add`，`.mcp.json` |
| **Skill** | 可复用的能力包（渐进披露/懒加载） | 用户或条件触发 | 仅在激活时注入 prompt | 无 | `.claude/skills/` 目录，`/skill-name` |
| **SubAgent** | 独立上下文的子代理（上下文隔离） | 主 Agent 决策 | 独立上下文窗口 | 任务周期 | Agent 工具（子代理类型） |
| **Memory** | 跨会话持久化信息存储 | 框架自动 | AGENTS.md/记忆文件 | **永久** | MEMORY.md、AGENTS.md |
| **Hook** | 凌驾模型之上的确定性脚本 | 事件触发（工具调用前后/停止时） | 无（确定性执行） | 配置持久 | PreToolUse/PostToolUse/Stop |
| **Plugin** | 打包分发的可复用能力单元 | 安装/配置后生效 | 类似 Skill/MCP | 安装持久 | 第三方 MCP Server、Skill 包 |
| **Goal/Plan** | 长程任务的持久化状态与子任务跟踪 | 框架自动 | 计划状态追踪 | 任务周期 | Plan 模式、TodoWrite |

### 三个阅读角度

1. **上下文成本**：Tool/MCP/Skill 每次调用都消耗 token → 必须选"值得占位置的"。Hook 不占上下文（确定性执行），Goal/Plan 适度占（结构化的计划比散装对话省 token）

2. **谁说了算（控制层级）**：
   - 模型说了算：Tool Call、MCP（模型决定是否调用）
   - 用户说了算：Skill（用户触发 `/skill-name`）
   - 框架说了算（不可绕过）：**Hook**——唯一能凌驾模型之上的机制
   - 模型+框架协商：Goal/Plan

3. **活多久（生命周期）**：
   - 一次性：Tool Call、MCP 调用
   - 任务周期：SubAgent、Goal/Plan
   - 永久/跨会话：Memory（最持久）、Hook（配置级）

> 面试加分点：能讲出"为什么要有 Hook？因为模型不可靠——有些事情必须确定性执行，不能交由模型决策。比如敏感操作审计日志、成本上限熔断、合规检查——这些必须走 Hook 而不是等模型自己做决定。"

---

## 四、构建 Harness 的六大组件

**来源**：[一文讲透如何构建Harness——六大组件全解析 (腾讯云)](https://cloud.tencent.com.cn/developer/article/2648873)

### 4.1 文件系统（外部大脑）

Agent 的工作空间 = 它的外部记忆和行动场所。
- 项目文件 = 它理解任务的基础
- AGENTS.md / CLAUDE.md = 它的"操作手册"
- 输出文件 = 它的工作成果

### 4.2 Bash + 沙箱（自我验证循环）

Agent 不仅能"想"，还能"做"、"验证"、"修正"：
- 写代码 → 编译运行 → 看到报错 → 修复 → 再运行 → 成功
- Claude Code 中最常见的模式：工具调用 → 观察结果 → 调整下一步

### 4.3 记忆系统（跨会话持久化）

包含三个层次：
- **文件记忆**：AGENTS.md、MEMORY.md（不改模型权重，但改变模型看到的东西）
- **向量记忆**：RAG 检索，用于大规模历史对话
- **结构化记忆**：用户画像、偏好、关键事件

详见 [记忆系统.md](./记忆系统.md)

### 4.4 Web Search + MCP（信息接入）

Agent 不能只靠训练数据——需要实时获取外部信息：
- Web Search：网络搜索结果注入上下文
- MCP：标准化协议接入数据库、API、文件系统等

### 4.5 上下文工程（对抗 Context Rot）

Context Rot = 上下文腐烂：随着 Agent 执行越来越多步骤，上下文被无效信息填满，模型性能急剧下降。

**关键指标**：上下文利用率 40% 警戒线——当有效信息占比低于 40%，Agent 开始"忘记"早期指令，做出错误决策。

应对策略：
- 上下文压缩（摘要化早期内容）
- 子代理隔离（独立上下文）
- 渐进披露（Skill 按需加载 prompt）
- 记忆分层（热/温/冷数据分离）

### 4.6 编排 + Hooks（流程控制）

- **编排**：决定 Agent 的执行顺序——串行/并行/条件分支/循环
- **Hooks**：在关键节点插入确定性逻辑——PreToolUse 检查权限、PostToolUse 记录日志、Stop 时清理资源

---

## 五、Harness 成熟度模型 L0-L4

| 级别 | 名称 | 特征 | 对齐岗位要求 |
|------|------|------|------------|
| **L0** | 裸调 API | 直接调模型接口，无工具、无记忆 | — |
| **L1** | 单步工具 | 模型能调用工具，但无编排 | — |
| **L2** | 反馈循环 | 工具输出 → 模型判断 → 再调工具；**CI/CD 介入** | "CI/CD 流水线"、"灰度发布" |
| **L3** | 专业 Agent | 多 Agent 分工 + 持久记忆 + 权限治理 | "流程编排"、"人设记忆"、"权限治理" |
| **L4** | 自主循环 | 自愈、自我评估、长期任务自主推进 | "主动交互"、"场景化任务自主执行" |

> 岗位要求的能力横跨 L2-L4：L2 的 CI/CD + L3 的多 Agent 编排和记忆 + L4 的自主任务。这意味着你需要在面试中展现对 Harness 全栈成熟度的理解。

---

## 六、岗位职责 → Harness 概念映射（面试必背）

| 岗位原文关键描述 | 对应的 Harness 概念 | 一句话解释 |
|-----------------|-------------------|----------|
| **对话引擎** | Agent Loop（ReAct 循环） | 用户输入 → 注入上下文 → 模型推理 → 工具调用 → 观察结果 → 继续推理 → 返回 |
| **人设记忆** | Memory + System Prompt 注入 | 人设 Prompt 放在 System Prompt 层（不可被对话覆盖），动态记忆通过 RAG 注入 |
| **交互逻辑** | 编排状态机 | 确定性逻辑放 Hook/状态机，灵活性放模型决策——"该确定的地方确定，该灵活的地方灵活" |
| **任务体系** | Task 状态机 + TodoWrite | 长程任务分解为子任务，跟踪状态（pending → in_progress → completed），后台异步执行 |
| **流程编排** | Pipeline / 编排循环 | 多步骤串行/并行编排，条件分支 |
| **权限治理** | 权限管线（fail-closed） | 默认为拒绝，AllowList/DenyList、沙箱隔离、操作前确认 |
| **流水线部署 / CI/CD** | Harness 成熟度 L2 反馈循环 | Agent 输出 → 自动测试 → 反馈 → 修正 → 再测试 |
| **环境配置** | 配置文件体系 | settings、AGENTS.md、环境变量管理 |
| **资源调度** | 并发控制 + 执行环境调度 | 工具并行/串行分组、Worktree 隔离、信号量并发限制 |
| **灰度发布** | 渐进发布 + 评估反馈 | 10% 流量 → 评估 → 50% → 评估 → 100%（每一步由评估结果决定是否回滚） |

---

## 七、Claude Code —— 你每天在用的工业级 Agent Harness

这是你在面试中**最有力的武器**。不要只说"我研究过 Claude Code"，要说"我每天都在深度使用 Claude Code，下面是我亲历的 Harness 机制"：

### 7.1 你在 Claude Code 中亲历的 Harness 机制

| Harness 概念 | Claude Code 中的体现 | 你的亲历 |
|-------------|-------------------|---------|
| **工具系统** | Read / Write / Edit / Bash / WebFetch / WebSearch / Agent 等 15+ 工具 | 每次交互都在调用多个工具 |
| **权限控制** | 工具调用前的权限确认、AllowList/DenyList、fail-closed 默认 | 每次 Bash/Write 都要确认（或配置白名单） |
| **Skill 系统** | `.claude/skills/` 目录，`/skill-name` 触发，渐进披露（不调用不加载 prompt） | 使用 `/review`、`/dataviz`、`/loop` 等 |
| **MCP 协议** | `claude mcp add` 配置，`.mcp.json` 管理 | 可接入外部 MCP Server |
| **SubAgent** | Agent 工具，独立上下文，上下文隔离 | 多 Agent 并行执行不同任务 |
| **Hook 系统** | PreToolUse / PostToolUse / Stop / Notification | 关键节点的确定性逻辑 |
| **Plan 模式** | Plan/Act 分离架构 | 先出计划 → 审核 → 再执行 |
| **TodoWrite** | 任务追踪 | 分解复杂任务、跟踪进度 |
| **上下文管理** | 自动压缩、子代理隔离 | 长对话不丢失关键信息 |
| **AGENTS.md / MEMORY.md** | 文件系统记忆 | 跨会话持久化项目知识 |

### 7.2 面试话术

> "虽然我没有在生产环境用过贵司的自研 Harness，但我每天都在深度使用 Claude Code——业界公认目前最完整的工业级 Agent Harness 实现。我亲历了它的工具系统（15+ 工具协作）、权限管线（fail-closed 默认 + AllowList）、Skill 懒加载、MCP 标准协议接入、SubAgent 上下文隔离、Hook 确定性执行、Plan 模式等全部核心机制。这些和贵司 Harness 框架的'流程编排、任务管理、权限治理、资源调度'是一一对应的。我可以快速将这套理解迁移到贵司的框架上。"

---

## 八、Harness 插件开发（Skill 与 MCP）

### 8.1 Skill 开发流程

```
需求分析 → Prompt 设计 → 工具绑定 → 测试 → 灰度发布
```

**Skill 的本质**：一组 prompt 指令 + 工具权限声明，按需加载（不调用不占上下文）。

**数字人场景案例 —— "人设管理 Skill"**：

```markdown
# Skill: 人设一致性检查
## 触发条件
- 每次对话生成后，或用户主动 /check-persona

## 指令
检查当前回复是否满足以下约束：
1. 人称/名字/背景故事与 AGENTS.md 中定义一致
2. 知识边界不越界（数字人不应知道"训练数据外"的信息）
3. 语气风格一致（俏皮/温柔/专业……）

## 工具权限
- Read（读取 AGENTS.md 人设定）
- WebFetch（可选：查外部事实）

## 输出
- 通过 → 继续
- 不通过 → 标注偏差项，请求模型修正
```

### 8.2 MCP Server 开发流程

```
定义 Tool/Resource → 实现 Server → 注册到 Harness → 测试 → 部署
```

**数字人场景案例 —— "记忆 MCP Server"**：

```python
# 伪代码：MCP Server 暴露三个工具
@mcp.tool()
def remember_user_fact(user_id: str, fact: str, category: str):
    """将事实存入用户长期记忆"""
    
@mcp.tool()  
def search_memory(user_id: str, query: str, top_k: int = 5):
    """检索用户相关记忆"""
    
@mcp.tool()
def forget(user_id: str, fact_id: str):
    """选择性遗忘（合规/用户请求）"""
```

详见 [skill与mcp.md](./skill与mcp.md)

---

## 九、面试高频追问与标准应答

### Q: "你用过 Harness 吗？"

**标准应答**（诚实但有力量）：

> "我先确认一下——您说的 Harness 是指 Agent Harness（智能体驾驭层），不是 Harness.io 那个 CI/CD 平台，对吗？
>
> 如果是指 Agent Harness，我虽然没有用过贵司自研的框架，但我每天都在深度使用 Claude Code——业界公认目前最完整的工业级 Agent Harness。我理解 Harness 的核心是'Agent = Model + Harness'——模型负责推理，Harness 负责把推理变成可靠的行动。这个体系包含工具系统、权限管线、Skill/MCP 插件、记忆系统、编排循环、Hook 等核心抽象。
>
> 同时我在 LangGraph 上做过 5 个 Agent 协作的编排落地，对 Harness 的工程实践有完整的体感。我相信可以在很短的时间内上手贵司的框架。"

### Q: "Claude Code 和我们的 Harness 不一样吧？"

> "底层实现肯定不一样——每个公司的 Harness 都是定制化的。但抽象层是相通的：任何 Agent Harness 都要解决工具调用、权限治理、记忆存储、流程编排、上下文管理这五个核心问题。Claude Code 是这五个维度目前最完整的实现之一，我在日常使用中对每个维度都有体感。这些设计原则迁移到贵司的 Harness 上，主要差异是 API 和配置语法，概念层是高度一致的。"

---

## 今晚记忆要点

1. **Agent = Model + Harness**——模型是引擎，Harness 是底盘（工具/编排/记忆/安全/CI/CD）
2. Harness 不是 Harness.io——面试官问的是 Agent Harness（驾驭层工程）
3. 八大抽象：Tool / MCP / Skill / SubAgent / Memory / Hook / Plugin / Goal-Plan
4. Hook 是唯一凌驾模型之上的机制——用于审计、熔断、合规（不可绕过）
5. Claude Code 是工业级 Harness 的完整实现——你每天在用，这是你的最强实践
6. 岗位要求全部可以映射到 Harness 概念（对话引擎 → Agent Loop，权限治理 → fail-closed 管线，CI/CD → L2 反馈循环）
7. 诚实策略：不说"用过贵司 Harness"，说"深度使用 Claude Code + 理解 Harness 架构，可快速迁移"
