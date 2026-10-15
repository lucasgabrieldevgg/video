# Attribution & Sources

`video` is an original synthesis commissioned for this repo: the craft core and pipeline method were merged and rewritten from upstream skills, all ~33 recipes were re-authored for this skill's dual-stack (ffmpeg CLI + Python) format, and package versions were live-verified against npm and PyPI (13 October 2026; the generated-motion track and playwright version on 15 October 2026). The synthesis, structure, recipe code, and merged rule set are MIT. Upstream material retains its original license.

## Upstream skills (merged)

| Source | What was merged | License |
| --- | --- | --- |
| `nateherkai/hyperframes-student-kit` — cut-silences, cut-mistakes, video-storytelling, short-form-edit | The EDL + re-timed-transcript pipeline shape, silence thresholds and breath-preservation values (0.55 s gap; 0.14–0.24 s breath; 0.22/0.34 s head/tail), the review-gated mistake flow with two-state decisions, the five-pass single-criterion scan concept, first-3-seconds gate, phone safe-area coordinates, open-loop shapes table, spotlight/attention rules, room-tone air repair (10 ms fades, 40 ms equal-power crossfades), frame-strip verification, fresh-ASR-on-final-audio audit | MIT |
| `kurbaitaev/ghost-editor` | Take selection on word times (−0.10..0.15 in / +0.20..0.30 out), Whisper-merges-restarts warning and re-transcribe-the-cut audit, hard-cut calibration data (113/119 transitions, ~3 s median shot, B-roll 20–35%), SFX level targets vs voice p95 (−14/−12/−8/−3 dB), jump-cut masking (0.2 s dissolves, ≤3 punch-ins), meme air rule (0.6–1 s), the −16 ±1.5 LUFS / ≤−0.5 dBTP reel standard, prep-order discipline (voice normalized before SFX) | MIT |
| `louisedesadeleer/clipify` | Dialogue/action/hybrid moment-finding modes, spectral-flux transient detection concept, in-frame candidate verification ("audio detection ≠ visible event"), 16:9→9:16 strategy set (pan-between-faces via ROI motion energy, split-screen speaker swap, center crop), speed-ramp bands (1.5×/2×/4×), atempo chain limits, caption style set (opus/karaoke/minimal), re-whisper the trimmed clip for relative timestamps | MIT |
| `6missedcalls/video-editing-skill` | The operations model (trim / jumpcut / captions / overlay / speed), pipeline operation order, caption style trio incl. word-by-word impact style, Whisper model guidance per source quality | MIT |
| `jianshuo/claude-skills` — wjs-burning-subtitles, wjs-reframing-video, wjs-dubbing-video, wjs-transcribing-audio, wjs-syncing-multicam, wjs-segmenting-video, wjs-overlaying-video, wjs-localizing-video | libass verification-before-promising + static-build fallback, force_style escaping, Fontsize-is-PlayRes calibration with longest-line frame check, burn vs soft-mux decision, SRT line-length discipline and `,mmm` normalization, dub-with-original-as-bed mixing (0.15–0.25, `normalize=0`), MAR-variance active-speaker detection with 5 fps sampling and ≥1.5 s hysteresis, step-function crop as hard-cut feel, zero-detection deterministic fixed-crop fallback, HLG→SDR tone-map chain, one-pass final composite (no cascade) | MIT |
| `AgriciDaniel/claude-shorts` | The longform→shortform interactive pipeline shape (score candidates, snap boundaries to natural cut points), Remotion-rendered animated caption styles (Bold/Bounce/Clean), platform-optimized export framing | MIT |
| `haidrrrry/claude-remotion-skill` | Remotion-as-editing-surface structure (create + edit professional videos in code), motion-graphics reference organization | MIT |
| `video-db/skills` | The server-side perception→understanding→action model (ingest, index, search moments, timeline ops) for the scale section | MIT |
| `vitormenoli/claude-motion-studio-skill` — CLAUDE.md ruleset | The generated-motion track (v1.1.0, `references/generated-motion.md` + R33): the TTS-first pipeline (voice owns the timeline), script-sizing rules (~10 chars/s, ~130 words per 60 s, spoken-form numbers, 48 s text → 52–58 s audio), TTS tool discipline + the production-validated PT-BR phonetic spelling table, 124 BPM beat-grid sync (scene-on-beat, 8-beat recap, fixed-seed tracks, risers, SFX from word times), the Canvas→Playwright→FFmpeg render stack + its gotchas (file:// fetch, @fontsource → glyph, seed registration, ~3 fps budget, 4:5 upscale ×1.43), contact-sheet pre-render QC (10–12 frames, 3–5 defects/video), the fact-check gate (2-source rule, on-screen source lines), original-character brand stand-ins, the technique catalog, TikTok zone map granularity, Instagram 4:5 rules (frame-0 loop, CTA-clean keywords, no-swipe last slide, emoji-as-drawn-icons), <20 MB export + source .zip discipline | No LICENSE file upstream at merge time — **authorized for merge by the author (Vitor Menoli) via Discord DM, October 2026**; this row updates if a license lands upstream |

## Method/concept credits (not code-merged)

| Source | Concept credited | License |
| --- | --- | --- |
| `remotion-dev/skills` (official Remotion Agent Skills) | Studio-first workflow (preview before render; render only on explicit request), the `@remotion/captions` `Caption` type shape, render/still/`--frames` discipline, Mediabunny for in-browser media introspection, SaaS/Lambda rendering path | No LICENSE file in the upstream repo — official public skills; credited as a method source. If the author specifies terms, this table updates |
| `Agentchengfeng/chengfeng-videocut-skills` | Dictionary-first transcript correction (fix the characters before judging the words), the five-pass one-criterion mistake scan, transcript/EDL/subtitle ledger separation (subtitles store word ids + display text, seconds always recomputed), dry-run-before-render exports, revision-discipline lessons | Apache-2.0 (its product-locked implementation was not merged; concepts only) |
| `oil-oil/oil-subtitle` | Glossary/review discipline for ASR correction, punctuation-free caption display, CJK↔Latin boundary spacing, decimal-point preservation rule, chapter-progress concept | No LICENSE file in the upstream repo — credited as a method source |
| `zhuyansen/awesome-claude-video-skills` | Discovery aid (the ~180-repo landscape index that mapped this domain); no content merged | CC0 (the list) |

## Original verification work

- **Version verification**: checked live against npm and PyPI on 2026-10-13 — remotion 4.0.535, @remotion/captions 4.0.535, @remotion/install-whisper-cpp 4.0.535, moviepy 2.2.1, faster-whisper 1.2.1, openai-whisper 20250625, auto-editor 29.3.1, yt-dlp 2026.8.19 — reflected in `references/programmatic.md`; playwright 1.64.1 added 2026-10-15 for the generated-motion lane.
- **Upstream landscape survey**: 12 repositories cloned and studied before synthesis (GitHub search on 2026-10-12/13: "claude skill ffmpeg", "claude skill video editing", "remotion claude skill", "claude skill subtitles/captions", plus the remotion-dev org and ClawHub listings). Generation-focused skills (explainer/promo/AI-video generators) were identified and excluded by scope decision — this skill edits footage, it does not generate it. That scope was deliberately widened in v1.1.0 (15 October 2026): the TTS-first motion-graphics *generation* lane was added from `vitormenoli/claude-motion-studio-skill`, after author authorization, because it is the pipeline-specific complement to this skill's editing core — script-to-rendered-video with the same ledger discipline. Generic AI-video generators remain out of scope.
- **Recipe corpus**: all 33 recipes in `references/recipes-cut.md` and `references/recipes-deliver.md` were written for this repo — merging technique ideas from upstream (ROI motion-energy speaker detection, xfade offset math, two-pass loudnorm, palettegen GIF, room-tone repair; R33's voice-package/beat-grid/contact-sheet commands from motion-studio) with original dual-stack implementations and guard lines.
- **The craft layer** (`references/the-edit.md`) synthesizes upstream editing judgment with standard practice: Walter Murch's priority order from *In the Blink of an Eye*, the Kuleshov effect, continuity grammar (180°/30°/eyeline), and format pacing norms.
- **The structure, routing, hard rules, and wording** are original to this repo.

## Individual credits

- **Nate Herk** — the hyperframes student kit whose pipeline agents (cut-silences, cut-mistakes) and storytelling/short-form skills form this skill's largest merged backbone.
- **kurbaitaev** — ghost-editor's take-selection, calibration, and QA discipline.
- **Louise de Sadeleer** — clipify's moment-finding and reframing strategies.
- **Ian Perez** — the minimal, honest operations skill.
- **Jianshuo Wang** — the wjs video-ops family (subtitles, reframing, dubbing, multicam) and its battle-tested libass/MediaPipe discipline.
- **Daniel Agrici** — claude-shorts' interactive longform→shortform pipeline.
- **haidrrrry** — the Remotion editing-surface skill.
- **The VideoDB team** — the server-side video model.
- **Vitor Menoli** — motion-studio's production-validated TTS-first pipeline: the voice-owns-the-timeline rule, the PT-BR phonetic table, beat-grid sync, and the contact-sheet gate now live in `references/generated-motion.md`.
- **The Remotion team** (Jonny Burger and maintainers) — the official Agent Skills and the framework itself.
- **chengfeng** — the five-pass scan and ledger separation.
- **oil-oil** — the subtitle correction discipline.
- **zhuyansen** — the awesome list that mapped the landscape.
- **Walter Murch** — the priority order every editor in this file quietly obeys.
- The maintainers of FFmpeg, whisper (OpenAI), faster-whisper, MoviePy, auto-editor, and yt-dlp — the toolchain this entire skill stands on.

Any errors in the merged synthesis are this repo's, not theirs.

If you redistribute `video`, keep this file and the per-file credits with it.
