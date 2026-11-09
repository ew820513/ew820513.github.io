---
title: "RAG Chunking Strategy Selection: A Decision Framework with Cost, Latency, and Recall Formulas"
date: 2026-11-09
draft: true
tags: ["architecture", "rag", "ai", "chunking", "retrieval"]
---

## Problem

A team builds a RAG pipeline for customer support tickets. They start with fixed-size chunking (500 tokens, 50 overlap). It retrieves relevant documents roughly 60% of the time. Semantic chunking improves recall to 75% but doubles embedding cost and increases retrieval latency by 3x. Late chunking keeps fine-grained context but needs specialized embedding models. The team doesn't know which chunking strategy fits their documents until they're in production.

**A practicing engineer will encounter this when:**

- A ticket about "API rate limit error" gets split so the error code lands in one chunk and the context (which endpoint, which user, which limit) in another — retrieval fails even though the answer is in the corpus
- The embedding bill quietly doubles after switching to semantic chunking on a large corpus
- Retrieval latency triples because chunks grew, or hierarchical/parent pipelines add extra round-trips
- Someone asks "which chunking strategy should we use?" and the answer is a shrug and a guess

Without a systematic pre-production selection method, the choice is vibes — and the failure only becomes visible as production incidents.

## The Chunking Strategies, and What They Actually Cost

| Strategy | How it works | Strengths | Costs / pitfalls |
|----------|-------------|-----------|------------------|
| **Fixed-size** | Split every N tokens with M overlap | Fast, zero ingest-time embeddings, works with any model | Splits mid-sentence/mid-paragraph; error code in one chunk, context in another |
| **Semantic** | Break at sentence/paragraph boundaries via embedding similarity | Preserves semantic units, big recall win | Embeds every boundary candidate at ingest → cost scales with document volume; slower |
| **Hierarchical** | Small chunks for precision + parent chunks for context | Combines precision and context | Two-pass retrieval; complex pipeline |
| **Parent-document** | Small chunks for retrieval, larger parent passed to LLM | Fine-grained retrieval + full context | Needs document-store mapping + two-step retrieval |
| **Late chunking** | Embed whole doc, derive chunk vecs from token-level states | Keeps full context | Needs specialized embedding models; more complex |

The core lesson: **there is no universally best chunking strategy.** The right choice depends on your document type, your retrieval-quality requirements, and your budget for embedding cost and latency.

## Design Decision 1: Chunk Size vs. Overlap Trade-off

The two dials you control:

- **Too small** → loses context. A 100-token chunk about "rate limit" may not include the error code or the endpoint it applies to.
- **Too large** → dilutes signal. A 2000-token chunk buries the answer under unrelated text and slows retrieval.
- **Overlap** → prevents boundary splits, but every overlap token is embedded twice, increasing cost and index size linearly.

**The failure you care about is the boundary split**: the error code in one chunk, its context in the next. Overlap exists specifically to mitigate this — at a per-token cost.

**Where to start**: 512 tokens with 50–100 overlap is a sane default for prose. But treat size and overlap as *parameters in your evaluation harness*, not constants. The optimum shifts by document type: support tickets want units that hold a full conversational turn; financial tables must not be split at all.

## Design Decision 2: Strategy vs. Document Type

Chunk boundaries should follow the document's *natural semantic structure*:

| Document type | Preferred strategy | Why | Suggested unit |
|---------------|--------------------|-----|----------------|
| **Code docs** | Fixed-size with structural splitter (or parent) | Keep class/function boundaries | 128–256 tokens |
| **Support tickets** | Semantic (or parent-document) | Keep conversation threads together | 256–512 tokens |
| **Financial docs** | Fixed-size large, or hierarchical | Keep tables/columns fully intact | 512–1024 tokens |
| **API specs** | Parent-document retrieval | Keep endpoint + schema + examples together | endpoint-sized |
| **Mixed corpora** | Hybrid: route by document type | Best of both | varies |

**Why it matters**:
- **Code docs**: a function split from its signature and docstring retrieves as meaningless fragments. Use a structure-aware splitter (respecting AST/class/function boundaries), not a token counter.
- **Support tickets**: a multi-turn thread is one logical unit. Semantic chunking breaks at turn boundaries so the whole question→answer→follow-up flow stays retrievable.
- **Financial docs**: a balance-sheet table split mid-way isn't just wrong — it actively misleads the LLM. Keep tables whole; use hierarchical parents if you need both precision and the full table.
- **API specs**: retrieve a small chunk (the operation), but pass the *entire endpoint block* to the LLM. That's parent-document retrieval by nature.

## Design Decision 3: Embedding Cost Model (per GB)

**The core cost difference**: semantic chunking embeds *every sentence/paragraph boundary candidate* at ingest time, then discards most. Fixed-size embeds each token roughly once.

Let:
- `D` = document volume (GB)
- `t` = tokens per GB ≈ 250,000 (typical prose)
- `P` = price per 1M tokens (e.g. `text-embedding-3-small` ≈ $0.02/1M)

**Fixed-size** tokens embedded ≈ `GB × 250K` (each token once, plus overlap).

