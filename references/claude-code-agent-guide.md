# Claude Code Agent 功能完全指南：从单兵到军团

> 这是 Claude Code 中文手册系列的第三篇。第一篇讲 CLI 启动参数，第二篇讲交互式 Slash 命令，这篇专门深挖 Agent 体系。

原文出处: Claude Code 中文手册系列

---

Claude Code 的 Agent 功能让你可以创建专门的 AI 助手来处理特定任务。本指南从零开始，深入讲解三种 Agent 模式、实战案例、以及成本控制策略，**每个案例均可直接复制使用**。

---

## 第零章：搞懂 Agent 到底是什么

### 一句话定义

> Agent = 一个有独立上下文窗口、独立系统提示词、独立工具权限的 Claude 实例。

你平时用 Claude Code，就是一个「主 Agent」在干活。Agent 功能让你可以「分身」出更多 Claude 实例，每个分身专注做一件事，互不干扰。

### 类比理解

想象你是一个创业者，原来所有事情你一个人干——写代码、写文档、做测试、审查安全。现在你可以「雇人」了：

```
你（主 Agent）
├── 雇了一个测试工程师（子 Agent）→ 专门跑测试
├── 雇了一个安全审计师（子 Agent）→ 专门查漏洞
└── 雇了一个技术作家（子 Agent）→ 专门写文档
```

每个「员工」有自己的工位（上下文窗口）、自己的工作职责（系统提示词）、自己能用的工具（权限）。他们干完活后，把结果汇报给你。

### 为什么需要 Agent？

| 问题 | 现象 | Agent 解法 |
|------|------|------------|
| 上下文污染 | 搜索了 20 个文件做调研，上下文满了一半 | 子 Agent 独立搜索，只返回 200 字摘要 |
| 串行瓶颈 | 一个实例串行做前端→后端→测试 | 三个 Agent 并行，时间缩短到 1/3 |
| 角色切换损耗 | 一会儿架构师、一会儿码农、一会儿 QA | 每个 Agent 只扮演一个角色，更深入 |

---

## 第一章：三种 Agent 模式 — 用哪个？

Claude Code 提供三种并行工作的方式，从轻到重：

```
轻量                                │                  │                              重量
子 Agent（Subagent）  →  后台任务（Background）  →  Agent Team（团队）
│                      │                          │
在你的会话内部运行      后台独立运行                 完全独立的 Claude 实例
结果返回给你           你可以继续对话               彼此可以互相通信
不能互相通信           不能互相通信                 共享任务清单、可以互相挑战
Token 消耗低           Token 消耗中                 Token 消耗高（线性增长）
```

### 选择决策树

```
你的任务需要多个 Agent 吗？
  ├── 不需要 → 直接用单会话就行
  └── 需要 →
        多个 Agent 之间需要互相通信吗？
          ├── 不需要 → 子 Agent 或后台任务
          │     任务是否耗时、你想继续对话？
          │       ├── 是 → 后台任务
          │       └── 否 → 子 Agent
          └── 需要 → Agent Team
```

---

## 第二章：子 Agent（Subagent）：你的专项外包

### 是什么

子 Agent 在你的主会话内部生成一个独立的 Claude 实例。它有自己的上下文窗口，干完活后把结果返回给你的主会话。就像你发了一个工单给外包团队，他们交付结果后工单就关闭了。

### 两种使用方式

**方式一：临时口头调用（Ad-hoc）**

直接用自然语言告诉 Claude "派一个子 Agent 去做什么"。

```
> 用一个子 Agent 去搜索整个项目里所有的 TODO 注释，
> 按优先级分类后给我一份摘要

# Claude 会启动一个子 Agent，它独立搜索了可能几十个文件
# 搜索过程中的所有文件内容都在子 Agent 自己的上下文里
# 最后只给你返回一份精简的摘要
# 你的主上下文几乎没有增加
```

**方式二：预定义自定义 Agent（Custom Agent）**

在项目中创建一个 Markdown 文件，定义好 Agent 的名字、角色、工具权限。之后 Claude 会根据任务自动匹配调用，或者你也可以手动指名调用。

```
.claude/agents/xxx.md       ← 项目级（团队共享）
~/.claude/agents/xxx.md     ← 用户级（你个人所有项目通用）
```

### 实战案例 1：代码审查 Agent

**用途**: 审查代码质量、安全性和可维护性。代码变更、PR 审查时自动调用。

**文件**: `.claude/agents/code-reviewer.md`

````markdown
---
name: code-reviewer
description: 审查代码质量、安全性和可维护性。代码变更、PR 审查时自动调用。
tools: Read, Grep, Glob
model: sonnet
---

