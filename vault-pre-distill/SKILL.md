---
name: vault-pre-distill
description: 将 K10-Source/ 下的素材文件**批量预分类**到 sort/ 的 4 个桶（macro_active / macro_past / meso / g），作为 distill 的前置步骤。当用户说「预分类」、「pre-distill」、「分桶」、「sort素材」、「整理素材」时触发。
user_invocable: true
---

## 上游依赖

- [[结构设计#流转主链]] — K10 → Inbox → KF/KG 走向
- [[vault-distill]] — 下游消费者，桶名给 distill 操作者提供粗判提示

## 定位

**预分类 ≠ 萃取**。本 skill 只做粗粒度的文件级分桶，不读内容细节、不拆原子、不写入知识页。

分桶后由 `vault-distill` 消费——桶名是提示，distill 仍按自己的路由树拆分每个内容单元。

---

## 触发方式

```
/vault-pre-distill                          → 处理 K10-Source/Clippings/
/vault-pre-distill K10-Source/Weread/       → 处理指定目录
```

自然语言：「预分类一下 Clippings」「把这些素材分桶」「sort 一下」

---

## 4 桶定义

| 桶 | 判据 | distill 去向（提示） |
|----|------|---------------------|
| `macro_active` | 宏观 + 仍在演进、跨实体传导 | KF31 Narrative |
| `macro_past` | 宏观 + 已沉淀为截面/机理/历史 | KF31 Entity |
| `meso` | 行业/赛道/公司/产品 | KF32 / KF33 Entity |
| `g` | 非 F 域（投资无关） | KG30-Topics |

**判据优先级**：先判断 F/G（是否投资相关）→ 再判断宏观/中观微观 → 宏观内部判断 active/past。

---

## 执行步骤

### Step 1：确定范围

默认处理 `K10-Source/Clippings/`。用户指定路径时处理指定目录。

确认目录下存在 `.md` 文件，为空则退出。

### Step 2：逐文件分类

对每个 `.md` 文件，读标题 + 前 20 行（必要时读全文），按 4 桶判据归类。

**宏观 vs 中观微观**：
- 涉及地缘政治、货币政策、利率汇率、财政政策、全球贸易、商品周期 → 宏观
- 涉及具体行业/赛道/公司/产品 → 中观微观
- 涉及多个国家/区域的传导链 → 宏观

**宏观 active vs past**：
- 事件仍在演进、结论尚未沉淀、涉及多方博弈 → active
- 已结束的事件、稳定机理、历史总结 → past

**F vs G**：
- 与投资/交易/经济/产业直接相关 → F
- AI 工程工具、心理学/行为科学书籍、纯技术教程 → G

### Step 3：处理重复

同名去 ` 1.md` 后缀或 URL 编码重复（`+` 替代空格）的文件，保留规范命名版本，删除重复。

### Step 4：执行移动

将文件 `mv` 到 `K10-Source/sort/<桶名>/`。

### Step 5：汇总

输出分类结果表 + 各桶数量。

---

## 边界

- **不确定的放一边**：标题模糊、难以判断 F/G 或宏观/中观的文件，单独列出请用户决策
- **macro_past 为空正常**：大多数素材是当前内容，past 桶常为空
- **本 skill 不改文件内容**：只 mv，不改写、不加 tag、不 distill
