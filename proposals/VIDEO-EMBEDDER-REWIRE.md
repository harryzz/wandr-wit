# wandr:video as embedder — layering + rewire design (task 120)

Status: **WIT landed, apps NOT rewired.** The three contracts below are stripped/
re-expressed and resolve (`wasm-tools component wit`) with no forced change back into the
extraction — the necessary-and-sufficient proof. This doc records the decisions and the
per-app rewire for a **later execution pass** (which will rebuild all consumers + restart
the zygote — see `[[feedback_shared_wit_rebuild_all_consumers]]`).

## The layering

```
wasi:video-codec@0.0.1   codec BASIC — WebCodecs VideoDecoder/VideoEncoder/VideoFrame.
                         Surface-free, virtualizable. Holds the opaque `frame`.
wasi:camera@0.0.1        capture SOURCE — W3C Media Capture. Produces `frame`.
        │  (wasi:* = universal primitive a guest cannot do itself)
        ▼  use / import
wandr:video@0.1.0        EMBEDDER — imports both; adds ONLY host-fill compositing:
                         `present.video-surface` (attach / present(frame,at-ns) /
                         set-rect / presented-rect / set-rotation) + `capture-encode.
                         call-encoder` (fused camera→encoder + PiP). wandr:* = our
                         stack's own decisions (host-fill decode-to-surface is not
                         expressible on upstream wasi-gfx:surface, which is guest-pump).
wandr:video-diag@0.1.0   test/diag — list-decoders / implementation / decoded-frames.
```

## Decisions (this pass)

- **n&s = "necessary for the app *class* the runtime must serve," not "what the proof app
  calls today."** The proof apps only prove/test. So `read-rgba` (guest-side pixel readers:
  screenshot/ML/software-encode/canvas), `encoded-chunk.decrypt` (encrypted playback:
  jellyfin/DASH CENC), and camera **enumeration** (`list-cameras`/`open-device`, needed on
  desktop where `facing` cannot pick a device) all STAY — each justified by a real app
  class + a web-standard analogue, even without a current call site.
- **`frame` stays in `wasi:video-codec`** (parallel to `wasi:audio-codec` holding
  `AudioData`; WebCodecs defines VideoFrame). A future `wasi:screen-capture` imports it the
  same way `wasi:camera` does. (`wasi:video-frame` extraction considered and declined.)
- **Camera enumeration kept + privacy made expressible:** `camera-info.name:
  option<string>` (`none` = label withheld pre-permission), empty list allowed before a
  grant, `device-id` opaque-until-granted — so the contract can honor the getUserMedia rule.
- **`facing.external`** kept as a deliberate divergence from W3C `VideoFacingModeEnum` (it
  names a USB/desktop webcam with no integrated front/back); select such a device by id.
- **Dropped for good** (embedder concerns, not codec): `connect(ctx)`/graphics-context,
  `decoded-frame`+`present`/`discard` (collapsed into `frame`; present→wandr), codec
  `set-rotation`, `ready`, `surface-unavailable` (tail-removed), `open-accelerated`
  (acceleration is in `decoder-config`), the 90 kHz `encoded-frame` (one µs `encoded-chunk`;
  RTP guests convert). `set-visible`/`set-preview-visible` dropped (0 consumers, additive
  re-add).
- **Audit outcome:** both `wasi:*` interfaces PASS all six WASI criteria; removing
  `present`/`connect` is a net **virtualizability** win. Full evidence: session audits
  against live WebCodecs + W3C Media Capture/Image Capture specs.

## Deferred WebCodecs/W3C fields (additive-later, no ABI break)

decoder-config `rotation`/`flip`/`description`; encoder-config `bitrate-mode`/
`scalability-mode`(SVC)/`latency-mode`; `frame` `flip`/`duration`/`color-space`/`format` +
a `copy-to(dest,format)`; encoder first-output SVC/`decoder-config` metadata; camera
`configuration-change` event + full zoom `MediaSettingsRange`; a `wasi:io/poll` pollable on
`next-decoded`/`next-frame` for a self-scheduling PULL guest.

