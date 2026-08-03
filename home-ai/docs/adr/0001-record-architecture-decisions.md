# 0001 — Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-07-27

## Context

This project starts with a deliberate planning phase and several deferred,
hard-to-reverse decisions (stack, mail protocol, storage, model serving, UI
shell). We need a lightweight, durable way to capture *why* each decision was
made, so the reasoning survives past the moment and future changes are made with
full context.

## Decision

We will use **Architecture Decision Records (ADRs)**, stored as numbered Markdown
files in `docs/adr/`. Each significant architectural decision gets its own ADR.
ADRs are immutable once accepted; a decision that changes is captured by a new
ADR that supersedes the prior one.

## Consequences

- Every major decision has a discoverable rationale.
- Onboarding (including future-self) is faster.
- Slight overhead per decision — acceptable for decisions that are costly to
  reverse; we do **not** write ADRs for trivial or easily reversible choices.
- The first real decisions to record will be the outputs of Phase 1 spikes:
  the stack choice, DB/vector index, and the model-serving approach.

## Alternatives considered

- **A single running "decisions" log** — simpler, but entries get lost and lack
  the structured context/consequences framing.
- **Only capturing decisions in commit messages / PRs** — not discoverable later
  and entangled with implementation detail.
- **No formal record** — rejected; the whole point of the planning phase is to
  preserve reasoning.
