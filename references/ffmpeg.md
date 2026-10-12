# ffmpeg mastery — the scalpel

*Original content for this skill, cross-checked against ffmpeg documentation and battle-tested patterns from the upstream skills (jianshuo/claude-skills, clipify, hyperframes).*

ffmpeg is the editor's scalpel: deterministic, scriptable, and honest about what it does. This file is the concept layer — recipes live in `recipes-cut.md` and `recipes-deliver.md`.

## 1 · Probe discipline

`ffprobe -v error -print_format json -show_format -show_streams` is the only acceptable way to learn a file. Read:

- **`r_frame_rate` vs `avg_frame_rate`** — equal ⇒ CFR; different ⇒ VFR (screen recordings, some phone/DRM rips). VFR must be CFR-normalized (`fps` filter or `-vsync cfr`) *before* any other filter, or everything downstream drifts.
- **Rotation** — `side_data_list` with `rotation: -90`. The file is stored landscape, displayed portrait. Filters act on *stored* pixels; players apply rotation. Decide one representation and stay in it.
- **Color tags** — `color_space`, `color_primaries`, `color_transfer` = `unknown` ⇒ the washed-out bug is one re-encode away (see `color.md`).
- **Audio** — sample rate, channel layout, codec. `pcm_*` inside MP4 is a mux time bomb (remux to MKV or re-encode to AAC).

## 2 · Stream copy vs re-encode — the decision tree

```
Are you only changing container, metadata, or cutting on keyframes?
├─ YES → -c copy (instant, zero quality loss)
└─ NO  → re-encode — but only ONCE, with the full filter chain in ONE pass
```

- `-c copy` cuts **snap to the previous keyframe** — a "-ss 00:01:23" copy-trim actually starts at the keyframe before it (up to `keyint` early, typically ≤2 s).
- Frame-accurate trim, scaling, filters, caption burn-in, transitions ⇒ re-encode. Chain *all* of them in a single `-filter_complex` — three sequential encodes visibly degrade; one pass doesn't.
- `-c copy` **can still write color metadata** (`-colorspace bt709 ...`) because the `colr` box lives in the container — the cheapest fix in this skill (R26).

## 3 · Seeking semantics

- `-ss` **before** `-i`: fast seek, decoder jumps to the nearest keyframe — with re-encode it is frame-accurate (`-accurate_seek` is default); with `-c copy` it snaps backward to keyframes.
- `-ss` **after** `-i`: decodes and discards everything before the point — frame-exact, slow for late timestamps.
- Put `-ss` before `-i` and `-t` after: `ffmpeg -ss 00:02:00 -t 45 -i in.mp4 ...` (input seek + output duration). For surgical sub-second work, seek by exact frame: `-ss` with 6-decimal seconds, computed from word times.

## 4 · GOP, keyframes, and why lossless cuts are coarse

Keyframes (I-frames) are self-contained; B/P frames depend on neighbors. `-g` sets the GOP (keyframe interval); `-keyint_min` the floor. For delivery: `keyint = 2 × fps` (2-second GOP) balances seekability and size. For edit intermediates: all-I (`-g 1`) or long GOP with copy — never re-encode intermediates you'll cut again. `-force_key_frames "0:05,0:12.4"` plants exact cut points for later copy-trims — the trick behind "lossless-ish" precise cutting.

## 5 · Concat — three mechanisms

| Method | Command shape | Works when |
| --- | --- | --- |
| **Demuxer** | `concat` list file + `-c copy` | identical codec, resolution, fps, timebase, audio params — the only true lossless join |
| **Filter** | `-filter_complex "[0:v][1:v]concat=n=2:v=1:a=1"` | heterogeneous inputs (re-encodes; rescale/normalize inputs first) |
| **Protocol** | `cat a.ts b.ts > out.ts` (MPEG-TS only) | legacy TS streams, quick and dirty |

Demuxer requirements are strict: probe both files; if fps, SAR, resolution, or profile differ, the join glitches (frozen frames, A/V drift). Normalize to a common spec first, then concat.

## 6 · Speed math

- Video: `setpts=PTS/2` = 2× (PTS is *presentation* time — dividing speeds up).
- Audio: `atempo` accepts **0.5–2.0 per instance** — 4× = `atempo=2.0,atempo=2.0`. Pitch is preserved; without `atempo`, `setpts` alone gives chipmunks.
- Time-lapse from decimation: `select='not(mod(n,30))',setpts=N/FRAME_RATE/TB` (keep 1 of 30).
- Speed ramps = per-segment constant speeds + concat (R09) — one expression per segment, hard cuts between; a true continuous ramp needs `setpts` expressions with `if(gt(t,X),...)` ladders and is rarely worth it.

