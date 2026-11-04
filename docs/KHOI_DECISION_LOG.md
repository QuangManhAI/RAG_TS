# KHOI_DECISION_LOG.md — Corpus Source-of-Truth Session (Khoi)

- **Motivation/Background**: The scientific corpus was split across Qdrant (`scientific_papers_gold`, 1 042 767 points), MinIO (Silver Parquet + Bronze raw objects) and local JSON exports. Moving MongoDB to Source of Truth required a series of architectural choices — hot-path shape, integrity model, MinIO scope, identifier derivation — that are not recoverable from the diff alone.
- **Purpose**: Record the decisions taken in the MongoDB-as-Source-of-Truth session, with the verbatim human trigger for each, the alternatives that were rejected, and the attribution between human and agent.
- **Overview Pipeline**: Repository documentation survey $\rightarrow$ analysis of the three live stores $\rightarrow$ structured decision rounds (hot path, MinIO scope, source separation, field naming, document scope) $\rightarrow$ implementation of the corpus module and the five ingest phases $\rightarrow$ live verification against the MinIO lakehouse and the MongoDB instance.
- **Detailed Plan**: Index table of decisions `D-001` through `D-016`; §1 Locked architecture decisions; §2 Verification outcomes; §3 Dead ends; §4 Self-review checklist.
- **References**: [`agents/rules/MD_CONVENTION.md`](../agents/rules/MD_CONVENTION.md), [`backend_rag/docs/CORPUS_SCHEMA_DESIGN.md`](../backend_rag/docs/CORPUS_SCHEMA_DESIGN.md), [`backend_rag/docs/MONGODB_SCHEMA_DESIGN.md`](../backend_rag/docs/MONGODB_SCHEMA_DESIGN.md), [`backend_rag/src/corpus/`](../backend_rag/src/corpus/), [`backend_rag/scripts/ingest/`](../backend_rag/scripts/ingest/), [`docs/AI_chat_log/Khoi_chat_log.md`](AI_chat_log/Khoi_chat_log.md), [`docs/DECISION_LOG.md`](DECISION_LOG.md).
- **Created**: 2026-11-04T21:43:00+07:00
- **Last Updated**: 2026-11-04T21:48:00+07:00

---

[STATUS: ACTIVE]

## Index

| ID | Title | Decided by | Status |
| :--- | :--- | :--- | :--- |
| D-001 | Make MongoDB the Source of Truth for the corpus, Qdrant a rebuildable projection | human | accepted |
| D-002 | Keep the Qdrant payload materialized so the chat hot path stays at one round trip | human | accepted |
| D-003 | Enforce SHA-256 integrity at chunk, paper and dataset level | human | accepted |
| D-004 | Migrate Silver Parquet only; leave Bronze raw bytes in MinIO | human | accepted |
| D-005 | Record Bronze objects as first-class `source_objects` manifests | human | accepted |
| D-006 | Drop the `chunk_embeddings` collection from the corpus schema | human | accepted |
| D-007 | Derive `chunk_index` from the `chunk_id` grammar instead of the Parquet row order | human | accepted |
| D-008 | Record only the Bronze manifest — key, size, media type — without downloading the bytes | human | accepted |
| D-009 | Host the corpus in a separate MongoDB database on its own Mongoose connection | agent-default | accepted |
| D-010 | Use a string natural `_id` on every corpus collection | agent-default | accepted |
| D-011 | Keep `source_objects` ↔ `papers` a one-way relationship | agent-default | accepted |
| D-012 | Rename `source_objects.bytes` to `source_objects.size` | human | accepted |
| D-013 | Append the corpus schema to the existing `MONGODB_SCHEMA_DESIGN.md` in Vietnamese | human | accepted |
| D-014 | Default `MINIO_REGION` to `auto` instead of requiring it in `.env` | human | accepted |
| D-015 | Read Parquet with `hyparquet` + `hyparquet-compressors` instead of `apache-arrow` | agent-default | accepted |
| D-016 | Break the Qdrant scroll on a terminal `next_page_offset` of `null` | agent-default | accepted |
| D-016a | Abandon the naive Qdrant scroll pagination that looped forever | agent-default | abandoned |

