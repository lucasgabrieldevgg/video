# Recipes — cut & assemble (R01–R16)

*Dual-stack: ffmpeg CLI (deterministic) + Python (MoviePy for composition, plain orchestration for CLI-native ops). Every recipe ends with its guards. Craft context lives in `the-edit.md`; the method in `the-pipeline.md`.*

*Recipe corpus written for this skill, merging technique ideas from upstream — the EDL + review-gate shape (nateherkai/hyperframes-student-kit), reframing and speed-ramp strategies (louisedesadeleer/clipify), multicam/xfade battle patterns (jianshuo/claude-skills) — re-authored as original dual-stack implementations. Full credits: ATTRIBUTION.md.*

### R01 · Probe & audit

**When:** always, before any other recipe.

```bash
ffprobe -v error -print_format json -show_format -show_streams "$IN" > probe.json
```

```python
import json, subprocess
def probe(path):
    out = subprocess.run(["ffprobe","-v","error","-print_format","json",
                          "-show_format","-show_streams",path],
                         capture_output=True, text=True).stdout
    d = json.loads(out)
    v = next(s for s in d["streams"] if s["codec_type"]=="video")
    return {"vfr": v["r_frame_rate"] != v["avg_frame_rate"],
            "fps": eval(v["r_frame_rate"]),
            "size": (v["width"], v["height"]),
            "rotation": next((int(x.get("rotation",0)) for x in v.get("side_data_list",[])), 0),
            "color": (v.get("color_space"), v.get("color_primaries"), v.get("color_transfer")),
            "duration": float(d["format"]["duration"])}
```

Guards: VFR→normalize first (R27); unknown color→tag on final encode; rotation→decide representation; PCM in MP4→re-mux.

### R02 · Trim — lossless vs frame-accurate

```bash
# lossless (snaps to keyframe): instant
ffmpeg -ss 00:02:10 -t 45 -i in.mp4 -c copy cut.mp4
# frame-accurate (word-time surgical): re-encode once
ffmpeg -ss 00:02:10.240 -t 44.86 -i in.mp4 -c:v libx264 -crf 18 -pix_fmt yuv420p \
  -colorspace bt709 -color_primaries bt709 -color_trc bt709 -c:a copy cut.mp4
```

```python
from moviepy import VideoFileClip
clip = VideoFileClip("in.mp4").subclipped(130.240, 175.10)
clip.write_videofile("cut.mp4", codec="libx264", audio_codec="copy",
                     ffmpeg_params=["-crf","18","-pix_fmt","yuv420p"])
```

Guards: `-ss` before `-i`; word-time math from the transcript (in = first word −0.10..0.15 s); `-c copy` for drafts, re-encode only for the conform.

### R03 · Silence removal → EDL

```bash
# 1. detect
ffmpeg -i in.mp4 -af silencedetect=noise=-30dB:d=0.55 -f null - 2> silences.txt
# 2. build keep-ranges complement; 3. render one filtergraph pass
ffmpeg -i in.mp4 -filter_complex_script edl.filter.txt -c:v libx264 -crf 18 \
  -pix_fmt yuv420p -colorspace bt709 -color_primaries bt709 -color_trc bt709 tight.mp4
```

`edl.filter.txt` (generated): `[0:v]trim=0:12.4,setpts=PTS-STARTPTS[v0]; [0:a]atrim=0:12.4,asetpts=PTS-STARTPTS[a0]; ... [v0][a0][v1][a1]...concat=n=K:v=1:a=1[v][a]`

```python
import re
def silences_to_keep(text, min_breath=0.24):
    spans = [(float(a), float(b)) for a, b in
             re.findall(r"silence_start: ([\d.]+).*?silence_end: ([\d.]+)", text, re.S)]
    keep, cur = [], 0.0
    for s, e in spans:
        keep.append((cur, s + min_breath)); cur = e          # breath biased to previous phrase
    keep.append((cur, None))
    return keep                                            # → build trim/atrim filtergraph
```

