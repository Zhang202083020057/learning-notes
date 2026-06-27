# Claude Code Harness 工程视角 — 补充笔记

> 基于 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 仓库的学习补充
>
> 与上一篇《Claude Code 架构设计研究报告》的差异：上一篇是**产品分析报告**（"它是什么"），这一篇是**工程教育视角**（"怎么造"）。

---

## 一、核心哲学：Agency 来自模型，不是代码

learn-claude-code 仓库开篇就给出了一句我上一篇报告没有抓住的话：

> **"Agency — 感知、推理、行动的能力 — 来自模型训练，不是来自外部代码的编排。Agent 产品 = 模型 + Harness，缺一不可。"**

### 心智转换：从 "开发 Agent" 到开发 Harness

当一个人说"我在开发 Agent"时，只有两个可能的意思：

1. **训练模型**：通过 RL/微调/RLHF 调整权重。这是 DeepMind、OpenAI、Anthropic 在做的事。
2. **构建 Harness**：编写代码，为模型提供可操作的环境。**这是我们大多数人在做的事。**

仓库把"提示词管道工"（prompt plumber）——用 if-else 分支、节点图、硬编码路由拼出来的"Agent 平台"——称为**"GOFAI 的现代还魂"**。真正的 Agent 不需要那些：模型负责智能，Harness 负责执行环境。

### 历史铁证

| 年份 | 事件 | 核心事实 |
|------|------|----------|
| 2013-2015 | DeepMind DQN 玩 Atari | 一个神经网络，只看像素和分数，学会 49 款游戏 |
| 2019 | OpenAI Five 征服 Dota 2 | 10 个月自我对弈 45000 年，2-0 击败世界冠军 OG |
| 2019 | AlphaStar 制霸星际争霸 II | 宗师段位，前 0.15% |
| 2019 | 腾讯绝悟统治王者荣耀 | 一天等于人类 440 年训练，击败 KPL 职业选手 |
| 2024-2026 | LLM Agent 重塑软件工程 | Claude/GPT 在人类全部代码上训练，部署为编程 Agent |

每个里程碑都指向同一个事实：**Agency 是训练出来的，不是编出来的。** 但每个 Agent 都需要 Harness（Atari 模拟器/Dota 2 客户端/IDE+终端）才能工作。

---

## 二、Harness 的定义

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

    Tools:          文件读写、Shell、网络、数据库、浏览器
    Knowledge:      产品文档、领域资料、API 规范、风格指南
    Observation:    git diff、错误日志、浏览器状态、传感器数据
    Action:         CLI 命令、API 调用、UI 交互
    Permissions:    沙箱隔离、审批流程、信任边界
```

**Harness 工程师真正的工作**：

| 职责 | 含义 | 对应章节 |
|------|------|----------|
| 实现工具 | 给 Agent 双手 | s01, s02 |
| 策划知识 | 给 Agent 领域专长（按需加载） | s07 |
| 管理上下文 | 子 Agent 隔离 + 压缩 + 任务持久化 | s06, s08, s12 |
| 控制权限 | 沙箱边界 + 审批流程 | s03 |
| 收集任务过程数据 | 感知-推理-行为轨迹 → 训练下一代 Agent 的原材料 | 全局 |

---

## 三、核心模式：Agent Loop（s01）

整个 Claude Code 的灵魂就是下面这个循环：

```python
def agent_loop(messages):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return  # 模型决定停止，对话结束

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = TOOL_HANDLERS[block.name](**block.input)
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
        # 循环继续 → 模型看到工具结果后决定下一步
```

关键洞察：
- **循环属于 Agent，机制属于 Harness**。20 个章节每章加一个机制，循环本身始终不变。
- s01 只有 1 个工具（bash）就能工作。"One loop & Bash is all you need"
- s02 加工具只加一个 handler，dispatch map 不变
- s20 全部机制合体：27 个工具 + hooks + 权限 + 任务 + 团队 + MCP

---

## 四、20 章递进架构全景

```
阶段一：能动手（s01-s05）
  s01  Agent Loop        "One loop & Bash is all you need"
  s02  Tool Use          "加一个工具，只加一个 handler"
  s03  Permission        "先划边界，再给自由"
  s04  Hooks             "挂在循环上，不写进循环里"
  s05  TodoWrite         "没有计划的 agent 走哪算哪"

阶段二：能做复杂任务（s06-s11）
  s06  Subagent          "大任务拆小，每个小任务干净上下文"
  s07  Skill Loading     "用到时再加载，别全塞 prompt 里"
  s08  Context Compact   "上下文总会满，要有办法腾地方"
  s09  Memory            "记住该记的，忘掉该忘的"
  s10  System Prompt     "prompt 是组装出来的，不是写死的"
  s11  Error Recovery    "错误不是终点，是重试的起点"

阶段三：能记住和恢复（s12-s14）
  s12  Task System       "大目标拆成小任务，排好序，持久化"
  s13  Background Tasks  "慢操作丢后台，agent 继续思考"
  s14  Cron Scheduler    "定时触发，不需要人推"

