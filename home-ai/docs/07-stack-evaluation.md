# 07 — Stack Evaluation: Python vs. Node/TypeScript

The stack is **deliberately undecided**. This doc is the framework for making the
call — and for keeping the decision cheap to defer. Record the final choice as an
ADR ([`adr/`](adr/)).

## Decision principle

The [architecture](03-architecture.md) is built around a **stable AI Inference
Gateway** (an HTTP boundary to a local model server) and narrow interfaces for
the Mail and Storage layers. Because of that, **the language choice is more
reversible than it looks** — we can prototype the risky parts (local inference
quality/latency, mail sync) before committing, and even mix (e.g. a Python
inference helper behind an HTTP gateway consumed by a Node UI).

## Candidate: Python

**Strengths**
- Deepest local-AI ecosystem: model runtimes, embeddings, quantization tooling,
  and inference servers are Python-first.
- Fast to prototype triage/summarize/RAG logic.
- Rich email/IMAP libraries in the standard library and ecosystem.

**Weaknesses**
- Weaker story for a polished keyboard-first desktop/web UI (usually pair with a
  JS front end anyway).
- Packaging a desktop app for an end user is clunkier.
- Async concurrency is workable but less natural than Node for a
  server-pushes-to-UI model.

**Best fit if:** the hard part is the AI/inference/RAG layer and the UI can be a
simple local web page.

## Candidate: Node / TypeScript

**Strengths**
- Excellent UI story (local web or desktop shell like Tauri/Electron), which is
  central to the keyboard-first, low-latency experience.
- One language across UI + core; strong typing for the data model.
- Great streaming/event model for "model works ahead of the user."
- Good JMAP/Gmail-API client story.

**Weaknesses**
- Local-inference tooling is thinner; typically calls a model server rather than
  hosting the model in-process.
- Heavier lifting for embeddings/quantization if done in-process (mitigated by
  the local-server approach).

**Best fit if:** the hard part is the UX and we treat inference as an HTTP
service (which the architecture already recommends).

## The "don't choose yet" option (recommended near-term)

Because the Gateway is an HTTP boundary:
1. Stand up a **local model server** (OpenAI-compatible) — language-neutral.
2. Prototype **inference quality/latency** against the author's real mail using
   throwaway scripts in *whichever* language is fastest to hack (likely Python).
3. Prototype the **UX feel** separately.
4. Let the results — not a preference — pick the app language.

This keeps the expensive, hard-to-reverse decision open while we de-risk the two
genuinely uncertain things (local inference quality, and UX latency).

## Decision matrix (fill in during prototyping)

| Criterion | Weight | Python | Node/TS | Notes |
|-----------|:------:|:------:|:-------:|-------|
| Local inference ecosystem | H |  |  | |
| UI quality / keyboard UX | H |  |  | |
| Streaming / pre-compute model | M |  |  | |
| Mail protocol libraries | M |  |  | |
| Packaging for the home machine | M |  |  | |
| Single-maintainer velocity | H |  |  | |
| Type safety for the data model | L |  |  | |

Score after the prototypes, then write the ADR.

## Recommendation (provisional, revisit after prototyping)

Lean toward **Node/TypeScript for the app (core + UI)** with the **local model
server doing inference**, and Python only if a specific inference/embedding task
is materially easier there (invoked behind the HTTP gateway). Rationale: the
architecture already isolates inference behind a server, so the UX-heavy part —
where the "superhuman" feel is won or lost — benefits most from Node's strengths.
**This is provisional** and must be confirmed by the prototypes above.
