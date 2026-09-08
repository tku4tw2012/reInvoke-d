---
title: Wake and language support
description: Chinese wake-word and mixed Mandarin-English posture for reInvoke-d
ms.date: 2026-09-07
ms.topic: concept
---

## Language layers

Wake detection, command recognition, language understanding, and speech
synthesis are separate layers. Supporting Chinese in one does not silently
enable it in all four.

The initial posture is:

| Layer | Initial choice |
|---|---|
| Wake | Configurable Chinese phrase |
| Commands | English |
| Routing | Unicode-safe |
| Responses | English |
| Code-switching | Disabled until household evaluation |

This supports a Chinese wake phrase followed by an English command without
paying the larger bilingual ASR cost on every request.

## Wake word

The selected candidate is:

```text
sherpa-onnx-kws-zipformer-zh-en-3M-2025-12-20
```

It is about 39 MiB extracted and runs at 0.038 to 0.106 real-time factor on the
target AVX-only Mac. Use its chunk-8 int8 encoder and joiner with the float32
decoder, one inference thread, and 16 kHz float32 mono audio.

A raw keyword line has this shape:

```text
<Chinese phrase> :1.5 #0.35 @WAKE_NAME
```

Tokenize it with sherpa-onnx's `phone+ppinyin` tokenizer and the model's
`en.phone` lexicon. The score and threshold are starting values only. The chosen
phrase must pass representative household noise, TV, music, and both speakers.

## Mixed Mandarin-English commands

The measured bilingual candidate is:

```text
sherpa-onnx-streaming-zipformer-small-bilingual-zh-en-2023-02-16
```

Float32 runs at 0.131 real-time factor on the target host and preserves mixed
Chinese-English text. Int8 is slower at 0.222 and drops materially more speech
on this CPU, which has AVX but no AVX2 or VNNI.

Full bilingual mode remains off by default. It becomes a configured option only
after a held-out corpus covers:

* Chinese wake plus English command
* English sentence with Mandarin room, music, plant, or routine names
* Mandarin sentence with English technical nouns
* Full Mandarin household requests
* Unrelated Chinese and English speech that must not trigger an action

## Routing

The text normalizer uses Unicode character classes. It lowercases Latin text and
removes punctuation without deleting Chinese characters. The deterministic
broker may add Chinese exact patterns later, but the Capability Catalog and
Spines remain language-neutral.

The Model Gateway's `qwen2.5:0.5b` model can classify Chinese text, but that does
not make ASR or TTS bilingual. Every enabled language still needs measured
speech input and output.

## Speech output

English uses the measured Piper VITS voice through sherpa-onnx. Chinese and
mixed-language TTS are not selected yet. A bilingual voice must preserve
Response Frame placeholders, run faster than real time on the target CPU, and
pronounce household names acceptably.

Until then, a Chinese wake phrase receives English responses. This is a
deliberate supported mode, not partial failure.

