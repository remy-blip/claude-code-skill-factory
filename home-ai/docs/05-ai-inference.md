# 05 — AI Inference on the Home Machine

The differentiator is running everything locally. This doc frames the model and
serving decisions. Actual model picks depend on the machine's real specs — record
those in [`09-open-questions.md`](09-open-questions.md).

## Workloads

| Workload | Frequency | Latency need | Notes |
|----------|-----------|--------------|-------|
| Classify/triage | every new message | pre-computed | short output; can be a smaller/faster model |
| Summarize thread | per thread | pre-computed or on-demand | needs decent context window |
| Draft reply | on user action | interactive | quality matters most; streamed |
| Embed for search | on sync | batch/background | embedding model, not a chat model |
| Answer (RAG) | on user query | interactive | retrieval + generation |

Observation: not every workload needs the biggest model. A **tiered** approach —
a small fast model for classification, a stronger model for drafting/answering —
often fits home hardware better than one large model for everything.

## Serving options (to evaluate)

1. **OpenAI-compatible local server** (e.g. a local runtime exposing an
   `/v1/chat/completions`-style API).
   - Pro: clean HTTP boundary = easy swap; both Python and Node can call it
     identically; matches the "AI Inference Gateway" seam in the architecture.
   - Pro: keeps model runtime out of the app process.
   - Con: an extra process to manage.
   - **This is the recommended default** because it keeps the stack decision and
     the model decision fully decoupled.

2. **In-process model library / bindings.**
   - Pro: fewer moving parts; tighter control.
   - Con: couples app runtime to the model runtime; harder to swap; language-
     specific.

3. **Hybrid:** embeddings in-process (cheap, high-volume), chat via local server.

## Model selection axes

- **Context window** — long threads and RAG want more context.
- **Quality vs. speed vs. memory** — the classic trade; pick per workload.
- **Quantization** — fit within the machine's VRAM/RAM; measure quality impact.
- **Embedding model** — separate choice; dimension affects the vector index.
- **Licensing** — ensure models permit local personal use.

## The Gateway contract

Regardless of what's behind it, the app calls a stable interface:

```
summarize(thread)        -> text + source refs
classify(message|thread) -> {priority, category, confidence}
draft(thread, intent?)   -> text (streamed)
embed(texts[])           -> vectors[]
answer(query, context[]) -> text + citations (streamed)
```

Keeping this contract stable is what lets us change models, quantization, or even
the serving option without touching application logic — and lets us prototype the
whole app against a placeholder before the home machine's models are finalized.

## Pre-computation strategy

The "superhuman" feel comes from doing work before the user asks:
- On sync/new mail → immediately classify + summarize + embed.
- During idle GPU time → pre-draft likely replies for high-priority threads.
- Cache all AI-derived artifacts (see [data model](04-data-model.md)); only
  recompute when the underlying mail or the model version changes.

## To measure early (prototype goals)

- Tokens/sec for each candidate model at the target quantization.
- End-to-end triage time for a realistic new-mail batch.
- VRAM/RAM headroom with the tiered model set loaded.
- Summary/draft quality on a sample of the author's real mail.