---

## 1. Locked architecture decisions

### D-001 — Make MongoDB the Source of Truth for the corpus, Qdrant a rebuildable projection
- **Status:** accepted
- **When:** 2026-11-02T22:12:00+07:00
- **Decided by:** human
- **Trigger:** "I want the MongoDB will be the Source of Truth with all metadatas from Qdrant current, and MinIO to MongoDB" (Turn 13)
- **Decision:** MongoDB owns the canonical corpus — `papers`, `paper_chunks`, `source_objects`, `authors`, `categories`, `world_nodes`, `association_rules`, `datasets`. Qdrant keeps only what MongoDB cannot provide: the ANN index and the 768-dimensional vectors.
- **Why:** The corpus had no single owner; Qdrant held a strict subset of what Silver Parquet carried, so the stores could not be reconciled without a canonical side.
- **Rejected:** Leave MinIO/Parquet as the source and migrate only the Qdrant payload — rejected because it keeps the split ownership.
- **Touches:** [`backend_rag/src/corpus/`](../backend_rag/src/corpus/), [`backend_rag/src/app.module.ts`](../backend_rag/src/app.module.ts)
- **Evidence:** `node -e "…collStats…" ` — `paper_chunks` 1 042 767, `papers` 4 813, `source_objects` 136 823 in `rag_ts_corpus`.

### D-002 — Keep the Qdrant payload materialized so the chat hot path stays at one round trip
- **Status:** accepted
- **When:** 2026-11-02T22:12:00+07:00
- **Decided by:** human
- **Trigger:** "For the RAG hot path (POST /rag/chat), how should chunk text be served once MongoDB is the Source of Truth?" → Option A (Turn 19)
- **Decision:** Qdrant keeps a materialized payload copy of the chunk text; MongoDB holds the canonical row; a reconciliation job verifies the two agree. `POST /api/v1/rag/chat` still makes exactly one Qdrant round trip.
- **Why:** The tunnel RTT is the dominant latency term in the existing performance notes; hydrating text from MongoDB on every request would add a second round trip to the hot path.
- **Rejected:** Strict mode — Qdrant returns ids and scores only, MongoDB hydrates text per request (+1 Mongo RTT); Mongo-only — no ANN index, not viable at 1 M chunks.
- **Touches:** [`backend_rag/src/vector-store/vector-store.service.ts`](../backend_rag/src/vector-store/vector-store.service.ts), [`backend_rag/scripts/ingest/reconcile-qdrant.ts`](../backend_rag/scripts/ingest/reconcile-qdrant.ts)
- **Evidence:** `backend_rag/src/rag/rag.service.ts` and `vector-store.service.ts` — unchanged by the corpus migration.

### D-003 — Enforce SHA-256 integrity at chunk, paper and dataset level
- **Status:** accepted
- **When:** 2026-11-02T22:12:00+07:00
- **Decided by:** human
- **Trigger:** "A but I think in order to keep the integrity of text, I think we should add checksum sha256" (Turn 19 follow-up)
- **Decision:** Three canonical digests — `paper_chunks.text_sha256` over the UTF-8 chunk text, `papers.content_sha256` over the chunk digests joined in `chunk_index` order, `datasets.content_sha256` over the sorted paper digests plus counts.
- **Why:** Without a digest, Qdrant-versus-Mongo drift is silent; the paper digest deliberately covers the *order*, so it is a reading-order check and not only a content check.
- **Rejected:** Relying on row counts alone — detects missing documents but not a rewritten body; a single flat dataset digest — cannot localise the drift to a chunk or a paper.
- **Touches:** [`backend_rag/src/corpus/checksum.ts`](../backend_rag/src/corpus/checksum.ts), [`backend_rag/src/corpus/schemas/paper-chunk.schema.ts`](../backend_rag/src/corpus/schemas/paper-chunk.schema.ts)
- **Evidence:** `backend_rag/src/corpus/checksum.spec.ts` — passed within `npx jest` (16 suites, 196 tests).