# code-reviewer

代码审查专家 — 检查代码质量、安全性和最佳实践

## 系统提示词

你是一个有 10 年经验的高级代码审查员。

审查流程：
1. 阅读变更的所有文件
2. 检查以下维度：
   - 安全性：是否有注入、XSS、硬编码密钥等风险
   - 性能：是否有 N+1 查询、不必要的循环、内存泄漏
   - 可维护性：命名是否清晰、函数是否过长、是否有重复代码
   - 边界情况：null/undefined 检查、错误处理是否完整
3. 对每个问题给出：位置、风险等级（高/中/低）、修复建议

输出格式：按风险等级从高到低排列，每条包含文件路径、行号、问题描述和修复代码。

## 触发条件

- "审查代码"
- "code review"
- "PR review"
- "检查代码"
````

**关键配置解释：**

| 配置 | 说明 |
|------|------|
| `tools: Read, Grep, Glob` | 只能读，不能编辑/执行命令。安全 |
| `model: sonnet` | 用 Sonnet 而非 Opus，省钱。代码审查不需要最强推理 |
| `description` 关键词 | "代码变更""PR 审查"→ Claude 自动匹配调用 |

### 使用方式

```
# 方式 A：Claude 自动调用（你提到"审查"相关的需求时）
> 审查一下我这次提交的代码变更

# 方式 B：你手动指名调用
> 用 code-reviewer 审查 src/auth/ 目录下的所有变更
```

---

### 实战案例 2：调试专家 Agent

**用途**: 调试错误、测试失败和异常行为。遇到 bug 时自动调用。

**文件**: `.claude/agents/debugger.md`

````markdown
---
name: debugger
description: 调试错误、测试失败和异常行为。遇到 bug 时自动调用。
tools: Read, Edit, Bash, Grep, Glob
model: sonnet
---

# debugger

调试专家 — 分析错误，追溯根因

## 系统提示词

你是一个专精于根因分析的调试专家。

调试流程：
1. 捕获完整的错误信息和堆栈跟踪
2. 确定复现步骤
3. 定位故障代码位置
4. 形成假设并逐个验证
5. 实施最小修复
6. 验证修复有效

每个 bug 的输出必须包含：
- 根因解释（不只是症状）
- 支撑诊断的证据
- 具体的修复代码
- 防止复发的建议

## 触发条件

- "调试"
- "debug"
- "报错"
- "bug"
- "不工作"
````

**使用场景：**
```
> 这个测试一直挂，帮我 debug：npm test -- --grep "auth middleware"
```

> **💡 Claude 执行过程：**
>
> // Claude 判断这是调试任务 → 自动调用 debugger Agent
> // debugger 在独立上下文中：
> //   1. 执行测试命令看完整错误输出
> //   2. 读取相关源代码文件
> //   3. 分析、定位、修复
> // 把修复结果返回给主会话

---

### 实战案例 3：一人开发流水线

这个案例展示如何把三个子 Agent 串联成自动流水线，模拟一个小团队的开发流程：

```
pm-spec（产品经理）→ architect（架构师）→ implementer（实现者）
```

**Agent 1：产品经理** — 文件 `.claude/agents/pm-spec.md`

**用途**: 将需求描述转化为结构化的产品规格文档。

````markdown
---
name: pm-spec
description: 将需求描述转化为结构化的产品规格文档
tools: Read, Write, Grep, Glob
model: sonnet
---

# pm-spec

产品经理 Agent — 将需求描述转化为结构化的产品规格文档

## 系统提示词

你是产品经理。接收一个功能需求的简短描述，输出：
1. 功能摘要（一段话）
2. 用户故事列表（As a X, I want Y, so that Z）
3. 验收标准（可测试的条件列表）
4. 技术约束和假设
5. 不在本次范围内的内容

输出保存为 `specs/[功能名].md` 文件。

## 触发条件

- "产品规格"
- "需求分析"
- "用户故事"
- "pm spec"
````

**Agent 2：架构师** — 文件 `.claude/agents/architect.md`

**用途**: 审查产品规格并输出技术设计文档。

````markdown
---
name: architect
description: 审查产品规格并输出技术设计文档
tools: Read, Write, Grep, Glob
model: sonnet
---

# architect

系统架构师 Agent — 审查产品规格并输出技术设计文档

## 系统提示词

