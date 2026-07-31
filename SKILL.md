---
name: video-subtitle-translator
description: 'Create subtitles for audio or video, translate them when requested, infer the target language from the user request when omitted, and default subtitled video delivery to burned-in MP4, with soft-subtitle MKV as the secondary mode. Requires the oo CLI. Use when the user asks to transcribe uploaded media, generate SRT/VTT subtitles, translate subtitles with an OOMOL-hosted LLM, or add subtitles to a video using FFmpeg and Fusion API ASR.'
metadata:
  icon: ':lucide:captions:'
  title: Video Subtitle Translator
  packageName: '@alwaysmavs/video-subtitle-translator'
  version: 0.0.12
---

# Video Subtitle Translator

## When to Use

Use this skill when the user asks to create subtitles for a local audio or
video file, translate subtitles, produce `.srt` or `.vtt` sidecar files, embed
soft subtitles into a video, or burn translated subtitles into video frames.

The ASR path is OOMOL built-in Fusion API Qwen ASR file transcription. Do not
ask the user for an ASR provider key and do not rediscover the ASR capability at
runtime. The LLM translation path uses the model configuration from `oo llm
config --json`; do not ask the user for an OpenAI key unless `oo llm config`
fails or returns unusable values.

## Inputs

- Required: a local audio or video path, or a publicly reachable audio URL.
- Translation intent: most video-subtitle product requests are for translated
  subtitles because the viewer usually does not share the source language.
  Translate when the user explicitly asks for translated subtitles, a
  translated video, video translation, or "subtitles and translation".
- If the user asks to "add subtitles", "make subtitles", "subtitle this video",
  or otherwise requests video subtitles without clearly saying whether they want
  translation, pause before execution and ask them to choose a target language
  for translation or explicitly confirm same-language/no-translation subtitles.
- If the request is only to transcribe, generate captions, or add same-language
  subtitles, and that same-language/no-translation intent is explicit, do not
  translate.
- Default translation target: when translation is clearly requested and the
  user does not name a target language, infer the target from the natural
  language of the user's request. The language they used to ask is usually
  their preferred subtitle language. For example, a Chinese request defaults to
  Simplified Chinese (`zh`), a Japanese request defaults to Japanese (`ja`), a
  Spanish request defaults to Spanish (`es`), and an English request defaults
  to English (`en`). For Chinese, use Simplified Chinese unless the user writes
  in Traditional Chinese or explicitly asks for Traditional Chinese,
  Cantonese, or another variant.
- Ask when unclear: if the wording makes it unclear whether translation is
  wanted, ask the user which language to translate subtitles into, or ask them
  to confirm they want same-language/no-translation subtitles. If translation
  is clearly wanted but the user's preferred target language cannot be inferred
  from the request, ask which target language to use.

- Defaults: source language omitted for unknown or multilingual audio;
  `enableWords: true`; `enableITN: false`; SRT output; burned-in MP4 for
  translated videos; sidecar-only output for audio.
- Default video delivery mode: burned-in MP4. Most users want a single MP4 file
  whose subtitles are visible everywhere, especially on social platforms,
  mobile players, and upload workflows that ignore subtitle tracks.
- Secondary video delivery mode: soft-subtitle MKV. Use MKV when the user asks
  for editable/selectable subtitles, no re-encoding, subtitle tracks, multiple
  languages, MKV specifically, soft subtitles, or when burn-in encoding is not
  available or unsuitable.
- Optional: source language code, output directory, VTT output, soft subtitle
  mode (`soft-mkv` or `soft-mp4`), burn-in mode, subtitle language code, and
  whether to keep both original and translated subtitle tracks.
