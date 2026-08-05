# wasi:camera — design notes (DRAFT, NOT WIRED)

Greenfield draft (2026-08-05). The camera **source**, factored out of the codec
(`wasi:video-codec`). No upstream `wasi:camera` exists at any phase — derived from the
W3C family the web platform already uses, the same playbook as `wasi:canvas` /
`wasi:audio`.

## Why a separate package (the W3C precedent)

The web platform draws the capture/codec line for us:

| Layer | W3C spec (WG) | wasi mapping |
|---|---|---|
| camera **source** | Media Capture and Streams — `getUserMedia`/`MediaStreamTrack` (WebRTC WG) | **`wasi:camera`** (`facing` == `facingMode`) |
| camera **controls / stills** | Image Capture — torch/zoom/focus, `takePhoto` (WebRTC WG) | `wasi:camera` `set-torch`/`set-zoom` (+ additive) |
| screen **source** | Screen Capture — `getDisplayMedia` (WebRTC WG) | future sibling lane (NOT camera) |
| raw-frame **bridge** | MediaStreamTrack Insertable Streams — `MediaStreamTrackProcessor` → `VideoFrame` | the shared `wasi:video-codec` `frame` |
| **codec + frame** | WebCodecs — `VideoDecoder`/`VideoEncoder`/`VideoFrame` (one spec) | **`wasi:video-codec`** (one package) |

So the web pipeline `getUserMedia → MediaStreamTrackProcessor → VideoEncoder` is exactly
`wasi:camera → wasi:video-codec`. Keeping the SOURCE separate from the CODEC means the
codec is device-agnostic and the camera is codec-agnostic — each necessary-and-sufficient
on its own.

## The frame is the currency (zero-copy default)

Captured frames are the shared **`wasi:video-codec` `frame`** — an OPAQUE, host-held
handle (= WebCodecs `VideoFrame`), NOT a camera-owned type: a codec must not depend on a
specific source, exactly as WebCodecs `VideoFrame` is source-agnostic (a decoder outputs
one, a camera/canvas makes one, an encoder consumes one). The camera PRODUCES it
(`next-frame`); `wasi:video-codec`'s encoder CONSUMES it (`encode`). Pixels stay
host-side; the handle is forwarded — to the encoder (HW encode) or a surface (viewfinder)
— without crossing the sandbox. `frame.read-rgba` is the EXPENSIVE opt-out for the
pixels-needed path (ML, software encode), preserving wandr's "pixels never enter the
sandbox" invariant. (`frame.rotation` carries the facing-adjusted sensor CVO — the
encoder's old `display-rotation`, now correctly a source property.)

Distinct from `wasi:video-codec`'s `decoded-frame`, which adds display scheduling
(`present(at-ns)`/`discard`); the two stay separate until/unless a consumer needs them to
converge.

## Viewfinder = the fifth graphics-context producer

`connect-preview(ctx)` composites the live capture into a connected surface's buffers
(`proposals/wasi-surface/DESIGN.md`, the fifth type). Host-fill: the camera CLAIMS the
context but does not pump it (claim-not-pump); placement / visibility / z are the
surface's contract. This is where the shipped fused encoder's `preview`/
`set-preview-rect`/`set-preview-visible` land — they were always the CAMERA's viewfinder,
not the codec's.

## The codec side (wasi:video-codec)

The encoder half is now part of the consolidated **`wasi:video-codec`** package (parallel
to `wasi:audio-codec`: one package = decoder + encoder + the shared frame). Its encoder is
a pure WebCodecs `VideoEncoder`: `open(config)` (no `facing`/camera), `encode(frame)`,
`next-frame`; no viewfinder, no surface dep. So `wasi:camera` (source) and
`wasi:video-codec` (codec) are cleanly separated, exactly as W3C separates Media Capture
from WebCodecs. The fused `wandr:video` keeps shipping the call path (R3 coexistence)
until the wasi wiring trigger.

## Named deferrals

| Deferral | Lane |
|---|---|
| Screen-share source (`getDisplayMedia`) | sibling future package (a screen is a source like a camera, not a camera mode) |
| Still capture (`takePhoto` / `PhotoCapabilities`) | additive to `frame`/`camera` when a consumer ships the need |
| Focus / exposure / white-balance controls | additive `set-*` methods (R2), as MediaTrackCapabilities exposes them |
| Depth / multi-stream / multi-cam sync | out of scope for the first draft |

## Status

Design-only, unwired. wandr's host already captures camera → HW encode zero-copy
internally (the shipped `wandr:video` encoder + `sf_media` preview surface), so wiring is
a re-skin of running machinery, triggered with the rest of the wasi-surface /
video-codec factoring. `wasm-tools component wit wasi-camera/wit/` resolves clean.
