# OpenMontage — Accumulated Knowledge Base

This file is checked into the repo so it transfers to every new installation.
It captures hard-won fixes, gotchas, and workflow rules discovered in production.
Read this after `AGENT_GUIDE.md`. It overrides any contradictory defaults.

---

## Remotion — Asset Loading

### Always pass `--public-dir` when rendering

Remotion bundles to a temp directory at render time. Assets inside `assets/` are NOT
copied into that bundle automatically. Without `--public-dir`, every video/audio/image
asset will 404 in the headless Chromium renderer.

**Rule**: always pass `--public-dir=<absolute-path-to-assets>` to every `npx remotion render` call.

```bash
npx remotion render Explainer \
  --props=.remotion_props.json \
  --public-dir=/absolute/path/to/project/assets \
  --codec=h264 \
  --output=renders/final.mp4
```

**Rule**: always use paths relative to the `assets/` root in props JSON — never `file://` URIs,
never absolute paths. `OffthreadVideo` and `Audio` only accept paths that Remotion's HTTP
proxy can serve from inside `--public-dir`.

---

## Remotion — Theme and Colors

### Always include `themeConfig` in props for dark-background videos

If `themeConfig` is absent from the props JSON root, Remotion uses `DEFAULT_THEME` which
has `textColor: "#1F2937"` (near-black). On dark backgrounds ALL chart labels, axis text,
KPI subtitles, and category names become invisible.

**Rule**: always include a `themeConfig` block whenever you write `.remotion_props.json` directly.

```python
"themeConfig": {
    "primaryColor": "#C8963E",
    "accentColor": "#C8963E",
    "backgroundColor": "#0D0D0D",
    "surfaceColor": "#1A1A1A",
    "textColor": "#FFFFFF",          # critical — without this, labels are invisible
    "mutedTextColor": "#A0A0A0",
    "headingFont": "Inter",
    "bodyFont": "Inter",
    "monoFont": "JetBrains Mono",
    "chartColors": ["#C8963E", "#00A8CC", "#FF3B30", "#34C759", "#5856D6", "#F5A623"],
    "springConfig": {"damping": 20, "stiffness": 120, "mass": 1},
    "transitionDuration": 0.4,
    "captionHighlightColor": "#C8963E",
    "captionBackgroundColor": "rgba(15,23,42,0.75)"
}
```

### Chart components don't inherit theme — pass `textColor` explicitly

`BarChart`, `LineChart`, `PieChart`, and `KPIGrid` receive their text and grid colors only
via explicit props. In `Explainer.tsx`, the chart dispatch block must compute dark/light
variants and pass them:

```tsx
const isDarkBg = !isLightColor(bgColor === "transparent" ? theme.backgroundColor : bgColor);
const chartGridColor = isDarkBg ? "rgba(255,255,255,0.12)" : "#E5E7EB";
const kpiCardBg = isDarkBg ? "rgba(255,255,255,0.07)" : "#F9FAFB";
// then: textColor={textColor} gridColor={chartGridColor} cardBackgroundColor={kpiCardBg}
```

This fix is already applied to `Explainer.tsx`. Do not remove it.

### HeroTitle uses `accentColor` prop, not hardcoded colors

The `HeroTitle` component applies `accentColor` to the first word and `color` to the
rest. Both props are wired through `Explainer.tsx`. Pass them in cut props:

```json
{ "type": "hero_title", "text": "The Title", "accentColor": "#C8963E", "color": "#FFFFFF" }
```

For all-white titles (no accent): `"accentColor": "#FFFFFF"`. This fix is in the codebase.

### `ExplainerShorts` composition for vertical (9:16) video

For YouTube Shorts, use `ExplainerShorts` (1080×1920) — not `Explainer` (1920×1080).
Both are registered in `Root.tsx`. All scene components adapt automatically via `AbsoluteFill`.

```bash
npx remotion render ExplainerShorts --props=... --public-dir=...
```

---

## Remotion — Component Data Contracts

### KPIGrid requires numeric `value`, not strings

