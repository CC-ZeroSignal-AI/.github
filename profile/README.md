# ZeroSignal

**Edge-native intelligence. Zero signal required.**

ZeroSignal is an offline-first RAG AI assistant for Android that runs entirely on-device using NexaSDK. It combines Llama 3.2 3B inference with local vector search over downloadable knowledge packs — delivering expert-level answers in emergency medicine, wilderness survival, vehicle repair, and more, with zero cloud dependency.

## What It Does

- **Offline AI Chat** — Ask natural-language questions and get grounded, context-aware answers powered by on-device Llama 3.2 3B inference
- **Local RAG Pipeline** — Questions are vectorized and matched against pre-embedded knowledge chunks using ObjectBox's HNSW vector index; top matches are injected into the LLM prompt for factual, hallucination-resistant responses
- **Pack Store** — Browse and download curated knowledge packs (emergency field medicine, wilderness survival, Yosemite navigation, vehicle mechanics, daily life essentials) from a FastAPI backend; downloads are cursor-paginated and resumable
- **NPU-Accelerated** — Prioritizes Qualcomm NPU acceleration via NexaSDK for fast inference on Snapdragon devices, with automatic CPU/GPU fallback

## Architecture

```
ZS-android/                          ZS-web/
├── app/          (Compose UI)        ├── server/    (FastAPI read-only API)
├── core/domain/  (Models, UseCases)  └── pipeline/  (Scrape → Chunk → Embed → Upload)
├── core/ai/      (NexaSDK Engine)
└── core/data/    (ObjectBox, Networking)
```

**Android** — 4-module Clean Architecture with Hilt DI, Jetpack Compose, and Material3
**Backend** — FastAPI serving pre-computed packs from Qdrant Cloud (deployed on Vercel)
**Pipeline** — Local Python tool that scrapes web sources, chunks text, summarizes with LLM, generates embeddings (all-MiniLM-L6-v2), and uploads to Qdrant

## How to Build and Run

### Prerequisites

- Android Studio Hedgehog (2023.1.1) or later
- JDK 17
- A physical Snapdragon Android device (min SDK 26 / Android 8.0)
- ~6GB free storage on device (for model files)

### Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/CC-ZeroSignal-AI/ZS-android.git
   cd ZS-android
   ```

2. **Open in Android Studio**
   - File → Open → select the `ZS-android` directory
   - Wait for Gradle sync to complete

3. **Connect a Snapdragon device**
   - Enable USB debugging on your device
   - Connect via USB and verify with `adb devices`

4. **Build and run**
   ```bash
   ./gradlew installDebug
   ```
   Or use the Run button in Android Studio targeting your connected device.

5. **First launch — Setup**
   - The app will detect your device hardware
   - It downloads the Llama 3.2 3B model (~5.5GB total: NPU + GGUF fallback) from HuggingFace
   - Progress is shown in real-time; wait for "Ready"

6. **Use the app**
   - **Chat tab** — Ask questions; the app retrieves relevant context from local knowledge packs and streams an AI-generated answer
   - **Pack Store tab** — Browse and download additional knowledge packs from the ZeroSignal server

### Backend

No setup needed — the Pack Store backend is already deployed at `https://zerosignal-web.vercel.app` and the app comes pre-configured to use it. Knowledge packs are available out of the box.

## Where and Why NexaSDK Is Used

NexaSDK is the core inference engine powering all on-device AI in ZeroSignal. It is integrated in the `:core:ai` module:

### Files

| File | Role |
|------|------|
| `core/ai/src/.../engine/NexaSdkEngine.kt` | Main inference wrapper — loads models, manages NPU/CPU runtime, streams token generation |
| `core/ai/src/.../repository/NexaModelRepository.kt` | Downloads model files from HuggingFace (NPU plugin + GGUF fallback) |
| `core/ai/src/.../util/ModelFileListingUtil.kt` | Dynamic discovery of model files from HuggingFace repo manifests |
| `core/ai/src/.../di/AiModule.kt` | Hilt DI — provides NexaSDK engine with device capability detection |
| `core/ai/build.gradle.kts` | Declares NexaSDK dependency (`ai.nexa.sdk:nexa:0.0.22`) |

### Why NexaSDK

1. **NPU-first inference** — NexaSDK enables direct execution on Qualcomm's Neural Processing Unit, delivering faster and more power-efficient inference than CPU-only approaches. ZeroSignal uses a dual-runtime strategy: it attempts NPU loading first (`NexaAI/Llama3.2-3B-NPU-Turbo-NPU-mobile`), then falls back to CPU/GPU via GGUF (`bartowski/Llama-3.2-3B-Instruct-GGUF`).

2. **Streaming generation** — NexaSDK's `generateStreamFlow()` API enables real-time token streaming to the Compose UI, creating a responsive chat experience identical to cloud-based assistants.

3. **Chat template support** — `applyChatTemplate()` correctly formats multi-turn conversations with Llama 3.2's expected format, including system prompts with RAG context injection.

4. **Lightweight integration** — A single Maven dependency (`ai.nexa.sdk:nexa:0.0.22`) replaces what would otherwise require bundling and managing ONNX Runtime, custom JNI bridges, or other heavyweight inference frameworks.

### Inference Flow

```
User Question
    ↓
ObjectBox HNSW vector search → Top-K relevant chunks
    ↓
PromptBuilder assembles Llama 3.2 chat template with context
    ↓
NexaSdkEngine.streamCompletion()
    ├── applyChatTemplate()
    └── generateStreamFlow(maxTokens=2048)
    ↓
Streaming tokens → Compose UI
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| AI Inference | NexaSDK 0.0.22 (Llama 3.2 3B) |
| Vector DB | ObjectBox 4.0.3 (HNSW, 768-dim) |
| UI | Jetpack Compose + Material3 |
| DI | Hilt 2.51.1 + KSP |
| Networking | Retrofit2 + Moshi + OkHttp |
| Backend | FastAPI + Qdrant Cloud |
| Embeddings | all-MiniLM-L6-v2 (384-dim, server-side) |
| Language | Kotlin 1.9.24 |

## Knowledge Packs

Five sample packs ship for this hackathon, but the architecture is designed to scale to any domain with new packs addable on demand:

| Pack | Domain | Use Case |
|------|--------|----------|
| Emergency Field Pack | Emergency Response | First aid, CPR, hyperbaric medicine |
| Wilderness Survival | Survival | Fire-making, water purification, shelter |
| Yosemite Navigation | National Parks | Trails, geography, visitor guidance |
| Vehicle Mechanic | Auto Repair | Roadside diagnostics, engines, batteries |
| Offline Daily Life | General | Weather, maps, emergency services |

## Roadmap

- **On-device PDF ingestion** — Drop any PDF into ZeroSignal and have it chunked, embedded, and searchable locally — turning personal documents into queryable offline knowledge
- **Agentic image search** — A dedicated agent that indexes and searches photos on your phone's gallery using natural-language descriptions, powered by on-device vision inference via NexaSDK
- **Task agent** — Chain multiple on-device tools together with an agentic orchestration layer — all running offline
- **Expandable Pack Store** — Community and on-demand pack creation for any domain, from medical references to trade manuals

## Team

Built by engineers who believe the future of AI isn't bigger servers — it's smarter devices.

---

**ZeroSignal = AI that works even when the signal doesn't.**
