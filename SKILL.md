---
name: watch-video
description: SLASH-COMMAND-ONLY. Invoke ONLY when the user explicitly types the literal `/watch-video` slash command. Never auto-trigger on phrases like "watch this video," "summarize this video," "take notes on this video," or on a bare YouTube URL. For ad-hoc YouTube questions, fetch the transcript directly (YouTube page or yt-dlp) and skim — do not run this heavyweight pipeline.
---

# Watch Video

Claude can't stream video directly. This skill fakes it: a Python pipeline (vendored from [bradautomates/claude-video](https://github.com/bradautomates/claude-video) under `scripts/`) downloads the video, extracts auto-scaled JPEG frames with ffmpeg, pulls a timestamped transcript (native captions first, Whisper API fallback), and prints a markdown report listing every frame path. Claude then `Read`s each frame, aligns it to the spoken text, and writes a structured notes file.

## When to invoke

**Slash-command only.** Run this skill ONLY when the user literally types `/watch-video`. That is the sole trigger.

Do NOT invoke on:
- Casual phrases like "watch this video," "summarize this," "take notes on this YouTube video," "analyze this reel"
- A bare YouTube URL pasted with a question ("what useful tips are in this video?" + URL)
- Any natural-language ask about a video that lacks the explicit `/watch-video` command

**Default for YouTube questions without the slash command:** pull the transcript the fastest way available — fetch the YouTube page / use yt-dlp captions / a transcript site — skim it, and answer from that. The frame-extraction pipeline is overkill unless the user explicitly asks for it via `/watch-video`.

> If you'd rather have auto-trigger behavior, edit the `description:` line above to match the keywords you want Claude to fire on (e.g. "Use when the user wants Claude to watch, analyze, or take notes on a video"). Slash-only is the default in this repo because explicit invocation prevents accidental token burn on long videos.

## Dependencies

- **ffmpeg + ffprobe** on PATH — for frame and audio extraction
- **yt-dlp** on PATH — for downloading and caption fetching
- **Python 3.9+** — the bundled scripts use `from __future__ import annotations` so 3.9 works
- **Optional:** Whisper API key for videos without native captions. Set `GROQ_API_KEY` (preferred — cheaper/faster, runs `whisper-large-v3`) or `OPENAI_API_KEY` in `~/.config/watch/.env`. Without one, captioned videos work fine; uncaptioned videos return frames-only.

Run `python scripts/setup.py --check` to verify dependencies, or `python scripts/setup.py` to scaffold the `.env` and check binaries. On macOS, the installer auto-installs missing binaries via Homebrew. On Linux/Windows, it prints exact install commands.

## Pipeline

The work happens in `scripts/watch.py`. It downloads, extracts, transcribes, and prints a markdown report to stdout that lists every frame path. The pipeline auto-scales the frame budget by duration (hard cap 100 frames / 2 fps), so no manual interval tuning.

```
python scripts/watch.py "<youtube-url-or-local-path>" [flags]
```

Flags worth knowing:
- `--start T` / `--end T` — focus on a section (`SS`, `MM:SS`, or `HH:MM:SS`). Auto-scales fps denser inside the range. Use this for any question about a specific moment, or for any video > 10 min where the user's question is about one part.
- `--max-frames N` — lower the cap for tighter token budget (default 80, hard max 100).
- `--resolution W` — frame width in px (default 512; bump to 1024 only if on-screen text is unreadable).
- `--whisper groq|openai` — force a specific Whisper backend (default: prefer Groq if both keys exist).
- `--no-whisper` — skip transcription entirely if no captions. Frames-only output.
- `--out-dir DIR` — keep working files somewhere specific (default: an auto-generated tmp dir).

Auto-fps budgets (full-video mode):
- ≤30s → up to 30 frames
- 30s–1min → ~40 frames
- 1–3min → ~60 frames
- 3–10min → ~80 frames
- \>10min → 100 frames sparse (warning printed; consider `--start`/`--end`)

## Step-by-step workflow

### 1. Run the pipeline

