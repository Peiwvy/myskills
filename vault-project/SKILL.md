---
name: vault-project
description: vault 项目管理：加载项目 context、脚手架新项目、参与方配置、路由规则。当用户说「进入 XX 项目」「切换到 XX」「打开 XX」「新建项目」时触发。
---

# vault-project

本 vault 的 演化协议 落地。协议见 [[Z98-Design/1 设计原理层/演化协议|演化协议]]。

---

## 上游依赖

- [[Z98-Design/1 设计原理层/演化协议|演化协议]] — 五阶段闭环协议
- [[结构设计]] — vault 全景（目录树 + 流转主链）
- [[目录规范]] — CLAUDE.md 写作模板（新建项目时用）

> 完整依赖图见 [[依赖地图.html]]。

---

## 一、参与方配置

本 vault 各范围的工作流模式：

| 层 | 范围 | 模式 | openspec 位置 |
|----|------|------|-------------|
| vault 设计层 | Z98-Design | `full` | `Z98-Design/openspec/` |
| 项目层 | 01-通用交易系统 | `full` | `01-Work&Career/01-通用交易系统/openspec/` |
| 项目层 | 03-SLAM项目 | `full` | `01-Work&Career/03-SLAM项目/openspec/` |
| 项目层 | 02-Manifold | `lite` | `01-Work&Career/02-Manifold/openspec/INBOX.md`（仅） |
| 项目层 | 04-AI-Agent编排 | `lite` | `01-Work&Career/04-AI-Agent编排/openspec/INBOX.md`（仅） |
| 知识库层 | KF / KG 域 | — | 无独立 openspec；结构变更走 Z98（跨项目），内容增补走 vault-distill |

---

## 二、两级 openspec 路由

| 级 | 位置 | 职责 |
|----|------|------|
| **vault 级** | `Z98-Design/openspec/` | vault 整体架构 + 跨多个区/项目的变更 |
| **项目级** | `<project>/openspec/` | 单一项目内部变更 |

**判据**：
- 变更影响**多个项目** → vault 级（Z98）
- 变更影响**单个项目** → 项目级
- 还不知归哪的 → Z98 INBOX

`01-Work&Career/` 自身是项目集容器，不参与流程。

---

## 三、项目路由表

「进入 XX 项目」时查此表（SSOT 在 `01-Work&Career/CLAUDE.md`）：

| 触发词 | 目标 | 模式 |
|--------|------|------|
| 交易系统 / 交易 / 通用交易 | [[01-Work&Career/01-通用交易系统/CLAUDE]] | `full` |
| Manifold / 留形 | [[01-Work&Career/02-Manifold/CLAUDE]] | `lite` |
| SLAM / lvi-slam | [[01-Work&Career/03-SLAM项目/CLAUDE]] | `full` |
| Agent编排 / agent team | [[01-Work&Career/04-AI-Agent编排/CLAUDE]] | `lite` |
| Z98 / vault设计 / 设计层 | [[Z98-Design/CLAUDE]] | `full` |

---

## 四、/vault-project load —— 加载项目

触发：`/vault-project load <name>` 或自然语言「进入 XX 项目」「切换到 XX」。

### 信号双层

| 层 | 信号源 | 谁维护 |
|----|--------|--------|
| 静态 | CLAUDE.md：项目定位 / 物理位置 / 关键文件锚点 | 人（低频） |
| 动态 | `openspec/`（加载 change + 最近 archive + INBOX） | AI 加载时扫描，无人工维护 |

### 执行

1. 读 `01-Work&Career/CLAUDE.md` 路由表，按参数匹配触发词（如 "SLAM" → `03-SLAM项目/`）
2. 读 `<project>/CLAUDE.md`：frontmatter 取 `workflow` 模式 + 项目定位 + 物理位置 + 关键文件
3. **按模式分支**：
   - `full`：列出 `openspec/changes/` 中所有 change（按 status 分组：draft / plan / exec）+ `openspec/archive/` 最近 5 条 + `INBOX.md` 头部（最近 5 条未处理条目）
   - `lite`：仅读 `openspec/INBOX.md`
   - `none`：跳过 openspec/，按项目 CLAUDE.md 自定指引
4. 输出「当前快照」：