```json
{ "label": "Year", "value": 105, "suffix": " AD", "icon": "📜" }   ✓
{ "label": "Year", "value": "105 AD" }                               ✗ → NaN count-up
```

Use `prefix`/`suffix` for units. `formatDisplayValue` auto-formats ≥1M → "3.5M", ≥1K → "1.5K".

### LineChart `x` must be a number, not a string

```json
{ "x": 1820, "y": 42 }   ✓
{ "x": "1820", "y": 42 } ✗
```

---

## TTS — edge-tts

### Generate TTS sentence by sentence — never as a single block

edge-tts prosody degrades severely on long scripts. Passing a 10-minute narration as
one block flattens all intonation — every sentence gets "mid-clause" delivery with no
proper sentence-ending fall. Each sentence only gets natural prosody when rendered on
its own individual call.

**Rule**: split the narration at sentence boundaries (`.`, `!`, `?`), call edge-tts
once per sentence writing to a numbered temp file, then concatenate with ffmpeg.

```python
import re, subprocess

sentences = [s.strip() for s in re.split(r'(?<=[.!?])\s+', script) if s.strip()]
for i, sent in enumerate(sentences):
    subprocess.run(
        ['edge-tts', '-t', sent, '-v', voice, '--write-media', f'/tmp/seg_{i:03d}.mp3'],
        check=True, stdin=subprocess.DEVNULL, capture_output=True,
    )

with open('/tmp/tts_list.txt', 'w') as f:
    for i in range(len(sentences)):
        f.write(f"file '/tmp/seg_{i:03d}.mp3'\n")

subprocess.run(
    ['ffmpeg', '-y', '-f', 'concat', '-safe', '0',
     '-i', '/tmp/tts_list.txt', '-c', 'copy', narration_path],
    check=True, stdin=subprocess.DEVNULL, capture_output=True,
)
```

Use correct full-sentence punctuation in each sentence. Never flatten sentences into
comma-separated runs — that destroys natural intonation.

Note: edge-tts XML-escapes all input before sending to the SSML API. Custom `<break/>`
or `<emphasis>` tags will not work — they are passed through as literal text.

### Write full grammatical English — apostrophes and possessives

edge-tts handles apostrophes correctly. Dropping them causes mispronunciation.

```
"China's gift"    → correct pronunciation  ✓
"China gift"      → "China gift" (wrong)   ✗
```

Always write: contractions, possessives, full grammar. edge-tts handles all of it.

### Always loudnorm the narration after generating it

edge-tts outputs audio that peaks at exactly 0 dBFS. Passing that directly to Remotion
or FFmpeg causes audible clipping. **Always run loudnorm immediately after TTS generation**,
before measuring duration or using the file for anything else:

```python
import subprocess, shutil

def normalize_narration(src: str, dst: str) -> None:
    """Normalize TTS audio to -14 LUFS, TP=-1.5 dB. Overwrites dst."""
    subprocess.run(
        ['ffmpeg', '-y', '-i', src,
         '-af', 'loudnorm=I=-14:LRA=11:TP=-1.5',
         dst],
        check=True, capture_output=True,
    )
```

Run this immediately after `edge-tts` finishes, before anything else touches the file.
The normalized file replaces the raw output. This also applies to `google_tts` and any
other TTS provider — always normalize before use.

### Generate TTS first, then build visual timings (TTS-first workflow)

Never plan cut `in_seconds`/`out_seconds` from estimated durations. TTS output varies.

**Correct workflow**:
1. Write script
2. Generate TTS
3. **Normalize the audio** (see loudnorm rule above)
4. Measure actual duration with `ffprobe`
5. Derive section start times from cumulative durations (or faster-whisper word timestamps)
6. Build cuts from real audio timestamps

```python
# Measure actual TTS duration
result = subprocess.run(
    ['ffprobe', '-v', 'quiet', '-print_format', 'json', '-show_streams', path],
    capture_output=True, text=True
)
dur = float(json.loads(result.stdout)['streams'][0]['duration'])
```