**Semantic chunking** tokens embedded ≈ tokens once *plus* every sentence re-embedded as a candidate boundary. Since each sentence gets embedded at least once more for the similarity check, semantic chunking effectively embeds **≈ 2× the tokens of fixed-size** (often 2×–3× depending on sentence length and the boundary-splitting algorithm).

**Worked example — 100 GB corpus, semantic vs fixed:**

```
Fixed-size:   100 GB × 250K tok/GB = 25,000M tokens
Semantic:     ≈ 2.5× that          ≈ 62,500M tokens

At $0.02/1M tokens:
  Fixed-size:  25,000M × 0.02/1M = $500
  Semantic:    62,500M × 0.02/1M = $1,250   (extra ~$750 but recall 60%→75%)
```

Upfront quote for the whole decision: **semantic chunking adds roughly $7.50 per GB of documents** over fixed-size at $0.02/1M. On a 10 GB support corpus that's ~$75 per full re-index — fine for one-off, painful if you re-index daily.

**Latency formula** for retrieval:
`L_total ≈ L_query_encode + L_ann_search + L_rerank + (L_fetch if parent/hierarchical)`
and generation cost/latency also scales with the *chunk size shipped to the LLM* — semantic parents and parent-document parents can be 3–10× the tokens of a small chunk.

## Design Decision 4: Evaluation Before Production

You don't need production pain to pick a strategy. Build a small **holdout evaluation harness** and compare strategies on your real data before deciding.

### The minimum viable harness

1. **Sampling**: take 30–100 real documents; write 1–3 realistic queries each (queries a user would actually type), and label the golden chunk/answer for each query.
2. **Chunk**: apply each candidate strategy (fixed-size@400/50, fixed-size@512/100, semantic, parent-document, hierarchical) to the same corpus.
3. **Retrieve**: run retrieval for each labeled query against each strategy's index.
4. **Score** with RAGAS (or hand-rolled equivalents):

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)

results = evaluate(dataset=holdout, metrics=[
    context_precision, context_recall,
    faithfulness, answer_relevancy,
])
print(results)
```

**Metrics that matter**:
- **Context precision** — did retrieved chunks contain only what's relevant?
- **Context recall** — did retrieval surface the golden chunk at all? This is the direct read on chunking quality (did a boundary split bury the answer?).
- **Faithfulness / answer relevancy** — end-to-end, is the LLM able to answer correctly from the retrieved context?
- **Cost & latency** — measure tokens embedded at ingest and p95 latency per query.

### Read the result as a table, not a shootout

| Strategy | Recall | Ingest cost/GB | p95 latency | Note |
|---|---|---|---|---|
| Fixed 512/100 | 0.58 | $0.50 | 180ms | cheap, boundary splits |
| Semantic | 0.75 | $1.25 | 620ms | best recall, dear |
| Parent-doc | 0.72 | $0.60 | 410ms | recall without the semantic cost |

**Decision rule**: pick the strategy with the *highest recall above your quality floor* at a cost you can sustain — not the raw maximum. A 5× cost to go from 0.70 → 0.75 recall is often the wrong trade for a support-ticket corpus where 0.70 surfaces the right ticket.

## Design Decision 5: Hybrid Approaches

The strongest production answers are hybrids:

- **Fixed-size + semantic**: run semantic chunking only on doc types where boundaries matter (conversation threads), fixed-size elsewhere. Caps ingest cost while getting recall where it counts.
- **Small chunks + parent context** (parent-document retrieval): retrieve small, precise chunks, then fetch the enclosing parent and hand *that* to the LLM. Two-step retrieval + a doc-store mapping is the price.
- **Late chunking on high-value docs only**: spend the specialized-embedding budget where it buys the most.

**The trade-off**: hybrids buy recall/context at the price of *pipeline complexity* — more moving parts, more failure modes, more to operate. Only adopt complexity where the measured gain justifies it.

## Putting It Together: A Decision Framework

**Step 1 — Classify your documents.** Group by the boundary that must be preserved (code/ticket/table/API-section).

**Step 2 — Compute the cost floor.** Estimate fixed-size ingest cost for your volume, then apply the ~2× semantic multiplier to get the semantic budget. If 2× semantics blows the budget, hybrid or parent-doc are your answer regardless of recall.

**Step 3 — Build the holdout and run the harness.** Chunk with each candidate, retrieve against labeled queries, score with RAGAS. This is the step that turns chunking from a gut call into an evidence table.

**Step 4 — Pick against the floor + budget.** Highest recall above your quality floor, within your cost and latency budget.

**Step 5 — Re-evaluate on changes.** Re-run the harness when your document mix or embedding model changes. Chunking is a tuning decision, not a one-time architectural vote.

## The Bottom Line

The problem isn't that semantic chunking is "better" or fixed-size is "worse" — it's that teams lack a *repeatable, pre-production method* to choose. The decision framework is: **classify your document types → model the embedding cost per GB → evaluate each strategy on a labeled holdout with RAGAS → pick the highest-recall option inside your budget.** Run that before you deploy, and the production pain — and the surprise embedding bills — never show up.
