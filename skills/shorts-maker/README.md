# shorts-maker

Turn one long tutorial or talking-head video into 3–5 clean vertical (1080x1920) YouTube Shorts — no burned-in captions — plus a matching glossy vertical thumbnail for each clip.

## What it does

- Transcribes the source video (Whisper) purely to find good clip boundaries — the transcript is never burned onto the video or delivered
- Picks 3–5 self-contained ~25–60 second segments, each with a hook up front and a payoff by the end
- Renders each as a clean vertical Short with a blurred, scaled-up background fill (no black bars, no captions)
- Generates a matching glossy thumbnail per clip: dark glow background, a circular headshot in a ring light, and a punchy 2-line headline
- Names every file descriptively (`<Prefix>-<Descriptive Title>.mp4`) instead of leaving you with `clip_1.mp4`, `clip_2.mp4`

## Why this exists

Built from doing this exact workflow end-to-end: picking clips that actually stand alone, avoiding the "as you can see here" trap when there's no visual context, and getting a thumbnail that doesn't look like a generic auto-generated frame grab.

## How to use it

Install the skill (see the repo's main [README](../../README.md) for installation), then just name a video:

> "Make shorts from my_interview.mp4"

The skill will check if it can reach your machine to run `ffmpeg`/`whisper` directly; if not, it falls back to an assisted mode where it gives you the exact commands to run yourself.

## Notes

- Needs `ffmpeg`, `whisper` (or a lighter Whisper variant like `faster-whisper`), and Python with `pillow` + `numpy` — the skill installs what's missing where it can.
- The thumbnail script needs a headshot photo of the presenter; a clean cut-out or plain background works best.
- `<Prefix>` in filenames is whatever short tag you want on your clips — your name, your channel name, or nothing at all.
- The `--brand` thumbnail option is optional and just adds a small channel tag at the bottom of the thumbnail.
