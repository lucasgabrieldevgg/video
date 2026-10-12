# video

A big skill for **professional video editing by AI agents** — born from merging the best video-editing skills found on GitHub (Remotion's official agent skills included, as method credit) into one craft-first discipline: judgment lives in ledgers, pixels render once, nothing ships unverified.

```
The transcript is the timeline. The EDL is the plan.
Silence is mechanical; mistakes are judgment — gated, never auto-cut.
One filtergraph, one final encode, and a frame-strip before "done".
```

## What's inside

| Piece | What it does |
| --- | --- |
| **The edit** (`references/the-edit.md`) | The 70%: Murch's priority order, the full cut vocabulary (J/L/match/jump/smash/cross-cut), continuity grammar (180°/30°/eyeline/cut-on-action), pacing by format, retention mechanics (first-3-seconds gate, spatial open loops), attention discipline (one spotlight), talking-head editing, the 14-item editing-slop death list |
| **The pipeline** (`references/the-pipeline.md`) | The agent-native method: probe → word-level transcript (dictionary-first) → EDL with silence rules and the five-pass review-gated mistake scan → one-pass conform → re-timed transcript → enhance in order → the verification gate (frame strips, fresh ASR, measured loudness) |
| **ffmpeg mastery** (`references/ffmpeg.md`) | Stream copy vs re-encode decision tree, seeking semantics, GOP/keyframes, three concat mechanisms, speed math, SAR/DAR, filtergraph hygiene + escaping, delivery encoding (H.264/HEVC/AV1, CRF, faststart, hwaccel), the classic bug gallery, proxies |
| **Audio** (`references/audio.md`) | Platform loudness table (−14/−16/−23 LUFS), two-pass loudnorm, sidechain ducking, bed/SFX level discipline vs voice p95, denoise ladder, room-tone air repair, J/L cut mechanics, dub-with-bed |
| **Color** (`references/color.md`) | The four color questions, the washed-out bug and both fixes (copy re-tag / re-encode), HDR→SDR tone-map, correction-before-look order, scopes, LUTs, film looks that read as craft |
| **Captions** (`references/captions.md`) | Burn vs soft decision, libass discipline (verify, escape, calibrate, longest-line frame), three house styles + karaoke, timing rules (word onsets, reading speed), CJK rules, face-safe placement, platform safe areas |
| **Programmatic** (`references/programmatic.md`) | When code beats CLI: Remotion studio-first (Caption type, render/`--frames`), MoviePy assembly, faster-whisper, auto-editor, server-side scale — versions live-verified |
| **Cut recipes** (`references/recipes-cut.md`) | R01–R16: probe, trims, silence→EDL, review-gated mistake cuts, concat ×2, J/L cuts, speed + ramps, 16:9→9:16 (three strategies + split-screen), freeze/zoom punch, boomerang, timelapse, multicam FFT sync, scene detection |
| **Deliver recipes** (`references/recipes-deliver.md`) | R17–R32: transcription→SRT, caption burn/soft-mux, two-pass loudnorm, bed+ducking, denoise, SFX layer, grade/LUT, washed-out fix, VFR/deinterlace, watermark/PiP, chroma key, thumbnail/GIF, platform presets + the full verification gate |

Plus a Cursor rules export at `exports/video.mdc`.

## Install

**Claude Code / GLM / any SKILL.md host:**
```bash
git clone https://github.com/lucasgabrieldevgg/video.git ~/.claude/skills/video
```

**Cursor:** copy `exports/video.mdc` to `.cursor/rules/video.mdc`.

**ClawHub / skills.sh:** the folder is standard skill layout — publish as-is.

## The method (why craft is 70%)

Editing is judgment before it is tooling: which take lives, where the breath stays, what the first three seconds promise and what the last frame pays off. The tools (ffmpeg, whisper, Remotion) are deterministic — which is exactly why the transcript-and-ledger method works for agents: word-level transcripts make timing computable, EDLs make plans reversible, review gates keep judgment where it belongs, and frame-strip verification replaces "looks about right" with "I looked at the frames". The slop death list exists because AI-edited video has its own tells — crossfades everywhere, captions on faces, zero-gap stutters, center-cropped interviews, and "verified" claimed from code that never rendered a single frame. The recipe catalog serves the loop; it never replaces it.

## Verification

The toolchain moves fast. Package versions in `references/programmatic.md` (remotion 4.0.535, moviepy 2.2.1, faster-whisper 1.2.1, openai-whisper 20250625, auto-editor 29.3.1, yt-dlp 2026.8.19) were live-checked against npm/PyPI on 2026-10-13. ffmpeg filter behavior (xfade offsets, loudnorm two-pass, libass PlayRes scaling) carries inline caveats — re-verify against your local build (`ffmpeg -filters`, `ffmpeg -version`) before relying on a filter.

## Attribution

`video` is an original synthesis (MIT) merging upstream material from `nateherkai/hyperframes-student-kit` (MIT), `kurbaitaev/ghost-editor` (MIT), `louisedesadeleer/clipify` (MIT), `6missedcalls/video-editing-skill` (MIT), `jianshuo/claude-skills` (MIT), `AgriciDaniel/claude-shorts` (MIT), `haidrrrry/claude-remotion-skill` (MIT), `video-db/skills` (MIT), with method concepts from `remotion-dev/skills` (official, credited), `Agentchengfeng/chengfeng-videocut-skills` (Apache-2.0), `oil-oil/oil-subtitle` (credited), and discovery via `zhuyansen/awesome-claude-video-skills` (CC0), plus original recipe authorship and live version verification. See [`ATTRIBUTION.md`](ATTRIBUTION.md).

## License

MIT for this repo's original content. Upstream material retains its own licenses — see ATTRIBUTION.md.
