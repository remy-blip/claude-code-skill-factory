# 08 — Roadmap

Phased plan from planning to daily driver. Each phase has an exit criterion; do
not start the next phase until the current one's exit is met. Phases are
deliberately front-loaded with **de-risking** — prove the hard, uncertain things
before building polish.

## Phase 0 — Planning (current)
Design docs, requirements, architecture, and the stack framework (this repo).
- **Exit:** open questions in [`09-open-questions.md`](09-open-questions.md) that
  block prototyping are resolved (esp. machine specs, target mail account).

## Phase 1 — De-risk the two hard things (spikes, throwaway)
Prove the uncertain assumptions cheaply, in whatever language is fastest.
1. **Inference spike:** stand up a local model server on the home machine; run
   triage/summarize/draft/embed against a sample of *real* mail. Measure quality,
   latency, tokens/sec, memory headroom.
2. **Mail-sync spike:** connect to the target account; pull messages into a scratch
   local store; confirm threading and incremental sync work.
- **Exit:** confidence that (a) local models are good enough for the workloads and
  (b) mail sync is tractable. Results feed the stack ADR.

## Phase 2 — Decide & scaffold
- Fill the [decision matrix](07-stack-evaluation.md) from Phase 1 results.
- Write the **stack ADR** and any other blocking ADRs (DB, vector index, model
  server, UI shell).
- Scaffold the chosen stack with the layered architecture and the Gateway seam.
- **Exit:** a running skeleton with the Mail/Storage/AI-Gateway/Core/UI boundaries
  in place, no real features yet.

## Phase 3 — Walking skeleton (read path)
End-to-end, thin: sync a real inbox → store → **triage + summarize** → display in
a keyboard-navigable UI. AI-derived data cached.
- **Exit:** author can open the app and see their real inbox auto-triaged and
  summarized, fully locally.

## Phase 4 — Search & recall
Full-text + semantic search; RAG "answer with citations" over the mailbox.
- **Exit:** natural-language queries reliably surface the right thread.

## Phase 5 — Drafting & send
Reply drafting in the user's voice; send path (with explicit confirmation).
Pre-computation of likely replies during idle time.
- **Exit:** ≥50% of drafts sent with only minor edits (success criterion #3).

## Phase 6 — Daily-driver hardening
Reliability (crash-safe store, resumable sync), the learning loop (triage
corrections), privacy verification (egress audit doc + packet-capture check),
performance tuning.
- **Exit:** the 2-week daily-driver test (success criterion #7) passes.

## Later / maybe
Multiple accounts · additional providers · action-item extraction · calendar
integration · desktop packaging · another user besides the author. Revisit scope
in [`01-goals-and-scope.md`](01-goals-and-scope.md) before pulling any of these in.

## Sequencing rationale

The risky, expensive-to-reverse uncertainties are **local inference quality** and
**the stack choice**. Phase 1 attacks both with throwaway code so the Phase 2
commitment is made on evidence, not guesswork — consistent with the
"reversible decisions first" principle.