Guards: keep 0.14–0.24 s breath (sentence-aware: 0.20 after `.?!`); threshold −30 dB / 0.55 s defaults; room-tone repair for joins (Phase 3); re-time the transcript after.

### R04 · Mistake cuts (review-gated)

Candidates from the transcript scan (one criterion per pass: repetition → restarts → corrections → stutters → fillers). Then:

```bash
ffmpeg -i in.mp4 -filter_complex_script cuts.filter.txt -c:v libx264 -crf 18 ... clean.mp4
```

```python
def apply_cuts(keep, cuts):
    """keep: [(start,end)] on source timeline; cuts: [(start,end,reason)] — both from the approved ledger."""
    return sorted([r for r in keep if not any(c[0] < r[1] and c[1] > r[0] for c in cuts)])
```

Guards: two-state decisions only; every candidate read in context before cutting; 20–30 ms fade on hard word-to-word joins; never cut emphatic repetition; zero cuts is a valid outcome.

### R05 · Concat — same spec (lossless)

```bash
printf "file 'a.mp4'\nfile 'b.mp4'\nfile 'c.mp4'\n" > list.txt
ffmpeg -f concat -safe 0 -i list.txt -c copy joined.mp4
```

```python
from moviepy import VideoFileClip, concatenate_videoclips
joined = concatenate_videoclips([VideoFileClip(p) for p in ["a.mp4","b.mp4","c.mp4"]])
joined.write_videofile("joined.mp4", codec="libx264", audio_codec="aac")
```

Guards: probe all inputs first — identical codec/fps/SAR/resolution or the demuxer join glitches; MoviePy path re-encodes (normalize specs when mixed).

### R06 · Concat with transitions (xfade + acrossfade)

```bash
ffmpeg -i a.mp4 -i b.mp4 -filter_complex \
 "[0:v][1:v]xfade=transition=fade:duration=0.4:offset=5.6[v]; \
  [0:a][1:a]acrossfade=d=0.4[a]" -map "[v]" -map "[a]" \
 -c:v libx264 -crf 18 -pix_fmt yuv420p \
 -colorspace bt709 -color_primaries bt709 -color_trc bt709 out.mp4
```

(offset = `len(a) − fade`; for a 6 s first clip and 0.4 s fade → 5.6)

```python
def xfade_offsets(durations, fade=0.4):
    """offset_n = cumulative visible length before clip n+1 minus n*fade."""
    offs, t = [], 0.0
    for d in durations[:-1]:
        t += d; offs.append(round(t - fade * (offs.__len__() + 1), 3))
    return offs    # chain: [0][1] xfade at offs[0], result [2] xfade at offs[1] ...
```

Guards: offsets are cumulative (off-by-one-fade is the classic desync); inputs normalized to same fps/size first; hard cuts are the default — transitions must argue for themselves (the-edit.md §3).

### R07 · J-cut / L-cut

```bash
# L-cut: picture cuts at 10s, voice of shot A runs 1.5 s over B's picture
ffmpeg -i a.mp4 -i b.mp4 -filter_complex \
 "[0:v]trim=0:10,setpts=PTS-STARTPTS[v0]; \
  [1:v]trim=0:8.5,setpts=PTS-STARTPTS[v1]; \
  [0:v][1:v]concat=n=2:v=1:a=0[v]; \
  [0:a]atrim=0:11.5,asetpts=PTS-STARTPTS,apad=whole_dur=18.5[a]" \
  -map "[v]" -map "[a]" -c:v libx264 -crf 18 -pix_fmt yuv420p \
  -colorspace bt709 -color_primaries bt709 -color_trc bt709 lcut.mp4
```

J-cut mirrors it: `[1:a]adelay=1500|1500[bd];[0:a][bd]amix=normalize=0[a]` — B's audio leads 1.5 s under A's tail.

```python
from moviepy import VideoFileClip, CompositeAudioClip
a, b = VideoFileClip("a.mp4"), VideoFileClip("b.mp4")
voice = a.audio.subclipped(0, 11.5)                     # voice trails picture
picture = concatenate_videoclips([a.subclipped(0,10), b.subclipped(0,8.5)])
final = picture.with_audio(CompositeAudioClip([voice]))
```

