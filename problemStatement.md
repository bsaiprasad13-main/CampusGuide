---
project: CampusGuide
document: Problem Statement
phase: 0
last_updated: 2026-07-05
---

# CampusGuide — Problem Statement

## 1. What CampusGuide Is

CampusGuide is a Retrieval-Augmented Generation (RAG) chatbot that answers factual
questions from IIT Madras freshers using **only** content curated in a private
GitHub Markdown repository (`campusguide-content`). It is not a general-purpose
assistant — it is a grounded, corpus-restricted Q&A system.

## 2. Who It's For

- **Primary users:** Incoming and current-year freshers at IIT Madras seeking
  factual, procedural, or orientation information (hostel rules, clubs,
  governance bodies, onboarding steps, campus life logistics, etc.).
- **Secondary stakeholders:** DoSt team, WebOps, and content owners who maintain
  the corpus and expect answers to faithfully reflect what they've published —
  nothing more, nothing less.

## 3. Core Problem It Solves

Freshers currently rely on scattered sources — Notion pages, seniors, WhatsApp
groups, institutional PDFs — to answer basic "how do I..." and "what is..."
questions. This information is fragmented, inconsistently updated, and not
always accessible on demand. CampusGuide centralizes this into a single,
always-available, source-grounded interface.

## 4. What CampusGuide Must Do

- Answer only from content present in `campusguide-content` at the time of the
  most recent re-index.
- Attribute every answer to the specific source file(s) it drew from, along
  with that file's `last_updated` date.
- Treat every `.md` file as one self-contained topic; retrieval and citation
  should map cleanly back to specific files/sections, not blended guesses.
- Support an open-ended, growing set of categories — new category folders
  (beyond `general_onboarding`, `campus_life`, etc.) must be usable without
  code changes, since categorization is not a fixed taxonomy.
- Answer questions that are loosely related to fresher/campus life even if not
  explicitly covered by an exact-match file, as long as the answer stays
  within the corpus. If nothing relevant exists, decline rather than invent.

## 5. What CampusGuide Must Never Do

- **Never invent facts.** If the corpus doesn't contain the answer, CampusGuide
  says so — it does not fall back on the model's general knowledge to fill
  gaps.
- **Never answer confidently on a weak or ambiguous match.** If retrieved
  content doesn't clearly address the question, CampusGuide asks the user to
  rephrase or clarify rather than guessing.
- **Never give advice outside factual/informational scope** — no personal,
  medical, legal, financial, or emotional counseling, even if a fresher asks.
  It redirects such questions to appropriate human resources (e.g. DoSt,
  counseling services) where the corpus itself provides that redirection.
- **Never answer questions unrelated to IIT Madras fresher life** (e.g. generic
  Chennai tourism, unrelated academic subject tutoring, general chit-chat)
  unless the corpus explicitly covers it.
- **Never present stale information as current** without surfacing its
  `last_updated` date, so users can judge relevance themselves.

## 6. Success Criteria for Phase 0

- A clear, shared understanding (this document) of scope and boundaries before
  any pipeline code is written.
- An architecture that enforces these constraints structurally (via retrieval
  thresholds, citation requirements, and category-agnostic ingestion), not
  just through prompt instructions.