```
📁 <项目名> [<模式>]

加载 change：
  · 2026-06-06-macro-clarity [draft] — 宏观分析结构梳理
  · 2026-06-05-xxx [exec] — ...

最近归档：
  · 2026-06-03-establish-openspec-workflow
  · ...

INBOX 待处理：3 条（最近：2026-06-06 ...）
```

5. **等待用户下一步指令**——「继续 <change-slug>」加载具体 change，「新建 change」起新提案，或直接描述其他操作。

### 自然语言触发

「进入 SLAM」「切换到交易系统」「打开 Manifold」→ AI 按上述流程执行，不强制要求 slash command。

### 跨 cwd 工作流

一个项目通常有多个物理位置（知识库在 vault 内、代码仓在 `~/code/` 下、数据在数据盘）。无论从哪个 cwd 开始对话：

1. 先 `/vault-project load <项目>`，加载项目 CLAUDE.md（含「物理位置」段）
2. AI 据此知道：plan 应落哪、代码改哪、change 起在哪个 openspec/
3. 跑 `planning-with-files` 等会产物的 skill 时，AI 用绝对路径覆写 skill 的默认 cwd 行为

---

## 五、继续 / 新建 change

项目加载后，用户通过自然语言操作 change：

### 继续已有 change

「继续 macro-clarity」「继续 2026-06-06-macro-clarity」→ AI：

1. 在 `<project>/openspec/changes/<slug>/` 定位 change 文件夹
2. 读 `proposal.md` → 了解要做什么
3. 读 `tasks.md` → 找到第一个未勾选的任务
4. 开始执行，完成后勾选

### 新建 change

「新建 change」「起个提案」→ AI：

1. 在 `<project>/openspec/changes/` 下创建 `YYYY-MM-DD-<slug>/`
2. 创建 `proposal.md`（含 frontmatter `id` + `status: draft`）
3. 用户描述要做什么 → AI 写入 proposal 正文
4. 需要时创建 `tasks.md`

### 跨项目 change

如果变更影响多个项目 → change 起在 `Z98-Design/openspec/changes/`（vault 级）。

---

## 六、/vault-project new —— 脚手架新项目

触发：`/vault-project new <slug> <mode>`

例：`/vault-project new 05-货币研究 lite`

1. 在 `01-Work&Career/<slug>/` 下创建 CLAUDE.md，frontmatter 含 `workflow: <mode>`，正文骨架：

```markdown
---
workflow: <mode>
---

# <项目名>

## 项目定位

...

## 物理位置

- vault 内：`01-Work&Career/<slug>/`
- 代码仓：`~/code/<repo>/`（如有）
```

2. 按模式创建 `openspec/`：
   - `full` → INBOX.md + changes/ + archive/

   **INBOX.md 模板**：
   ```markdown
   ---
   tags: [type/inbox]
   ---
   # <项目名> INBOX
   
   随手写。一行 bullet + 日期即可。
   ```

   - `lite` → 仅 INOX.md
   - `none` → 不创建

3. `01-Work&Career/CLAUDE.md` 路由表追加一行（项目 + 模式 + 触发词）
4. 输出 `✅ 已初始化 <slug> [<mode>]`，提示后续可 `/vault-project load <slug>` 启动

---

## 七、与既有机制的边界

| 机制 | 用途 | 关系 |
|------|------|------|
| `<project>/.plans/` | `planning-with-files` skill 临时工作区 | **正交** — complete 后沉淀进 change |
| `<project>/.claude/` | 项目专属 slash command / agent / skill | **正交** — 工具配置，不进 openspec |
| 项目内 `tasks/` | 季度/月度任务清单 | **lite 模式** — 复杂任务单独起 change |
| `00-DailyNotes/` | 日记 | **正交** — 想法整理时 distill 到 INBOX |

---

## 八、wikilink 引用约定

多个 openspec/ 有同名文件（`INBOX.md`、`proposal.md`），所有指向 openspec/ 内文件的 wikilink **必须用完整 vault-relative 路径 + 别名**：

- ✅ `[[01-Work&Career/03-SLAM项目/openspec/INBOX|SLAM INBOX]]`
- ✅ `[[Z98-Design/openspec/INBOX|Z98 INBOX]]`
- ❌ `[[INBOX]]`（撞名）
- ❌ `[[openspec/INBOX]]`（Obsidian 不一定能解析）

项目内引用自己的 INBOX 也**用全路径**。