- Optional translation style inputs: `translation_profile`, `domain`,
  `audience`, `style_notes`, `glossary`, and `video_context`.
  - `translation_profile` defaults to `general`.
  - Supported profiles: `general`, `film_tv`, `youtube_explainer`,
    `technical_course`, `interview_podcast`, `news_documentary`,
    `business_training`, `gaming_stream`, and `kids_content`.
  - Infer `translation_profile` from the user's wording when obvious. Use
    `film_tv` for films, TV dramas, sitcoms, streaming shows, and scripted
    dialogue. Use `youtube_explainer` for YouTube-style explainers, reviews,
    tutorials, and creator videos. Use `technical_course` for courses, API
    walkthroughs, coding videos, academic talks, or other terminology-heavy
    material. Use `interview_podcast` for interviews, podcasts, panel
    conversations, or unscripted long-form speech. Use `news_documentary` for
    news, documentary, or factual narration. Use `business_training` for
    corporate, sales, compliance, or onboarding material. Use `gaming_stream`
    for game streams and esports content. Use `kids_content` for children's
    videos. If the scene is unclear, keep `general` instead of asking.
  - `domain` should briefly describe the topic when known, such as
    `AI developer tools`, `medical lecture`, or `sitcom dialogue`.
  - `audience` should describe the expected viewer when known, such as
    `general viewers`, `software developers`, or `adult streaming viewers`.
  - `style_notes` should preserve the user's requested style, for example
    `natural Netflix-style Chinese subtitles` or `accurate but not too formal`.
  - `glossary` is an optional list of source-to-target term mappings for names,
    products, acronyms, technical terms, and recurring phrases.
  - `video_context` is optional high-level context such as title, description,
    speaker notes, episode setting, or project-specific terminology.
- Supported Fusion API source language codes are `zh`, `yue`, `en`, `ja`,
  `de`, `ko`, `ru`, `fr`, `pt`, `ar`, `it`, `es`, `hi`, `id`, `th`, `tr`,
  `uk`, `vi`, `cs`, `da`, `fil`, `fi`, `is`, `ms`, `no`, `pl`, and `sv`.
  Omit `language` instead of guessing when the user says auto, unknown, or
  multilingual.

## Execution

This skill ships a bundled helper script at `scripts/subtitle-tools.mjs`.
Resolve it relative to this `SKILL.md` directory and prefer it for local
transcript-to-subtitle conversion and LLM subtitle translation. Do not recreate
the conversion or translation code inline when the script is available.

### Required oo CLI Gate

The OOMOL `oo` CLI is mandatory for this skill. Verify it before starting media
processing:

```bash
oo --version
```

If the shell reports that `oo` is missing, or if a later `oo` command fails
because authentication or configuration is unavailable, read
[`references/oo-cli-setup.md`](references/oo-cli-setup.md) completely and
follow it. Load that reference only for setup or recovery; do not load it when
`oo` is already available and working. Do not silently replace Fusion API ASR
or OO-hosted translation with another provider. If the user declines the
required installation or sign-in, stop and explain that this skill cannot run.

### 1. Check FFmpeg First

Run:

```bash
ffmpeg -version
ffprobe -version
```

If either command is missing, stop and guide the user to install FFmpeg before
processing video or extracting audio. Suggested installs:

- macOS with Homebrew: `brew install ffmpeg`
- Windows/Linux: download the matching prebuilt FFmpeg archive from
  `https://github.com/jellyfin/jellyfin-ffmpeg/releases`, extract it, and add
  the directory containing `ffmpeg` and `ffprobe` to `PATH`.

Do not recommend `sudo apt install ffmpeg` for Linux by default. It is more
invasive than needed for this workflow and may install an older or differently
configured distro build.

After installation, ask the user to open a new terminal or refresh `PATH`, then
rerun the version checks. Do not continue with video work when FFmpeg is
missing. For a publicly reachable audio URL and sidecar-only subtitle output,
FFmpeg is not needed unless conversion, muxing, or burning is requested.

### 2. Check Node.js First

The bundled helper script requires a local JavaScript runtime. Use Node.js 18
or newer because the script is an ES module and uses the built-in `fetch` API.

Run:

```bash
node --version
node -e "const major=Number(process.versions.node.split('.')[0]); process.exit(major >= 18 ? 0 : 1)"
```

