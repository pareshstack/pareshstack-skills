---
name: youtube-shorts
description: "Turn a long tutorial/talking video into several captioned vertical YouTube Shorts. Use when the user names a video to make Shorts/clips from, e.g. 'make shorts from sample2.mp4'."
---

# YouTube Shorts maker

Turn one long tutorial/talking-head video into 3–5 captioned, vertical (1080x1920) YouTube Shorts. The user gives a video filename; produce finished `.mp4` Shorts (or, when their computer can't be reached, finished caption files plus ready-to-run commands).

> This skill burns captions onto the video. If you'd rather have clean clips (no captions) plus a separate matching thumbnail image per clip, use a "clean Shorts + thumbnails" variant instead — the two serve different styles.

## Where things live

The user keeps source videos and this kit in one folder, by default `~/Downloads/shorts`. If the user names a different folder, use it. The folder should contain (this skill recreates any that are missing) `make_short.sh` and `clip_captions.py` — their full contents are at the end of this file.

## Step 1 — Can you reach their machine?

Call `get_device_info` (retry once if it reports not connected).
- **Reachable** → do the **Auto-run path** (Step 2), running every command on their machine with `device_bash`.
- **Not reachable** → do the **Assisted path** (Step 3). Tell the user plainly you can't reach their computer right now and are switching to assisted mode.

Before rendering, confirm the source video exists in the folder (`ls`). If not, ask for the exact filename.

## Step 2 — Auto-run path (machine reachable)

All commands run via `device_bash`, from inside the shorts folder.

1. **Ensure the kit exists.** Check for `make_short.sh` and `clip_captions.py`; recreate any missing one from the contents at the end of this file, then `chmod +x make_short.sh`.
2. **Ensure tools exist.** Check `ffmpeg -version` and `whisper --help`. If ffmpeg is missing, install it with the platform's package manager (`brew install ffmpeg` on macOS, your distro's `ffmpeg` package on Linux). If whisper is missing: `pip3 install --break-system-packages openai-whisper` (or `brew install openai-whisper` on macOS). If the required package manager itself is missing, stop and give the user the install line rather than trying to install it silently.
3. **Transcribe.** `whisper "VIDEO" --model small --output_format srt --output_dir .` — this writes `VIDEO.srt` (same basename). The first run downloads the model (~500MB), which is normal.
4. **Read `VIDEO.srt` and pick clips** using the heuristics in Step 4.
5. **Generate caption files.** For each chosen clip, run `python3 clip_captions.py VIDEO.srt START_SEC END_SEC clipN_slug.srt` (START/END in seconds, decimals allowed).
6. **Render.** For each clip, run `./make_short.sh VIDEO 00:MM:SS.s 00:MM:SS.s shortN.mp4 clipN_slug.srt`.
7. **Report.** List the finished `shortN.mp4` files with their one-line topic, and for each Short give a title, a 1–2 sentence description, and 3–5 hashtags written to match what that clip actually says. Then remind the user to eyeball each Short before posting (auto-picked clips are usually ~80% right).

## Step 3 — Assisted path (machine not reachable)

1. Ask the user to transcribe with `whisper "VIDEO" --model small --output_format srt --output_dir transcripts` and paste the resulting `.srt`, OR paste a transcript they already have.
2. Pick clips using Step 4.
3. Recreate the caption files yourself in the workspace: write the pasted transcript to `master.srt`, run `clip_captions.py` for each clip (write the helper locally first), and deliver the resulting `clipN_slug.srt` files.
4. Give the user the exact `./make_short.sh VIDEO ...` command for each clip, plus titles/descriptions/hashtags as in Step 2.7.

## Step 4 — How to pick clips

Aim for **3–5 self-contained segments**, each **~25–60 seconds**. A good Short clip:
- Has a **hook** (a question, a claim, "here's the trick") near the start and a **payoff** by the end.
- Is understandable **without the on-screen visuals** — skip segments that lean on "as you can see here / on this slide," since the vertical frame won't show them clearly.
- **Starts and ends on complete sentences.** Set the START at the first word of the opening sentence and the END a hair before the next sentence begins, so no fragment of the next topic bleeds in. (Check the last caption line after generating — if a sliver of the next sentence appears, pull the END back by ~0.1–0.3s and regenerate.)
- Prefer segments with a **counterintuitive or genuinely useful takeaway** over generic intros/outros.