Default invocation, no flags:

```
python scripts/watch.py "<source>"
```

For long videos where the user asked about a specific moment, pass `--start`/`--end`:

```
python scripts/watch.py "<source>" --start 2:15 --end 2:45
```

The script writes everything to a tmp working directory and prints a markdown report to stdout. Capture the stdout — it contains:
- Header (Title, Uploader, Duration, Transcript source: `captions` / `whisper (groq)` / `whisper (openai)` / `none available`)
- `## Frames` section with `- \`<absolute-path>\` (t=MM:SS)` lines
- `## Transcript` section with `[MM:SS] text...` lines
- Footer with `Work dir: <path>`

### 2. Read every frame

Read all the listed frame paths in a single message (parallel `Read` tool calls). The Read tool renders JPEGs as images. Each frame's filename + `t=MM:SS` from the report tells you when it occurred — pair each frame with the matching transcript line at that timestamp.

For very long videos (>10 min, sparse mode): the budget already capped at 100 frames, so reading all of them is fine.

### 3. Write the summary

Output file: `<work-dir>/<slug>-notes.md` by default. If the user said "save notes somewhere permanent," ask where.

Structure the markdown like this:

```markdown
# <Title>

**Source:** <URL or local path>
**Duration:** <mm:ss>
**Uploader:** <if from YouTube>
**Transcript source:** <captions / whisper (groq) / whisper (openai) / none>

## One-line summary
<≤20 words — the core claim or hook of the video>

## TL;DR
<3–5 bullet points capturing the main arguments, moments, or beats>

## Timeline
- **[00:00]** <what's happening visually + the key line being said>
- **[00:15]** ...
<one row per meaningful beat, not per frame>

## Key quotes
> "<verbatim quote>" — [mm:ss]

## Visual notes
<what the video shows that the transcript alone would miss — setting, B-roll, on-screen text, graphics, transitions, subject's emotion>

## Takeaways (optional)
<Only include if the content is directly relevant to the user's domain or goals. Omit otherwise.>
```

### 4. Clean up

After the `.md` is written, delete the work dir (it contains the full downloaded video + frames, which is large). The script prints `Work dir: <path>` in the footer of its report; pass that path to `rm -rf`.

If the user specified a non-tmp `--out-dir`, ask before deleting.

## Common gotchas

- **YouTube Shorts / age-gated / members-only** — yt-dlp may fail. Surface its stderr verbatim; don't retry silently.
- **No captions + no Whisper key** — the report says `Transcript: none available` and points at `setup.py`. Tell the user they can add a Groq key to `~/.config/watch/.env` for Whisper, or use `--no-whisper` for frames-only.
- **Local file with no audio track** — Whisper extraction errors out cleanly. Use `--no-whisper` for frames-only.
- **Very long videos (>30 min)** — confirm with the user before running. The pipeline caps at 100 frames so the budget is bounded, but a sparse 100-frame scan of a 60-min video isn't very useful. Almost always better to run focused on the specific section.
- **Cloudflare 403 on Groq** — `whisper.py` already sets a custom User-Agent to clear Cloudflare's default-Python-UA block. If you ever see a 403, that's the failure mode.
- **No automatic Groq → OpenAI fallback** — the script picks one Whisper backend at the start of each run (Groq if its key is set, else OpenAI) and stays on it. If Groq rate-limits, the script retries Groq twice then errors out. To use OpenAI instead, pass `--whisper openai`.

## What NOT to do

- Don't attempt to "watch" a video by inventing content based on title or thumbnail. If the pipeline fails, say so.
- Don't write the summary before actually reading frames. The transcript alone misses visual context (B-roll, on-screen text, emotion, graphics, transitions).
- Don't skip cleanup. Frame dumps + downloaded videos are large.
- Don't auto-fire on a video URL. Slash-command-only — see "When to invoke" above.

## Engine attribution

The pipeline scripts under `scripts/` are vendored from [bradautomates/claude-video](https://github.com/bradautomates/claude-video) (MIT). See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for the full upstream license.
