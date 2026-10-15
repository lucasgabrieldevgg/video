# Programmatic editing — when code beats the CLI

*Merged from: remotion-dev/skills (official agent skills — studio-first workflow, Caption type, render discipline; method credit), AgriciDaniel/claude-shorts (Remotion caption components), haidrrrry/claude-remotion-skill (skill structure), video-db/skills (server-side perception model), vitormenoli/claude-motion-studio-skill (Canvas→Playwright→FFmpeg lane — authorized by the author). Versions live-verified 2026-10-13; playwright 2026-10-15.*

The CLI is the scalpel; code is the factory. Reach for code when any of these is true:

- timing comes from **data** (word onsets, JSON, a spreadsheet — not human patience)
- the video is a **template** that renders many variants (titles, numbers, languages)
- the composition logic is genuinely conditional (per-word layout, beat-driven scenes)
- you must preview and iterate interactively before paying for a render

| Tool | Version (2026-10-13) | Lane |
| --- | --- | --- |
| remotion / @remotion/captions | 4.0.535 | React video as code, studio preview, Lambda renders |
| @remotion/install-whisper-cpp | 4.0.535 | bundled whisper.cpp transcription for Remotion projects |
| moviepy | 2.2.1 | scriptable assembly of modest clips (Python) |
| faster-whisper | 1.2.1 | GPU transcription (CTranslate2, ~4× faster) |
| openai-whisper | 20250625 | reference ASR |
| auto-editor | 29.3.1 | instant silence-based rough cuts |
| playwright | 1.64.1 (2026-10-15) | headless frame capture for the Canvas→Playwright→FFmpeg lane (`generated-motion.md`) |
| yt-dlp | 2026.8.19 | ingest (respect platform ToS) |

## 1 · Remotion — video as React

A Remotion composition is a React component rendered once per frame; `useCurrentFrame()` + `useVideoConfig()` drive everything.

```tsx
import {useCurrentFrame, useVideoConfig, interpolate, spring, Sequence, OffthreadVideo} from 'remotion';

export const Highlight = ({word}: {word: {text: string; startMs: number}}) => {
  const frame = useCurrentFrame();
  const {fps} = useVideoConfig();
  const scale = spring({frame, fps, config: {stiffness: 400, damping: 30}});
  return (
    <Sequence from={Math.round((word.startMs / 1000) * fps)} durationInFrames={20}>
      <div style={{transform: `scale(${scale})`}} className="caption">{word.text}</div>
    </Sequence>
  );
};
```

The official workflow, as the Remotion agent skills teach it:

1. **Studio first.** `npx remotion studio` — preview in the browser before any render. Only render when explicitly asked ("render", "export", "give me the MP4"): preview is the default deliverable of the inner loop.
2. **Captions are data, not files.** The `@remotion/captions` `Caption` type is the canonical unit:
   ```ts
   type Caption = { text: string; startMs: number; endMs: number;
                    timestampMs: number | null; confidence: number | null;
                    pageBreakAfter?: boolean };
   ```
   `pageBreakAfter` decides when a caption page fills — pagination is a rendering concern, not a transcription concern.
3. **Media via `<OffthreadVideo>` / `<Audio>`**; get durations/dimensions in-project with Mediabunny (browser-native media introspection) rather than shelling out.
4. **Render**: `npx remotion render [comp] out.mp4`; stills via `remotion still`; specific frames via `render --frames=0,30,90 --image-format=png` (that last one *is* your frame-strip verification).
5. **Scale**: Lambda / Vercel / Cloudflare render targets when volume grows (the `remotion-saas` skill owns that territory).

Word-timed animated captions (Bold / Bounce / Clean styles) are the canonical Remotion + editing bridge: transcript JSON in, styled word components out, rendered once — the same ledger-to-pixels discipline as the ffmpeg pipeline.

## 2 · MoviePy — Python assembly

```python
from moviepy import VideoFileClip, concatenate_videoclips, TextClip, CompositeVideoClip

a = VideoFileClip("a.mp4").subclipped(2.0, 12.4)      # frame-accurate trim
b = VideoFileClip("b.mp4").with_effects([vfx.MultiplySpeed(1.5)])
txt = TextClip(text="Chapter 2", font="Inter-Bold", font_size=88, color="white")
                      .with_position(("center", 200)).with_duration(2.5)
final = concatenate_videoclips([a, b], method="compose")
out = CompositeVideoClip([final, txt])
out.write_videofile("final.mp4", codec="libx264", audio_codec="aac",
                    preset="medium", ffmpeg_params=["-crf", "19", "-pix_fmt", "yuv420p"])
```

Honest lanes: MoviePy shines for *scriptable composition* — conditional layout, generated text cards, loops over data. It re-encodes, holds long videos in memory, and can't stream-copy or two-pass-loudnorm; when a recipe is "one filtergraph", the CLI is simply better. The dual-stack recipes pair them accordingly.

## 3 · Transcription at speed

- **faster-whisper** (GPU): same models, CTranslate2 runtime, ~4× faster — the bulk/long-form path.
- **whisper.cpp** — CPU-friendly; Remotion's `@remotion/install-whisper-cpp` bundles it for in-project transcription.
- Ladder by job: `tiny.en` candidate-scanning · `base` default · `medium`/`large` for noisy sources and the **final verification pass** (a fresh, stronger model re-reading the shipped audio is the acoustic audit in `the-pipeline.md` Phase 5).
- Always `word_timestamps=True`. Always extract 16 kHz mono first (`-vn -ac 1 -ar 16000`).

## 4 · auto-editor — the rough-cut machine

```bash
auto-editor raw.mp4 --silent-speed 99999 --video-speed 1 --margin 0.2sec -o rough.mp4
```

A deterministic first pass over silence — the mechanical half of Phase 2 in one command. Its output still needs the mistake pass, breath repair, and the review gate; it removes dead air, not bad takes. Use `--margin` to keep natural breath (the equivalent of the 0.14–0.24 s rule) and export the edit list (`--export`) when you want its decisions as an EDL to fold into your own ledger.

## 5 · Server-side and scale

When the job is *many* videos or *finding* moments across hours: managed pipelines (the VideoDB model — ingest → run analyzers (speech, scenes, OCR) → index → search moments with timestamps → timeline ops → render) beat local CLI loops. The ledger discipline is unchanged: transcripts and edit decisions remain data; rendering remains last and once. For local scale: the proxy workflow (`ffmpeg.md` §11) plus a job-list JSON and a `--dry-run` pass before execution.

## 6 · Choosing, in one line each

- **One known transform over known files** → ffmpeg CLI.
- **Data-driven timing or templated renders** → Remotion (Studio to iterate, render once).
- **Plain-Canvas scenes, no React, dependency-light** → Playwright frame capture + ffmpeg assemble (`generated-motion.md` §5).
- **Scriptable glue / generated cards / conditional assembly** → MoviePy.
- **Bulk transcription** → faster-whisper (or whisper.cpp in Remotion projects).
- **Instant rough cut from silence** → auto-editor, then the judgment passes.
- **Search across a library** → server-side indexing, then cut from timestamps.
