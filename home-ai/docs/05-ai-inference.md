# 05 — AI Inference on the Home Machine

**Status:** Research complete (2026-07-27). Findings below are from a dedicated
research pass. **Confidence is marked throughout** — see
[Verification & caveats](#verification--caveats) before acting on any number.

---

## The headline finding

> **Serving-engine choice is worth roughly 10–20× on the initial backfill.**
> This is the single largest technical decision in the project.

Backfilling ~50k emails (embed + classify + summarize) is an **overnight job on
vLLM** and a **~1.5–2 day job on Ollama**. At 200k emails it's a comfortable
weekend versus over a week. Everything else in this document is a second-order
optimization by comparison.

The cause is architectural, not tuning: vLLM has **PagedAttention + continuous
batching + Automatic Prefix Caching (APC)**. Ollama and llama.cpp allocate KV
cache per-request from a slot pool. Our triage workload sends the *same system
prompt and schema 50,000 times*, which is precisely the best case for APC.

---

## Two workload profiles

| Profile | Shape | Dominated by |
|---|---|---|
| **Bulk backfill** — embed/classify/summarize the mail history | 10k–200k items, offline, batchable | **Throughput** |
| **Interactive** — draft a reply, answer a question | 1–3 concurrent humans, streamed | **Latency** |

These want different things, but **not different engines**.

## Engine recommendation

| Job | Engine | Why |
|---|---|---|
| **Bulk backfill** | **vLLM** | Only mature option with PagedAttention + continuous batching + prefix caching. APC is worth more here than SGLang's RadixAttention edge. |
| **Interactive** | **vLLM (same server)** | At 1–3 concurrent users we are not latency-starved. TabbyAPI/ExLlamaV3 wins raw single-stream tok/s but self-describes as rough, and a second stack isn't worth ~20–30%. |
| **Embeddings** | **TEI or Infinity**, separate process | Purpose-built, ~1–2 GB resident, doesn't contend with the LLM engine's memory pool. vLLM *can* (`--runner pooling`) but its own docs call that unoptimized. |
| **Apple Silicon** | **MLX**, not vLLM | vLLM's CUDA optimizations don't apply. Note prefill is the Apple weak spot — bad for backfill, fine for interactive. |
| **Prototyping only** | Ollama | Zero-config and genuinely pleasant. **Do not run the backfill on it.** |

**Run one vLLM instance for LLM work + one TEI instance for embeddings. Do not
run two LLM engines.**

### vLLM quantization trap
vLLM's GGUF support is experimental and slow. Use **AWQ or GPTQ INT4 on Ampere
(3090)**, **FP8 on Ada/Hopper+ (4090/5090)**. A 3090 *cannot* do FP8 (compute
capability 8.6 < 8.9). If a model you want ships only as GGUF, that pushes you to
llama.cpp and you eat the throughput loss.

---

## Tiered model stack

Don't run one large model for everything — it's the wrong shape for this
workload. Proposed stack for a **24GB** card:

| Tier | Model | Quant | ~VRAM | Job |
|---|---|---|---|---|
| 0 | **bge-m3** | FP16 | 1.2 GB | Embeddings |
| 0b | bge-reranker-v2-m3 *or* Qwen3-Reranker-0.6B | FP16 | 0.6–1.2 GB | Rerank RAG hits |
| 1 | Qwen3.5-2B/4B | AWQ-INT4 | 1.5–2.5 GB | Triage × 50k, JSON-constrained |
| 2 | Qwen3.5-9B **+ user LoRA/RoSA** | AWQ-INT4 | ~6 GB | Summarize, draft, RAG QA |
| 2-alt | **Hermes 4 14B** | AWQ-INT4 | ~9 GB | Drafting (see below) |
| — | KV cache + activations | | 8–13 GB | |

The whole stack co-resides on 24GB. Tier 2 can swap to a 35B-A3B MoE at INT4
(~19 GB) for a dedicated summarization phase via vLLM **sleep mode**.

**Co-reside the small models** (embed + rerank + triage ≈ 5 GB — always needed);
use **phase separation** for the large drafting model. Don't architect around
swapping the 4B; it's too small to be worth the complexity.

### Why tiering matters
A 2–4B model at INT4 is ~2 GB and can sit alongside everything else. Anything
larger for triage buys accuracy you can recover more cheaply with few-shot
examples + constrained decoding.

**Consider skipping the LLM entirely for the highest-volume labels:** fine-tune a
300M–600M encoder classifier (EmbeddingGemma or a bge backbone + linear head) on
labels bootstrapped from an LLM. ~100× cheaper per email and more consistent. Use
the LLM only for the ambiguous tail.

---

## Task assignments

### (a) Triage / classification
**Qwen3.5-2B/4B, AWQ-INT4.** Bounded-label task with a fixed schema.

> ⚠️ **Never use a reasoning model here.** Thinking traces multiply output tokens
> 10–50× across 50k emails. This is the single most expensive mistake available in
> the design — it turns a 40-minute stage into a full day. Disable thinking mode
> explicitly and **assert on it**.

### (b) Thread summarization
**Qwen3.5-35B-A3B (MoE) at INT4** if VRAM allows, else Qwen3.5-9B / Gemma 4
26B-A4B. Summarization is prefill-heavy (long thread in, short summary out), so
MoE is the right shape: 35B of knowledge at ~3B active. A 27–31B *dense* model
gives similar quality at ~9–10× the decode cost.

### (c) Reply drafting in the user's voice
**This is where method matters far more than model choice.**

Prior art that maps almost exactly onto this requirement —
[**PanzaMail** (IST-DASLab)](https://github.com/IST-DASLab/PanzaMail):
- **Data playback** — convert the user's sent mail into synthetic
  instruction→email pairs by reverse-instruction.
- **Local PEFT fine-tuning** using **RoSA** (low-rank + sparse), chosen
  specifically because it suits *style transfer* better than plain LoRA.
- **RAG at serve time** over past sent mail.
- ~1,000 emails, **under an hour**, on a **single 16–24 GiB GPU**.

**Recommendation:** 8–14B dense instruct model + RoSA/LoRA adapter on the sent
folder + RAG over past sent mail. Voice is exactly what prompting is worst at, and
a persistent adapter costs **zero prompt tokens per call** — which compounds when
drafting all day.

**Hermes 4 14B** is the strongest no-fine-tune baseline (see
[Hermes](#hermes-where-it-fits)). Quantize the drafting model *less* aggressively
than the triage model — Q5_K_M/AWQ or better. Style and fluency degrade before
benchmarks do, and voice-matching is precisely a fluency task.

### (d) RAG question-answering
Reuse the summarization/drafting model. **The model is not the bottleneck —
retrieval is.** A reranker moves answer quality more than 9B → 31B does.
`Qwen3-Reranker-0.6B` is instruction-steerable ("relevance = commitments I made"),
which is genuinely useful for email.

### (e) Embeddings
**bge-m3.** Beats the higher-MTEB Qwen3-Embedding-8B *for this application*:
8K context (a whole thread in one chunk), 100+ languages, ~1.2 GB instead of
~16 GB, and — critically — **native dense + sparse + multi-vector from one
forward pass**. Email retrieval is unusually keyword-sensitive (invoice numbers,
project codenames, ticket IDs) and pure dense embeddings systematically lose
those. Free hybrid retrieval beats 3–4 MTEB points here.

**Do not use** any 512-token model (multilingual-e5-large, most gte/E5 variants).
Email threads blow past it constantly.

---

## Quoted-reply bloat — the highest-leverage preprocessing step

Not minor cleanup. Quoted text is typically **50–80% of raw bytes** and is
near-duplicated across every message in a thread. Left in, it:
1. Multiplies embedding compute 2–5×,
2. Causes **retrieval degeneracy** — every message in a thread embeds to nearly
   the same vector,
3. Burns classification prefill on text the model already saw.

**Approach:**
- **Use a battle-tested library, don't regex it.** `talon` (Mailgun) or
  `email-reply-parser` (GitHub's). Regex-only fails on non-English clients and
  Outlook's `-----Original Message-----` variants.
- **Layer three strategies:** structural (`<blockquote>`, `>`-prefixed lines) →
  header-marker (`On <date>, <person> wrote:`) → **hash-based dedup** (hash
  normalized paragraphs, drop any already seen in-thread). Dedup is the
  client-agnostic backstop.
- **Strip signatures and legal disclaimers** separately (talon has an extractor).
- **Keep the raw original.** Store `body_raw` *and* `body_clean`. Embed/classify
  on clean, display raw. You *will* want to re-run cleaning with a better
  algorithm without re-fetching mail.
- **Measure it.** Log compression ratio per message; >95% removed usually means a
  parse failure, not a short reply.

## Chunking

1. **Chunk at the message boundary**, not by character count. One email = one
   chunk. Split only messages exceeding context (rare after cleaning).
2. **Two-level index:** individual messages *and* an LLM-generated thread summary
   as its own chunk. Thread-level questions hit the summary; specific questions
   hit the message.
3. **Prepend `From:/To:/Date:/Subject:`** to the embedded text so the vector
   carries participant and temporal signal — *and* store them as filterable
   metadata. Dense retrieval is bad at "emails from Sarah in March"; metadata
   filters are perfect at it.
4. **Attachments separately**, linked to the parent message.

---

## Structured output for triage

**Use vLLM + XGrammar with a JSON Schema.** XGrammar caches compiled grammars and
we reuse *exactly one schema 50,000 times* — its best case. Keep vLLM's `auto`
backend so it falls back if the schema hits a coverage gap.

**Constrained decoding over native tool/function calling:** function calling is a
prompting convention plus a parser, not a guarantee. We need a hard guarantee and
the model isn't *choosing* a tool — there's one output shape.

> ⚠️ **Constrained decoding can reduce *semantic* accuracy even while guaranteeing
> valid syntax.** Masking high-probability tokens distorts the distribution.
> Reported: one extraction task fell **86.9% → 70.0%** under structured mode.

**Schema design rules — these matter more than backend choice:**
1. **Enums, not free strings.** A free-text category field is where accuracy dies.
2. **Put reasoning outside the constrained region** — vLLM's `structural_tag`
   constrains only content inside given tags. Directly mitigates the degradation
   above.
3. **No float confidence.** `["low","medium","high"]` — a 3B model's `0.87` is noise.
4. **Flat objects, fixed key order.** Nesting and optionals multiply grammar states.
5. **Validate post-hoc anyway.** Grammar guarantees shape, never correctness.
   Budget a 1–3% re-queue rate.

---

## Throughput estimate — 50k emails, 24GB GPU

> ⚠️ **ESTIMATE, not measurement.** Derived from component figures plus roofline
> arithmetic. Trust the order of magnitude; **benchmark 1,000 emails first.**

Assumptions: ~300 tokens/email after cleaning (40% reduction), ~3 emails/thread
(~17k threads), 4B triage model, 8–9B summarizer, vLLM with APC.

| Stage | Estimate |
|---|---|
| Embedding (bge-m3 on TEI, 18M tok) | 10–30 min — **not the bottleneck** |
| Triage (4B, 15M prefill + 2M decode) | ~40 min |
| Summarization (8–9B, 15.3M prefill + 2M decode) | ~70 min |
| **Pure GPU compute** | **~2–2.5 h** |
| **Realistic** (I/O, retries, parse failures, swaps) | **4–8 h — overnight** |
| Same work on **Ollama** | **~38 h** |
| 200k emails on vLLM | ~16–32 h |

### Where this estimate breaks
| Risk | Impact |
|---|---|
| Quote-stripping only removes 15%, not 40% | ±2× — **measure this first** |
| Reasoning traces enabled on triage | **10–50×** — catastrophic |
| `--max-model-len` left at 128K | up to **25×** (batch size collapses) — cap at 8K for backfill |
| Prefix caching not actually hitting | ~1.5× — verify vLLM's logged hit rate |
| 3090 vs 4090 | ~1.3–1.5×, plus no FP8 |

### Backfill implementation
Run **three separate single-model passes**, not one interleaved pipeline. Each
loads one model, runs at max batch, unloads. Maximizes batch size per stage and
avoids swap thrash. **Checkpoint per-email to the DB** — an 8-hour job *will* get
interrupted.

---

## Hermes: where it fits

The user runs Hermes Agent locally. Assessment:

**Use Hermes for reply drafting (and optionally the agent layer). Not for
triage, not for embeddings.**

- **Not triage** — smallest current Hermes is 14B; we need 2–4B at that volume.
- **Not embeddings** — Nous doesn't publish an embedding model.
- **Yes drafting** — the real argument is **refusal rate**. Genuine correspondence
  includes collections demands, terminations, legal disputes, and firm pushback. A
  model that softens or refuses these is *actively breaking the product*. Hermes is
  tuned for neutral alignment and low refusal, which matters here far more than it
  sounds. It also holds a persona consistently over long context.
- **But** the edge shrinks if we fine-tune: a RoSA adapter on the sent folder
  overwrites the assistant register directly. Hermes is strongest in the
  *no-fine-tune* case.

> ⚠️ **Architectural conflict:** Hermes Agent requires **≥64K context** for
> reliable tool use. A 36B model at Q4 on 24GB leaves only ~32K. **Hermes 4 14B
> AWQ-INT4 (~9 GB) is the only Hermes that clears 64K on a 24GB card.**

**Decisive experiment:** run Hermes 4 14B AWQ against Qwen3.5-9B+LoRA on 50 real
drafts from the sent folder, scored blind — plus a **refusal regression suite**
(termination letter, collections demand, legal-threat response, firm complaint).
Any model that softens or refuses these fails. That test alone may settle it.

---

## Verification & caveats

**Research egress was restricted.** `huggingface.co`, `arxiv.org`, `docs.vllm.ai`,
`nousresearch.com`, and others returned 403. Model cards and papers could not be
read directly; GitHub was reachable and was used as the anchor wherever possible.

**VERIFIED** (read from primary source): vLLM structured-output backends and
`structural_tag`; vLLM sleep mode; Ollama `format` param and `/api/embed`;
ExLlamaV3 capabilities and its own stability disclaimer; PanzaMail architecture,
RoSA, ~1000-email requirement, 16–24 GiB; Hermes Agent's 64K context floor.

**UNCERTAIN** (search summaries only): all throughput multipliers and tok/s
figures; all MTEB scores; Gemma 4 / Qwen3.5 specs; quantization perplexity deltas.

**ESTIMATED** (arithmetic, not measured): the entire throughput section.

**Open conflicts to resolve before committing:**
- Qwen3-Embedding max context — 8K vs 32K (determines chunk size)
- EmbeddingGemma max context — 2K vs 8K
- XGrammar vs Outlines reliability ranking
- Hermes 4 14B's own license tag (base is Apache 2.0, but the fine-tune's tag is
  unconfirmed)
- No current BFCL tool-calling score found for Hermes 4/4.3 — conspicuous given
  Nous markets tool use heavily

**Verify these on the target machine**, where those domains are presumably
reachable, before committing to a model.

---

## Next actions

1. Record the machine's real specs in [`09-open-questions.md`](09-open-questions.md).
2. **Benchmark 1,000 emails end-to-end** before trusting the throughput table.
3. Measure actual quote-stripping compression ratio — the biggest estimate lever.
4. Run the drafting bake-off + refusal regression suite.
