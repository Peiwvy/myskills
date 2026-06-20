# 小红书视频处理：下载 → 转写 → 保存

适用于小红书视频笔记（无内嵌字幕），需要获取完整文字稿的场景。

## 前置条件

- `opencli xiaohongshu login` 已完成
- `mlx-whisper` 已安装（macOS Apple Silicon）或 `whisper`（通用）
- `ffmpeg` 可用

## 完整流程

### Step 1：获取笔记元信息

```bash
opencli xiaohongshu note "<完整URL>" -f json
```

返回 title、author、content（正文摘要）、tags 等。

### Step 2：下载视频到 /tmp

```bash
opencli xiaohongshu download "<完整URL>" --output-dir /tmp/xhs-video
```

> **注意**：默认下载到当前目录的 `xiaohongshu-downloads/`，视频帖必须指定 `--output-dir /tmp/...`，处理完即删，不留在 vault 目录。

### Step 3：检查视频信息

```bash
# 查看时长和音轨
ffprobe -v quiet -print_format json -show_streams /tmp/xhs-video/<video-file>.mp4 | \
  python3 -c "import sys,json; d=json.load(sys.stdin); streams=[s for s in d.get('streams',[]) if s['codec_type'] in ('audio','subtitle')]; print(json.dumps(streams, indent=2, ensure_ascii=False))"

# 查看时长
ffprobe -v quiet -show_entries format=duration -of csv=p=0 /tmp/xhs-video/<video-file>.mp4
```

### Step 4：提取音频（16kHz 单声道 WAV）

```bash
ffmpeg -y -i /tmp/xhs-video/<video-file>.mp4 \
  -vn -acodec pcm_s16le -ar 16000 -ac 1 \
  /tmp/xhs_audio.wav
```

### Step 5：语音转文字

**macOS Apple Silicon（推荐，速度快）**：

```bash
mlx_whisper --model mlx-community/whisper-large-v3-turbo \
  --language zh \
  --output-format txt \
  --output-dir /tmp \
  /tmp/xhs_audio.wav
```

**通用（PyTorch，有 GPU 则更快）**：

```bash
whisper /tmp/xhs_audio.wav \
  --model large-v3 \
  --language zh \
  --output_format txt \
  --output_dir /tmp
```

### Step 6：修正转写结果

Whisper 对中文专有名词识别不佳，常见需要手动修正：

| Whisper 输出 | 正确 |
|-------------|------|
| OP&A | OpenAI |
| Emeda | NVIDIA |
| Ansropic / Anthopic | Anthropic |
| Topic | Grok |
| 生争亦 / 生亦 | 孙正义 |
| Orako | Oracle |
| 秉化 | 饼画 |
| 环教组 | 黄教主 |
| GIMM | Gemini |

### Step 7：清理临时文件

```bash
rm -rf /tmp/xhs-video /tmp/xhs_audio.wav /tmp/xhs_audio_*.txt
```

### Step 8：写入 vault

按 `K10-Source/Media/小红书/<作者>/` 路径保存，frontmatter 含 source、author、date、tags、url、duration 等字段。

## 模型选择

| 场景 | 推荐模型 | 速度 (M 系列) |
|------|---------|--------------|
| 中文为主，快速 | `mlx-community/whisper-large-v3-turbo` | ~30s / 5分钟音频 |
| 中英混合，高精度 | `mlx-community/whisper-large-v3` | ~60s / 5分钟音频 |
| 非 Apple Silicon | `large-v3` (PyTorch whisper) | 取决于 GPU |
