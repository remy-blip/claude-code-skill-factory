# 09 — Open Questions

Unknowns to resolve. **Bold** items block prototyping (Phase 1) and should be
answered first. Convert answered questions into ADRs or updates to the relevant
doc.

## Hardware / environment
- **What are the home inference machine's specs?** (GPU model + VRAM, system RAM,
  CPU, disk, OS.) Everything about model selection depends on this.
- Is the machine always-on (for background pre-computation), or does it sleep?
- Is it headless (server) or does it have a display (affects UI shape)?

## Mail account(s)
- **Which mail provider/account is the first target?** (Gmail, generic IMAP,
  Fastmail/JMAP, self-hosted…)
- OAuth vs. app-password vs. IMAP login for that provider?
- Approximate mailbox size (message count) — affects initial sync + index cost.

## Models
- Which local chat model(s) for drafting/answering, at what quantization?
- Which embedding model (and vector dimension)?
- Do we go tiered (small classifier + larger drafter) or single-model?
- Licensing check for each candidate model (personal local use OK?).

## Storage & index
- Embedded DB choice for the mail store.
- Vector index choice (standalone vs. an extension of the mail DB).
- Encrypt at rest by default, or opt-in?

## Serving & runtime
- Which local model server (OpenAI-compatible) to standardize on?
- Where does it bind, and how do we prevent LAN/internet exposure?

## UX
- Local web UI vs. desktop shell?
- Which triage buckets exactly? (Now/Later/FYI/Noise is a starting proposal.)
- Keyboard scheme — model after an existing client the author likes?

## Stack
- Final language/runtime for the app — pending Phase 1 spikes (see
  [`07-stack-evaluation.md`](07-stack-evaluation.md)).

## Product scope
- Single account only for v1, or multi-account from the start?
- Is sending in scope for v1, or read/triage first?

---

### How to use this list
1. Answer the **bold** items before starting Phase 1.
2. For each resolved question, either update the relevant doc or write an ADR.
3. Keep this list pruned — delete resolved items once captured elsewhere.
