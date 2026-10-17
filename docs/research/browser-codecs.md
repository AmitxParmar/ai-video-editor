# Browser codec and WebCodecs support matrix (2026)

Research ticket: [#49](https://github.com/AmitxParmar/ai-video-editor/issues/49)

Scope: what browsers can actually decode in 2026, both through `WebCodecs` and through
`MediaSource`/`<video>`, and what that means for the codec used for editing proxies and
previews. Facts checked against MDN browser-compat-data, caniuse, W3C specs, and vendor
release notes in October 2026.

## Why this matters

The editor decodes source media in the browser. There are two possible paths:

1. **`<video>` + `MediaSource`/byte-range seeking** — the classic path. Good coverage, but
   frame-accurate random access and non-realtime scrubbing are limited.
2. **`WebCodecs`** (`VideoDecoder`/`AudioDecoder`) — frame-accurate, scrub-friendly, but
   requires a JS demuxer (e.g. mp4box.js, Mediabunny) and has uneven browser support.

The proxy codec choice determines whether path 2 is viable everywhere or whether a
`<video>`-based fallback is needed on some browsers.

## Support matrix

### API floor: WebCodecs interfaces

| Interface | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| `VideoDecoder` / `VideoEncoder` | 94+ (Sep 2021) | 130+ desktop only (Sep 2024); Firefox Android: never | 16.4+ (Mar 2023) |
| `AudioDecoder` / `AudioEncoder` | 94+ | 130+ desktop only | **26+ only** (Sep 2025) |
| `MediaSource` | 23+ | 42+ | 8+ (fMP4 only); iOS only via ManagedMediaSource |
| `ManagedMediaSource` | not implemented | not implemented | Safari 17 macOS / 17.1 iOS+ |
| `OffscreenCanvas` (2D/bitmaprenderer) | 69+ | 105+ | 16.4+; offscreen WebGL 17+ |
| `AudioWorklet` | 66+ | 76+ | 14.1+ |

Key asymmetry: **Safari shipped audio decoding (`AudioDecoder`) only in Safari 26**, three
years after video. On Safari 16.4–18.7 WebCodecs is video-only, so audio must go through
`<video>`/`Web Audio`, not the WebCodecs audio path. Firefox on Android has no WebCodecs at
all.

### Video codec decode support

| Codec | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| H.264/AVC | Software + hardware since forever | Software + platform | Software + hardware |
| VP8/VP9 | Software + hardware | Software + hardware | **No software fallback**; hardware VP9 via platform |
| AV1 | Software (dav1d) + hardware | Software (dav1d) + hardware | **Hardware-only** (Safari 17+; M3 Mac / A17 Pro-class devices); no software fallback |
| HEVC/H.265 | **Hardware-only**, Windows/macOS/ChromeOS/Android; WebCodecs HEVC 8-bit from 107, 10-bit from 108 | Platform decoder only, and only when the OS/hardware provides it (133 Windows, 136 macOS HW-accelerated, 137 Linux VA-API) | Full support incl. software; WebCodecs HEVC from 17.4 |

Global decode support (WebCodecs Fundamentals 2026 dataset): H.264 ~99.9 %, VP9 ~99.99 %,
AV1 ~91.5 %, HEVC ~85.1 % (HEVC is strong on Safari, weak on Firefox/Edge).

### Audio codec decode support

| Codec | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| AAC-LC | yes | yes | yes |
| MP3 | yes | yes | yes |
| Opus (WebM/Ogg) | yes | yes | Ogg Opus since 18.4 (platform-dependent; probe) |
| Vorbis | yes | yes | Ogg Vorbis since 18.4 (probe) |
| FLAC | yes | yes | via platform (probe) |

MDN's codec-selection table pairs H.264 with MP4, VP9/AV1 with WebM, HEVC with MP4, and
names **H.264 + AAC + MP4 as the common baseline container pair**.

### Practical notes

- **WebCodecs does not demux.** Whatever codec is chosen, a JS demuxer is required
  (`mp4box.js` / `Mediabunny` for MP4, `webm-muxer`-style libs for WebM).
- **Hardware probing is unreliable.** `prefer-hardware` is a hint;
  `isConfigSupported()` with `hardwareAcceleration: 'require-hardware'` can return false on
  otherwise-capable machines (w3c/webcodecs#896). Probe, don't assert.
- **Safari VP9/AV1 support in WebCodecs is not officially documented** beyond HEVC. Treat
  as "probe with `isConfigSupported()`" rather than asserting availability.
- `VideoFrame`/`AudioData` must be `close()`d or decoding stalls (W3C spec explicitly warns);
  MDN notes <100 unclosed frames can crash a page via GPU memory. Use
  `decodeQueueSize`/`dequeue` for backpressure (~10 frames in flight is safe).

### Background throttling and memory (affects long exports)

- Chrome 88+ "intensive throttling": chained timers are checked once/minute after 5 minutes
  hidden; audio playing within the last 30 s exempts the page. `requestAnimationFrame` (and
  `requestVideoFrameCallback`) pause when hidden, so neither can drive a background export loop.
- Page Lifecycle API: hidden pages get frozen (timers/fetch suspended) or discarded (no
  unload events). Checkpoint state on `visibilitychange`; don't rely on tab staying hot.
- iOS/Safari evicts `MediaSource` buffers under memory pressure (`ManagedMediaSource` does it
  cooperatively); large 4K frames are ~33 MB each in RGBA.

## Bottom line for the decision

For proxies and previews that must decode in the browser via WebCodecs:

1. **Primary proxy codec: H.264 High profile, 8-bit 4:2:0, in fMP4, with AAC-LC audio.**
   This is the only combination that reaches ~99.9 % of the 2026 browser install base on
   both the `WebCodecs` and `<video>`/`MediaSource` paths, is well documented, and works on
   Safari 16.4+ (video) with the audio fallback. Codec string: `avc1.4d0034` (High, level 5.2)
   with `mp4a.40.2`.
2. **Secondary: VP9 in WebM (`vp09.*`)** for browsers/environments that prefer it — good
   everywhere except Safari's software fallback gap; safe on Chrome/Firefox/Edge.
3. **Optional: AV1** only when hardware decode is present (~91.5 % installs, Safari
   hardware-only). **HEVC: do not use as a delivery/proxy default** (~85.1 %, Firefox/Edge
   weak, Chrome hardware-only) — it's a display-optimization, not a baseline.
4. **Audio:** `AudioDecoder` exists only in Safari 26+; on Safari 16.4–18.7 route audio
   through `<video>`/Web Audio. Opus (in WebM/Ogg) is fine on modern Safari but still probe.

### Candidate minimum browser matrix

| Tier | Chrome/Edge | Firefox | Safari |
|---|---|---|---|
| Full editing experience (WebCodecs video+audio, workers) | ≥ 94 API floor; ≥ ~120 practical | ≥ 130 desktop (≥ 141 for WebGPU) | ≥ 26 |
| Video-only WebCodecs tier (audio via `<video>`) | — | — | 16.4–18.7 (≥ 17.4 for HEVC) |
| Unsupported for WebCodecs path | — | Firefox Android | — |

Treat the "minimum" for Chrome/Edge as the API floor (94) but the *practical* floor as ~120:
older versions have partial implementations and buggy hardware paths. Firefox Android has no
WebCodecs, so it falls back to progressive `<video>` playback.

## Sources

- MDN WebCodecs codec selection (updated Aug 2026): https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API/Codec_selection
- MDN WebCodecs usage guide: https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API/Using_the_WebCodecs_API
- W3C WebCodecs spec: https://www.w3.org/TR/webcodecs
- MDN browser-compat-data (VideoDecoder, AudioDecoder, ManagedMediaSource, OffscreenCanvas, AudioWorklet): https://raw.githubusercontent.com/mdn/browser-compat-data/main/api/VideoDecoder.json and sibling files
- caniuse WebCodecs: https://caniuse.com/webcodecs — MediaSource: https://caniuse.com/mediasource — AV1: https://caniuse.com/av1
- WebKit blog — WebCodecs in Safari 16.4: https://webkit.org/blog/13966
- WebKit blog — HEVC in WebCodecs, iOS WebM (Safari 17.4): https://webkit.org/blog/15063
- WebKit blog — Safari 18.4 (Ogg Opus/Vorbis): https://webkit.org/blog/14735
- Apple — Safari 26 release notes (AudioDecoder/AudioEncoder, MediaSource in-band tracks): https://developer.apple.com/documentation/safari-release-notes/safari-26-release-notes
- WebKit blog — Safari 27: https://webkit.org/blog/18325
- Chromium — supported media formats and HEVC hardware decoding: https://www.chromium.org/audio-video
- Chromium HEVC hardware-decoding support tracker (StaZhu): https://github.com/StaZhu/enable-chromium-hevc-hardware-decoding
- Chrome timer throttling: https://developer.chrome.com/blog/timer-throttling-in-chrome-88
- Chrome Page Lifecycle API: https://developer.chrome.com/docs/web-platform/page-lifecycle-api
- Chrome WebCodecs best practices (frame lifecycle, queue backpressure): https://developer.chrome.com/docs/web-platform/best-practices/webcodecs
- Firefox 130 release notes (WebCodecs desktop): https://www.firefox.com/en-US/firefox/130.0/releasenotes
- Firefox HEVC via platform decoder (137 Linux VA-API): https://9to5linux.com/mozilla-firefox-137-is-now-available-with-hevc-playback-support-on-linux
- WebCodecs Fundamentals codec-support dataset (2026): https://webcodecsfundamentals.org/datasets/codec-analysis-2026
- Browser version baselines (Oct 2026): https://endoflife.date/api/chrome.json and https://endoflife.date/api/firefox.json