Guards: 0.5–2 s lead/lag; audio aligned to word onsets; `amix=normalize=0` when mixing.

### R08 · Constant speed

```bash
ffmpeg -i in.mp4 -vf "setpts=PTS/1.5,fps=30" -af "atempo=1.5" \
 -c:v libx264 -crf 18 -pix_fmt yuv420p -c:a aac spd.mp4
```

```python
from moviepy import VideoFileClip, vfx
VideoFileClip("in.mp4").with_effects([vfx.MultiplySpeed(1.5)]).write_videofile("spd.mp4")
```

Guards: `atempo` 0.5–2.0 per instance (chain for more); 1.5× reads "brisk", 2× "punchy", 3–4× timelapse; mute ambient audio rather than chipmunk it.

### R09 · Speed ramp

```bash
# segment speeds then concat: 1x (0-5s) → 4x (5-20s) → 1x (20-25s)
ffmpeg -i in.mp4 -filter_complex \
 "[0:v]trim=0:5,setpts=PTS-STARTPTS[v1];[0:a]atrim=0:5,asetpts=PTS-STARTPTS[a1];\
  [0:v]trim=5:20,setpts=(PTS-STARTPTS)/4[v2];[0:a]atrim=5:20,asetpts=(PTS-STARTPTS)/4,atempo=4.0[a2];\
  [0:v]trim=20:25,setpts=PTS-STARTPTS[v3];[0:a]atrim=20:25,asetpts=PTS-STARTPTS[a3];\
  [v1][a1][v2][a2][v3][a3]concat=n=3:v=1:a=1[v][a]" -map "[v]" -map "[a]" ramp.mp4
```

```python
from moviepy import VideoFileClip, vfx, concatenate_videoclips
c = VideoFileClip("in.mp4")
segments = [(0,5,1.0),(5,20,4.0),(20,25,1.0)]
concatenate_videoclips(
  [c.subclipped(s,e).with_effects([vfx.MultiplySpeed(sp)]) for s,e,sp in segments]
).write_videofile("ramp.mp4")
```

Guards: ramps preserve the whole arc (trimming drops beats); audio muted if ambient; hard cuts between segments read intentional.

### R10 · 16:9 → 9:16, three strategies

```bash
# (a) blur-pad (no faces lost; last resort aesthetically)
ffmpeg -i in.mp4 -filter_complex \
 "[0:v]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920,boxblur=20:2[bg];\
  [0:v]scale=1080:-2[fg];[bg][fg]overlay=(W-w)/2:0" -c:v libx264 -crf 19 vertical.mp4
# (b) center/smart crop (single talker)
ffmpeg -i in.mp4 -vf "crop=608:1080:x=656:0,scale=1080:1920:flags=lanczos" -c:a copy crop916.mp4
```

(c) **active-speaker pan**: sample faces at ~5 fps (MediaPipe FaceLandmarker), speaker = highest mouth-aspect-ratio variance over a 1 s window, hysteresis ≥1.5 s per segment; render a step-function crop x:

```python
def pan_expr(chunks, W=1920, cw=608):        # chunks: [(t0,t1,cx)]
    parts = [f"between(t,{t0},{t1})*{max(0,min(W-cw,cx-cw//2))}" for t0,t1,cx in chunks]
    return "+".join(parts)                   # → crop=608:1080:x='EXPR':y=0
```

Guards: never center-crop a two-person interview (lands between faces); 0-detections → deterministic fixed crops per speaker, not the auto-center; `setsar=1` after crop; 1.78× upscale is visible on text — prefer native crop dims when overlays must stay sharp; split-screen (R11) when both faces matter.

### R11 · Split-screen with speaker swap

```bash
ffmpeg -i in.mp4 -filter_complex \
 "[0:v]split=2[a][b];[a]crop=608:1080:0:0,scale=1080:960[Lt];[b]crop=608:1080:1312:0,scale=1080:960[Rt];\
  [Lt]split[L0][L1];[Rt]split[R0][R1];[L0][R0]vstack[layoutL];[R1][L1]vstack[layoutR];\
  [layoutL][layoutR]overlay=0:0:enable='RIGHT_SPEAKS'" -map "[v]" -map 0:a split.mp4
```

