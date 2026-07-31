# Video Subtitle Translator 详细指南

[返回中文 README](../README.zh-CN.md) | [English](guide.md)

本文档收纳主 README 中刻意省略的环境配置和实现细节。

## 目录

- [环境要求与安装](#环境要求与安装)
- [处理流程](#处理流程)
- [字幕翻译与排版](#字幕翻译与排版)
- [视频交付](#视频交付)
- [输出文件](#输出文件)
- [ASR 清理策略](#asr-清理策略)
- [安全与隐私](#安全与隐私)
- [故障处理](#故障处理)
- [开发与贡献](#开发与贡献)
- [演示素材](#演示素材)

## 环境要求与安装

- Node.js 18 或更高版本
- FFmpeg 和 FFprobe
- 已登录的 OOMOL `oo` CLI，并具备文件上传、Fusion API，以及翻译时使用
  `oo llm config` 的权限

检查本地环境：

```bash
node --version
ffmpeg -version
ffprobe -version
oo --version
```

按照 [官方准备指南](https://static.oomol.com/oo-cli/skill-install-guide/install.md)
安装 `oo`，然后登录：

```bash
oo auth login
```

从 OOMOL Hub 安装 Skill：

```bash
oo skills install @alwaysmavs/video-subtitle-translator
```

如需参与开发，也可以克隆到 Agent Skills 目录：

```bash
git clone https://github.com/oomol-lab/video-subtitle-translator.git \
  ~/.agents/skills/video-subtitle-translator
```

安装后请重启或刷新 Agent 环境。

## 处理流程

### 1. 准备并上传音频

需要规范化时，本地视频会先转换成单声道 16 kHz WAV：

```bash
ffmpeg -y -i "$INPUT_MEDIA" -vn -ac 1 -ar 16000 \
  -c:a pcm_s16le "$WORK_DIR/audio.wav"
```

随后通过 `oo file upload` 上传。只有返回的 `downloadUrl` 会提交给 Fusion API；
本地文件系统路径不会直接传给云端 Action。

### 2. 使用 Fusion API 识别

流程会提交 ASR 任务、轮询状态，并获取最终结果：

```text
qwen_asr_filetrans_submit
        ↓
qwen_asr_filetrans_state
        ↓
qwen_asr_filetrans_result
```

原始结果保存为 `transcript.json`。Fusion 时间戳按毫秒读取，并在本地转换成秒。

### 3. 在本地生成字幕

内置工具提供以下命令：

```bash
node scripts/subtitle-tools.mjs fusion-to-subtitles --help
node scripts/subtitle-tools.mjs translate-srt --help
node scripts/subtitle-tools.mjs prepare-display-srt --help
node scripts/subtitle-tools.mjs srt-to-burn-ass --help
```

工具会把逐字时间戳转换成字幕 cue，生成适合屏幕阅读的换行，并根据真实视频分辨率
输出 ASS 样式。

## 字幕翻译与排版

翻译使用以下命令返回的 OpenAI-compatible 模型：

```bash
oo llm config --json
```

工具只翻译 cue 文本，保留 cue 序号和时间轴。支持通用、影视、科普、技术课程、
访谈、新闻、企业培训、游戏和儿童内容等翻译类型。

翻译结果会持续写入 checkpoint。使用 `--resume` 时，只有 cue 序号和保存的原文都与
当前 SRT 匹配，才会复用已有结果。

CJK 字幕默认最多两行，每行约 18 个字符。过长的 cue 会先拆分，再生成 ASS，避免
把大量文字挤在同一画面。

## 视频交付

### 烧录字幕 MP4——默认

显示用 SRT 会根据原视频分辨率转换成 ASS，再通过 FFmpeg 和 libass 烧录进画面。
适用于社交平台、移动端、上传流程，以及要求字幕始终可见的场景。

### 软字幕 MKV——可选

SRT 或 ASS 可以作为可开关字幕轨封装进 MKV，无需重新编码视频。适用于可编辑字幕、
多语言字幕，或需要随时开关字幕的场景。

也支持使用 `mov_text` 的软字幕 MP4，但它无法保留 ASS 的字体、描边和精确位置。

## 输出文件

一次典型任务可能生成：

```text
job.created.json
job.done.json
transcript.json
transcript.txt
transcript.srt
transcript.word-timed.srt
translation.<language>.json
translation.<language>.srt
translation.<language>.display.srt
translation.<language>.display.ass
<name>.burned.mp4
<name>.subtitled.mkv
```

## ASR 清理策略

清理严格限制在 Fusion ASR 输入层：

- `000` 之类的纯零 token 会被视为疑似 ASR 噪声。
- 相同标点不会被重复追加。
- 正常的 `...`、`??`、`!!` 和 `?!` 会被保留。
- 只有明显过长的异常标点序列会被规范化。
- 用户 SRT、LLM 翻译和 checkpoint 文本保持内容保真。

## 安全与隐私

- 本地媒体会被上传，以便 Fusion API 访问。
- 请求翻译时，字幕文本会发送给 `oo llm config` 配置的 LLM。
- API Key 只在运行时读取，不应打印或写进输出文件。
- 原始逐字稿可能包含敏感语音，公开前应先检查。

## 故障处理

- 缺少 `oo` 或认证失败：查看 [oo CLI 安装与恢复](../references/oo-cli-setup.md)。
- 缺少 FFmpeg/FFprobe：安装完整 FFmpeg，并刷新 `PATH`。
- 缺少 Node.js：安装 Node.js 18 或更高版本。
- ASR 超时：保留已保存的 session id，之后继续轮询。
- 烧录失败：确认 FFmpeg 包含 libass 和 libx264，或改用软字幕 MKV。

## 开发与贡献

提交修改前至少运行：

```bash
node --check scripts/subtitle-tools.mjs
node scripts/subtitle-tools.mjs --help
```

修改时间戳、标点、语言或 checkpoint 行为时，应添加一个覆盖相应边界情况的小型
fixture。

## 演示素材

README 演示使用：

- [KI-Campus《Generative AI explained in 2 minutes》](https://commons.wikimedia.org/wiki/File:Generative_AI_explained_in_2_minutes.webm)
- [Simpleshow Japan《Job Interview》](https://commons.wikimedia.org/wiki/File:Job_Interview.webm)

两者均采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 授权。
演示录像展示了 Agent 的执行过程和经过翻译、排版、烧录的字幕输出。