阶段四：能协作、能扩展（s15-s20）
  s15  Agent Teams       "一个搞不定，组队来"
  s16  Team Protocols    "队友之间要有约定"
  s17  Autonomous Agents "队友自己看板，有活就认领"
  s18  Worktree Isolation "各干各的目录，互不干扰"
  s19  MCP Plugin        "能力不够？插上 MCP"
  s20  Comprehensive     "机制很多，循环一个"
```

> 来源：[learn-claude-code README](https://github.com/shareAI-lab/learn-claude-code)

---

## 五、关键机制深度解析

### 5.1 Subagent（s06）：上下文隔离

**问题**：Agent 修 bug 时读了 30 个文件、聊了 60 轮。messages 涨到 120 条，大部分是"追踪调用链"的中间过程，和"修 bug"无关。Agent 越来越"健忘"。

**方案**：`task` 工具 spawn 一个子 Agent，拥有全新的 `messages[]`，跑自己的循环，结束只回传摘要文本。

```python
def spawn_subagent(description: str) -> str:
    sub_tools = [bash, read_file, write_file, edit_file, glob]
    # 注意：子 Agent 没有 task 工具，禁止递归 spawn
    messages = [{"role": "user", "content": description}]  # 全新 messages[]
    for _ in range(30):  # 安全限制
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM, messages=messages, tools=sub_tools)
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        # 执行工具...
    return extract_text(messages[-1]["content"])  # 只回传结论
```

三个关键设计决策：

| 决策 | 选择 | 原因 |
|------|------|------|
| 上下文隔离 | 全新 `messages[]` | 子 Agent 的中间过程不污染主 Agent |
| 只回传结论 | `extract_text(last_message)` | 不是回传整个 messages 列表 |
| 禁止递归 | 子 Agent 无 task 工具 | 防止子 Agent 再 spawn 新的子 Agent |
| 安全策略不跳过 | 子 Agent 工具调用也走 PreToolUse hook | 上下文隔离不代表权限隔离 |

> **真实 CC 源码的三种模式**（教学版简化了）：
>
> 1. **Normal Subagent**：全新 messages[]，只有 prompt
> 2. **Fork Subagent**：通过 `buildForkedMessages()` 构造 cache-friendly 前缀，共享 prompt cache
> 3. **General-Purpose**：同 Normal
>
> Fork 模式的核心目的是让 Anthropic API 的 prompt cache 命中——父子 Agent 的 system prompt、tools、messages 前缀完全一致，API 端不需要重算。

### 5.2 Agent Teams（s15）：文件收件箱 + 队友线程

**问题**："重构整个后端"涉及认证、数据库、API、测试。单个 Agent 的上下文窗口覆盖不了所有模块。s06 的子 Agent 是临时工，但有些任务需要能通信、能协作的队友。

**子 Agent vs 队友**：

| | s06 子 Agent | s15 队友 |
|---|---|---|
| 生命周期 | 一次性，用完销毁 | 多轮（真实 CC 用 idle loop） |
| 通信 | 只回传结论 | 异步收件箱，随时通信 |
| 上下文 | 完全隔离 | 通过消息共享信息 |
| 数量 | 一个主 + 偶尔子 | 一个 Lead + 多个队友 |

**MessageBus：文件收件箱**

每个 Agent（包括 Lead 和队友）有一个 `.jsonl` 邮箱。发消息 = append 一行 JSON。读消息 = 读文件 + 删除（消费式）：

```
~/.claude/teams/{team}/inboxes/{agentName}.json
```

为什么用文件而不是内存队列？直观、跨线程可观察。真实 CC 用 `proper-lockfile` 防并发写冲突。

**15 种消息类型**（真实 CC，`teammateMailbox.ts`）：

| 类型 | 方向 | 用途 |
|------|------|------|
| `plain text` | 双向 | 普通队友间通信 |
| `idle_notification` | 队友→Lead | 队友完成一轮工作 |
| `permission_request` | 队友→Lead | 队友需要操作审批 |
| `permission_response` | Lead→队友 | Lead 审批结果 |
| `shutdown_request` | Lead→队友 | 请求体面关机 |
| `shutdown_approved` | 队友→Lead | 确认关机 |
| `task_assignment` | Lead→队友 | 分配任务 |
| `plan_approval_request` | 队友→Lead | 队友提交计划待审 |

**权限冒泡**（真实 CC）：

1. 队友遇到需要审批的操作 → 发 `permission_request` 到 Lead 收件箱
2. Lead 的 `useInboxPoller`（每 1 秒轮询）检测到 → 路由到审批队列
3. 用户审批后 → Lead 发 `permission_response` 回队友
4. 队友的 `useSwarmPermissionPoller`（每 500ms 轮询）收到 → 继续或拒绝

### 5.3 Context Compact（s08）：四层压缩

**问题**：上下文窗口有限，Agent 运行久了 messages 会溢出。

从便宜到贵：

```
Snip Compact  →  Microcompact  →  Context Collapse  →  Auto Compact
   (免费)          (低API成本)       (低信息损失)        (高API成本)
  直接裁剪         替换占位符         折叠摘要          模型总结
