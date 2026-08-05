---
name: vault-project
description: vault 项目管理 coordinator。加载 context、处理 INBOX、跟踪进展、制定方案协调子 agent 执行。触发词：「进入 XX 项目」「切换 XX」「dump一下」「处理 INBOX」「做个方案」「开始执行」「新建项目」。
---

# vault-project

vault 项目管理 coordinator。两层适用范围：

| 层级 | 位置 | 触发词 |
|------|------|--------|
| vault 设计层 | `Z98-Design/` | Z98 / vault设计 / 设计层 |
| 项目层 | `01-Work&Career/<project>/` | 见 `01-Work&Career/CLAUDE.md` 路由表 |

---

## 操作速查

| 操作 | 触发 | 做什么 | 详见 |
|------|------|--------|------|
| **load** | 「进入 XX」 | 读路由 → 检查 inprogress-* → 快照 | [[ref/load]] |
| **process-inbox** | 「处理 INBOX」 | 随想区 → 归类编号 → 梳理区 | [[ref/design]] |
| **start-task** | 「开始做 #A」 | 检查架构 → 建 inprogress | [[ref/file-model]] |
| **dump** | 「dump一下」 | 更新 inprogress 进展/问题/决策 | [[ref/dump]] |
| **complete** | 做完 / 「完成了」 | 问写入内容文件 → 清理 cockpit | [[ref/plan-execute]] |
| **new** | 「新建项目」 | 脚手架（待写） | — |

---

## 文件模型

见 [[ref/file-model]]。

`_cockpit/` 之下只有两个文件：
- `INBOX.md`（随想区 + 梳理区）
- `inprogress-{id}.md`（临时，完成即删）

CLAUDE.md 是路由，不是架构。内容文件才是架构。

---

## 加载规则

load 时只读本文件（SKILL.md）+ 目标 CLAUDE.md + 检查 `inprogress-*`。遇到以下情况时按需读 ref/：

- 用户说「处理 INBOX」→ 读 `ref/design.md`
- 用户说「dump」→ 读 `ref/dump.md`
- 用户说「做个方案」「开始执行」→ 读 `ref/plan-execute.md`
- 遇到子 agent 协调问题 → 读 `ref/coordinator.md`

INBOX 不主动加载。内容文件按 CLAUDE.md 路由去读。
