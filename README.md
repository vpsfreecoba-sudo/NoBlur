# NoBlur — Post TikTok Videos Without the Blur

NoBlur is a premium, client-side web application that processes MP4 and MOV video containers locally directly in your browser to bypass aggressive server-side recompression when uploading to TikTok. It offers two pipelines: a full re-encode path with MP4 sample-table frame density inflation, and a 60fps VFI interpolation path. The result preserves original quality, visual fidelity, and audio-video synchronization.

All processing is performed client-side using JavaScript, ArrayBuffers, Blobs, and FFmpeg.wasm. No data is uploaded to external servers.

---

## Technical Architecture

NoBlur runs two distinct pipelines depending on the Interpolation toggle.

### Non-Interpolation Path (Pure Binary Patching)

The primary path for bypassing TikTok recompression. It rewrites the MP4 sample table using pure binary patching — no FFmpeg re-encode, preserving 100% video quality with 10-100x faster processing.

1. **ZeroLoss Track Bypass:** Rebuilds `edts`/`elst` atoms; the video track receives an edit-list `mediaTime` offset of 6000 ticks for AV sync alignment.
2. **Quantum Matrix Patch:** Patches the `mvhd` display matrix in-place.
3. **Udta Strip:** Force-applied — removes encoder signature from the `udta` atom, creating an empty udta if missing for consistency.
4. **Tkhd Matrix Reset:** Resets track header matrices to identity, preserving rotation metadata when present.
5. **Frame Density Inflation:** Inflates the sample table by a configurable multiplier (default 10x). Real frames are kept; codec-aware dummy samples are appended with `stts`/`stsz`/`stco`/`stsc` patched and padding written at EOF. Supports VFR (variable frame rate), 64-bit chunk offsets (co64), and per-codec dummy sizes (avc1/avc3: 8B, hvc1/hev1: 16B, vp09/av01: 4B). TikTok reads the inflated frame count as high-density content and skips heavy recompression.
6. **Comment Udta Injection:** Writes an Apple iTunes-style `©cmt` comment tag.

### Interpolation Path (60fps VFI + Binary Patch Pipeline)

When the Interpolation toggle is enabled, FFmpeg.wasm is lazy-loaded to run motion-compensated frame interpolation (`minterpolate`) to 60fps using the output resolution setting (1080p or 2K). Audio is copied without re-encoding (`-c:a copy`) for faster processing. The interpolated video is then passed through the same binary patching pipeline described above to ensure TikTok bypass compatibility. The FFmpeg instance is reset after VFI completes to prevent stale state errors.

---

## Key Features

- **Pure Binary Patching:** No FFmpeg re-encode in the main path — preserves 100% video quality and processes 10-100x faster than transcoding.
- **TikTok Compression Bypass:** Codec-aware frame density inflation (10x default) makes videos pass TikTok's quality-preservation threshold, avoiding the blur from server-side recompression. Works for both 1080p and 2K output.
- **Codec-Aware Inflation:** Per-codec dummy sample sizes (avc1/avc3: 8B, hvc1/hev1: 16B, vp09/av01: 4B), VFR support, and 64-bit chunk offset (co64) support for maximum container compatibility.
- **Selectable Output Resolution:** Choose between 1080p and 2K (1440p) when interpolation is enabled.
- **Client-Side Only:** 100% of processing happens locally within your browser, ensuring total data privacy.
- **Multi-Format & Codec Input:** Accepts MP4 and MOV containers with H.264, HEVC/H.265, and other codecs.
- **Bulk Processing Queue:** Drag and drop or select multiple videos to process in a sequential batch.
- **Screen Wake Lock:** Keeps the screen awake on mobile during processing; re-acquires the lock if the tab loses and regains visibility.
- **TikTok Studio Shortcut:** Direct upload button to TikTok Studio web; on mobile, a modal guides the user to enable desktop mode first.
- **Fast-Start Container Fix:** Recalculates chunk offsets (`stco`/`co64`) on every structural shift to keep output playable.
- **High-Contrast Dark Neo-Brutalist UI:** Flat offset shadows, solid dark panels, tactile click feedback, neon accents.
- **Responsive Mobile Layout:** Relocates the upload drop zone dynamically on mobile viewports; stat text wraps correctly on narrow screens.
- **Local History:** IndexedDB history with output-buffer thumbnails.

---

## File Structure

```text
NoBlur/
├── public/
│   └── coi-serviceworker.js
├── src/
│   ├── mp4-boxes.mjs
│   ├── mp4-patches.mjs
│   ├── mp4-strip.mjs
│   ├── mp4-inflate.mjs
│   ├── changelog.mjs
│   ├── changelog-data.mjs
│   └── changelog.test.mjs
├── index.html
├── style.css
├── app.js
├── db.js
├── coi-serviceworker.js
├── vite.config.js
├── package.json
├── biome.json
├── README.md
└── CHANGELOG.md
```

See [CHANGELOG.md](./CHANGELOG.md) for the full release history.

---

## Disclaimer

This utility rewrites MP4 container metadata using binary patching to bypass platform recompression. No video or audio data is re-encoded in the main pipeline, preserving original quality. The interpolation path (optional) uses FFmpeg.wasm for frame rate conversion only. It is designed to work with valid MP4 and MOV containers. Always keep backups of your original video files before processing.
