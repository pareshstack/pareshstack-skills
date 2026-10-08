---
name: shorts-maker
category: Video and Shorts
description: "Turn a long tutorial/talking video into several clean vertical YouTube Shorts (1080x1920, no burned-in captions) AND a matching glossy vertical thumbnail for each clip. Names files descriptively, e.g. 'Clip-Explaining the thing nobody gets.mp4'. Use whenever the user names a video to make Shorts/clips from, e.g. 'make shorts from sample2.mp4'."
---

# Shorts maker (clean video + thumbnails)

Turn one long tutorial/talking-head video into 3–5 **clean** vertical (1080x1920) YouTube Shorts — **no burned-in captions, no subtitle files** — and a matching **glossy vertical thumbnail** for each clip. A transcript is made only to choose the clips; it is never rendered onto the video or delivered.

Deliverables per run, in the user's shorts folder:
- `<Prefix>-<Descriptive Title>.mp4` — the clean Short, one per clip.
- `thumbnails/<Prefix>-<Descriptive Title>.png` — a 1080x1920 thumbnail matching each Short.

`<Prefix>` is whatever short tag the user wants on their clips (their name, channel name, or nothing at all — just drop the dash). Pick one with the user up front if they haven't said; reuse it for every clip in the run.

## File naming (IMPORTANT)

