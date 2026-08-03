# 11 — Mail Plumbing: Auth, Sync, Push, Access

**Status:** Research complete (2026-07-27). This is where projects like this die,
so gotchas are called out explicitly.

> ⚠️ **Research caveat:** `developers.google.com` and `support.google.com` returned
> 403 in this environment. Google OAuth/CASA claims below are corroborated across
> multiple independent secondary sources that cross-agree, **not read from
> Google's primary docs**. Verify before implementing.

---

## The headline finding

> **Use IMAP + App Password for personal Gmail. This sidesteps Google's entire
> restricted-scope CASA verification burden.**

Reaching for the Gmail API "because it's the modern way" would drag us into
**CASA** (Cloud Application Security Assessment) — Tier 2 costs ~$540–1,000 in
lab fees, Tier 3 runs to thousands and weeks, and it's **re-assessed annually**.
For a personal self-hosted app that is pure, avoidable cost.

**Reading mail requires *restricted* scopes** (`gmail.readonly`, `gmail.modify`,
`https://mail.google.com/`), which is exactly what triggers CASA. Send-only
(`gmail.send`) is merely "sensitive" and skips it.

**The escape hatches, and why they don't help:**
- **Stay in "Testing" status** (≤100 test users, no CASA) → **refresh tokens
  hard-expire after exactly 7 days**, forcing interactive browser re-consent
  weekly. Fatal for an unattended home server.
- **"Internal" app type** bypasses CASA, the user cap, *and* the 7-day expiry →
  **only available if the account is in a paid Google Workspace org.** A plain
  `@gmail.com` account can only ever be "External."

**So: IMAP + App Password**, unless we specifically need Gmail Pub/Sub push.

---

## 1. Per-provider auth

| Provider | Recommended path | Notes |
|---|---|---|
| **Gmail (personal `@gmail.com`)** | **IMAP + App Password** | Requires 2FA enabled. Zero OAuth/CASA exposure. Blocked if Advanced Protection is on. |
| **Google Workspace (paid)** | **OAuth, "Internal" app type** | ⚠️ Since **May 2025, Workspace accounts cannot use IMAP with password/App Password at all** — OAuth is mandatory. Internal type then bypasses CASA/cap/expiry. |
| **Microsoft (personal or M365)** | **Graph API + OAuth** | **No CASA equivalent, no 100-user cap.** Register a multi-tenant+personal Entra app and consent to it yourself. |
| **Fastmail** | IMAP or **JMAP** + App Password | Clean, no OAuth. |
| **Proton** | IMAP via mandatory local **Bridge** (paid plan) | No server-side IMAP exists — zero-access encryption. |
| **iCloud / generic IMAP** | IMAP + app-specific password | ⚠️ iCloud caps at **~5 concurrent connections**. |

### Microsoft's deprecation clock (hard dates)
Basic Auth for IMAP/POP/SMTP on Exchange Online: **rejections ramp from April 30,
2026 → default-disabled for existing tenants ~December 2026 → full removal H2
2027.** Treat OAuth as mandatory for anything new.

Nuance: Microsoft deprecated **Basic Auth over IMAP**, not IMAP itself. IMAP with
XOAUTH2 keeps working. But since you're doing OAuth anyway, **Graph is
preferable** — delta queries and a richer API for the same app registration.

Personal Outlook.com lost password-based IMAP in third-party clients on
**Sept 16, 2024**. Whether any App-Password fallback survives is **UNCERTAIN
(sources conflict)** — plan for OAuth and verify empirically.

### Gmail's IMAP oddity
Gmail supports **IDLE and CONDSTORE but NOT QRESYNC** — the one major provider
with that combination. Code around it explicitly.

---

## 2. Push without a public endpoint

The home server may be behind NAT/CGNAT. What actually works:

