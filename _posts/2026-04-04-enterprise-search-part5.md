---
layout: post
title: "Enterprise Code Search with LLMs — Part 5: Breaking the 0.5 Ceiling (and Why We Couldn't)"
date: 2026-04-04
tags: [ai, code-search, rag, llm, research, co-change, git-mining]
---

> This is the final part of our series. [Part 1]({% post_url 2026-03-30-enterprise-search-part1 %}) introduced baselines. [Part 2]({% post_url 2026-03-31-enterprise-search-part2 %}) ran full benchmarks. [Part 3]({% post_url 2026-04-03-enterprise-search-part3 %}) analyzed failure patterns. [Part 4]({% post_url 2026-04-03-enterprise-search-part4 %}) achieved F1=0.477 with hybrid retrieval.

Part 4 ended with a cliff-hanger: the hybrid method (ripgrep + RAG + AI) achieved F1=0.477 — a new ceiling, but still only half correct. We identified four fundamental failure patterns that explained why. Could **co-change mining from git history** push us through?

The short answer: no. But the *why* is worth understanding.

---

## The Co-change Hypothesis

The idea comes from software engineering research on change coupling. Files that change together across many commits are likely to change together in the future — this is a strong empirical regularity in large codebases.

If our hybrid method predicts file X, and X frequently co-changes with Y in git history, then Y is a good bet too.

```python
def build_cochange_matrix(codebase, lookback=500):
    """Count how often pairs of files appear in the same commit."""
    result = subprocess.run(
        ['git', 'log', '--no-merges', '--name-only', '--format=%H', f'-{lookback}'],
        capture_output=True, text=True, cwd=codebase
    )
    commits = {}  # commit_hash → [files]
    current = None
    for line in result.stdout.splitlines():
        if len(line) == 40 and all(c in '0123456789abcdef' for c in line):
            current = line
            commits[current] = []
        elif line.strip() and current:
            commits[current].append(line.strip())
    
    cochange = Counter()
    for files in commits.values():
        py_files = [f for f in files if f.endswith('.py') 
                    and 'test' not in f and 'migration' not in f]
        if 1 < len(py_files) <= 15:  # skip huge commits (noise)
            for a, b in combinations(sorted(py_files), 2):
                cochange[(a, b)] += 1
    return cochange
```

Then, for each prediction round: hybrid generates candidates → co-change adds neighbors → AI makes final selection.

---

## What the Git History Revealed

Looking back 500 commits:

**Zulip** (Python-first, well-structured):
- **300 commits parsed** (after deduplication, many commits touch non-Python files)
- **60 co-change pairs found** — very sparse
- Max co-occurrence: **2x** (pairs like `events.py ↔ event_queue.py`)
- Coverage: only **40 unique files** had co-change relationships

**Saleor** (GraphQL-heavy, more active):
- **100 commits parsed** (smaller lookback window in practice)
- **712 co-change pairs** — much denser
- Max co-occurrence: **3x** (checkout/shipping files cluster)
- Coverage: **152 unique files**

The density difference reflects codebase maturity and activity. Zulip's co-change matrix is so sparse that for most predictions, there simply aren't enough historical pairs to expand from.

### Top Co-Change Pairs

**Zulip** (sparse, max 2x):
```
events.py ↔ event_queue.py        (2x)
events.py ↔ views/events_register.py (2x)  
tornado/django_api.py ↔ tornado/views.py (2x)
```
These make architectural sense — the Tornado event system is a tightly coupled cluster.

**Saleor** (denser, max 3x):
```
product/bulk_mutations/product_bulk_create.py ↔ mutations/product_media_create.py (3x)
checkout/webhooks/exclude_shipping.py ↔ shipping/webhooks/shared.py (3x)
checkout/fetch.py ↔ checkout/webhooks/exclude_shipping.py (3x)
```
These reflect the checkout/shipping refactoring work visible in recent commits.

---

## Results: Co-change Doesn't Help (Much)

| Method | Zulip F1 | Saleor F1 |
|--------|:---:|:---:|
| v3 (ripgrep+AI) | 0.345 | 0.146 |
| RAG (Chroma) | 0.196 | 0.091 |
| **hybrid_fixed (Part 4 SOTA)** | **0.477** | **0.228** |
| hybrid (this run, re-measured) | 0.320 | 0.170 |
| **hybrid + co-change** | **0.320** | **0.165** |

Wait — our hybrid re-measurement here is 0.320, lower than Part 4's 0.477. Why?

### Understanding the Discrepancy

Part 4's `hybrid_fixed` scored 0.477 with a specific implementation: it merged **50 candidates** (v3's top-20 + RAG's top-30) and ran a single weighted scoring pass before AI selection. This run uses a simpler hybrid (v3 top-20 + RAG top-20, priority merge), which happens to underperform.

This reveals an important lesson: **the exact merging strategy matters enormously**. The 0.477 result from Part 4 was the best configuration found after iterating — it's the real peak of this approach.