If `node` is missing or the version check fails, stop and guide the user to
install Node.js 18 LTS or newer before running `scripts/subtitle-tools.mjs`.
Suggested installs:

- macOS with Homebrew: `brew install node`
- Windows with winget: `winget install OpenJS.NodeJS.LTS`
- Ubuntu/Debian: install Node.js 18+ with NodeSource or `nvm`; distro
  `apt install nodejs npm` is acceptable only when it provides Node.js 18+

After installation, ask the user to open a new terminal or refresh `PATH`, then
rerun the version checks. Do not continue with local subtitle conversion or LLM
translation when the JavaScript runtime is missing or older than Node.js 18.

### 3. Prepare Audio

Create a stable work directory such as `outputs/subtitles-<input-name>/`. Reuse
the same directory on reruns when the user is retrying the same input. Do not
reuse translation checkpoint entries unless `--resume` is explicitly passed;
the helper only resumes entries whose saved source text still matches the
current subtitle cue.

For local video or any media that needs normalization, extract mono 16 kHz WAV:

```bash
ffmpeg -y -i "$INPUT_MEDIA" -vn -ac 1 -ar 16000 -c:a pcm_s16le "$WORK_DIR/audio.wav"
```

For a local audio file that Fusion API can read directly, uploading the
original file is acceptable. If the upload or ASR rejects the format, convert it
with the same FFmpeg command and upload `audio.wav`.

Upload the audio that will be transcribed:

```bash
oo file upload "$AUDIO_PATH" --json
```

Use the returned `downloadUrl` as `fileURL`. Do not pass local filesystem paths
to Fusion API connector actions.

### 4. Submit Fusion API ASR

Use this exact connector action:

```bash
oo connector run "fusion-api" \
  --action "qwen_asr_filetrans_submit" \
  --data @submit-asr.json \
  --json
```

Payload skeleton:

```json
{
  "fileURL": "https://...",
  "language": "en",
  "enableITN": false,
  "enableWords": true
}
```

Rules:

- `fileURL` is required and must be the uploaded audio `downloadUrl` or a
  public audio URL.
- Omit `language` when unknown or multilingual; otherwise use one supported
  code from the input list above.
- Set `enableWords: true` so the result can be converted into timed subtitles.
- Omit `channelID` unless the user specifically asks for one or more channels.
- The submit response returns `sessionId`. Save it in `job.created.json`.

### 5. Poll and Fetch Result

Poll state with:

```bash
oo connector run "fusion-api" \
  --action "qwen_asr_filetrans_state" \
  --data "{\"sessionID\":\"$SESSION_ID\"}" \
  --json
```

Expected states:

- `{"state":"processing","progress":...}`: wait and poll again.
- `{"state":"completed"}`: fetch the result.
- `{"state":"not_found","error":"..."}`: stop and report the missing session.

Fetch result with:

```bash
oo connector run "fusion-api" \
  --action "qwen_asr_filetrans_result" \
  --data "{\"sessionID\":\"$SESSION_ID\"}" \
  --json
```

The completed result has `state: "completed"` and useful transcript data at
`data`. Save the full response as `job.done.json` and `data` as
`transcript.json`. Schema indicates `data` includes `taskID`,
`transcriptionURL`, `usage`, and `transcription` details. This field shape was
schema-confirmed; exact nested transcript content can vary by media.

### 6. Build Source Subtitles

Use the bundled script to convert the saved Fusion API result or
`transcript.json` into timed subtitle files:

```bash
node "$SKILL_DIR/scripts/subtitle-tools.mjs" fusion-to-subtitles \
  --input "$WORK_DIR/transcript.json" \
  --out-dir "$WORK_DIR" \
  --time-unit ms \
  --formats srt
```

Pass `--formats all` when VTT was requested. The script contains the
Sublinea-style timed-word cue segmentation defaults and writes
`transcript.txt`, `transcript.srt`, `transcript.word-timed.srt`, and optionally
`transcript.word-timed.vtt`.

Convert Fusion API transcript data into an internal timed-word list:

