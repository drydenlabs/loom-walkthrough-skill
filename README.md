# loom-walkthrough

Turn any Loom screen recording into a Claude Code-ready handoff in one command. Pastes a complete `scaffold.md` (transcript + frame index + contact sheets) onto your clipboard. Drop it into chat with Claude and it has everything it needs to ship code from your voice walkthrough.

## Why

Describing UI changes in writing is slow and lossy. Talking through them on a Loom is fast — but a voice walkthrough doesn't slot into an AI coding assistant directly. The model can't watch a video; it can only read.

This skill bridges that gap. Record a Loom, run one command, paste once into Claude. The assistant now has both **what you said** (timestamped transcript) and **what was on your screen at each moment** (sampled frames + contact-sheet collages).

## What you get

| | |
|---|---|
| **One-command entry point** | `loom-fetch <loom-share-url>` does everything end-to-end |
| **Transcript with timestamps** | Pulled from Loom's CDN, rendered as `[MM:SS] phrase` lines |
| **Frames every N seconds** | Sampled with `ffmpeg`, default 5s interval |
| **Contact-sheet collages** | 5×5 grids per sheet so Claude can scan whole-recording arc first |
| **A `scaffold.md` paste-target** | Single file Claude reads — clipboard-ready |
| **Self-contained scripts** | Two bash files, ~440 lines total, glue around standard CLI tools |

## Install

### 1. Install the four CLI dependencies

```bash
brew install ffmpeg yt-dlp jq curl
```

(Linux: use your package manager. `pbcopy` on macOS becomes `xclip -selection clipboard` or `wl-copy` on Linux — edit the scripts if needed.)

### 2. Clone this repo

```bash
git clone https://github.com/drydenlabs/loom-walkthrough-skill.git
cd loom-walkthrough-skill
```

### 3. Symlink the scripts onto your PATH

```bash
mkdir -p ~/bin
ln -s "$(pwd)/scripts/loom-fetch" ~/bin/loom-fetch
ln -s "$(pwd)/scripts/loom-prep"  ~/bin/loom-prep
```

If `~/bin` isn't already on your PATH, add it:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 4. (Optional) Register as a Claude Code skill

The `SKILL.md` at the repo root is a Claude Code Skills definition. Symlink the repo into your Claude Code skills directory so the assistant can call the workflow by name:

```bash
ln -s "$(pwd)" ~/.claude/skills/loom-walkthrough
```

## Usage

### One-command flow

```bash
loom-fetch https://www.loom.com/share/<your-video-id>
```

Watch it work:

```
Loom URL:  https://www.loom.com/share/4964ed45cacf4a3fa19435b556b2fafe
Video ID:  4964ed45cacf4a3fa19435b556b2fafe
Interval:  every 5s
Workspace: /Users/you/Downloads/loom-walkthroughs

→ Fetching share page HTML…
  ✓ 25477 bytes
→ Extracting transcript URL from HTML…
  ✓ signed transcript URL extracted
→ Downloading transcript JSON…
  ✓ 27 phrases
→ Downloading .mp4 via yt-dlp…
  ✓ UniquelyGourmet-…mp4 ( 34M)
→ Running loom-prep…
  ✓ /Users/you/Downloads/loom-walkthroughs/loom-prep-20260528-162037

==========================================
  Done.
  Folder:    /Users/you/Downloads/loom-walkthroughs/loom-prep-20260528-162037
  ✓ scaffold.md copied to clipboard — paste into chat with Claude.
==========================================
```

Open Claude Code (or claude.ai), paste, send. Claude scans the contact sheets for whole-recording context, then drills into individual frames as the transcript references them.

### Tuning the frame interval

The second argument controls how often a frame is sampled (default: 5 seconds).

```bash
loom-fetch <url> 3   # 1 frame every 3s — very dynamic screens
loom-fetch <url> 10  # 1 frame every 10s — talking-head style
```

More frames = more context, but bigger contact sheets. The 5s default is tuned for screen recordings.

### Skipping the share URL

If you've already downloaded an `.mp4` from anywhere, `loom-prep` works on it directly:

```bash
loom-prep ~/Downloads/some-recording.mp4
```

You'll need to paste the transcript manually into the `scaffold.md` placeholder before sending to Claude.

## How it works

`loom-fetch` is the orchestrator. It does seven things in order:

1. Fetches the Loom share page HTML.
2. Extracts the signed transcript URL (Loom embeds it in the page; it's a CloudFront-signed CDN URL valid ~24h).
3. Downloads + formats the transcript JSON into `[MM:SS] phrase` lines via `jq`.
4. Downloads the `.mp4` via `yt-dlp`.
5. Calls `loom-prep` on the `.mp4`, which:
   - Extracts one frame every N seconds via `ffmpeg`.
   - Renders 5×5 contact-sheet collages (one per 25 frames) so Claude can scan the whole arc fast.
   - Writes a `manifest.md` (timestamps per frame) and a `scaffold.md` (paste-target).
6. Substitutes the transcript into the `scaffold.md` placeholder.
7. Copies the final `scaffold.md` to clipboard and opens the output folder in Finder.

## What ships in the scaffold

The `scaffold.md` Claude reads looks like this:

```markdown
> Source: https://www.loom.com/share/...

## Loom walkthrough — Recording.mp4

- Duration: 5m 17s
- 63 frames at 1 every 5s
- 3 contact sheet(s) (5×5 grids) at: /Users/.../sheets
- Individual frames at: /Users/.../

**Claude — please examine all contact sheets first for whole-recording context,
then drill into specific individual frames as needed.**

### Transcript

[0:00] Hey team, I wanted to process this video for you to go over...
[0:14] So this is the Sales Rep portal. Uh, overall, the dashboard looks good...
[0:24] I'm gonna scroll down and just, um, take a look at a couple of things...
...

### Frame index

- 00:00  frame_001.jpg
- 00:05  frame_002.jpg
- 00:10  frame_003.jpg
...
```

Claude reads it as a single readable document. The contact sheets give it visual context; the timestamped transcript ties words to visuals.

## File layout

```
loom-walkthrough-skill/
├── LICENSE
├── README.md
├── SKILL.md          ← Claude Code Skills definition (optional install path)
├── scripts/
│   ├── loom-fetch    ← one-command orchestrator
│   └── loom-prep     ← frame extractor + scaffold writer
└── docs/
    └── overview.html ← shareable single-page community write-up of the workflow
```

## Steal it

MIT licensed — fork it, rewrite it in Python, swap `ffmpeg` for something else, deploy it as a hosted service. If you build something interesting on top, open an issue and tell me about it.

## Credits

Built by [John Dryden Nanna](https://drydenlabs.com) (Dryden Labs) for use across our client engineering work. Inspired by the [Claude Code Skills](https://docs.claude.com/en/docs/claude-code/skills) pattern.