```

教学版的压缩在 LLM 调用前跑，真实 CC 还有 reactive compact（prompt too long 时触发）。

### 5.4 Task System（s12）：跨会话持久化

与 s05 的 `todo_write`（内存内、当前会话）不同，s12 的任务图是**文件持久化**的：

```
.tasks/task_001.json  →  {"id": "task_001", "title": "...", "deps": [], "status": "pending"}
.tasks/task_002.json  →  {"id": "task_002", "title": "...", "deps": ["task_001"], "status": "claimed"}
```

- Agent 崩溃或重启后，任务还在
- 多 Agent 可以通过任务文件协作
- 队友认领任务（s17）依赖任务图

### 5.5 s20 全景：所有组件在循环中的位置

s20 把所有机制放回同一个可运行系统：

```
用户输入
  → UserPromptSubmit hooks
  → cron/background 通知注入
  → context compact（压缩管线）
  → memory + skills + MCP 状态组装 system prompt
  → LLM 调用（带 error recovery）
  → 有 tool_use block？
      否 → Stop hooks → 返回
      是 → PreToolUse hooks + permission
          → TOOL_HANDLERS / MCP handlers / background dispatch
          → PostToolUse hooks
          → tool_result / task_notification 回 messages
          → 下一轮
```

s20 的工具池包含 27 个工具，`assemble_tool_pool()` 每轮组装 `BUILTIN_TOOLS + connected MCP tools`。

---

## 六、与上一篇报告的视角差异总结

| | 产品分析报告 | Harness 工程视角 |
|---|---|---|
| **问题意识** | "这个产品怎么设计的" | "怎么从零造一个" |
| **核心命题** | 组件清单 + 竞品对比 | Agency 从哪来 + Harness 怎么做 |
| **输出** | 架构图 + 对比表 + 要点 | **可运行的 Python 代码**（20 个 code.py） |
| **对提示词工程的态度** | 未涉及 | 明确批判 "prompt plumber" |
| **对安全的理解** | 八层防线清单 | 权限冒泡：bubble to parent |
| **对子 Agent 的理解** | "上下文隔离" | 三种模式：Normal/Fork/General-Purpose，Fork 为共享 prompt cache |
| **对团队协作的理解** | "多 Agent 并行" | 文件 JSONL 收件箱 + 15 种消息类型 + 权限冒泡轮询 |
| **可运行** | ❌ | ✅ `python s01_agent_loop/code.py` |

---

## 七、为什么这个仓库值得学

1. **它不是"讲解 Claude Code"，是"教你造 Harness"**。学完之后理解的不只是 Claude Code 怎么工作，而是适用于任何领域、任何 Agent 的 Harness 工程通用原则。

2. **哲学正确**：Agency 是模型训练出来的，工程师的职责是把 Harness 造好。"造好 Harness，Agent 会完成剩下的。"

3. **能跑**：每个章节有独立的 `code.py`，从 s01 的一个 bash 工具到 s20 的 27 个工具 + 团队 + MCP，递进清晰。

4. **模式可泛化**：

```
庄园管理 Agent  = 模型 + 物业传感器 + 维护工具 + 租户通信
农业 Agent      = 模型 + 土壤/气象数据 + 灌溉控制 + 作物知识
酒店运营 Agent  = 模型 + 预订系统 + 客户渠道 + 设施 API
制造业 Agent    = 模型 + 产线传感器 + 质量控制 + 物流系统
```

循环永远不变。工具在变，知识在变，权限在变。**Agent = 模型 + Harness。**

---

## 八、快速上手

```bash
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
cp .env.example .env   # 填入 ANTHROPIC_API_KEY

python s01_agent_loop/code.py        # 起点 — 一个循环 + bash
python s06_subagent/code.py          # 子 Agent 上下文隔离
python s15_agent_teams/code.py       # Agent 团队协作
python s20_comprehensive/code.py     # 终点章: 全部机制合体
```

---

## 九、关键来源

- [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) — 本笔记的主要来源
- [s01: Agent Loop 源码](https://github.com/shareAI-lab/learn-claude-code/blob/main/s01_agent_loop/code.py)
- [s06: Subagent 设计文档](https://github.com/shareAI-lab/learn-claude-code/blob/main/s06_subagent/README.md)
- [s15: Agent Teams 设计文档](https://github.com/shareAI-lab/learn-claude-code/blob/main/s15_agent_teams/README.md)
- [s20: Comprehensive Agent 设计文档](https://github.com/shareAI-lab/learn-claude-code/blob/main/s20_comprehensive/README.md)

---

*本笔记是对上一篇报告的 Harness 工程视角补充，基于 2026 年 6 月 learn-claude-code 仓库最新内容编写。*
