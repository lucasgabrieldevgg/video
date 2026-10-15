# Generated motion video — brief to rendered, voice-first

*Merged from: vitormenoli/claude-motion-studio-skill (the TTS-first pipeline and its script-sizing rules, TTS tool discipline + PT-BR phonetic spelling table, 124 BPM beat-grid sync, the Canvas→Playwright→FFmpeg stack and its production gotchas, contact-sheet QC, fact-check gate, original-character stand-ins, technique catalog, TikTok zone map + Instagram 4:5 rules — production-validated ruleset). Authorized by the author via Discord DM, October 2026; the upstream repo carries no LICENSE file — see ATTRIBUTION.md. Re-authored for this skill.*

The generation lane: there is no footage to edit — a brief becomes script, voice, music, animated scenes in code, and a final MP4. The discipline is unchanged from editing: judgment lives in ledgers (script, per-word times, beat grid), pixels render once, nothing ships unverified.

The one rule that shapes everything else: **the voice owns the timeline.** The beat grid is a snap-to layer for cuts and sound effects — never the master clock.

## 1 · The pipeline order

1. Script → fact-check gate (§7) → user approval.
2. Generate the voices, per scene (§3).
3. Clean and time-compress each audio (§3).
4. Align: match speech segments to script phrases; compute per-word times.
5. Build the timeline on the beat grid (§4).
6. Generate the synthesized track (fixed seed); mix with ducking (R21).
7. Write the scenes as Canvas/WebGL code (§5).
8. Contact-sheet QC on key frames — loop until clean (§6).
9. Full render → FFmpeg assembly → delivery pass (R33, §10).

## 2 · Script sizing (before any audio exists)

- TTS speaks at **~10 characters/second** — a 60 s script is ~130 words. Over that, either the compression gets unnatural or the video runs long.
- Written time is not real time: ~48 s of text renders **52–58 s** of audio. Budget the slack before promising a duration.
- **Numbers spelled out in the alignment text**: "2026" is 4 characters but "dois mil e vinte e seis" is 23 of speech — alignment math must use the spoken form, not the written one.
- Every animation triggers on the word it illustrates; word durations are proportional to length. This is the same word-onset discipline as `the-pipeline.md` — generation just builds the transcript instead of extracting it.

## 3 · Voice generation (TTS) discipline

- Generate **per scene**, not one blob: scene-level files are the alignment units and the re-roll units (a fact fix re-voices one scene, not the video).
- Clean chain per stem: trim edge silence → cap internal pauses at **0.17–0.3 s** → compress with `atempo` **1.12–1.38×** (pitch-preserving; choose by script overshoot) → normalize the voice stem around **−15 LUFS**. `atempo` outside 0.5–2.0 needs chaining (R08).
- **Transcribe every generated audio back and diff against the script** before building the timeline — TTS mispronunciations are caught here, never in the final render.
- Tool discipline (production-tested): never pass `instruct`-style parameters to cloned voice profiles (HTTP 400, repeatedly); calls can hang ~60 s and then resolve — check the output folder and confirm by transcription before re-rolling.
- PT-BR phonetic spellings that work (extend the table as new names get tested — keep it a growing glossary, like `glossary.md` in `the-pipeline.md`):

| Term | Spelling that works | Term | Spelling that works |
| --- | --- | --- | --- |
| xAI | "xis ei ai" | OpenAI | "Ópen ei ái" |
| Grok Bot | "Grók Bót" | Anthropic | "Ântrópic" |
| Dots | "Dóts" | Google | "Gúgol" |
| Muse | "Miús" | Hugging Face | "Râguin Fêis" |
| Superpowers | "Súper Páuers" | Codex | "Côdex" |
| Opus 5.5 | "Ópus cinco ponto cinco" | Yann LeCun | "Iân Lecân" |
| Reels | "rílis" | Geoffrey Hinton | "Djéfri Rínton" |
| EUA | write "Estados Unidos" (or it spells it out) | IA | "í á" — mid-phrase it vanishes; use "inteligência artificial" |
| Numbers | always spelled out | Auckland | "Ócland" |

## 4 · Beat grid and music

- Synthesized track with a **fixed seed** → re-renders are reproducible; 124 BPM house default (**0.4839 s/beat** — the grid every cut snaps to).
- Every scene starts on a beat; the recap/final screen lasts **8 beats, one topic per beat**.
- Impact SFX on every scene cut; a riser before the turn and the final screen.
- Sound mirrors the visual concept — a trail/echo video gets echoing, decaying SFX; the mix is characterful, not stock.
- SFX derive from the **same per-word times the animation uses** — one timing ledger feeds both picture and sound.
- Ducking and bed levels per `audio.md`; final master to the platform target (−14 LUFS TikTok house default).

## 5 · The render stack — Canvas → Playwright → FFmpeg

The third programmatic lane (with Remotion and MoviePy, `programmatic.md`): scenes are deterministic Canvas (2D/WebGL) functions of time; Playwright screenshots the frames; FFmpeg assembles and mixes. Playwright **1.64.1** (live-verified 2026-10-15). Choose it when the project is plain Canvas with no React — fewer dependencies than a Remotion project, same ledger discipline.

Production gotchas (each one cost a real render):

- `fetch()` does not work under `file://` — embed data as `.js` files, never fetched JSON.
- `@fontsource` latin subsets lack the `→` glyph — draw arrows as paths, not font characters.
- **Register every seed** (tremor, dust, glitch) — a re-render must be pixel-identical, or QC loses meaning.
- Render budget: **~3 fps → 10–16 min per final minute**. Run in background; never block the session waiting — and never start it before the contact sheet is clean (§6).
- 16:9 reference photos cropped to 4:5 upscale ×1.43 — ask for higher-resolution sources up front.