- Iterate `data.transcription.transcripts[]`.
- Use `transcript.text` for plain text when present.
- For each `transcript.sentences[]`, use sentence `beginTime`, `endTime`,
  `text`, `language`, and `words[]`.
- For each word, use `beginTime`, `endTime`, `text`, and optional
  `punctuation`.
- Preserve punctuation by appending it to the preceding word when present.

Apply conservative noise cleanup only while reading Fusion ASR output:

- Drop a word or sentence token only when its complete text is three or more
  zeroes, such as `000`; never apply this rule to user-provided SRT or LLM
  translations.
- Do not append `punctuation` when the word text already ends with the same
  punctuation.
- Preserve normal expressive forms such as `...`, `??`, `!!`, and `?!`.
- Normalize only clearly excessive runs: four or more periods to `...`, three
  or more identical question or exclamation marks to two, long mixed
  question/exclamation runs to their first meaningful pair, and repeated
  commas, semicolons, colons, or enumeration commas to one.
- Report the number of probable zero artifacts removed and punctuation runs
  repaired. Keep user subtitles and translated subtitles content-preserving.

Normalize timestamps to seconds. Fusion API timestamps use milliseconds, so
the helper defaults to `--time-unit ms` and divides them by `1000`. For a
non-Fusion transcript whose timestamps are already seconds, pass
`--time-unit s` explicitly. Do not infer the unit from timestamp magnitude.
Keep a copy of the raw `transcript.json`.

Segment timed words into subtitle cues with these defaults, adapted from the
Sublinea project:

- maximum cue duration: `4.2` seconds
- target cue duration: `2.8` seconds
- maximum cue characters: `54`
- maximum words per cue: `12`
- split at pauses of at least `0.55` seconds
- cue start padding: `0.08` seconds
- cue end padding: `0.16` seconds
- minimum gap between cues: `0.05` seconds

Write:

- `transcript.txt`
- `transcript.srt`
- `transcript.word-timed.srt`
- `transcript.word-timed.vtt`, when VTT was requested

Use SRT as the stable exchange format for translation and soft subtitle muxing.
For burned-in subtitles, convert the final SRT to ASS first so font size,
outline, alignment, and bottom margin are interpreted in an explicit script
resolution instead of relying on FFmpeg's SRT-to-ASS defaults.

### 7. Translate Subtitles With OO LLM Config

When translation is requested, use the target language named by the user. If
the user clearly requested translation but omitted the target language, infer
the target from the natural language of the user's request. Then run:

```bash
oo llm config --json
```

Use the returned `apiKey`, `baseUrl`, and `model` for an OpenAI-compatible chat
completions request to `${baseUrl without trailing slash}/chat/completions`.
Do not hardcode, persist, log, or print the API key.

Then use the bundled script to translate SRT cue text while preserving cue
indexes and timing. Example for a Chinese-language request:

```bash
node "$SKILL_DIR/scripts/subtitle-tools.mjs" translate-srt \
  --input "$WORK_DIR/transcript.srt" \
  --out-dir "$WORK_DIR" \
  --source-language auto \
  --target-language "Simplified Chinese" \
  --target-code zh \
  --profile youtube_explainer \
  --formats srt
```

Adjust `--target-language`, `--target-code`, `--source-language`, `--profile`,
`--domain`, `--audience`, `--style-notes`, `--glossary-json`, and
`--video-context-json` from the user's request and the inferred target language.
For example, use `--target-language "Simplified Chinese" --target-code zh` for
a Chinese-language request, `--target-language "Japanese" --target-code ja` for
a Japanese-language request, and `--target-language "Spanish" --target-code es`
for a Spanish-language request. If the request language is mixed, ambiguous, or
not a stable signal of the user's preferred subtitle language, ask for the
target language before translating. Pass `--formats all` when VTT was requested.
The script calls `oo llm config --json`, sends OpenAI-compatible chat
completions requests, writes `translation.<target-code>.json` as a resumable
checkpoint after each batch, retries failed batches at smaller sizes, and writes
`translation.<target-code>.srt` plus optional VTT.

