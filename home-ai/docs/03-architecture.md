# 03 — Architecture (proposed, stack-agnostic)

This describes *components and boundaries*, not a specific language. The stack
decision lives in [`07-stack-evaluation.md`](07-stack-evaluation.md); everything
here is expressed so it can be implemented in either Python or Node/TypeScript.

## Component overview

```
┌─────────────────────────────────────────────────────────────┐
│                          UI Layer                            │
│   keyboard-first client (desktop app or local web UI)        │
└───────────────▲──────────────────────────▲──────────────────┘
                │ reads/commands           │ streamed AI output
┌───────────────┴──────────────────────────┴──────────────────┐
│                     Application Core                         │
│  orchestration · triage pipeline · draft/summarize services  │
│  search service · feedback/learning loop                     │
└───┬───────────────┬───────────────────┬─────────────────────┘
    │               │                   │
┌───▼─────┐   ┌─────▼──────┐      ┌──────▼────────┐
│  Mail   │   │  Storage   │      │  AI Inference │
│  Layer  │   │  & Index   │      │   Gateway     │
│ IMAP/   │   │ mail store │      │ local LLM +   │
│ JMAP/   │   │ + vector   │      │ embeddings    │
│ API     │   │   index    │      │ (home machine)│
└─────────┘   └────────────┘      └───────────────┘
```

## Components

### 1. Mail Layer
- Talks to the provider (IMAP/JMAP/Gmail API) for fetch and send.
- Normalizes messages into the internal [data model](04-data-model.md).
- Handles incremental sync, threading, and outbound send.
- **Boundary:** the rest of the app never speaks a wire protocol directly.

### 2. Storage & Index
- **Mail store:** canonical local copy of messages/threads (embedded DB).
- **Search index:** full-text + a **vector index** for semantic search/RAG.
- Must survive crashes; sync state is persisted here.

### 3. AI Inference Gateway
- Single interface the app uses for all model calls: `summarize`, `classify`,
  `draft`, `embed`, `answer`.
- Wraps the local model server on the home inference machine (see
  [`05-ai-inference.md`](05-ai-inference.md)).
- **Swappable:** the model/runtime behind this gateway can change without
  touching application logic. This is the key seam that keeps us
  stack-and-model flexible.

### 4. Application Core
- **Triage pipeline:** on new mail → classify + summarize + prioritize → store.
- **Services:** summarize-thread, draft-reply, semantic-search, catch-me-up.
- **Feedback loop:** capture user corrections to improve future triage.
- **Orchestration:** schedules pre-computation during idle GPU time so results
  are ready before the user asks.

### 5. UI Layer
- Keyboard-first client. Two candidate shapes:
  - **Local web UI** served by the core (browser on the same machine), or
  - **Desktop app** (native/Electron/Tauri-style shell).
- Consumes streamed AI output; never blocks on the model.

## Key design decisions to make (tracked as ADRs)

- Mail protocol(s) to support first — see [`09-open-questions.md`](09-open-questions.md).
- Embedded DB + vector index choice.
- Local model runtime (e.g. an OpenAI-compatible local server vs. a bound
  library).
- UI shell (local web vs. desktop).
- Language/runtime — see [`07-stack-evaluation.md`](07-stack-evaluation.md).

## Cross-cutting concerns

- **Privacy boundary:** the AI Inference Gateway must only ever reach the local
  model server — enforced and documented (see
  [`06-privacy-security.md`](06-privacy-security.md)).
- **Interfaces over implementations:** Mail Layer, Storage, and AI Gateway are
  each defined by a narrow interface so pieces can be swapped as we learn.
- **Pre-computation:** treat idle inference capacity as the budget that buys the
  "superhuman" feel.