你是系统架构师。读取产品经理输出的规格文档，输出：
1. 技术方案选型（有哪些选择、为什么选这个）
2. 数据库变更（新增表/字段/索引）
3. API 接口设计（路径、方法、请求体、响应体）
4. 文件变更清单（哪些文件需要新增/修改）
5. 风险评估

输出保存为 `specs/[功能名]-architecture.md` 文件。

## 触发条件

- "架构设计"
- "architecture"
- "技术方案"
- "系统设计"
````

**Agent 3：实现者** — 文件 `.claude/agents/implementer.md`

**用途**: 根据架构设计文档实现代码并编写测试。

````markdown
---
name: implementer
description: 根据架构设计文档实现代码并编写测试
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

# implementer

全栈实现 Agent — 根据架构设计文档实现代码并编写测试

## 系统提示词

你是全栈开发工程师。读取架构设计文档，执行：
1. 按照文件变更清单逐个实现
2. 为每个新功能编写单元测试
3. 运行 lint 和测试确保通过
4. 输出变更摘要

## 触发条件

- "实现功能"
- "implement"
- "功能开发"
- "写代码实现"
````

**串联使用：**

```
> 新功能需求：用户可以导出自己的数据为 CSV 格式。
> 先用 pm-spec 写规格，然后 architect 设计方案，最后 implementer 实现。

# Claude 会按顺序调用三个 Agent：
# pm-spec → 输出 specs/data-export.md
# architect → 读取规格 → 输出 specs/data-export-architecture.md
# implementer → 读取设计 → 写代码、跑测试
```

这个流程的价值在于：每个 Agent 各自用自己的上下文窗口，主会话几乎不消耗上下文。即使是大型功能开发，你的主会话也能保持清爽。

---

### 实战案例 4：只读探索 Agent

你想让 Claude 深入调研一个陌生代码库，但不希望调研过程中的大量文件内容污染你的主上下文。

```
> 启动一个子 Agent，用 Explore 模式调研整个 src/payments/ 目录。
> 搞清楚支付流程的完整链路：从用户点击支付按钮到最终扣款，
> 经过了哪些文件、哪些函数调用、哪些外部服务。
> 画一张调用链路图给我。

# 子 Agent 可能读了 30+ 个文件
# 但最终只返回一张精简的调用链路图和一段摘要
# 你的主上下文只增加了几百个 token
```

### 子 Agent 的关键配置项

```yaml
---
name: agent-name            # 名字：小写字母 + 连字符
description: 什么时候调用    # Claude 根据这个判断是否自动匹配
tools: Read, Grep, Glob     # 工具权限白名单（不写 = 继承全部）
model: sonnet                # 用哪个模型（sonnet/opus/haiku/inherit）
background: true             # 是否后台运行（不阻塞主会话）
isolation: worktree          # 是否在独立 git worktree 中运行
skills:                      # 预加载的 Skills
  - my-skill-name
---
```

### 工具权限对照表

| Agent 类型 | 推荐工具 | 原因 |
|-----------|---------|------|
| 审查类（只看不改） | Read, Grep, Glob | 不能修改代码，安全 |
| 调研类 | Read, Grep, Glob, WebFetch, WebSearch | 可以搜索但不改代码 |
| 开发类（能改代码） | Read, Write, Edit, Bash, Glob, Grep | 完整开发权限 |
| 文档类 | Read, Write, Edit, Glob, Grep | 能写文件但不能执行命令 |

### 模型选择建议

| 任务复杂度 | 推荐模型 | 原因 |
|-----------|---------|------|
| 简单机械任务（格式化、改名） | haiku | 最便宜最快 |
| 常规开发任务 | sonnet | 性价比最高 |
| 复杂架构/设计决策 | opus | 推理能力最强 |
| 跟主会话一样 | inherit | 继承主 Agent 的模型 |

---

## 第三章：后台任务：不阻塞你的对话

### 是什么

你让 Claude 做一件耗时的事（跑测试、编译、分析），但不想干等着。后台任务让它在后台执行，你继续和 Claude 聊别的。

### 使用方式

```bash
# 方式 1：用 & 前缀（类似 Unix shell）
> & npm run test:all

# 方式 2：自然语言
> 在后台跑一下完整的测试套件

# 方式 3：让 Claude 在后台执行一个子 Agent
> 后台启动一个子 Agent，让它分析整个项目的依赖树并生成报告
```

### 管理后台任务

```
/tasks          # 查看所有后台任务
/bashes         # 查看后台运行的 shell 命令
/kill task-123  # 终止某个任务
Ctrl+T          # 快捷键：切换任务列表显示
```

### 实战案例：边聊边干

