# watch-video skill

A Claude skill that lets Claude "watch" videos by extracting a time-synced transcript plus still frames, then reading them together to produce structured notes.

Works with any Claude surface that supports the skill format — **Claude Code**, **Claude Desktop**, and apps built on the **Claude Agent SDK**.

## What it does

Claude models can see images but can't stream video. This skill fakes video comprehension:

1. Pulls a transcript — YouTube captions first, [Whisper](https://github.com/openai/whisper) as a local fallback.
2. Extracts still frames at a configurable interval via ffmpeg.
3. Aligns each frame with the sentence being spoken at that timestamp.
4. Claude reads frames + transcript and writes a markdown notes file (one-line summary, TL;DR, timestamped timeline, key quotes, visual notes).

Works with YouTube URLs and local video files.

## Use cases

**1. Onboarding Claude to a new project or tool.** Before starting work on something Claude hasn't seen before — an unfamiliar codebase, a niche CLI tool, a custom extension — point it at one or two walkthrough videos of the thing in action. Claude sees what the tool does, what the output looks like, and what the workflow is, then plans from a much clearer starting point instead of guessing from docs alone.

![Use case 1 — onboarding Claude to an unfamiliar tool by showing it a walkthrough video of the UI and output. Screenshot heavily blurred for privacy.](docs/images/use-case-1-onboarding.jpg)

**2. Learning from coaching / tutorial videos.** When someone walks you through a workflow on video — screen recording, narration, screenshots of real conversations or real data — Claude can absorb the whole thing end-to-end in one pass. Screenshots, spoken reasoning, and the transitions between steps all land in the same context. Much tighter than pasting a transcript plus a few images separately.

![Use case 2 — Claude watching a coaching video that includes real DM conversation screenshots, absorbing the workflow and real examples end-to-end. Screenshot heavily blurred for privacy.](docs/images/use-case-2-tutorial.jpg)

**3. Show, don't tell — for content generation.** Building a content-generation tool (e.g. a short-form clip generator, a thumbnail skill) and the first attempt doesn't match your vision? Show Claude a single "this is what the perfect output looks like" video. It sees the pacing, cuts, text placement, music cues, everything — and the next iteration tracks much closer to the target.

<p align="center">
  <img src="docs/images/use-case-3-reel-a.jpg" alt="Use case 3a — example reel frame: mic close-up with bold white + accent green on-screen text" width="45%" />
  &nbsp;&nbsp;
  <img src="docs/images/use-case-3-reel-b.jpg" alt="Use case 3b — example reel frame: tight close-up with all-caps text overlay" width="45%" />
</p>

**4. Style cloning from your favorite creators.** Have Claude watch two or three videos from a YouTuber/editor whose style you want to match. It picks up the cadence, cut timing, overlay patterns, and on-screen text style. Paired with a programmatic video editor like [Remotion](https://www.remotion.dev/) or Hyperframes, Claude can then replicate the style directly in code — since it has the timestamps and what was on screen at each beat, it can reproduce the pattern.

## Install

Clone into your Claude skills folder:

```bash
# macOS / Linux
git clone https://github.com/Newuxtreme/watch-video-skill.git ~/.claude/skills/watch-video

# Windows
git clone https://github.com/Newuxtreme/watch-video-skill.git %USERPROFILE%\.claude\skills\watch-video
```

For Claude Desktop or Claude Agent SDK apps, clone into whatever folder that environment loads skills from.

### Dependencies

- **ffmpeg** — [download](https://ffmpeg.org/download.html) or `brew install ffmpeg` / `choco install ffmpeg`
- **yt-dlp** — `python -m pip install --user yt-dlp`
- **openai-whisper** (optional, only needed for videos without captions) — `python -m pip install --user openai-whisper`

## Usage

Once installed, just ask Claude to watch a video:

```
Watch this: https://www.youtube.com/watch?v=...
Take notes on this reel: https://...
Summarize this video: /path/to/local/video.mp4
```

Claude invokes the skill automatically when the request matches.

## Direct CLI usage

The extractor can run standalone:

```bash
python scripts/extract_video.py "<url-or-path>" --output-dir ./out --interval 1.0
```

Flags:
- `--interval N` — seconds between frames (default 1.0)
- `--whisper-model MODEL` — `tiny` / `base` / `small` / `medium` / `large` (default `base`)
- `--no-whisper` — skip transcription fallback

Outputs a `manifest.json` with frame paths, timestamps, and aligned transcript segments.

## Troubleshooting

**Windows + broken Python pip:** On some Windows setups, Python 3.12 and 3.13 ship with pip installs that fail on yt-dlp / whisper. If your default Python chokes, install Python 3.9 via [python.org](https://www.python.org/downloads/) and invoke explicitly: `py -3.9 -m pip install --user yt-dlp`.

**yt-dlp fails on YouTube Shorts / age-gated content:** The script uses the `android` player client as a fallback, which works for most cases. Members-only and region-locked content may still fail — yt-dlp will surface a clear error.

**"No module named whisper":** Either install whisper (`python -m pip install --user openai-whisper`) or pass `--no-whisper` to get frames-only output.

## License

MIT — see [LICENSE](LICENSE).
