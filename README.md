# UtilVision: In-Browser Edge Vision Pipeline for Digital Utility Meters

UtilVision is an edge-native, browser-based computer vision pipeline engineered to detect and parse digital utility meter LCD counters in real time directly on client hardware. Designed for automated utility meter reading (AMR) and tenant billing platforms, UtilVision executes end-to-end neural inference entirely within the client web browser session with zero server inference cost, zero cloud API dependencies, complete tenant data privacy, and full offline functionality.

Production Deployment: [https://utilvision.vercel.app/](https://utilvision.vercel.app/)  
Architecture and Benchmarks: [https://utilvision.vercel.app/about.html](https://utilvision.vercel.app/about.html)  
Hugging Face Model Repository: [https://huggingface.co/majorpurple/utilvision-onnx](https://huggingface.co/majorpurple/utilvision-onnx)

---

## Technical Overview

General-purpose commercial OCR services and linguistic vision models struggle with digital utility meters, frequently achieving below 40% exact-match accuracy due to three physical failure modes:
1. Fragmented Seven-Segment Numerals: Physical unlit gaps between segments cause generic OCR engines to hallucinate phonetic letters (for example, misreading 0 as N, or 8 as B).
2. Faceplate Clutter and Unit Symbols: High-density printed serial numbers, barcodes, brand logos, and adjacent measurement units (such as kWh or kW) distract text localizers.
3. Leading Zero Truncation: Utility billing ledgers demand verbatim leading zero preservation (such as `007888.54`), which standard numeric parsers automatically discard.

UtilVision solves this through a decoupled two-stage architecture that isolates the active digital register before sequence decoding, paired with a restricted 11-token CTC vocabulary that mathematically prevents alphanumeric hallucinations.

---

## Pipeline Architecture

```mermaid
flowchart LR
    A["Raw Photo / Live Video Feed"] --> B["Stage 2 Detector (Anchor-Free FPN)"]
    B --> C["Canonical 96px Aspect-Preserving Normalization"]
    C --> D["Stage 3 Recognizer (SVTR Token Attention)"]
    D --> E["11-Token CTC Sequence Decoder"]
    E --> F["Verified Numeric Reading (kWh Ledger Entry)"]
```

### 1. Stage 2 Detector (2.62M Parameters / 10.6 MB)
* Anchor-free feature pyramid network (YOLO11n) containing 2,624,389 weights, optimized for LCD register localization.
* Input frame resized to 640x640 letterbox buffer.
* Isolates the primary numeric register from complex background faceplates, rejecting barcodes, manufacturer ratings, and peripheral dials.
* Equipped with conditional optical test-time augmentation (TTA) that automatically triggers a 75% center zoom crop on distant or low-confidence captures.

### 2. Canonical 96px Normalization
* Standardizes cropped meter displays to a fixed canonical height of 96 pixels while dynamically calculating width to preserve natural aspect ratios.
* Eliminates vertical stroke distortion and preserves fine-pitch decimal point spacing across varying single-phase and three-phase aspect ratios (from 2.5:1 up to 5:1).

### 3. Stage 3 Recognizer (15.51M Parameters / 31.4 MB FP16 / 62.3 MB FP32)
* Single Visual Token Representation (SVTR) architecture containing 15,506,550 weights, modeling character-to-character spatial dependencies without recurrent vanishing gradients.
* Combined end-to-end pipeline parameter footprint: 18.13M parameters (18,130,939 total weights).
* Constrained to an 11-token vocabulary (digits 0 through 9 plus decimal point).
* Zero hallucination: extraneous alphanumeric characters and measurement units cannot be emitted.
* Dual precision support: default FP16 half precision (31.4 MB) halves memory bandwidth while maintaining exact floating-point recognition accuracy, with on-demand FP32 (62.3 MB) availability.

---

## Benchmark Results

Evaluated on a master held-out test split under a Strict Physical Meter-Grouped Stratification Protocol:

| Metric | UtilVision Production Performance | Benchmark Specification |
| :--- | :---: | :--- |
| Display Localization Recall | 100% | Bounding box IoU > 0.50 |
| Digit-Level Accuracy | 99.4% | Normalized Levenshtein distance |
| Master Exact Match | 96.5% | Verbatim character and decimal equality |
| Industrial Multi-Line (Set C) | 96.2% | Exact match on 5:1 three-phase registers |
| Test Partition Stratification | 150 Unique Meters | Zero meter ID overlap between splits |

---

## Edge Runtime and Hardware Latencies

Inference latencies measured using the High Resolution Time API (`performance.now()`) across 50 warm iterations:

| Environment | Execution Provider | Precision | End-to-End Latency |
| :--- | :--- | :---: | :---: |
| Apple M2 / NVIDIA RTX 4060 | WebGPU (WGSL Shaders) | FP16 | 78.7 ms (Stage 2: 31.2 ms, Stage 3: 47.5 ms) |
| Intel Core i7-13700H | WASM SIMD (4 Threads) | FP16 | 214.3 ms (Stage 2: 74 ms, Stage 3: 140 ms) |
| OnePlus 13R (Snapdragon 8 Gen 3) | WASM SIMD (4 Cores) | FP16 | Sub-1s / 650 ms to 950 ms |
| OnePlus 11R (Snapdragon 8+ Gen 1) | WASM SIMD (4 Cores) | FP16 | 1.0s to 1.5s |
| Cloud Vision APIs (Broadband) | Remote HTTPS Endpoint | N/A | 1,100 ms to 2,200 ms (Roundtrip) |
| Cloud Vision APIs (Cellular) | Remote HTTPS Endpoint | N/A | 2,500 ms to 5,000+ ms (Roundtrip) |

---

## In-Browser Engineering Stack

* Multi-Threaded WebAssembly: Configured with `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: credentialless` headers via Vercel, enabling shared memory parallelism (`SharedArrayBuffer`) across physical CPU cores without third-party credential blocking.
* Split Bundle Delivery: Desktop clients load `ort.all.min.js` (710 KB) for native WebGPU hardware acceleration; mobile browsers load the stripped `ort.min.js` (300 KB) for direct 4-core WASM SIMD execution, avoiding experimental mobile GPU/NPU driver compilation stalls.
* Persistent Model Caching: Model binaries are saved locally via the browser CacheStorage API (`utilvision-models-v1`) upon initial download. Subsequent sessions load models instantly from local storage, requiring zero network bandwidth.
* Camera Stream Amortization: During live video inspection, the Stage 2 Detector runs every 5th frame (`DETECT_EVERY = 5`), reusing the detected bounding box across intermediate frames to maintain responsive video playback.
* Offline Operation: The application functions completely disconnected from the network once cached, enabling reliable meter audits in subterranean basements and metal enclosure cabinets.

---

## Repository Structure

```text
├── index.html          # Core interactive camera and file inspection reader
├── about.html          # Technical documentation, interactive evolution chart, and benchmarks
├── vercel.json         # Vercel deployment configuration (COOP/COEP security headers)
├── models/
│   ├── stage2.onnx        # Stage 2 Detector ONNX graph (10.6 MB)
│   ├── stage3_fp16.onnx   # Stage 3 Recognizer FP16 ONNX graph (31.0 MB)
│   └── stage3_fp32.onnx   # Stage 3 Recognizer FP32 ONNX graph (62.0 MB)
└── samples/
    ├── img (104).jpg      # Residential single-phase reference sample (Set A)
    ├── img (118).jpg      # Commercial utility reference sample (Set B)
    └── img (110).jpg      # Industrial three-phase reference sample (Set C)
```

---

## Model Repository and Open-Access Weights

The production ONNX model graphs are hosted on Hugging Face Hub under AGPL-3.0 copyleft terms:
* Hugging Face Model Repository: [https://huggingface.co/majorpurple/utilvision-onnx](https://huggingface.co/majorpurple/utilvision-onnx)
* Direct Edge Fetching: Model files are delivered directly to client browser sessions via cross-origin requests (`Access-Control-Allow-Origin: *`) and permanently stored in client `CacheStorage` for zero-bandwidth subsequent visits.

---

## Production Deployment

This project is deployed to production on Vercel:
* Main Production URL: [https://utilvision.vercel.app/](https://utilvision.vercel.app/)
* Architecture and Benchmarks: [https://utilvision.vercel.app/about.html](https://utilvision.vercel.app/about.html)

Vercel provides the requisite security headers (`COOP: same-origin` and `COEP: credentialless`) necessary to unlock high-performance multi-threading across Web Workers without browser origin isolation errors.