## 6 · Contact-sheet QC (the pre-render gate)

Frame strips verify *after* render (`the-pipeline.md` Phase 5); the contact sheet catches defects *before* the full render costs 10+ minutes. Extract **10–12 key frames** (one per scene + every transition), montage into a single grid, and read it:

```bash
ffmpeg -framerate 30 -i frames/f%04d.png \
  -vf "select='not(mod(n,75))',scale=270:480,tile=layout=4x3" -frames:v 1 sheet.png
```

Typically catches **3–5 problems per video**: text past the margin, a seal cut by the punch-in, an element invisible against same-color background, tremor baked into the drawing (slows the render for nothing), liquid transitions misaligned. Loop: fix → new sheet → render only when clean. Map user attachments **by brief description, not attach order** — they arrive inverted often enough to have a rule.

## 7 · Fact-check gate (before narration)

- In production, **4 of 5 scripts carried a factual error before review**: every number, date, and "who did what" is checked against **2 independent sources**; if sources disagree, fix the text before the voice exists.
- Opinion sold as fact gets softened: "it became the standard" → "it became a reference".
- On-screen numbers carry the source line small underneath: `418 mil · 14/set · via Bloomberg`.
- Corrections go to the user for approval before re-voicing — this is the review gate, applied one pipeline earlier.

## 8 · Original characters, never real logos

The hard rule already bans fabricated logos (`SKILL.md`); this is the positive counterpart: a brand stand-in is a **text seal + an original character doing the same visual job** (the user overlays official art in a editor later if needed). Reuse characters across videos when the theme repeats — consistency is a feature, and so is the character table (a pulsing 3-dot orb for the assistant, a screen-bot that jumps and waves, a living drop that points and blinks: simple rigs, animated physics, memorable silhouettes).

Character physics: squash/stretch on the jump, **appendages lagging** (antenna, cape — follow-through), **arcs with trails**, elastic settle on arrival. Full craft treatment (anticipation, spacing, timing curves) lives in the `motion` skill's craft reference — load it when scenes get complex.

## 9 · Technique catalog (condensed)

| Family | The moves |
| --- | --- |
| Hooks (opening) | freeze-to-motion (the frozen frame *is* the break pose — confirm with `freezedetect`); slice-break offset by 4 frames; a giant number entering before the rest of the text |
| Kinetic type | mask-reveal line-by-line, no fade (the default); decode letter-by-letter; typewriter with balanced line breaks; keyword **slam** with tremor; letters entering one-by-one (names); diagonal strike for negation |
| Camera | punch-in 8–15% on every cut, settling ~0.5 s; continuous slow push; 1-frame solid-color flash on cuts; seed-registered tremor on impacts |
| Data/stats | count-up that decelerates onto the real number; exponential growth (1 → 10.000); fan of pages; masked spreadsheet; dot network with cables snapping + sparks; RGB-split slice glitch |
| Fake 3D (no WebGL) | hand-projected wireframe, back edges weaker; hand-rotated cube; glass drawn back-to-front; glassmorphism via gradient + light border + reflection; light cone with falling intensity + suspended dust |
| Trail/echo | the object drawn at past times with geometric opacity falloff — one function powers motion trails *and* spatial echoes (any "copying/repeating" concept) |
| Mixed media | torn paper with organic (not serrated) edges; halftone dots; risograph ghost; taped stickers on fixed frames |
| Ready-made metaphors | liquid fusion of two seals; scrambled-text glitch for URL swaps; viewfinder + flash for "screenshot"; card flip for before/after; sonar rings for "running in background"; limping letters for error; a dot running along a connection line |

## 10 · Social delivery rules (generation-specific)

**TikTok/Reels/Shorts (9:16, 1080×1920)** — granular zones beyond the caption safe area (`captions.md` §6): top `y ≤ 240` platform header; main content `y ≈ 260–1450/1550`; below ~1550 the caption bar; right of `x ≈ 920–940` the button rail; **80 px margin, 900 px content column** as the Swiss-grid base; every text block auto-fits width, never overflows.

**Instagram feed/carousel (4:5, 1080×1350, 60 fps, H.264 yuv420p, ~−16 LUFS):**

- **Frame 0 is the final scene** — it becomes the thumbnail/grid frame, and the loop must close without a visible jump.
- The **CTA keyword stays clean** (no glued punctuation or emoji) — DM automation matches the exact text.
- The last slide is never "swipe" — nothing comes after it; the arrow points at Save.
- Emojis become **drawn icons in the post palette** — colored emoji break the palette.
- Audio fixes are audio-only: `loudnorm` then remux with `-c:v copy` (`audio.md` §1) — never re-render video to fix a mix.

Export budget: **<20 MB (CRF 21–22)**; keep the project source `.zip` next to the MP4 — pipelines get reused, not rebuilt.

## References for the look

Video beats stills for timing references (scene durations, transition types, layer counts); stills serve palette and composition only. One **technique** per reference — never copy a whole layout — written up as a card: name, what it communicates, Canvas implementation, which scene. Pinterest blocks automation and needs login: the user drops 5–10 screenshots into `assets/refs/`; the agent reads those, never logs in. Where real coded motion lives: Codrops (best for Canvas/WebGL with open code), Behance/Dribbble motion galleries, Vimeo Staff Picks, studio accounts (Buck, ManvsMachine, Ordinary Folk). Taste snapshot (Oct 2026): type with physical weight on rigid grids, short palettes, grain, morphs, seamless transitions, 2D+3D mixes, native vertical.
