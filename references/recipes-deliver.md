# Recipes — enhance & deliver (R17–R32)

*Dual-stack, same contract as recipes-cut.md. Delivery verification (R32) is not optional — it is the last recipe every project runs.*

### R17 · Word-level transcription → SRT

```bash
ffmpeg -i in.mp4 -vn -ac 1 -ar 16000 audio.wav
whisper audio.wav --model base --word_timestamps True --output_format srt --language en
```

```python
from faster_whisper import WhisperModel   # GPU bulk path
model = WhisperModel("base", device="cuda", compute_type="float16")
segs, info = model.transcribe("audio.wav", word_timestamps=True)
words = [w for s in segs for w in s.words]     # (start, end, word)
```

Group words into cues (≤42 Latin / ~15 CJK per line, 2 lines max, reading speed ≤20 chars/s, minimum 0.3 s). Normalize ms separators to `,`.

Guards: glossary pass before any judgment; `tiny.en` for scanning, `base` default, `medium`+ for the final verification pass; store word JSON — the SRT is derived, never the source of truth.

### R18 · Burn captions (libass)

```bash
ffmpeg -filters 2>&1 | grep -E "subtitles|ass"      # verify BEFORE promising
ffmpeg -t 30 -i in.mp4 -vf "subtitles=captions.srt:force_style='Fontname=Inter\,Fontsize=12\,PrimaryColour=&H00FFFFFF\,OutlineColour=&H00000000\,BorderStyle=1\,Outline=2\,Shadow=1\,MarginL=20\,MarginR=20\,MarginV=40'" \
  -c:v libx264 -crf 20 -c:a copy preview.mp4        # 30s preview checkpoint
ffmpeg -ss 25 -i preview.mp4 -frames:v 1 longest_line.png   # read THIS frame
# full render only after the frame reads right:
ffmpeg -i in.mp4 -vf "subtitles=captions.srt:force_style='...'" \
  -c:v libx264 -crf 19 -pix_fmt yuv420p \
  -colorspace bt709 -color_primaries bt709 -color_trc bt709 -c:a copy burned.mp4
```

```python
def to_ass(words, style="impact"):
    """word-timed ASS: one word per event, active word highlighted via karaoke \\k tags."""
    events = []
    for i, w in enumerate(words):
        dur = int((w.end - w.start) * 100)   # centiseconds
        events.append(f"Dialogue: 0,{ts(w.start)},{ts(w.end)},Default,,0,0,0,,"
                      f"{{\\k{dur}}}{escape(w.text)}")
    return ass_header(style) + "\n".join(events)
```

Guards: fontsize is PlayRes-scaled, calibrate per video (start 12); commas inside `force_style` escaped `\,`; longest-line cue is the frame you check; never over the face; safe area x 90–930 / y 200–1600 on 1080×1920.

### R19 · Soft-mux (toggleable track)

```bash
ffmpeg -i in.mp4 -i captions.srt -map 0:v -map 0:a -map 1:0 \
  -c:v copy -c:a copy -c:s mov_text \
  -metadata:s:s:0 language=eng out_softsub.mp4
```

Guards: zero re-encode (instant, reversible); YouTube auto-detects, players honor it — feeds like WeChat/Douyin do not, burn instead; ship the raw `.srt` alongside either way.

### R20 · Two-pass loudness normalization

```bash
ffmpeg -i in.mp4 -af loudnorm=I=-14:TP=-1:LRA=11:print_format=summary -f null -   # pass 1: read measured_*
ffmpeg -i in.mp4 -af loudnorm=I=-14:TP=-1:LRA=11:measured_I=-18.6:measured_TP=-2.1:\
measured_LRA=5.3:measured_thresh=-29.8:linear=true -c:v copy -c:a aac -b:a 192k out.mp4
```

```python
def parse_loudnorm(stderr):
    import re
    keys = ["measured_I","measured_TP","measured_LRA","measured_thresh"]
    vals = {k: re.search(rf"{k}: ([-\d.]+)", stderr).group(1) for k in keys}
    return (f"loudnorm=I=-14:TP=-1:LRA=11:{' '.join(f'{k}={v}' for k,v in vals.items())}:linear=true")
```

