# YouTube (youtube) opencli 使用指南

## 站点特征

- 有 `login` 命令，走浏览器自动化登录
- 视频字幕通过 `opencli youtube transcript` 获取
- 部分视频无字幕（既无人工也无自动字幕），需走语音转文字兜底

## 登录

```bash
# 检查登录状态
opencli youtube whoami

# 登录（自动打开浏览器扫码/登录 Google 账号）
opencli youtube login
```

## 提取视频字幕/文稿

### Step 1：解析 URL，提取视频 ID

- 标准链接：`https://www.youtube.com/watch?v=<video_id>` → 取 `<video_id>`
- 短链接：`https://youtu.be/<video_id>` → 取 `<video_id>`

### Step 2：获取视频元信息

```bash
opencli youtube video "<url>" -f json
```

取 `title`、`author`（频道名）等字段用于后续文件命名。

### Step 3：下载字幕

```bash
opencli youtube transcript "<url>" -f plain > /tmp/yt_<video_id>_字幕.txt
```

检查条目数：

```bash
wc -l /tmp/yt_<video_id>_字幕.txt
```

- **≥ 10 条** → 字幕可用，以此为源文件进入总结
- **< 10 条** 或返回无字幕错误 → 走语音转文字兜底

### Step 4（兜底）：语音转文字

当 `opencli youtube transcript` 无字幕时，用 yt-dlp 下载音频 + whisper 转写：

```bash
# 1. 下载音频
yt-dlp -f "bestaudio" --extract-audio --audio-format mp3 \
  -o "/tmp/yt_<video_id>.%(ext)s" \
  "<视频URL>"

# 2. Whisper 转写（后台运行）
whisper /tmp/yt_<video_id>.mp3 \
  --language Chinese \
  --model tiny \
  --output_dir /tmp \
  --output_format txt
```

- Whisper 路径：`/opt/homebrew/bin/whisper`（通过 Homebrew 安装）
- 模型选 `tiny`（约 75MB，首次自动下载），比 `small` 快 5-10 倍
- 转写速度约 1 分钟音频 / 15-20 秒（M 系列 Mac CPU）
- 21 分钟视频总耗时约 5-8 分钟（含模型加载）
- 输出：`/tmp/yt_<video_id>.txt`（带时间戳格式 `[00:01.200 --> 00:04.800] 内容`）

### Whisper 同音字修正常见词

Whisper tiny 对中文专有名词识别不稳定，总结时需根据上下文修正：

| Whisper 输出 | 正确应为 |
|-------------|---------|
| 偷肯 | Token |
| 欧盆入特 | OpenRouter |
| 处能 | 储能 |
| 书电 | 输电 |
| 编压器 / 编达气王 | 变压器 |
| 鸡电保护 | 继电保护 |
| 气风气光 | 弃风弃光 |
| 连价 | 廉价 |

## 保存路径确认（必须）

**每次抓取前必须先询问用户保存路径**。如果用户没有指定，默认保存到当前 IDE 打开的工作路径（即 vault 根目录）。

## 可用命令速查

| 命令 | 说明 |
|------|------|
| `opencli youtube whoami` | 查看当前登录账号 |
| `opencli youtube login` | 浏览器登录 |
| `opencli youtube video <url>` | 获取视频元信息 |
| `opencli youtube transcript <url>` | 获取视频字幕/文稿 |
| `opencli youtube search <query>` | 搜索视频 |
| `opencli youtube comments <url>` | 获取评论 |
| `opencli youtube channel <id>` | 获取频道信息和视频列表 |

## 踩坑记录

1. **无字幕** — 部分视频既无人工字幕也无自动字幕，`transcript` 和 `yt-dlp --list-subs` 均确认无字幕时走 whisper 兜底
2. **whisper 同音字** — 见上表，总结时需上下文修正
3. **短链接** — `youtu.be/xxx` 直接取 video_id 即可，无需重定向解析
