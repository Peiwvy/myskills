# load（加载项目）

触发：「进入 XX 项目」「切换到 XX」或自然语言提及项目名。

## 步骤

1. 读 `01-Work&Career/CLAUDE.md` 路由表，匹配触发词 → 定位目标目录
2. 读 `<scope>/CLAUDE.md`（路由）
3. 检查 `_cockpit/` 下是否有 `inprogress-*.md`：
   - 有 → 读取，输出当前进行中的工作
   - 无 → 「没有进行中的任务」
4. 输出快照：

```
📁 <名称>

路由：
  · <一句话定位>
  · 关键入口：<子目录/关键文件>

🚁 驾驶舱：
  进行中：inprogress-A（<目标一句话>）
          inprogress-B（<目标一句话>）
  或无进行中任务 → 看 INBOX？
```

## 不主动加载

- **INBOX.md** — 用户说「处理 INBOX」才读
- **内容文件的全文** — 需要时按 CLAUDE.md 路由去读
- **子 scope 的 _cockpit/** — 进入子 scope 时才加载

## 跨 cwd

项目可能有多个物理位置。从 CLAUDE.md「物理位置」段获知：

| 路径类型 | 写法 | 例子 |
|---------|------|------|
| vault 内部 | vault-relative | `01-Work&Career/03-SLAM项目/` |
| vault 外部 | `~/` 起始 | `~/code/lvi-slam/` |
| 本机特定 | 注明 | `192.168.0.119:1080`（本机） |