Name each Short for what the clip is about: `<Prefix>-<Descriptive Title>.mp4`, where the title is a short human-readable summary (e.g. `<Prefix>-Explaining the thing nobody gets.mp4`, `<Prefix>-The trick nobody mentions.mp4`). Filesystem-safe: no `/`, `\`, or `:`; spaces and apostrophes are fine. Each thumbnail uses the **same base name** with `.png`, inside a `thumbnails/` subfolder.

## Where things live

Default folder `~/Downloads/shorts` (use another if the user names one). The skill recreates any missing kit file: `make_short.sh`, `make_thumbnail.py`, and a `fonts/` folder with `Anton.ttf` + `Montserrat-Black.ttf`. Full contents / fetch commands are at the end of this file.

## Step 1 — Can you reach their machine?

Call `get_device_info` (retry once if it reports not connected).
- **Reachable** → **Auto-run path** (Step 2), every command via `device_bash`.
- **Not reachable** → **Assisted path** (Step 3); tell the user plainly you're switching to assisted mode.

Confirm the source video exists in the folder (`ls`) before rendering.

## Step 2 — Auto-run path (machine reachable)

All commands run via `device_bash`, from inside the shorts folder.

1. **Ensure the kit.** Recreate `make_short.sh` and `make_thumbnail.py` if missing (`chmod +x make_short.sh`). Ensure `fonts/Anton.ttf` and `fonts/Montserrat-Black.ttf` exist; if not, fetch them (Step 2.2).
2. **Ensure tools + fonts.** Check `ffmpeg -version`, `whisper --help`, and `python3 -c "import PIL, numpy"`. Install what's missing: ffmpeg (`brew install ffmpeg` on macOS, your package manager's `ffmpeg` on Linux); whisper `pip3 install --break-system-packages openai-whisper` (or `brew install openai-whisper` on macOS); Python libs `pip3 install --break-system-packages pillow numpy`. Fetch fonts once from the Google Fonts GitHub mirror (raw.githubusercontent.com is usually reachable):
   ```bash
   mkdir -p fonts
   B=https://raw.githubusercontent.com/google/fonts/main
   curl -sSL -o fonts/Anton.ttf "$B/ofl/anton/Anton-Regular.ttf"
   curl -sSL -o fonts/Montserrat-Black.ttf "$B/ofl/montserrat/Montserrat%5Bwght%5D.ttf"
   ```
   (In a restricted Linux shell without Homebrew, install whisper with `pip3 install --break-system-packages faster-whisper` or `pywhispercpp`; if the whisper model host is blocked, fetch a model via the user's browser or fall back to the Assisted path. Whisper only picks clips, so accuracy matters less.)
3. **Transcribe (clip selection only).** `whisper "VIDEO" --model small --output_format srt --output_dir .` → writes `VIDEO.srt`. First run downloads the model (~500MB), which is normal.
4. **Read `VIDEO.srt` and pick clips** using Step 4. Note precise START/END seconds.
5. **Render clean Shorts.** For each clip: `./make_short.sh VIDEO 00:MM:SS.s 00:MM:SS.s "<Prefix>-<Descriptive Title>.mp4"` — no subtitle argument, so nothing is burned in. Quote the output name.
6. **Get a headshot photo for the thumbnails.** Ask the user for a photo of the presenter (a clean cut-out or plain background is best; higher-res looks sharper, but a low-res one still works inside the glow ring). If they've provided or used one before, reuse it. Note its path as PHOTO.
7. **Generate a thumbnail per clip.** `mkdir -p thumbnails`. For each clip, write a short punchy hook (Step 5) and run:
   ```bash
   python3 make_thumbnail.py --photo "PHOTO" \
     --out "thumbnails/<Prefix>-<Descriptive Title>.png" \
     --kicker "WRITE BETTER" --l1 "FIX YOUR" --l2 "WRITING" \
     --brand "<Your Channel Name>"
   ```
   (`--brand` is optional; omit for no channel tag. Line 1 renders white, line 2 yellow.)
8. **Clean up.** Don't leave `VIDEO.srt` or any `.srt` in the folder — the deliverable is clean video + thumbnails. Keep transcripts in scratch space; if an `.srt` landed in the shorts folder, ask before removing (deletes there need permission).
9. **Report.** List each `<Prefix>-<Title>.mp4` with its matching `thumbnails/<Prefix>-<Title>.png`, plus a title, 1–2 sentence description, and 3–5 hashtags per Short. Remind the user to eyeball each Short + thumbnail before posting (auto-picked clips are ~80% right), and offer easy thumbnail tweaks (hook wording, accent color, face size, a higher-res photo).

## Step 3 — Assisted path (machine not reachable)

1. Ask the user to transcribe (`whisper "VIDEO" --model small --output_format srt --output_dir transcripts`) and paste the `.srt`, or paste a transcript they have.
2. Pick clips (Step 4) and write a hook per clip (Step 5).
3. Give the user, per clip, the exact `./make_short.sh ...` command (no subtitle arg) and the `python3 make_thumbnail.py ...` command, plus a note to place `make_thumbnail.py`, `make_short.sh` and `fonts/` in the folder. Provide titles/descriptions/hashtags as in Step 2.9.

## Step 4 — How to pick clips

Aim for **3–5 self-contained segments**, each **~25–60 seconds**. A good clip has a **hook** near the start and a **payoff** by the end; is understandable **without on-screen visuals** (skip "as you can see here"); **starts and ends on complete sentences** (START at the first word, END a hair before the next sentence so nothing bleeds in); and carries a **counterintuitive or genuinely useful takeaway**.

## Step 5 — Writing the thumbnail hook

Per clip, write a 2-line headline plus a small kicker, all derived from what the clip actually says:
- **Kicker** (`--kicker`): a 2–3 word benefit label, e.g. `WRITE BETTER`, `BEAT THE BLOCK`, `THE ONE PHRASE`.
- **Line 1** (`--l1`, white): the setup, e.g. `FIX YOUR`, `20 IDEAS,`, `STOP AI`.
- **Line 2** (`--l2`, yellow): the punchy payoff, e.g. `WRITING`, `NOT 5`, `GUESSING`.
Keep each line short (Anton is very wide; ~8–9 characters per line reads best at this size). Punchy and curiosity-driving beats descriptive. Example set: `MAKE IT / CLICK`, `PLAN THE / CHAOS`, `FIX YOUR / WRITING`, `20 IDEAS, / NOT 5`, `STOP AI / GUESSING`.

## Reference: make_short.sh

```bash
#!/bin/bash
# make_short.sh — turn one clip into a clean vertical YouTube Short
# Usage: ./make_short.sh INPUT START END OUTPUT [SUBTITLES.srt]
# This skill calls it WITHOUT the subtitles argument (clean video, no burned-in captions).
set -e
INPUT="$1"; START="$2"; END="$3"; OUTPUT="$4"; SUBS="$5"
if [ -z "$OUTPUT" ]; then echo "Usage: ./make_short.sh INPUT START END OUTPUT [SUBTITLES.srt]"; exit 1; fi
if [ ! -f "$INPUT" ]; then echo "Can't find the input file: $INPUT"; exit 1; fi
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

