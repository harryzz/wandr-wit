# wasi:camera — design notes (DRAFT, NOT WIRED)

Greenfield draft (2026-08-05). The camera **source**, factored out of
`wasi:video-encoder`. No upstream `wasi:camera` exists at any phase — this is derived
from the W3C family the web platform already uses, the same playbook as
`wasi:canvas` / `wasi:audio`.

## Why a separate package (the W3C precedent)

The web platform draws the capture/codec line for us:

| Layer | W3C spec (WG) | wasi mapping |
|---|---|---|
| camera **source** | Media Capture and Streams — `getUserMedia`/`MediaStreamTrack` (WebRTC WG) | **`wasi:camera`** (`facing` == `facingMode`) |
| camera **controls / stills** | Image Capture — torch/zoom/focus, `takePhoto` (WebRTC WG) | `wasi:camera` `set-torch`/`set-zoom` (+ additive) |
| screen **source** | Screen Capture — `getDisplayMedia` (WebRTC WG) | future sibling lane (NOT camera) |
| raw-frame **bridge** | MediaStreamTrack Insertable Streams — `MediaStreamTrackProcessor` → `VideoFrame` | `wasi:camera` `frame` (the currency) |
| **codec** | WebCodecs — `VideoEncoder`/`VideoDecoder`/`VideoFrame` | `wasi:video-encoder` / `wasi:video-decoder` |

So the web pipeline `getUserMedia → MediaStreamTrackProcessor → VideoEncoder` is
exactly `wasi:camera → wasi:video-encoder`. Keeping them separate means the encoder is
orientation- and device-agnostic, and the camera is codec-agnostic — each is
necessary-and-sufficient on its own.

## The frame is the currency (zero-copy default)

A `frame` is an OPAQUE, host-held handle — the WebCodecs `VideoFrame`. Pixels stay
host-side; the handle is forwarded to an encoder (HW encode) or a surface (viewfinder)
without crossing the sandbox. `frame.read-rgba` is the EXPENSIVE opt-out for the
pixels-needed path (ML, software encode). This is the whole reason the split works
without regressing wandr's "pixels never enter the wasm sandbox" invariant.

`frame` here and `wasi:video-decoder`'s `decoded-frame` are both opaque host-held
VideoFrames — a strong candidate to hoist into a shared `wasi:media-types` /
`wasi:video-frame` package (same call as the shared `acceleration`/`support`
vocabulary). Kept local until a second consumer needs it.

## Viewfinder = the fifth graphics-context producer

`connect-preview(ctx)` composites the live capture into a connected surface's buffers
(`proposals/wasi-surface/DESIGN.md`, the "camera preview / fifth type"). Host-fill:
the camera CLAIMS the context but does not pump it (claim-not-pump); placement /
visibility / z are the surface's contract. This is where the shipped fused encoder's
`preview`/`set-preview-rect`/`set-preview-visible` land — they were always the
CAMERA's viewfinder, not the encoder's.

## Companion change to wasi:video-encoder — DONE (2026-08-05)

The source is factored out and the encoder is now a pure codec:

- `wasi:video-encoder@0.0.2` DROPPED `camera-facing`, `encoder-config.facing`,
  `connect-preview` (viewfinder → this package), and `display-rotation`; ADDED
  `encode(frame: borrow<frame>)` — exactly WebCodecs `VideoEncoder.encode(VideoFrame)`.
- The frame is the SHARED **`wasi:video-frame`** package, NOT camera-owned: a codec
  must not depend on a specific source (WebCodecs `VideoFrame` is source-agnostic).
  `wasi:camera` PRODUCES it (`next-frame`), `wasi:video-encoder` CONSUMES it
  (`encode`). This is the "hoist when the second consumer arrives" trigger firing.
- `display-rotation` became `frame.rotation` — the CVO now travels on the frame.

The fused `wandr:video` encoder keeps shipping the call path (R3 coexistence) until the
wasi wiring trigger, same as every other lane in this extraction.

## Named deferrals

| Deferral | Lane |
|---|---|
| Screen-share source (`getDisplayMedia`) | sibling future package (a screen is a source like a camera, not a camera mode) |
| Still capture (`takePhoto` / `PhotoCapabilities`) | additive to `frame`/`camera` when a consumer ships the need |
| Focus / exposure / white-balance controls | additive `set-*` methods (R2), as MediaTrackCapabilities exposes them |
| Depth / multi-stream / multi-cam sync | out of scope for the first draft |
| Shared `VideoFrame` type | DONE — hoisted to `wasi:video-frame@0.0.1` (camera produces, encoder consumes). The decoder's `decoded-frame` stays separate (it carries display scheduling); converge later if a consumer needs it |

## Status

Design-only, unwired. wandr's host already captures camera → HW encode zero-copy
internally (the shipped `wandr:video` encoder + `sf_media` preview surface), so wiring
is a re-skin of running machinery, triggered with the rest of the wasi-surface /
video-encoder factoring. `wasm-tools component wit wasi-camera/wit/` resolves clean.
