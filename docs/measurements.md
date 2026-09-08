---
title: Host measurements
description: Controlled measurements of the reInvoke-d voice stack on the target Mac mini
ms.date: 2026-09-08
ms.topic: reference
---

## Method

Every figure here was measured on the deployment host itself, which is also the
development workstation. Each run warms the component first, then repeats each
case and reports the median. Language-model figures use the Ollama server's own
`prompt_eval_count`, `eval_count`, and `eval_duration` counters rather than wall
clock alone, so prompt evaluation and generation are separated.

These results supersede earlier exploratory observations that mixed prompt
shapes, cold starts, and co-resident load. The harnesses that produced them
were removed in the application-layer reset; the numbers stand as the measured
baseline for this host and are reproducible against the same models through
Ollama's API.

## Host

| Property | Value |
|---|---|
| Hardware | Apple Macmini5,2 |
| CPU | Intel i5-2520M at 2.50 GHz, 2 cores, 4 threads |
| Vector extensions | `avx` only, no `avx2`, no `fma` |
| Memory | 15 GiB |
| OS | Ubuntu 22.04.5 LTS |
| Python | 3.10.12 |
| Sound server | PulseAudio, protocol 35 |
| Bluetooth | BlueZ 5.64 |

## Language model

Model `qwen2.5:0.5b` (Q4_K_M) through Ollama, `temperature=0`, fixed seed,
warm, no competing model resident.

| Case | Prompt tokens | Generated tokens | Time to first token | Wall clock |
|---|---|---|---|---|
| Domain classification | 66 | 2 | 0.16 s | 0.28 s |
| Intent classification | 64 | 2 | 0.21 s | 0.37 s |
| Slot extraction | 57 | 3 | 0.14 s | 0.40 s |
| Combined JSON | 60 | 18 | 0.09 s | 1.71 s |
| Response rendering | 74 | 18 | 0.11 s | 1.59 s |

Generation rate is **12.1 tokens per second** measured over evaluation time
alone. Time to first token for a warm model is **0.09 to 0.21 seconds**.

Three sequential micro-calls totalled **1.05 s** against **1.71 s** for one
combined JSON call, so narrow sequential calls remain the cheaper shape. The
margin is smaller than earlier exploratory numbers suggested, and latency is
roughly five times better than previously assumed.

### Accuracy, not only latency

The combined JSON call returned `{"domain":"aliyun","intent":"set pasta
time","minutes":12}`. The domain is fabricated and the intent is not a member of
any closed set. The same utterance handled as three narrow calls returned
`timer`, `create`, and `12`, each correct and each inside its declared set.

This is direct evidence for the central architectural rule: ask one small
question with a closed answer set, never one broad question. It also anticipates
the tool-calling result below — a small model fabricates freely when the answer
space is open, and stays honest when it is closed.

### Model comparison

`qwen2.5:1.5b` under identical conditions:

| Metric | `qwen2.5:0.5b` | `qwen2.5:1.5b` |
|---|---|---|
| Generation rate | 12.1 tok/s | 9.1 tok/s |
| Three micro-calls | 1.05 s | 1.52 s |
| Combined JSON | 1.71 s | 3.89 s |
| Intent classification | `create`, correct | `List`, incorrect |
| Combined JSON output | Bare JSON | Wrapped in Markdown fences |

The larger model is slower **and** less accurate on these narrow tasks. It also
leaked Markdown fences, which a validator must strip. `qwen2.5:0.5b` is
confirmed as the default rather than merely assumed.

### Tool-calling latency, 2026-09-08

Measured warm, with the tool schema already in context, against three
household utterances and a three-tool schema. This is the evidence base for
the two-tier split in [`design.md`](./design.md).

| Path | Median | Prompt eval | Generation | Correct |
|---|---|---|---|---|
| Declared pattern match | **~0 ms** | — | — | exact |
| `nomic-embed-text:v1.5` nearest-neighbour | ~1,180 ms | — | — | 5 of 6 |
| `llama3.2:1b` tool call | **8.62 s** | 233 tok @ ~1,000 tok/s | 22 tok @ **3.7 tok/s** | 2 of 3 |
| `qwen2.5:3b` tool call | **12.88 s** | 222 tok @ ~340 tok/s | 21 tok @ **1.8 tok/s** | 3 of 3 |

Effectively all latency is token generation. Prompt evaluation runs at
340–1,000 tok/s, models were already resident, and the host has an SSD with
5 GiB free RAM — so neither disk, nor model loading, nor memory is the
constraint. Only CPU is.

`llama3.2:1b` selected `get_weather` for "I watered the tomatoes" on every
attempt, warm and cold, and once returned the tool schema itself as the
argument object. Speed does not rescue a wrong tool call.

Two conclusions follow: a model cannot sit in the path of a routine command,
and agentic reasoning remains worthwhile only where tens of seconds are
acceptable.

An earlier single-shot reading of 96.61 s for `qwen2.5:3b` was discarded as
contaminated by cold load and first-call schema processing.

## Text to speech

`sherpa-onnx` 1.13.7, VITS Piper voices, CPU provider, one thread.

| Voice | Sample rate | Load | Median real-time factor |
|---|---|---|---|
| `en_US-amy-low` | 16 kHz | 2.52 s | 0.175 |
| `en_US-lessac-medium` | 22.05 kHz | 1.90 s | 0.251 |