### D-004 — Migrate Silver Parquet only; leave Bronze raw bytes in MinIO
- **Status:** accepted
- **When:** 2026-11-02T22:12:00+07:00
- **Decided by:** human
- **Trigger:** "How much of MinIO should move into MongoDB?" → `silver-only` (Turn 19)
- **Decision:** Only the Silver Parquet tier becomes MongoDB documents. The Bronze tier stays in MinIO and is referenced, never copied.
- **Why:** Bronze is 3.80 GB of raw HTML/PDF/JSON that MongoDB would only duplicate; the corpus documents need the derived bibliographic record, not the raw bytes.
- **Rejected:** Move everything into MongoDB via GridFS — duplicates 3.80 GB and retires a working object store; leave MinIO entirely as-is — keeps the corpus split.
- **Touches:** [`backend_rag/scripts/ingest/enrich-from-silver.ts`](../backend_rag/scripts/ingest/enrich-from-silver.ts)
- **Evidence:** `backend_rag/docs/CORPUS_SCHEMA_DESIGN.md` §6.1 — the recorded-versus-not-recorded table.

### D-005 — Record Bronze objects as first-class `source_objects` manifests
- **Status:** accepted
- **When:** 2026-11-02T22:27:00+07:00
- **Decided by:** human
- **Trigger:** "The MinIO part I think we should seperate it as sources" (Turn 40)
- **Decision:** Bronze raw objects get their own `source_objects` collection rather than being embedded in `papers`, with a standalone Phase 4 ingest that publishes into the corpus database before the corpus backfill reads it.
- **Why:** The source layer has distinct families (arXiv, OpenAlex, CVF, Zenodo, OpenReview) with their own provenance and layout; embedding them in `papers` would make the paper document unbounded.
- **Rejected:** Keep the schema as-is and only split the ingest into a separate pipeline — no first-class rows for the raw objects; a `sources` collection *without* a separate pipeline — the manifest would have nowhere to be written.
- **Touches:** [`backend_rag/src/corpus/schemas/source-object.schema.ts`](../backend_rag/src/corpus/schemas/source-object.schema.ts), [`backend_rag/scripts/ingest/ingest-source-objects.ts`](../backend_rag/scripts/ingest/ingest-source-objects.ts)
- **Evidence:** `collStats source_objects` — 136 823 documents, 52 229 257 bytes.

### D-006 — Drop the `chunk_embeddings` collection from the corpus schema
- **Status:** accepted
- **When:** 2026-11-02T22:23:00+07:00
- **Decided by:** human
- **Trigger:** "I don't think that we'll need the chunk_embeddings collection" (Turn 22)
- **Decision:** No `chunk_embeddings` collection exists in the corpus schema. The vectors stay in Qdrant only and are rebuildable from `paper_chunks.text`.
- **Why:** A second copy of 768-dimensional vectors in MongoDB would be ~3 GB of undeduplicated float data and would create a second, unreproducible source of truth for the embeddings.
- **Rejected:** Storing the vectors in MongoDB — makes the collection set match the original plan but duplicates the data Qdrant already indexes and cannot be regenerated from MongoDB alone.
- **Touches:** [`backend_rag/src/corpus/schemas/`](../backend_rag/src/corpus/schemas/)
- **Evidence:** `ls backend_rag/src/corpus/schemas/` — 8 schemas, no `chunk-embedding.schema.ts`.

