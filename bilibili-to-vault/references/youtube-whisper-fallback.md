# YouTube 无字幕兜底：音频下载 + Whisper 转写

## 适用场景

YouTube 视频通过 `opencli youtube transcript` 和 `yt-dlp --list-subs` 均确认无字幕（既无人工字幕也无自动字幕）时的备用方案。

## 完整流程

### 1. 下载音频

```bash
yt-dlp -f "bestaudio" --extract-audio --audio-format mp3 \
  -o "/tmp/yt_${video_id}.%(ext)s" \
  "https://www.youtube.com/watch?v=${video_id}"
```

- 输出：`/tmp/yt_${video_id}.mp3`
- 21 分钟视频约 19MB，< 30 秒完成

### 2. Whisper 转写

```bash
whisper /tmp/yt_${video_id}.mp3 \
  --language Chinese \
  --model tiny \
  --output_dir /tmp \
  --output_format txt
```

- Whisper 路径：`/opt/homebrew/bin/whisper`（已通过 Homebrew 安装）
- 模型选择：`tiny` 足够中文转写，比 `small` 快 5-10 倍
- ⚠️ 必须在后台运行（`background=true`），否则 600s 超时
- 21 分钟视频用 tiny 模型约 4-5 分钟完成

### 3. 检查输出

```bash
wc -l -m /tmp/yt_${video_id}.txt
```

输出为带时间戳的纯文本格式：`[00:01.200 --> 00:04.800] 转写内容`

## 已知词错与修正

Whisper tiny 对中文专有名词识别不稳定，常见错误：

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

总结 Agent 需根据上下文自动修正这些同音字错误。

## 成本

- 模型下载：tiny 约 75MB（首次），后续缓存
- 转写速度：约 1 分钟音频 / 15-20 秒（M 系列 Mac CPU）
- 21 分钟视频总耗时约 5-8 分钟（含模型加载）
