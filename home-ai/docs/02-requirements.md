# 02 — Requirements

Requirements are tagged **[MUST]**, **[SHOULD]**, **[COULD]** (MoSCoW). This is a
living document; refine as prototypes reveal reality.

## Functional requirements

### Mail access & sync
- **[MUST]** Connect to an existing mailbox and sync messages locally.
- **[MUST]** Support incremental sync (only fetch new/changed messages).
- **[SHOULD]** Support at least IMAP; evaluate JMAP and the Gmail API.
- **[SHOULD]** Send mail (SMTP or provider API) for replies.
- **[COULD]** Multiple accounts in one unified view.

### Triage
- **[MUST]** Auto-assign each new thread a priority bucket (e.g. Now/Later/FYI/Noise).
- **[MUST]** Generate a one-line summary per thread.
- **[SHOULD]** Learn from user corrections (re-categorization feedback).
- **[COULD]** Detect action items and deadlines within a thread.

### Summarization
- **[MUST]** Summarize a single thread on demand.
- **[SHOULD]** Summarize automatically for long threads.
- **[COULD]** "Catch me up" digest across the whole inbox since last session.

### Drafting
- **[MUST]** Generate a reply draft given a thread and optional user intent.
- **[SHOULD]** Match the user's writing style/voice.
- **[COULD]** Offer multiple draft tones (brief / formal / friendly).

### Search & recall
- **[MUST]** Full-text search over synced mail.
- **[SHOULD]** Semantic/natural-language search ("find the thread where…").
- **[COULD]** Answer questions with cited source threads (RAG over mailbox).

### UX
- **[MUST]** Keyboard-first navigation for all common actions.
- **[MUST]** UI actions feel instant; model work happens ahead of the user.
- **[SHOULD]** Show provenance for AI output (which messages informed a summary).
- **[COULD]** Undo/override any AI decision.

## Non-functional requirements

### Privacy & security
- **[MUST]** No message content sent to any third-party service or AI API.
- **[MUST]** Mail-at-rest handling documented; encryption option evaluated.
- **[MUST]** Credentials/tokens stored securely (OS keychain or equivalent).
- **[SHOULD]** Network behavior auditable (a "what does this app phone home?" doc).

### Performance
- **[MUST]** UI interactions <100ms perceived latency.
- **[SHOULD]** Triage of new mail completes before the user would read it.
- **[SHOULD]** Model serving fits within the home machine's VRAM/RAM budget.

### Reliability
- **[MUST]** Local mail store survives crashes without corruption.
- **[SHOULD]** Sync is resumable after interruption.

### Maintainability
- **[MUST]** Clear separation between mail layer, AI layer, and UI.
- **[SHOULD]** Model and provider choices are swappable behind interfaces.

### Portability
- **[SHOULD]** Runs on the target home machine's OS; document assumptions.
- **[COULD]** Reproducible setup (containers or a documented install).

## Traceability

Each requirement should eventually map to (a) an architecture component in
[`03-architecture.md`](03-architecture.md) and (b) a roadmap phase in
[`08-roadmap.md`](08-roadmap.md).