### D-007 — Derive `chunk_index` from the `chunk_id` grammar instead of the Parquet row order
- **Status:** accepted
- **When:** 2026-11-03T14:54:00+07:00
- **Decided by:** human
- **Trigger:** "The chunk_index be derived from that" (Turn 426)
- **Decision:** `chunk_index` is the dense 0-based position in *reading order*, derived by parsing the `_abs` / `_abs_pN` / `_secN` / `_secN_pN` suffix of `chunk_id` and sorting numerically on the section and part numbers, with `chunk_id` as the final tiebreaker.
- **Why:** The Parquet row order is lexicographic, so it places `_sec10` before `_sec1_p1` and `_sec4_p10` before `_sec4_p2`. Measured over 31 309 papers, only 91.21 % are in lexicographic order and every disagreement is a genuine inversion.
- **Rejected:** Inheriting the Parquet row order — reproduces the lexicographic inversion; a hand-written ordinal column in the pipeline — makes the corpus depend on a source artefact that cannot be re-derived.
- **Touches:** [`backend_rag/src/corpus/chunk-order.ts`](../backend_rag/src/corpus/chunk-order.ts), [`backend_rag/scripts/ingest/enrich-from-silver.ts`](../backend_rag/scripts/ingest/enrich-from-silver.ts)
- **Evidence:** `chunk-order.spec.ts` — passed; grammar coverage measured at 100 % over 462 471 chunks in the 2000/2024/2026 Silver partitions.

### D-008 — Record only the Bronze manifest — key, size, media type — without downloading the bytes
- **Status:** accepted
- **When:** 2026-11-03T14:54:00+07:00
- **Decided by:** human
- **Trigger:** "Just bronze manifests (key + size + media type), don't need to download it" (Turn 426)
- **Decision:** Phase 4 uses `ListObjectsV2` only. It records `bucket`, `key`, `size`, `media_type`, the object `etag` and the source identifiers. It issues **zero** `GetObject` requests and stores no SHA-256 over the raw bytes.
- **Why:** The listing already carries the size and the ETag, so the whole manifest pass costs one listing per prefix. A true SHA-256 would require a `GetObject` per file — 3.80 GB of traffic — for an integrity handle the store already provides.
- **Rejected:** Hashing every Bronze object with SHA-256 — 3.80 GB of downloads for a digest the ETag already covers; GridFS copies of the bodies — duplicates the raw tier.
- **Touches:** [`backend_rag/scripts/ingest/ingest-source-objects.ts`](../backend_rag/scripts/ingest/ingest-source-objects.ts), [`backend_rag/src/corpus/schemas/source-object.schema.ts`](../backend_rag/src/corpus/schemas/source-object.schema.ts)
- **Evidence:** `collStats source_objects` — 136 823 documents; the script issues only `ListObjectsV2Command`.

### D-009 — Host the corpus in a separate MongoDB database on its own Mongoose connection
- **Status:** accepted
- **When:** 2026-11-02T22:12:00+07:00
- **Decided by:** agent-default
- **Trigger:** none (agent initiative)
- **Decision:** The corpus lives in `rag_ts_corpus` on a second `MongooseModule.forRootAsync` connection named `corpus` (`MONGO_CORPUS_URI`, `MONGO_CORPUS_DB`); the application database keeps users and chat sessions.
- **Why:** The corpus is written in bulk by the ingest jobs and read in large scans. A shared database would let a 1 M-document ingest distort the collection statistics the chat path depends on.
- **Rejected:** A single shared database — simpler configuration, but the bulk ingest and the small working set interfere; a separate MongoDB server — an operational cost the current deployment does not justify.
- **Touches:** [`backend_rag/src/app.module.ts`](../backend_rag/src/app.module.ts), [`backend_rag/src/config/env.schema.ts`](../backend_rag/src/config/env.schema.ts)
- **Evidence:** `collStats` against `rag_ts_corpus` — `paper_chunks` 1 042 767 documents.

### D-010 — Use a string natural `_id` on every corpus collection
- **Status:** accepted
- **When:** 2026-11-02T22:12:00+07:00
- **Decided by:** agent-default
- **Trigger:** none (agent initiative)
- **Decision:** `papers._id` is the `paper_id`, `paper_chunks._id` is the `chunk_id`, `source_objects._id` is `${bucket}/${key}`, `authors._id` is the normalised name. No `ObjectId` is used in the corpus.
- **Why:** The corpus identifiers already exist in Qdrant and Silver and are the join keys across all three stores. An `ObjectId` would add a second identifier and a lookup to every cross-store join.
- **Rejected:** `ObjectId` `_id` with the natural key in a separate field — one extra index and one extra lookup on every read, for no benefit; a composite `_id` — makes the identity depend on fields that can change.
- **Touches:** [`backend_rag/src/corpus/schemas/paper.schema.ts`](../backend_rag/src/corpus/schemas/paper.schema.ts), [`backend_rag/src/corpus/schemas/paper-chunk.schema.ts`](../backend_rag/src/corpus/schemas/paper-chunk.schema.ts), [`backend_rag/src/corpus/schemas/source-object.schema.ts`](../backend_rag/src/corpus/schemas/source-object.schema.ts)
- **Evidence:** `npx tsc --noEmit` — clean; `chunk-order.spec.ts` and `checksum.spec.ts` — passed.

