---
name: bilibili-to-vault
description: 视频 → 字幕提取 → vault-summarize 总结。支持 B站 / YouTube，支持批量 URL 全并行处理。当用户发视频链接并要求「总结」「整理成笔记」时触发。也接受直接贴链接无额外说明。
---

## 视频 → Obsidian 笔记 多 Agent 管线（B站 + YouTube）

### 触发条件

- 用户发送 bilibili.com / b23.tv / youtube.com / youtu.be 链接
- 用户说「总结这些视频」「把这些视频整理成笔记」等
- 用户直接贴链接无额外说明 → 默认处理

### 架构：主 Agent 调度 + 每视频一个独立子 Agent（全并行）

```
用户发 N 个链接
    │
    ▼
┌──────────────────────────────────────┐
│  主 Agent (你)                        │
│  解析所有 URL，识别平台，获取元信息    │
│  构建 N 个独立 task，一次性 delegate   │
└──────┬───────────────────────────────┘
       │
       │ delegate_task(tasks=[...])  ← N 个任务全并行
       │
       ├──▶ 子 Agent A: 下载BV1xx字幕 → 总结 → 返回
       ├──▶ 子 Agent B: 下载BV2yy字幕 → 总结 → 返回
       └──▶ 子 Agent C: 下载YT字幕    → 总结 → 返回
```

每个子 Agent 完全独立：下载字幕 → 读全 → 按 rules.md 写笔记 → 自检 → 返回结果。彼此零依赖，并行执行。

### 主 Agent 执行流程

#### Step 0: 解析所有 URL

从用户输入提取所有视频链接。对每个 URL：

1. **识别平台 + 提取 ID**：
   - `bilibili.com/video/BV` → B站，取 BV 号
   - `b23.tv/` → `curl -sI` 跟随重定向，取最终 URL 的 BV 号
   - `youtube.com/watch?v=` / `youtu.be/` → YouTube，取完整 URL

2. **获取元信息**（用于 context 中的文件命名）：
   - B站：`opencli bilibili video <bvid> -f json`
   - YT：`opencli youtube video "<url>" -f json`
   - 某视频元信息获取失败 → 标记跳过，不放入 tasks

#### Step 1: 一次性派发所有子 Agent（并行）

用 `delegate_task(tasks=[...])` 批量派发。每个 task 是一个完整独立的"下载+总结"子 Agent。

**单个 B站 task：**
```
goal: 下载B站视频 BV{bvid} 字幕并整理为 Obsidian 笔记

context 中包含:

【第一步：下载字幕】
  opencli bilibili subtitle {bvid} -f plain > /Users/ztyu/{bvid}_字幕.txt
  sed -n 's/^content: //p' /Users/ztyu/{bvid}_字幕.txt > /tmp/{bvid}_纯字幕.txt
  wc -l -m /tmp/{bvid}_纯字幕.txt
  → 若条目 ≥ 10 → 进入第二步
  → 若条目 < 10 → 进入兜底：语音转文字

【兜底：语音转文字（无字幕时自动触发）】
  1. yt-dlp -f "bestaudio" --extract-audio --audio-format mp3 -o "/tmp/{bvid}.%(ext)s" "{url}"
  2. whisper /tmp/{bvid}.mp3 --language Chinese --model tiny --output_dir /tmp --output_format txt
  3. 转写结果在 /tmp/{bvid}.txt，以此为源文件继续第二步总结
  4. 注意：whisper 输出可能有同音字错误，总结时需根据上下文修正

【第二步：总结】
  输出: /Users/ztyu/{bvid}-{主题}-总结.md
  视频: 标题="{title}", 作者="{author}", URL={url}
  
  ===== RULES.MD 完整内容 =====
  {从 ~/.hermes/skills/vault-summarize/rules.md 读取的完整内容}
  ===== RULES.MD 结束 =====
  
  要求: read_file读全源 → 按rules整理 → self-check → 返回路径

【第三步：返回】
  JSON: {"status": "OK/SKIPPED", "platform": "bilibili", "bvid": "...", 
         "title": "...", "summary_path": "...", "char_count": N}

toolsets: ["file","terminal"]
```

**单个 YouTube task：**
```
同上，下载命令替换为:
  opencli youtube transcript "{url}" -f plain > /Users/ztyu/yt_{video_id}_字幕.txt
  若条目 ≥ 10 → 进入总结
  若条目 < 10 → 兜底：yt-dlp下音频 → whisper tiny转文字 → 以转写结果为源文件总结
  返回 JSON 中 platform 为 "youtube"

如果 opencli youtube transcript 无字幕 → 改用 whisper 兜底方案
 （详见 references/youtube-whisper-fallback.md）：
  1. yt-dlp 下载音频 mp3
  2. whisper --language Chinese --model tiny（后台 background=true）
  3. 读取 /tmp/yt_{video_id}.txt 作为字幕源
  4. 按 rules.md 整理时自动修正常见同音字错误
```

> [!important] 必须嵌入 rules.md
> 子 Agent 无 memory。每个 task 的 context 必须完整粘贴 rules.md。主 Agent 在构建 tasks 前先读取 rules.md 文件内容。

#### Step 2: 汇总

收集全部子 Agent 返回，输出：

```
✅ 管线完成：处理 N 个，成功 X 个，跳过 Y 个

| # | 平台 | 视频 | 状态 | 总结 |
|---|------|------|------|------|
| 1 | B站 | 动态工作流 | ✅ | BV...-总结.md |
| 2 | B站 | 算力出海    | ✅ | BV...-总结.md |
| 3 | YT  | GPT-5解读   | ⏭️ | 无字幕        |
```

### 架构原则

| 原则 | 说明 |
|------|------|
| 主 Agent 只调度 | 解析 URL、获取元信息、构建 tasks、派发、汇总。不亲自下载或总结 |
| 子 Agent 全自治 | 每个独立完成下载→总结→自检，互不依赖 |
| 全并行 | N 个视频一次性 delegate_task(tasks=[...])，不串行等待 |
| 自包含 context | 每个 task 的 context 嵌入完整 rules.md，子 Agent 无需外部知识 |

### 注意事项

- 并发上限 3（`delegation.max_concurrent_children`），其余自动排队
- b23.tv 短链接需 curl -sI 解析真实 URL
- **字幕兜底**：无字幕时自动走 yt-dlp 下音频 → whisper tiny 转文字 → 总结。whisper 转写结果可能有同音字错误，总结 Agent 需根据上下文修正（如"偷肯"→Token、"处能"→储能）
- 超长视频（>30k 汉字）→ context 中指示按拆分模式
- whisper 转写耗时约 3-5 分钟（21 分钟视频），tiny 模型在 CPU 上可接受
