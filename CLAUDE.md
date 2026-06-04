# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

- **cc-grow** — 项目名称，当前为绿色字段（新建空仓库）
- **默认分支**: `main`
- **平台**: Windows (PowerShell 7+)

## Git 工作流

```bash
# 首次提交
git add .
git commit -m "chore: init"

# 日常
git add <files>
git commit -m "type(scope): message"
```

## 项目约定

- 代码注释必须完整，遵循 `CLAUDE.md` 中的注释保护规则
- 关键决策记录在 `docs/` 目录下
- 文档维护：`STATE.md`（当前进展）、`WARN.md`（重要提醒）、`DONE.md`（已完成）、`HISTORY.md`（归档）

## 关联指令

- `/init` — 重新初始化/更新 CLAUDE.md
- `/cc-kit:grow-dream` — 对话回顾与经验沉淀
