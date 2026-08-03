# 12 — Build vs. Fork: OSS Landscape

**Status:** Research complete (2026-07-27). License text and commit dates were
read from primary sources (`raw.githubusercontent.com` and repo pages). **Star
and issue counts were scraped and are unreliable** — the GitHub API was
403-blocked.

---

## The verdict

> **There is no credible fork base. Building is correct — but only for three
> layers. Do not build the other three.**

This is not "the search came up empty." Each category of existing project fails
on a different **structural** axis:

| What exists | Why it fails |
|---|---|
| Live mail proxies (Roundcube, Cypht, SnappyMail) | Stateless IMAP renderers — **no local corpus**, so nothing to RAG over |
| Rules engine over a provider API (inbox-zero) | **Stores no message bodies at all** — not a mail client |
| Desktop apps (Mailspring, EmailOps, Thunderbird) | No web UI, no mobile |
| Dead (Mailpile, RainLoop, nylas/sync-engine, Mail-0/Zero, mujmap) | — |

The specific combination we want — **normalized local multi-account store +
local inference + web UI + Android** — genuinely does not exist in open source.
That's a real gap, and it's consistent with the finding that Superhuman itself
has no unified inbox.

### Build / buy split

| **DO build** | **Do NOT build** |
|---|---|
| The normalized multi-account schema — *the product's spine* | IMAP protocol handling → **ImapFlow (MIT)** |
| The web UI | Vector + FTS indexing → **SQLite FTS5 + sqlite-vec** |
| The AI orchestration layer | Inference serving → **Ollama / vLLM** |
| | Android client → fork **Thunderbird for Android (Apache-2.0)** eventually |

> **Do not write raw IMAP.** UIDVALIDITY invalidation, CONDSTORE/QRESYNC,
> per-server quirks, Gmail's labels-as-folders — the single most reliable way to
> burn six months.

---

## Candidate assessments

### `elie222/inbox-zero` — alive and serious, but not a mail client
- **Liveness: excellent.** Eight commits on Jul 26, 2026; full-time commercial team.
- **⚠️ License: AGPL-3.0 + added restrictions — NOT open source.** Verbatim from
  `LICENSE`: a **commercial monetization restriction**, and an **enterprise use
  limitation** for orgs with **≥5 employees**. Exemptions cover personal,
  educational, and research use. (Legally incoherent — AGPL §7 forbids adding
  such terms — but the *intent* is unambiguous.) **Fine personally; radioactive
  as a product.**
- **✅ Local models: the best design found anywhere.** Ships `DEFAULT_LLMS=ollama:…`,
  an OpenAI-compatible option for LM Studio/vLLM/LocalAI, and — the good part —
  **per-role model lists with ordered fallbacks**: `NANO_/ECONOMY_/DEFAULT_/CHAT_/DRAFT_LLMS`.
  **Steal this pattern regardless of what we build.**
- **❌ The disqualifier:** `prisma/schema.prisma` has **no message or body
  entity**. It stores `messageId`/`threadId` *references*; bodies are fetched
  live. **No corpus → no offline, no RAG over 100k emails.**
- No generic IMAP. Issue #925 (opened by the maintainer Nov 2025) still open,
  unassigned, and self-described as *"generated in a discussion with Cursor…
  hasn't been deeply reviewed."*

**→ Mine the ideas, don't fork the code.**

### `Mail-0/Zero` — ⚠️ zombie; the most-hyped, least usable
- **The org pivoted away from email.** `github.com/Mail-0` now presents as
  *"Orchid — the delegation layer for work."* Zero last touched **May 26, 2026**;
  commit log reads `"ugh"`, `"idk atp"`, then a nine-month gap before that.
- License: MIT (best license of any candidate, on the deadest project).
- **❌ Local models: NO.** `.env.example` on `main` and `staging` contains only
  `OPENAI_API_KEY` / `PERPLEXITY_API_KEY`. **Third-party blogs claiming Zero
  supports Ollama out of the box are false** — traceable to one incorrect post
  that propagated into search summaries, and contradicted by the repo.
- **❌ Hard-locked to Cloudflare proprietary services** — `wrangler.jsonc` binds
  Vectorize, R2, Durable Objects, Hyperdrive. **The mailbox abstraction *is* a
  Durable Object.** Vectorize and Durable Objects cannot be self-hosted. The
  README's "self-hosting" is aspirational.

### `emailops/emailops` — ⭐ the sleeper; closest architectural match
- **License: Apache-2.0** — cleanest of any live candidate.
- **Liveness: high but young.** Commit on Jul 27, 2026; **8 releases in 2 months**
  (v0.5.0 May 30 → v0.6.4 Jul 24); ~100 commits total.
