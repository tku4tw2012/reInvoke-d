---
title: Endpoint protocol version 1
description: Binary capture and control contract between reInvoke and reInvoke-d
ms.date: 2026-09-07
ms.topic: reference
---

## Scope

Each reInvoke endpoint opens one outbound TCP connection to the Mac. Version 1
carries microphone capture, privacy state, physical events, liveness, and
errors. It does not carry playback: response audio and music use the existing
Bluetooth A2DP path.

The protocol never proxies arbitrary WAMP calls, shell commands, files, model
requests, or capability actions.

## Trust boundary

The first household deployment uses one high-entropy token in mode-0600
operator-local configuration. The endpoint presents it during `HELLO`; the host
uses constant-time comparison and rejects a mismatch.

This authenticates a configured endpoint against accidental or casual peers. It
does not encrypt room audio or protect the token from a compromised peer
already observing the home LAN. If the threat model changes, wrap the protocol
in standard TLS with pinned identities rather than adding custom cryptography.

## Header

Every frame begins with this 28-byte network-byte-order header:

| Offset | Bytes | Field | Meaning |
|---|---:|---|---|
| 0 | 4 | magic | ASCII `RIVD` |
| 4 | 1 | version | `1` |
| 5 | 1 | type | Message type below |
| 6 | 2 | flags | Zero in version 1 |
| 8 | 4 | epoch | Connection generation assigned by the host |
| 12 | 4 | sequence | Strictly increasing within the epoch |
| 16 | 8 | sample position | Source audio frames sent before this frame |
| 24 | 4 | payload length | At most 1 MiB |

## Message types

| Value | Name | Direction | Payload |
|---:|---|---|---|
| 1 | `HELLO` | Endpoint to host | JSON identity, token, capture format, capabilities |
| 2 | `HELLO_ACK` | Host to endpoint | JSON assigned endpoint and epoch |
| 3 | `CAPTURE_START` | Host to endpoint | JSON negotiated capture format |
| 4 | `CAPTURE_STOP` | Host to endpoint | Empty |
| 5 | `CAPTURE_PCM` | Endpoint to host | Raw PCM, initially stereo 48 kHz S32_LE |
| 6 | `PRIVACY_STATE` | Endpoint to host | JSON Boolean `muted` |
| 7 | `EVENT` | Endpoint to host | Bounded JSON physical event |
| 8 | `KEEPALIVE` | Either direction | Empty |
| 9 | `ERROR` | Either direction | Bounded JSON error |

All control payloads are compact JSON objects. PCM is raw bytes.

## Handshake

1. Endpoint connects and sends `HELLO` with epoch and sequence zero.
2. Host validates endpoint id, token, capabilities, and capture format.
3. Host increments that endpoint's epoch and sends `HELLO_ACK`.
4. Every later endpoint frame carries the assigned epoch and an increasing
   sequence.
5. Endpoint sends the authoritative microphone privacy state.
6. Host may send `CAPTURE_START` only when it is ready to drain audio.

A reconnect always creates a new epoch. Frames from an older epoch are rejected,
not queued.

## Privacy behavior

The bridge starts muted and fails closed.

* No `CAPTURE_PCM` is valid while privacy state is muted.
* A confirmed mute immediately stops the capture source and purges queued PCM.
* Invalid or missing privacy state ends the connection.
* The host rejects PCM from an endpoint it believes is muted.
* The final live backend must obey reInvoke's capture-owner sequence. It may not
  infer privacy from network state or bypass the MCU controller.

## Backpressure

The host uses a bounded queue, initially 50 frames. A queue overflow ends the
stream epoch rather than buffering stale speech. The endpoint uses period-sized
frames, initially 2,048 bytes (256 stereo S32_LE frames).

TCP head-of-line blocking is accepted for version 1. If measurement shows that
control latency or audio continuity cannot meet the household gate, a later
version may separate control and media.

## Compatibility

The Python and Go codecs share a byte-exact golden frame in their tests. Both
reject:

* Wrong magic or version
* Unknown message type
* Payload over 1 MiB
* Truncated header or payload
* Non-increasing sequence
* Stale epoch
* Backward sample position
* PCM while muted

Protocol changes require a new version. The host may refuse an unsupported
endpoint rather than guessing compatibility.

