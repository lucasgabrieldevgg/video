---
name: video
description: "Professional video editing for AI agents: transcript-driven and craft-first. TRACK 1 - CRAFT (the 70%): the grammar of the cut (hard cut, J-cut, L-cut, match cut, montage, cross-cutting), Murch's priority order (emotion > story > rhythm > eye-trace), pacing by format (long-form talking-head vs vertical short), retention mechanics (the first-3-seconds gate, open loops, pattern interrupts), attention discipline (one spotlight, eye-trace continuity), talking-head editing (cut on word times, breath preservation, jump-cut masking), and an editing-slop death list. TRACK 2 - THE PIPELINE: probe-first ffprobe audits (VFR, rotation, color tags), word-level transcripts as the timeline, EDL-driven cuts with review gates (silence is mechanical, mistakes are judgment), one-pass ffmpeg filtergraphs, audio finishing (two-pass loudnorm to platform LUFS targets, sidechain ducking, SFX level discipline), color (space flags, the washed-out bug, LUTs, scopes), captions (SRT/ASS/VTT, burn vs soft, libass discipline, face-safe placement, CJK rules), programmatic paths (Remotion studio-first, MoviePy, faster-whisper, auto-editor), platform delivery specs, and ~32 dual-stack recipes (ffmpeg CLI + Python), each with its guards. Hard rules throughout: probe before you cut, the transcript is the timeline, one re-encode budget, loudness is a delivery requirement, captions are not optional, verify by frame strips and fresh ASR before shipping. Use for ANY video editing task: 'cut this', 'remove the silence / mistakes', 'make clips / shorts / reels from this', 'add captions', 'fix washed-out color', 'normalize the audio', 'reframe to vertical', 'sync multicam', 'concat with transitions', 'export for YouTube / TikTok / WeChat', 'make a video from code or data' - even when the user just says 'make this watchable'."
version: 1.0.0
license: MIT for the synthesis; upstream content keeps its own licenses - see ATTRIBUTION.md
---

# video — editing as a craft, executed through a ledger

`video` edits the way a professional editor thinks and the way an agent executes: judgment lives in ledgers (transcript, EDL, subtitle text), pixels are rendered once, and nothing ships unverified.

## The edit contract

1. **Serve the story, then the rhythm.** Every cut answers emotion, story, or rhythm — in that order (Murch: emotion 51%, story 23%, rhythm 10%, eye-trace 7%, plane 5%, space 4%). A cut that answers none of these is filler and gets removed.
2. **The transcript is the timeline.** A word-level transcript plus the EDL is the single source of truth. Every timing decision lands on word onsets (±2 frames), never on segment guesses — and never on Whisper's sentence-level timestamps.
3. **Probe before you touch.** `ffprobe` JSON first: frame rate (constant or VFR?), rotation metadata, color tags, audio layout. Screen recordings and phone files lie about all four.
4. **Plan in ledgers, render once.** An EDL edit costs milliseconds; a rendered file is irreversible. Dry-run the plan, gate the judgment calls, then conform in ONE filtergraph pass. Decode-re-encode cascades are a quality bug, not a style.
5. **Judgment is gated, mechanics are not.** Silence removal is mechanical. Mistake removal is judgment — stutters look identical to emphatic repetition. Surface candidates with context and reasons; never auto-cut emphasis, or a retake the speaker meant.
6. **Audio is half the perceived quality.** Loudness normalized to the platform target (two-pass, measured), music ducks under every spoken line, SFX land on impacts never on words, and breath is preserved with real room tone — zero-gap joins are heard as stutters.
7. **Captions always ship.** Most feed viewing is muted; captions are a default-on deliverable. They never cover the face, sit inside platform safe areas, and the longest line is verified by an actual frame before render.
8. **Verify what you ship.** Contiguous frame strips across every transition, a fresh ASR pass over the final audio, measured loudness, and a probed output file. No pass is ever claimed from code inspection or file existence.

## Routing table

Read only what the task needs:

