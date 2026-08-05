---
name: vault-material-process
description: 素材处理编排器——串联「BibiGPTRAW 总结 → Clippings 预分类 → meso distill 咨询」。当用户说「处理素材」、「整理素材」、「素材处理」、「处理 Clippings」、「处理 BibiGPT」等时触发。支持 Phase 内并行 subagent 加速。
---

<essential_principles>
## 编排定位

本 skill 是编排器，不自己做 summarizer / pre-distill / distill。它按固定流程依次调用下游 skill，循环「执行 → 汇总 → 下一个」直到所有素材处理完毕。

**核心约束**：
- 每一步**完整执行**后才进入下一步——不跳步、不并行跨阶段
- **Phase 内可并行**：同一 Phase 内独立任务用多个 subagent 并发处理（如 Phase 1 多篇 BibiGPT 同时总结），缩短总耗时
- 每步结束后输出**进度汇总**再继续
- 遇到下游 skill 范围外的结构问题 → 暂停，告知用户
</essential_principles>

<objective>
一键处理 K10-Source/ 下所有待处理素材：

1. **Phase 0**：资产盘点
2. **Phase 1**：BibiGPTRAW → vault-summarize 总结 → sort/g/（并行 subagent 加速）
3. **Phase 2**：Clippings → vault-pre-distill 预分类 → sort/ 各桶
4. **Phase 3**：汇总 sort/ 待处理清单
5. **Phase 4**：咨询用户是否 distill meso/
</objective>

<quick_start>
触发方式（自然语言）：
- 「处理素材」→ 执行全部 Phase（含 Phase 4 咨询）
- 「处理素材 --dry-run」→ 只扫描统计，不实际处理
- 「处理 BibiGPT」→ 只执行 Phase 0 + 1
- 「处理 Clippings」→ 只执行 Phase 0 + 2 + 3 + 4
</quick_start>

<workflow>
按 Phase 顺序执行。每个 Phase 内部循环处理直到该阶段清空。

## Phase 0：资产盘点（每次必做）

**扫描以下位置，输出结构化盘点**：

```
📦 素材资产盘点

BibiGPTRAW/（待总结）：
  - xxx.md (45KB)
  - yyy.md (12KB)
  （共 N 个，已有总结 M 个）

Clippings/（待预分类）：
  - aaa.md
  - bbb.md
  （共 N 个，无 Clippings 则为 0）

sort/ 桶（待 distill）：
  - macro_active/: N 个
  - macro_past/: N 个
  - meso/: N 个
  - g/: N 个
```

如果所有目录都为空 → 「没有待处理的素材」→ 结束。

## Phase 1：BibiGPTRAW 总结

对 K10-Source/BibiGPTRAW/ 下**每篇无对应 `-总结.md` 的文件**：

### 并行策略

| 文件数 | 策略 |
|--------|------|
| 1-2 篇 | 顺序处理，每篇完成后告知用户 |
| ≥ 3 篇 | **首篇先行**（用户 review 风格确认）→ 剩余篇目 **并行派发 subagent**（每批 ≤ 3 个） |

首篇完成后暂停，告知用户「第 1 篇已完成 [文件名]，请 review 风格。后续将并行处理剩余 N-1 篇。」用户确认后，将剩余文件按每批 ≤ 3 个派发给 summarize subagent，并发写入 sort/g/。

### 处理规则

1. 判断文件大小：< 3 万字 → 直接模式；≥ 3 万字 → 拆分模式
2. 输出到 `K10-Source/sort/g/`（**直接进 g 桶**——视频/播客默认非投资类）
   - 输出命名：`讲者-主题-总结.md`
3. 全部完成后输出 Phase 1 汇总（处理数 + 输出文件清单）

**特殊情况**：若用户明确说某 BibiGPT 内容是投资类 → 用户指示分桶方向（meso/macro_active/macro_past），不走 g。

## Phase 2：Clippings 预分类

**前提：K10-Source/Clippings/ 下有 .md 文件**（否则跳过）。

执行 `vault-pre-distill`——**并行读取**所有文件标题 + 前 20 行，按 4 桶判据分到 `K10-Source/sort/<桶名>/`：

| 桶 | 判据 |
|----|------|
| `macro_active` | 宏观 + 仍在演进、跨实体传导 |
| `macro_past` | 宏观 + 已沉淀为截面/机理/历史 |
| `meso` | 行业/赛道/公司/产品 |
| `g` | 非 F 域（投资无关） |

**并行读取**：一次性并发 Read 所有 Clippings 文件的标题+前 20 行 → 统一分桶 → 批量 mv。重复文件（sort/ 已有同名）→ 删除 Clippings 副本。

输出分桶结果表 + 各桶数量。

## Phase 3：汇总待处理清单

输出 sort/ 各桶汇总 + 后续建议：

```
✅ 素材预处理完成

Phase 1 BibiGPT 总结：处理 N 个，输出到 sort/g/
Phase 2 Clippings 预分类：处理 N 个，分桶结果 [表]

sort/ 各桶待 distill：
  - 🔵 meso/: N 个 → 用 /vault-distill 逐篇处理
  - 🟢 g/: N 个 → 用 /vault-distill 逐篇处理
  - 🔴 macro_active/: N 个 → 手动合并到 Pivot
  - ⚪ macro_past/: N 个 → 手动确认是否仍有价值
```

## Phase 4：咨询 meso distill

汇总输出后，**必须询问用户**：

> meso/ 桶有 N 个待 distill 文件。是否需要我现在逐篇萃取（/vault-distill）？

- 用户同意 → 对 sort/meso/ 下每个文件依次调用 `vault-distill` skill
- 用户拒绝 → 结束，提醒用户可随时自行 distill
- meso/ 为空 → 跳过此 Phase

**meso distill 并行策略**：meso 文件之间独立无依赖 → 每批 ≤ 3 个并发派发 subagent，每个 subagent 调用 vault-distill 处理单篇。
</workflow>

<success_criteria>
素材处理流程完成当：
- [ ] Phase 0 盘点输出完毕
- [ ] Phase 1 所有无总结 BibiGPTRAW 文件已总结并移入 sort/g/
- [ ] Phase 2 Clippings 已全部预分类到 sort/ 各桶
- [ ] Phase 3 汇总已输出，各桶待 distill 清单已呈现
- [ ] Phase 4 meso distill 咨询已发起（用户选择执行或跳过）
- [ ] 任何中断 / 结构问题已告知用户
</success_criteria>
