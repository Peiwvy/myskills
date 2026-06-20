---
name: opencli-bridge
description: 通用 opencli 站点适配器配置与使用指南。当用户需要配置某个 opencli 站点适配器的凭证（尤其是没有内置 login 命令的站点）、排查 opencli 认证问题、或需要了解某个站点适配器是否支持浏览器登录时使用。触发词：opencli 配置、opencli 登录、opencli 认证、opencli token、opencli credentials、某个 opencli 站点怎么用、怎么配。
---

# opencli-bridge

opencli 已经封装了 150+ 站点适配器（weread / xiaoyuzhou / xiaohongshu / douyin / zhihu / bilibili / twitter / youtube ...），大部分走浏览器自动化，少数需要手动配凭证。这个 skill 告诉你两类站点分别怎么处理。

## 快速判断站点类型

```bash
opencli <site> --help 2>&1 | head -5
```

看两点：
1. **有没有 `login` 命令** → 有就是"路径一"，没有就看"路径二"
2. **提示 "requires local credentials"** → 就是路径二

---

## 路径一：站点自带 `login`（浏览器自动登录）

大多数主流站点都支持，一条命令搞定：

```bash
opencli xiaohongshu login
opencli weibo login
opencli zhihu login
opencli douyin login
opencli twitter login
opencli bilibili login
opencli youtube login
```

会自动打开浏览器 → 扫码/登录 → 会话自动保存。之后所有命令直接可用。

---

## 路径二：手动配凭证文件

部分站点（如小宇宙 xiaoyuzhou）没有 `login` 命令，需要手动创建 `~/.opencli/<site>.json`。

通用流程：

### Step 1：浏览器登录该站点

在 Chrome/Edge 里打开站点并登录，确保 cookie 里有登录态。

### Step 2：用 opencli browser 连接已登录的标签页

```bash
# 打开目标页面（会复用已有的登录 session）
opencli browser <session> open <url>

# 抓 cookie
opencli browser <session> eval "document.cookie"
```

### Step 3：提取 token 并写入配置

根据站点的 cookie / localStorage 结构，提取对应的 token 值，写入 `~/.opencli/<site>.json`。

具体每个站点的 token 字段名和配置格式不同，见 `references/` 目录下的站点专属文档。

### Step 4：验证

```bash
opencli <site> whoami
# 或直接执行目标命令
```

---

## 视频处理工作流（小红书等）

对于视频笔记（无内嵌字幕），`opencli note` 只返回标题和摘要文字，完整文字稿需要额外处理：

```
note（元信息）→ download（视频 → /tmp）→ ffmpeg（提取音频）
→ mlx-whisper / whisper（语音转文字）→ 修正专有名词 → 写入 vault → 清理 /tmp
```

详见 [references/xiaohongshu-video.md](references/xiaohongshu-video.md)。

关键约束：
- 视频下载到 `/tmp`，处理完即删，不留在 vault 目录
- Whisper 对中文专有名词识别差（OP&A → OpenAI / Emeda → NVIDIA / Ansropic → Anthropic 等），必须手动修正

---

## 已整理的站点参考

| 站点 | 类型 | 参考文档 |
|------|------|---------|
| 华尔街见闻 (wallstreetcn) | 走通用 `web` 适配器 — 浏览器自动化抓取 | [references/wallstreetcn.md](references/wallstreetcn.md) |
| 小宇宙 (xiaoyuzhou) | 路径二 — 手动配凭证 | [references/xiaoyuzhou.md](references/xiaoyuzhou.md) |
| 小红书 (xiaohongshu) | 路径一 — `login` 命令，文字帖/图片帖(OCR)/视频帖(Whisper) | [references/xiaohongshu.md](references/xiaohongshu.md) |
| 小红书视频处理 | 工作流 — 下载→转写→保存 | [references/xiaohongshu-video.md](references/xiaohongshu-video.md) |
| B站 (bilibili) | 路径一 — `login` 命令，字幕提取+whisper 兜底 | [references/bilibili.md](references/bilibili.md) |
| YouTube (youtube) | 路径一 — `login` 命令，字幕提取+whisper 兜底 | [references/youtube.md](references/youtube.md) |
| Discord (discord) | 路径二 — 手动配 token + 直接调 API（必须走代理） | [references/discord.md](references/discord.md) |
| 腾讯会议 (meeting.tencent.com) | **无内置 adapter** — 用 browser bridge 抓 DOM，分段滚动虚拟列表 | [references/tencent-meeting.md](references/tencent-meeting.md) |

---

## 通用调试技巧

```bash
# 查看站点适配器支持的完整命令列表
opencli <site> --help -f yaml

# 用 browser 模式直接执行 JS（适用于支持 browser 的站点）
opencli browser <session> eval "JSON.stringify(localStorage)"
opencli browser <session> eval "document.cookie"

# 查看当前登录账号
opencli <site> whoami

# 带 verbose 调试
opencli <site> <command> -v
```
