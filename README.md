# Light

**Convert anything, by conversation.** — an offline, in-browser file converter with a small AI model trained entirely from scratch.

![Offline](https://img.shields.io/badge/offline-100%25-12b76a)
![No server](https://img.shields.io/badge/server-not%20required-12b76a)
![Model](https://img.shields.io/badge/model-4.95M%20params-c8102e)
![Validation](https://img.shields.io/badge/validation-99.66%25-c8102e)
![Stack](https://img.shields.io/badge/stack-vanilla%20HTML%20%2F%20CSS%20%2F%20JS-4f7dff)
![License](https://img.shields.io/badge/license-not%20set-lightgrey)

Attach a file, paste a direct link, or drag anything onto the page — then just tell Light what it should become (*“make it an MP3”*, *“turn this into a PDF”*, *“convert to Excel”*). Everything runs on your machine: no uploads, no accounts, no server, works from `file://`.

| Home | Chat | Generating a file |
| --- | --- | --- |
| ![Home](docs/screenshot-home.png) | ![Chat](docs/screenshot-chat.png) | ![Generating](docs/screenshot-generating.png) |

---

## ✨ Features

- **One chat box, every conversion** — Light’s intent model reads plain English and routes to the right engine.
- **Truly local** — all engines are vendored; the page works offline with no network access.
- **Magic-byte detection** — files are identified by signature (≈60 formats), not by extension, so renamed files still route correctly.
- **Real engines, not wrappers** — vector PDF layout, WASM audio encoders, ffmpeg.wasm for video, and a hand-written XLSX reader/writer.
- **Demo-quality UI** — glassmorphism, animated aurora background, live progress orb with percentage, and a “generating → done” animation.
- **Multi-turn aware** — the model is trained with conversation context (`[ctx:<previous intent>]`), and a deterministic rules layer guarantees formats, identity and safety questions.

## 🔄 Supported conversions

### Documents
| From | To |
| --- | --- |
| PDF | Word `.docx` (exact page layout), Word `.docx` (editable text), TXT, PNG, JPG |
| Word `.docx` | PDF (vector text), TXT, HTML |
| TXT · MD · HTML · source code (`.js`, `.ts`, `.py`, `.css`, `.sql`, …) | PDF, HTML, TXT, CSV, TSV, JSON, Excel |

### Spreadsheets & data
| From | To |
| --- | --- |
| Excel `.xlsx` | CSV, TSV, JSON, JSONL, XML, YAML, TXT, HTML |
| CSV · TSV · JSON · JSONL · XML · YAML · plain text | Excel (`.xlsx` / `.xls`*), CSV, TSV, JSON, JSONL, XML, YAML, TXT, HTML |

\* `.xls` requests produce a modern `.xlsx` workbook, since the legacy binary format cannot be written in a browser.

### Images
| From | To |
| --- | --- |
| PNG · JPG · WebP · GIF · BMP · SVG · AVIF · HEIC · ICO | PNG, JPG, WebP, PDF |
| PDF (multi-page → ZIP) | PNG, JPG |

### Audio
| From | To |
| --- | --- |
| MP4 · WebM · MOV · AVI · MKV · M4A · AAC · OGG · WAV · FLAC · Opus · WMA · AIFF | MP3, WAV, OGG, FLAC, AIFF (fast — WebAudio decode + LAME / Vorbis / FLAC WASM encoders) |
| Audio or video | M4A / AAC, Opus (via MediaRecorder, encoded in real time) |

### Video
| From | To |
| --- | --- |
| Any audio/video | MP4, WebM, MOV, AVI, MKV, FLV, 3GP, M4V, GIF (ffmpeg.wasm) |
| Video | PNG / JPG frame grab |

### 3D & links
- **STL ⇄ OBJ** mesh conversion (pure JS).
- **Direct file links** — paste `https://…/clip.mp4` with a target format; Light fetches it (when the host allows CORS) and converts it like a local file.

## 🚀 Quick start

No installation, no build step, no server required.

1. Download or clone this folder.
2. Open `index.html` in a modern browser (Chrome, Edge or Firefox recommended).

That’s it — the app runs from `file://`. If you prefer a classic local server:

```bash
python -m http.server
# open http://localhost:8000
```

**Notes**
- Audio/video conversions lazy-load a ~31 MB WASM engine the first time (instant afterwards). Document, image and data conversions never touch it.
- M4A/AAC/Opus encoding uses the browser’s MediaRecorder, so those three run in real time (a 45 s track takes ~45 s).
- Video transcoding uses single-thread WASM: small clips take seconds, large files can take minutes.

## 🧠 How it works

```
                 ┌─────────────┐
 attach file ───▶│  Detector   │  magic-byte signature → real file type
 or link         └──────┬──────┘
                        │
 “make it MP3” ─▶┌──────▼──────┐
                 │   Router    │  Light model (93 intents) + deterministic rules
                 └──────┬──────┘
                        │ target format
                 ┌──────▼──────┐
                 │  Pipelines  │  picks an engine for the source × target pair
                 └──────┬──────┘
        ┌───────────────┼────────────────┬───────────────┐
        ▼               ▼                ▼               ▼
   PDF / Word      Images & data     Audio (WASM)   Video (ffmpeg.wasm)
   vector PDF,     canvas, XLSX,      LAME, Vorbis,   inline worker +
   docx building   JSZip, js-yaml     FLAC, AIFF      embedded WASM binary
                        │
                 ┌──────▼──────┐
                 │  File card  │  download button, never uploaded anywhere
                 └─────────────┘
```

The model decides *what you mean*; the engines do the work — the two never blur together, so a bad guess can never corrupt a file.

## 📦 The Light model

| Field | Value |
| --- | --- |
| Type | 2-layer neural network (softmax intent classifier) — *not* a transformer, by design |
| Parameters | **4,952,893** |
| Input | 12,288 hashed n-gram features (word unigrams, word bigrams, char 3-grams, char 4-grams) |
| Hidden layer | 400 units, ReLU, dropout 15 % during training |
| Output | **93 classes** — 69 target formats + 24 conversational intents |
| Optimizer | Adam implemented from scratch, weight decay 1e-4 |
| Training data | **751,900 unique examples**, zero duplicates |
| Validation accuracy | **99.66 %** (75,190 held-out examples) |
| Held-out challenge set | 74.5 % for the model alone on adversarially novel phrasing — the shipped router adds deterministic rules on top and passes all real-world probe tests |
| Quantization | int8 (≈ 1 byte/parameter) |
| Context | multi-turn markers `[ctx:<previous intent>]` baked into training and inference |
| Container | custom binary `model/light-model.bin` (magic header `LGHT`), plus `model/light-model.js` — a masked loader used only because browsers cannot read local binaries from `file://` |

**Why an MLP instead of a transformer?** The task is a closed-set classification (69 target formats, 24 conversational intents) over short utterances. A hashed-n-gram MLP reaches 99.66 % validation at ~5 M parameters, trains in minutes on a CPU, runs instantly in the browser, and cannot hallucinate a format that does not exist. Conversation quality — variety, no repetition, context references — lives in the dialogue layer, not in generation.

**Conversational intents:** greeting, wellbeing, capabilities, thanks, goodbye, acknowledgment, identity, creator, skepticism, how-works, privacy, question-file, confused, joke, compliment, insult, age, origin, languages, limitations, quality, free, speed, unknown.

## 🗂 Repository layout

```
Light/
├── index.html                    # the entire UI
├── styles.css                    # glassmorphism theme + animations
├── light.js                      # ALL app code in one file: detector, model router,
│                                 #   PDF⇄Word, image/audio/recorder/video/data/xlsx/
│                                 #   mesh/media engines, pipelines, dialogue, app
├── conversion.js                 # ALL third-party engines in one file
│                                 #   (pdf.js + inlined worker, mammoth, html2pdf, docx,
│                                 #    jsPDF, JSZip, js-yaml, lamejs, Ogg/FLAC WASM,
│                                 #    ffmpeg worker + core sources)
├── conversion.wasm.js            # 31 MB ffmpeg WASM binary (base64, lazy-loaded)
├── model/
│   ├── light-model.bin           # the trained model (binary, LGHT container)
│   └── light-model.js            # masked loader for file:// usage
├── model-train/
│   ├── generate-dataset.js       # deterministic dataset generator (no duplicates)
│   ├── train.js                  # from-scratch trainer (Adam, dropout, int8 export)
│   ├── dataset-stats.json        # dataset statistics
│   └── training-report.json      # metrics, per-class P/R/F1, confusion matrix
└── docs/                         # screenshots used above
```

## 🔁 Retrain the model

Only Node.js is needed (no ML libraries — the trainer is hand-written).

```bash
cd model-train
node generate-dataset.js   # writes dataset.jsonl  (~751,900 unique examples)
node train.js              # ~30 min on a modern CPU
                           # writes ../model/light-model.bin + light-model.js
```

The dataset is generated from templates with synonym slots, context markers, typos and deduplication on every normalized phrase; the trainer prints validation accuracy, the held-out challenge score and probe sentences.

## 🔒 Privacy

- **Nothing is uploaded.** There is no backend in this project.
- Files are read with the browser’s File APIs, converted in memory, and delivered as a download.
- The only bytes ever fetched from disk are the app’s own assets (engines + model).

## ⚠️ Limitations (honest list)

- **No OCR** — scanned/image-only PDFs cannot be turned into editable text.
- **Detect-only formats** — Light recognises but cannot yet convert: legacy `.doc` / `.xls` / `.ppt`, Apple Pages/Keynote, WPS, PSD, AI, EPS, MOBI/AZW3/DjVu, 7z/RAR/TAR.GZ, FBX, DWG, DXF.
- **No platform downloads** — YouTube/Instagram links are not supported (and never will be here); only direct file links that permit CORS.
- **Video transcoding is slow** — single-thread WASM; a 58 MB MP4 → AVI took ~8 minutes in testing, small clips take seconds.
- **M4A/AAC/Opus are real-time**; Opus lands in WebM on Chrome and in Ogg on Firefox.
- **HEIC decoding** depends on the browser’s own codec support.
- Cannot create EXE, APK, ISO, PSD or installers.

## 🧪 Verification

Tested end-to-end in headless Chrome against real files (a 4-page PDF directory and a 44-second, 58 MB MP4):

- **24/24 conversions passed** in the full suite — documents (exact & editable), PDF→TXT/PNG/JPG, all audio targets (MP3, WAV, OGG, FLAC, AIFF, M4A, Opus), images, data and spreadsheet round-trips, and video via ffmpeg.
- The bundled build was re-tested over both `http://` and `file://`.
- Dataset verified programmatically: 751,900 lines = 751,900 unique normalized phrases.
- Model file verified: `LGHT` binary container parses, int8 weights dequantize, all metrics reproduced.

## 🧩 Third-party components

| Component | Purpose | License |
| --- | --- | --- |
| [pdf.js](https://github.com/mozilla/pdf.js) | PDF parsing & rendering | Apache-2.0 |
| [mammoth.js](https://github.com/mwilliamson/mammoth.js) | Word → HTML | BSD-2-Clause |
| [html2pdf.js](https://github.com/eKoopmans/html2pdf.js) | HTML → PDF (fallback renderer) | MIT |
| [docx](https://github.com/dolanmiu/docx) | Word document generation | MIT |
| [jsPDF](https://github.com/parallax/jsPDF) | Vector PDF generation | MIT |
| [JSZip](https://github.com/Stuk/jszip) | ZIP/XLSX containers | MIT |
| [js-yaml](https://github.com/nodeca/js-yaml) | YAML parsing | MIT |
| [lamejs](https://github.com/zhuker/lamejs) | MP3 encoding | LGPL-3.0 |
| [wasm-media-encoders](https://github.com/arseneyr/wasm-media-encoders) | Ogg Vorbis encoding | MIT |
| [libflacjs](https://github.com/mmig/libflac.js) / [@audio/encode-flac](https://github.com/audiojs/encode-flac) | FLAC encoding | libFLAC (BSD) |
| [ffmpeg.wasm](https://github.com/ffmpegwasm/ffmpeg.wasm) + FFmpeg core | Audio/video transcoding | MIT wrapper · FFmpeg LGPL-2.1+ |

> If you plan to redistribute, review the LGPL components (LAME, FFmpeg) for your use case.

## 👤 Author

**Khwaja** — creator of Light and this toolkit.

Light identifies itself accordingly when asked: *“I’m Light — a small AI model designed for this desktop toolkit by Khwaja.”*

## 📄 License

No license file is included yet. Add one (MIT, Apache-2.0, …) before publishing if you want others to reuse the code — and keep the third-party license notes above in mind.