- **⚠️ ~6 stars, 0 forks, apparently one developer.** No community, no bus factor.
- Stack: Tauri 2 + Rust (Tokio, `rusqlite`+`refinery`, `oauth2`, `keyring`) + React/TS.
- **Local models best-in-class:** embedded **`llama-cpp-2`** as the *default* —
  in-process, no daemon, no network. Ollama/OpenRouter opt-in, switchable per feature.
- **Full multi-account:** Gmail, MS Graph, **and generic IMAP/SMTP**. Unified inbox.
- Local SQLite store; tokens in OS keychain; local embeddings for semantic search.
- **Substance check passed** — `Cargo.toml` carries real engineering notes (pinning
  `rusqlite` because `refinery` caps its range; dropping `staticlib` to avoid a
  ~590MB macOS relink). Not AI-generated slop.
- **Gaps:** macOS-only, desktop app not web, no Android path.

**→ Best architectural template under the best license — but forking means owning
it from day one.** Recommend **watching it for 2–3 months**: sustained shipping
makes it far more attractive; a stall costs nothing to have learned.

### `nerdyabhi/better-mail` — vaporware
Markets itself as "the open-source Superhuman alternative." `LICENSE` says
**"MIT License with Commons Clause Restriction, Copyright (c) 2026 YOUR_NAME"** —
unedited template placeholder, and Commons Clause isn't open source. Recent
commit: *"removed email engine service."* Ignore.

---

## The sync layer

### Disqualified
- **⚠️ `postalsys/emailengine` — PROPRIETARY, worse than AGPL.** `package.json`
  declares `"license": "LICENSE_EMAILENGINE"`. The agreement grants **14 days**
  of free use, after which *"the core functionality of the Software ceases to
  operate until a valid License Key is provided,"* and forbids tampering with
  the key validation. **Completely disqualified.**
- **☠️ `nylas/sync-engine` — archived Jun 25, 2026**, Python 2. Frequently
  recommended as "just fork this." Do not.
- **☠️ `mujmap`** — last commit May 2023.

### Recommended: `postalsys/imapflow` (MIT, TypeScript)
- Last release **Jul 23, 2026 (v1.5.0)**. By Andris Reinman (author of Nodemailer).
- Recent work includes *"add STATUS SIZE/DELETED and rev2 BINARY fetch support,
  fix protocol bugs found in an RFC 9051 review"* — serious spec compliance.
- **The irony worth noting: ImapFlow is the MIT engine underneath the proprietary
  EmailEngine.** We get the good part for free.
- We write the sync state machine; we get correct protocol handling free — and
  protocol handling is the trap.

**Runner-up: `Foundry376/Mailspring-Sync` (GPL-3.0, C++)** — the only thing that
already *is* what we want: one process per account, SQLite store, **emits
modified objects as newline-delimited JSON on stdout** (clean language-agnostic
IPC), exploits CONDSTORE/QRESYNC, special-cases Gmail via `X-GM-LABELS`,
bandwidth-aware. Alive (Jul 18, 2026). Costs: GPL-3.0, heavy C++/MailCore2 build,
someone else's schema.

**⚠️ License trap:** `bamthomas/aioimaplib` is **GPL-3.0 — on a library**.
Linking it makes the whole app GPL-3.0. If we went Python, `imap_tools`
(Apache-2.0) is license-safe but **lacks CONDSTORE/MODSEQ**, which we need at
100k scale. This is one more reason the TypeScript path is cleaner.

---

## Search and vector storage

**Right-size first:** 100k emails × ~1–3 chunks ≈ **100k–300k vectors** ≈
0.3–1 GB at 768-dim float32, less when quantized. Brute-force cosine over 300k
vectors is *tens of milliseconds*.

> **We need a vector *column*, not a vector *database*.**

| Option | License | Verdict |
|---|---|---|
| **SQLite FTS5** | Public domain | ⭐ Use it. BM25, zero ops |
| **`sqlite-vec`** | Apache-2.0/MIT | ⭐ Best fit — same file as FTS5 and the store. ⚠️ **pre-v1, "expect breaking changes"** |
| **LanceDB** | Apache-2.0 | ⭐ Strong alternative: embedded, vector+FTS, ~4MB idle |
| Qdrant | Apache-2.0 | ❌ Separate server, ~400MB RAM constant — unjustified |
| Meilisearch | MIT | Good, but a second server |
| Typesense | ⚠️ GPL-3.0 | ❌ Copyleft + whole index in RAM + separate server |
| Xapian / notmuch | GPL | ❌ No native vector search |

