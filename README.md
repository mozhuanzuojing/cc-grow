# cc-grow

> Claude Code 自定义 Agent 集合 — 13 个开箱即用的专用 AI 助手

## 快速开始

```bash
# 复制到你的项目即可使用
cp -r .claude/agents/ <你的项目>/.claude/agents/
# 重启 Claude Code，Agent 自动生效
```

## 使用方法

### 自动匹配

描述任务时包含自然语言触发词，Claude 自动调度对应 Agent：

```
> 审查一下这次提交的代码变更     → code-reviewer 自动介入
> 这个测试一直挂帮我看看          → debugger 自动介入
> 帮我设计用户模块的 API          → api-designer 自动介入
```

### 手动指名

直接指定 Agent 名称：

```
> 用 code-reviewer 审查 src/auth/ 目录
> 让 security-auditor 扫描整个项目
```

### 流水线串联

三个 Agent 接力完成从需求到代码的全流程：

```
> 新功能：用户可导出数据为 CSV。
> 先用 pm-spec 写规格，再 architect 设计，最后 implementer 实现
```

```
┌──────────┐     specs/foo.md     ┌───────────┐  specs/foo-arch.md  ┌──────────────┐
│  pm-spec │ ──────────────────→ │ architect │ ──────────────────→ │ implementer  │
│ 产品规格  │                     │ 技术设计   │                     │ 代码实现+测试  │
└──────────┘                     └───────────┘                     └──────────────┘
   sonnet                           sonnet                            sonnet
```

## Agent 清单

| Agent | 模型 | 用途 |
|-------|------|------|
| code-reviewer | sonnet | 代码质量/安全审查 |
| debugger | sonnet | 错误诊断与根因分析 |
| test-writer | sonnet | 自动生成测试用例 |
| security-auditor | sonnet | OWASP 安全漏洞审计 |
| doc-writer | haiku | README / API 文档 / 注释 |
| git-expert | haiku | Git 操作与冲突解决 |
| api-designer | sonnet | RESTful API 设计 |
| performance-optimizer | sonnet | 性能瓶颈分析 |
| database-expert | sonnet | Schema / SQL / 迁移 |
| refactor-assistant | sonnet | 代码重构 |
| pm-spec | sonnet | 需求 → 产品规格 |
| architect | sonnet | 规格 → 技术设计 |
| implementer | sonnet | 设计 → 代码实现 |

## 项目环境

- **平台**: Windows
- **Shell**: PowerShell 7+
- **默认分支**: `main`

## 参考文档

- [Claude Code Agent 完全指南](references/claude-code-agent-guide.md)
- [10 Essential Agents (英文原文)](references/claude-code-essential-agents-en.md)
- [10 Essential Agents (中文翻译)](references/claude-code-essential-agents-zh.md)
