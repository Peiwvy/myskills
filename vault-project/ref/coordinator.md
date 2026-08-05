# coordinator 行为准则

## 关于判断

- **你做判断，子 agent 做执行**。不把需要判断的任务派给子 agent
- 子 agent 返回后，对照检验方法验证
- 不通过 → 告诉子 agent 哪里不对，让它重来
- 通过 → 更新 inprogress，继续

## 关于内容文件（架构）

- 内容文件是项目实际知识的载体——架构说明、设计文档、领域知识
- 每次 complete 时检查是否需要更新内容文件
- 新增的结构、发现的模式、踩过的坑 → 写进内容文件（不是 CLAUDE.md）
- CLAUDE.md 只管路由：内容结构变了就更新路由指针
- 内容文件长了就拆子目录，保持每个文件可一次读完

## 关于上下文

- 进入项目只读 CLAUDE.md + 检查 inprogress-*.md——最小启动上下文
- 遇到不熟悉的 → 按 CLAUDE.md 路由去读内容文件
- 不把整个 vault 塞进上下文窗口

## 关于 INBOX

- 随想区：人随便写。AI 不主动整理，除非人说「处理 INBOX」
- 梳理区：AI 整理后的可执行条目，编号 A-Z
- 梳理区条目完成 → 删除（不留"已处理"痕迹）
- 跨 scope 的想法 → 升格到父级 \_cockpit

## 关于 inprogress 生命周期

```
随想区 → process-inbox → 梳理区 → start-task → inprogress → execute → complete
                                                    │              │
                                                    │        问：写入内容文件？
                                                    │         ├─ 是 → 更新内容文件
                                                    │         └─ 否 → 跳过
                                                    │              │
                                                    │        删 INBOX 条目
                                                    │        删 inprogress 文件
                                                    │
                                              跨级修改 → 升格父级
```