**Recommendation: ONE SQLite file** — normalized messages + FTS5 + sqlite-vec.
Hybrid retrieval = BM25 ∪ kNN fused with **Reciprocal Rank Fusion**. Zero extra
processes; backup is `cp` of one file.

**Hedge on sqlite-vec's pre-v1 status:** keep embeddings in a plain table and
treat the ANN index as a **rebuildable derived artifact**. A breaking change then
costs a reindex, not a migration. If that risk is unacceptable, use LanceDB.

---

## Android base

**`thunderbird/thunderbird-android` (ex-K-9) — best base.** Apache-2.0, Kotlin,
clean `/backend` `/core` `/mail` module split, large team.

**⚠️ JMAP reality check:** there's an open milestone, and the 2026 roadmap lists
JMAP as something to *"explore"*, ranked below rearchitecture. **Exploratory, not
shipping — don't plan around it.**

Two ways to point it at our hub: (a) **expose IMAP from our hub** → works today,
zero client changes, but no AI affordances; (b) fork and add a custom backend →
real work, correct long-term given Apache-2.0.

`M66B/FairEmail` is maintained but IMAP-coupled by design and largely
single-maintainer. **Thunderbird wins** on license, modularity, and team size.

---

## Recommended assembly

```
Sync:   ImapFlow (MIT) + Gmail IMAP/App-Password + MS Graph
        one worker per account
Store:  ONE SQLite file — messages + FTS5 (BM25) + sqlite-vec (kNN), RRF fusion
AI:     Ollama/vLLM behind ONE OpenAI-compatible interface
        role-based routing (from inbox-zero): NANO/ECONOMY → 4B, DRAFT/CHAT → larger
UI:     Web UI → installable PWA for Android (Phase 1)
        optionally expose JMAP → Thunderbird-Android fork later
```

**Three leverage points:**
1. **The OpenAI-compatible shim is the key abstraction.** Ollama, vLLM, LM Studio,
   and llama.cpp all speak it, so the inference backend stays swappable — which is
   exactly the seam [`03-architecture.md`](03-architecture.md) already calls for.
2. **Exposing JMAP decouples clients** and gives a real Android path.
3. **Keep the AI service separate from sync.** Sync is I/O-bound and
   latency-sensitive; inference is compute-bound and slow. Coupling them means a
   30-second draft blocks mailbox refresh. Mailspring-Sync's stdout-NDJSON
   boundary is the model to copy.

---

## Liveness reference

**☠️ DEAD — frequently mis-recommended:**
**Mailpile** (last commit Nov 2023; a 2022 commit is literally titled *"Explain
that this repo is dormant (for now)"* — still appears in listicles) ·
**RainLoop** (Aug 2022) · **nylas/sync-engine** (archived Jun 2026) ·
**Mail-0/Zero** · **mujmap** (May 2023).

**⚠️ DORMANT:** **SnappyMail** — one commit in the last ~12 months against a
151-issue backlog, single maintainer. Fine client; don't build a product on it.

**✅ ALIVE (verified commit dates):** inbox-zero (Jul 26, 2026) · Roundcube
(Jul 25) · **Cypht** (Jul 26 — LGPL-2.1, best multi-protocol reference:
IMAP+JMAP+EWS) · **Mailspring (Jul 20 — widely assumed stagnant; it is not)** ·
Mailspring-Sync (Jul 18) · **ImapFlow (Jul 23)** · EmailOps (Jul 27) · maddy ·
OfflineIMAP3 · Stalwart · Dovecot · Thunderbird.

**Thunderbird desktop as an extension host — rejected.** Experiment APIs (the only
route to deep functionality) become **ESR-only in 2026** and bypass the permission
system entirely, prompting for *"full, unrestricted access to Thunderbird, and
your computer."* Narrowing API surface, ESR-only distribution, still no web or
Android.

---

## Verification status

**VERIFIED (read from primary sources, 2026-07-27):** all license file contents
quoted above; all commit and release dates; inbox-zero's Ollama config, role-based
model lists, and **absence of message-body storage**; Zero's OpenAI-only config
and Cloudflare bindings; the Mail-0 → Orchid pivot; EmailOps' stack and
`Cargo.toml`; Mailspring-Sync's architecture.

**⚠️ UNCERTAIN — verify before acting:**
- **All star and issue counts** (GitHub API was 403-blocked; scraped from rendered
  pages). Two are implausible on their face.
- **Roundcube's license** — no `LICENSE` at repo root; GPLv3-with-exceptions per
  its website, not verified in-repo.
- **EmailOps' contributor count** — "solo dev" is *inferred*, not confirmed.
  Given it's the top architectural recommendation, **confirm the bus factor
  before committing.**
- **No project was run or tested.** All architectural claims derive from source
  and documentation.
