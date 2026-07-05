---
project: CampusGuide
document: Implementation Plan
phase: 0
last_updated: 2026-07-05
derived_from: problemStatement.md, architecture.md, edge-case.md
---

# CampusGuide — Implementation Plan

This plan is phased like a typical RAG build (docs → ingestion → embedding →
retrieval → generation → API → UI → automation → deployment), but each phase
is scoped to what CampusGuide actually needs — notably, **no HTML
scraping/cleaning phase**, since content already arrives as authored Markdown
in `campusguide-content`, and **no advisory-query classifier**, since
CampusGuide's refusal logic is scope/confidence-based (see `edge-case.md`),
not compliance-based.

## Phase 0 — Docs (Context Setup) ✅

- [x] `problemStatement.md` — scope, users, hard constraints
- [x] `architecture.md` — ingestion + query pipeline design
- [x] `edge-case.md` — weak-match, missing-content, out-of-scope, new-category
      behavior
- [x] `implementation-plan.md` — this document

## Phase 1 — Ingestion: Repo Parsing

No fetch/scrape/clean step is needed — content is already clean Markdown.
This phase is purely structural parsing.

- [ ] `ingestion/load_repo.py` — clone/pull `campusguide-content`; walk all
      top-level folders dynamically (no hardcoded category list)
- [ ] `ingestion/parse.py` — for each `.md` file:
  - Parse YAML frontmatter (`category`, `title`, `slug`, `tags`,
    `last_updated`, `source`)
  - Validate required fields present; log + skip files that fail validation
  - Extract plain-Markdown body
- [ ] `ingestion/section_split.py` — split body on `##` subheadings; files
      with no subheadings become a single chunk
- [ ] Attach frontmatter as metadata to every resulting chunk (`category`,
      `title`, `slug`, `last_updated`, `source`, `section_heading`)

## Phase 2 — Embedding & Vector Store

- [ ] `ingestion/embed.py` — embed each chunk with `bge-small-en-v1.5`
      (local, `sentence-transformers`)
- [ ] `ingestion/index.py` — upsert into Chroma (local persistent collection)
  - Deterministic chunk ID: `{slug}__{section-index}`
  - On file deletion from repo: diff against previous manifest and remove
    corresponding chunks from Chroma (no orphaned vectors)
- [ ] Maintain a lightweight ingestion manifest (JSON) recording per-file
      `slug`, `last_updated`, and chunk IDs — used for the delta sync above

## Phase 3 — Retrieval Strategy

- [ ] `app/retriever.py` — wrap Chroma as a LlamaIndex vector store; expose a
      `top_k` retriever (start k=5)
- [ ] Category is **metadata for filtering/citation, not a routing gate** —
      do not require the caller to specify category up front, since new
      categories must work automatically (per `edge-case.md` §5)
- [ ] Confidence thresholding: compute a minimum similarity-score cutoff;
      if the top result(s) fall below it (or scores are flat/ambiguous across
      unrelated categories), mark retrieval as **low-confidence** — this
      routes to the clarification-prompt path instead of generation
      (per `edge-case.md` §1)
- [ ] Tune the threshold against a small hand-written eval set of
      known-answerable vs. known-unanswerable questions before relying on it

## Phase 4 — Generation (Groq)

- [ ] `app/generator.py` — call Groq's chat completion API with:
  - System prompt: answer only from provided context; if context is
    insufficient, say so explicitly; do not use outside/general knowledge
  - Retrieved chunks (text + metadata) + user question as context
  - Pick a fast Groq-hosted model suited to short factual synthesis (e.g. a
    Llama 3.x instruct variant available on Groq) — confirm current model
    availability in Groq's console when you get here, since hosted model
    names change over time
- [ ] Keep generation stateless per request — no conversation memory needed
      for a single-turn factual Q&A assistant (revisit only if multi-turn
      becomes a requirement)

## Phase 5 — Validation & Response Formatting

- [ ] `app/validator.py` — post-generation groundedness check:
  - Confirm the answer doesn't introduce claims absent from retrieved chunks
  - If generation drifted into an unsupported claim or a non-answer,
    fall back to the clarification-prompt response rather than returning it
- [ ] `app/formatter.py` — build the final response contract:
  ```json
  {
    "answer": "string",
    "citations": [
      {"title": "string", "slug": "string", "last_updated": "YYYY-MM-DD"}
    ],
    "is_clarification": false
  }
  ```
- [ ] Clarification / decline responses (low-confidence, no-match,
      out-of-scope) use this same contract with `is_clarification: true` and
      an empty `citations` array, so the frontend has one consistent shape
      to render

## Phase 6 — API Layer (FastAPI)

- [ ] `app/main.py` — `POST /api/chat`, accepts `{"message": "string"}`
- [ ] Wire request → retriever → (confidence check) → generator → validator
      → formatter → response
- [ ] `GET /api/health` — basic liveness check (useful once deployed)
- [ ] No user accounts, no chat history persistence server-side (stateless
      per request, matching the single-turn scope above)

## Phase 7 — Frontend (Next.js)

- [ ] Minimal single-page chat UI
  - Welcome message explaining scope (fresher/campus-life Q&A, sourced from
    curated content)
  - 3 example questions covering different categories (e.g. one
    `general_onboarding`, one `campus_life`, one governance/policy question)
  - Chat input + message list
  - Each assistant message renders its citation(s) with `last_updated` date;
    clarification/decline messages render without citations
- [ ] Call `POST /api/chat` from the client; no client-side history sent —
      each message is a fresh stateless request (per Phase 6)

## Phase 8 — Automation: Push-Triggered Re-Index

This replaces the reference project's daily cron scheduler — CampusGuide's
content only changes when you push, so re-indexing on a fixed timer would be
wasted work and stale between pushes.

- [ ] `.github/workflows/reindex.yml` in `campusguide-content`, triggered
      `on: push` to relevant branches
- [ ] Workflow calls the ingestion entrypoint (Phases 1–2) against the new
      commit, so Chroma is rebuilt/updated immediately after each content
      change
- [ ] Log per-run: files added/changed/removed, chunk counts, any validation
      failures (missing frontmatter fields, etc.) — surfaced so you notice
      bad content pushes quickly
- [ ] Decide where ingestion actually runs (the Action itself vs. triggering
      a webhook to your backend host) once you pick a deployment target in
      Phase 9

## Phase 9 — Deployment Plan

To be detailed in a separate `deployment-plan.md` once Phases 1–8 are
working locally test-to-test. It will need to cover, at minimum:
- Where FastAPI + Chroma persistent volume live (must persist across
  deploys, since Chroma is local-disk-backed)
- How the push-triggered re-index (Phase 8) reaches the deployed backend
- Where the Next.js frontend is hosted and how it points at the backend API
- Groq API key management (env var / secret store, never committed)

## Notes on What's Deliberately *Not* Copied From the Reference Doc

- No query classifier / refusal-handler split for advisory vs. factual
  queries — CampusGuide doesn't have an "advice" category to gate; its
  refusal logic is entirely about *confidence* and *scope*, already
  specified in `edge-case.md`.
- No PII-stripping layer — no financial/identity data is in scope for this
  assistant, so that compliance surface doesn't apply here.
- No web-scraping/HTML-cleaning ingestion step — content is authored
  Markdown you control, not scraped third-party pages.
- No fixed/small corpus assumption — architecture and retrieval are built to
  scale to an unbounded, growing set of files and categories from day one.
