---
project: CampusGuide
document: Architecture
phase: 0
last_updated: 2026-07-05
---

# CampusGuide — Architecture

## 1. System Overview

CampusGuide has two pipelines that share the same underlying store:

1. **Ingestion Pipeline** — runs on every push to `campusguide-content` via a
   GitHub Action. Converts raw Markdown into embedded, searchable chunks.
2. **Query Pipeline** — runs per user question. Retrieves relevant chunks,
   generates a grounded answer, validates it, and returns it with citations.

```
 campusguide-content (GitHub repo)
          │  git push
          ▼
   GitHub Action (on: push)
          │
          ▼
   INGESTION PIPELINE  ──────►  Chroma (persistent vector store)
                                        ▲
                                        │  top-k retrieval
                                        │
   User question ──► QUERY PIPELINE ────┘──► grounded answer + citation
```

## 2. Ingestion Pipeline

**Trigger:** GitHub Action on push to `campusguide-content` (event-driven, not
scheduled — content only changes when pushed).

**Steps:**

1. **Load repo** — checkout/pull the latest state of `campusguide-content`.
   Because categories are not a fixed list, the loader walks the repo
   directory structure dynamically rather than iterating a hardcoded set of
   folder names. Any top-level folder containing `.md` files is treated as a
   category.
2. **Parse markdown + frontmatter** — for each `.md` file:
   - Extract YAML frontmatter fields: `category`, `title`, `slug`, `tags`,
     `last_updated`, `source`.
   - Extract the plain-Markdown body.
   - Validate required frontmatter fields exist; files missing required
     fields are logged and skipped (not silently ingested with gaps).
3. **Chunk by section** — split the body on `##` subheadings. Each chunk
   retains:
   - The parent file's frontmatter (category, title, slug, last_updated,
     source) attached as chunk metadata.
   - The subheading text itself (if any) so retrieval can match on
     sub-topics within a file.
   - Files with no `##` subheadings are treated as a single chunk (the whole
     body).
4. **Embed** — generate a vector embedding for each chunk using
   `bge-small-en-v1.5` (local, via `sentence-transformers`).
5. **Store in Chroma** — upsert chunks into the local persistent Chroma
   collection, keyed by a deterministic ID (e.g. `slug + section-index`) so
   re-running ingestion updates existing chunks rather than duplicating them.
   On file deletion in the repo, corresponding chunks are removed from Chroma
   (diff-based sync, not append-only).
6. **New category handling** — no allowlist or registration step is
   required. A newly added category folder is picked up automatically on the
   next push/re-index and its chunks become immediately queryable — the
   pipeline is category-agnostic by design (category is just metadata, not a
   routing key baked into the pipeline logic).

## 3. Query Pipeline

**Trigger:** incoming user question via the FastAPI backend.

**Steps:**

1. **Embed the question** — same `bge-small-en-v1.5` model, ensuring
   embedding-space consistency with the corpus.
2. **Retrieve top-k chunks** — via LlamaIndex's retriever interface over the
   Chroma vector store. `k` is tunable (starting point: k=5).
3. **Confidence check on retrieval** — before generation, evaluate the
   similarity scores of the top-k results against a minimum threshold:
   - **If the best match is weak/ambiguous** (below threshold, or top
     candidates are similarly low-scoring across unrelated categories), the
     pipeline **does not generate an answer**. It returns a clarification
     prompt asking the user to rephrase or be more specific, rather than
     answering off a poor match.
   - **If retrieval is strong**, proceed to generation.
4. **Generate a grounded answer** — pass only the retrieved chunks (with
   their metadata) plus the question to the LLM (Groq-hosted), with an
   explicit instruction to answer using only the provided context and to say
   so if the context is insufficient.
5. **Validate the generated answer** — a post-generation check that:
   - Confirms the answer doesn't introduce claims absent from the retrieved
     chunks (basic groundedness check).
   - Confirms at least one citation is attachable (i.e. generation didn't
     drift into a non-answer or an out-of-scope response without saying so).
   - If validation fails, fall back to the clarification-prompt response
     rather than returning an unverified answer.
6. **Return response** — final payload includes:
   - The generated answer text.
   - Citation(s): source file title/slug per chunk used.
   - `last_updated` date(s) from the cited file(s), so the user can judge
     freshness.

## 4. Scope Enforcement Boundary

Because freshers will ask questions that are *related* to campus life but not
an exact match to any single file (e.g. "what's there to do around campus on
weekends"), the query pipeline allows answers built from loosely-related
retrieved content, as long as it's still sourced from the corpus. It does
not fall back to the LLM's general/world knowledge — if nothing in the
corpus is relevant, the answer is a decline, not an invented response using
outside knowledge (see `edge-case.md` for exact behavior).

## 5. Component Summary

| Layer | Choice |
|---|---|
| Embeddings | `bge-small-en-v1.5` (local, sentence-transformers) |
| Vector store | Chroma (local persistent) |
| Retrieval framework | LlamaIndex |
| LLM (generation) | Groq-hosted (specific model confirmed at Phase 4 implementation time) |
| Backend | FastAPI |
| Frontend | Next.js |
| Content source | GitHub repo `campusguide-content` (private) |
| Re-index trigger | GitHub Action on push |
