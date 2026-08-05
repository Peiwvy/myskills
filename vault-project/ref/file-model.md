# 文件模型

每个 scope（项目 / 子项目 / vault 设计层）最多三类文件：

| # | 位置 | 本质 | 生命周期 |
|---|------|------|---------|
| 1 | `CLAUDE.md` | **路由**：入口 + 索引。告诉 AI/人 去哪找什么 | 持续演进 |
| 2 | `_cockpit/` | **驾驶舱**：项目管理文件（INBOX + inprogress） | INBOX 持续，inprogress 临时 |
| 3 | `<内容文件>/` | **内容**：项目的实际文件（架构说明、领域知识、参考等） | 持续生长 |

---

## 目录结构

```
<scope>/
├── CLAUDE.md                  ← 路由（必须）
├── _cockpit/                  ← 驾驶舱（管理文件）
│   ├── INBOX.md               ← 随想区 + 梳理区
│   └── inprogress-A.md        ← [临时] 对应梳理区 #A，无事则删
├── <内容文件>.md               ← 实际内容
└── <内容子目录>/
```

---

## 递归（嵌套 scope）

任何子目录如果自己是一个独立 scope，就可以有自己的 `_cockpit/`：

```
大型项目/
├── CLAUDE.md
├── _cockpit/                    ← 项目级：跨子项目协调
│   └── INBOX.md
├── 子项目A/
│   ├── _cockpit/                ← 子项目级（需要就有，不需要就无）
│   │   ├── INBOX.md
│   │   └── inprogress-A.md
│   └── <内容>/
└── 子项目B/
    ├── INBOX.md                 ← 只有 INBOX，无 _cockpit（无事则无）
    └── <内容>/
```

**生长规则**：
- scope 独立（有自己的 INBOX + 自己的工作流）→ 可以建 `_cockpit/`
- 该 scope 无活跃工作 → `_cockpit/` 删掉（INBOX 可保留）
- 编号本级闭环：每个 `_cockpit/INBOX.md` 的梳理区编号在自己 scope 内唯一

---

## 各文件详解

### CLAUDE.md（路由）

**本质**：入口 + 索引。不是架构——架构在内容文件里。CLAUDE.md 负责告诉读者「去哪找什么」。

**写什么**：项目定位、物理位置、子目录索引、路由判据、关键规则、与外部的关系（如代码仓库）。

**增长规则**：从 INBOX 吸收想法，渐进更新。复杂项目自然长出 `ref/` 或子目录。

### _cockpit/INBOX.md（随想池）

**写什么**：两个区。

```markdown
# INBOX

## 随想区
（人随便写，格式不限）

- 感觉某个地方漏了什么
- 要不要试试 xxx

## 梳理区
（AI 从随想区整理，编了号才能进 inprogress）

- A. 补充验证标准
- B. 重构目录结构
```

**规则**：
- 随想区：人随便扔。AI 不管格式。
- 梳理区：AI 整理 + 编号 + 归类。只有梳理区条目才能建 `inprogress-*.md`。
- 「处理 INBOX」：AI 读随想区 → 归类编号 → 移到梳理区。
- 条目完成后从梳理区删除。

### _cockpit/inprogress-{id}.md（进行中）

**本质**：临时工作区。存在 = 有事在做。无事则删。

**写什么**：

```markdown
# inprogress-A

- 来源：INBOX 梳理区 #A
- 目标：一句话说清要做什么
- 计划：怎么做
- 问题：遇到的阻塞
- 决策：为什么选 A 不选 B
- 检验：怎么验证做对了
```

**规则**：
- 文件名对应 INBOX 梳理区编号（A、B、C...）
- 可同时存在多个（A 做结构、B 做内容，互不阻塞）
- 每次 dump 更新此文件
- 完成 → 问「是否写入内容文件」→ 是则更新 → 删 INBOX 条目 + 删此文件
- 新对话：检查 `_cockpit/inprogress-*.md` 即可知道进行中的工作

---

## 跨级修改

做某个 inprogress 时发现影响范围超出当前 scope → **升格到最近公共父级的 `_cockpit/`**。

```
L2/_cockpit/inprogress-A.md 执行中
    │
    ▼ 发现需要改 L1 的东西
L2 管不了 L1 → 升格到项目级 _cockpit/INBOX 梳理区
    ├─ C. L1 需调整以支持 L2 #A [来源: L2 #A]
    │
    ▼ 评估
    ├─ 阻塞 L2 #A → 项目级建 inprogress-C.md，先做
    └─ 不阻塞   → L2 #A 继续，C 排队
```

**最近公共父级**：子模块之间互改 → 升到层 `_cockpit`；层之间互改 → 升到项目级 `_cockpit`。

---

## 五个流转行为

| # | 行为 | 触发 | 做什么 |
|---|------|------|--------|
| **add-inbox** | 人随便写 | 往随想区扔 | 追加 `_cockpit/INBOX.md` 随想区 |
| **process-inbox** | 「处理 INBOX」 | AI 读随想区 → 归类编号 → 移到梳理区 | 更新 `_cockpit/INBOX.md` |
| **start-task** | 「开始做 #A」 | 读架构 → 结构清晰则建 inprogress，不清晰则暂停等人 | 创建 `_cockpit/inprogress-A.md` |
| **dump** | 「dump 一下」 | 更新 inprogress 的进展/问题/决策 | 更新 `_cockpit/inprogress-*.md` |
| **complete** | 做完 | 问是否写入内容文件 → 是则更新 → 删 INBOX 条目 + 删 inprogress | 更新内容文件 + 清理 cockpit |