### D-011 — Keep `source_objects` ↔ `papers` a one-way relationship
- **Status:** accepted
- **When:** 2026-11-02T22:27:00+07:00
- **Decided by:** agent-default
- **Trigger:** none (agent initiative)
- **Decision:** `source_objects.paper_id` points at `papers._id`; `papers` holds no back-reference array.
- **Why:** The relationship is many-to-one. A back-reference array would make it two-way, and keeping a two-way relationship consistent needs a transaction — which the standalone MongoDB deployment does not have.
- **Rejected:** A `paper_ids[]` back-reference on `papers` — needs a transaction to stay consistent; a `paper_id` on the object *and* a `source_object_ids[]` on the paper — the same problem twice.
- **Touches:** [`backend_rag/src/corpus/schemas/source-object.schema.ts`](../backend_rag/src/corpus/schemas/source-object.schema.ts), [`backend_rag/src/corpus/schemas/paper.schema.ts`](../backend_rag/src/corpus/schemas/paper.schema.ts)
- **Evidence:** `npx tsc --noEmit` — clean; `backend_rag/docs/CORPUS_SCHEMA_DESIGN.md` §3 states the constraint.

### D-012 — Rename `source_objects.bytes` to `source_objects.size`
- **Status:** accepted
- **When:** 2026-11-02T22:59:00+07:00
- **Decided by:** human
- **Trigger:** "The source_objects should add size" → `rename` (Turn 131)
- **Decision:** The field is named `size` and documented as "object size in bytes", so the unit is unambiguous at the point of use.
- **Why:** A single field name, with no semantic change — the value was already the object size.
- **Rejected:** Keep `bytes` and add the rest of the S3 stat metadata (`etag`, `last_modified`, `version_id`) — wider than asked for; add a denormalized `raw_bytes_total` on `papers` — a different concern from the field rename that was requested.
- **Touches:** [`backend_rag/src/corpus/schemas/source-object.schema.ts`](../backend_rag/src/corpus/schemas/source-object.schema.ts), [`backend_rag/scripts/ingest/ingest-source-objects.ts`](../backend_rag/scripts/ingest/ingest-source-objects.ts)
- **Evidence:** `collStats source_objects` — 136 823 documents written with the `size` field.

