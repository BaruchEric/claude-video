# claude-video

Gives Claude a video input: the `/watch` skill downloads any URL or local file with yt-dlp, extracts auto-scaled frames with ffmpeg, pulls a timestamped transcript (captions, Whisper fallback), and hands frames + transcript to Claude so it can actually see and hear the video.

## TL;DR

- **What:** A `/watch` skill that lets Claude watch a video and answer questions grounded in what's actually on screen and in the audio — not the title or a partial transcript.
- **How:** `yt-dlp` downloads (URL) or `ffprobe` reads (local file) → `ffmpeg` extracts frames as JPEGs at a duration-aware fps → transcript from native captions, falling back to the Whisper API → Claude `Read`s every frame and aligns it to the timestamped transcript.
- **Stack:** Python 3 stdlib scripts shelling out to `yt-dlp` + `ffmpeg`/`ffprobe`; Whisper transcription via Groq (`whisper-large-v3`, preferred) or OpenAI (`whisper-1`). Ships as a Claude Code plugin, a claude.ai `watch.skill` bundle, and a Codex/manual skill.
- **Run:** In Claude Code, `/plugin marketplace add bradautomates/claude-video` then `/plugin install watch@claude-video`, and ask `/watch <url-or-path> <question>`. Direct CLI: `python3 scripts/watch.py "<url-or-path>" [flags]`.

---

