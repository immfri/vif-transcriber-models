# VIFUS — On-Device Speech Models

This repository hosts the on-device speech-processing models used by the
VIFUS Android app (`de.vif.transcriber`) that have **no publicly available
pre-converted release elsewhere** — a German Whisper fine-tune and a
custom punctuation model. Both are served as loose ONNX files (no
archive/unpacking step) so they can be downloaded directly by the app's
[`DownloadManager`](https://developer.android.com/reference/android/app/DownloadManager)-based
model loader.

Two other models used by the same app (streaming ASR alternative and
speaker identification) are **not** hosted here — they are downloaded
directly from their own official sources. See [Models used but not
hosted here](#models-used-but-not-hosted-here) below.

## Release

All files are attached to the [`models-v1`](https://github.com/immfri/vif-transcriber-models/releases/tag/models-v1)
release as flat, individually downloadable assets (no `.tar`/`.zip`
archive) so the app can fetch exactly the files it needs.

## Models in this repository

### `whisper_small_de` — German Whisper Small, ONNX int8

| | |
|---|---|
| Base model | [`openai/whisper-small`](https://huggingface.co/openai/whisper-small) (OpenAI, MIT license) |
| Fine-tune | [`NickyBricks/whisper-small-de-finetuned`](https://huggingface.co/NickyBricks/whisper-small-de-finetuned) (German, Apache-2.0) |
| Files | `whisper_small_de__encoder.int8.onnx`, `whisper_small_de__decoder.int8.onnx`, `whisper_small_de__tokens.txt` |

**Why this repo hosts it:** `NickyBricks/whisper-small-de-finetuned` is
published only as a HuggingFace `transformers` checkpoint (PyTorch
safetensors). No sherpa-onnx-compatible ONNX export of this specific
fine-tune was publicly available anywhere at the time of integration, so
the conversion below was done in-house.

**Exact conversion pipeline used:**

1. Downloaded the fine-tuned checkpoint from
   `NickyBricks/whisper-small-de-finetuned` (HuggingFace `transformers`
   format, `WhisperForConditionalGeneration`).
2. Converted the HuggingFace checkpoint into the original **OpenAI
   Whisper checkpoint format** (the `.pt` layout OpenAI's own `whisper`
   package expects), since the ONNX export tooling in step 3 operates on
   that format rather than the HF `transformers` layout directly.
3. Exported to ONNX using **k2-fsa's official Whisper export script**
   from the [`sherpa-onnx`](https://github.com/k2-fsa/sherpa-onnx)
   project (the same tooling k2-fsa uses to publish all their own
   `sherpa-onnx-whisper-*` models on HuggingFace) — producing a matching
   encoder/decoder ONNX graph pair plus a `tokens.txt` vocabulary file
   compatible with sherpa-onnx's `OfflineRecognizer` / `Whisper` API.
4. Quantized both the encoder and decoder graphs to **int8** (dynamic
   quantization) to reduce model size and speed up on-device CPU
   inference — this is what the `.int8.onnx` suffix indicates.
5. **Verified** the resulting ONNX model against the sherpa-onnx Python
   bindings on a held-out set of real German audio samples, comparing
   output transcripts against the original PyTorch/`transformers`
   reference pipeline — confirmed **identical transcript output**
   between the ONNX-converted model and the PyTorch reference before
   integrating it into the app.

No further fine-tuning, retraining, or architecture changes were made —
the acoustic/language behavior of the model is unchanged from
`NickyBricks/whisper-small-de-finetuned`; only the serialization format
and numeric precision changed.

**License:** Apache-2.0 (inherited from `openai/whisper-small` → MIT,
and from the `NickyBricks` fine-tune → Apache-2.0). See the fine-tune's
[model card](https://huggingface.co/NickyBricks/whisper-small-de-finetuned)
for the authoritative license statement.

### `punct_de` — German punctuation restoration, CNN-BiLSTM

| | |
|---|---|
| Origin | Trained from scratch (own work, no third-party base model) |
| Architecture | CNN-BiLSTM, compatible with sherpa-onnx's `OnlinePunctuation` API |
| Files | `punct_de__model.int8.onnx`, `punct_de__bpe.vocab` |

Restores punctuation for ASR output that doesn't already produce its own
sentence punctuation (needed for the streaming `zipformer_de` model —
Whisper already outputs its own punctuation and does not use this
model). Trained on German text data with a byte-pair-encoding (BPE)
vocabulary; exported to ONNX and quantized to int8.

**Known limitation (by design):** sherpa-onnx's built-in case-restoration
(`std::toupper` on raw bytes) is not UTF-8-safe and would corrupt German
uppercase umlauts (Ü/Ö/Ä) at the start of capitalized nouns. To avoid
this, the model was trained with case labels held constant — it inserts
**punctuation marks only**; capitalization is left exactly as the
upstream ASR model produced it.

**License:** own work, no upstream license constraints.

## Models used but not hosted here

For completeness — these two models are also used by VIFUS but are
downloaded directly from their own official public sources (see the
app's `ModelManager` class):

| Model | Official source |
|---|---|
| `zipformer_de` (streaming ASR, faster/lower-quality alternative) | [`csukuangfj/sherpa-onnx-streaming-zipformer-de-kroko-2025-08-06`](https://huggingface.co/csukuangfj/sherpa-onnx-streaming-zipformer-de-kroko-2025-08-06) on HuggingFace (k2-fsa maintainer account) |
| `speaker_id` (speaker embedding / diarization) | [`k2-fsa/sherpa-onnx`](https://github.com/k2-fsa/sherpa-onnx/releases/tag/speaker-recongition-models) GitHub release, `wespeaker_en_voxceleb_resnet34_LM.onnx` |

## Usage

These files are fetched automatically by the VIFUS Android app's
`ModelManager` on demand (WLAN by default) — end users do not need to
download anything from this repository manually. Direct download links
follow the pattern:

```
https://github.com/immfri/vif-transcriber-models/releases/download/models-v1/<filename>
```