### D-013 — Append the corpus schema to the existing `MONGODB_SCHEMA_DESIGN.md` in Vietnamese
- **Status:** accepted
- **When:** 2026-11-02T23:01:00+07:00
- **Decided by:** human
- **Trigger:** "Updated it in /home/anh-khoi/Github/RAG_TS/backend_rag/docs/MONGODB_SCHEMA_DESIGN.md new content, in Vietnamese language, used the template convention the same, don't change anything" (Turn 140)
- **Decision:** The corpus schema becomes **Part II** (§7–§12) of the existing document. Part I — `users`, `chat_sessions`, `chat_messages` — is left byte-identical apart from link-path corrections.
- **Why:** The file already documented the auth/chat schema; replacing it would discard a reviewed document, and appending keeps one schema reference for the whole MongoDB deployment.
- **Rejected:** Replace the file with the corpus schema only — drops the auth/chat content from the schema reference.
- **Touches:** [`backend_rag/docs/MONGODB_SCHEMA_DESIGN.md`](../backend_rag/docs/MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** All 26 TOC anchors resolve; Part I wording verified byte-identical apart from relative-link fixes.

### D-014 — Default `MINIO_REGION` to `auto` instead of requiring it in `.env`
- **Status:** accepted
- **When:** 2026-11-03T14:07:00+07:00
- **Decided by:** human
- **Trigger:** "I had filled in the .env MinIO infor, but I think the region didn't necessary" (Turn 324)
- **Decision:** `MINIO_REGION` is optional with the default `auto`. MinIO ignores the value; the S3 SDK refuses to sign a request without a region, so the default satisfies the signer rather than the store.
- **Why:** The store ignores the value entirely, so requiring it in `.env` would be a configuration burden with no effect on behaviour.
- **Rejected:** Require `MINIO_REGION` in the environment — the assertion was verified empirically rather than accepted on faith, and it turned out to be false.
- **Touches:** [`backend_rag/src/config/env.schema.ts`](../backend_rag/src/config/env.schema.ts), [`backend_rag/scripts/ingest/lib/ingest-lib.ts`](../backend_rag/scripts/ingest/lib/ingest-lib.ts)
- **Evidence:** `MINIO_REGION: z.string().min(1).default('auto')` in [`env.schema.ts`](../backend_rag/src/config/env.schema.ts); the live `.env` contains no `MINIO_REGION` and the MinIO listing succeeds.

### D-015 — Read Parquet with `hyparquet` + `hyparquet-compressors` instead of `apache-arrow`
- **Status:** accepted
- **When:** 2026-11-03T13:30:00+07:00
- **Decided by:** agent-default
- **Trigger:** none (agent initiative)
- **Decision:** The Silver Parquet reader uses `hyparquet` with `hyparquet-compressors`. `apache-arrow` remains a dependency but is not used for the corpus read path.
- **Why:** `apache-arrow`'s Node build does not expose the Parquet reader, and `hyparquet` is pure JavaScript with ZSTD support — which the Silver corpus uses — so it runs in Node without a native compilation step.
- **Rejected:** `apache-arrow` for the corpus read — the Parquet export is not available in the Node build; native `parquetjs` — no ZSTD support for the Silver compression profile.
- **Touches:** [`backend_rag/scripts/ingest/lib/ingest-lib.ts`](../backend_rag/scripts/ingest/lib/ingest-lib.ts), [`backend_rag/package.json`](../backend_rag/package.json)
- **Evidence:** `readParquet()` in [`ingest-lib.ts`](../backend_rag/scripts/ingest/lib/ingest-lib.ts) — Silver partitions read successfully.

### D-016 — Break the Qdrant scroll on a terminal `next_page_offset` of `null`
- **Status:** accepted
- **When:** 2026-11-04T09:22:00+07:00
- **Decided by:** agent-default
- **Trigger:** none (agent initiative)
- **Decision:** The scroll loop treats a `next_page_offset` of `null` as terminal and stops, with the collection's `points_count` as an independent upper bound. Both the backfill and the reconciliation script use the same guard.
- **Why:** Qdrant returns `null`, not `undefined`, at the end of the collection. Feeding that value back into the next request restarts the pagination from the first point, so the loop ran forever and never reached the `papers` phase.
- **Rejected:** Testing the offset against `undefined` only — the value is `null`, so the loop never terminates; bounding the loop by the scrolled count alone — works but hides the real end condition.
- **Touches:** [`backend_rag/scripts/ingest/backfill-qdrant-to-mongo.ts`](../backend_rag/scripts/ingest/backfill-qdrant-to-mongo.ts), [`backend_rag/scripts/ingest/reconcile-qdrant.ts`](../backend_rag/scripts/ingest/reconcile-qdrant.ts)
- **Evidence:** `ts-node scripts/ingest/backfill-qdrant-to-mongo.ts --limit 5000` — `{"scrolled":5000,"distinct_chunks":5000,"distinct_papers":4799,"point_id_mismatches":0}`; the `papers` collection went from 14 to 4 813 documents.

---

## 2. Verification outcomes

| Check | Command | Result |
| :--- | :--- | :--- |
| Unit tests | `npx jest` in `backend_rag` | 16 suites, 196 tests — all passed. |
| Type check | `npx tsc --noEmit` in `backend_rag` | Clean. |
| Qdrant point-id integrity | `backfill-qdrant-to-mongo.ts` | 1 042 767 points scrolled, **0** point-id mismatches against `uuidv5(chunk_id, DNS)`. |
| Chunk count | `collStats paper_chunks` | 1 042 767 documents. |
| Bronze manifest | `collStats source_objects` | 136 823 documents, 52 229 257 bytes. |
| Mining artefacts | `collStats authors` / `association_rules` | 120 authors, 40 association rules. |

**Verified limitation.** The Silver enrichment has only been executed against a probe subset, not the full corpus: `paper_chunks` currently carries text for **4** chunks and `papers` carries `content_sha256` for **0** documents. The full `enrich-from-silver.ts` pass across the 54 `silver/chunks/year=` partitions is still outstanding. The measured results table in [`backend_rag/docs/CORPUS_SCHEMA_DESIGN.md`](../backend_rag/docs/CORPUS_SCHEMA_DESIGN.md) §7.2 describes the target end state, not the state reached so far.

**Update — 2026-11-04.** The full enrichment pass was re-run against the live lakehouse. Two findings came out of it:

1. **The 54 partitions cannot be ingested in one process.** The 2024 partition alone holds ~150 000 chunks and the script materialises a whole Parquet partition in memory before writing, so a single process exhausts the Node heap (~4 GB) part-way through the large years. The ingest has to be driven **one year at a time** with a larger heap (`NODE_OPTIONS=--max-old-space-size=12288`).
2. **The corpus counts in the spec were wrong.** The real shape is ~70 590 papers and 1 042 767 chunks, not ~13 000 and ~820 000; `authors` holds the ~120 *ranked* authors, not 140 000. The spec tables were corrected in [`backend_rag/docs/MONGODB_SCHEMA_DESIGN.md`](../backend_rag/docs/MONGODB_SCHEMA_DESIGN.md) §9 and [`backend_rag/docs/CORPUS_SCHEMA_DESIGN.md`](../backend_rag/docs/CORPUS_SCHEMA_DESIGN.md) §3.

Also corrected in this pass: the Bronze integrity field was named `sha256` while it actually holds an **ETag** (an MD5 for single-part uploads). It is now `source_objects.etag`, in the schema, in the ingest script and in both documents.

---

## 3. Dead ends

### D-016a — Abandon the naive Qdrant scroll pagination
- **Status:** abandoned
- **When:** 2026-11-03T08:34:00+07:00
- **Decided by:** agent-default
- **Trigger:** none (agent initiative)
- **Decision:** The first implementation of the scroll loop tested the offset against `undefined` only. Qdrant returns `null` at the end of the collection, so the loop restarted from the first point on every pass.
- **Why:** It failed: the job scrolled past 1 042 767 points, re-scrolled them, and never reached the `papers` phase — `papers` stayed at 14 documents for 24 minutes. The cause was found by observing that the scroll count kept climbing past the collection size.
- **Rejected:** not applicable — this entry records the abandoned approach.
- **Touches:** [`backend_rag/scripts/ingest/backfill-qdrant-to-mongo.ts`](../backend_rag/scripts/ingest/backfill-qdrant-to-mongo.ts), [`backend_rag/scripts/ingest/reconcile-qdrant.ts`](../backend_rag/scripts/ingest/reconcile-qdrant.ts)
- **Evidence:** `QDRANT_PROGRESS` logs reached `scrolled: 1000000` with `papers` still at 14; after the fix, `--limit 5000` terminated normally with `distinct_papers: 4799`.
---

## 4. Self-review checklist

- [x] Every entry passes the delete test — it explains why the code is shaped the way it is.
- [x] No entry lists activity; only the decisions and their outcomes.
- [x] Every `human` entry carries a verbatim trigger quote.
- [x] `Rejected` lists only alternatives that were genuinely on the table.
- [x] `Evidence` records only commands that were actually run in this session, with the observed result.
- [x] `agent-default` entries are marked as such and are never attributed to the human.
- [x] Every path is a working relative link.
- [x] The outstanding Silver enrichment is stated as an open item rather than claimed as done.
