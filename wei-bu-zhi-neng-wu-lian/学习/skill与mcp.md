# Skill 与 MCP —— Agent Harness 插件体系

---

## 一、Skill（技能）

### 1.1 定义

Skill 是 Agent Harness 中的**可复用能力包**。本质公式：

```
Skill = 一段 Prompt 指令 + 工具权限声明 + 触发条件
```

核心设计理念：**渐进披露（Progressive Disclosure）**——不调用不加载，只在触发时才将指令注入上下文。

### 1.2 Claude Code 中的真实形态

**目录结构**：
```
.claude/
└── skills/
    ├── review.md        # /review 代码审查
    ├── dataviz.md       # /dataviz 数据可视化
    └── loop.md          # /loop 循环执行
```

**Skill 文件结构**：
```markdown
---
name: review
description: 代码审查——扫描变更，输出分级问题
---

# 代码审查 Skill

## 触发条件
用户输入 `/review` 或提到"帮我 review 代码"

## 审查规则
1. **正确性**：逻辑错误、边界条件
2. **安全性**：SQL 注入、XSS、密钥泄露
3. **性能**：不必要的循环、N+1 查询
4. **可维护性**：命名、注释、重复代码

## 输出格式
按严重程度分级（🔴高 🟡中 🟢低），每项包含：文件、行号、问题描述、修复建议
```

### 1.3 触发方式

| 方式 | 示例 | 场景 |
|------|------|------|
| 用户命令 | `/review` | 用户主动触发 |
| 关键词匹配 | "帮我 review 一下" | 自然语言触发 |
| 事件触发 | pre-commit hook | 自动化流程触发 |
| 条件触发 | 当 diff 超过 500 行时 | 智能触发 |

### 1.4 渐进披露的优势

**问题**：如果把所有 Skill 的 Prompt 都常驻在 System Prompt 中，假设有 20 个 Skill，每个 500 tokens，那就是 10K tokens——占上下文但不一定用得上。

**解决**：Skill 按需加载。
- 不调用时：只占一个名字（"可用技能：review, dataviz, loop" ~ 10 tokens）
- 调用时：完整 Prompt 注入（~500 tokens）
- 用完即释放：Skill 完成使命后，指令从活跃上下文中清除

**效果**：20 个 Skill → 平时只占 10 tokens，用到哪个加载哪个。

### 1.5 Skill 开发流程

```
需求分析 → Prompt 指令设计 → 工具权限绑定 → 内测验证 → 灰度发布
```

**数字人 Skill 案例：**

#### 案例 1：人设切换 Skill

```markdown
---
name: switch-persona
description: 切换数字人人设模式
trigger: /persona {mode}
---

# 人设切换 Skill

## 模式
- `friend`：知心朋友（温暖、共情、轻松）
- `coach`：人生教练（积极、鼓励、引导）
- `storyteller`：讲故事模式（生动、富有想象力）

## 切换逻辑
1. 更新 System Prompt 中的人设描述
2. 注入对应模式的语气示例（few-shot）
3. 提示用户："已切换到 {mode} 模式~"
```

#### 案例 2：倾听者 Skill

```markdown
---
name: active-listener
description: 深度倾听模式——当检测到用户情绪低落时自动激活
trigger: 情感检测 → 负面情绪 > 阈值
---

# 倾听者 Skill

## 行为约束
- 不急于给建议（除非用户明确要求）
- 多用反射式倾听（"听起来你感到..."）
- 适时追问细节（"当时发生了什么？"）
- 避免空洞安慰（"一切都会好的"）

## 退出条件
- 用户情绪评分恢复到中性以上
- 用户主动切换话题
```

---

## 二、MCP（Model Context Protocol）

### 2.1 定义

MCP 是 Anthropic 发布的**开放协议**，被业界称为**"AI 世界的 USB 接口"**。

它定义了 LLM 与外部工具/数据源之间的**标准化连接方式**，解决了"每个 AI 应用都要自己写各种 API 对接"的问题。

