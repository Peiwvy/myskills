---
name: vault-distill
description: 将**单个**素材文件（来自 K10-Source/）萃取路由到**已有结构**（KF32-Meso / KF33-Micro / KG30-Topics）。当用户说「萃取」、「处理这篇」、「读这篇笔记」、「整理这个文件」、「distill」、「这条素材怎么放」、「拆原子」、「写进 vault」时触发。**边界**：单文件 → 已有结构；遇结构问题 → escalate 到 vault-system-design。**宏观素材不在此 skill 处理**——有对应 Pivot 则合并更新，无则跳过。
date modified: 2026-06-22
---

## 上游依赖

- [[结构设计#流转主链]] — 路由判据（K10→Inbox→KF/KG 走向）
- [[schema_common]] — 三段骨架
- [[schema_pivot]] — Pivot 页面 schema
- [[schema_structure]] — Structure 页面 schema
- [[schema_archives]] — Archives 页面 schema
- [[schema_entity_meso]] — Meso Entity 写入格式
- [[schema_entity_micro]] — Micro Entity 写入格式
- [[schema_entity_person]] — Person Entity 写入格式
- [[schema_entity_concept]] — Concept Entity 写入格式
- [[schema_framework]] — Framework 写入格式
- [[schema_source]] — 输入格式定义

> 完整依赖图见 [[依赖地图.html]]。

---

## 知识萃取（Distill）

**萃取 = 操作要素的一种形态**：在已有结构下，把单篇素材转化为知识要素。

**核心原则：拆原子，归属主体。** 不按来源组织提取结果（那是归档），而是把每条有价值的信息拆到最小单元，归属到它描述的主体页面。

> [!important] 本 skill 范围
> distill 处理**中观、微观、G 域**。宏观素材的处理见 [[KF31-Macro/CLAUDE.md#EM 工作流：积累 → 沉淀]]——日常只在有对应 Pivot 时合并更新，否则跳过等每周沉淀。

### 触发方式

```
/vault-distill <文件路径>
```

---

## 萃取流程

### Step 0：读取原文 + 判断形态

**必须用 `Read` 工具读取原文**（Bash 命令会截断内容）。超过 2000 行时分段读取。确认全文已读完后再继续。

判断材料能否整体处理：

| 形态 | 判断 | 处理 |
|---|---|---|
| 单主题、单类型 | 整体作为一个单元 | 直接进 Step 2 |
| 多主题 / 热点+结构性混合 | 需拆分 | 先拆分，每个单元分别进 Step 2 |
| 多条汇总信息流 | 逐条拆散 | 每条独立进 Step 2 |

### Step 1：Escalation Gate

distill 处理「**单文件 → 已有结构**」。发现以下任一信号，停下当前 distill，告知用户并切换到 `vault-system-design`：

| 信号 | 切换对象 |
|---|---|
| 适用的 schema 不存在 / 现有 schema 与素材冲突 | `specify.md` |
| 同类待路由文件 ≥3 且需 MECE 分桶 / 子目录"放不下" | `relate.md` §拆分 |
| 大规模断链 / 孤立页堆积 / 反复"找不到" | `relate.md` 或回 `vault-system-design` 诊断 |

**建页边界**：单个新 entity 页面（适用现有 schema）属 distill 范围，可直接建。需共建多个同层级新 entity 或当前 schema 不适用 → 切 vault-system-design。

### Step 2：内容路由

```
素材进入
├── 非 F 域？                         → KG30-Topics/
└── F 域投资内容：
      ├── 宏观？（地缘/政策/货币/利率/全球传导）
      │     → 看因果图，有对应 Pivot？→ 更新 Pivot 时间线/判断
      │     → 没有对应 Pivot？→ 跳过，告知用户"无对应 Pivot，已跳过"
      └── 中观微观？（行业/赛道/公司/产品）
            ├── 行业/赛道               → KF32 Meso Entity
            └── 公司/产品               → KF33 Micro Entity
```

#### 宏观素材处理

**不建新页，不做即时 distill。** 流程：

1. 读素材 → 看 Excalidraw 因果图
2. 图上有对应 Pivot 节点？→ 更新该 Pivot 的时间线/当前判断
3. 图上没有？→ 不做任何写入，告知用户"素材 X 当前无对应 Pivot，已跳过"
4. Pivot 的新建/合并/拆分 → 留给每周沉淀（vault-converge）

#### 中观/微观 distill

**中观微观：只有 Entity。** 行业、公司、产品不涉及宏观因果网络。拆原子 → 路由 entity 截面/时间线/机理。

**Entity 内抽象层次**（替换测试）：把这段话从 entity A 搬到同类 entity B 页是否仍成立？成立→属更高抽象层（应提升到行业/市场 entity）。

### Step 3：写入目标页面

**Entity 写入规则（meso/micro）**：
- [[schema_entity_meso]] · [[schema_entity_micro]]
- Entity 只做：**截面** + **机理** + **时间线** + **连接层**
- 不做判断、不推演——方向判断归 Pivot

**Structure 更新规则（宏观，当有对应 Pivot 时）**：
- [[schema_structure]] — **截面 + 机理 + 状态指标**
- 仅当素材含新的状态指标数据时更新

**Pivot 更新规则（宏观，当有对应 Pivot 时）**：
- [[schema_pivot]] — 追加时间线条目 + 必要时更新当前判断
- 不建新 Pivot，不修改 upstream/downstream

**其他类型**：
- [[schema_entity_person]] · [[schema_entity_concept]] · [[schema_framework]]

**写入前必做**：

1. **查重**：grep 目标页面搜索日期 + 关键人名/机构名
2. **截面 vs 时间线**：新证据改变结构性理解→更新截面；只是新数据点→时间线条目
3. **交叉验证**：新源验证已有截面→注明；补充空白→更新；事实矛盾→标 ⚡ 分歧保留
4. **主线演化 Log**：判断翻转/本质重写→追加 Log；仅细节修正→仅覆盖主线

**硬规则**（SSOT 在 [[schema_common]]，违反即错误）：

- **保留旧内容**：frontmatter 属性全部保留
- **可验证性**：有源、缺源即空、禁模糊主语
- **实体纯粹性**：水平（跨实体驱动用双链）+ 垂直（替换测试）
- **解读列依据**：每条主逻辑必列依据
- **结构性推断 🔗 优先识别**

#### Meso→Micro 回链（强制）

distill 写入 meso entity 时，若 Q3 WHICH 表中列出了公司/标的，必须完成三步联动：

**① Q3 公司名必须用 `[[wikilink]]`**：不允许纯文本或加粗公司名——双链是 KF32↔KF33 双向可达的唯一保证。

**② 对应 KF33 页面必须更新 `关联主线`**：
- **先检查 KF33 页面是否已存在**（用 Read 工具读取目标路径，不要假设状态）
- 读取 Q3 中每个 `[[wikilink]]` 指向的 KF33 页面
- 在 frontmatter `关联主线` 中追加当前 meso entity 的 `[[wikilink]]`
- 若 `关联主线` 字段不存在 → 新建

**⚡ 硬规则：禁止覆盖已有 KF33 页面**。若页面已存在 → 仅增量操作：
  - 追加 `关联主线`（不删不改已有内容）
  - 从 meso Q3 表更新 `## 矛盾` 区（增量补充，不删除已有的赌什么/为什么/认错/盯什么）
  - 不改动 `## 故事 & 模式`、`## 体检`、`## 时间线` 及其他已有内容

**若公司尚无 KF33 页面** → 创建最小骨架页：
  - frontmatter: `tags: [type/entity/micro, type/<asset|earnings|growth>]` + `关联主线` + `状态` + `备注`
  - 四段：`## 故事 & 模式` + `## 体检` + `## 矛盾` + `## 时间线`
  - 其中 `## 故事 & 模式` 写入 meso Q3 表中对该公司的描述（如有一句概括）
  - `## 体检`/`## 矛盾`/`## 时间线` 均标 `[待补]`，不编造内容

**③ KF33 页面的 `## 矛盾` 区**：从 meso Q3 表的「核心逻辑」→**赌什么**、「命门」→**认错**。不做额外推演。格式见 [[schema_entity_micro]] 四段骨架。

**检查清单**（distill 完成前逐项确认）：
- [ ] Q3 表中所有公司名都是 `[[wikilink]]`
- [ ] 每个有 KF33 页面的公司 `关联主线` 已包含本 meso entity
- [ ] 无 KF33 页面的公司已创建最小骨架页（或已告知用户）

### Step 4：二次校验 + 完成

**A. Traceability**：硬规则全过。**B. 内容完整性**：每个信息点都有去处。**C. 格式**：时间线条目符合 [[schema_common#时间线条目格式]]。

**完成处理**：
- 原文 frontmatter 加入 `state: source/distilled`（**强制**——标注后用户可 grep 区分已处理/未处理素材）
- 告知用户：路由了哪些内容单元 → 去了哪里（含新建/更新的 entity 清单 + KF33 回链更新清单）

---

## 边界情况

- 宏观素材无对应 Pivot → 跳过，不等同于"有结构问题"，不切 vault-system-design
- 分类不明 → 停，问用户

## 核心设计原则

**拆原子，归属主体**。**结构性推断优先识别**。**竞争而非替换、可追溯性高于完整性**。