On a retry, pass `--resume` to reuse completed checkpoint entries. Without
`--resume`, translate from the beginning and overwrite the checkpoint. When
resuming, the helper reuses an entry only when both its cue index and saved
source text match the current SRT; old checkpoints without saved source text
are ignored.

Translate only subtitle cue text. Preserve cue indexes and all timing fields.
Use batches of about 30 cues with a small context window before and after the
batch. Include any available translation style inputs in the user payload:
`translation_profile`, `domain`, `audience`, `style_notes`, `glossary`, and
`video_context`. Keep the prompt stable across profiles; let the profile and
metadata drive style adaptation. Require the model to return JSON:

```json
{
  "items": [
    { "index": 1, "text": "translated subtitle text" }
  ]
}
```

Recommended request body:

```json
{
  "model": "<oo llm model>",
  "temperature": 0.2,
  "messages": [
    {
      "role": "system",
      "content": "You are a professional subtitle translator.\n\nReturn only valid JSON with this exact shape: {\"items\":[{\"index\":1,\"text\":\"translated subtitle text\"}]}.\n\nHard requirements:\n- Translate only subtitle cue text.\n- Preserve every requested cue index exactly.\n- Do not add, remove, merge, split, or reorder subtitle cues.\n- Preserve meaning, speaker intent, tone, names, numbers, dates, brands, code terms, and important repeated phrases.\n- Keep each subtitle concise, natural, and readable on screen.\n- Use context_before and context_after only to resolve meaning, references, pronouns, tone, and continuity.\n- Follow glossary entries when provided. Keep source terms unchanged when the glossary says so or when a product, API, command, code symbol, or proper noun should remain in the source language.\n- Do not add explanations, notes, markdown, or extra JSON fields.\n\nStyle adaptation:\n- If translation_profile is \"film_tv\", translate into natural spoken dialogue. Preserve character emotion, humor, subtext, register, and relationship dynamics. Avoid stiff literal phrasing. Adapt slang, insults, jokes, and profanity to an equivalent natural intensity in the target language.\n- If translation_profile is \"youtube_explainer\", translate clearly and naturally for online video viewers. Keep domain terminology accurate while avoiding overly academic wording. Preserve tool names, product names, acronyms, and technical concepts unless a standard target-language translation exists.\n- If translation_profile is \"technical_course\", prioritize precision, terminology consistency, and instructional clarity. Use standard technical terms. Avoid embellishment or casual paraphrase that may reduce accuracy.\n- If translation_profile is \"interview_podcast\", preserve the speaker's tone and conversational rhythm. Lightly clean filler words only when they hurt subtitle readability, without changing the speaker's position.\n- If translation_profile is \"news_documentary\", use a neutral, accurate, polished style. Avoid slang unless it is essential to the source.\n- If translation_profile is \"business_training\", use concise professional language with consistent business terminology.\n- If translation_profile is \"gaming_stream\", use energetic, natural spoken language. Preserve game-specific terms, memes, reactions, and player intent.\n- If translation_profile is \"kids_content\", use simple, friendly, age-appropriate wording.\n- Otherwise use a natural general subtitle style."
    },
    {
      "role": "user",
      "content": "{\"source_language\":\"auto\",\"target_language\":\"Simplified Chinese\",\"translation_profile\":\"youtube_explainer\",\"domain\":\"AI developer tools\",\"audience\":\"general technical viewers\",\"style_notes\":\"Natural, accurate subtitles; keep product names and standard technical terms consistent.\",\"video_context\":{\"title\":\"Building an AI agent with tool calling\",\"description\":\"A YouTube tutorial for developers\",\"speaker_notes\":\"One speaker explaining a workflow casually.\"},\"glossary\":[{\"source\":\"agent\",\"target\":\"智能体\"},{\"source\":\"tool calling\",\"target\":\"工具调用\"}],\"context_before\":[],\"subtitles\":[{\"index\":1,\"text\":\"Today we're going to build a simple agent with tool calling.\"}],\"context_after\":[]}"
    }
  ]
}
```