```
# 1. 启动后台测试
> & npm run test:integration

# 2. 同时继续写代码（不用等测试跑完）
> 帮我实现用户头像上传功能

# 3. 测试跑完后，后台任务返回结果
# Claude 会通知你："后台测试完成，3 个失败"

# 4. 切过去处理失败的测试
> 帮我修复刚才那 3 个失败的测试
```

---

## 第四章：Agent Team：真正的 AI 团队

### 是什么

Agent Team 是 Claude Code 最重型也最强大的多 Agent 模式。它不是「一个 Claude 带几个子进程」，而是多个完全独立的 Claude 实例，各自有自己的终端面板（如果你用 tmux），可以互相通信、互相发现对方的工作成果、互相挑战观点。

有一个 Lead Agent（团队领导）负责协调，其他 Teammate 并行工作。

### 和子 Agent 的核心区别

| 维度 | 子 Agent | Agent Team |
|------|---------|------------|
| 独立性 | 在主会话内部运行 | 每个都是完全独立的 Claude 实例 |
| 通信 | 只能把结果返回给主 Agent | Teammate 之间可以互相发消息 |
| 可见性 | 你看不到子 Agent 的过程 | 用 tmux 可以实时看每个 Teammate 在干什么 |
| 互动 | 不能中途干预子 Agent | 你可以直接跟任何一个 Teammate 对话 |
| 任务管理 | 无 | 共享任务清单，进度实时同步 |
| 成本 | 低（共享主会话） | 高（N 个 Teammate ≈ N 倍 token） |
| 状态 | 正式功能 | 实验性功能（需手动开启） |

### 开启方法

Agent Team 默认关闭，需要手动开启。

```json
// settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

```bash
# 或环境变量
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
claude
```

> **推荐安装 tmux**：有了 tmux，每个 Teammate 会在自己的终端面板里运行，你可以实时观察每个人在干什么。没有 tmux 也能用，但所有输出混在一个终端里，很难看清。

### 实战案例 1：全栈功能并行开发

这是 Agent Team 最经典的用法——前端、后端、测试三个方向同时推进。

```
> 创建一个 Agent Team 实现「用户可以通过 OAuth 登录」功能：
>
> Teammate 1（后端工程师）：
> - 实现 OAuth 2.0 授权码流程
> - 只在 src/server/ 目录下工作
> - 创建 /auth/google、/auth/callback 路由
>
> Teammate 2（前端工程师）：
> - 实现登录按钮和回调页面
> - 只在 src/client/ 目录下工作
> - 处理 token 存储和刷新
>
> Teammate 3（测试工程师）：
> - 为前后端的 OAuth 流程写集成测试
> - 只在 tests/ 目录下工作
> - 覆盖正常流程、token 过期、用户取消授权等场景
>
> 后端先完成 API，通知前端对接；
> 前后端都完成后，测试开始跑。
```

> **关键技巧：用目录隔离防止冲突。** 每个 Teammate 只操作自己的目录，不会出现两个 Agent 同时改同一个文件的情况——这是 Agent Team 最常见的坑。

---

### 实战案例 2：一周社交媒体内容生产

Agent Team 不只能写代码。有人用它做了一个内容生产流水线。

```
> 创建一个 Agent Team 为我的技术博客生产下周的社交媒体内容：
>
> Teammate 1（调研员）：搜索本周 AI 和编程领域的热门话题
> Teammate 2（内容策划）：为每天规划内容（X 帖子 + LinkedIn 长文）
> Teammate 3（撰稿人）：撰写所有内容
> Teammate 4（审校员）：事实核查、语法、品牌一致性审查，低于 7 分打回
>
> 所有最终内容保存到 content/week-[日期]/ 目录。
```

---

### 实战案例 3：数据库迁移安全网

```
> 创建一个 Agent Team 来执行数据库从 PostgreSQL 迁移到新 schema 的工作：
>
> Teammate 1（迁移工程师）：编写迁移脚本，输出回滚脚本
> Teammate 2（数据验证员）：审查迁移脚本，验证数据完整性
> Teammate 3（应用层检查员）：检查所有引用旧 schema 的代码
>
> # 亮点：Teammate 2 发现问题后可直接发消息给 Teammate 1
> # 要求修改，不需要经过你中转 — 这是 Agent Team 的核心优势
```

---

### 实战案例 4：接手陌生代码库的高效方式

```
> 创建一个 Agent Team 来帮我快速理解这个项目：
>
> Teammate 1（架构分析师）：分析整体架构、模块依赖关系
> Teammate 2（数据流追踪者）：追踪用户请求→数据库→响应的完整链路
> Teammate 3（技术债探测器）：搜索 TODO/FIXME/HACK、重复代码、缺失测试
>
> 最终输出一份 ONBOARDING.md，让新人看完就能上手。
```

### Agent Team 的管理操作

```
/agents          # 查看团队状态
# 在 tmux 中切换到对应面板可直接与 Teammate 对话
Shift+Tab        # 切换模式：Normal → Auto-Accept → Plan → Delegate
```

**Delegate 模式（委派模式）**：启用后，Lead Agent 只负责任务分配和协调，自己不写代码。模拟「只管不做」的项目经理角色。

### Agent Team 的已知限制

| 限制 | 说明 |
|------|------|
| 不能嵌套 | Teammate 不能再创建自己的子团队 |
| Lead 不能转让 | 你不能中途把领导权交给某个 Teammate |
| 权限统一 | 所有 Teammate 继承相同的权限配置，不能单独设置 |
| 终端要求 | 分屏需要 tmux 或 iTerm2，不支持 VS Code 终端和 Windows Terminal |
| 恢复不稳定 | 团队会话的恢复功能还不太靠谱 |
| 费用高 | 3 个 Teammate ≈ 3-4 倍的单会话 token 消耗 |

---

## 第五章：省钱心法：不同 Agent 模式的成本控制

Agent 功能是 Claude Code 最烧钱的地方。以下是经过验证的省钱策略：

### 策略 1：模型分层

| 角色 | 推荐模型 | 原因 |
|------|---------|------|
| Lead Agent / 架构 Agent | opus | 需要最强推理 |
| 实现 Agent / 测试 Agent | sonnet | 性价比最高 |
| 格式化 / 改名 / 简单任务 | haiku | 最便宜 |

```bash
# 全局设置子 Agent 默认模型
export CLAUDE_CODE_SUBAGENT_MODEL=haiku
```

### 策略 2：工具最小化

给每个 Agent 只分配它需要的最少工具。工具越少，Agent 的决策空间越小，消耗的思考 token 越少。

```yaml
# 差：不限制工具（继承全部）
---
name: reviewer
---

