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

Claude can see pictures. It cannot watch a video on its own. This skill fixes that.

Here is how it works:

1. It pulls the transcript. If the video has captions, it uses those. If it does not, it sends the audio to Whisper. Groq is the default and OpenAI is the backup.
2. It pulls still frames from the video. Short videos get more frames, long videos get fewer. The cap is 100 frames so the token cost stays low.
3. It matches each frame to the words said at that moment.
4. Claude reads the frames and the transcript together. Then it writes a clean notes file with a one line summary, a TL;DR, a timeline, key quotes, and visual notes.

It works on YouTube and on every other site yt-dlp supports (Vimeo, TikTok, X, Twitch, Loom, Instagram). It also works on local video files (mp4, mov, mkv, webm, avi).

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

I swapped the engine in v2. The old version used a local script I wrote. The new version uses the engine from [bradautomates/claude-video](https://github.com/bradautomates/claude-video) (MIT). It is a big step up.

Here is what changed:

- **Frames are auto scaled.** The script picks how many frames to grab based on the video length. A 30 second video gets about 30 frames. A 10 minute video gets about 80. You do not set the interval by hand anymore.
- **Whisper now runs in the cloud.** The old version used the local `openai-whisper` package, which was slow and a pain to install. The new version sends the audio to the Groq API or the OpenAI API. It is faster and cheaper. It works on any machine that has Python.
- **Focused mode.** You can zoom into one part of a long video. Use `--start` and `--end` to pick a window. The script packs more frames into that window. This is useful when the user asks about one specific moment.
- **Hard caps on cost.** The script will never grab more than 100 frames or run faster than 2 fps. A long video will not blow up your token bill.

## What this skill adds beyond bradautomates/claude-video

This skill takes Brad's engine and adds four things on top.

- **Slash command only.** Brad's skill fires on any video URL or any phrase like "watch this video." That can burn tokens on long videos you did not mean to watch. This skill only fires when you type `/watch-video`. If you want the old auto trigger back, edit the `description:` line in [SKILL.md](SKILL.md). There is a note in that file that walks you through it.
- **Notes file you can keep.** Brad's skill answers the user in chat and that is it. This skill writes a real markdown file with a Title, a TL;DR, a Timeline, Key quotes, and Visual notes. You can read it later, link to it from other notes, or feed it into another agent.
- **Cleanup is built in.** When the script writes the notes file, it wipes the temp folder. No leftover videos or frames cluttering up your disk.
- **Frame sampling guide.** This skill tells Claude which frames to look at, based on how long the video is. Short videos get every frame read. Medium videos get sampled every five seconds. Long videos get sampled every ten seconds. Claude does not brute force every single frame into the context window.

If you want the pure Brad pipeline with no wrapper on top, install [bradautomates/claude-video](https://github.com/bradautomates/claude-video). It is excellent.

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