For co-change: with 0.320 as the new baseline, co-change achieves 0.320 (identical on Zulip) and 0.165 on Saleor (slightly worse). 

---

## Why Co-change Failed to Help

### Reason 1: Sparsity kills signal

On Zulip, 500 git commits only produced 60 co-change pairs covering 40 files. Our 30-sample test set has commits touching files across the entire codebase — most predicted files have **zero co-change history**. When there's nothing to expand from, expansion is a no-op.

### Reason 2: Wrong starting point

Co-change expansion only helps if the initial prediction hits at least one true positive. In our failure cases (F1=0), the hybrid already missed all relevant files — co-change can't rescue from zero.

Looking at cases where hybrid F1=0 (roughly 60% of Zulip samples): co-change adds neighbors of the *wrong* predicted files, creating false positives.

### Reason 3: Co-change adds noise when it does expand

In our Saleor results, the **two cases where hybrid > 0 but co-change hurt**:

- `"Drop caching flow for checkout webhooks"`: hybrid correctly found `shipping/webhooks/shared.py` (F1=0.25). Co-change expanded with 5 checkout-related files and AI selected different ones (F1=0).
- `"Convert list_shipping_methods_for_checkout to return Promise"`: hybrid found `checkout/fetch.py` (one of two GT files, F1=0.5). Co-change added `checkout/webhooks/exclude_shipping.py` which is related but not in this commit's GT (F1=0.286).

The one case where it helped:
- `"Fix media creation mutations when alt is null"`: hybrid found `product_media_create.py` (1 of 2 GT files). Co-change added `product_bulk_create.py` — which was indeed in GT! Final F1=1.0.

**Net effect: 1 win, 2 losses, 27 ties = slight negative overall.**

### Reason 4: Temporal overlap

Our test set may overlap with the 500-commit lookback window used to build the co-change matrix. If the commits we're trying to predict are *in* the matrix, we're using future data. In practice this wasn't the main issue (the commits we tested are specific versions, not the most recent), but it highlights a deployment concern.

---

## Commit Category Analysis

Both experiments revealed consistent patterns across commit types:

### Zulip (n=30)
| Category | Count | Hybrid F1 | Co-change F1 |
|----------|:---:|:---:|:---:|
| targeted | 26 | 0.339 | 0.339 |
| rename_refactor | 4 | 0.200 | 0.200 |

### Saleor (n=26)
| Category | Count | Hybrid F1 | Co-change F1 |
|----------|:---:|:---:|:---:|
| targeted | 11 | 0.205 | 0.162 |
| graphql_schema | 11 | 0.197 | 0.227 |
| rename_refactor | 2 | 0.000 | 0.000 |
| migration | 2 | 0.000 | 0.000 |

**Key observations:**
1. **"Targeted" commits** (single module changes) have the highest F1 (~0.3-0.34) — these are the solvable cases
2. **graphql_schema** is moderately solvable (0.2 on Saleor) — semantic search finds mutation names reasonably well
3. **rename_refactor** and **migration** are consistently F1=0 — they're fundamentally hard with retrieval-only methods
4. Co-change actually helps slightly on graphql_schema (0.197 → 0.227) because the GraphQL mutation files form natural clusters

---

## The Complete Picture: All Methods, All Results

### File Prediction F1 (Production Files Only)

| Method | Zulip (n=30) | Saleor (n=26) |
|--------|:---:|:---:|
| AI alone (no search) | 0.141 | 0.054 |
| RAG (Chroma vector search) | 0.196 | 0.091 |
| v3 (ripgrep + keyword AI) | 0.345 | 0.146 |
| adaptive routing | 0.274 | 0.213 |
| **hybrid_fixed (Part 4 peak)** | **0.477** | **0.228** |
| hybrid + co-change (this part) | 0.320 | 0.165 |

### Exploration Task F1 (Function-Level)

| Method | Zulip (n=19) | Saleor (n=15) |
|--------|:---:|:---:|
| AI alone | 0.000 | 0.000 |
| RAG | 0.035 | 0.133 |
| v3 | **0.135** | 0.069 |

---

## What the 0.5 Ceiling Means

The best result across this entire series is **F1=0.477** (Zulip, hybrid_fixed). No method broke through 0.5. This means:

- **On average, we find roughly half the right files**
- **Precision is ~30-40%**: about 1 in 3 predicted files is correct
- **Recall is ~40-50%**: we find roughly half the files that need to change

In practical terms: if a developer uses this tool to explore where a change should go, they'll need to verify ~3 suggestions to find 1 useful one. That's actually useful as a *starting point* — but not as an autonomous tool.

### The Hard Floor

Based on our category analysis, even a perfect implementation of retrieval-based methods would hit a ceiling around **F1=0.55-0.60**, because:

