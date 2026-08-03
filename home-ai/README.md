# home-ai

A **superhuman mail** application designed to run entirely on a local home AI
inference machine. The goal is an email client that reads, triages, summarizes,
drafts, and reasons over your mail with the help of local large language
models — fast, keyboard-driven, and **fully private** because nothing leaves
your hardware.

> **Status: Planning.** This repository currently contains no application code.
> It holds the design work — vision, requirements, architecture, and a stack
> decision framework — that will precede the first line of implementation. The
> stack (Python vs. Node/TypeScript) is intentionally undecided; see
> [`docs/07-stack-evaluation.md`](docs/07-stack-evaluation.md).

---

## Why

Mainstream "AI email" products send your mailbox — arguably your most sensitive
data — to a third-party cloud. A home AI inference machine changes the trade-off:
you can run capable models locally, keep every message on your own disk, and
still get the assistant experience. This project is about building that.

## What "superhuman" means here

- **Instant triage** — inbox is automatically categorized, prioritized, and
  summarized on arrival.
- **Zero-latency feel** — keyboard-first UX; the model works ahead of you.
- **Draft & reply assistance** — context-aware drafting in your own voice.
- **Semantic search & recall** — "find the thread where Sam agreed to the
  budget" works.
- **Local & private by default** — no message content leaves the machine.

## Repository layout (planning phase)

```
home-ai/
├── README.md                     # You are here
├── docs/
│   ├── 00-vision.md              # The north star
│   ├── 01-goals-and-scope.md     # In scope / out of scope, success criteria
│   ├── 02-requirements.md        # Functional & non-functional requirements
│   ├── 03-architecture.md        # Proposed components (stack-agnostic)
│   ├── 04-data-model.md          # Core entities & storage concepts
│   ├── 05-ai-inference.md        # Local model serving on home hardware
│   ├── 06-privacy-security.md    # Threat model & privacy guarantees
│   ├── 07-stack-evaluation.md    # Python vs Node/TS decision framework
│   ├── 08-roadmap.md             # Phased plan from prototype to daily driver
│   ├── 09-open-questions.md      # Unknowns to resolve before building
│   └── adr/                      # Architecture Decision Records
└── LICENSE
```

## How to use this repo right now

1. Read [`docs/00-vision.md`](docs/00-vision.md) for the north star.
2. Work through the numbered docs in order.
3. Resolve the items in [`docs/09-open-questions.md`](docs/09-open-questions.md).
4. Make the stack call in [`docs/07-stack-evaluation.md`](docs/07-stack-evaluation.md)
   and record it as an ADR under [`docs/adr/`](docs/adr/).
5. Only then scaffold the chosen stack.

## License

MIT — see [`LICENSE`](LICENSE).
