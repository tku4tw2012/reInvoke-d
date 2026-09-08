---
title: reInvoke-d
description: Two-tier local voice daemon for Harman Kardon Invoke speakers, built on OpenClaw and Ollama
ms.date: 2026-09-08
ms.topic: overview
---

## Purpose

reInvoke-d gives [reInvoke](https://github.com/tku4tw2012/reInvoke) hardware a
voice. It is a personal, non-commercial project for one household, built
specifically for Harman Kardon Invoke speakers.

It runs on the same host as OpenClaw and Ollama and is deliberately narrow:

> reInvoke-d owns the ear, the mouth, the ring, and the reflexes.
> OpenClaw owns the thinking and the memory.

The project name is intentionally `reInvoke-d`: reInvoke plus a small daemon.

## Status

> [!IMPORTANT]
> **This repository has been reset to its design.** The implementation is being
> rebuilt from scratch against a revised architecture. There is no working code
> here yet, and there has never been a release or a deployment.

A first implementation was written and then discarded. A design review found it
was built on the wrong premise: it owned household knowledge that OpenClaw
already owns, and it put a language model in the path of routine commands. The
measured evidence for that conclusion is in
[host measurements](docs/measurements.md).

What carries forward is the knowledge, not the code:

* [Design](docs/design.md) — the two-tier architecture
* [Measurements](docs/measurements.md) — what this host can actually do
* [Protocol](docs/protocol.md) — the endpoint wire format
* [Light art](docs/light-art.md) — the recovered 13-intensity-frame ring format
* [Language](docs/language.md) — bilingual wake and command evaluation

## Two tiers

The split is imposed by measured hardware, not chosen for elegance. On the
target host, warm:

| Path | Latency |
|---|---|
| Declared pattern match | ~0 ms |
| `llama3.2:1b` tool call | ~8.6 s |
| `qwen2.5:3b` tool call | ~12.9 s |

**Tier 1 — reflex.** Declared patterns resolve frequent, latency-sensitive, or
consequential commands with no model call at all. Timers, volume, music
transport, lights, ring art, time.

**Tier 2 — deliberation.** Anything else escalates to OpenClaw, announced out
loud, with the top ring showing the pending state. Twenty seconds of silence
reads as a broken device; twenty seconds after "let me look into that" reads as
thinking.

The boundary between them is configuration, not structure. The host is expected
to be replaced, and a faster CPU moves the dial without changing the design.

**Skill** means an OpenClaw skill and **agent** means the OpenClaw agent.
reInvoke-d implements neither and calls both across a process boundary.

## Growth

reInvoke-d does not try to anticipate every capability. Tier 2 absorbs whatever
was not anticipated, and OpenClaw's existing skill-workshop proposal queue turns
repeated requests into a reviewable definition. Anything asked often enough to
deserve instant response gets promoted into a Tier 1 reflex.

The house may suggest. It may not rewrite itself: proposals stay pending until a
human approves them.

## Deployment target

| Component | Target |
|---|---|
| Host | Macmini5,2, Ubuntu 22.04, i5-2520M, AVX only, 15 GiB, SSD |
| Co-resident | OpenClaw gateway (loopback) and Ollama |
| Endpoints | Up to three named Invoke rooms |
| Invoke runtime | ARMv7 BG2CDP |
| Network | Trusted private home LAN |

CPU time is the binding constraint. Not disk, not memory.

## Repository boundary

reInvoke and reInvoke-d remain independent repositories.

| Repository | Responsibility |
|---|---|
| `reInvoke` | Device boot and runtime, hardware control, microphone privacy, speaker safety, endpoint bridge, RAM-image packaging |
| `reInvoke-d` | Host daemon, voice pipeline, tier routing, endpoint protocol |

The versioned endpoint protocol is the shared boundary. Neither repository
imports the other's application code or Git history. The
[reInvoke product contract](https://github.com/tku4tw2012/reInvoke/blob/main/docs/current-product-contract.md)
is authoritative for device-side safety and ownership.

## Governing rules

* Deterministic code owns truth, decisions, and every consequential action.
* Household knowledge belongs to OpenClaw, not to reInvoke-d.
* Escalation to OpenClaw is announced aloud, never silent.
* Unsupported requests fail clearly rather than reaching a generic-model
  fallback.
* Microphone privacy is a trusted software boundary on the Invoke, not an
  electrical disconnect, and the host cannot override it.
* Voice playback extends the existing reInvoke playback-authorization boundary
  rather than bypassing it.
* Raw audio and interaction transcripts are not retained by default.

## License

[MIT](LICENSE)
