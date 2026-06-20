# B站 (bilibili) opencli 使用指南

## 站点特征

- 有 `login` 命令，走浏览器自动化登录
- 视频字幕通过 `opencli bilibili subtitle` 获取
- 部分视频无字幕，需走语音转文字兜底

## 登录

```bash
# 检查登录状态
opencli bilibili whoami

# 登录（自动打开浏览器扫码）
opencli bilibili login
```

## 提取视频字幕/文稿

### Step 1：解析 URL，提取 BV 号

- 标准链接：`https://www.bilibili.com/video/BV1xx...` → 取 `BV1xx...`
- 短链接 `b23.tv/xxx`：`curl -sI "<url>"` 跟随重定向，从最终 URL 取 BV 号

### Step 2：获取视频元信息

```bash
opencli bilibili video <bvid> -f json
```

取 `title`、`author` 等字段用于后续文件命名。

### Step 3：下载字幕

```bash
opencli bilibili subtitle <bvid> -f plain > /tmp/<bvid>_字幕.txt
```

提取纯字幕内容：

```bash
sed -n 's/^content: //p' /tmp/<bvid>_字幕.txt > /tmp/<bvid>_纯字幕.txt
```

检查条目数：

```bash
wc -l /tmp/<bvid>_纯字幕.txt
```

- **≥ 10 条** → 字幕可用，以此为源文件进入总结
- **< 10 条** → 无有效字幕，走语音转文字兜底

### Step 4（兜底）：语音转文字

当 opencli 无法获取字幕时，用 yt-dlp 下载音频 + whisper 转写：

```bash
# 1. 下载音频
yt-dlp -f "bestaudio" --extract-audio --audio-format mp3 \
  -o "/tmp/<bvid>.%(ext)s" "<视频URL>"

# 2. Whisper 转写（后台运行）
whisper /tmp/<bvid>.mp3 \
  --language Chinese \
  --model tiny \
  --output_dir /tmp \
  --output_format txt
```

- Whisper 路径：`/opt/homebrew/bin/whisper`
- 模型选 `tiny`，转写速度约 1 分钟音频 / 15-20 秒（M 系列 Mac）
- 输出：`/tmp/<bvid>.txt`（带时间戳格式 `[00:01.200 --> 00:04.800] 内容`）
- whisper 转写有同音字错误，总结时需根据上下文修正

## 保存路径确认（必须）

**每次抓取前必须先询问用户保存路径**。如果用户没有指定，默认保存到当前 IDE 打开的工作路径（即 vault 根目录）。

## 可用命令速查

| 命令 | 说明 |
|------|------|
| `opencli bilibili whoami` | 查看当前登录账号 |
| `opencli bilibili login` | 浏览器登录 |
| `opencli bilibili video <bvid>` | 获取视频元信息 |
| `opencli bilibili subtitle <bvid>` | 获取视频字幕 |
| `opencli bilibili search <query>` | 搜索视频 |
| `opencli bilibili comments <bvid>` | 获取评论 |
| `opencli bilibili download <bvid>` | 下载视频（需 yt-dlp） |

## 踩坑记录

1. **b23.tv 短链接** — 需 `curl -sI` 跟随重定向获取真实 BV 号
2. **部分视频无字幕** — `subtitle` 返回条目 < 10 时走 whisper 兜底
3. **whisper 同音字** — tiny 模型对中文专有名词识别不稳定，常见：偷肯→Token、处能→储能、编压器→变压器，总结时需上下文修正