### 2.2 三大原语

| 原语 | 作用 | 示例 |
|------|------|------|
| **Tools** | 模型可调用的函数 | 查数据库、发邮件、调 API |
| **Resources** | 模型可读取的数据 | 文件内容、数据库记录、API 响应 |
| **Prompts** | 预定义的 Prompt 模板 | "帮我写一封英文邮件"的模板 |

### 2.3 Client-Server 架构

```
┌──────────────┐         ┌──────────────────┐
│  MCP Client  │ ◄─────► │   MCP Server     │
│  (Harness)   │  协议    │ (工具/数据提供方)  │
└──────────────┘         └──────────────────┘

工作流程：
1. 启动 (Launch)      —— Client 启动 Server 进程
2. 握手 (Handshake)   —— 协商协议版本和能力
3. 发现 (Discovery)   —— Server 告知 Client 它有哪些 Tools/Resources/Prompts
4. 注册 (Register)    —— Client 将工具注册到 Agent 的可用工具列表
5. 调用 (Invoke)      —— 模型决策调用 → Client 转发 → Server 执行 → 返回结果
```

### 2.4 Claude Code 中的配置

**方式一：命令行**
```bash
claude mcp add myserver -- node server.js
```

**方式二：配置文件（`.mcp.json`）**
```json
{
  "mcpServers": {
    "database": {
      "command": "python",
      "args": ["-m", "mcp_server_postgres"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-filesystem", "/path/to/data"]
    }
  }
}
```

### 2.5 MCP Server 开发示例（Python）

```python
from mcp.server import Server, stdio_server
from mcp.types import Tool, TextContent

server = Server("memory-server")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="remember_user_fact",
            description="存储用户的个人事实到长期记忆",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_id": {"type": "string"},
                    "fact": {"type": "string", "description": "要记住的事实"},
                    "category": {"type": "string", "enum": ["basic_info", "preference", "event", "emotion"]}
                },
                "required": ["user_id", "fact"]
            }
        ),
        Tool(
            name="search_memory",
            description="检索用户相关的记忆",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_id": {"type": "string"},
                    "query": {"type": "string"},
                    "top_k": {"type": "integer", "default": 5}
                },
                "required": ["user_id", "query"]
            }
        ),
        Tool(
            name="forget",
            description="删除指定的记忆（合规/用户要求）",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_id": {"type": "string"},
                    "memory_id": {"type": "string"}
                },
                "required": ["user_id", "memory_id"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "remember_user_fact":
        # 生成 embedding + 写入 pgvector
        return [TextContent(type="text", text=f"已记住：{arguments['fact']}")]
    elif name == "search_memory":
        # 语义检索
        results = pgvector_search(arguments["user_id"], arguments["query"], arguments.get("top_k", 5))
        return [TextContent(type="text", text=format_results(results))]
    elif name == "forget":
        # 删除
        pgvector_delete(arguments["user_id"], arguments["memory_id"])
        return [TextContent(type="text", text="已删除")]

if __name__ == "__main__":
    import asyncio
    asyncio.run(stdio_server(server))
```

---

## 三、Skill vs MCP —— 关系与分工

### 3.1 核心区别

| 维度 | Skill | MCP |
|------|-------|-----|
| **本质** | 行为封装 | 能力接入协议 |
| **视角** | 使用者视角（"帮我做 X"） | 提供者视角（"我提供 Y 能力"） |
| **内容** | Prompt 指令 + 工具权限 | Tools / Resources / Prompts |
| **加载方式** | 按需懒加载（渐进披露） | 启动时注册，常驻工具列表 |
| **生命周期** | 调用时激活，用完释放 | Server 进程持续运行 |
| **上下文成本** | 按需注入（低） | 工具定义常驻（低，仅元数据） |

### 3.2 一句话总结