For film, TV, and other scripted dialogue, prefer a payload like:

```json
{
  "source_language": "auto",
  "target_language": "Simplified Chinese",
  "translation_profile": "film_tv",
  "domain": "scripted dialogue",
  "audience": "adult streaming viewers",
  "style_notes": "Natural spoken Chinese subtitles; avoid translationese.",
  "video_context": {
    "title": "Episode or scene title when known",
    "description": "Brief setting, relationship, or plot context when known"
  },
  "glossary": [],
  "context_before": [
    { "index": 28, "text": "What the hell are you doing here?" }
  ],
  "subtitles": [
    { "index": 29, "text": "I told you, I had nowhere else to go." }
  ],
  "context_after": [
    { "index": 30, "text": "You shouldn't have come back." }
  ]
}
```

Validate that every requested cue index has a non-empty translated text. On
partial or invalid JSON, retry with a smaller batch; for a single-cue failure,
report the model error. Write:

- `translation.<target-code>.json` as a resumable checkpoint
- `translation.<target-code>.srt`
- `translation.<target-code>.vtt`, when VTT was requested

### 8. Prepare Display Subtitles

Before creating external subtitles, soft subtitles, or burned-in subtitles,
normalize the final translated SRT into a display SRT. This keeps all delivery
forms consistent: sidecar SRT, styled ASS, soft MKV ASS, and burned-in MP4
should use the same cue text and line breaks.

For Simplified Chinese and other CJK subtitles, do not use the English-style
`37` characters per line as the visual line length. Use a CJK-aware line limit:

- default CJK line length: `18` characters per line
- strict Simplified Chinese delivery: `16` characters per line
- maximum lines per cue: `2`
- prefer one-line subtitles when the cue fits
- when two lines are needed, prefer a bottom-heavy shape and avoid leaving only
  one or two characters on the top line
- split overlong cues into multiple sequential cues before generating ASS,
  rather than forcing a second line beyond the line limit

Run:

```bash
node "$SKILL_DIR/scripts/subtitle-tools.mjs" prepare-display-srt \
  --input "$WORK_DIR/translation.$TARGET_CODE.srt" \
  --output "$WORK_DIR/translation.$TARGET_CODE.display.srt" \
  --cjk-line-length 18 \
  --max-lines 2
```

Use `--cjk-line-length 16` when the user asks for stricter professional or
Netflix-style Simplified Chinese line limits. Use the display SRT as
`$SUBTITLE_SRT` for the rest of the workflow.

### 9. Add Subtitles to Video

For audio-only inputs, deliver sidecar subtitle files. For video inputs, choose
the user's requested mode or default to burned-in MP4.

Default mode rules:

- If the user does not specify an output mode, create a burned-in MP4.
- If the user asks for "subtitled video", "video with subtitles", "translated
  subtitles on this video", "add subtitles to this video", or similar generic
  wording, create a burned-in MP4 unless the user asks for soft/selectable
  subtitle tracks.
- If the user asks for styled subtitles, visual parity, social sharing, upload
  compatibility, or subtitles that always display, prefer burned-in MP4.
- If the user asks for "烧录", "硬字幕", "hard subtitles", "burned-in",
  "permanent subtitles", "export MP4", or "MP4 with subtitles", create a
  burned-in MP4.
- If the user asks for "soft subtitles", "外挂字幕", "可开关字幕", "subtitle
  track", "selectable subtitles", "no re-encode", "MKV", or multiple subtitle
  languages in one file, create a soft MKV.
- When the user wants sidecar subtitles and burned-in subtitles from the same
  job, generate both from the display SRT so cue boundaries and line breaks
  match.

Sidecar files:

- Return `translation.<target-code>.display.srt` as the compatibility-first
  external subtitle file.
- Also return `translation.<target-code>.display.ass` when styled external
  subtitles are useful or when the user wants visual parity with burned-in
  output.

