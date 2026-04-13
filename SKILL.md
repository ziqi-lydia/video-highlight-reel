---
name: video-highlight-reel
description: Cut a long video into a polished highlight reel with transitions, AI voice-over, subtitles, background music, and branded intro/outro cards. Use when asked to edit, cut, or create a short demo/highlight from a longer video.
---

# Video Highlight Reel Creator

Create polished highlight reels from long videos with professional editing: segment cutting, xfade transitions, AI voice-over, subtitles, background music, and branded intro/outro title cards.

## When to Use
- User sends a long video and wants it cut into a shorter highlight reel or demo
- User asks for a "30-second version", "highlight reel", "demo video", "use case video"
- User wants professional editing applied to raw screen recordings or footage

## Prerequisites (install if missing)
```bash
pip install edge-tts Pillow imageio-ffmpeg
```

Find ffmpeg binary:
```python
import imageio_ffmpeg
ffmpeg = imageio_ffmpeg.get_ffmpeg_exe()
```

## Pipeline Overview

The full pipeline has 7 phases. Execute them in order. Each phase produces intermediate files in a working directory.

```
Phase 1: Analyze > Phase 2: Cut Segments > Phase 3: Intro/Outro Cards
> Phase 4: Join with Transitions > Phase 5: Voice-Over
> Phase 6: Background Music + Audio Mix > Phase 7: Subtitles + Final Render
```

---

## Phase 1: Analyze the Source Video

### 1a. Get video metadata
```bash
ffmpeg -i "SOURCE_VIDEO" -hide_banner
```
Extract: duration, resolution, fps.

### 1b. Extract frames for analysis
Extract one frame every 15 seconds (adjust based on video length):
```bash
ffmpeg -i "SOURCE_VIDEO" -vf "fps=1/15" -q:v 2 "WORKDIR/frame_%04d.png"
```

### 1c. Analyze frames with vision model
Inspect each extracted frame using inspect({ image: "path" }) to identify:
- Key moments and scene changes
- Most visually engaging segments
- Start/end timestamps for each interesting segment
- Any unwanted content (recording UI, blank screens, etc.)

### 1d. Plan the highlight reel
Decide on:
- **Target duration**: Usually 30-45s for social media, up to 60s for demos
- **Number of segments**: 6-10 segments works well
- **Segment durations**: 2-9 seconds each
- **Narrative arc**: Opening hook > progression > climax > conclusion
- **Voice-over script**: Short narration for each segment

Store the plan:
```javascript
var editPlan = {
  source: "path/to/source.mp4",
  resolution: "1920x1080",
  fps: 30,
  segments: [
    { start: 5.0, end: 9.5, label: "Opening hook", voiceover: "Watch how..." },
    { start: 23.0, end: 28.0, label: "Key feature", voiceover: "With one click..." },
  ],
  intro: { title: "Product Name", subtitle: "Tagline here" },
  outro: { title: "Try it today", subtitle: "website.com" },
  totalDuration: 40
}
```

---

## Phase 2: Cut Segments

Cut each segment from the source video. Use re-encoding to ensure clean cuts and uniform codec:

```bash
ffmpeg -y -ss START -to END -i "SOURCE_VIDEO" \
  -vf "scale=WIDTH:HEIGHT" \
  -c:v libx264 -preset fast -crf 18 \
  -r FPS -pix_fmt yuv420p \
  -an "WORKDIR/seg_01.mp4"
```

