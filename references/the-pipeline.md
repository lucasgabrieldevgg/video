# The pipeline — transcript-driven editing for agents

*Merged from: nateherkai/hyperframes-student-kit (cut-silences, cut-mistakes: EDL + review gates), Agentchengfeng/chengfeng-videocut-skills (five-pass scan, dictionary-first correction, ledger separation — concepts), 6missedcalls/video-editing-skill (operation chaining). Re-authored for this skill.*

The agent-native way to edit: **judgment lives in ledgers, pixels render once.** Three ledgers, one timeline:

- **transcript** — *what was said* (word-level, source timeline)
- **EDL** — *what plays* (keep/delete ranges, edit timeline)
- **subtitles** — *what to show* (word ids + display text; seconds are always recomputed, never stored)

A ledger edit is milliseconds and reversible; a rendered file is minutes and destructive. Everything below defers rendering to the last responsible moment.

## Phase 0 · Probe

Before touching anything:

```bash
ffprobe -v error -print_format json -show_format -show_streams input.mp4
```

Audit four things, because these inputs lie:

| Check | Lie it catches |
| --- | --- |
| `r_frame_rate` vs `avg_frame_rate` differ → VFR | screen recordings, phone clips; filters desync on VFR |
| `side_data_list` rotation 90/270 | phone portrait footage stored as landscape |
| `color_space` / `color_transfer` = "unknown" | will ship washed out after re-encode |
| audio: sample rate, channels, `pcm_*` in MP4 | mux failures downstream |

Normalize the pathological cases first (recipes R27 VFR, `ffmpeg.md` §rotation) — fixing them after cutting doubles the re-encode count.

## Phase 1 · Transcribe (the timeline is born)

Word-level, always. Segment-level timestamps are a summary, not a timeline.

```bash
# fastest useful path (English)
whisper audio.wav --model tiny.en --word_timestamps True --output_format json
# non-English
whisper audio.wav --model base --word_timestamps True --output_format json
# GPU / bulk
faster-whisper / whisper.cpp per programmatic.md
```

Model ladder: `tiny.en` for candidate finding and short clips · `base` default · `medium`/`large` for long or noisy sources, and for the final verification pass.

**Correct the transcript before judging it** (dictionary-first). ASR mangles names, products, and jargon; judgments made on wrong words are wrong cuts. Maintain a glossary file (`glossary.md`: `heard → correct` pairs, product names, speaker-specific spellings). Rules: correct *spelling*, never *times*, never *word counts* — downstream word ids must stay stable. If the dictionary can't fix it and the speaker has a script, align to the script; if neither, report it — a guessed name is worse than a heard-wrong name.

**Extract the audio right:**
```bash
ffmpeg -i input.mp4 -vn -ac 1 -ar 16000 audio.wav
```

## Phase 2 · Plan (the EDL)

One JSON, the plan for the whole edit:

```json
{
  "source": "raw.mp4",
  "keep": [ {"start": 0.0, "end": 12.40, "reason": "hook"} ],
  "cuts":  [ {"start": 12.40, "end": 14.10, "reason": "retake of next line", "risk": "high"} ],
  "params": {"gap": 0.55, "breath": "auto"}
}
```

### Silence pass — mechanical

Default thresholds (talking-head YouTube): trim pauses **≥ 0.55 s**, keeping a natural breath — **0.24 s** for long pauses (≥2 s), **0.20 s** after sentence enders, **0.14 s** otherwise, biased toward the previous phrase. Head dead air → 0.22 s; tail → 0.34 s. This removes 15–20% of a typical raw take. Lower `gap` for punch, raise it to preserve breathing room.

Pauses **below** the threshold are natural speech rhythm — leave them.

### Mistake pass — judgment, review-gated

Scan in **separate passes, one judgment per pass** (quality of one-criterion reading beats six-criteria reading):

1. **Repetition** — most objective; said twice, adjacent.
2. **Abandoned phrases / restarts** — half-sentence that restarts.
3. **Misspoken then corrected** — wrong number/name/word, later restated right.
4. **Stutters** — immediate word repeats ("the the", "I- I-").
5. **Fillers** — most subjective; density is a taste decision.

