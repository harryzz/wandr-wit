# wasi-video-decoder — the playback lane: necessary and sufficient

Status note (2026-08-02). The `video-decoder` resource has two lanes that share
one decoder + compositor but never mix on a single instance:

- the **call/RTP lane** (`submit` of an `encoded-frame` with a 90 kHz transport
  timestamp; decode-to-surface, `ready`/`decoded-frames`) — shipped and verified
  against live WebRTC calls; and
- the **playback lane** (added task 117 M2): `submit-timed(timed-frame)` →
  `next-decoded() -> decoded-frame` → `decoded-frame.present(at-ns)`, plus
  `flush` / `reset` and `open-accelerated`.

This note records the evidence that the **playback lane's shape is *necessary and
sufficient*** — every verb is load-bearing, and no verb was missing — gathered by
building real streaming clients on it rather than reasoning in the abstract. This
is the "prove with real consumers first" step for feeding the lane upstream.

## The consumers (the evidence base)

All run on the SAME import (`wandr:video/decoder`) + the same reactor exports
(frame handler; the guest owns demux + the A/V clock, the host owns decode +
composite). They share one extracted engine, `crates/wandr-media-engine`:

| Consumer | Source of media | Containers | Notable exercise |
|---|---|---|---|
| `wandr.jellyfin` | a real Jellyfin server, DirectPlay | MP4/MOV, MKV/WebM | seek, in-place audio-track switch, resume |
| `wandr.dash` | open DASH (`.mpd`) | fragmented-MP4 / CMAF | segmented, adaptive **bitrate switch**, seek |
| `wandr.dash` | open HLS (`.m3u8`) | fragmented-MP4 / CMAF | byte-range **and** whole-file segments |

Spanning **3 container families, 2 adaptive streaming protocols + DirectPlay,
H.264 (with H.265/VP9/AV1 lanes wired), byte-range and whole-file segments, seek,
and mid-stream bitrate switching** — the breadth a "sufficient" claim needs.

## Necessary — each playback verb, and the consumer behaviour that requires it

- **`decoded-frame.present(at-ns)`** — the guest computes the deadline from its
  audio-master clock (`wasi:audio` `playback.position()`), the *only* real media
  clock in the system, and the host presents at that instant. This is where A/V
  sync physically lives: with the tiny 224×100 DASH rep decoding far faster than
  realtime, correct `at-ns` scheduling is the difference between smooth playback
  and a 2.5× runaway. Not derivable host-side (the host has no audio clock);
  not derivable by "present immediately" (that *is* the runaway).

- **`timed-frame.timestamp-us` / `decoded-frame.timestamp-us`** — a real
  presentation timestamp (µs, the WebCodecs unit), carried IN and back OUT in
  **display** order. Necessary because B-frame reorder makes decode order ≠
  display order: the guest submits in decode order and must learn each decoded
  frame's own PTS to schedule it and to run the clock. The 90 kHz RTP
  `timestamp` on `encoded-frame` cannot serve this — wrong unit, wrong (decode)
  order.

- **`submit-timed` (with back-pressure semantics)** — `queue-full` means "retry
  the same frame after `next-decoded` drains one", NOT "dropped, resync on the
  next keyframe". A file player cannot skip to a keyframe on overflow without a
  visible glitch; every consumer relies on this back-pressure to pace decode to
  the clock. This is why the RTP `submit` could not simply be reused.

- **`next-decoded`** — the pull half of the decode-feed/drain split that lets the
  host run GStreamer/MediaCodec off the guest thread; the guest drains in display
  order and pacing is driven by `frame-pacing`/`on-frame`, not a spin loop.

- **`reset`** — proven by TWO independent behaviours: **seek** (discard queued
  input + reference frames + scheduled presentations, resume from a keyframe) and
  the **B2 bitrate switch** (re-init on a resolution/SPS change). Without `reset`
  the only option is close+reopen the decoder; both behaviours would be far more
  disruptive.

- **`flush`** — end-of-segment / end-of-file: release the frames a codec holds for
  reordering, or the tail is silently lost. Exercised at every stream end and at
  each per-segment boundary in the streaming demux.

- **`open-accelerated` + `implementation`** — hardware-preference open (kept
  additive so the shipped `open`/`decoder-config` ABI is untouched) plus a way to
  learn which implementation was actually granted, so "why is this dropping
  frames" is answerable from inside the app. Every consumer opens this way.

- **`presented-rect`** — where the host actually placed the video after
  letterbox/pillarbox + rotation. Necessary because subtitles and transport
  controls are drawn in the GUEST's UI layer over the picture (and in the black
  bars) — the pixels never cross the boundary, so the guest needs the landed
  rect. (`wandr.jellyfin` draws its VTT subtitles and scrub bar against it.)

## Sufficient — zero contract growth across all of the above

Across all four consumers and the whole variety above, **no host verb was ever
missing and the decoder WIT was never edited.** The engine extraction that these
clients share was an *imports-only* re-bindgen — not one verb, record, or field
of `wasi-video-decoder` changed while adding DASH, then HLS, then seek, then
adaptive bitrate switching. A contract that absorbs three containers, two adaptive
protocols, and rendition switching without a single new verb is, by the only test
that matters, sufficient.

## Boundary discipline (why this stayed small)

The demux, the A/V clock, subtitle rendering, and segment/adaptive logic are all
GUEST-side (`wasi-media-source` territory); only *decode + composite* is the
host's. Compressed AUs and codec config cross in; opaque decoded frames are
presented by handle and never return as pixels. That split is what keeps the
decoder contract this small while the consumers stay this varied.

— Evidence: `apps/user/wandr.jellyfin`, `apps/user/wandr.dash`,
`crates/wandr-media-engine`; wandr task 119 (Part B) closing task 117 M2.