Create the styled ASS sidecar from the display SRT:

```bash
node "$SKILL_DIR/scripts/subtitle-tools.mjs" srt-to-burn-ass \
  --input "$SUBTITLE_SRT" \
  --output "$WORK_DIR/translation.$TARGET_CODE.display.ass" \
  --video-width "$VIDEO_WIDTH" \
  --video-height "$VIDEO_HEIGHT" \
  --font-name "PingFang SC" \
  --font-size 56 \
  --margin-v 38 \
  --margin-l 80 \
  --margin-r 80 \
  --outline 5
```

Soft MKV, compatibility-first:

```bash
ffmpeg -y -i "$INPUT_VIDEO" -i "$SUBTITLE_SRT" \
  -map 0:v? -map 0:a? -map 1:0 \
  -c copy -c:s srt \
  -disposition:s:0 default \
  -metadata:s:s:0 language="$LANG_CODE" \
  -metadata:s:s:0 title="Translated subtitles" \
  "$OUTPUT_VIDEO.mkv"
```

Soft MKV, style-consistent with burn-in:

```bash
ffmpeg -y -i "$INPUT_VIDEO" -i "$SUBTITLE_ASS" \
  -map 0:v? -map 0:a? -map 1:0 \
  -c copy -c:s ass \
  -disposition:s:0 default \
  -metadata:s:s:0 language="$LANG_CODE" \
  -metadata:s:s:0 title="Styled translated subtitles" \
  "$OUTPUT_VIDEO.styled.mkv"
```

Soft MP4:

```bash
ffmpeg -y -i "$INPUT_VIDEO" -i "$SUBTITLE_SRT" \
  -map 0:v? -map 0:a? -map 1:0 \
  -c copy -c:s mov_text \
  -disposition:s:0 default \
  -metadata:s:s:0 language="$LANG_CODE" \
  -metadata:s:s:0 title="Translated subtitles" \
  "$OUTPUT_VIDEO.mp4"
```

Burned-in MP4:

Before burning, convert the display SRT to ASS with explicit video resolution
and bottom-centered styling:

```bash
node "$SKILL_DIR/scripts/subtitle-tools.mjs" srt-to-burn-ass \
  --input "$SUBTITLE_SRT" \
  --output "$WORK_DIR/subtitles.burn.ass" \
  --video-width "$VIDEO_WIDTH" \
  --video-height "$VIDEO_HEIGHT" \
  --font-name "PingFang SC" \
  --font-size 56 \
  --margin-v 38 \
  --margin-l 80 \
  --margin-r 80 \
  --outline 5
```

Use `ffprobe` to set `VIDEO_WIDTH` and `VIDEO_HEIGHT` from the actual input
video. For 1920x1080 videos, the default burn-in style is bottom-centered
Chinese subtitles with `PlayResX: 1920`, `PlayResY: 1080`, `Alignment=2`, and
`MarginV=38`. Keep this as the fixed default unless the user explicitly asks
for a different position. Do not reposition subtitles by checking individual
screenshots frame by frame.

Then burn the ASS file:

```bash
ffmpeg -y -i "$INPUT_VIDEO" \
  -vf "ass=$WORK_DIR/subtitles.burn.ass" \
  -c:v libx264 -crf 18 -preset medium -c:a copy -sn \
  "$OUTPUT_VIDEO.burned.mp4"
```

Use burned-in MP4 as the normal video delivery default. Prefer soft subtitles
only when the user values editability, selectable tracks, multiple languages,
smaller processing cost, or avoiding video re-encoding over universal playback.

Soft MP4 subtitles (`mov_text`) cannot preserve the same typography, outline,
or exact positioning as ASS/burn-in. Treat soft MP4 as a niche compatibility
mode, not the default. If visual consistency matters, prefer burned-in MP4; use
ASS sidecar or soft MKV with ASS only when the user asks for editable or
selectable subtitles.

Burn-in positioning rules:

- Prefer ASS over direct `subtitles=$SUBTITLE_SRT` for burned-in output.
- Always include `PlayResX` and `PlayResY` matching the input video. ASS
  margins, font size, and coordinates are script-resolution pixels; mismatched
  or implicit resolution can make `MarginV` appear much higher than intended.
- For normal horizontal subtitles, use bottom center (`Alignment=2`) and a
  fixed bottom margin. On 1080p output, use `MarginV=38`; for other heights,
  default to about `3.5%` of video height, with a floor near `28` pixels.
- Treat the whole video as one canvas by default. Only avoid lower-third
  graphics or on-screen text when the user specifically requests manual
  per-scene placement.
- If the user says the subtitles are too high or too low, adjust the ASS
  `MarginV` only. Smaller `MarginV` moves subtitles closer to the bottom;
  larger `MarginV` moves them upward.
- Use white text with a black outline by default for readability. For Chinese
  on 1080p, `Fontsize=56`, `Outline=5`, `Shadow=0`, `MarginL=80`, and
  `MarginR=80` are the fixed defaults unless the user asks otherwise.

## Result Handling

Report the generated files with clear local paths. At minimum, return the source
SRT path and, when requested, the translated SRT path. For video inputs, also
return the subtitled video path when soft muxing or burning was requested.

If a generated video or subtitle file is practical to preview in the current
environment, open or display it for the user; otherwise provide the exact path.
Do not print raw transcript JSON unless the user asks for debugging details.
Do not print the OO LLM API key.

Typical output names:

- `job.created.json`
- `job.done.json`
- `transcript.json`
- `transcript.txt`
- `transcript.srt`
- `transcript.word-timed.srt`
- `translation.<target-code>.json`
- `translation.<target-code>.srt`
- `translation.<target-code>.display.srt`
- `translation.<target-code>.display.ass`
- `translation.<target-code>.burn.ass`
- `<name>.subtitled.mkv`
- `<name>.styled.mkv`
- `<name>.subtitled.mp4`
- `<name>.burned.mp4`

## Failure Handling

- Missing `oo` CLI or unavailable OO authentication/configuration: read
  `references/oo-cli-setup.md` and follow its matching recovery path. Do not
  treat `oo: command not found` as an ordinary upload failure.
- Missing FFmpeg or FFprobe: stop, give install instructions, and ask the user
  to rerun after installation.
- Missing input media: stop and ask for the path or URL.
- `oo file upload` fails: report the upload error; for local files, verify the
  path exists and retry with a normalized audio file when format rejection is
  likely.
- Fusion API submit schema rejection: check `fileURL`, language code,
  `enableWords`, and `channelID`; do not rediscover actions.
- ASR timeout: report the saved `sessionId` and explain that the agent can
  resume by polling `qwen_asr_filetrans_state` and
  `qwen_asr_filetrans_result`.
- ASR result has no word timestamps: write `transcript.txt` if text exists, but
  explain that timed subtitles require `enableWords: true` or a transcript with
  sentence/word timing.
- `oo llm config --json` fails: report that OO-hosted LLM configuration is not
  available and ask the user to fix oo CLI authentication/configuration.
- LLM translation returns invalid JSON or missing cue indexes: retry smaller
  batches, then fail with the specific cue indexes that could not be translated.
- Display SRT still has CJK lines over the requested limit: rerun
  `prepare-display-srt` with a smaller `--cjk-line-length`, usually `16`, and
  regenerate ASS from the display SRT.
- FFmpeg soft MP4 subtitle muxing fails: use burned-in MP4 as the default
  fallback. If burn-in is unavailable or the user explicitly needs selectable
  subtitle tracks, retry soft MKV.
- Burn-in encoding fails because `libx264` is unavailable: ask the user to
  install a full FFmpeg build with `libx264` support or choose soft subtitles.
- Burned-in subtitles look too high: verify that the burn-in input is ASS, not
  raw SRT, and that the ASS file has `PlayResX` and `PlayResY` matching the
  video. If those are correct, reduce `MarginV` instead of moving subtitles
  with per-frame screenshot adjustments.