| Mechanism | Public endpoint? | Verdict |
|---|---|---|
| **IMAP IDLE** | No — outbound long-lived connection | ✅ Best natural fit. Per-account connection caps (Gmail ~15, Yahoo ~10, **iCloud ~5**); ~30 min IDLE timeout needs reissue. |
| **Gmail Pub/Sub — *pull* subscription** | **No** | ✅ **Key finding.** Push-mode needs a public endpoint; **pull-mode does not** — our server polls Pub/Sub's API outbound-only and still gets push-like latency with zero exposed surface. `watch()` needs renewal ≤7 days. |
| **Graph webhooks** | **Yes, mandatory** — no pull alternative | ⚠️ Needs Cloudflare Tunnel, or fall back to delta-query polling. |
| Plain polling | No | Always-works fallback. |

**Tailscale/Cloudflare Tunnel only change the answer for Graph.** Gmail (pull)
and generic IMAP (IDLE) need neither.

---

## 3. The Android push problem

**This is the constraint most likely to force a privacy compromise, and the two
research passes disagreed on the recommendation. Both agreed on the facts:**

- **Chrome on Android routes Web Push through Google's FCM infrastructure —
  unavoidably.** VAPID lets you skip having an *FCM account*; it does not
  remove FCM as the *transport*. A PWA on Android **cannot** avoid Google as a
  delivery relay.
- **However**, Web Push payloads are **end-to-end encrypted per RFC 8291**.
  Google sees *that* a notification was delivered and when — **metadata, not
  content**.

**So the real question is whether notification metadata leaking to Google is
acceptable.** That's a judgment call, not a technical one — flagged in
[`09-open-questions.md`](09-open-questions.md).

| Option | Google-free? | Effort |
|---|---|---|
| **PWA** (installable from our web UI) | ❌ metadata via FCM; content encrypted | Lowest — ships with the web UI |
| **Native Kotlin + UnifiedPush + self-hosted ntfy** | ✅ fully | High — real app development |
| Fork Thunderbird for Android (Apache-2.0) | ✅ if UnifiedPush added | High |

**Recommended path: PWA first, native fork later if the metadata leak bites.**
Android (unlike iOS) supports Background Sync and Periodic Background Sync in
Chromium, so a PWA is genuinely capable there. Ship value fast; keep the native
option open.

> ⚠️ **Verify end-to-end if going the UnifiedPush route.** element-x-android
> issue #6551 (Apr 2026) documents apps claiming UnifiedPush support that still
> required `fcm.googleapis.com` reachability.

**FairEmail does NOT implement UnifiedPush** (confirmed via their FAQ — it relies
on device-side IMAP IDLE). K-9/Thunderbird-Android support is **UNCERTAIN** —
verify before betting a fork strategy on it.

### The architectural decision that dissolves most of this

> **The always-on home server holds *all* provider connections (IMAP IDLE,
> Pub/Sub pull, Graph polling). The phone only ever receives a lightweight
> "something changed, come pull from the hub" ping.**

This decouples the provider-protocol problem from the phone-push problem
entirely, and it's the right design regardless of push transport. It also avoids
fighting Android's Doze mode, which aggressively kills long-lived background
connections — holding IMAP IDLE from the phone is fighting the OS.

---

## 4. Remote access

| Option | Verdict |
|---|---|
| **Tailscale** (WireGuard mesh) | ⭐ **Primary recommendation.** Automatic NAT traversal, MagicDNS, ACLs, publicly audited, private keys stay on-device. Zero public exposure for both Android and web. |
| Raw WireGuard | No third-party trust, but you own NAT traversal and key distribution. |
| **Cloudflare Tunnel** | Outbound-only, works behind CGNAT — but Cloudflare becomes a **full traffic intermediary**, not just metadata. Reserve for the one thing needing public reachability (Graph webhooks). |
| Reverse proxy + exposed port | ❌ Lowest setup friction, highest ongoing security burden. Better options exist. |

---

## 5. Multi-account unification