`RIGHT_SPEAKS` = `between(t,a,b)+between(t,c,d)...` from the speaker timeline (R10 detection).

Guards: active speaker on top; same 5 fps sampling + hysteresis; audio stays stereo-original.

### R12 · Freeze frame + zoom punch

```bash
ffmpeg -i in.mp4 -filter_complex \
 "[0:v]select='lt(t,10)'[pre];[0:v]select='gte(t,10)*lt(t,10.04)',loop=loop=25:size=1,trim=0:1,setsar=1,\
  zoompan=z='min(zoom+0.004,1.12)':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=25:s=1920x1080[frz];\
  [pre][frz]concat" punch.mp4
```

```python
from moviepy import VideoFileClip, ImageClip, vfx, concatenate_videoclips
c = VideoFileClip("in.mp4")
frame = ImageClip(c.get_frame(10.0), duration=1.0).resized(1.10)  # subtle push
concatenate_videoclips([c.subclipped(0,10), frame, c.subclipped(10.04, None)])
```

Guards: freeze ≤1 s (land the punchline, don't live there); zoom punch ≤3 per short reel.

### R13 · Loop / boomerang

```bash
ffmpeg -i clip.mp4 -filter_complex "[0:v]reverse[r];[0:v][r]concat=n=2:v=1:a=0[v]" \
  -map "[v]" -c:v libx264 -crf 18 -pix_fmt yuv420p boomerang.mp4
# "3-hit" loop: [0:v][r][0:v]concat=n=3 instead
```

Guards: end frame must flow into start (loop-crafting, the-edit.md §7); 1 forward + 1 reverse + 1 forward for a "3-hit" rhythm.

### R14 · Timelapse / decimation

```bash
ffmpeg -i frames%04d.jpg -vf "setpts=0.25*PTS" -r 30 tl.mp4
# from video: keep 1-in-30
ffmpeg -i in.mp4 -vf "select='not(mod(n,30))',setpts=N/FRAME_RATE/TB,fps=30" tl.mp4
```

Guards: `setpts` factor = input_fps/30; no `atempo` needed (no live audio).

### R15 · Multicam sync (FFT cross-correlation)

```python
import numpy as np, subprocess
def audio(path):  # decode to mono 48k float32
    raw = subprocess.run(["ffmpeg","-i",path,"-vn","-ac","1","-ar","48000",
                          "-f","f32le","-"], capture_output=True).stdout
    return np.frombuffer(raw, dtype=np.float32)

def find_offset(ref, other, sr=48000):
    ref, other = audio(ref), audio(other)
    n = 1 << int(np.ceil(np.log2(len(ref) + len(other))))
    R, O = np.fft.rfft(ref, n), np.fft.rfft(other, n)
    corr = np.fft.irfft(R * np.conj(O), n)
    lag = np.argmax(corr)
    return (lag if lag < n/2 else lag - n) / sr   # seconds `other` lags `ref`
```

```bash
# apply: delay the lagging angle (-3.2s found)
ffmpeg -i cam_a.mp4 -i cam_b.mp4 -filter_complex "[1:v]setpts=PTS+3.2/TB[v];[1:a]adelay=3200|3200[a]" ...
```

Guards: sync on continuous audio (claps are brittle; correlation over the whole track isn't); verify by flipping between angles on a hard plosive; trim all angles to the common timeline before the edit.

### R16 · Scene detection & contact sheets

```bash
ffmpeg -i in.mp4 -vf "select='gt(scene,0.3)',showinfo" -f null - 2> scenes.txt   # cut list
ffmpeg -i in.mp4 -vf "fps=1/10,scale=216:384,tile=6x3" -frames:v 1 sheet.jpg     # contact sheet
```

Guards: 0.3 threshold catches hard cuts, misses dissolves; the contact sheet is for *you* to read — look at it before planning (right person? face position? brightness?); use frame strips (Phase 5) for motion defects.