> **Status:** This is a personal fork of [bradautomates/claude-video](https://github.com/bradautomates/claude-video) — all install paths and package attribution point at the upstream marketplace, which is where `watch@claude-video` actually resolves. No fork-specific code changes; the skill works as published upstream.

## Overview

Claude can read a webpage, run a script, browse a repo. What it can't do out of the box is *watch a video* — paste a YouTube link and it guesses from the title or pulls a transcript that misses everything on screen.

`/watch` closes that gap. You paste a URL or local path, ask a question, and the skill downloads the video, extracts frames at an auto-scaled rate, pulls a timestamped transcript (free captions when available, Whisper API as fallback), and prints frame paths. Claude `Read`s each frame as an image and answers grounded in what it saw and heard.

Common uses (all from actual skill behavior):

- **Analyze someone else's content** — `/watch <viral-video> what hook did they open with?`
- **Diagnose a bug from a screen recording** — `/watch bug-repro.mov what's going wrong?`
- **Summarize a video** — `/watch <long-thing> summarize this`

## Tech stack

- **Language:** Python 3, standard library only (no pip dependencies in the skill scripts).
- **External binaries:** `yt-dlp` (download + native caption extraction), `ffmpeg` / `ffprobe` (frame extraction, metadata probing, audio extraction for Whisper).
- **Transcription APIs (optional fallback):** Groq `whisper-large-v3` (preferred — cheaper, faster) or OpenAI `whisper-1`. Keys live in `~/.config/watch/.env`.
- **Distribution surfaces:** Claude Code plugin (`.claude-plugin/`), claude.ai `watch.skill` bundle, Codex / manual git-clone skill.

## Getting started

### Install

| Surface | Install |
|---------|---------|
| **Claude Code** | `/plugin marketplace add bradautomates/claude-video` then `/plugin install watch@claude-video` |
| **claude.ai** (web) | [Download `watch.skill`](https://github.com/bradautomates/claude-video/releases/latest) → Settings → Capabilities → Skills → `+` |
| **Codex** | `git clone https://github.com/bradautomates/claude-video.git ~/.codex/skills/watch` |
| **Manual / dev** | `git clone https://github.com/bradautomates/claude-video.git ~/.claude/skills/watch` |

On claude.ai, enable "Code execution and file creation" first — the skill shells out to `ffmpeg` and `yt-dlp`. Update the Claude Code plugin later with `/plugin update watch@claude-video`.

### First run

On the first `/watch` call the skill runs `scripts/setup.py --check` (a sub-100ms lookup, silent on success). If `ffmpeg` / `yt-dlp` are missing or no Whisper key is set, the installer walks you through it:

- **macOS** — auto-runs `brew install ffmpeg yt-dlp`.
- **Linux / Windows** — prints the exact `apt` / `dnf` / `pipx` / `winget` / `pip` commands.
- **API key** — scaffolds `~/.config/watch/.env` (mode `0600`) with commented `GROQ_API_KEY` (preferred) and `OPENAI_API_KEY` placeholders, and writes a `SETUP_COMPLETE` marker once deps + a key are in place.

### Bring your own keys

Captions cover most public videos for free; the Whisper fallback only fires when a video genuinely has no caption track (local files, some TikToks/Vimeos, caption-less uploads).

| Capability | What you need | Cost |
|------------|---------------|------|
| Download + native captions | `yt-dlp` + `ffmpeg` | Free |
| Whisper fallback (preferred) | [Groq API key](https://console.groq.com/keys) — `whisper-large-v3` | Cheap, fast |
| Whisper fallback (alt) | [OpenAI API key](https://platform.openai.com/api-keys) — `whisper-1` | Standard pricing |
| Disable Whisper entirely | `--no-whisper` | Free, frames-only when no captions |

### Usage

```
/watch https://youtu.be/dQw4w9WgXcQ what happens at the 30 second mark?
/watch https://www.tiktok.com/@user/video/123 summarize this
/watch ~/Movies/screen-recording.mp4 when does the UI break?
```

Focus on a specific section for a denser frame budget at lower token cost:

```
/watch https://youtu.be/abc --start 2:15 --end 2:45
/watch video.mp4 --start 50 --end 60
/watch "$URL" --start 1:12:00            # from 1h12m to end
```

## Scripts & flags

The entry point is `scripts/watch.py`, runnable directly: `python3 scripts/watch.py "<url-or-path>" [flags]`.

| Flag | Effect |
|------|--------|
| `--start T` / `--end T` | Focus on a section (`SS`, `MM:SS`, or `HH:MM:SS`). Switches to denser focused-mode fps budgets; transcript is filtered to the same range. |
| `--max-frames N` | Lower the frame cap for a tighter token budget (default 80, hard max 100). |
| `--resolution W` | Frame width in px (default 512; bump to 1024 when Claude needs to read on-screen text). |
| `--fps F` | Override the auto-fps calculation (still clamped to 2 fps max). |
| `--whisper groq\|openai` | Force a specific Whisper backend (default: prefer Groq, fall back to OpenAI). |
| `--no-whisper` | Disable the Whisper fallback entirely — frames-only if no captions. |
| `--out-dir DIR` | Keep working files somewhere specific (default: an auto-generated tmp dir). |

## How it works

1. **You paste a video and a question.** URL (anything yt-dlp supports) or a local path (`.mp4`, `.mov`, `.mkv`, `.webm`, …).
2. **`yt-dlp` downloads it** (URLs → temp workdir) or **`ffprobe` reads it in place** (local files).
3. **`ffmpeg` extracts frames at an auto-scaled rate.** The frame budget is duration-aware (≤30s ≈ 30 frames, 30–60s ≈ 40, 1–3min ≈ 60, 3–10min ≈ 80, longer = 100 sparsely). Hard ceilings: 2 fps, 100 frames. JPEGs at 512px wide by default.
4. **Transcript comes from one of two places.** First `yt-dlp` native captions (free); fallback extracts a mono 16 kHz audio clip and ships it to Whisper (Groq, then OpenAI).
5. **Frames + transcript are handed to Claude.** The script prints frame paths with `t=MM:SS` markers and the transcript with timestamps; Claude `Read`s each frame in parallel.
6. **Claude answers** grounded in what's on screen and in the audio, then cleans up the working directory if you're not asking follow-ups.

### Frame budget

| Duration | Default frame budget | What you get |
|----------|---------------------|--------------|
| ≤30 s | ~30 frames | Dense — basically every key moment |
| 30 s – 1 min | ~40 frames | Still dense |
| 1 – 3 min | ~60 frames | Comfortable |
| 3 – 10 min | ~80 frames | Sparse but workable |
| > 10 min | 100 frames | "Sparse scan" warning — re-run focused |

## Limits

- **Best accuracy: under 10 minutes.** Past that the script prints a "sparse scan" warning — re-run focused with `--start`/`--end`.
- **Hard caps: 2 fps, 100 frames** — enforced even when auto-fps math implies higher.
- **Whisper upload limit: 25 MB** (~50 min of mono 16 kHz audio). Longer videos need captions or a smaller window.
- **No private platforms** — public URLs and local files only; the skill never logs in.

## Project structure

```
.
├── SKILL.md                 # skill contract — loaded by all three surfaces
├── scripts/
│   ├── watch.py             # entry point — orchestrates download → frames → transcript
│   ├── download.py          # yt-dlp wrapper
│   ├── frames.py            # ffmpeg frame extraction + auto-fps logic
│   ├── transcribe.py        # VTT parsing + dedupe + Whisper orchestration
│   ├── whisper.py           # Groq / OpenAI clients (pure stdlib)
│   ├── setup.py             # preflight + installer
│   └── build-skill.sh       # build dist/watch.skill for claude.ai upload
├── hooks/                   # SessionStart status hook (Claude Code only)
├── commands/watch.md        # makes /watch callable via the Claude Code plugin
├── .claude-plugin/          # plugin.json + marketplace.json (Claude Code)
├── .codex-plugin/           # codex packaging
└── .github/workflows/       # release.yml — auto-builds watch.skill on tag push
```

## Develop & release

```bash
bash scripts/build-skill.sh      # → dist/watch.skill (claude.ai upload bundle)
```

Releasing: tag `vX.Y.Z`, push the tag. The workflow builds `dist/watch.skill` and attaches it to the GitHub release. See [CHANGELOG.md](CHANGELOG.md) for version history.

## License

MIT © Bradley Bonanno. Built on `yt-dlp`, `ffmpeg`, and Claude's multimodal `Read` tool. Whisper transcription via [Groq](https://groq.com) or [OpenAI](https://openai.com).

---

Upstream: [github.com/bradautomates/claude-video](https://github.com/bradautomates/claude-video) · [LICENSE](LICENSE)
