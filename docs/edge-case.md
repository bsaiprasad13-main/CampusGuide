---
project: CampusGuide
document: Edge Cases
phase: 0
last_updated: 2026-07-05
---

# CampusGuide — Edge Case Handling

This document defines expected behavior for scenarios outside the
happy-path "clear question → clear match → grounded answer" flow.

## 1. Weak / Ambiguous Retrieval Match

**Scenario:** The question embeds close to no chunk with high confidence, or
top-k results are scattered across unrelated categories with low, similar
scores.

**Behavior:** CampusGuide does **not** guess or answer from a weak match. It
asks the user to rephrase or narrow their question.

> Example response: "I'm not fully sure which topic you mean — could you
> rephrase that, or mention which area it's about (e.g. hostel rules, a
> specific club, travel)?"

This applies even if some chunk technically has the highest score among
retrieved candidates — a "highest of a bad batch" is still a bad batch, and is
treated the same as no confident match.

## 2. Question About Content Not Yet in the Corpus

**Scenario:** The question is clearly in-scope (fresher/campus life) but no
file or section addresses it — e.g. a policy that hasn't been documented yet.

**Behavior:** Decline explicitly rather than answering from general knowledge,
and be transparent that it's a coverage gap, not a "no such thing exists"
claim.

> Example response: "I don't have information on that yet in my current
> content — it may not be documented, or I just haven't been updated with it.
> You may want to check with [DoSt/relevant governance body] directly."

This is distinct from Edge Case 1: here, retrieval confidently returns "no
relevant chunk," rather than returning a weak/ambiguous one.

## 3. Out-of-Scope Questions

**Scenario:** The question has nothing to do with IIT Madras fresher/campus
life (e.g. "write my essay," "what's the weather in Paris," general chit-chat,
personal/medical/legal advice).

**Behavior:** Decline clearly and redirect to the intended scope, without
answering from general knowledge even if the model could.

> Example response: "That's outside what I can help with — I only answer
> questions about IIT Madras campus life and onboarding using my curated
> content. Try asking about hostels, clubs, or orientation steps!"

## 4. Loosely-Related / Adjacent Questions

**Scenario:** The question isn't an exact match to any single file but is
plausibly connected to fresher/campus life — e.g. "what do people usually do
around campus on weekends," where the answer can be pieced together from
`campus_life` content on clubs, events, and hangout spots.

**Behavior:** CampusGuide **attempts an answer** if retrieval surfaces
relevant (even if not exact-match) chunks from the corpus with reasonable
confidence, clearly grounding the answer in what's actually documented. If
retrieval comes back weak for this too, it falls through to Edge Case 1
(ask to rephrase) rather than inventing filler content.

It does **not** supplement with outside/general knowledge (e.g. suggesting a
real Chennai attraction not mentioned anywhere in the corpus) — the boundary
is "loosely related but still corpus-sourced," not "loosely related so
anything goes."

## 5. New / Unrecognized Category Folder

**Scenario:** A new category folder (e.g. `academics`, `mess_and_dining`) is
added to `campusguide-content` for the first time.

**Behavior:** Fully automatic — no manual allowlist, registration, or code
change is required. Once the GitHub Action re-indexes after the push, files
in the new folder are chunked, embedded, and stored exactly like any existing
category, and become queryable immediately. Category is treated purely as
metadata attached to chunks (used for citation/filtering), not as a routing
key the pipeline needs to know about in advance.

**Implication for content owners:** Adding a genuinely new type of content
just means creating the folder and files with correct frontmatter — no
separate deployment step is needed for CampusGuide to start using it.

## 6. Conflicting or Duplicate Information Across Files

**Scenario:** Two files in the corpus appear to address the same question
with different details (e.g. an outdated policy file and its replacement).

**Behavior (Phase 0 default — flag for future refinement):** Since one file =
one self-contained topic, near-duplicate coverage across files should be rare
by design. If it happens, CampusGuide surfaces both with their respective
`last_updated` dates so the user can judge which is current, rather than
silently picking one. This should be treated as a content hygiene issue to
fix at the source, and revisited once real occurrences are observed.

## 7. Stale Content

**Scenario:** A cited file has an old `last_updated` date relative to when
the question is asked.

**Behavior:** CampusGuide always surfaces the `last_updated` date alongside
the answer (per architecture.md), rather than deciding on the user's behalf
whether the information is "too old" to trust.
