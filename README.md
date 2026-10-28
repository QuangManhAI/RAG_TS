# Scientific Literature RAG System (TypeScript)

[![TypeScript](https://img.shields.io/badge/TypeScript-5.7-3178C6?style=flat&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![NestJS](https://img.shields.io/badge/NestJS-11.0-E0234E?style=flat&logo=nestjs&logoColor=white)](https://nestjs.com/)
[![React](https://img.shields.io/badge/React-19.0-61DAFB?style=flat&logo=react&logoColor=black)](https://react.dev/)
[![Vite](https://img.shields.io/badge/Vite-6.4-646CFF?style=flat&logo=vite&logoColor=white)](https://vitejs.dev/)
[![Three.js](https://img.shields.io/badge/Three.js-R3F-black?style=flat&logo=threedotjs&logoColor=white)](https://threejs.org/)
[![Qdrant](https://img.shields.io/badge/Qdrant-Vector_DB-DC2626?style=flat&logo=qdrant&logoColor=white)](https://qdrant.tech/)
[![MongoDB](https://img.shields.io/badge/MongoDB-7.0-47A248?style=flat&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![CUDA](https://img.shields.io/badge/Hardware-NVIDIA_CUDA_%26_Metal-76B900?style=flat&logo=nvidia&logoColor=white)](https://developer.nvidia.com/cuda-toolkit)

An end-to-end, high-performance Retrieval-Augmented Generation (RAG) platform implemented in **TypeScript / NestJS 11** with an interactive **3D Vector Knowledge Tree** frontend. The system performs dense semantic search over scientific papers in **Qdrant**, reranks candidates with **PageRank author authority**, accelerates LLM inference locally via **`node-llama-cpp` (CUDA/Metal)**, and provides **offline-first chat persistence via IndexedDB (Dexie.js)**.

---

## 🏛️ System Architecture

```mermaid
flowchart TD
    subgraph DATA_TIER["Storage & Data Tier"]
        Mongo[("MongoDB (Auth, Users, Sessions)\nTunneled or Local")]
        Qdrant[("Qdrant Vector Database\n(scientific_papers_gold)")]
        MiningData[("Mining Artifacts\n• PageRank Graph\n• Association Rules")]
        LLMWeights[("Quantized LLM Weights\nQwen2.5-7B Instruct GGUF\n(node-llama-cpp CUDA)")]
    end

    subgraph BACKEND["Backend Service (backend_rag: NestJS 11 :3000)"]
        Swagger["OpenAPI 3.0 / Swagger UI\n(/api/v1/docs & /api/v1/docs-json)"]
        Embedder["Embedding Engine\n(Nomic v1.5 with 'search_query:')"]
        Retriever["Qdrant Vector Retriever\n(Cosine Distance ANN)"]
        Reranker["Graph Hybrid Reranker\n(0.85 Sim + 0.15 PageRank Authority)"]
        LLMInference["LLM Generator & Citation Grounding\n(node-llama-cpp, 99 GPU Layers)"]
        AuthModule["Argon2id Auth & JWT Family Service\n(/api/v1/auth)"]
        SessionModule["Chat Sessions Service\n(/api/v1/chat-sessions)"]
        HealthModule["Health Telemetry (/api/v1/health)"]

        AuthModule <--> Mongo
        SessionModule <--> Mongo
        Retriever <--> Qdrant
        Reranker <--> MiningData
        LLMInference <--> LLMWeights
    end

    subgraph FRONTEND["Frontend Client (frontend_rag: React 19 + Vite :5174)"]
        WorldCanvas["3D Knowledge Tree Canvas\n(React Three Fiber / Three.js)"]
        ChatUI["Interactive Research Chat\n(Markdown + KaTeX Math)"]
        DexieDB[("Browser IndexedDB (Dexie.js)\nOffline-First Chat Cache")]
        SessionDrawer["History Drawer\n(+ New Chat, Multi-Session, Trash Delete)"]

        ChatUI <--> DexieDB
        SessionDrawer <--> DexieDB
    end

    FRONTEND <-->|"REST & SSE Stream (/api/v1)"| BACKEND
```

---

## 📦 Repository Organization

| Directory / Package                         | Description                                                                                                         | Technology Stack                                                                                           |
| :------------------------------------------ | :------------------------------------------------------------------------------------------------------------------ | :--------------------------------------------------------------------------------------------------------- |
| **[`backend_rag/`](backend_rag/)**   | Backend NestJS service providing vector retrieval, author reranking, local GPU LLM streaming, auth, and Swagger UI. | NestJS 11, `@qdrant/js-client-rest`, `node-llama-cpp`, `@huggingface/transformers`, Mongoose, Argon2 |
| **[`frontend_rag/`](frontend_rag/)** | Interactive web client featuring 3D Knowledge Tree, streaming chat, and offline-first IndexedDB persistence.        | React 19, Vite 6, `@react-three/fiber`, Three.js, Dexie.js, Tailwind CSS v4, Zustand                     |
| **[`docs/`](docs/)**                 | Architecture documentation, API specifications, and benchmark guides (`RUN_GUIDE.md`, `API_DESIGN.md`).         | Markdown, Mermaid                                                                                          |
| **[`models/`](model/)**              | Local quantized GGUF model weights for local LLM generation.                                                        | GGUF (Qwen2.5-7B Instruct Q4_K_M)                                                                          |
| **[`data/`](data/)**                 | Procedural 3D world exports (`tree.json`) and data mining artifacts (`gold/mining`).                            | JSON                                                                                                       |

---

## ⚡ Quick Start: Step-by-Step Startup Guide

Follow these steps to run the complete stack: **Cloudflare Tunnel $\rightarrow$ Local LLM $\rightarrow$ Backend (BE) $\rightarrow$ Frontend (FE)**.

### 1. Prerequisites

- **Node.js**: v22 LTS or higher
- **npm**: v10+
- **C++ Build Tools & CMake**: Required for building native bindings in `node-llama-cpp` (e.g. Visual Studio C++ Build Tools on Windows, Xcode on macOS)
- **GPU Acceleration**: NVIDIA GPU with CUDA Toolkit installed (or Apple Silicon Metal on macOS)
- **Cloudflare Tunnel (`cloudflared`)**: Required for bridging remote MongoDB TCP connection

---

### 2. Setup Local LLM Weights (LLMs)

Download or place the quantized Qwen2.5-7B GGUF weights in the `model/` directory:

```bash
# Model target path (relative to backend_rag):
# ../models/qwen2.5-7b-instruct-q4_k_m/qwen2.5-7b-instruct-q4_k_m.gguf
# Or ../model/qwen2.5-7b-instruct-q4_k_m.gguf
```

> **Note**: On startup, `node-llama-cpp` automatically probes hardware capabilities, initializes CUDA / Metal acceleration, offloads 99 GPU layers into VRAM, and prints:
> `GPU: CUDA (offload 99 layers) | VRAM: ~4.68 GiB allocated`.

---

### 3. Open MongoDB Tunnel (`cloudflared`)

The remote MongoDB server runs behind a Cloudflare Access tunnel. Start the TCP tunnel daemon in an open terminal and leave it running:

```bash
cloudflared access tcp --hostname mongodb.minio-quangmanh.io.vn --url localhost:27017
```

> If port `27017` is occupied by a local MongoDB instance, use `localhost:27018` and adjust the port in `backend_rag/.env`.

---

### 4. Setup & Start Backend (`backend_rag`)

1. Navigate to the backend directory and install dependencies:

   ```bash
   cd backend_rag
   npm install
   ```
2. Configure `backend_rag/.env`:

   ```dotenv
   PORT=3000
   NODE_ENV=development
   CORS_ORIGINS=http://localhost:5174

   # MongoDB (via cloudflared tunnel on localhost:27017)
   MONGO_URI=mongodb://app_user:YqihzvCaDSkqGQEoR8xWwVZ3@localhost:27017/rag_ts?authSource=admin&directConnection=true

   # Qdrant (read-only vector store via HTTPS)
   QDRANT_URL=https://qdrant.minio-quangmanh.io.vn
   QDRANT_API_KEY=uth_guest_readonly_key_4b71a8e9d2c
   QDRANT_COLLECTION=scientific_papers_gold

   # JWT Secrets (at least 32 characters each)
   JWT_ACCESS_SECRET=rag-ts-dev-access-secret-32-chars-minimum-token
   JWT_REFRESH_SECRET=rag-ts-dev-refresh-secret-32-chars-minimum-token

   # Embedding Model
   EMBEDDING_MODEL=nomic-ai/nomic-embed-text-v1.5

   # LLM Configuration
   MODEL_PATH=../models/qwen2.5-7b-instruct-q4_k_m/qwen2.5-7b-instruct-q4_k_m.gguf
   LLM_GPU=auto
   LLM_MAX_QUEUE=16
   LLM_CONCURRENCY=4
   LLM_TIMEOUT_MS=60000

   # Mining & World Data
   WORLD_DATA_DIR=../data/world
   MINING_DIR=../data/gold/mining
   ```
3. Build and launch the backend server:

   ```bash
   # Development with auto-reload:
   npm run start:dev

   # Or production compile and run:
   npm run build
   node dist/main.js
   ```
4. Verify backend health:

   ```bash
   curl http://localhost:3000/api/v1/health
   ```

   **Expected Response**:

   ```json
   {
     "status": "ok",
     "version": "1.0.0",
     "components": {
       "mongo": true,
       "qdrant": true,
       "llm": true
     },
     "llm": {
       "gpu": "cuda",
       "queue_length": 0
     }
   }
   ```

   * **Swagger Interactive UI**: [http://localhost:3000/api/v1/docs](http://localhost:3000/api/v1/docs)

---

### 5. Setup & Start Frontend (`frontend_rag`)

1. Open a new terminal, navigate to `frontend_rag`, and install dependencies:

   ```bash
   cd frontend_rag
   npm install
   ```
2. Start the development server connected to the live backend:

   ```bash
   # Connects to backend at http://localhost:3000 via proxy:
   npm run dev:live
   ```

   > **Windows PowerShell Tip**: If `npm run dev:live` fails due to POSIX environment variable syntax, run:
   >
   > ```powershell
   > $env:VITE_USE_MOCK="false"; npx vite
   > ```
   >
   > To run in **Mock mode** (standalone frontend with simulated 3D world and streaming responses, no backend required):
   >
   > ```bash
   > npm run dev:mock
   > ```
   >
3. Open **`http://localhost:5174`** in your browser.

---

## 💾 Offline-First Chat Persistence

The application includes an **Offline-First Chat Persistence Engine** built with **Dexie.js (IndexedDB)**:

- **Local Truth (Cache-First)**: Conversations and messages are saved directly into the browser's IndexedDB.
- **Backend Survivability**: If the backend service is offline, killed, or unreachable, previous conversations, cited academic papers, and latency timings remain **100% viewable and navigable**. An offline badge (`Ngoại tuyến`) notifies the user while keeping historical chats accessible.
- **Multi-Session History Drawer**:
  - Click the **History** icon (`History`) in the chat header to open the collapsible drawer.
  - Click **"+ Đoạn chat mới"** (`+ New Chat`) to start a fresh conversation.
  - Chat titles are automatically generated from the first prompt slice (`query.slice(0, 48)`).
  - Inline **Trash Bin (`Trash2`)** with 2-step confirmation (`Xóa?` / `Huỷ`) to safely delete conversations.
- **Cloud Sync**: When online and logged in, sessions and exchanges are synchronized with the backend MongoDB database.

---

## 🧪 Testing & Verification

### Backend Tests

```bash
cd backend_rag
npm test                   # Run unit & integration tests
npm run test:e2e           # Run end-to-end integration flows
```

### Frontend Tests

```bash
cd frontend_rag
npm test                   # Run all 210 Vitest unit test cases
npm run typecheck          # Verify TypeScript zero-error compliance
npm run build              # Compile production bundle
```

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).
