# youtube-shorts

Turn one long tutorial or talking-head video into 3–5 captioned vertical (1080x1920) YouTube Shorts.

## What it does

- Transcribes the source video (Whisper) to find good clip boundaries and generate captions
- Picks 3–5 self-contained ~25–60 second segments, each with a hook up front and a payoff by the end
- Re-times a slice of the transcript to each clip and burns it in as styled captions
- Renders each as a vertical Short with a blurred, scaled-up background fill (no black bars)
- Writes a title, description, and hashtags per clip based on what it actually says

## Why this exists

Built from doing this exact workflow end-to-end: picking clips that stand alone without needing on-screen visuals, keeping caption timing tight so sentences don't bleed into the next clip, and getting a vertical crop that doesn't look stretched or letterboxed.

## How to use it

Install the skill (see the repo's main [README](../../README.md) for installation), then just name a video:

> "Make shorts from my_interview.mp4"

The skill checks if it can reach your machine to run `ffmpeg`/`whisper` directly; if not, it falls back to an assisted mode where it gives you the exact commands and caption files to run yourself.

## Notes

- Needs `ffmpeg`, `whisper` (or a lighter Whisper variant like `faster-whisper`), and Python — the skill installs what's missing where it can.
- Captions are burned into the video. If you want clean clips with no captions plus a separate thumbnail image per clip instead, look for a "clean Shorts + thumbnails" style skill — this one and that one serve different styles, not the same job twice.