# 好：只给需要的工具
---
name: reviewer
tools: Read, Grep, Glob
---
```

### 策略 3：控制 Agent Team 规模

- **黄金法则**：3-5 个 Teammate，每个 Teammate 5-6 个任务
- 超过 5 个 Teammate → 协调开销急剧增加，收益递减
- 每个 Teammate 超过 10 个任务 → 上下文膨胀，质量下降

### 策略 4：能用子 Agent 就不用 Agent Team

Agent Team 的额外成本来自：
- 每个 Teammate 的独立上下文窗口
- Teammate 之间的通信消息
- Lead Agent 的协调开销

> 如果任务之间不需要互相通信 → 用子 Agent 就够了

### 实际费用参考

| 场景 | 模式 | 大约花费 |
|------|------|---------|
| 单文件代码审查 | 单会话 | \$0.10-0.30 |
| 多文件调研 + 实现 | 子 Agent | \$0.50-2.00 |
| 三层 Agent 流水线 | 子 Agent 链 | \$1.00-5.00 |
| 3 Teammate 全栈开发 | Agent Team | \$3.00-15.00 |
| 一周社交媒体内容 | 4 Teammate | \$5.00-10.00 |

---

## 第六章：我该怎么开始？

### 第一周：入门

不要直接上 Agent Team。先从最简单的临时子 Agent 开始：

```
> 用一个子 Agent 帮我搜索项目里所有的 deprecated 函数
```

观察效果：主上下文是否更干净了？结果质量如何？

### 第二周：自定义

根据日常工作，创建 2-3 个最常用的自定义 Agent。推荐从这三个开始：

| Agent | 用途 |
|-------|------|
| code-reviewer | 只读审查 |
| test-writer | 写测试 |
| debugger | 调试修 bug |

### 第三周：组合

- 尝试把子 Agent 串联起来（pm-spec → architect → implementer）
- 用后台任务并行处理耗时操作

### 第四周：团队

- 开启 Agent Team 实验功能
- 从一个小任务试起
- 安装 tmux 获得最佳体验

### 持续优化

每月跑一次 `/insights`，根据报告优化 Agent 配置：

```
/insights
# 报告会告诉你：
# "你每次手动指定 code-reviewer 时都用 opus，建议改成 sonnet 可节省 40% 费用"
# "你的 debugger Agent 平均每次读取 45 个文件，建议限制搜索范围"
```

---

*最后更新: 2026-03*

---

*Copied from Claude Code 中文手册系列*
