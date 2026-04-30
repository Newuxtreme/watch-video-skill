# watch-video skill

A Claude skill that lets Claude "watch" videos by extracting a time-synced transcript plus auto-scaled still frames, then reading them together to produce structured markdown notes.

Works with any Claude surface that supports the skill format — **Claude Code**, **Claude Desktop**, and apps built on the **Claude Agent SDK**.

## Watch the tutorial

<p align="center">
  <a href="https://www.youtube.com/watch?v=U10NUi4FqnU">
    <img src="https://img.youtube.com/vi/U10NUi4FqnU/maxresdefault.jpg" alt="Watch the tutorial on YouTube" width="720" />
  </a>
</p>

> **Note:** the linked tutorial covers the original (v1) version of this skill. The pipeline has since been re-engineered around [bradautomates/claude-video](https://github.com/bradautomates/claude-video) — see "What's new in v2" below.

## What it does

Claude models can see images but can't stream video. This skill fakes video comprehension:

1. Pulls a transcript — YouTube captions first, [Groq Whisper API](https://console.groq.com/) (or OpenAI Whisper) as the fallback for videos with no captions.
2. Extracts still frames at an **auto-scaled** rate based on video duration — short videos get dense coverage, long videos get sparse coverage, hard-capped at 100 frames / 2 fps so token cost stays bounded.
3. Aligns each frame with the spoken text at that timestamp.
4. Claude reads frames + transcript and writes a structured markdown notes file (one-line summary, TL;DR, timestamped timeline, key quotes, visual notes).

Works with YouTube URLs, every other site `yt-dlp` supports (Vimeo, TikTok, X, Twitch clips, Loom, Instagram, etc.), and local video files (`.mp4`, `.mov`, `.mkv`, `.webm`, `.avi`, etc.).

## Use cases

**1.** If you don't understand a tool or concept, you can make Claude watch a video and then plan from there on out; Claude Code will have a much clearer understanding. I saw a custom extension being built for downloading courses and started vibe-coding Claude on that, and it's doing a really, REALLY good job ;)

![Use case 1 — screenshot heavily blurred for privacy.](docs/images/use-case-1-onboarding.jpg)

**2.** Someone was giving me screenshots and walking me through a video on how to do a funnel better. In trying to make Claude learn it, it was much easier for it to just watch the whole video, including the screenshots of the conversations that were being had. This gave it a real, live example of how DM conversations go.

![Use case 2 — screenshot heavily blurred for privacy.](docs/images/use-case-2-tutorial.jpg)

**3.** I'm creating my own Opus Clip Claude Code skill. The difference between the first example that Claude Code made versus the final example it produced is significant, because I was able to show it a demo of what my perfect reel actually looks like.

<p align="center">
  <img src="docs/images/use-case-3-reel-a.jpg" alt="Use case 3a — example reel frame" width="45%" />
  &nbsp;&nbsp;
  <img src="docs/images/use-case-3-reel-b.jpg" alt="Use case 3b — example reel frame" width="45%" />
</p>

**4.** If you like a certain YouTuber's style of editing videos, you can make Claude Code watch two or three of their videos to understand their editing style!

Now, with the new video editing tools like Remotion and Hyperframes, you can actually edit your entire videos through Claude Code in exactly that manner. Since it can see what is on screen and understand how it works along with the timestamps, it can replicate that specific style for you.

## What's new in v2

The pipeline that does the actual download / frame extraction / transcription was rewritten in v2 by replacing our original local extractor with the engine from [bradautomates/claude-video](https://github.com/bradautomates/claude-video) (MIT). Big upgrades:

- **Auto-scaled frame budget** — picks a sensible frame count based on duration (≤30s → ~30 frames; 1–3min → ~60; 3–10min → ~80; >10min → 100 sparse). No more hand-tuning `--interval`.
- **Whisper API fallback (Groq or OpenAI)** instead of the old local `openai-whisper` install — cloud-fast transcription, no Python pip pain, works on every platform without compiled deps.
- **Focused mode (`--start`/`--end`)** — zoom into one section of a long video at higher frame density. Far more useful than a sparse scan when the user asks "what happens at 2:30?".
- **Hard token-cost caps** — 100 frames / 2 fps maximum, so a 60-minute video doesn't accidentally burn 50k tokens of frames.
- **Pure stdlib HTTP** — no `pip install groq` or `pip install openai` needed for the API calls.

### What this skill adds beyond bradautomates/claude-video

- **Slash-command-only invocation guard** — Brad's skill auto-fires on any video URL or "watch this" phrasing, which can accidentally trigger heavy pipelines you didn't ask for. This skill ONLY fires on the literal `/watch-video` slash command. (You can switch back to auto-trigger behavior by editing the `description:` line in [SKILL.md](SKILL.md) — see the note in that file.)
- **Structured persistent notes file** — Brad's skill answers the user's question in chat. This skill always writes a Title / TL;DR / Timeline / Key quotes / Visual notes markdown file you can come back to later, link from other notes, or feed into another agent.
- **Mandatory cleanup workflow** — explicit, opinionated cleanup of the temp work dir after the `.md` is written.
- **Full sampling guidance** — opinionated rules for which frames to read for short / medium / long videos so Claude doesn't brute-force every frame into context.

If you want the pure Brad pipeline without the wrapper, install [bradautomates/claude-video](https://github.com/bradautomates/claude-video) instead — it's excellent.

## Install

Clone into your Claude skills folder:

```bash
# macOS / Linux
git clone https://github.com/Newuxtreme/watch-video-skill.git ~/.claude/skills/watch-video

# Windows (Git Bash / WSL)
git clone https://github.com/Newuxtreme/watch-video-skill.git ~/.claude/skills/watch-video

# Windows (PowerShell / cmd)
git clone https://github.com/Newuxtreme/watch-video-skill.git "$env:USERPROFILE\.claude\skills\watch-video"
```

For Claude Desktop or Claude Agent SDK apps, clone into whatever folder that environment loads skills from.

### Dependencies

- **ffmpeg + ffprobe** — [download](https://ffmpeg.org/download.html), or `brew install ffmpeg` (macOS) / `winget install Gyan.FFmpeg` (Windows) / `apt install ffmpeg` (Linux)
- **yt-dlp** — `winget install yt-dlp.yt-dlp` (Windows), `brew install yt-dlp` (macOS), or `pipx install yt-dlp` (Linux). Must be on `PATH` as a standalone binary.
- **Python 3.9+** on `PATH` as `python` (or `python3` on macOS/Linux). The scripts use `from __future__ import annotations`, so 3.9 is enough.
- **Optional: Whisper API key.** For videos without native captions. Get one at:
  - **Groq** (recommended — cheaper, faster, runs `whisper-large-v3`): [console.groq.com/keys](https://console.groq.com/keys)
  - **OpenAI** (fallback): [platform.openai.com/api-keys](https://platform.openai.com/api-keys)

Run the included setup wizard to scaffold the `.env` and check dependencies:

```bash
python scripts/setup.py
```

It creates `~/.config/watch/.env` with placeholder lines for both keys. Edit the file and paste in whichever key you want to use. Without a key, the skill still works on any video that has captions (i.e. most of YouTube).

## Usage

Once installed, invoke the skill with the slash command:

```
/watch-video https://www.youtube.com/watch?v=...
/watch-video /path/to/local/video.mp4
/watch-video https://youtu.be/abc123 — focus on the 2:00 to 3:00 mark
```

The skill is configured slash-only by default to prevent accidental invocation. To switch to auto-trigger (Claude fires the skill on any "watch this video" / URL request), edit the `description:` line at the top of [SKILL.md](SKILL.md) — there's an inline note explaining how.

## Direct CLI usage

The pipeline can run standalone:

```bash
python scripts/watch.py "<url-or-path>" [flags]
```

Flags:
- `--start T` / `--end T` — focus on a section (`SS`, `MM:SS`, or `HH:MM:SS`)
- `--max-frames N` — cap on frame count (default 80, hard max 100)
- `--resolution W` — frame width in px (default 512)
- `--fps F` — override auto-fps (clamped to 2 fps max)
- `--whisper groq|openai` — force a specific Whisper backend
- `--no-whisper` — disable Whisper fallback (frames-only if no captions)
- `--out-dir DIR` — keep working files somewhere specific (default: tmp)

The script prints a markdown report to stdout listing every extracted frame path, the timestamped transcript, and the working directory.

## Troubleshooting

**Whisper request returns 403:** `whisper.py` sets a custom User-Agent already to clear Cloudflare's default-Python-UA block. If you still see 403s, your key is likely invalid — check it at the provider's console.

**`python` command not found on Windows:** the Microsoft Store stub `python3` doesn't run scripts. Install Python 3.9+ from [python.org](https://www.python.org/downloads/) and use `python` (or `py -3.9`).

**yt-dlp fails on YouTube Shorts / age-gated content:** the bundled `download.py` lets yt-dlp pick its own player client. Members-only and region-locked content may still fail — yt-dlp will surface a clear error.

**No transcript and no Whisper key:** the report will say `Transcript: none available`. Either install a Whisper key (instructions above) or use `--no-whisper` for frames-only output.

**Long videos (>10 min) come back sparse:** that's intentional — frame budget caps at 100. For dense coverage of one section, pass `--start`/`--end` to use focused mode.

## Credits

- Pipeline engine — [bradautomates/claude-video](https://github.com/bradautomates/claude-video) (MIT). Brad wrote the auto-scaled frame extractor, the Whisper API client, and the setup wizard. Vendored under `scripts/`. See [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for the upstream license.
- Wrapper skill (slash-only guard, structured notes template, install + usage docs, original v1 local-Whisper pipeline) — [Newuxtreme](https://github.com/Newuxtreme).

## License

MIT — see [LICENSE](LICENSE). Bundled scripts under `scripts/` retain their original MIT license; see [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