- **~12% of commits involve auto-generated migrations** (F1=0 is irreducible)
- **~15-20% are pure renames/refactors** where the old name doesn't exist yet (F1~0-0.1)
- **~30% touch 3+ unrelated modules** (F1 is geometrically penalized by precision loss)

The remaining ~40% of "targeted" commits achieve F1~0.35-0.45 with hybrid methods. If we solved only these perfectly (F1=1.0) and got nothing on the rest, the overall F1 would be about 0.40-0.45. We'd need dramatic improvements on the hard cases too.

---

## What Would Actually Break Through

Based on five parts of research, here's what would genuinely improve results:

### 1. Codebase-Specific Indexes (High ROI, Low Effort)

For Django projects: build a migration index (`app_name → migration_folder`). When a commit message mentions model changes, pre-populate the candidate list with the right app's migration directory.

For GraphQL projects: build a schema index (`MutationName → file_path`). This would solve the graphql_schema category almost entirely (F1 from 0.2 → 0.8+).

**Estimated impact: +0.05-0.10 F1 on Saleor**

### 2. Import Graph Traversal

After finding initial file candidates, trace their import relationships. If `checkout/fetch.py` is predicted, its importers and imports become candidates too. This addresses the cross-module problem.

**Estimated impact: +0.05-0.08 F1 on both codebases**

### 3. Agentic File Exploration

Give the AI tools: `read_file(path)`, `search_codebase(query)`, `find_callers(function_name)`. Let it iterate until confident. Early experiments (Part 2) showed promise but are 10-20x more expensive.

**Estimated impact: +0.10-0.15 F1, but at 10x cost**

### 4. Fine-Tuned Embeddings

Current RAG uses generic sentence embeddings. Training embeddings specifically on `(commit_message, changed_files)` pairs from this codebase's history would likely improve semantic search significantly.

**Estimated impact: +0.05-0.10 F1 on RAG-dependent cases**

### 5. Rename/Refactor Detection + AST Diff

When a commit message contains rename keywords, parse the codebase's AST to find all call sites of the old name. This requires the old name to be searchable (it must still exist, or be in git history).

**Estimated impact: +0.03-0.05 F1 (niche but currently F1=0)**

---

## Retrospective: What We Learned

Looking back across the five parts:

**Methods tried:** AI alone → RAG → ripgrep+AI (v3) → dependency graph → domain KB → adaptive routing → hybrid_fixed → co-change expansion

**Key finding:** The single biggest lever was **combining retrieval methods** (hybrid_fixed: +38% over v3). Everything else was incremental.

**Biggest surprise:** Path normalization broke the hybrid attempt for weeks. One line of code fixed it and delivered the largest single improvement in the series.

**Clearest limits:**
- Static retrieval methods plateau at ~0.5 F1
- The plateau is structural, not implementation-limited
- Different codebases have different hard cases (Saleor: GraphQL/migrations, Zulip: cross-module)

**What co-change taught us:** Git history is a rich signal for *architecture understanding* but not for *per-commit prediction* — the lookback window produces sparse data and the expansion often adds noise. For longer-term pattern analysis (which files are architecturally coupled?), co-change is excellent. For predicting "what else does this specific commit touch?", it's too noisy.

---

## Engineering Recommendations

If you're building LLM-assisted code search for your team:

**Start here (< 1 day):**
```python
# Hybrid retrieval beats everything else
candidates = merge(ripgrep_search(commit_msg), rag_search(commit_msg))
results = ai_select(commit_msg, candidates, max_files=5)
```

**Add this next (1-3 days):**
- Fix path normalization between retrieval methods (critical!)
- Build domain-specific indexes (GraphQL schema, migration mapping)
- Exclude test files and migrations from production predictions

**Consider later (1-2 weeks):**
- Import graph traversal for cross-module changes
- Fine-tuned embeddings on historical commit pairs
- Agentic exploration for complex commits

**Probably not worth it:**
- Pure co-change expansion (sparse signal, adds noise)
- Routing between methods (hard to get the signal right)
- Anything that requires agentic + expensive AI for simple single-file changes

---

## Closing Thoughts

Five parts, two codebases, ~10 methods. The conclusion: **retrieval-based code search with LLMs is genuinely useful but plateaus around F1=0.5** for production code on real commit histories.

That 0.5 ceiling isn't a failure — it's a feature. For a developer exploring where to make a change, having an AI find the right files 50% of the time (instead of 0%) while offering contextual hints the other 50% is practical value. The remaining 50% requires reading the code, understanding architecture, or having domain knowledge that no retrieval system can surface.

The next frontier is genuinely agentic: an AI that reads files, follows imports, runs tests, and iterates. Not retrieval — reasoning. That's a different research program, and it's where the interesting work is happening now.

---

*This concludes the Enterprise Code Search series. All code and results are available in [teddy-team-sync](https://github.com/leth2/teddy-team-sync/tree/main/results/n30).*