Name each caption file `clipN_slug.srt` (e.g. `clip4_notspecified.srt`) so the topic is obvious.

## Reference: make_short.sh

```bash
#!/bin/bash
# make_short.sh — turn one clip from a video into a vertical YouTube Short
# Usage: ./make_short.sh INPUT START END OUTPUT [SUBTITLES.srt]
# Example: ./make_short.sh sample.mp4 00:01:30 00:02:05 short1.mp4 clip1.srt
set -e
INPUT="$1"; START="$2"; END="$3"; OUTPUT="$4"; SUBS="$5"
if [ -z "$OUTPUT" ]; then
  echo "Usage: ./make_short.sh INPUT START END OUTPUT [SUBTITLES.srt]"
  exit 1
fi
if [ ! -f "$INPUT" ]; then
  echo "Can't find the input file: $INPUT"; exit 1
fi
echo "Cutting $INPUT from $START to $END ..."
if [ -n "$SUBS" ] && [ -f "$SUBS" ]; then
  ffmpeg -y -ss "$START" -to "$END" -i "$INPUT" -filter_complex \
  "[0:v]split=2[bg][fg]; \
   [bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=25[bgblur]; \
   [fg]scale=1080:-1[fgs]; \
   [bgblur][fgs]overlay=(W-w)/2:(H-h)/2[bv]; \
   [bv]subtitles='$SUBS':force_style='Fontname=Arial,Fontsize=16,Bold=1,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=2,Alignment=2,MarginV=120'[v]" \
  -map "[v]" -map 0:a -c:v libx264 -preset medium -crf 20 -c:a aac -b:a 192k "$OUTPUT"
else
  ffmpeg -y -ss "$START" -to "$END" -i "$INPUT" -filter_complex \
  "[0:v]split=2[bg][fg]; \
   [bg]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=25[bgblur]; \
   [fg]scale=1080:-1[fgs]; \
   [bgblur][fgs]overlay=(W-w)/2:(H-h)/2[v]" \
  -map "[v]" -map 0:a -c:v libx264 -preset medium -crf 20 -c:a aac -b:a 192k "$OUTPUT"
fi
echo "Done. Your Short is saved as: $OUTPUT"
```

## Reference: clip_captions.py

```python
#!/usr/bin/env python3
"""clip_captions.py — extract a time window from a master .srt and re-time it to start at 0.
Usage: python3 clip_captions.py master.srt START_SECONDS END_SECONDS output.srt"""
import re, sys
def to_seconds(ts):
    h, m, rest = ts.split(":"); s, ms = rest.split(",")
    return int(h)*3600 + int(m)*60 + int(s) + int(ms)/1000
def to_ts(sec):
    if sec < 0: sec = 0
    ms = round((sec - int(sec)) * 1000); sec = int(sec)
    h, m, s = sec//3600, (sec%3600)//60, sec%60
    return f"{h:02d}:{m:02d}:{s:02d},{ms:03d}"
master, start, end, out = sys.argv[1], float(sys.argv[2]), float(sys.argv[3]), sys.argv[4]
text = open(master, encoding="utf-8").read()
blocks = re.split(r"\n\s*\n", text.strip())
line_re = re.compile(r"(\d{2}:\d{2}:\d{2},\d{3})\s*-->\s*(\d{2}:\d{2}:\d{2},\d{3})")
out_blocks, n = [], 1
for b in blocks:
    lines = b.strip().split("\n")
    time_line = next((l for l in lines if line_re.search(l)), None)
    if not time_line: continue
    mobj = line_re.search(time_line)
    st, en = to_seconds(mobj.group(1)), to_seconds(mobj.group(2))
    if en <= start or st >= end: continue
    new_st = max(st, start) - start; new_en = min(en, end) - start
    caption = "\n".join(l for l in lines if not line_re.search(l) and not l.strip().isdigit())
    out_blocks.append(f"{n}\n{to_ts(new_st)} --> {to_ts(new_en)}\n{caption}")
    n += 1
open(out, "w", encoding="utf-8").write("\n\n".join(out_blocks) + "\n")
print(f"Wrote {out} with {n-1} caption lines.")
```
