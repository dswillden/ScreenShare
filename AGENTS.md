# AGENTS.md — ScreenBridge Live

## Repo structure
- **Entire app is one file**: `index.html`. No build step, no npm, no bundler.
- Deployed via GitHub Pages: `master` branch, repo root (`/`). URL: `https://dswillden.github.io/ScreenShare/`
- The file also works opened locally as `file:///...` (sender side only; QR auto-fill disabled in that mode).

## Architecture
- **Transport**: Ably Realtime SDK (loaded from CDN at runtime via `loadAbly()`). Pure HTTPS/WSS — no WebRTC.
- **Two channels per session**: `sb-ctrl-{6digit}` (control events) and `sb-frames-{6digit}` (JPEG frames).
- **Frame pipeline**: `captureFrame()` → canvas JPEG → base64 → chunked at 48 000 chars → Ably publish → reassembled on receiver by `{fid, i, total}`.
- **API key flow**: stored in `localStorage` as `sb_ably_key`. Also auto-loaded from URL hash (`#k=<key>&c=<code>`) after QR scan; hash is stripped immediately via `history.replaceState`.
- **No backend**. Ably Root API key is entered per-device by the user.

## Key constraints
- Deployed on a Zscaler corporate network — UDP and WebRTC are blocked. Do not reintroduce WebRTC or PeerJS.
- Ably free tier: 6 M messages/month, 3 GB transfer. Chunking multiplies message count; keep chunk size at 48 000 chars or larger, not smaller.
- QR code renders only when `location.protocol !== 'file:'` — guard must stay in `renderQR()`.
- The API key input must keep `autocorrect="off" autocapitalize="none"` — mobile browsers silently corrupt the key otherwise.
- `validateAblyKey()` and `formatAblyError()` unwrap Ably's `ConnectionStateChange.reason` — do not flatten back to `String(e)`.

## Deploy
```
git add index.html
git commit -m "<message>"
git push   # triggers GitHub Pages rebuild (~1 min)
```
No CI, no preview environments, no PR required — push to `master` is production.

## Common edit locations
| What | Where in index.html |
|---|---|
| API key input / paste / save | ~line 169 (HTML), ~line 303 (JS) |
| QR rendering | `renderQR()` function |
| Sender init / Ably connect | `initSender()` |
| Receiver init / Ably connect | `initReceiver()` |
| Frame capture loop | `captureFrame()` / `captureRaf` |
| Frame reassembly | `rxFrameChannel.subscribe('frame', ...)` |
| Error formatting | `formatAblyError()`, `validateAblyKey()` |

## Gotchas
- `location.hash` is used for QR key passing — don't repurpose it for routing.
- `window.__autoJoinCode` is set by hash autoload and consumed by the `load` event listener to auto-click Connect on the receiver.
- The repo root (`C:\Users\2020303\Code\ScreenShare`) is a **separate git repo** from the user's home directory (`C:\Users\2020303`) which is also a git working tree. Always use `-C` flag or `workdir` when running git commands; never rely on `cwd`.
