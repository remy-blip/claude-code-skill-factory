# Architecture Decision Records (ADRs)

This directory records significant, hard-to-reverse decisions and the reasoning
behind them, so future-you knows *why* — not just *what*.

## What is an ADR?

A short document capturing one architectural decision: its context, the options,
the choice, and the consequences. ADRs are **immutable once accepted** — if a
decision changes, write a new ADR that supersedes the old one rather than editing
history.

## When to write one

Write an ADR for decisions that are costly to reverse or that others (or future
you) will question, e.g.:
- The language/runtime for the app (see [`../07-stack-evaluation.md`](../07-stack-evaluation.md))
- Mail protocol(s) supported first
- Embedded DB and vector index choices
- Local model server / inference approach
- UI shell (local web vs. desktop)

## How to add one

1. Copy the format of [`0001-record-architecture-decisions.md`](0001-record-architecture-decisions.md).
2. Number it sequentially: `NNNN-short-title.md`.
3. Status starts as `Proposed`, becomes `Accepted` when decided, `Superseded by
   NNNN` when replaced.

## Template

```markdown
# NNNN — <Title>

- **Status:** Proposed | Accepted | Superseded by NNNN
- **Date:** YYYY-MM-DD

## Context
What's the situation and the forces at play?

## Decision
What did we decide, stated plainly?

## Consequences
What becomes easier, harder, or newly required as a result?

## Alternatives considered
What else was on the table, and why not?
```

## Index

| # | Title | Status |
|---|-------|--------|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted |
