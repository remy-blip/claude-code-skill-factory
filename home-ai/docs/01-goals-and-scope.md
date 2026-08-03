# 01 — Goals and Scope

## Primary goal

Deliver a personal email client that runs on a home AI inference machine and
uses local LLMs to triage, summarize, search, and draft — with a keyboard-first,
low-latency UX and a strong privacy guarantee.

## In scope (v1 thinking)

- Connecting to at least one existing mail account (start with a single provider).
- Local storage and indexing of mail for fast search and AI context.
- AI-assisted **triage** (priority + category + one-line summary per thread).
- AI-assisted **thread summarization**.
- AI-assisted **reply drafting** in the user's voice.
- **Semantic search** over the mailbox.
- Local model serving integrated with the home inference machine.
- A privacy posture where message content never leaves the machine.

## Out of scope (for now)

- Hosting/serving mail (we are a client, not an MX).
- Multi-user / team features, shared inboxes, delegation.
- Mobile apps (desktop/web-local first; revisit later).
- Calendar, contacts CRM, task management (integrations later, not core).
- Cloud sync of the AI index across devices.
- Sending your mail to any third-party AI API.

## Success criteria (measurable)

| # | Criterion | Target |
|---|-----------|--------|
| 1 | Time to triage a fresh inbox of N unread | Automatic, complete before user reads |
| 2 | Summary quality | User trusts summaries enough to skip opening "FYI" threads |
| 3 | Draft acceptance | ≥50% of AI drafts sent with minor edits |
| 4 | Search recall | Natural-language queries surface the right thread top-3 |
| 5 | Latency | Common actions feel instant (<100ms UI; model work pre-computed) |
| 6 | Privacy | Zero message content egress, verifiable by network inspection |
| 7 | Daily-driver test | Author uses it as their only mail client for 2 weeks |

## Constraints

- **Hardware:** must run within the home inference machine's resources
  (document actual specs in [`09-open-questions.md`](09-open-questions.md)).
- **Offline-capable:** core reading/triage should work without internet once
  mail is synced.
- **Single maintainer initially:** favor simple, well-understood components over
  cleverness.

## Guiding principles

1. **Privacy is non-negotiable** — a feature that requires egress of content is
   not shipped.
2. **Pre-compute over on-demand** — spend idle GPU cycles so the user never waits.
3. **Boring where possible** — proven storage/protocol choices; save novelty for
   the AI layer.
4. **Reversible decisions first** — prototype cheaply before committing to a stack
   or model.
