# 小红书 (xiaohongshu) opencli 使用指南

## 站点特征

- 有 `login` 命令，走浏览器自动化登录
- 笔记内容分三种：**文字帖**（直接获取正文）、**图片帖**（需要下载图片后 OCR）、**视频帖**（下载视频 → 提取音频 → Whisper 转写）
- **必须使用完整签名 URL**（含 `xsec_token`），仅传 note-id 会报错

## 登录

```bash
# 检查登录状态
opencli xiaohongshu whoami

# 登录（自动打开浏览器扫码/手机号登录）
opencli xiaohongshu login
```

## 获取笔记内容

### 文字帖

直接获取正文内容：

```bash
opencli xiaohongshu note "<完整URL（含xsec_token）>" --format md
```

### 图片帖

如果 `note` 返回的 `content` 只有标签没有实质正文，说明是图片帖，需要下载图片后 OCR 提取文字。

#### Step 1：下载图片

```bash
cd <用户指定的保存目录> && opencli xiaohongshu download "<完整URL（含xsec_token）>"
```

图片保存在 `<保存目录>/xiaohongshu-downloads/<note-id>/` 下。

#### Step 2：OCR 识别

使用 macOS 原生 Vision 框架（中文识别准确度高，无需额外依赖）：

```swift
// 保存为 /tmp/ocr.swift
import Vision
import AppKit
import Foundation

let args = CommandLine.arguments
guard args.count >= 2 else { print("Usage: swift ocr.swift <image_path>"); exit(1) }

let imgPath = args[1]
guard let img = NSImage(contentsOfFile: imgPath),
      let cgImage = img.cgImage(forProposedRect: nil, context: nil, hints: nil) else {
    print("Failed to load image: \(imgPath)")
    exit(1)
}

let request = VNRecognizeTextRequest()
request.recognitionLanguages = ["zh-Hans", "zh-Hant", "en"]
request.recognitionLevel = .accurate
request.usesLanguageCorrection = true

let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
do {
    try handler.perform([request])
    guard let observations = request.results else { exit(0) }
    for obs in observations {
        if let candidate = obs.topCandidates(1).first {
            print(candidate.string)
        }
    }
} catch {
    print("OCR error: \(error)")
    exit(1)
}
```

```bash
swiftc /tmp/ocr.swift -o /tmp/ocr
/tmp/ocr "<图片路径>"
```

#### Step 3：整理为 .md 文件

将 OCR 结果整理为结构化 Markdown，写入 `<保存目录>/<标题>.md`，包含：

```markdown
---
source: <原始URL>
author: <作者>
platform: xiaohongshu
tags:
  - <标签1>
  - <标签2>
date: <抓取日期>
---

# <标题>

<OCR 正文内容>
```

### 视频处理

**视频帖不要保留在 vault 目录**。下载到 `/tmp`，处理完即删。

### 视频帖

如果 `note` 返回的 `content` 只有标签和简短摘要，且 `download` 下载的是 `.mp4` 文件，说明是视频帖。视频通常无内嵌字幕，需要提取音频后语音转文字。

详见 [references/xiaohongshu-video.md](references/xiaohongshu-video.md)，流程概要：

```
note（元信息）→ download（视频 → /tmp）→ ffmpeg（提取音频 wav）
→ mlx-whisper（语音转文字）→ 修正专有名词 → 写入 vault → 清理 /tmp
```

关键约束：
- 视频下载到 `/tmp`，处理完即删，**不留在 vault 目录**
- Whisper 对中文专有名词识别差，常见修正见 [xiaohongshu-video.md](xiaohongshu-video.md) Step 6

## 保存路径确认（必须）

**每次抓取前必须先询问用户保存路径**。如果用户没有指定，默认保存到当前 IDE 打开的工作路径（即 vault 根目录）。

## 可用命令速查

| 命令 | 说明 |
|------|------|
| `opencli xiaohongshu whoami` | 查看当前登录账号 |
| `opencli xiaohongshu login` | 浏览器登录 |
| `opencli xiaohongshu note <url>` | 获取笔记正文 |
| `opencli xiaohongshu download <url>` | 下载笔记图片/视频 |
| `opencli xiaohongshu search <query>` | 搜索笔记 |
| `opencli xiaohongshu user <id>` | 查看用户公开笔记 |
| `opencli xiaohongshu comments <note-id>` | 查看评论 |

## 踩坑记录

1. **必须用完整签名 URL** — `note` 和 `download` 命令现在要求传入完整 URL（含 `xsec_token`），仅传 note-id 会报 `ARGUMENT` 错误
2. **登录过期** — 返回 `AUTH_REQUIRED` 时执行 `opencli xiaohongshu login` 重新登录
3. **图片帖 content 为空** — `note` 返回的 content 只有 `#tag1 #tag2` 时，正文在图片里，走下载+OCR 流程
