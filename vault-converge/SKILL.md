---
name: vault-converge
description: 每周 Pivot 形成（EM 的 M 步）。遍历现有 Pivots + 未处理的宏观素材 → 推荐独立/合并/新建 → 更新因果图。当用户说「沉淀」、「周沉淀」、「本周总结」、「converge」、「形成 Pivot」时触发。
date modified: 2026-06-17
---

## 定位：EM 的 M 步

```
E 步（日常积累）         M 步（本 skill，每周触发）
素材 → Inbox             遍历 Pivots → 读素材 → 推荐 → 更新因果图
```

宏观知识不是来一篇炼一篇。**日常只积累，每周你做一次主动合成**——你来把关什么值得进入因果图。

## 上游依赖

- [[KF31-Macro/CLAUDE.md#EM 工作流：积累 → 沉淀]] — EM 方法论
- [[schema_pivot]] — Pivot 页面 schema
- [[schema_structure]] — Structure 页面 schema
- [[schema_archives]] — Archives 页面 schema

---

## 周沉淀流程

### Step 0：盘点现状

**列出现有 Pivots 全景**：
- 遍历所有 `CTY-XX/Pivots/` 和 `REG-XX/Pivots/`
- 对每个 Pivot，提取：一句话方向判断、当前状态标记、上次更新时间
- 呈现当前因果图概览：哪些 Pivot 密集连接（hub）？哪些边缘？

**列出本周未处理素材**：
- 扫描 `K10-Source/sort/macro_active/` 中本周新增的文件
- 列出文件名 + 一句话主题

### Step 1：逐素材分析

对每篇未处理素材：

1. **主题与哪个 Pivot 相关？** — 一一对照因果图
2. **提出建议**（三选一）：

| 建议 | 判断条件 |
|------|---------|
| **合并到已有 Pivot** | 素材主题是图上某个 Pivot 的新信息/更新——追加时间线、更新当前判断 |
| **新建独立 Pivot** | 素材揭示了一个新的因果节点——有独立方向判断、有上下游、因果链独立 |
| **跳过** | 素材不重要、不含新信息、或暂时不适合形成 Pivot |

3. **如果新建**：提议放在哪个国家/地区下？它和现有 Pivots 的传导关系是什么？

### Step 2：用户决策

呈现**建议清单**，用户逐条决定：
- 同意 → 执行
- 调整 → 按用户意见修改
- 否决 → 跳过

**用户同时审定因果图**：加新节点？加新边？删失效节点？调整传导方向？

### Step 3：执行写入

- **合并**：更新 Pivot 页面的时间线和当前判断
- **新建 Pivot**：按 [[schema_pivot]] 创建页面，写入 frontmatter + 当前判断
- **更新因果图**：在 Excalidraw 中反映本次所有结构变更
- **素材标记已处理**：移入 `macro_past/` 或加 tag `state/source/distilled`

### Step 4：汇报

列出本次沉淀的变更摘要：
- 新建了哪些 Pivots
- 更新了哪些 Pivots
- 图中新增/修改了哪些传导边
- 跳过了什么、为什么

---

## 边界

- 不碰 Archives、不碰 Structure（除非素材含新的状态指标数据）
- 不确定时列出选项，由用户决策，不自行新建 Pivot
- 因果图的改动由用户最终确认