Guards: two-pass with `linear=true` (one-pass is dynamic and pumps); targets −14 web / −16 short-form / −23 broadcast; `-c:v copy` — only audio is touched.

### R21 · Music bed + ducking

```bash
ffmpeg -i voice.mp4 -i music.mp3 -filter_complex \
 "[1:a]volume=0.12[bed];[bed][0:a]sidechaincompress=threshold=0.03:ratio=8:\
attack=5:release=300:makeup=1[duck];[0:a][duck]amix=normalize=0[a]" \
 -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k mixed.mp4
```

```python
from moviepy import VideoFileClip, AudioFileClip, CompositeAudioClip
voice, bed = VideoFileClip("voice.mp4"), AudioFileClip("music.mp3").with_volume_scaled(0.12)
final = voice.with_audio(CompositeAudioClip([voice.audio, bed]))
```

Guards: bed 14–20 dB under voice; ducks under *every* spoken line; `normalize=0`; dub tracks keep original as 0.15–0.25 bed.

### R22 · Speech denoise

```bash
ffmpeg -i in.mp4 -af "afftdn=nr=12:nf=-30" -c:v copy -c:a aac clean_audio.mp4
# heavier: DeepFilterNet3 (cap 20 dB — full strength sounds watery)
```

Guards: denoise is the *first* audio op; steady noise → `afftdn`, speech restoration → DeepFilterNet; verify by listening to a loud phrase and a quiet one.

### R23 · SFX layer

```bash
ffmpeg -i video.mp4 -i hit.wav -i whoosh.wav -filter_complex \
 "[1:a]adelay=3200|3200,volume=-8dB[h];[2:a]adelay=5400|5400,volume=-12dB[w];\
  [0:a][h][w]amix=inputs=3:normalize=0[a]" -map 0:v -map "[a]" -c:v copy sfx.mp4
```

Guards: levels relative to the finished voice's p95 (ui −14 / whoosh −12 / impact −8 / meme −3 dB); onset aligned to the visual landing within 2 frames (mind leading silence in files); never on top of a word; each sound once per deliverable.

### R24 · Color correction + grade (one pass)

```bash
ffmpeg -i in.mp4 -vf "eq=contrast=1.06:saturation=1.05,curves=b='0/0.05 1/0.97',vignette=PI/5,format=yuv420p" \
 -c:v libx264 -crf 19 -colorspace bt709 -color_primaries bt709 -color_trc bt709 -c:a copy graded.mp4
```

Guards: order = white balance → exposure → contrast → saturation → look; grade *after* conform, *before* caption burn; check a skin-tone frame and a wide at delivery size.

### R25 · Apply a LUT

```bash
ffmpeg -i in.mp4 -vf "lut3d=file='teal_cube.cube',format=yuv420p" \
 -c:v libx264 -crf 19 -colorspace bt709 -color_primaries bt709 -color_trc bt709 -c:a copy lutted.mp4
```

Guards: LUTs are looks *after* correction; one LUT per output; log footage gets its normalization LUT first.

### R26 · Fix washed-out color (both paths)

```bash
# path A: metadata only, instant
ffmpeg -i in.mp4 -c copy -colorspace bt709 -color_primaries bt709 -color_trc bt709 fixed.mp4
# path B: baked wrong (bt601 matrix in an HD file) — re-encode with explicit conversion
ffmpeg -i in.mp4 -vf "scale=in_color_matrix=bt601:out_color_matrix=bt709,format=yuv420p" \
 -c:v libx264 -crf 18 -colorspace bt709 -color_primaries bt709 -color_trc bt709 fixed.mp4
# HDR→SDR (PQ/HLG source to an SDR platform):
ffmpeg -i in.mp4 -vf "zscale=t=linear:npl=203,format=gbrpf32le,tonemap=tonemap=hable:desat=0,\
zscale=w=1920:h=1080:t=bt709:m=bt709:p=bt709:r=tv,format=yuv420p" -c:v libx264 -crf 19 sdr.mp4
```

Guards: diagnose first (probe tags; does the shift appear only after upload?); never apply the HDR chain to SDR footage.

### R27 · Fix VFR (screen recordings)

```bash
ffmpeg -i vfr.mp4 -vsync cfr -r 30 -vf "fps=30,format=yuv420p" \
 -c:v libx264 -crf 18 -c:a aac cfr.mp4
```

