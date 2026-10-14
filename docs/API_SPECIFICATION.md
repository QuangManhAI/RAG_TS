# API_SPECIFICATION.md — Scientific Literature RAG REST & OpenAPI Specification

- **Motivation/Background**: To allow frontend applications, evaluation harnesses, and external researcher tools to consume the Scientific RAG Backend consistently, a formalized REST and OpenAPI 3.0 specification is required.
- **Purpose**: Document all request schemas, query parameters, response structures, HTTP status codes, and Swagger UI routes for the backend service.
- **Overview Pipeline**: Documented from the NestJS 11 `RagController`, `QueryDto`, and OpenAPI decorators, aligned with interactive Swagger UI served at `/api/docs`.
- **Detailed Plan**: §1 Endpoint Inventory; §2 POST /api/rag/chat; §3 GET /api/rag/health; §4 OpenAPI & Swagger UI Access; §5 Error Handling; §6 References.
- **References**: `agents/rules/MD_CONVENTION.md`, `agents/rules/COMMIT_CONVENTION.md`, `docs/SCIENTIFIC_RAG_BACKEND.md`, `backend_rag/src/rag/rag.controller.ts`.
- **Created**: 2026-10-14T17:05:00+07:00
- **Last Updated**: 2026-10-14T17:05:00+07:00

---

[STATUS: COMPLETED]

## Table of Contents

- [1. Endpoint Inventory](#1-endpoint-inventory)
- [2. POST /api/rag/chat](#2-post-apiragchat)
- [3. GET /api/rag/health](#3-get-apiraghealth)
- [4. OpenAPI & Swagger UI Access](#4-openapi--swagger-ui-access)
- [5. Error Handling](#5-error-handling)
- [6. References](#6-references)

---

## 1. Endpoint Inventory

| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/rag/chat` | Primary RAG synthesis over scientific paper chunks | None |
| `GET` | `/api/rag/health` | Service, vector store, and GPU hardware status | None |
| `GET` | `/api/docs` | Interactive Swagger UI API documentation | None |
| `GET` | `/api/docs-json` | Raw OpenAPI 3.0 JSON specification | None |

---

## 2. POST /api/rag/chat

Executes the full RAG pipeline: embedding generation, LanceDB nearest neighbor search, graph data mining reranking, and local Qwen2.5-7B grounded response synthesis.

### Request Body (`application/json`)

| Field | Type | Required | Default | Validation Constraints | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `query` | `string` | **Yes** | — | Non-empty string | The user's scientific query or question. |
| `top_k` | `number` | No | `5` | Integer, $1 \le \text{top\_k} \le 20$ | Number of grounded chunks to include. |
| `category` | `string` | No | `undefined` | String | Optional category filter (e.g. `cs.CV`, `cs.LG`, `cs.CL`). |
| `use_authority_boost` | `boolean` | No | `true` | Boolean | Whether to apply PageRank author authority boost ($0.15$). |
| `expand_query` | `boolean` | No | `false` | Boolean | Whether to apply association-rules query expansion. |

#### Example Request:
```json
{
  "query": "How does LoRA optimize large models?",
  "top_k": 3,
  "category": "cs.LG",
  "use_authority_boost": true,
  "expand_query": false
}
```

### Response Body (`application/json` - HTTP 200 OK)

| Field | Type | Description |
| :--- | :--- | :--- |
| `query` | `string` | The original user query. |
| `answer` | `string` | The synthesized grounded answer with `[Paper: {id}]` citations. |
| `citations` | `string[]` | Array of unique paper IDs cited in the answer. |
| `grounded_chunks` | `GroundedChunkResult[]` | Array of chunks retrieved, scored, and passed to LLM context. |
| `latency_ms` | `number` | End-to-end execution time in milliseconds. |

#### Example Response:
```json
{
  "query": "How does LoRA optimize large models?",
  "answer": "LoRA (Low-Rank Adaptation) optimizes large models by freezing pre-trained weights and injecting trainable low-rank decomposition matrices... `[Paper: 2106.09685]`",
  "citations": [
    "2106.09685"
  ],
  "grounded_chunks": [
    {
      "chunk_id": "2106.09685_sec8",
      "paper_id": "2106.09685",
      "title": "LoRA Fine-Tuning of a 3B Code LLM for Algorithmic Efficiency",
      "authors": ["J. Edward Hu", "Yelong Shen", "Phillip Wallis"],
      "section_title": "8 Conclusion and Future Work",
      "text": "Fine-tuning enormous language models is prohibitively expensive...",
      "similarity_score": 0.5278,
      "authority_score": 0.3059,
      "final_score": 0.4945,
      "citation_tag": "[Paper: 2106.09685]"
    }
  ],
  "latency_ms": 14185
}
```

---

## 3. GET /api/rag/health

Reports the live readiness of all system components including LanceDB, local LLM, and GPU hardware acceleration.

### Response Body (`application/json` - HTTP 200 OK)

```json
{
  "status": "ok",
  "service": "scientific-rag-backend",
  "timestamp": "2026-10-14T09:55:47.807Z",
  "components": {
    "vectorStore": "ready",
    "llm": "ready",
    "gpu": {
      "hasGpu": true,
      "type": "cuda",
      "platform": "win32",
      "arch": "x64",
      "details": "NVIDIA CUDA (NVIDIA GeForce RTX 4060 Laptop GPU, 581.80, 8188 MiB)",
      "nodeLlamaGpuOption": "cuda"
    }
  }
}
```

---

## 4. OpenAPI & Swagger UI Access

When the backend server is running, the interactive documentation is accessible in any browser:
- **Interactive UI**: `http://localhost:8000/api/docs`
- **OpenAPI 3.0 Schema**: `http://localhost:8000/api/docs-json`

Features supported in the Swagger UI:
- Interactive "Try it out" execution for `POST /api/rag/chat` and `GET /api/rag/health`.
- Explicit schema definitions for all DTOs and sub-objects.
- Pre-filled sample payloads.

---

## 5. Error Handling

Standard HTTP status codes are returned:

| HTTP Status | Reason | Trigger |
| :--- | :--- | :--- |
| `400 Bad Request` | Validation Error | Missing `query`, invalid `top_k` (e.g. $>20$), unrecognized parameters. |
| `500 Internal Server Error` | Pipeline Error | Vector search or LLM engine failure. |
| `503 Service Unavailable` | Initialization | LLM model or vector store not yet loaded. |

---

## 6. References

- [`backend_rag/src/rag/rag.controller.ts`](backend_rag/src/rag/rag.controller.ts)
- [`backend_rag/src/rag/dto/query.dto.ts`](backend_rag/src/rag/dto/query.dto.ts)
- [`docs/SCIENTIFIC_RAG_BACKEND.md`](docs/SCIENTIFIC_RAG_BACKEND.md)