> **Skill 声明"做什么"（行为封装），MCP 提供"用什么做"（能力接入）。**
>
> 一个 Skill 可以调用多个 MCP 工具来完成复杂任务。

### 3.3 协作示例

```
用户: "/remember-mode" (触发"记忆模式" Skill)

Skill "记忆模式" 激活：
  1. 加载 Prompt: "用温暖的方式回顾和用户的美好回忆"
  2. 调用 MCP Server "记忆服务"：
     ├─ search_memory(user_id, "美好回忆", top_k=5)
     └─ → 获得 5 条相关记忆
  3. LLM 用获得的记忆生成一段温暖的对话
  4. Skill 释放，上下文回到正常模式
```

---

## 四、数字人场景 Skill/MCP 矩阵

### 4.1 MCP Server 规划

| MCP Server | 提供的工具 | 用途 |
|-----------|----------|------|
| **记忆服务** | `search_memory`、`remember_fact`、`forget` | 用户记忆的增删查 |
| **人设管理** | `get_persona`、`update_persona_trait` | 读取/更新人设配置 |
| **外部 API** | `get_weather`、`get_news`、`search_web` | 实时信息获取 |
| **多模态** | `generate_image`、`text_to_speech` | 图片/语音生成 |

### 4.2 Skill 规划

| Skill | 触发条件 | 调用的 MCP | 用途 |
|-------|---------|-----------|------|
| `switch-persona` | `/persona {mode}` | 人设管理 MCP | 切换人设模式 |
| `active-listener` | 情感检测 | 记忆服务 MCP | 深度倾听模式 |
| `daily-briefing` | 每日首次对话 | 记忆 + 外部 API MCP | 每日问候+个性化推送 |
| `memory-review` | `/remember` | 记忆服务 MCP | 回顾美好回忆 |

---

## 五、面试话术

> "Skill 和 MCP 是 Agent Harness 插件体系的两个核心概念。我每天都在 Claude Code 中使用它们。
>
> **Skill** 是行为封装——本质是一段 Prompt 指令加上工具权限，按需懒加载。比如我用 `/review` 触发代码审查 Skill，它只在调用时才把审查规则注入上下文，用完就释放——这就是渐进披露的设计理念，避免上下文被不活跃的指令占满。
>
> **MCP** 是能力接入协议——被业界叫做'AI 世界的 USB 接口'。它定义了 Tools、Resources、Prompts 三种原语，通过 Client-Server 架构让 Agent 标准化地接入外部能力。比如一个'记忆 MCP Server'暴露了 search_memory、remember_fact 等工具——任何 Agent（不管是 Claude Code 还是贵司的 Harness）只要支持 MCP 协议，就能直接使用。
>
> **两者的关系**：Skill 回答'做什么'，MCP 回答'用什么做'。比如一个'记忆回顾 Skill'——它封装了'如何温柔地回忆'的行为逻辑，但它实际检索记忆时调的是'记忆 MCP Server'的 search_memory 工具。
>
> 如果让我为贵司的数字人产品设计插件，我会围绕人设、记忆、外部能力三个维度规划 MCP Server，然后在上面搭建面向用户场景的 Skill——这样底层能力复用、上层行为灵活组合。"

---

## 今晚记忆要点

1. **Skill** = 可复用能力包（Prompt + 工具权限），渐进披露，按需懒加载
2. **MCP** = AI 世界的 USB 接口，三大原语（Tools/Resources/Prompts），Client-Server 架构
3. **Skill vs MCP**：Skill 声明做什么（行为封装），MCP 提供用什么做（能力接入）
4. MCP 工作流程：启动 → 握手 → 发现 → 注册 → 调用 → 返回
5. 渐进披露的优势：不调用不占上下文，20 个 Skill 平时只占 10 tokens
6. Claude Code 真实经验：`/review`、`/dataviz`、`/loop` 的日常使用
7. 数字人插件案例：记忆 MCP Server + 人设切换 Skill + 倾听者 Skill
