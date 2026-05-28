---
name: loom-walkthrough
description: One-command Loom URL → Claude-ready handoff. `loom-fetch <url>` downloads the .mp4, scrapes the transcript from Loom's page, extracts frames at 5s intervals, builds 5×5 contact sheets, and copies a complete paste-ready scaffold (with transcript pre-filled) to the clipboard. Invoke /loom-walkthrough.
---

## Purpose

Bridge a Loom video — which Claude can't watch — into a set of artifacts Claude CAN consume: still frames (read natively as images), contact sheets (5×5 grids for fast scanning), and the transcript pre-filled into a paste-ready scaffold. The whole walkthrough is examinable. Claude scans contact sheets first for whole-recording context, then drills into individual frames for detail. No frames get filtered out.

## When to use

- The user needs to show Claude something dynamic and visual — UI walkthroughs, comparing Figma to a live build, pointing out polish issues across many screens, narrating business rules over a live demo.
- More than 2–3 screenshots of context is needed, OR
- The reasoning matters as much as the visuals (spoken thought process captured via Loom's auto-transcription).

## When NOT to use

- A single static screenshot answers the question — just drop the image.
- A 30-second recording with 1–2 key moments — manual workflow is faster.

## Inputs

- A public Loom share URL: `https://www.loom.com/share/<video-id>`

That's it. The script handles everything else.

## Steps (automated flow — primary path)

1. **The user records** the Loom — narrates clearly, mentions tab names, file paths, business rules, what's being compared (e.g. Figma vs localhost vs live).
2. **The user runs**: `loom-fetch <loom-url>` from any terminal directory.
3. The script does end-to-end (~30–60s for a 10-min recording):
   - Fetches the share page HTML
   - Extracts the signed transcript URL (Loom embeds it in the page; CloudFront-signed, expires ~24h)
   - Downloads the transcript JSON and renders to plain text with `[MM:SS]` timestamps per phrase
   - Uses `yt-dlp` to download the .mp4
   - Runs `loom-prep` to extract frames + contact sheets
   - Splices the transcript into the scaffold, prepends source URL
   - Copies the complete scaffold to clipboard
   - Opens the output folder
4. **The user pastes the scaffold** into chat with Claude — already complete, nothing else to add.
5. **Claude IMMEDIATELY starts processing after verifying the artifacts exist** — no waiting for an explicit "start now" prompt, no clarifying questions before reading. The artifacts are the green light:
   a. Quick sanity check: contact sheets, frames, transcript, manifest all present in the output dir referenced by the scaffold.
   b. Read ALL contact sheets first (parallel tool calls) for whole-recording context.
   c. Drill into specific individual `frame_NNN.jpg` files for detail when a sheet shows something worth examining closely.
   d. Cross-reference with the transcript's `[MM:SS]` timestamps.
   e. Produce structured notes per stage/tab/topic and group fixes or follow-ups before any execution.
6. Only AFTER the structured pass — if there are genuinely ambiguous decision points — ask the user targeted questions. Don't pause the read to ask "should I proceed?" — the recording itself is the briefing.

## Steps (fallback — when `loom-fetch` can't reach the URL)

If a Loom is private/auth-gated or the transcription hasn't finished:

1. Manually download the .mp4 from Loom's web UI (3-dot menu → Download).
2. Run `loom-prep ~/Downloads/<file>.mp4`.
3. Manually copy the transcript from Loom's sidebar (Transcript → Copy).
4. Paste the scaffold + manually replace the placeholder line with the transcript text.

## Interval override

Default is 5 seconds — tuned for dynamic recordings where screens change frequently. Override per call:

```bash
loom-fetch <url> 3    # 1 frame every 3s — very fast screen changes
loom-fetch <url> 10   # 1 frame every 10s — talking-head recording
```

## Output

Everything for one recording lives in `~/Downloads/loom-walkthroughs/loom-prep-<timestamp>/`:

- `frame_001.jpg` … `frame_NNN.jpg` — full-res individual frames
- `sheets/sheet_001_frames_001-025.jpg` … — 5×5 contact sheets
- `<title>-<video-id>.mp4` — the source recording itself
- `transcript.json` — raw Loom transcript JSON
- `manifest.md` — every frame with timestamp
- `scaffold.md` — complete paste-to-Claude block with transcript pre-filled

The scaffold is auto-copied to the macOS clipboard.

## Tooling

- `ffmpeg` — frame extraction + contact-sheet tiling
- `ffprobe` — video duration probing
- `yt-dlp` — .mp4 download (supports Loom's share URLs natively)
- `jq` — JSON parsing for the transcript
- `curl` — page fetch
- `python3` — multi-line transcript splice into scaffold

All installed via Homebrew on macOS (`brew install ffmpeg yt-dlp jq curl`). On Linux use your package manager; replace `pbcopy` in the scripts with `xclip` or `wl-copy`.

Loom's own developer API was evaluated and skipped — direct page-HTML scraping is more reliable for transcript access (the API is enterprise-focused and doesn't cleanly expose transcripts).

## Scripts

- `scripts/loom-fetch` — primary, URL-driven, automated
- `scripts/loom-prep` — invoked by `loom-fetch`; usable standalone on a manually-downloaded .mp4

Symlink both into `~/bin/` (or anywhere on PATH) so they run from any directory.

## Why this design

- **Contact sheets are critical for efficiency** — reading 60 individual frames per recording costs 60 tool calls; reading 3 sheets is ~3 calls with no loss of context for whole-recording scanning.
- **Full-res frames stay on disk** — when a contact sheet shows something worth examining closely, Claude reads the high-res individual frame for that moment.
- **All frames preserved** — no filtering or summarization, so no important moment gets silently dropped.
- **Transcript is `[MM:SS]`-prefixed per phrase** so Claude can correlate any moment in the narration with the corresponding frame index.
