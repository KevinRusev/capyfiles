# On-device AI

Capy Files runs five AI features entirely in the browser. Nothing about a photo or a recording is sent to a server: the
model comes to the data, not the other way round.

## The models

| Feature | Model | Licence | Why this one |
| --- | --- | --- | --- |
| Background removal | **BiRefNet** (Lite 512 and 1024, full 1024 in fp16) | MIT | Sharp edges on hair and products, and a licence that allows commercial use. It replaced a popular model whose licence doesn't |
| 4× upscaling | **RealPLKSR** | MIT | Good detail on photos at a size a laptop GPU handles; images are processed in overlapping tiles |
| Face blur | **YuNet** | MIT | Tiny, fast face detector, run on overlapping tiles so small faces in big photos are found; the blur is applied locally |
| Transcription | **Whisper** base, small, large-v3 turbo | Apache-2.0 / MIT | Multilingual speech-to-text with automatic language detection |
| OCR | **Tesseract** (LSTM) | Apache-2.0 | Proven text recognition in dozens of languages, compiled to WebAssembly |

Every model file is pinned to an exact revision and checked by size and SHA-256 before it's published. The licence
compliance check covers models as well as code.

## How inference runs

- **Transformers.js + ONNX Runtime Web** run the neural networks; Tesseract runs as its own WebAssembly build.
- Inference always happens in a **Web Worker**, so the page never freezes.
- **Backend selection per device:**
  1. **WebGPU with half precision** if the GPU supports 16-bit shaders: the fastest path, often 5–10× quicker than the
     CPU.
  2. **WebGPU in full precision** otherwise.
  3. **WebAssembly on the CPU**, using SIMD and multiple threads when the page is cross-origin isolated.
- The choice is automatic: on a device without WebGPU the tool is slower, but it works the same.

## Delivering big models to a browser

Model files range from a few megabytes to several hundred. That has to work on hotel Wi-Fi.

- **Size first.** Before the first run, each AI tool shows how much it will download for *this* device (the GPU and CPU
  paths use different files), or that the model is already on the device.
- **Progress** is shown during the download.
- **Resumable downloads.** If the connection drops mid-file, the download continues from the last received byte with an
  HTTP range request, after checking the file hasn't changed on the server. Server errors are retried with backoff.
- **Cached once.** Models are stored in the browser's cache storage, so the second run starts immediately and works
  offline. Settings show how much space they take, with one button to delete them.
- **Self-hosted.** All weights, runtimes and language data are served from the site's own edge storage, never from a
  third-party CDN. A test fails the build if runtime code points to an outside model host.

## Whisper in a tab

Transcription is the heaviest feature:

- The user chooses **fast** or **accurate**; the device decides the rest. Fast uses **base**; accurate uses
  **large-v3 turbo** on a WebGPU device and **small** on the CPU. The download size is shown up front.
- Audio is decoded and resampled locally, then transcribed **30 seconds at a time** with Whisper's own long-form method:
  when a window ends mid-sentence, the next one starts where that sentence began, so nothing is cut off or doubled.
- Whisper tends to invent text for silence. Silent stretches are skipped before they reach the model, and phrases it
  typically "hears" in quiet passages are filtered out.
- The spoken language is detected automatically. Output is plain text or timed subtitles (SRT, WebVTT).

## Privacy, verified

- Files are read with browser APIs, processed in memory and written back as downloads. No request carries file data.
- The Content Security Policy only lets the page talk to its own origin plus the payment and sign-in providers, so even
  a bug couldn't quietly send a file somewhere else.
