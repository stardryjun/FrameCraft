# FrameCraft

Capture the exact frame you paused on — at the source's full resolution and bit depth, HDR included — with every subtitle track kept as a switchable layer.

FrameCraft asks your player where it is paused, reads only the few megabytes of the source file around that point, decodes the frame on screen and exports:

- `*_clean.png` — the frame without subtitles, 16-bit, original resolution (HDR PQ/HLG preserved)
- `*_subbed.png` — the same frame with the subtitles you picked burned in
- `*_layered.svg` — the frame plus every subtitle track as a separate layer, for design tools

[中文说明](README.zh-CN.md)

## Features

- **Frame-accurate**: picks the frame whose display interval contains the paused position, and verifies the decoded timestamp before and after encoding.
- **Works on huge remote files**: for MKV/WebM (Cues index) and MPEG-TS/M2TS (PES timestamps) it locates the byte range itself and downloads only that window over parallel HTTP Range requests — a 57 GB M2TS capture takes about a minute instead of timing out.
- **Blu-ray ISO / BDMV** (experimental): reads the disc's playlist and clip index through libbluray, remotely over Range requests.
- **Subtitle layers**: PGS, SRT, ASS/SSA, WebVTT, mov_text and more. Switch tracks on and off, stack two languages, nudge positions, or OCR an image subtitle into editable text. Changes save automatically when you leave the studio.
- **A/B clips**: mark a start point, then export the clip up to the current position (stream copy, no re-encode).
- **Background queue**: captures run in separate processes, so Raycast stays responsive and several captures can run at once.

## Requirements

- macOS
- [FFmpeg](https://ffmpeg.org) — `brew install ffmpeg`
- For Blu-ray ISO / BDMV sources only: Xcode Command Line Tools (`xcode-select --install`) and `brew install libbluray libudfread curl pkg-config`

## Supported players

| Source | How FrameCraft finds the position | Notes |
|---|---|---|
| Any Jellyfin client | Jellyfin `/Sessions` API | Accuracy depends on the client (see below) |
| Any Emby client | Emby `/Sessions` API | **Experimental** — same API family, not yet tested against a real Emby server |
| IINA | mpv IPC socket | Local files and direct URLs; one-time setup below |
| QuickTime Player | AppleScript | Local files |

### How accurate is the reported position?

FrameCraft decodes exactly the frame at the position the player reports. Whether that is the frame you *see* depends on the player. Measured against screen captures:

| Player | Result |
|---|---|
| **VidHub** (mpv engine) | Same frame as the screen on MKV, TS and Blu-ray ISO. Recommended. |
| **IINA** (mpv engine) | Reads mpv's own position directly. |
| **Infuse** | Off by about −24 to +11 frames (≈ ±1 s), no fixed pattern. |
| **Swiftly** (VLC engine) | Also noticeably off. |

mpv-based players report the timestamp of the frame on screen. Players that report their playback clock (Infuse, VLC-based clients) can stop anywhere between two frames and add audio/display latency, which cannot be corrected from outside the player.

## Setup

### Jellyfin / Emby

Open FrameCraft's preferences in Raycast (`⌘ ,` on any FrameCraft command) and fill in:

- **Server URL** — the address you open in a browser, including any sub-path such as `/jellyfin`.
- **API Key** — create one on the API Keys page of the server dashboard (Jellyfin: *Dashboard → API Keys*). Not your login password.
- **User Name Filter** (optional) — if several people stream at the same time.

Play something and pause, then run **Capture Frame**. The player must report its progress to that server; playing the file directly over SMB/WebDAV creates no session.

### IINA

1. IINA → Settings → Advanced → enable advanced settings.
2. Under *Additional mpv options*, add `input-ipc-server` with the value `/tmp/framecraft-iina.sock` (no leading `--`).
3. Quit IINA completely and reopen your video.

The socket can control IINA and is only reachable from your Mac. Remove the option to turn it off.

## Commands

| Command | What it does |
|---|---|
| **Capture Frame** | Queue a capture of the paused frame. Optional argument: `clean`, `dual` (Chinese + English), or `playlist=<number>` for discs. |
| **Capture Gallery** | Progress, results and failed tasks. Press ↵ on a capture to open the Subtitle Layer Studio. |
| **Mark Clip Start** | Remember the current position as the clip start. |
| **Export Clip** | Export from the start mark to the current position (the last 15 s if no mark). |
| **Copy Stream URL** | Copy the path or URL being read. It may contain your API key — don't share it publicly. |

## Preferences

| Preference | Default | |
|---|---|---|
| Save Location | `~/Pictures/FrameCraft` | |
| Default Subtitles | Primary track | Which tracks appear in `_subbed.png`. All tracks stay in the SVG. |
| FFmpeg Path | auto | Searches Homebrew locations. |
| Network | direct | Ignores proxy environment variables when reading video. |
| Clipboard | on | Copies the finished image. |

## Limits

- The server must support HTTP Range requests (206). A remote capture reads at most 512 MiB / 512 requests and never falls back to a full download.
- Blu-ray support is screenshot-only and experimental; encrypted discs and DVD ISOs are not supported. If a disc has more than three candidate playlists, pass `playlist=<number>`.
- HDR subtitles are composited in linear light at 203-nit reference white (PQ). HLG subtitle color conversion is not implemented.
- Long-running subtitles and complex ASS styling are not fully verified.

## Privacy

FrameCraft only talks to the media server you configure. Your API key is stored by Raycast and is redacted from error messages. Captures, history and caches stay on your Mac (cache entries expire after 10 minutes).

## Development

```bash
npm install
npm run dev      # builds the background worker and starts Raycast development mode
npm run build    # production build
npm run lint
```

`assets/worker.cjs` is the background worker bundle generated from `src/worker/` by `npm run build-worker`. Rebuild it after changing anything it imports. `assets/disc-reader.c` is compiled on first use when a disc source is captured.

## License

[MIT](LICENSE)
