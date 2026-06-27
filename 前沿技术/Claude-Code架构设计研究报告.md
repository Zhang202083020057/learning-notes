# Claude Code 架构设计深度研究报告

> 研究日期：2026-06-27  
> 研究方向：Anthropic Claude Code Agent 架构设计与实现

---

## 一、概述与定位

Claude Code 是 Anthropic 推出的终端 AI 编程代理（agentic coding tool）。官方定义其为"an agent that reads your codebase, edits files, and runs commands across your terminal, IDE, desktop app, and browser"。它直接运行在命令行中，能理解整个项目代码库、编辑文件、执行Shell命令。

### 核心设计哲学：KISS (Keep It Simple, Stupid)

Claude Code 的成功并非来自复杂的技术堆砌，而是一种极致的克制和简单化。AI公司MinusX团队通过拦截并分析Claude Code的每个网络请求，得出了一个反直觉的结论：Claude Code几乎在每个技术路线上都选择了架构层面的简化，包括单一主循环、用搜索而非RAG、朴素清晰的To-Do列表等。

团队负责人Boris Cherny的名言代表了其核心范式转变："我已经不写 prompt 了，我写 loop。"

> 来源：[Anthropic官方](https://www.anthropic.com/claude-code) | [MinusX团队分析](http://m.toutiao.com/group/7542490225214407178/)

---

## 二、整体架构设计

### 2.1 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| UI渲染 | React 18 + Ink | 终端UI渲染、权限对话框、进度条、多面板布局 |
| 核心语言 | TypeScript | 约4600+源文件、55+目录 |
| 平台 | Node.js (npm包) | 跨平台CLI工具 |

### 2.2 架构分层

```
+---------------------+     +---------------------------+
|    CLI 层            |     |    外围系统                |
+---------------------+     |  - 工具体系 (Bash/Read/  |
                             |    Write/Edit/Glob/Grep) |
+---------------------+     |  - MCP 客户端              |
|    核心引擎          |     |  - 安全层 (8层防护)       |
|  - query.ts(主循环)  |<--->|  - 配置系统               |
|  - 权限系统           |     |  - Bridge (远程控制)      |
|  - 配置解析           |     +---------------------------+
|  - 上下文压缩         |
|  - 工具调度           |
+---------------------+
          |
+---------------------+
|   Claude API 层      |
|  - 流式模型响应       |
|  - 工具调用协议       |
|  - fallback 机制      |
+---------------------+
```

> 来源：[解读Claude Code核心架构](http://m.toutiao.com/group/7635163948036063784/) | [source map泄露分析](http://m.toutiao.com/group/7623801447620854323/)

---

## 三、核心组件详解

### 3.1 智能体主循环 (Agentic Loop) -- query.ts

Claude Code 的核心是一个**异步生成器式主循环**，位于 `query.ts` 文件中。

#### 3.1.1 六步处理管线

```
请求前压缩 → 流式调用 → 错误恢复 → 停止钩子 → 工具执行 → 下一轮准备
```

#### 3.1.2 状态管理关键设计

- `QueryParams`：不可变输入参数（初始消息、系统提示、工具权限、最大轮次等）
- `State`：每轮变化的运行态（消息列表、工具上下文、压缩跟踪、turnCount等）
- `Continue Site`：一次性构造新state（类似React setState），避免半更新状态

#### 3.1.3 九类退出条件

1. 用户终止
2. 工具调用达到最大轮次
3. Token 预算耗尽
4. 输出超限
5. 任务完成
6. 最大输出恢复次数超限
7. 无工具待执行
8. Stop Hook 通过
9. 收益递减检测（连续多轮新增token极少）

#### 3.1.4 单一主线程 + 一层子代理分支

坚持**单一主线程**，所有工作发生在一条扁平的消息历史上。需要处理复杂任务时，主Agent可以**派生克隆子Agent**，但**严格限制不再继续派生**，任务树最多一层分支。

#### 3.1.5 小模型大量使用

超过50%的API调用落在便宜的claude-haiku上，用于读大文件、解析网页、处理Git历史、总结长上下文、生成UI进度标签等辅助任务。

> 来源：[解读Claude Code核心架构](http://m.toutiao.com/group/7635163948036063784/)

### 3.2 文件系统交互

Claude Code 不使用传统的 RAG（检索增强生成）+ 向量化代码库，而是让LLM**像人一样去搜索**：

| 方式 | 传统工具 (Cursor) | Claude Code |
|------|-------------------|-------------|
| 代码检索 | 向量化代码库 + 语义相似度搜索 | LLM生成grep/ripgrep/find/jq命令直接搜索 |
| 特点 | "猜"你可能需要哪些文件 | 100%确定且精准，先看结构再决定深度 |
| 优势 | 快速 | 行为可学习、可调试，模型可通过RL优化搜索策略 |

**工具分层**：
- **低层**：Bash、Read、Write -- 保证灵活性
- **中层**：Edit、Grep、Glob -- 高频动作封装
- **高层**：Task、WebFetch、Agent -- 降低用户感知延迟

### 3.3 命令执行与安全模型（八层防线）

**危险Bash模式硬编码阻止**：
- 解释器：python/node/ruby (尤其 `python -c` 和 `node -e`)
- 网络工具：curl/wget/nc
- 加密工具：openssl
- 远程访问：ssh/scp
- 提权操作：sudo/docker
- 文件权限：chmod/chown/mount

**八层安全模型**：

1. **构建门 (Build Gate)**：构建期裁剪未发布功能分支
2. **特性开关 (GrowthBook)**：服务端kill switch
3. **配置规则**：来自8类来源，合并优先级
4. **危险模式检测**：硬编码阻止高风险操作
5. **路径校验**：文件系统边界保护
6. **AI看AI分类器**：Auto mode下额外分类器判断工具使用安全性
7. **信任对话框**：新项目首次运行时展示潜在风险
8. **拒绝回退**：分类器过度保守时自动回退到提示模式

### 3.4 上下文管理 -- 四层压缩体系

| 层级 | 方式 | 成本 | 信息损失 | 适用场景 |
|------|------|------|----------|----------|
| **Snip Compact** | 直接剪掉较早消息块 | 无API成本 | 高 | headless后台会话 |
| **Microcompact** | 选择性替换旧工具结果为占位符 | 低 | 中 | 需保护prompt cache |
| **Context Collapse** | 预览/提交两阶段，折叠摘要存入单独store | 低 | 低 | 类似数据库view |
| **Auto Compact** | 模型总结整段会话 | 高 | 最低 | 成本最高的常规手段 |

**Token告警状态机**：Normal (剩余>20K) → Warning (13K-20K) → Error (3K-13K) → AutoCompact → BlockingLimit

> 来源：[治理Claude Code上下文膨胀](http://m.toutiao.com/group/7655668602405782016/) | [Claude Code上下文管理](http://m.toutiao.com/group/7651586180177248794/)

---

## 四、关键技术体系

### 4.1 Tool Use (工具调用)

约20个常驻工具：Bash、Read、Write、Edit、Glob、Grep、WebSearch、WebFetch、Task、Agent、TodoWrite等。另有条件工具按环境启用和特性门控工具。

**ToolUseContext**：每个工具运行在统一上下文中，包含工具列表、主循环模型、MCP客户端、预算上限、AbortController、文件读取缓存、权限跟踪等。

### 4.2 MCP (Model Context Protocol)

MCP是Anthropic在2024年底提出的开放协议，被比喻为"AI时代的USB-C"：标准化了AI与外部世界的连接方式。

### 4.3 PTC (Programmatic Tool Calling)

解决传统工具调用"乒乓球效应"：LLM推理→生成JSON工具调用→后台执行→结果返回→LLM再推理→下一个调用...

**PTC解决方案**：LLM直接编写并执行一段Python代码，代码中包含对多次MCP工具的调用、循环、条件判断，只有最终结果返回给LLM上下文。

> 来源：[什么是PTC](https://juejin.cn/post/7653050742082338862)

### 4.4 Skills (技能) -- "知识胶囊"

Skills是模块化给LLM注入专业技能的机制。物理上是一个文件夹，包含SKILL.md、相关脚本/代码、其他资源文件。

**渐进式披露三层机制**：
1. **第一层**：仅加载name和description注入系统Prompt
2. **第二层**：匹配后才加载完整SKILL.md
3. **第三层**：使用技能时按需读取文档/模板/脚本

### 4.5 Sub-agents (子代理)

- **主AI**：总指挥，分析任务并决定是否委派给专家
- **子代理**：各领域专家，有独立的System Prompt、上下文记忆、可用工具

四种核心优势：
1. 上下文隔离：每个子代理在独立上下文窗口中运行
2. 专业化配置：独立的System Prompt、模型选择、可用工具
3. 可重用性：用户级子代理跨项目复用
4. 灵活权限：精细控制每个子代理可用的工具

> 来源：[Claude Code Sub-agent模式详解](https://juejin.cn/post/7539839677826809883)

### 4.6 Dynamic Workflows (动态工作流)

2026年5月底的重大更新，标志着从"一个聪明的编码Agent"走向"能自写编排器的Agent系统"。

**六种官方编排模式**：

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| **Classify-and-act** | 分类后路由 | 先判断任务类型再分配 |
| **Fan-out-and-synthesize** | 扇出合成 | 大批量同质步骤并行 |
| **Adversarial verification** | 对抗验证 | 安全审计、架构方案 |
| **Generate-and-filter** | 生成过滤 | 大量生成候选后按rubric去重 |
| **Tournament** | 锦标赛 | 多Agent同题竞争 |
| **Loop until done** | 循环直到停 | 未知工作量，直到无新发现 |

**Bun Zig-to-Rust迁移案例**：产出约75万行Rust代码，测试通过率约99.8%，数百Agent并行。

> 来源：[Dynamic Workflows来了](https://cloud.tencent.com/developer/article/2694452) | [Anthropic官方博客](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)

### 4.7 CLAUDE.md -- 协作式记忆与偏好

CLAUDE.md是Claude Code的项目上下文文件，在每次请求中都以user message形式注入。它承载了代码库推断不出的隐性知识：忽略的目录、强制使用的库、测试命令、代码风格与约束。

---

## 五、与Cursor、Copilot、Windsurf的架构差异

### 5.1 核心定位对比

| 维度 | Claude Code | Cursor | GitHub Copilot | Windsurf |
|------|------------|--------|----------------|----------|
| **定位** | CLI Agent（自主编程Agent） | AI增强型IDE | IDE实时补全副驾驶 | AI增强型IDE |
| **交互范式** | 终端交互，Agentic Loop自主执行 | GUI Diff→Review→Accept | 行内补全+Chat | 流式交互 |
| **本质** | "替你干活" | "帮你写代码" | "帮你写代码" | "帮你写代码" |
| **自主程度** | 可独立完成需求到PR | 每步需人类视觉确认 | 代码补全为主 | 中等自主 |

### 5.2 为什么同样模型下Claude Code表现更好？

- **交互范式**：Claude Code是宏观目标→自动执行，Cursor需要人类频繁确认
- **上下文获取**：Claude Code用grep/ripgrep实地考察，100%确定；Cursor用黑盒RAG可能找错文件
- **System Prompt**：Claude Code专为Claude 4系列深度定制，Cursor需同时兼容多模型
- **错误反馈闭环**：Claude Code全闭环（构建→捕获错误→自动修复→再运行），Cursor需人工复制粘贴
- **计划与执行**：Claude Code有TodoWrite/Task让模型自己维护计划

> 来源：[为什么Cursor跑不过Claude Code](http://m.toutiao.com/group/7610994387438322226/) | [为何Claude Code能替你通宵干活](http://m.toutiao.com/group/7655866619574469172/)

### 5.3 架构范式对比

| 维度 | Claude Code | Cursor/Copilot |
|------|------------|----------------|
| 代码检索 | LLM直接搜索 (grep/find) | RAG向量检索 |
| 控制循环 | 单一主线程 + 一层分支 | 多轮对话但无显式循环 |
| 工具调用 | 原厂定制 (结合PTC) | 通用兼容层 |
| 模型混合 | Haiku占50%+ | 相对单一 |
| 上下文压缩 | 四层压缩体系 | 依赖IDE自身管理 |
| 安全模型 | 八层防线 + AI看AI | IDE权限模型 |
| 生态扩展 | MCP + Skills + Workflows | 插件市场 |

---

## 六、工作流设计：Plan → Execute → Iterate

### 6.1 规划层 (Plan)

1. **TodoWrite/Task工具**：模型主动维护待办列表
2. **交错思考**：执行中动态增删或拒绝任务
3. **claude.md**：项目上下文约束计划方向
4. **System Prompt章节化**：给出流程化算法指令而非零散规则

### 6.2 执行层 (Execute)

1. 六步管线：请求前压缩→流式调用→错误恢复→停止钩子→工具执行→下一轮准备
2. 流式工具并行：只读工具可并行；会改变环境的工具必须独占
3. 错误恢复优先级：先走免费恢复路径

### 6.3 验证层 (Verify)

1. **Stop Hook**：Claude想结束时先验证（如测试必须通过才能停）
2. **Token budget收益递减检测**：连续多次继续但新增token很少时停止
3. **对抗验证（Dynamic Workflows）**：每个产出Agent配一个挑刺Agent

### 6.4 迭代层 (Iterate)

1. `/loop`命令：周期性triage、调研、验证
2. `/goal`命令：给Workflow设硬完成条件
3. **Loop until done**：循环spawn Agent直到无新发现
4. **可恢复性**：中断后resume会话从断点继续

### 6.5 Loop Engineering范式

传统：写prompt→模型输出→人工检查→再写prompt→...
Loop：定义循环条件→模型执行→自动验证→循环继续或停止

这与Addy Osmani的Loop Engineering框架一致：把控制流交给代码，把意图推理留给AI。

> 来源：[Claude Code作者Boris谈Loop](http://m.toutiao.com/group/7654529196337496619/) | [Loop Engineering实战指南](https://www.51cto.com/article/847167.html)

---

## 七、关键技术洞察总结

1. **大道至简**：给模型搭稳固的工具箱后把舞台交给模型
2. **LLM搜索优于RAG**：让模型像人一样搜索代码库，行为可学习、可调试
3. **单一主循环+一层子代理分支**：避免深层嵌套的调试噩梦
4. **50%+ Haiku调用**：做好任务分级，辅助工作全交给小模型
5. **四层上下文压缩**：成本递增但信息保留递增
6. **PTC减少"乒乓球效应"**：确定性强的工作用代码批量执行
7. **Skills渐进式披露**：不一次性加载所有知识，按需匹配
8. **Dynamic Workflows = Agent自写编排器**：2026年最重要的架构演进
9. **八层安全模型**：从构建门到AI看AI，纵深防御
10. **Loop Engineering范式**：从写prompt到写loop的工程范式转变

---

## 八、参考资源

- [Anthropic官方Claude Code页面](https://www.anthropic.com/claude-code)
- [Claude Code官方文档](https://code.claude.com/docs/en/overview)
- [Anthropic "A harness for every task" 博客](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
- [解读Claude Code核心架构](http://m.toutiao.com/group/7635163948036063784/)
- [Claude Code好用的秘密藏在它的Agent设计里](http://m.toutiao.com/group/7542490225214407178/)
- [深度拆解Claude Agent架构：MCP+PTC+Skills+Subagents](http://m.toutiao.com/group/7602879935119655451/)
- [Dynamic Workflows详解](https://cloud.tencent.com/developer/article/2694452)
- [什么是PTC](https://juejin.cn/post/7653050742082338862)
- [Claude Code Sub-agent模式详解](https://juejin.cn/post/7539839677826809883)
- [为什么Cursor跑不过Claude Code](http://m.toutiao.com/group/7610994387438322226/)
- [Loop Engineering实战指南](https://www.51cto.com/article/847167.html)

---

*本报告基于2026年6月最新研究与公开资料编写，持续更新中...*