## Per-app rewire (verb map; each app vendors the layered world under `wit/deps/`)

Each consumer's `wit/deps/` gains `video/` (the embedder) **plus** `video-codec/`,
`camera/`, `eme/` (the transitive wasi deps — same flat-vendoring pattern as
`contracts/wit/deps/`, which now holds them for the canonical). Each keeps its own
`wit_bindgen::generate!`.

### Signal — `apps/user/wandr.signal/engine/src/call.rs` (RTP; the only encoder/camera consumer)
- **Outgoing** `VideoEncoder::open(EncoderConfig{source_camera,facing,preview,preview_layer})`
  → `call-encoder::open(call-encoder-config{codec:encoder-config{…}, facing, preview,
  preview-layer})`. `next_frame`→`next-chunk`; `request_keyframe`/`set_bitrate`/
  `set_preview_rect`/`display_rotation` unchanged (now on `call-encoder`).
- **Incoming** `VideoDecoder::open(DecoderConfig{codec,rect,rotation,layer})` → split:
  `video-decoder::open(decoder-config{codec, acceleration:no-preference}, none)` (codec) +
  `video-surface::open(rect, layer, rotation)` (wandr) + `video-surface::attach(&dec)`
  (AUTO). `submit(EncodedFrame{ts:u32 90kHz})` → convert to µs (`ts*100/9`) →
  `submit(encoded-chunk{timestamp-us})`. `set_rect`→`video-surface.set-rect`;
  `set_rotation`→`video-surface.set-rotation`. `decoded_frames()`→`wandr:video-diag`.

### video.player — `apps/user/wandr.video.player/src/lib.rs` (GUEST-TIMED)
- `list_decoders()`/`implementation()` → `wandr:video-diag`.
- `VideoDecoder::open_accelerated(cfg, accel)` → `video-decoder::open(decoder-config{codec,
  acceleration:accel}, none)` + `video-surface::open(rect, layer, 0)`.
- `submit_timed(TimedFrame)` → `submit(encoded-chunk)` (already µs). `next_decoded()` →
  `next-decoded() -> frame`. `decoded-frame.timestamp_us()`→`frame.timestamp-us()`.
  `decoded-frame.present(at_ns)` → `video-surface.present(frame, at_ns)`.
  `presented_rect()`/`set_rect()` → on `video-surface`. `flush`/`reset` on the codec decoder.

### media-engine — `crates/wandr-media-engine/src/lib.rs` (GUEST-TIMED; jellyfin delegates here)
- Same decoder→surface pattern as video.player, incl. `reset()` (seek) on the codec decoder.
  `implementation()`→diag. `jellyfin` needs no direct rewire (it only calls the engine API).

## Verify (later pass)
Rebuild all four consumers + `wandr:video-diag` host impl; restart zygote; device-verify:
Signal call A/V (RTP AUTO-present, PiP, keyframe-on-PLI), video.player + jellyfin playback
(A/V sync via `present(frame, at-ns)`, seek via `reset`, subtitles via `presented-rect`),
desktop playback. Host impl split: `wasi:video-codec`/`wasi:camera` host impls + a thin
`wandr:video` embedder that owns the child surfaces and the camera→encoder glue.

## Forward-looking (not a dependency): wasi-gfx alignment
`wasi:canvas` is genuinely guest-pump (`get-current-buffer`/`present`) → a `surface-canvas`
pairing against upstream `wasi-gfx:surface@0.2.0` is legitimate (see
`wasi-canvas/connection.wit`). Video/camera are **host-fill / claim-not-pump** → they use
neither upstream verb, so `wandr:video`'s `video-surface` is correctly a wandr extension,
not a wasi-gfx pairing. A host-fill decode-to-surface path for wasi-gfx would need a new
upstream producer verb (WebAssembly/wasi-webgpu#55 is open) — a future proposal, never a
`wandr:video` dep.
