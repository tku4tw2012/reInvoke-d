---
title: reInvoke-d Voice System Design
description: Two-tier voice design for reInvoke hardware, built on OpenClaw and Ollama
ms.date: 2026-09-08
ms.topic: concept
keywords:
  - reInvoke-d
  - reInvoke
  - OpenClaw
  - local voice
  - Harman Kardon Invoke
estimated_reading_time: 11
---

> [!IMPORTANT]
> This is a personal, non-commercial project for one household. **The repository
> has been reset to this design and contains no implementation.** A first attempt
> was written and discarded: it owned household knowledge that OpenClaw already
> owns, and it put a language model in the path of routine commands. What
> survives is the design, the recovered device formats, and the measured
> evidence in [`measurements.md`](./measurements.md).

## Purpose and scope

reInvoke-d gives reInvoke hardware a voice. It is purpose-built for Harman
Kardon Invoke speakers and the existing reInvoke runtime, for one household
(2 people, up to 3 speakers).

It runs on the same host as OpenClaw and Ollama and is deliberately narrow:

> reInvoke-d owns the ear, the mouth, the ring, and the reflexes.
> OpenClaw owns the thinking and the memory.

reInvoke-d does not reimplement an agent, does not own household knowledge, and
does not answer open-domain questions. Anything outside its declared reflexes is
escalated to OpenClaw explicitly and audibly.

## Why two tiers

The tier split is imposed by measured hardware, not chosen for elegance. On the
target host (`OCmini52`, i5-2520M, 4 cores, 15 GiB, SSD, no usable GPU), warm,
with a tool schema already in context:

| Path | Latency | Accuracy |
|---|---|---|
| Deterministic pattern match | **~0 ms** | exact, for phrasings that were declared |
| `nomic-embed-text` semantic match | ~1,180 ms | 5 of 6 |
| `llama3.2:1b` tool call | ~8,600 ms | mis-selected the tool repeatably |
| `qwen2.5:3b` tool call | ~12,900 ms | 3 of 3 |

Effectively all of that latency is token generation: `qwen2.5:3b` generates at
~1.8 tok/s here, `llama3.2:1b` at ~3.7 tok/s. Prompt processing (300–1000 tok/s),
model loading, and disk are not the constraint. Only the CPU is.

Two consequences follow directly:

* **A model cannot sit in the path of a routine command.** "What time is it"
  must not cost 9 seconds.
* **Agentic reasoning is still worth having**, because 15–30 seconds is
  acceptable for a question you are not standing over the counter waiting on.

Therefore: a reflex tier with no model at all, and a deliberation tier that
delegates to OpenClaw.

## Tier 1 — Reflex

A declared, deterministic table. Pattern match resolves the command and its
slots, a handler runs, the answer is spoken. No model call, no network call to
OpenClaw, no possibility of hallucination.

Tier 1 exists only for commands that are frequent, latency-sensitive, or
consequential. It is deliberately small; a large Tier 1 recreates the closed
library this design is replacing.

Candidate reflexes: timers, speaker volume and mute, music transport, light
scenes and ring art, current time and date, and system status.

## Tier 2 — Deliberation

Anything Tier 1 does not confidently match escalates to OpenClaw:

```text
utterance
   │
   ├─ pattern match? ──────────── yes ──► TIER 1   (~0 ms)
   │
   └─ no ──► acknowledge aloud + ring "thinking"
              │
              └─► openclaw agent -m "<utterance>" --json --session-id <room>
                     │  (agent composes its own skills and reInvoke-d MCP tools)
                     ▼
                  speak the answer, ring returns to idle
```

Escalation is **announced, never silent**. reInvoke-d speaks a short
acknowledgement immediately and lights the ring, then speaks the answer when it
arrives. Twenty seconds of silence reads as a broken device; twenty seconds
after "let me look into that" reads as thinking.

This makes the top ring load-bearing rather than decorative: it is the
affordance that makes Tier 2 tolerable. See [`light-art.md`](./light-art.md).

## The tier boundary is a dial

What escalates is configuration, not structure. The host CPU is the binding
constraint today, and the host is expected to be replaced. A machine 5–8× faster
on token generation turns a 13-second tool call into 2–3 seconds, at which point
more categories can escalate without touching the architecture.

Nothing in this design may assume the current latency budget is permanent.

## State ownership

This is the boundary that most needs holding, because getting it wrong produces
two brains with two memories that never reconcile.

| Owner | Holds |
|---|---|
| **OpenClaw** | household knowledge, notes, observations, history, preferences, anything worth remembering across days |
| **reInvoke-d** | device and session state only: endpoints, rooms, privacy state, stream epochs, active session, accepted timers |
| **reInvoke (device)** | current bridge, audio, control, and privacy state; discarded on power loss |

