# Captions — the default-on deliverable

*Merged from: jianshuo/claude-skills (wjs-burning-subtitles: libass discipline, fontsize calibration), 6missedcalls/video-editing-skill (style presets), louisedesadeleer/clipify (opus/karaoke styles, re-whisper discipline), oil-oil/oil-subtitle (CJK spacing, punctuation rules — concepts), kurbaitaev/ghost-editor (face-safe placement), nateherkai/hyperframes-student-kit (word-timed captions). Re-authored for this skill.*

Most feed viewing happens muted. Captions are not an accessibility garnish — they are the delivery format for the muted majority, and the first thing broken in amateur edits.

## 1 · Burn vs soft — the decision

| Situation | Do |
| --- | --- |
| WeChat / Douyin / feeds that ignore embedded tracks | **Burn in** (pixels) |
| YouTube, VLC, QuickTime, web players | **Soft-mux** (`mov_text`) or upload the SRT separately |
| Word-by-word animated / styled captions | Render as graphics (ASS karaoke or Remotion) — one caption system per output, never two |
| Muting-toggle matter of taste, source is pristine | soft-mux (zero re-encode, reversible) |

Soft-mux:

```bash
ffmpeg -i in.mp4 -i subs.srt -map 0:v -map 0:a -map 1:0 \
  -c:v copy -c:a copy -c:s mov_text \
  -metadata:s:s:0 language=eng -metadata:s:s:0 title="English" out.mp4
```

## 2 · Formats and their contracts

- **SRT** — the interchange format. Timestamps use **commas** for milliseconds (`00:00:01,240`). Whisper writes `.` in some builds — normalize before anything downstream parses it. No styling.
- **ASS/SSA** — full styling: fonts, colors, position, karaoke `\k` tags, `\b`old. The format for styled burn-in.
- **VTT** — web `<track>`; SRT plus header, cue settings, `.` milliseconds.

## 3 · The libass discipline (burn-in)

1. **Verify the filter exists before promising anything:**
   ```bash
   ffmpeg -filters 2>&1 | grep -E "subtitles|ass"
   ```
   Many package-manager builds ship without libass. On macOS drop in the static evermeet build rather than fighting it.
2. **Escaping**: inside `force_style`, every comma is `\,` (the filtergraph parser eats bare commas as chain separators). Paths on Windows: escape the drive colon.
3. **Fontsize is not pixels.** libass scales its PlayRes to the video resolution — a value that fit one video overflows another. Start `Fontsize=12`, render a 30 s preview, extract a frame at the **longest-line cue**, and look at it. Two short lines beat one long one.
4. **Preview checkpoint**: burn-in re-encodes the entire video — render `-t 30` first, check the frame, then commit.

Style skeleton that survives busy backgrounds:

```
force_style='Fontname=Inter\,Fontsize=12\,PrimaryColour=&H00FFFFFF\,OutlineColour=&H00000000\,BorderStyle=1\,Outline=2\,Shadow=1\,MarginL=20\,MarginR=20\,MarginV=40'
```

- `BorderStyle=1` (outline) default; `BorderStyle=3` (opaque box) only when the background genuinely fights the text.
- Set margins explicitly — defaults clip text on vertical phone video.

## 4 · The three house styles (+1)

| Style | Look | Build |
| --- | --- | --- |
| **standard** | bottom-center, white, black outline, semi-transparent — the subtitle | SRT + `force_style` burn |
| **minimal** | small lower-third, no outline weight, quiet | SRT + slim burn |
| **impact** | big centered word-by-word, active word highlighted — the retention style | ASS karaoke from word times, or Remotion `<Captions>` |
| **karaoke** (variant) | 3–4 word chunks, green sweep | ASS `\k` timing per word |

Impact captions are timed to **word onsets**, ±2 frames, from the *re-timed* transcript (Phase 3) — never from the source-transcript times, and never from segment times. Keyword emphasis may bold a word, never paraphrase the speaker.

## 5 · Timing and reading rules

- Reading speed budget: ~17–20 characters/second; a 2-word pop needs ≥0.3 s on screen; nothing under 0.3 s.
- Line length: ≤ ~42 Latin characters, ≤ ~15 CJK characters per line. Break lines explicitly (`\n`), never trust auto-wrap.
- One caption system per output — if Remotion renders animated captions, libass stays out of that file.
- Captions **follow the cut**: after any EDL change, re-derive times from the re-timed transcript (word ids are stable; seconds are always recomputed).

## 6 · Placement: the face and the platform

- **Never cover the face.** Track the face (or check the contact sheet) and place the caption band above the chin zone / below the face block; the default bottom band works only when the face sits high.
- **Platform safe area** (1080×1920): keep text inside **x 90–930, y 200–1600** — the right edge and bottom strip belong to platform UI (username, buttons, caption bar).
- Verify placement at phone scale on the encoded file, not the preview canvas.
- If the source has burned-in lower-thirds and you're reframing to vertical, they get cropped or stretched — strip or avoid them before the reframe.

## 7 · Text on screen — general rules

- No em dashes in on-screen text; punctuation-free caption display by default (commas and periods become pauses in timing, not glyphs).
- CJK ↔ Latin/number boundaries get one half-width space (`使用 Claude Code 做剪辑`) — applied mechanically at prepare time, never hand-typed per line.
- Correct ASR-able proper nouns from the glossary *before* caption generation; a spelled-wrong product name in captions is a permanent, visible error.
- Keep numbers exact across the chain: verify the reviewed transcript, the prepared caption text, and the final SRT agree on every digit (decimal points included).

## 8 · The caption gate

Before calling captions done: longest-line frame extracted and read at phone scale; no overlap with the face (contact sheet check); times spot-checked against three word onsets; the `.srt` sidecar ships next to the MP4 even when captions are burned (search, translation, and accessibility all want the text).
