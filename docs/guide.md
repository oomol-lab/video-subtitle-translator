# Video Subtitle Translator Guide

[Back to README](../README.md) | [简体中文](guide.zh-CN.md)

This guide contains the setup and implementation details intentionally kept out
of the main README.

## Contents

- [Requirements and setup](#requirements-and-setup)
- [Processing pipeline](#processing-pipeline)
- [Translation and formatting](#translation-and-formatting)
- [Video delivery](#video-delivery)
- [Output files](#output-files)
- [Cleanup policy](#cleanup-policy)
- [Security and privacy](#security-and-privacy)
- [Troubleshooting](#troubleshooting)
- [Development](#development)
- [Demo media](#demo-media)

## Requirements and setup

- Node.js 18 or newer
- FFmpeg and FFprobe
- The authenticated OOMOL `oo` CLI with access to file upload, Fusion API, and
  `oo llm config` when translation is requested

Check the runtime:

```bash
node --version
ffmpeg -version
ffprobe -version
oo --version
```

Install `oo` by following the
[official preparation guide](https://static.oomol.com/oo-cli/skill-install-guide/install.md),
then sign in:

```bash
oo auth login
```

Install the skill from OOMOL Hub:

```bash
oo skills install @alwaysmavs/video-subtitle-translator
```

For development, clone the repository into an agent skills directory:

```bash
git clone https://github.com/oomol-lab/video-subtitle-translator.git \
  ~/.agents/skills/video-subtitle-translator
```

Restart or refresh the agent environment after installation.

## Processing pipeline

### 1. Prepare and upload audio

Local video is converted to mono 16 kHz WAV when normalization is needed:

```bash
ffmpeg -y -i "$INPUT_MEDIA" -vn -ac 1 -ar 16000 \
  -c:a pcm_s16le "$WORK_DIR/audio.wav"
```

The audio is uploaded with `oo file upload`. Only its returned `downloadUrl` is
sent to Fusion API; local filesystem paths are not passed to the cloud action.

### 2. Transcribe with Fusion API

The workflow submits an ASR job, polls its state, and fetches the completed
result:

```text
qwen_asr_filetrans_submit
        ↓
qwen_asr_filetrans_state
        ↓
qwen_asr_filetrans_result
```

The raw response is preserved as `transcript.json`. Fusion timestamps are
treated as milliseconds and normalized to seconds locally.

### 3. Build subtitles locally

The bundled helper exposes these commands:

```bash
node scripts/subtitle-tools.mjs fusion-to-subtitles --help
node scripts/subtitle-tools.mjs translate-srt --help
node scripts/subtitle-tools.mjs prepare-display-srt --help
node scripts/subtitle-tools.mjs srt-to-burn-ass --help
```

It converts timed words into cues, prepares display-friendly line breaks, and
generates resolution-aware ASS styling.

## Translation and formatting

Translation uses the OpenAI-compatible model returned by:

```bash
oo llm config --json
```

The helper translates only cue text and preserves cue indexes and timestamps.
It supports general, film/TV, explainer, technical course, interview, news,
business, gaming, and children’s-content profiles.

Translations are checkpointed. `--resume` reuses a cue only when both its index
and saved source text match the current SRT.

For CJK subtitles, the default display layout uses up to two lines with about
18 characters per line. Overlong cues are split before ASS generation instead
of being compressed into unreadable lines.

## Video delivery

### Burned-in MP4 — default

The display SRT is converted to ASS using the source video dimensions, then
burned into the image with FFmpeg and libass. Use this for social media,
uploads, mobile playback, and any environment where subtitles must always be
visible.

### Soft-subtitle MKV — optional

SRT or ASS can be packaged as a selectable subtitle track without re-encoding
the video. Use this for editable subtitles, multiple languages, or when users
need to turn subtitles on and off.

Soft MP4 with `mov_text` is supported as a niche compatibility mode, but it
cannot preserve ASS typography and positioning.

## Output files

A typical job may produce:

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

## Cleanup policy

Cleanup is intentionally limited to Fusion ASR input:

- Pure zero tokens such as `000` are treated as probable ASR artifacts.
- Duplicate punctuation is not appended twice.
- Normal forms such as `...`, `??`, `!!`, and `?!` are preserved.
- Only clearly excessive punctuation runs are normalized.
- User SRT, LLM translations, and checkpoint text remain content-preserving.

## Security and privacy

- Local media is uploaded so Fusion API can access it.
- Subtitle text is sent to the LLM configured by `oo llm config` when
  translation is requested.
- API keys are read at runtime and must not be printed or written to outputs.
- Raw transcript files may contain sensitive speech and should not be published
  without review.

## Troubleshooting

- Missing `oo` or authentication errors: follow
  [oo CLI setup and recovery](../references/oo-cli-setup.md).
- Missing FFmpeg/FFprobe: install a full FFmpeg build and refresh `PATH`.
- Missing Node.js: install Node.js 18 or newer.
- ASR timeout: keep the saved session id and resume polling later.
- Burn-in failure: confirm that FFmpeg includes libass and libx264, or choose
  soft-subtitle MKV.

## Development

Before submitting changes, run at minimum:

```bash
node --check scripts/subtitle-tools.mjs
node scripts/subtitle-tools.mjs --help
```

Changes to timestamp, punctuation, language, or checkpoint behavior should
include a small fixture covering the affected edge case.

## Demo media

The README demos use:

- [“Generative AI explained in 2 minutes” by KI-Campus](https://commons.wikimedia.org/wiki/File:Generative_AI_explained_in_2_minutes.webm)
- [“Job Interview” by Simpleshow Japan](https://commons.wikimedia.org/wiki/File:Job_Interview.webm)

Both are licensed under
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). The recordings
show the agent workflow and modified subtitle outputs.