reInvoke-d does not implement a household knowledge store. If a fact should be
remembered, it belongs to OpenClaw.

Accepted timers are the one deliberate exception: an alarm must fire at a
specific speaker whether or not OpenClaw is healthy, so reInvoke-d persists them
in SQLite with a wall-clock deadline for restart recovery.

## MCP tool surface

reInvoke-d exposes to OpenClaw only what OpenClaw cannot do itself — the things
that require the hardware:

* speak text on a named endpoint
* set, list, and cancel timers on a named endpoint
* read and set speaker volume and mute
* control music transport and top-ring art
* report endpoint, room, and privacy state

Tools that take a consequential action must be marked as requiring
confirmation, and the agent must not fire them unattended. This is a required
property of the tool surface, not an optional field.

Household data tools are deliberately absent: OpenClaw already owns that.

## Promotion

Tier 2 is how the house discovers what it should know; Tier 1 is how that
becomes fast. OpenClaw already implements this loop and reInvoke-d feeds it
rather than reinventing it.

OpenClaw's `autocapture` extracts durable signals from completed agent turns —
including failed turns, on the grounds that a correction after a bad answer is
the strongest signal — and groups repeats into a single skill-workshop proposal.

```text
repeated Tier 2 request
        │
        ▼
  openclaw skills workshop list      ← the queue
  openclaw skills workshop inspect   ← the definition
        │
        ├─► needs to be instant  ──►  write a Tier 1 reflex
        └─► does not             ──►  workshop apply, stays Tier 2
```

Configuration must keep the human in the loop:

* `skills.workshop.autonomous.enabled` — allow unprompted proposals
* `skills.workshop.approvalPolicy: "pending"` — queue for review, never
  self-apply

`approvalPolicy: "auto"` is out of scope. The house may suggest; it may not
rewrite itself.

## Deployment envelope

| Property | Design basis |
|---|---|
| Host | Ubuntu 22.04 on Macmini5,2, i5-2520M, AVX only (no AVX2/FMA), 15 GiB, SSD |
| Co-resident | OpenClaw gateway (loopback) and Ollama, already installed |
| Invoke endpoints | Up to three fixed named rooms, static configuration |
| Invoke compute | ARMv7 BG2CDP; 247 MiB runtime RAM observed |
| Network | Trusted personal home LAN |
| Recovery after outage | Manual RAM boot over USB |

Component selection follows measurement on this host, not benchmark claims.

## Responsibility boundary

All reInvoke-d logic runs on the host. The Invoke streams timestamped
microphone PCM and reports Action, rotary, privacy, and health events; enforces
microphone privacy and Bluetooth playback safety locally; and owns no timers,
household state, ASR, TTS, models, or capability logic.

The host runs wake detection, VAD, ASR, tier routing, TTS, and escalation, and
sends response audio through the existing Bluetooth A2DP path.

## Repository boundary

* `reInvoke` owns the device runtime, hardware safety, microphone privacy,
  playback authorization, the Endpoint Bridge, and RAM-image packaging.
* `reInvoke-d` owns the host daemon, tier routing, voice engines, the endpoint
  protocol, and protocol conformance tests.