| When | Read |
| --- | --- |
| Choosing cuts, pacing, structure, retention hooks — or cutting editing slop | `references/the-edit.md` |
| Running the pipeline: probe, transcribe, EDL, review gates, conform, verify | `references/the-pipeline.md` |
| ffmpeg concepts: stream copy vs re-encode, seeking, concat, speed, filtergraphs, codecs, delivery encoding, classic bugs | `references/ffmpeg.md` |
| Loudness targets, two-pass loudnorm, ducking, denoise, SFX levels, room tone repair | `references/audio.md` |
| Color spaces and flags, the washed-out bug, LUTs, scopes, grading order, HDR→SDR | `references/color.md` |
| Captions: formats, burn vs soft, libass discipline, styles, timing rules, CJK | `references/captions.md` |
| When code beats CLI: Remotion, MoviePy, faster-whisper, auto-editor, server-side | `references/programmatic.md` |
| Cut & assemble recipes (R01–R16, dual-stack) | `references/recipes-cut.md` |
| Enhance & deliver recipes (R17–R32, dual-stack) | `references/recipes-deliver.md` |

## The one-glance task matrix

| Task | First call |
| --- | --- |
| Remove silence / dead air | `silencedetect` → EDL → trim+concat (R03) |
| Remove stutters / retakes / false starts | transcript scan, review gate, then cut-mistakes flow |
| Long video → shorts / reels | moment scan → active-speaker reframe (R10) → captions (R18) |
| Add captions | transcribe (R17) → burn or soft-mux (R18/R19) |
| Video looks washed out on YouTube | color-flag audit → re-tag bt709 (R26) |
| Too quiet / too loud / inconsistent | two-pass loudnorm (R20) |
| Join clips with transitions | xfade + acrossfade, offset math (R06) |
| Convert 16:9 ↔ 9:16 | reframe strategy table (R10) |
| Sync multicam angles | FFT cross-correlation (R15) |
| Speed ramp / timelapse | segment setpts + atempo (R08/R09) |
| Video from code, data, or a template | Remotion, studio-first (`programmatic.md`) |
| Export for a platform | delivery presets + verification (R32) |

Rule of thumb: if the task is expressible as one ffmpeg filtergraph over known inputs, it is a CLI job. The moment composition logic, word-timed graphics, or data-driven rendering enters, it is a Remotion job. MoviePy sits in between for scriptable assembly of modest clips.

## The loop in one breath

**Probe** (ffprobe JSON; audit VFR / rotation / color / audio) → **Transcribe** (word-level; correct the dictionary errors before judging) → **Plan** (EDL with reasons; silence mechanically, mistakes through the review gate; moment selection for shorts) → **Conform** (one filtergraph pass; re-time the transcript onto the edited timeline) → **Enhance** (audio → captions → color → graphics, in that order) → **Verify** (frame strips across every transition, fresh ASR, measured loudness, probed output) → **Deliver** (platform spec, captions sidecar, thumbnail, report with real numbers).

## Hard rules

- Never assume fps, rotation, aspect, or color space — the probe shows them, and VFR input gets CFR before any filter.
- Stream copy when parameters allow; otherwise exactly one re-encode, with the whole filter chain in one pass.
- Color metadata is explicit on every encode (`-colorspace bt709 -color_primaries bt709 -color_trc bt709`) — unspecified tags are how videos ship washed out.
- Loudness is a delivery requirement, measured: −14 LUFS web, −16 short-form, −23 EBU broadcast; true peak ≤ −1 dBTP.
- Captions: never over the face; inside the safe area (1080×1920: x 90–930, y 200–1600); longest line verified by frame.
- The bed ducks under speech; SFX hit on impacts, never on top of a word; a meme or stinger needs 0.6–1 s of air after the line.
- Dead air is repaired with real room tone and 40 ms equal-power crossfades — digital silence between words is an audible dropout.
- Every cut on a word is `first word − 0.10..0.15 s` in, `last word + 0.20..0.30 s` out; breath of 0.14–0.24 s survives every trim.
- Nothing is called done without: contiguous frame strips at every transition, a fresh ASR diff on final audio, measured LUFS, and a probed output.
- Never fabricate: no invented logos or brand marks, no synthetic "evidence", no manufactured retention or engagement claims.

## Sources

This skill is an original synthesis (MIT) merging upstream skills — nateherkai/hyperframes-student-kit, kurbaitaev/ghost-editor, louisedesadeleer/clipify, 6missedcalls/video-editing-skill, jianshuo/claude-skills, AgriciDaniel/claude-shorts, haidrrrry/claude-remotion-skill, video-db/skills, plus method concepts from remotion-dev/skills (official), Agentchengfeng/chengfeng-videocut-skills, and oil-oil/oil-subtitle. Full credits and licenses: `ATTRIBUTION.md`.
