# Audio — the half everyone underestimates

*Merged from: kurbaitaev/ghost-editor (SFX targets, prep-order discipline, calibration), AgriciDaniel/claude-shorts (bed levels), nateherkai/hyperframes-student-kit (room-tone repair, acoustic-level auditing), jianshuo/claude-skills (dub/bed mixing), vitormenoli/claude-motion-studio-skill (TTS stem treatment chain — authorized by the author). Re-authored for this skill.*

Order of operations is law: **denoise → normalize the voice → bed and ducking → SFX measured against the finished voice → final master verify.** Fixing levels after SFX, or lifting the finished mix, undoes everything before it.

## 1 · Loudness targets (measured, never guessed)

| Target | Integrated | True peak | Notes |
| --- | --- | --- | --- |
| YouTube / Spotify | −14 LUFS | −1 dBTP | both platforms normalize to −14 anyway |
| Reels / TikTok / Shorts | −14 to −16 LUFS | −1 dBTP | platforms re-normalize; aim mid-range, keep headroom |
| Apple Music / podcasts | −16 LUFS | −1 dBTP | the comfortable speech standard |
| EBU R128 broadcast | −23 LUFS | −1 dBTP (−2 for lossy) | EU TV delivery |
| Talking-head reel default | −16 ±1.5 LUFS | ≤ −0.5 dBTP | the ghost-editor house standard |

Two-pass loudnorm — the transparent way (pass 1 measures, pass 2 applies *linear* gain, no pumping):

```bash
# pass 1: measure (read measured_I, measured_TP, measured_LRA, measured_thresh from stderr)
ffmpeg -i in.mp4 -af loudnorm=I=-14:TP=-1:LRA=11:print_format=summary -f null -
# pass 2: apply with the measured values + linear=true
ffmpeg -i in.mp4 -af loudnorm=I=-14:TP=-1:LRA=11:measured_I=-18.6:measured_TP=-2.1:measured_LRA=5.3:measured_thresh=-29.8:linear=true -c:a aac -b:a 192k out.mp4
```

One-pass `loudnorm` is dynamic normalization — it pumps. Use it only when the source dynamics are unusable. For a simple gain trims: `volume=6dB` (or `loudnorm` off). Never fix loudness by re-encoding the video: `-c:v copy` and touch only the audio stream.

## 2 · The voice is the master

- Normalize the voice **in prep, before any SFX exists**. Every later level is measured *relative to this voice* — the reference is the voice's p95 peak, not a number you typed.
- **Generated (TTS) voices enter the same way**: per-scene stems cleaned (edge trim, 0.17–0.3 s pause cap), `atempo` 1.12–1.38× compressed, normalized ~−15 LUFS, and transcribed back before any timeline math — full treatment in `generated-motion.md` §3.
- Duck the bed under **every** spoken line — not just the loud ones: `sidechaincompress` (R21). Under a freeze or a payoff, the bed nearly disappears.
- Bed level: start 14–20 dB under the voice, then adjust to the actual track. "Music too loud" is fixed by lowering the bed *relative to the reviewed version* — verify the ratio changed, not just the master level.
- `amix=normalize=0` always — the default auto-attenuation silently drops both inputs ~3 dB when they overlap.

## 3 · SFX discipline

SFX are built from the visual-event map, not sprinkled: paper impacts land on object contact, swipes support movement, success tones support checks.

| Role | Level vs voice p95 |
| --- | --- |
| UI / subtle | ≈ −14 dB |
| Whoosh / swipe | −12 dB |
| Impact / landing | −8 dB |
| Meme / stinger | −3 dB |

- Align the **audible onset** (not the file start — files carry leading silence) within 2 frames of the visual landing.
- Never place a sound **on top of a word** — a meme after a line needs the take extended 0.6–1 s so the sound has air.
- No sound effect twice in one deliverable; 1–3 memes per minute is the ceiling; some cuts pass without a whoosh.
- Let a bed dip before a contact instead of raising the contact — dynamics beat loudness.

## 4 · Noise, room tone, and air

- **Denoise before anything else listens well.** ffmpeg `afftdn` for steady noise; `arnndn` (RNN models) for heavier cases; DeepFilterNet3 for speech (cap at 20 dB reduction — full strength sounds thin and watery).
- **Room tone is a donor organ.** Repair thin gaps with a slice of the track's own silence: 10 ms edge fades, spliced into the word tail's own decay with a 40 ms equal-power crossfade. Hard butt-splices amputate 60–120 ms natural decays and are heard as stutters; `anullsrc` digital silence is heard as a dropout against a −50 dBFS noise floor.
- **Keep the breath.** Sentence-boundary gaps ~10× tighter than the speaker's median are the "stutter" clients report. The ear flags envelope cliffs that a transcript-derived check will report as clean.
- **Audit the audio acoustically, not textually.** In yield order: whole-track dead-span scan (runs of digital-zero inside pauses → fill with room tone), 1 ms-resolution envelope forensics at every join, and a fresh ASR pass over the delivered audio diffed against expected text. What clients call "stutters" is usually truncation cliffs and dead holes, not doubled words.

## 5 · Dialogue mechanics (J and L cuts)

The emotional power of an edit often lives in audio leading or lagging the picture:

```bash
# L-cut: voice of shot A continues 1.5 s over shot B (B-roll lands early)
ffmpeg -i a.mp4 -i b.mp4 -filter_complex \
  "[0:v]trim=0:10,setpts=PTS-STARTPTS[v0]; \
   [1:v]trim=0:8.5,setpts=PTS-STARTPTS[v1]; \
   [0:v][1:v]concat=n=2:v=1:a=0[v]; \
   [0:a]atrim=0:11.5,asetpts=PTS-STARTPTS,apad=whole_dur=18.5[a]" \
  -map "[v]" -map "[a]" lcut.mp4
```

J-cut is the mirror (B's audio starts before B's picture — `adelay` B's audio, `amix` under A's tail). Use 0.5–2 s of lead/lag; longer reads as an error. See R07 for the full recipe.

## 6 · Voice replacement and dubs

- Keep the original as a **bed** under a dub (0.15–0.25 gain ≈ −16 to −12 dB): breath, emphasis, and laughter survive under the new voice, which is what "professional translation" sounds like.
- Match loudness of the dub to the original's LUFS before mixing, not after.
- One ffmpeg pass does dub + burn + bed (R18/R21 composed — never a cascade).

## 7 · The mix gate

Before "done": integrated LUFS and true peak measured and written down; bed-to-voice ratio recorded (and reduced if the user said "too loud"); no clip in any stem; the last syllable of the last word survives the final fade; a fresh ASR pass still reads the final mix cleanly.