Key flags:
- `-an` strips audio (we'll add voice-over later)
- `-crf 18` high quality
- `-pix_fmt yuv420p` ensures compatibility
- `-r FPS` forces consistent frame rate

---

## Phase 3: Intro/Outro Title Cards

Generate title card images with Pillow, then convert to video segments.

```python
from PIL import Image, ImageDraw, ImageFont

def create_title_card(width, height, title, subtitle, bg_color, text_color, output_path):
    img = Image.new("RGB", (width, height), bg_color)
    draw = ImageDraw.Draw(img)
    try:
        title_font = ImageFont.truetype("arial.ttf", int(height * 0.08))
        sub_font = ImageFont.truetype("arial.ttf", int(height * 0.04))
    except:
        title_font = ImageFont.load_default()
        sub_font = ImageFont.load_default()
    bbox = draw.textbbox((0, 0), title, font=title_font)
    tw = bbox[2] - bbox[0]
    draw.text(((width - tw) // 2, height // 2 - int(height * 0.08)),
              title, fill=text_color, font=title_font)
    if subtitle:
        bbox2 = draw.textbbox((0, 0), subtitle, font=sub_font)
        sw = bbox2[2] - bbox2[0]
        draw.text(((width - sw) // 2, height // 2 + int(height * 0.03)),
                  subtitle, fill=text_color, font=sub_font)
    img.save(output_path)
```

Convert images to video:
```bash
ffmpeg -y -loop 1 -i "WORKDIR/intro_card.png" -t 3 \
  -vf "scale=WIDTH:HEIGHT" \
  -c:v libx264 -preset fast -crf 18 \
  -r FPS -pix_fmt yuv420p \
  -an "WORKDIR/intro.mp4"
```

---

## Phase 4: Join Segments with Transitions

Use ffmpeg's xfade filter to create smooth transitions between segments.

For 2 segments (A > B) with 0.5s crossfade:
```bash
ffmpeg -y -i A.mp4 -i B.mp4 \
  -filter_complex "[0:v][1:v]xfade=transition=fade:duration=0.5:offset=OFFSET_A" \
  -c:v libx264 -preset fast -crf 18 -pix_fmt yuv420p \
  -an "output.mp4"
```

Supported xfade transitions: fade, wipeleft, wiperight, wipeup, wipedown, slideleft, slideright, circlecrop, rectcrop, fadeblack, fadewhite, radial, dissolve, pixelize, diagtl, diagtr, diagbl, diagbr

---

## Phase 5: AI Voice-Over

Generate voice-over audio with edge-tts:
```python
import edge_tts
import asyncio

async def generate_voiceover(segments, output_dir):
    for i, seg in enumerate(segments):
        if not seg.get("voiceover"):
            continue
        communicate = edge_tts.Communicate(
            seg["voiceover"],
            voice="en-US-AndrewMultilingualNeural",
            rate="+5%"
        )
        audio_path = f"{output_dir}/vo_{i:02d}.mp3"
        await communicate.save(audio_path)

asyncio.run(generate_voiceover(segments, "WORKDIR"))
```

Good voice choices:
- en-US-AndrewMultilingualNeural - professional male
- en-US-AriaNeural - professional female
- en-US-GuyNeural - casual male

---

## Phase 6: Background Music + Audio Mix

Mix background music with voice-over:
```bash
ffmpeg -y -i "WORKDIR/with_vo.mp4" -i "WORKDIR/ambient_music.m4a" \
  -filter_complex "[0:a]volume=1.0[voice];[1:a]volume=0.12[music];[voice][music]amix=inputs=2:duration=first:dropout_transition=0[aout]" \
  -map 0:v -map "[aout]" \
  -c:v copy -c:a aac -b:a 192k -shortest \
  "WORKDIR/with_music.mp4"
```

---

## Phase 7: Subtitles + Final Render

Burn subtitles into video (hardcoded):
```bash
ffmpeg -y -i "WORKDIR/with_music.mp4" \
  -vf "subtitles=WORKDIR/subs.srt:force_style='FontName=Arial,FontSize=22,PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,Outline=2,Shadow=1,Alignment=2,MarginV=40'" \
  -c:v libx264 -preset fast -crf 18 -pix_fmt yuv420p \
  -c:a copy \
  "OUTPUT_PATH/highlight_reel.mp4"
```

---

## Quality Checklist

Before delivering, verify:
1. Video plays smoothly - no encoding errors
2. Transitions are smooth - no black frames or glitches
3. Voice-over is synced with video segments
4. Subtitles match voice-over text and timing
5. Background music doesn't overpower voice-over
6. Intro/outro cards are clean and readable
7. Total duration matches target