## 7 · Scale, crop, and aspect

- `scale=w:h:flags=lanczos` — lanczos for downscaling quality; never upscale beyond 1.2× without admitting it (608×1080 → 1080×1920 is a 1.78× upscale — fine for faces, visible on text).
- **SAR/DAR**: after odd crops, `setsar=1` prevents stretched playback. Compute crops as exact world windows — a "scale 0.6" framing slices words at frame edges; a window `x=140:w=904` doesn't.
- Fit-in-box with letterbox: `scale=w:h:force_original_aspect_ratio=decrease,pad=W:H:(ow-iw)/2:(oh-ih)/2`.
- Odd dimensions break yuv420p (H.264 needs even w/h — always `scale=trunc(iw/2)*2:trunc(ih/2)*2` at the end of a chain with arbitrary crops).

## 8 · Filtergraph hygiene

- **One pass**: every video filter chained comma-separated inside one `-vf`/`-filter_complex`; audio separately with `-af`. Sequential ffmpeg invocations = generation loss.
- `-filter_complex_script file.txt` beats inline for long graphs (EDL trim+concat lives here).
- **Escaping**: filter arguments are parsed at three levels. `subtitles` style commas → `\,`; `drawbox`/`drawtext` colons in text → `\:`; Windows paths need `\` doubled and the drive colon escaped; prefer `subtitles=filename='C\:/path/sub.ass'` form.
- `format=yuv420p` at the end of every chain destined for H.264 — `yuv444` or GBR output plays black on half the players.
- Hardware decode/encode (`-hwaccel videotoolbox` / `h264_nvenc` / `hevc_qsv` / `vaapi`) is ~5-20× faster and slightly worse per bitrate — fine for drafts and proxies, questionable for finals. `-preset ultrafast` for previews, `slow`/`medium` for masters.

## 9 · Delivery encoding

| Target | Video | Audio | Notes |
| --- | --- | --- | --- |
| Web master (H.264) | `-c:v libx264 -crf 18-21 -preset slow/medium -pix_fmt yuv420p -profile:v high` | `-c:a aac -b:a 192-256k` | CRF = quality target; 18 visually lossless, 23 visibly soft |
| Streaming cap | add `-maxrate 8M -bufsize 16M` | — | platform caps: e.g. WeChat Channels 10 Mbps → pass 8M |
| HEVC | `-c:v libx265 -crf 20-24 -tag:v hvc1` | aac/opus | `hvc1` or Apple players refuse it |
| AV1 | `-c:v libsvtav1 -crf 30-38 -preset 6` | opus | ~40% smaller than H.264; slow to encode |
| MP4 final | always `-movflags +faststart` | — | moves the `moov` atom to the front — instant streaming start |

Always ship explicit color flags on encodes: `-colorspace bt709 -color_primaries bt709 -color_trc bt709` (see `color.md`).

## 10 · The classic bug gallery

| Bug | Symptom | Fix |
| --- | --- | --- |
| VFR input into filters | progressive A/V desync, wrong durations | `-vsync cfr -r 30` (or `fps=30` filter) before anything else |
| Rotation metadata | sideways output after crop/scale | `-noautorotate` or pre-rotate with `transpose`, then drop the tag |
| Non-monotonic DTS | "Application provided invalid, non monotonically increasing dts" | `-fflags +genpts` on input |
| Washed-out upload | gray shift on YouTube/VLC, fine locally | explicit color tags on encode (R26) |
| PCM in MP4 | player-dependent silence / mux errors | `-c:a aac` or remux MKV |
| SRT with `.` ms separators | some parsers drop cues | normalize to `,mmm` |
| Caption hours off after EDL | subtitles late by exactly the trimmed time | re-time the transcript (Phase 3) — never hand-shift |
| `-t` after `-c copy` | cut drifts to keyframe | accept keyframe snap, or re-encode for frame accuracy |
| `yuv444p`/GBR output | black video on phones/web | `format=yuv420p` terminal filter |
| amix auto-attenuation | bed+voice quieter than expected | `amix=normalize=0` and set gains explicitly |

## 11 · Proxies and batch

For long sources or repeated experiments: render a proxy (`-vf scale=-2:540 -c:v libx264 -crf 28 -preset ultrafast -g 50`) once, plan against it, then **conform from the original** at the end (same EDL, full-resolution source — build final face crops from native resolution, never a downscaled proxy). Batch = one shell loop, one job list JSON, `--dry-run` first pass printing the plan (durations, sizes, commands), then execute — the same plan-then-render discipline as the EDL.