Both pass the design's 0.8 real-time-factor gate with wide margin. A typical
short household sentence synthesizes in 0.35 to 0.9 seconds.
`en_US-lessac-medium` is the default because its quality is better and its cost
is still low.

## Speech to text

`sherpa-onnx` streaming Zipformer, English, 20 M parameters, one thread,
greedy search, endpointing enabled.

| Precision | Overall real-time factor | First partial result |
|---|---|---|
| float32 | 0.084 | 0.05 to 0.23 s |
| int8 | 0.103 | 0.07 to 0.23 s |

Recognition runs about twelve times faster than real time. **float32 is faster
than int8 on this host**, which is consistent with a CPU that has AVX but no
AVX2 or VNNI to accelerate integer kernels. Quantization is not a speed win
here, so float32 is the default.

Transcription of a locally synthesized sentence returned `O THIS IS REINVOKE D
SPEAKING THROUGH THE SUN ROOUM`, which is a closed loop through the project's
own text-to-speech output. Output is upper case without punctuation, so
capability matching must normalize case.

### Mandarin-English recognition

The streaming
`sherpa-onnx-streaming-zipformer-small-bilingual-zh-en-2023-02-16` model was
tested with its six supplied mixed-language recordings.

| Precision | Overall real-time factor | First partial result |
|---|---|---|
| float32 | 0.131 | 0.04 to 0.40 s |
| int8 | 0.222 | 0.05 to 0.35 s |

The float32 model preserved mixed output such as `昨天是 MONDAY TODAY IS ...`
and ran about 7.6 times faster than real time. The int8 model was slower and
dropped substantially more speech. As with the English recognizer, float32 is
the correct starting point on this AVX-only CPU.

This proves that mixed Mandarin-English recognition fits the host's compute
budget. It does not establish household accuracy, which still requires both
people speaking at normal distances through the Invoke microphone path.

## Chinese-English wake model

The 39 MiB `sherpa-onnx-kws-zipformer-zh-en-3M-2025-12-20` model loaded and ran
with an int8 encoder and joiner, float32 decoder, one thread, and the shipped
Chinese-English keyword list.

| Supplied audio | Detection real-time factor |
|---|---|
| English | 0.054 to 0.076 |
| Chinese | 0.038 to 0.106 |

All four supplied Chinese recordings that contained listed names triggered the
correct configured keyword. The three Chinese negative recordings produced no
trigger. The two English recordings triggered their listed phrases.

A custom Chinese wake phrase still needs to be selected, tokenized with the
model's `phone+ppinyin` tokenizer, and tested for false accepts in household
audio.

## Audio output to the speaker

The host is already paired, trusted, and connected to an Invoke running the
reInvoke RAM platform as a Bluetooth A2DP sink named `reInvoke-RAM`, advertising
Audio Sink, A/V Remote Control Target, and A/V Remote Control.

A synthesized sentence was written to a WAV file and played to the
`bluez_sink.XX_XX_XX_XX_XX_XX.a2dp_sink` PulseAudio sink. Playback completed and
the sink returned to `IDLE`.

**Spoken output therefore already reaches the physical speaker with no new
device-side code.** It travels through reInvoke's existing, accepted playback
path, including the packaged BlueALSA player, the active-PCM lease, and the
mute-first MCU policy.

The negotiated sink format is `s16le` stereo at 44.1 kHz, while the microphone
capture path is `S32_LE` stereo at 48 kHz, so resampling is required in both
directions.

## Estimated round trip

Derived from the measurements above for a short command on one endpoint.

**Tier 1 — reflex, no model in the path:**

| Stage | Expected cost |
|---|---|
| Streaming recognition after speech ends | about 0.3 s |
| Declared pattern match | negligible |
| Deterministic response text | negligible |
| Speech synthesis of a short sentence | 0.35 to 0.9 s |
| Bluetooth transport and buffering | to be measured |

A reflex should answer in roughly one second, leaving comfortable headroom
under the four-second p95 gate.

**Tier 2 — escalation to OpenClaw:**

| Stage | Expected cost |
|---|---|
| Recognition and spoken acknowledgement | about 1 s, same as Tier 1 |
| OpenClaw agent turn | 13 s and up, per the tool-calling table |
| Speech synthesis of the answer | 0.35 to 0.9 s |

The acknowledgement, not the answer, is what must meet the four-second gate.
The answer arrives when it arrives, and the ring carries the wait.

One consequence needs measuring rather than assuming: an answer arriving tens
of seconds late may fall outside the MCU's 1.5-second mute holdoff and require
a fresh authorized unmute cycle.

## What remains unmeasured

* Microphone capture quality from the Invoke array, including far-field
  behavior, and whether any onboard processing is present in the owned path.
* Wake-word false accepts across real household audio.
* Contention when recognition, synthesis, audio transport, and an OpenClaw
  agent turn run together on four cores — and whether Tier 1 needs scheduling
  priority to stay instant while Tier 2 is thinking.
* Whether `qwen2.5:3b` is the right escalation model. It was accurate but slow;
  OpenClaw is currently configured for `llama3.2:1b`, which was faster and
  repeatably wrong on tool selection.
* End-to-end Tier 2 latency including recognition, escalation, and synthesis.
* Whether a late Tier 2 answer needs a fresh unmute cycle after the 1.5-second
  MCU mute holdoff.
* Bluetooth and Wi-Fi coexistence on the device radio.
* Multiple simultaneous A2DP connections from one host adapter.
* Sustained thermal behavior on 2011 hardware.