Our core differentiator (Superhuman has no unified inbox — see
[`10-superhuman-analysis.md`](10-superhuman-analysis.md)), so these pitfalls
matter.

### Deduplication
Use RFC 5322 **`Message-ID`** as the cross-account dedup key — stable when two
accounts both receive a copy (CC'd self, mailing lists).

> ⚠️ **Don't just discard the duplicate.** `Message-ID` is not cryptographically
> unique (buggy MTAs reuse or omit it), and copies carry **different per-account
> metadata** — flags, arrival time, folder. Dedup to **one canonical message row
> plus a per-account state layer**.

### Threading — three incompatible models will collide
1. **JWZ algorithm** (`Message-ID`/`In-Reply-To`/`References` tree) — the real
   cross-provider standard.
2. **Gmail's server-assigned `threadId`** — authoritative only *within* that
   Gmail account. It will disagree with JWZ threading computed for a non-Gmail
   copy of the same conversation.
3. **Subject-line fallback** — unreliable; wrongly merges or fragments.

> **Compute our own JWZ-based canonical thread ID from headers.** Treat Gmail's
> `threadId` as a same-account optimization hint only, never ground truth.
> Known JWZ failure mode: mailing lists that rewrite `Message-ID` produce either
> one giant thread or many fragments.

### Identity / send-as
Replies **must go out from the account or alias that received the original** —
not a fixed default. Getting this wrong breaks From-header correctness and can
break SPF/DKIM alignment by sending through the wrong account's credentials.

*(Context: Gmail's own unified-inbox importer caps at 5 external accounts as of
mid-2026 — this is not a solved problem even at Google's scale.)*

---

## 6. Incremental sync mechanics

| Protocol | Cursor | Full-resync trigger |
|---|---|---|
| **IMAP** | `UIDVALIDITY` + `UIDNEXT` + CONDSTORE `HIGHESTMODSEQ` (+ QRESYNC where supported) | `UIDVALIDITY` mismatch → discard local cache, refetch |
| **Gmail API** | `historyId` via `users.history.list` | `404` (historyId too old — window ~a week, inconsistently documented). IDs increase but are **non-contiguous** — don't gap-math, trust the 404 |
| **Graph** | Opaque `@odata.deltaLink` — never truncate or reconstruct | `410 Gone` / `resyncRequired` / `SyncStateNotFound`, commonly after ~7 days idle. Poll ≥daily to keep alive |
| **JMAP** | Opaque `state` string; `/changes` returns a diff | `cannotCalculateChanges` → client MUST invalidate. Servers *SHOULD* (not MUST) support 30-day-old states |

### What robust sync actually requires
1. **Persist the cursor in the same DB transaction as the data it describes.**
2. **Make writes idempotent (upsert)** so replay-after-crash is safe.
3. **Treat the full-resync path as equally important as the incremental path.**
   Every protocol above has a "your cursor is too old, start over" failure mode,
   and that is reportedly *the most commonly missed code path in practice*.

---

## Decisions this settles

- **Gmail: IMAP + App Password.** Avoids CASA entirely. → ADR.
- **Microsoft: Graph + OAuth.** Basic Auth is dying on a hard schedule.
- **Server holds all provider connections; phone gets a thin ping.** → ADR.
- **Tailscale for remote access.** Cloudflare Tunnel only if we implement Graph
  webhooks.
- **Compute canonical JWZ thread IDs; never trust provider threading
  cross-account.**
- **PWA for Android first**, native + UnifiedPush/ntfy if metadata leakage to
  Google proves unacceptable.

## Open questions raised

- Is Web Push **metadata** (not content) reaching Google acceptable? This decides
  PWA vs. native Kotlin — and it's a values call, not a technical one.
- Do we implement Graph webhooks (needs Cloudflare Tunnel) or accept delta
  polling for Microsoft accounts?
- Which accounts are actually in scope, and are any of them Workspace (which
  would change the Gmail path from IMAP to mandatory OAuth)?
