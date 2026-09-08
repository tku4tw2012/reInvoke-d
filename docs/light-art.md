---
title: Top-ring light art
description: Creative animation model for the 13-element Invoke top ring
ms.date: 2026-09-08
ms.topic: concept
---

## Purpose

The top ring is both a status surface and a small ambient canvas — movement,
rhythm, seasonal scenes, weather, and quiet room presence.

It is also **load-bearing, not decorative**. Tier 2 requests escalate to
OpenClaw and take tens of seconds on the current host. Silence for that long
reads as a broken device; a visibly working ring reads as thinking. The ring is
the affordance that makes the deliberation tier tolerable, so a pending-state
animation is a requirement rather than a flourish. See
[`design.md`](./design.md).

This is separate from smart-bulb control. Bulbs illuminate a room; light art
animates the speaker itself.

## Recovered format

One animation frame is exactly 13 bytes:

* Bytes 0 through 11: perimeter intensities
* Byte 12: center intensity

Each value is 0 through 255. The recovered format establishes intensity, not
RGB color, so pattern names describe motion rather than promising a color.

reInvoke accepts at most 1 MiB per asset, sends up to 390 animation bytes per
MCU chunk, and waits 280 ms between chunks. Thirty frames fit in one chunk.

## Built-in patterns

| Pattern | Motion |
|---|---|
| `breathe` | Whole ring rises and falls, center trailing |
| `orbit` | One soft point moves around the perimeter |
| `sunrise` | Light grows from one side and fills the center |
| `sparkle` | Seeded, repeatable soft twinkles |
| `christmas` | Alternating groups with seeded sparkle accents |
| `rain` | Sparse drops travel around the ring |

All built-ins are deterministic. The same pattern and seed produce byte-exact
assets, which makes them reviewable and testable.

## Privacy has priority

Light art never owns microphone privacy. reInvoke's LED player blocks ordinary
animation while privacy mute is active, and the protected red indication
continues until the MCU privacy controller confirms unmute.

No art capability may:

* Override privacy red
* Clear the ring while privacy red is required
* Resume a stale animation after a privacy transition
* Claim RGB color that the intensity format does not establish

## Current status

Nothing is implemented. The frame format above was recovered from the device and
is the reference for rebuilding.

Two things must be built, in this order:

1. A **pending-state animation** for Tier 2 escalation. This is the minimum
   viable ring behavior, because the deliberation tier is unusable without it.
2. Composition, preview, and export of `.bin` assets for ambient art.

Playback of custom assets additionally needs a reviewed reInvoke control surface
for either a bounded art-asset transfer or a curated set packaged into the RAM
image. That boundary stays with reInvoke and its hardware review; no host
implementation may use arbitrary shell access or place files on the live device.