The protocol is the shared boundary, not copied application code. The
authoritative device-side baseline is the
[reInvoke product contract](https://github.com/tku4tw2012/reInvoke/blob/main/docs/current-product-contract.md).

## Endpoint activation and audio safety

These constraints are established device behavior and survive the reset
unchanged.

**Playback authorization:** the existing reInvoke safe-playback path authorizes
the packaged BlueALSA player only with an active-PCM lease, ALSA ownership,
expected executable identity, and RUNNING state. reInvoke-d sends response audio
through that existing A2DP path and never becomes a second playback owner. The
MCU applies a 1.5-second mute holdoff across brief transport gaps, so a delayed
response may need a fresh authorized unmute cycle — this directly affects Tier 2,
whose answers arrive tens of seconds after the request. ALSA completion is not
proof of audibility: reInvoke has observed an unexplained media-volume-zero state
while ALSA remained RUNNING, so an alarm cannot be recorded as delivered from
transport completion alone.

**Microphone privacy:** authoritative on reInvoke; the host cannot override it.
This is a trusted software boundary, not an electrical disconnect — the MCU
occasionally emits no event for a physical Mic-Mute press, so a press alone does
not prove a state change. Configuring ALSA `hw_params` can overwrite an earlier
DSP mute route, so capture startup must configure, then request mute, then
discard until confirmed. The Endpoint Bridge must not open, read, or transmit
raw PCM while privacy state is confirmed muted.

Action short press is already Bluetooth play/pause and must not be taken over.
Action long press remains the activation candidate because it is unassigned.

## Endpoint protocol baseline

One outbound TCP connection per Invoke, capture and control only — playback stays
on the existing Bluetooth A2DP link. See [`protocol.md`](./protocol.md) for the
wire format. Every reconnect creates a new stream epoch; the host discards stale
or out-of-epoch capture and abandons any incomplete session tied to the old
connection.

This is a trusted-LAN design, not a hostile-network security boundary: one
high-entropy deployment token, no transport encryption, no PKI. Whenever an
endpoint is locally unmuted, captured room audio crosses the LAN in cleartext —
accepted for this household, not for a hostile network. If that threat model
changes, use standard TLS with pinned identities rather than custom cryptography.

## Language posture

Wake language and command language are separate choices. First posture:
configurable Chinese wake phrase (sherpa-onnx bilingual keyword model), English
command recognition, Unicode-safe routing so Chinese text is never erased, and
code-switching disabled until the bilingual recognizer passes a household
corpus. See [`language.md`](./language.md).

Tier 2 inherits whatever language ability the OpenClaw model has, which is
broader than Tier 1's declared patterns. Mixed-language requests are therefore
expected to escalate, and that is an acceptable first behavior.

## Acceptance gates

* Every selected binary or wheel runs on the AVX-only host without AVX2 or FMA.
* Wake testing: no unintended accepted session across 10+ hours of
  representative non-wake household audio.
* Streaming ASR faster than real time; refusal preferred over a wrong action.
* Tier 1 answers with playable response audio within 4 s p95 after the utterance
  ends, and makes no model or network call.
* Tier 2 speaks its acknowledgement within the same 4 s p95 budget, and the ring
  reflects the pending state for the whole escalation.
* A Tier 2 timeout or OpenClaw outage produces a clear spoken failure, never
  silence and never a guessed answer.
* Capture-owner sequence survives startup, Mic-Mute transitions, bridge restart,
  and reconnect — no PCM opened, read, or transmitted while confirmed muted.
* No regression to accepted Bluetooth playback, microphone privacy, recovery, or
  mute-first shutdown behavior.
* Consequential MCP tools cannot be fired by the agent without confirmation.

These are engineering targets, not product promises.

## Explicit non-goals

* Reimplementing agent behavior inside reInvoke-d
* A household knowledge store inside reInvoke-d
* Open-domain answers from model knowledge inside a Tier 1 reflex
* Self-applying skill proposals
* Hardware-neutral assistant infrastructure
* Enterprise discovery, identity, or authorization architecture
* Diarization or voice-based privileges
* Full duplex before measured AEC
* Timers, reminders, or household state on the Invoke
* Default raw-audio or transcript retention

## Unresolved design questions

1. Do the selected wake, VAD, ASR, and TTS components meet household accuracy
   with actual Invoke microphone audio?
2. What exact reInvoke capture-owner integration permits low-latency raw ALSA
   capture without weakening Mic-Mute privacy?
3. Which wake phrase meets recognition and household usability needs?
4. Does the 1.5-second mute holdoff require a fresh unmute cycle for Tier 2
   answers arriving tens of seconds late, and what does that cost?
5. How much interference occurs between an OpenClaw agent turn and concurrent
   ASR or TTS on four cores, and does Tier 1 need scheduling priority?
6. Is `qwen2.5:3b` the right escalation model given it was accurate but slow,
   and does OpenClaw's configured `llama3.2:1b` need to change?
7. What Action long-press and playback-resume behavior is least surprising
   before full-duplex activation exists?

## Design completion boundary

Nothing is implemented. This document and the recovered formats beside it are
the whole of the repository, and are the specification to build against.

Build order follows the constraints rather than the feature list:

1. Audio path — capture transport, ASR, TTS, playback through the existing A2DP
   route. Without ears and a mouth, neither tier exists.
2. Tier 1 reflexes — a deliberately small declared table.
3. Ring pending-state animation — required before Tier 2 is usable.
4. Tier 2 escalation and the MCP tool surface.
5. Promotion, wired to OpenClaw's existing proposal queue.

Unresolved questions require live capture and interaction testing; more abstract
architecture cannot answer them.

Implementation stays stage-gated: a failed measurement changes the smallest
affected candidate, threshold, or boundary. It does not reopen the project as a
general assistant, move application logic onto the Invoke, or move household
memory out of OpenClaw.
