# nanobot 架构深度分析

> 本文档基于 nanobot 源代码分析，用大白话解释它的工作原理和设计决策。

---

## 目录

1. [项目概述](#1-项目概述)
2. [整体架构](#2-整体架构)
3. [核心模块详解](#3-核心模块详解)
4. [设计决策背后的原因](#4-设计决策背后的原因)
5. [设计哲学总结](#5-设计哲学总结)

---

## 1. 项目概述

**nanobot** 是一个超轻量级的个人 AI 助手（Agent），灵感来自 OpenClaw，但代码量减少了 99%。

### 主要功能

- 连接各种聊天平台（Telegram、Discord、微信、飞书等）
- 支持多种 LLM 提供商（Claude、GPT、DeepSeek、OpenRouter 等）
- 内置工具（文件读写、Shell 执行、网页搜索、定时任务等）
- 长期记忆管理
- Skills 技能系统
- MCP (Model Context Protocol) 支持

---

## 2. 整体架构

### 核心流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户消息输入                              │
│  (CLI / Telegram / Discord / WeChat / Feishu / ...)            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        MessageBus (消息总线)                      │
│              解耦消息来源和 Agent 处理逻辑                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AgentLoop (总指挥)                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 1. 检查命令（如 /stop）                                   │  │
│  │ 2. 获取/创建 Session                                      │  │
│  │ 3. 检查是否需要 Memory Consolidation                      │  │
│  │ 4. 调用 ContextBuilder 构建提示词                         │  │
│  │ 5. 交给 AgentRunner 执行 LLM 循环                         │  │
│  │ 6. 保存会话，后台再检查 Memory Consolidation              │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ ContextBuilder  │  │  MemoryStore    │  │  SkillsLoader   │
│  (构建提示词)   │  │  (记忆管理)     │  │  (技能加载)     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     AgentRunner (LLM 循环)                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ for iteration in max_iterations:                          │  │
│  │   1. 调用 LLM                                             │  │
│  │   2. 如果有 tool_calls:                                   │  │
│  │      - 执行工具（支持并发）                               │  │
│  │      - 把工具结果加回消息历史                             │  │
│  │      - 继续下一轮                                         │  │
│  │   3. 否则: 返回最终回复                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ToolRegistry (工具注册表)                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 内置工具: read_file, write_file, exec, web_search, ...  │  │
│  │ MCP 工具: mcp_{server}_{tool}                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                         回复发送                                  │
│  (CLI / Telegram / Discord / WeChat / Feishu / ...)            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心模块详解

### 3.1 AgentLoop (`loop.py`)

**职责**：总指挥，协调整个 Agent 的运行。

**主要工作**：
- 从 MessageBus 接收消息
- 管理 Session（会话）
- 触发 Memory Consolidation
- 连接 MCP 服务器
- 调用 ContextBuilder 构建提示词
- 交给 AgentRunner 执行 LLM 循环
- 保存会话并发送回复

**关键数据结构**：
```python
class AgentLoop:
    bus: MessageBus              # 消息总线
    provider: LLMProvider        # LLM 提供商
    context: ContextBuilder      # 提示词构建器
    sessions: SessionManager     # 会话管理器
    tools: ToolRegistry          # 工具注册表
    runner: AgentRunner          # LLM 循环执行器
    memory_consolidator: MemoryConsolidator  # 记忆整合器
```

---

### 3.2 AgentRunner (`runner.py`)

**职责**：纯 LLM + 工具调用循环，不依赖任何产品级逻辑。

**核心循环**：
```python
for iteration in range(max_iterations):
    # 1. 调用 LLM
    response = provider.chat_with_retry(messages, tools, model)

    # 2. 如果有工具调用
    if response.has_tool_calls:
        # 把 LLM 回复加入消息历史
        messages.append(...)

        # 执行工具（支持并发）
        results = execute_tools(tool_calls)

        # 把工具结果加入消息历史
        for result in results:
            messages.append({"role": "tool", ...})

        # 继续下一轮
        continue

    # 3. 没有工具调用，结束
    final_content = response.content
    break
```

**为什么单独拆出来？**
- 可复用：subagent、后台任务都能用
- 可测试：不依赖文件系统、网络
- 关注点分离：只管 LLM 思考，不管消息怎么来

---

### 3.3 ContextBuilder (`context.py`)

**职责**：组装 System Prompt 和消息列表。

**System Prompt 组成顺序**：
```
1. Identity（身份）
   - "你是 nanobot"
   - 工作区路径
   - 平台策略（Windows/POSIX）

2. Bootstrap Files（引导文件）
   - AGENTS.md
   - SOUL.md（性格）
   - USER.md（用户信息）
   - TOOLS.md（工具说明）

3. Long-term Memory（长期记忆）
   - MEMORY.md 的内容

4. Always Skills（总是激活的技能）
   - 标记为 always=true 的技能

5. Skills Summary（所有可用技能列表）
   - 只给索引，需要时 Agent 自己读
```

**Runtime Context**：在每条用户消息前加：
```
[Runtime Context — metadata only, not instructions]
Current Time: 2026-03-30 10:30:00
Channel: telegram
Chat ID: 123456
```
→ 告诉 LLM 这只是元数据，不是指令。

---

### 3.4 Memory 系统 (`memory.py`)

**两层记忆设计**：

| 文件 | 作用 | 类比 |
|------|------|------|
| `MEMORY.md` | 长期记忆，重要事实 | 你记住的生日、住址 |
| `HISTORY.md` | 历史日志，可搜索的流水账 | 你的日记，哪天发生了什么 |

#### MemoryStore 类

**主要方法**：
- `read_long_term()`：读 MEMORY.md
- `write_long_term()`：写 MEMORY.md
- `append_history()`：追加到 HISTORY.md
- `consolidate()`：用 LLM 总结对话，更新记忆

**Consolidation 工作原理**：
```python
# 1. 给 LLM 看当前记忆 + 新对话
prompt = f"""
Current Memory: {current_memory}
Conversation: {messages}
"""

# 2. 强制 LLM 调用 save_memory 工具
response = provider.chat_with_retry(
    messages=...,
    tools=[save_memory_tool],
    tool_choice={"type": "function", "function": {"name": "save_memory"}}
)

# 3. 提取结果
history_entry = args["history_entry"]  # 一句话总结
memory_update = args["memory_update"]  # 更新后的完整记忆

# 4. 写入文件
append_history(history_entry)
write_long_term(memory_update)
```

#### MemoryConsolidator 类

**职责**：决定什么时候该 consolidation。

**触发条件**：当前会话历史 Token 数超过预算。

**预算公式**：
```
budget = context_window_tokens
         - max_completion_tokens  # 预留生成空间
         - _SAFETY_BUFFER          # 安全缓冲区（1024 token）
```

**Consolidation 步骤**：
1. 估算当前 Prompt Token 数
2. 如果超过 budget，找分界点（从 last_consolidated 开始）
3. 把这段消息拿去 consolidation
4. 更新 `session.last_consolidated` 标记

---

### 3.5 Skills 系统 (`skills.py`)

**什么是 Skill？** → 给 Agent 看的教程文件（SKILL.md）。

**Skill 存放位置**：
- 内置：`nanobot/skills/{skill-name}/SKILL.md`
- 用户自定义：`~/.nanobot/workspace/skills/{skill-name}/SKILL.md`

**Skill 格式**：
```markdown
---
description: "帮你管理 GitHub 仓库"
metadata: '{"nanobot": {"always": false, "requires": {"bins": ["git"]}}}'
---

这是 GitHub 技能。你可以用 git 命令来...
```

**SkillsLoader 主要方法**：
- `list_skills()`：列出所有可用技能
- `load_skill()`：加载某个技能内容
- `get_always_skills()`：获取 always=true 的技能
- `build_skills_summary()`：构建技能列表摘要

---

### 3.6 MCP 集成 (`tools/mcp.py`)

**MCP** = Model Context Protocol，让 Agent 使用外部工具的标准协议。

**支持的传输模式**：
1. `stdio`：本地进程（npx/uvx）
2. `sse`：SSE 远程
3. `streamableHttp`：HTTP 远程

**配置示例**：
```json
{
  "tools": {
    "mcpServers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
      },
      "my-remote": {
        "url": "https://example.com/mcp/",
        "headers": {"Authorization": "Bearer xxx"},
        "toolTimeout": 120
      }
    }
  }
}
```

**MCPToolWrapper 类**：
- 把 MCP 工具包装成 nanobot 工具
- 工具名：`mcp_{server_name}_{tool_name}`
- 处理超时、错误等

---

## 4. 设计决策背后的原因

### 4.1 为什么拆成 AgentLoop 和 AgentRunner？

| 原因 | 说明 |
|------|------|
| **可复用性** | `AgentRunner` 可以在 subagent、后台任务中复用 |
| **可测试性** | `runner.py` 不依赖文件系统、网络，易单元测试 |
| **关注点分离** | `loop.py` 管"消息怎么来"，`runner.py` 管"LLM 怎么想" |

---

### 4.2 Memory 为什么是两层纯文本文件？

| 原因 | 说明 |
|------|------|
| **可观测性** | 用户可以直接打开 `MEMORY.md` 看 Agent 记住了什么，不满意直接改 |
| **可调试性** | 纯文本，出问题一眼就能看明白 |
| **LLM 友好** | LLM 天生擅长读/写 Markdown，用数据库反而要序列化/反序列化 |
| **版本控制友好** | 纯文本文件可以 git 管理 |

---

### 4.3 Memory Consolidation 为什么用 LLM？

| 原因 | 说明 |
|------|------|
| **信息不会丢失** | 直接截断会丢重要信息，LLM 总结可以把关键信息提取到 MEMORY.md |
| **理解语义** | 向量数据库是相似度匹配，LLM 能理解"这段对话其实是在讲 X" |
| **灵活性** | 用户可以手动修改 MEMORY.md，LLM 会尊重并继续使用 |

---

### 4.4 Skills 为什么是 Markdown 文件？

| 原因 | 说明 |
|------|------|
| **LLM 能理解** | 不需要写插件代码，用自然语言教 Agent 就行 |
| **用户可扩展** | 任何人都能写技能，不需要懂 Python |
| **渐进式加载** | 先给技能列表，需要时 Agent 自己读详情，省 token |
| **可组合** | 多个技能可以一起用 |

---

### 4.5 上下文工程为什么这个顺序？

```
Identity → Bootstrap Files → Memory → Always Skills → Skills Summary
```

| 顺序 | 组件 | 原因 |
|------|------|------|
| 1 | Identity | 先定基调："你是谁"、"你的工作区在哪" |
| 2 | Bootstrap Files | 用户自定义覆盖默认身份 |
| 3 | Memory | 长期记忆很重要，放前面 |
| 4 | Always Skills | 常用技能直接加载 |
| 5 | Skills Summary | 只是索引，放最后 |

---

### 4.6 为什么用 MessageBus？

| 原因 | 说明 |
|------|------|
| **解耦** | AgentLoop 不需要知道消息来自 Telegram 还是 Discord |
| **异步友好** | 消息队列可以缓冲，避免阻塞 |
| **可扩展** | 想加新渠道，往总线发消息就行 |

---

### 4.7 MCP 工具为什么加前缀 `mcp_{server}_{tool}`？

| 原因 | 说明 |
|------|------|
| **避免命名冲突** | 两个 MCP 服务器可能都有 `read_file` |
| **可追溯** | 看工具名就知道来自哪个服务器 |
| **可筛选** | 可以通过前缀判断是不是 MCP 工具 |

---

### 4.8 为什么 Session 要分 `last_consolidated`？

| 原因 | 说明 |
|------|------|
| **保留近期上下文** | 最近的对话还在活跃，直接塞 Prompt 里，LLM 理解更好 |
| **减少 LLM 调用** | 不需要每次都总结所有消息，只总结旧消息 |
| **渐进式** | 消息多了才总结，消息少的时候不折腾 |

---

## 5. 设计哲学总结

nanobot 的核心设计理念：

| 原则 | 说明 |
|------|------|
| **简单优于复杂** | 纯文本文件 > 数据库 |
| **可观测性** | 人类能直接读/改所有状态 |
| **LLM 优先** | 用 LLM 能理解的方式（Markdown、自然语言） |
| **关注点分离** | 每个模块只做一件事 |
| **省 Token** | 渐进式加载、consolidation、截断 |
| **可扩展性** | 插件化设计，容易加新功能 |

这就是为什么它叫 "ultra-lightweight" — 用最少的代码实现最核心的功能！