## Reference: make_thumbnail.py

```python
#!/usr/bin/env python3
"""make_thumbnail.py - one glossy vertical (1080x1920) YouTube Short thumbnail.
Usage:
  python3 make_thumbnail.py --photo PHOTO --out OUT.png \
      --kicker "WRITE BETTER" --l1 "FIX YOUR" --l2 "WRITING" [--brand "Your Channel Name"] [--fonts fonts]
Style: dark background + warm orange glow, a sunburst mark top-left, the headshot
in a glowing circular ring, headline line 1 white / line 2 yellow.
Needs fonts/Anton.ttf and fonts/Montserrat-Black.ttf.
"""
import os, math, argparse, numpy as np
from PIL import Image, ImageDraw, ImageFont, ImageFilter

W, H = 1080, 1920
WHITE=(255,255,255); YELLOW=(255,201,38); ORANGE=(232,120,64); MARK=(222,110,64)

def _f(fonts,name,size): return ImageFont.truetype(os.path.join(fonts,name),size)
def _mont(fonts,size,wght=800):
    f=ImageFont.truetype(os.path.join(fonts,"Montserrat-Black.ttf"),size)
    try: f.set_variation_by_axes([wght])
    except Exception: pass
    return f

def make_bg():
    yy=np.linspace(0,1,H)[:,None]
    grad=(np.array([18,20,28])*(1-yy)+np.array([7,7,10])*yy)
    img=np.repeat(grad[:,None,:],W,axis=1)
    xs,ys=np.meshgrid(np.arange(W),np.arange(H))
    def glow(cx,cy,rad,col,strg):
        g=np.clip(1-np.sqrt((xs-cx)**2+(ys-cy)**2)/rad,0,1)**2
        for i in range(3): img[:,:,i]+=col[i]*g*strg
    glow(700,1230,760,(236,126,60),0.55); glow(300,470,640,(70,90,150),0.18); glow(540,1500,900,(150,70,40),0.12)
    im=Image.fromarray(np.clip(img,0,255).astype('uint8'),'RGB')
    vig=Image.new('L',(W,H),0); ImageDraw.Draw(vig).ellipse([-260,-360,W+260,H+360],fill=255)
    vig=vig.filter(ImageFilter.GaussianBlur(240))
    im=Image.composite(im,Image.new('RGB',(W,H),(0,0,0)),vig)
    return im.convert('RGBA')

def brand_mark(size,color=MARK):
    S=size*4; im=Image.new('RGBA',(S,S),(0,0,0,0)); d=ImageDraw.Draw(im)
    cx=cy=S/2; n=12; inner=S*0.05; midw=S*0.052; outer=S*0.46
    for k in range(n):
        a=math.pi*2*k/n-math.pi/2; ca,sa=math.cos(a),math.sin(a)
        pa=a+math.pi/2; cpa,spa=math.cos(pa),math.sin(pa)
        tip=(cx+ca*outer,cy+sa*outer); b=(cx+ca*inner,cy+sa*inner)
        d.polygon([(b[0]+cpa*midw*0.35,b[1]+spa*midw*0.35),(cx+ca*outer*0.5+cpa*midw,cy+sa*outer*0.5+spa*midw),tip,
                   (cx+ca*outer*0.5-cpa*midw,cy+sa*outer*0.5-spa*midw),(b[0]-cpa*midw*0.35,b[1]-spa*midw*0.35)],fill=color+(255,))
    d.ellipse([cx-inner*1.2,cy-inner*1.2,cx+inner*1.2,cy+inner*1.2],fill=color+(255,))
    return im.resize((size,size),Image.LANCZOS)

def make_avatar(photo,diam):
    im=Image.open(photo).convert('RGB'); im=im.resize((im.width*4,im.height*4),Image.LANCZOS)
    im=im.filter(ImageFilter.UnsharpMask(radius=3,percent=90,threshold=2))
    s=min(im.size); l=(im.width-s)//2; t=int((im.height-s)*0.32)
    im=im.crop((l,t,l+s,t+s)).resize((diam,diam),Image.LANCZOS)
    mask=Image.new('L',(diam,diam),0); ImageDraw.Draw(mask).ellipse([0,0,diam,diam],fill=255)
    av=Image.new('RGBA',(diam,diam),(0,0,0,0)); av.paste(im,(0,0),mask); return av

def ring(diam,width,color):
    S=diam+width*2; im=Image.new('RGBA',(S,S),(0,0,0,0))
    ImageDraw.Draw(im).ellipse([width/2,width/2,S-width/2,S-width/2],outline=color+(255,),width=width); return im

def paste_center(base,img,cx,cy): base.alpha_composite(img,(int(cx-img.width/2),int(cy-img.height/2)))
def tw(f,s): b=f.getbbox(s); return b[2]-b[0]

def kicker(base,fonts,txt,y):
    f=_mont(fonts,40,800); px,py=34,18
    w=tw(f,txt); th=f.getbbox(txt)[3]-f.getbbox(txt)[1]; bw=w+px*2; bh=th+py*2; x=(W-bw)/2
    d=ImageDraw.Draw(base); d.rounded_rectangle([x,y,x+bw,y+bh],radius=bh/2,fill=ORANGE+(255,))
    d.text((W/2,y+bh/2-2),txt,font=f,fill=(20,12,8,255),anchor='mm'); return y+bh

def headline(base,fonts,lines,top_y,fsize=210,lh=200):
    f=_f(fonts,"Anton.ttf",fsize); y=top_y
    for txt,col in lines:
        x=(W-tw(f,txt))/2
        lay=Image.new('RGBA',base.size,(0,0,0,0)); ImageDraw.Draw(lay).text((x+7,y+8),txt,font=f,fill=(0,0,0,180))
        base.alpha_composite(lay.filter(ImageFilter.GaussianBlur(6)))
        ImageDraw.Draw(base).text((x,y),txt,font=f,fill=col+(255,)); y+=lh

def lockup(base,fonts,x,y,label="Brand"):
    base.alpha_composite(brand_mark(96),(x,y))
    ImageDraw.Draw(base).text((x+112,y+48),label,font=_mont(fonts,52,800),fill=(238,238,240,255),anchor='lm')

def main():
    ap=argparse.ArgumentParser()
    ap.add_argument('--photo',required=True); ap.add_argument('--out',required=True)
    ap.add_argument('--kicker',required=True); ap.add_argument('--l1',required=True); ap.add_argument('--l2',required=True)
    ap.add_argument('--brand',default=''); ap.add_argument('--fonts',default='fonts')
    a=ap.parse_args()
    base=make_bg(); lockup(base,a.fonts,60,70,label=a.brand if a.brand.strip() else "Short")
    ky=kicker(base,a.fonts,a.kicker.upper(),320)
    headline(base,a.fonts,[(a.l1.upper(),WHITE),(a.l2.upper(),YELLOW)],ky+40)
    diam=620; cx,cy=540,1460
    paste_center(base,ring(diam,54,(236,126,60)).filter(ImageFilter.GaussianBlur(22)),cx,cy)
    paste_center(base,ring(diam,16,(245,150,80)),cx,cy)
    paste_center(base,make_avatar(a.photo,diam),cx,cy)
    if a.brand.strip():
        ImageDraw.Draw(base).text((W/2,1858),a.brand.upper(),font=_mont(a.fonts,30,700),fill=(180,182,190,235),anchor='mm')
    base.convert('RGB').save(a.out,quality=95); print("wrote",a.out)

if __name__=='__main__': main()
```
