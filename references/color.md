# Color — spaces, flags, and looks

*Original content for this skill; the HDR→SDR chain and active-speaker crop tone-mapping adapted from jianshuo/claude-skills (wjs-reframing-video).*

Color bugs in agent-edited video are almost never creative — they are *metadata* bugs. Fix the plumbing before touching the look.

## 1 · The four questions every file answers (or doesn't)

| Question | Values | Why it matters |
| --- | --- | --- |
| Primaries | bt709 / bt601 / bt2020 | which red/green/blue are "real" |
| Transfer (TRC) | bt709 / gamma22 / smpte2084 (PQ) / arib-std-b67 (HLG) | how code values map to brightness |
| Matrix (colorspace) | bt709 / bt601 / bt2020nc | how YUV was derived from RGB |
| Range | limited (tv, 16–235) / full (pc, 0–255) | where black and white actually sit |

SD (bt601) and HD (bt709) use *different matrices for the same resolution numbers*. A file with unspecified tags forces players to guess — YouTube guesses differently than your local player, which is exactly why footage "looked fine locally" and ships washed out or shifted.

## 2 · The washed-out bug (diagnosis and both fixes)

**Diagnose:** `ffprobe` shows `color_space=unknown` (typical after a filter pass, a crop, or a naive re-encode), or the grays look lifted only after upload.

**Fix A — no re-encode** (container metadata only, instant):

```bash
ffmpeg -i in.mp4 -c copy -colorspace bt709 -color_primaries bt709 -color_trc bt709 out.mp4
```

**Fix B — on the final encode** (do this every time):

```bash
ffmpeg -i in.mp4 -vf "..." -c:v libx264 -crf 19 -pix_fmt yuv420p \
  -colorspace bt709 -color_primaries bt709 -color_trc bt709 out.mp4
```

HDR sources (PQ/HLG, bt2020) going to SDR platforms need a real tone-map, not tags (R26b in recipes-deliver):

```bash
-vf "zscale=t=linear:npl=203,format=gbrpf32le,tonemap=tonemap=hable:desat=0,zscale=w=1920:h=1080:t=bt709:m=bt709:p=bt709:r=tv,format=yuv420p"
```

Never apply the HDR recipe to SDR footage — it mis-colors it. And never deliver `yuv444p` or full-range RGB to H.264 players: `yuv420p`, limited range, is the contract.

## 3 · Correction before look

The grading order that survives contact with real footage:

1. **White balance** (`colortemperature` or `grayworld`) — neutrals neutral.
2. **Exposure** — waveform: whites around 200–235, blacks near 16 (limited range), nothing clipped.
3. **Contrast** (`eq=contrast`, `curves`) — shape mids.
4. **Saturation** (`eq=saturation` — gently; 1.0–1.15 max before plastic).

Filters: `eq=brightness:saturation:contrast`, `curves=r='0/0.03 1/0.97'` (lifted blacks/rolled highlights), `hue=h=`, `vignette`, `unsharp` (sparingly). `colorchannelmixer` for channel-level surgery.

**Scopes, not vibes:**
- *Waveform* — exposure and clipping (ffmpeg: `ffplay -vf waveform` / extract a frame into any scope-capable tool; or `signalstats` numeric pass).
- *Vectorscope* — saturation and the skin-tone line.
- *RGB parade* — color casts across the frame.

## 4 · LUTs

```bash
ffmpeg -i in.mp4 -vf "lut3d=file='look.cube',eq=saturation=1.05" -c:v libx264 -crf 19 ...
```

- LUTs are **looks** (creative, applied after correction), not correction. A LUT over unbalanced footage bakes the imbalance in.
- `.cube` is the interchange format; one LUT per output — stacking LUTs is how "teal-orange everything" happened.
- Log footage needs its *normalization* LUT (camera-specific) before any creative LUT.

## 5 · Film looks that read as craft

- **Lifted blacks**: `curves=b='0/0.06 1/0.97'` — shadow lift, not gray wash.
- **Rolled highlights**: `curves=r='0/0 0.9/0.87 1/0.97'` (per-channel for warmth).
- **Split-tone**: warm highlights / cool shadows via `colorbalance`.
- **Grain**: `noise=alls=6:allf=t+u` — as much as moods require, less than you think; grain also dithers gradients and kills banding on dark scenes.
- **Halation** (expensive glow on highlights): approximation via a luma-keyed `gblur` screen-blend — use once per project, on the payoff shot.

Teal-and-orange is a *choice*, not a default. If every shot in the folder has it, it's not a look, it's a tic.

## 6 · Color in the pipeline

- Color correction sits **after** the cut is conformed (grading untrimmed footage wastes the encode budget) and **before** caption burn-in (text should stay pure white/black, not inherit the grade).
- One grade pass in the final filtergraph — grade, LUT, vignette chained together with everything else, one encode.
- Verify: extract a frame at a skin-tone moment and a wide shot, look at them at delivery size, and confirm the tags survived (`ffprobe` the output — see R32).
