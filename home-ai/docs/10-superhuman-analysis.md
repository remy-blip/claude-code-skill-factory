# 10 — Superhuman: Competitive Teardown

**Status:** Research complete (2026-07-27). Confidence markers throughout:
**VERIFIED** = direct excerpt from a Superhuman-owned source · **REPORTED** =
third-party press/review/forum · **UNCERTAIN** = conflicting or unconfirmed.

> ⚠️ **Research caveat:** WebFetch returned 403 on every domain in this
> environment (including control sites), so all findings come from web search,
> which surfaced substantive excerpts from primary sources. No claim below was
> read from a live page directly.

---

## The strategic finding

> **Superhuman has no unified inbox, and it's their most-cited structural gap.**
> It also supports **only Gmail and Microsoft 365 — no generic IMAP** (VERIFIED
> via their own help center, which routes complaints to a feedback form).

Our **primary requirement** — one hub where all accounts meet — is the thing
Superhuman users complain about most and the vendor has not addressed. That is
not a "we can match them" position; it's a "we can beat them on the axis that
matters most to us" position. Combined with generic IMAP support, this is the
clearest differentiation available and should be treated as a **core feature, not
a nice-to-have**.

Second differentiator: **privacy posture** (see [§4](#4-what-actually-touches-their-cloud)).

---

## 1. Feature inventory

### AI features

| Feature | What it does | Notes for us |
|---|---|---|
| **Ask AI** | NL search over mail + calendar, synthesized answers **with citations** back to source emails; can schedule meetings conversationally. Indexes up to 5 years of mail (excl. Trash/Spam); initial index can take **days** on large inboxes. | Confirms our RAG + reranker design, and confirms **backfill is a days-long job even for a funded team**. Citations are table stakes. |
| **Auto Summarize** | One-line AI summary above **every conversation in the inbox list**, updating live as messages arrive. | Not on-demand — this is continuous, whole-inbox summarization. Validates our **pre-computation** strategy and the tiered-model economics. |
| **Instant Reply / Auto Drafts** | Three ready-to-send drafts under each conversation. Auto Drafts writes **unprompted** for follow-ups, direct replies, and scheduling; primary draft + 2 alternates, with **tone personalized per-recipient** from your history with that specific person. | Per-recipient tone adaptation is a higher bar than "sounds like me." Our RAG-over-sent-mail approach can reach it — retrieve prior correspondence *with that contact*. |
| **Auto Labels** | Classifies incoming mail (marketing, cold pitch, social, needs-response) at ingest. **Custom Auto Labels** (Business tier) take a natural-language prompt with live preview + ✔️/✖️ feedback. | The live-preview-with-feedback UX is worth copying outright. **Known gap: custom prompts can't be edited, only recreated** (VERIFIED). Trivially beatable. |
| **Split Inbox** | User-defined inbox segments (VIP, Newsletters, Team…), with sensible defaults. | Our Now/Later/FYI/Noise proposal is the same idea. Their "defaults + full customization" split is the right pattern. |

**Model disclosure (REPORTED, TechCrunch July 2026):** uses "frontier models from
Anthropic and OpenAI." The only concrete provider disclosure found — third-party,
not confirmed by Superhuman. **This is the bar our local models must clear.**

### Ask AI architecture (VERIFIED — LangChain case study)
V1 was single-prompt RAG: LLM generates retrieval params via JSON mode → hybrid
search + heuristic reranking → LLM synthesizes. **Their documented failure modes
are a free list of what to design around:** unreliable instruction-following,
weak date/temporal reasoning, and failures on calendar-availability and
multi-step queries. Fixed with structured "chatbot rules" and repeated
instructions ("double dipping").

**Retrieval backend (VERIFIED — turbopuffer case study):** migrated to
turbopuffer (object-storage-backed) for cost at multi-year scale. Before that
they hit **self-imposed write throttles (~400 req/s/node) and >24h indexing lag
at peak**. A funded team hit real indexing-scale walls here — our single-machine
backfill estimates should be held humbly.

### Everything else
Snippets (variables, team-shared) · Command Palette (`Cmd+K`) · ~105 keyboard
shortcuts · Undo Send (10s delay, `Cmd+Shift+Z` instant-send opt-out) · Snooze ·
Follow-up reminders (1–7 day) · Read Statuses · Contact Pane / social insights
(**enriched via Clearbit — a third party**) · Team Comments & Shared
Conversations · Calendar (Share Availability, Find Time, auto-attached
Zoom/Meet/Teams links) · Search operators (`AND` default, `OR`, `-exclude`,
quoted exact, `from:`, `has:attachment`) · iOS/Android apps claiming parity via
gestures (swipe right = archive, left = snooze).

**Platform: web + Electron desktop** — not fully native. Relevant to our UI
decision: their famous speed is achieved *in Electron*, which means the
local-first data architecture is doing the work, not native code.

---

## 2. What actually makes it feel fast

This is the most transferable part of the teardown.

- **The 100ms doctrine (VERIFIED)** — a blog post explicitly invoking Paul
  Buchheit's 100ms "feels instant" threshold as a hard engineering constraint.
- **Measure against paint, not JS (VERIFIED)** — they use
  `requestAnimationFrame()` to measure against actual browser paint cycles
  rather than naive JS timers. Cheap to adopt; most teams never do it.
- **Local-first database (REPORTED, consistent)** — mail cached locally renders
  near-instantly and works offline; UI reads local state first, syncs async.
  **This is the single most important architectural lesson**, and it's already
  how our design works — the local mail store *is* the speed mechanism.
- **Predictive prefetch (REPORTED)** — pre-renders threads you're likely to open.
- **Per-provider backends, deliberately not abstracted (VERIFIED, direct quote):**
  *"what made Superhuman fast on Gmail would have made it slow on Outlook"* —
  they rebuilt data-fetch/sync from scratch per provider rather than share one
  abstraction. **This directly contradicts the instinct to build one clean
  provider-agnostic mail layer.** We should still normalize *downstream* of sync,
  but expect the sync path itself to be provider-specific. Worth an ADR.
- **Usage-frequency ranking in the command palette (VERIFIED)** — results ranked
  by expected invocation frequency, not fuzzy-match score ("Use Snippet" above
  "Create Snippet"). Small, high-impact.
- **Cache pre-warming on intent (VERIFIED)** — search index is warmed when Ask AI
  *opens*, not on a schedule.
- **Per-action safety margins** — Undo Send's 10s delay with an opt-out shows
  speed and safety tuned per-action rather than globally.

---

## 3. Criticisms and gaps — what to beat

All REPORTED unless noted.

1. **No unified inbox.** Most-cited structural gap. → **Our core feature.**
2. **No generic IMAP; Gmail + M365 only** (VERIFIED). → Ours should take IMAP.
3. **No shared/team inbox.** Not our target anyway (single-user).
4. **Price.** HN commentary estimates Gmail + shortcuts gets 70–80% of the value
   free. Self-hosted at zero marginal cost is a strong answer.
5. **⚠️ AI features reportedly made it *slower*, from ~July 2025.** Trustpilot
   reviews describe "extremely slow AI features," one user reporting emails taking
   "a couple of minutes just to open." **The company famous for 100ms shipped AI
   that broke its own core promise.** This is the single most important cautionary
   finding in the teardown — see [Design lessons](#design-lessons).
6. **Mobile friction** — awkward copy/open flows, though ratings stay ~4.6/5.
7. **Continuity anxiety** post-acquisition.
8. **Custom Auto Label prompts can't be edited** (VERIFIED).
9. **The 2019 Read Receipts scandal** — tracking pixels logged recipient opens by
   default without consent; early versions logged IP-derived location. Removed
   after backlash, but critics noted open-tracking of non-consenting third parties
   remains intrinsic to the feature. A durable trust deficit worth contrasting
   against — and a reason to make read receipts **off by default, if we ship them
   at all**.

**Corporate context:** Grammarly acquired Superhuman in July 2025; in October 2025
Grammarly renamed the *parent company* to "Superhuman." Widely-cited ~40M DAU /
~$700M ARR figures are for the **merged entity**, not Superhuman Mail. Don't cite
them as Mail's user base.

---

## 4. What actually touches their cloud

- **VERIFIED:** Gmail/Workspace and Outlook/M365 only. OAuth2 delegated to the
  provider; Superhuman never stores passwords.
- **VERIFIED:** Hosted on **Google Cloud Platform**; Cloud KMS for keys; TLS in
  transit; ALTS internally; SOC 2 Type 2, inherited ISO 27001.
- **⚠️ Contradiction, resolved:** one search-synthesized claim says "your inbox is
  never stored on Superhuman's servers, only on your machine." This conflicts with
  (a) the turbopuffer case study — years of email content indexed into a
  cloud-hosted search backend — and (b) Superhuman's own *Agents Privacy and
  Security Overview*, which states plainly: *"Ingested data is stored on
  Superhuman servers so the agent can use it across runs."*
  **Assessment: the "local-only" framing is very likely wrong.** The defensible
  model is an **encrypted server-side mirror + cloud vector index on GCP**, plus
  local client caching for speed.
- **VERIFIED:** No data sale, no ad targeting.
- Contact Pane enrichment sends contact addresses to **Clearbit**, a third party.

### The honest positioning
Not "they store your mail and we don't" — too glib. The accurate contrast:

> Their speed and AI **depend on** a server-side mirror of your mailbox plus a
> cloud vector index on GCP, and their frontier-model AI features send content to
> Anthropic/OpenAI. We do inference locally with **no server-side mirror at all**.

That's a claim we can *prove* with a packet capture — which is what makes it worth
more than marketing copy. See [`06-privacy-security.md`](06-privacy-security.md).

---

## 5. Pricing

| Tier | Monthly | Annual | Includes |
|---|---|---|---|
| **Starter** | $30/user/mo | ~$25/mo | Split Inbox, shortcuts, Write with AI, Auto Summarize, Read Receipts, Snippets, calendar |
| **Business** | $40/user/mo | ~$33/mo | **+ Auto Drafts, Ask AI, Custom Auto Labels**, HubSpot/Salesforce, recent-opens |
| **Enterprise** | Custom | — | Admin controls, custom security/compliance |

No free tier. Trial length **UNCERTAIN** (sources conflict: ~30 days vs 14 days).

**The flagship AI features are gated to the $40 tier.** For a single user that's
**~$400–480/yr** — a useful anchor for what this project is worth building.

---

## Design lessons

1. **Unified inbox + generic IMAP are the differentiation.** Not AI parity — AI is
   table stakes now. Promote both to core requirements in
   [`02-requirements.md`](02-requirements.md).
2. **⚠️ Do not let AI break the speed promise.** Superhuman — the 100ms
   company — reportedly shipped AI that made opening email take *minutes*. Our
   defense is architectural and already in the plan: **AI output is pre-computed
   and cached; the UI reads local state and never blocks on a model.** This should
   be a hard invariant, not a preference. Add it as a non-functional requirement
   with a test.
3. **Expect provider-specific sync paths.** Their explicit refusal to share a
   backend abstraction between Gmail and Outlook is a costly lesson learned in
   production. Normalize downstream of sync, not inside it.
4. **Citations are table stakes** for AI answers over mail.
5. **Per-recipient tone** is the real drafting bar, not generic voice-matching.
6. **Copy the live-preview + ✔️/✖️ feedback UX** for user-defined labels, and make
   prompts **editable** — a free win over a VERIFIED gap.
7. **Read receipts off by default**, if shipped at all.
8. **Steal the cheap speed tricks:** `requestAnimationFrame` measurement,
   usage-frequency command ranking, prefetch on intent.

---

## Open questions this raises

- Do we ship read receipts at all? (Ethics vs. parity — recommend no, or strictly
  opt-in per-message.)
- Is calendar integration in scope for v1? Superhuman treats mail+calendar as one
  surface, and Ask AI spans both. Currently **out of scope** in
  [`01-goals-and-scope.md`](01-goals-and-scope.md) — worth revisiting.
- Do we need per-recipient tone in v1, or is generic voice-matching enough to
  start?