For word-level timestamps (precise section boundaries):
```python
from faster_whisper import WhisperModel
model = WhisperModel('base', device='cpu', compute_type='int8')
segments, _ = model.transcribe('narration.mp3', word_timestamps=True, language='en')
words = [{'word': w.word.strip(), 'start': w.start} for seg in segments for w in seg.words]
```

---

## TTS — Google (AI Studio key routing)

If `GOOGLE_API_KEY` starts with `AQ.`, it is a Google AI Studio key. The Cloud TTS API
(`texttospeech.googleapis.com`) rejects these with 401. The fix is already applied in
`tools/audio/google_tts.py` — it detects AI Studio keys and routes to the Gemini TTS
endpoint (`generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts`).

Gemini TTS returns PCM audio — decoded from base64 and converted via ffmpeg:
`-f s16le -ar 24000 -ac 1 -i pipe:0 output.mp3`

---

## exec_command + write_stdin Is Broken — Never Use It

**`write_stdin` always fails in this environment** with:
`error=write_stdin failed: stdin is closed for this session`

`exec_command` closes the subprocess stdin immediately after launch. Any subsequent
`write_stdin` call will find the pipe already closed and error out, causing the render
to fail silently while the model reports success.

**Rule: never pipe data to a subprocess via `exec_command` + `write_stdin`.**

Instead, always write data to a file first, then pass the file path:

```python
# WRONG — will fail:
proc = exec_command("ffmpeg -f s16le -ar 24000 -i pipe:0 out.mp3")
write_stdin(proc.pid, pcm_bytes)

# CORRECT — write to file first:
Path("tmp_audio.pcm").write_bytes(pcm_bytes)
exec_command("ffmpeg -f s16le -ar 24000 -ac 1 -i tmp_audio.pcm out.mp3")
```

This applies to ffmpeg, any TTS tool, and any other subprocess that expects piped input.
Always use the OpenMontage tool wrappers (`google_tts`, `edge_tts`, `video_compose`, etc.)
which already handle file I/O correctly — do not call ffmpeg or TTS APIs manually via
`exec_command` + `write_stdin`.

---

## B-roll Pacing

### Max 5 seconds per footage cut (hard cap)

No single stock video or still-image cut may run longer than **5 seconds** in the final
timeline. This is a hard cap — hero and emotional slots included. Infotainment viewers
expect a visual change every 4–5 seconds. If a subject needs more screen time, use two
different clips of the same subject back-to-back.

**Planning rule**: for a section with N seconds of coverage, plan `ceil(N/5)` cuts and
acquire `ceil(N/5) × 1.5` unique clips (buffer for rejects).

For a 10-minute video: acquire at least 120 × 1.5 = 180 candidate clips (not 10–13).
With the 60-second reuse window, many of these can overlap across sections — plan
~30–40 distinct downloads minimum for a 10-min video.

**Exception**: animated Remotion scenes (`stat_card`, `bar_chart`, `kpi_grid`, etc.) may
run 8–20 seconds — internal animation keeps them engaging. The 5s cap applies only
to raw footage and still images.

---

## Subtitles — FFmpeg Burn-In

### Keep subtitles low and modestly sized

For landscape documentary videos, burned-in subtitles should sit near the lower edge,
not high in the frame. Use smaller text unless the user asks for large captions.

Recommended FFmpeg ASS style for 1080p documentary subtitles:
```
FontSize=14,Alignment=2,MarginV=34,BorderStyle=3,Outline=1
```

Avoid large subtitle settings like `FontSize=17` with `MarginV=72` on 1080p output;
that places the subtitle block too high and makes the frame feel crowded.

### Clip reuse — 60-second window, not global uniqueness

Do not reuse the same clip within 60 seconds of its previous appearance. Reuse across
well-separated sections of a long video is permitted — this is not a violation. The
rule prevents back-to-back or nearby repeats, not every occurrence of a clip.