Guards: run *before* any other filter or cut (everything downstream desyncs on VFR); `-r` matches the dominant rate (30 for screen, 24/25/30 for footage per probe).

### R28 · Deinterlace legacy footage

```bash
ffmpeg -i interlaced.mp4 -vf "bwdif=mode=send_field:parity=auto:deint=all,format=yuv420p" \
 -c:v libx264 -crf 18 -c:a copy progressive.mp4
```

Guards: `bwdif` > `yadif` for quality; confirm interlacing first (combing on motion in a paused frame); 25i→50p or 29.97i→59.94p preserves motion.

### R29 · Watermark / picture-in-picture

```bash
ffmpeg -i base.mp4 -i logo.png -filter_complex \
 "[1:v]scale=160:-1[wm];[0:v][wm]overlay=W-w-24:24:format=auto[v]" -map "[v]" -map 0:a \
 -c:v libx264 -crf 19 -pix_fmt yuv420p branded.mp4
# PiP (corner cam over screen recording):
"[1:v]scale=360:-2[cam];[0:v][cam]overlay=W-w-20:H-h-20"
```

Guards: keep logos out of the caption band and platform UI zones; verified brand assets only — never redraw or approximate a mark; alpha (`format=auto`) not a black box.

### R30 · Green screen key

```bash
ffmpeg -i bg.mp4 -i green.mp4 -filter_complex \
 "[1:v]chromakey=0x00FF00:0.28:0.1[fg];[0:v][fg]overlay[v]" -map "[v]" -map 1:a keyed.mp4
```

Guards: even lighting beats any keyer; spill suppression before edge softness; `colorkey` for non-green backgrounds; check hair edges at full resolution, not preview scale.

### R31 · Thumbnail & GIF derivatives

```bash
ffmpeg -ss 00:01:37 -i final.mp4 -frames:v 1 -q:v 2 thumbnail.jpg
ffmpeg -ss 15 -t 6 -i final.mp4 -vf "fps=12,scale=540:-1:flags=lanczos,split[a][b];\
[a]palettegen=max_colors=128[p];[b][p]paletteuse=dither=bayer:bayer_scale=3" loop.gif
```

Guards: thumbnail = a real expression/moment frame (face + emotion), not frame 0; GIF ≤6 s and ≤128 colors or it weighs like a video; palettegen+paletteuse as one chain, always.

### R32 · Platform presets + the verification gate

```bash
# YouTube 16:9 master
ffmpeg -i edit.mp4 -vf "scale=1920:1080:flags=lanczos,format=yuv420p" \
 -c:v libx264 -crf 18 -preset medium -profile:v high -g 60 -keyint_min 30 \
 -colorspace bt709 -color_primaries bt709 -color_trc bt709 \
 -c:a aac -b:a 256k -movflags +faststart youtube.mp4

# Shorts / Reels / TikTok 9:16
ffmpeg -i edit.mp4 -vf "scale=1080:1920:flags=lanczos,setsar=1,format=yuv420p" \
 -c:v libx264 -crf 19 -preset medium -g 60 \
 -colorspace bt709 -color_primaries bt709 -color_trc bt709 \
 -c:a aac -b:a 192k -movflags +faststart short.mp4
# caps: WeChat Channels ≤10 Mbps → add -maxrate 8M -bufsize 16M
```

Then the gate, in order, no exceptions:

```bash
# 1. frame strips across every transition (A,B = frames around each cut)
ffmpeg -i final.mp4 -vf "select='between(n,A,B)',scale=480:270,tile=layout=6x7" -frames:v 1 strip.png
# 2. fresh ASR over final audio, diffed against the expected transcript
ffmpeg -i final.mp4 -vn -ac 1 -ar 16000 final.wav && whisper final.wav --model medium ...
# 3. measured loudness
ffmpeg -i final.mp4 -af ebur128 -f null - 2>&1 | grep -A12 "Summary"
# 4. probe the shipped file
ffprobe -v error -print_format json -show_format -show_streams final.mp4
```

Guards: faststart on every MP4; captions sidecar (`.srt`) ships even when burned; safe-area text check at phone scale; the delivery report carries real numbers (LUFS, TP, duration, removed-s %, output hash) and the modality list actually used — a pass never comes from file existence.