Then **the gate**: candidates are surfaced with full context and a reason, and decisions are two-state (cut / keep) — the agent's reason never creates a third "suggested" state. Read every candidate in context: emphatic repetition ("never, never hands-off") is mechanically identical to a stutter, and only reading saves you from cutting emphasis the speaker meant. Default for misfires: keep the *later*, fluent take; keep one-breath runs as one take; don't cut natural speech to hit a quota — zero cuts on a clean take is a valid outcome.

Present the two tables (cut ledger + repeated-sentence ledger showing "said 3×, keeping #3") and get approval before rendering.

### Moment pass (for shorts)

From the transcript, scan for: punchlines and reactions ("wait", "no way", laughter), reversals (setup question → unexpected answer), awkward pauses, quotable one-liners, audio peaks (`volumedetect`), rapid back-and-forth (alternating short segments). Propose 3–5 candidates as `[start, end, why-it-works, suggested title]`, 10–25 s each. **Verify each candidate is actually in frame** — extract one frame per candidate time and look; the sound of an event is not proof the camera saw it. Build three materially different rough openings and choose with a recorded reason.

## Phase 3 · Conform

Render the plan in **one** filtergraph pass (`trim`/`atrim` + `concat`), A/V in sync:

```
ffmpeg -i raw.mp4 -filter_complex_script edl.filter.txt -c:v libx264 ... cut.mp4
```

Then **re-time the transcript onto the edited timeline** — deterministic arithmetic over the EDL (insertion-only changes shift words by the exact trimmed duration). This re-timed transcript feeds captions, graphics timing, and beat-sync validation. Never re-transcribe to get edited-timeline times; never trust un-re-timed times after a cut.

Air repair belongs here: where a gap reads as a stutter (≈10× tighter than the speaker's median), insert 4–8 frames of real room tone — donor slice from the track's own silence, 10 ms fades, 40 ms equal-power crossfade into the tail's own decay. Digital silence (`anullsrc`) is an audible dropout; hard butt-splices amputate 60–120 ms decays.

Re-transcribe the rendered cut when the source was messy, and diff it against the expected text — Whisper's own output hides restarts, so the cut must be re-read by a fresh pass.

## Phase 4 · Enhance (order matters)

1. **Audio**: denoise (if needed) → loudnorm two-pass → music bed + ducking → SFX layer. Voice is normalized *before* SFX levels are set; SFX are measured against the finished voice.
2. **Captions**: from the re-timed transcript; styled per `captions.md`; face-safe; longest line frame-checked.
3. **Color**: flags explicit → correction → look. (`color.md`)
4. **Graphics**: timed to word onsets ±2 frames, 0.15–0.25 s anticipation; one spotlight; no cross-fades on full-bleed layers.

## Phase 5 · Verify & deliver

The ship gate — all of it, every time:

- **Frame strips** across *every* transition (cut, dissolve, reframe, section seam):
  ```bash
  ffmpeg -i render.mp4 -vf "select='between(n,A,B)',scale=480:270,tile=layout=6x7" -frames:v 1 strip.png
  ```
  Defects live *between* beat frames: a jump into a move, a freeze then a second motion, a one-frame pop. Read the strip.
- **Fresh ASR** over the final audio, diffed against the expected text — catches amputated words and stutter-cliffs the transcript can't see.
- **Measured loudness** (`loudnorm` print or `ebur128`): integrated LUFS + true peak, written down.
- **Probed output**: duration, streams, fps, color tags, faststart flag.
- **Caption longest-line frame**: extracted and looked at, at phone scale.
- **Report**: real numbers (removed s/%, LUFS, TP, hashes of the final file), inspected artifacts, unresolved limits. A pass never comes from code inspection or file existence.

Deliver: final MP4 at platform spec (R32), the `.srt` sidecar, a thumbnail frame (R31 extraction), and the editable project (EDL, transcripts, reel/beat plan) — drafts are never labeled final.

## Failure modes that created these rules

- Cutting on Whisper *segment* times instead of word times → clipped words.
- Judging from `transcript.json` hand-assembled instead of playback order → "said twice" misread from skipped ids.
- Submitting EDL increments instead of the full ledger → silently lost cuts.
- Rendering between every phase → generation loss; one filtergraph, one final encode.
- Trusting a zero-candidate mistake report → an obvious retake survived every scanner; a human eye on the rough cut catches it.
- Fixing audio *after* SFX, or lifting the finished mix → re-do the chain in order.