Using global uniqueness (never reuse any clip ever) causes long-script refusals when
the model calculates hundreds of unique clips are required. The correct rule: 60-second
window. Search with varied queries per segment to minimize reuse naturally.

---

## Environment / `.env` Loading

### Strip inline comments from `.env` values

The project `.env` uses inline comments:
```
PIXABAY_API_KEY=abc123  # Pixabay stock footage
```

A naive `split('=', 1)[1].strip()` captures the comment as part of the value, causing
400 API errors. Always strip:

```python
val = raw_value.strip()
if " #" in val:
    val = val.split(" #")[0].strip()
val = val.strip('"').strip("'")
```

Or use `python-dotenv`: `from dotenv import load_dotenv; load_dotenv()` — it handles
inline comments correctly.

---

## Pipeline Configuration

### Each job starts completely fresh — no asset reuse across jobs

Every new job must use a **new project slug** and download all its own assets from
scratch. Never reference files from a previous job's `projects/<old_slug>/` or
`pipeline/<old_slug>/` directory.

**Why this matters**: The `projects/<slug>/assets/` directory is ephemeral. Trimmed
clips and downloaded raw footage are not guaranteed to persist between jobs. If you
reuse a stale `asset_manifest.json` or `checkpoint_*.json` from a previous run, the
`video_compose` tool will immediately fail with "Cut source not found" because the
files it references no longer exist. This is not a recoverable error — you cannot
compose without the physical clip files.

**Rules:**

- Generate a fresh project slug for every job. If the topic matches an existing
  directory, append a suffix (`-v2`, `-v3`, etc.) to guarantee a clean state.
- Never copy or reference `pipeline/<old_slug>/asset_manifest.json` as a starting
  point for a new job.
- Never reference `projects/<old_slug>/assets/` paths in new edit decisions. Those
  files are from a different job and will not be present.
- If an existing slug already has `checkpoint_compose.json` (fully done), it is a
  finished job — do not attempt to re-render it. Start fresh with a new slug.
- All footage must be downloaded fresh for each job via `direct_clip_search` or
  `corpus_builder`. Treat the stock sources as the only valid asset source.

---

### Never modify pipeline configuration files during a job

The following files define how the pipeline works. They are governance documents, not
job artifacts. **Do not read, edit, or overwrite any of these files during a job run:**

- `pipeline_defs/*.yaml` — pipeline stage definitions
- `schemas/pipelines/*.json` — JSON schema files
- `skills/pipelines/**/*.md` — stage director skill files
- `AGENT_GUIDE.md`, `AGENT_KNOWLEDGE.md` — this file and the routing guide

If you believe a rule in one of these files is wrong or missing, note it in your final
output report — do not patch the file yourself. Unauthorized edits to pipeline config
silently corrupt every future job that reads that file.

---

### Active production pipeline: `documentary-montage`

This system runs as a headless YouTube channel automation pipeline. The active
production pipeline is `documentary-montage`. All other pipelines in `pipeline_defs/`
(`animated-explainer`, `cinematic`, `hybrid`, `animation`, `clip-factory`,
`avatar-spokesperson`, `localization-dub`, `podcast-repurpose`, `screen-demo`,
`talking-head`, `character-animation`) are present for future use and are fully
functional — they are not the current default.

When the job payload does not specify a pipeline, always default to
`documentary-montage`. If a future job clearly requires a different pipeline (e.g., an
avatar presenter, an animated explainer, a screen walkthrough), the other manifests and
their stage director skills are available in `skills/pipelines/`.

**No human is ever attached to this system.** `noHumanIntervention` is always `true`.
See `AGENT_GUIDE.md` → Autonomous Mode for the full rule set.

---

## Platform Notes

### Linux / VPS (no Windows-specific issues)

On Linux the following Windows-specific problems do NOT apply:
- `exec(open(file).read())` cp1252 encoding issues — Python uses UTF-8 by default on Linux
- `npx` not found in subprocess without `shell=True` — `PATH` is inherited correctly on Linux
- Backslash path separators — use `Path` or forward slashes regardless

All other entries in this file apply on all platforms.
