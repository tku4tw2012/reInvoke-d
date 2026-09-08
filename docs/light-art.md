---
title: Top-ring light art
description: Creative animation model for the 13-element Invoke top ring
ms.date: 2026-09-07
ms.topic: concept
---

## Purpose

The top ring can display more than functional status. reInvoke-d treats it as a
small ambient canvas for movement, rhythm, seasonal scenes, weather, and quiet
room presence.

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

The host can compose, preview, and export valid `.bin` assets. Playback of
custom assets is not connected yet. It needs a reviewed reInvoke control surface
for either a bounded art-asset transfer or a curated set packaged into the RAM
image.

The implementation does not use arbitrary shell access or place files on the
live device. That boundary stays with reInvoke and its hardware review.

