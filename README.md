# Optical Transfer: fountain-coded QR file transfer

Send any file between two devices using nothing but a **screen and a
camera**. One page displays the file as an endless stream of animated QR
codes; another device points its camera at it and reconstructs the file.
**No network path between the devices, no app, no pairing, no permissions
beyond the camera.** The payload travels as light.

<p align="center">
  <img src="docs/receiving.jpg" width="420"
       alt="Phone receiving a 2 MB image over light: 129.2 KB/s goodput, decoding the sender's animated QR code" />
</p>
<p align="center"><em>Mid-transfer: a phone pulling a file out of the air at 129 KB/s.</em></p>

## Features

- **Any file, not just images.** Pick a file on the sender and it's
  transferred with its original filename and type; the receiver offers a
  proper download, previewing it inline first if it's an image.
- **No handshake, no pairing.** Every QR frame is fully self-describing —
  the receiver locks onto a stream mid-flight, in any order.
- **Fountain-coded, so dropped frames don't matter.** Camera blur, autofocus
  hunting, and missed frames cost a little time, never correctness.
- **Tunable.** Frame rate, bytes per frame, error-correction level, and
  display size on the sender; capture resolution, capture fps, and decode
  worker count on the receiver.

## How to use it

1. Install dependencies and start the dev server:

   ```bash
   npm install
   npm run dev
   ```

2. **On the sending device** (a laptop is easiest): open
   `https://localhost:5173/send/`. Open the **Settings** panel, either pick
   one of the built-in demo images or tap **choose file** and select any
   file of your own. The stream starts automatically. Max screen brightness
   helps.

3. **On the receiving device** (typically a phone): open the `Network` URL
   Vite prints in the terminal (something like
   `https://<lan-ip>:5173/receive/`). Accept the certificate warning once,
   tap **Start camera**, and point it steadily at the sender's screen — prop
   the phone against something if you can, since hand tremor causing
   autofocus hunting is the #1 throughput killer.

4. A few seconds later: *Transfer Complete!* — the file downloads with its
   original name and type, verified by hash. Images also get an inline
   preview.

5. To send a different file, just change the setting or pick a new one on
   the sender — the receiver detects the new session automatically and
   resets, no restart needed on its end.

**Why the dev server is https-only:** the receiver uses `getUserMedia`, and
browsers remove that API entirely on insecure origins — a phone reaching
your dev server over plain http has no camera, full stop (`localhost` is
exempt, but your phone isn't localhost). The dev server ships with a
self-signed certificate (`@vitejs/plugin-basic-ssl`) to satisfy this; the
browser will warn on first visit — tap "Show Details" then "visit this
website" (iOS) or "Advanced" then "Proceed" (Android/desktop).

## How it works

**The one-way channel problem.** A screen-to-camera link has no
back-channel: the receiver can't ask for retransmission, and it will
inevitably miss frames (blur, refresh straddling, autofocus). Looping the
frames and hoping is miserable — miss one frame and you wait a full cycle
for it to come back around.

**Fountain codes fix this completely.** The sender never sends the file's
blocks directly. Each frame is the XOR of a pseudorandom *subset* of blocks;
the subset is derived deterministically from the frame's sequence number,
with subset sizes drawn from a robust-soliton distribution ([Luby transform
coding](https://en.wikipedia.org/wiki/Luby_transform_code)). The receiver
collects **any** ~K·1.15 distinct frames, in any order, and peels the file
out of them. Dropped frames cost a little time, never correctness. Sender
and receiver frame rates don't need to match at all.

**Every frame is self-describing.** A 20-byte header carries the session
id, sequence number, block count/size, file length, and a hash. There's no
handshake: the receiver locks onto a stream mid-flight, and restarting the
sender (new session id) automatically resets the receiver.

**Filename and type ride the same channel.** Rather than a separate
side-channel, the sender packs a small metadata block (name + MIME type)
directly in front of the file bytes before fountain-coding it — so it's
just as reliably reconstructed and hash-verified as the file itself.

**Decoding.** Safari has never shipped `BarcodeDetector` (WebKit bug
281848), so decoding is [zxing-cpp](https://github.com/zxing-cpp/zxing-cpp)
compiled to WASM, running in workers fed by `requestVideoFrameCallback`.
Busy workers mean dropped frames, which the fountain happily absorbs.

## Project layout

```
├─ index.html          landing page
├─ send/index.html     sender entry point
├─ receive/index.html  receiver entry point
├─ src/
│  ├─ core/            shared, platform-agnostic logic
│  │  ├─ fountain.ts    LT encoder/decoder, robust-soliton distribution
│  │  └─ protocol.ts    frame header pack/parse, file metadata pack/parse
│  ├─ send/main.ts      QR generation + display loop
│  ├─ receive/
│  │  ├─ main.ts        camera capture, decode orchestration, download
│  │  └─ worker.ts      zxing-wasm QR decode worker
│  └─ styles/app.css    shared styling
├─ public/samples/      built-in demo images
└─ docs/                README assets
```

## Hard-won details baked into this project

- **JS engines disagree about `Math.log`** (it's implementation-approximated).
  Sender and receiver must build bit-identical soliton distributions, so
  `fountain.ts` includes a deterministic log built from exactly-specified
  IEEE-754 ops. V8 vs JavaScriptCore desync is a silent, total failure mode.
- **iOS lies about camera frame rate.** `frameRate: {ideal: 60}` silently
  delivers 30; you must demand `{exact: 60}` (works at 1280-wide capture)
  and fall back. Always read back `getSettings()`.
- **`requestVideoFrameCallback` chains outlive their stream** and resume on
  the next one; without a generation counter, every stop/start leaks a
  zombie capture loop.
- **Progress bars must track frames collected, not blocks solved.** LT
  peeling back-loads its solve cascade: block-count progress looks stalled
  for most of the transfer, then teleports to 100%.
- **QR error correction is set to the minimum (L).** In-frame ECC and the
  fountain layer solve different problems (corruption vs erasure), but at
  these frame sizes level L plus frame disposal is the better trade.

## Tuning

| setting | default | notes |
|---|---|---|
| tx fps | 24 | each frame must own at least 2 refresh cycles of the display |
| bytes / frame | 1465 (QR v27) | denser is faster if the receiver still decodes it; 2953 (v40) works phone-to-phone at close range |

Changing any sender setting restarts the stream; the receiver resets
automatically off the new session id — no action needed on its end.

## License

MIT — see [LICENSE](LICENSE). Built with
[node-qrcode](https://github.com/soldair/node-qrcode) and
[zxing-wasm](https://github.com/Sec-ant/zxing-wasm), both bundled under
their own permissive licenses.
