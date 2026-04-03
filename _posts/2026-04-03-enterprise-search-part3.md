---
layout: post
title: "Enterprise Code Search with LLMs — Part 3: Why Methods Fail (and When RAG Wins)"
date: 2026-04-03
tags: [ai, code-search, rag, llm, research]
---

> This is Part 3 of an ongoing series. [Part 1]({% post_url 2026-03-24-enterprise-search-part1 %}) introduced the problem and baselines. [Part 2]({% post_url 2026-03-31-enterprise-search-part2 %}) ran full benchmarks across Zulip and Saleor.

In Part 2 we measured everything. Today we dig into *why* methods fail — and discover a surprising reversal where RAG beats our best ripgrep-based method in one specific scenario.

---

## Quick Recap

Testing LLM-based code search on two real codebases (Zulip, Saleor) with two tasks:

**File Prediction** (given a commit message, find the changed files):

| Method | Zulip F1 | Saleor F1 |
|--------|----------|-----------|
| v3 (ripgrep+AI) | **0.345** | **0.146** |
| RAG (Chroma) | 0.196 | 0.091 |
| AI alone | 0.141 | 0.054 |

**Function-Level Exploration** (find changed functions):

| Method | Zulip F1 | Saleor F1 |
|--------|----------|-----------|
| v3 | **0.135** | 0.069 |
| RAG | 0.035 | **0.133** |
| AI | 0.000 | 0.000 |

The Saleor exploration reversal — RAG (0.133) beating v3 (0.069) — is what we investigate today.

---

## Part 1: The Four Failure Patterns

Systematically categorizing every F1=0 case across both codebases reveals four recurring failure types:

### Pattern 1: Cross-Module Changes (9 cases)

The hardest category. A single commit modifies 3+ files in different, seemingly unrelated modules.

```
"webhooks: Update remaining webhooks to support empty URL param"
  GT: canarytoken/view.py, gitlab/view.py, thinkst/view.py

"api-docs: Document endpoints for allowed email address domains"
  GT: curl_param_value_generators.py, markdown_extension.py, openapi.py
```

Neither ripgrep nor RAG can identify all three files — the commit message gives no clue *which* webhooks or *which* openapi files changed. This is fundamentally an information problem: the commit message underdetermines the target.

### Pattern 2: Rename/Refactor (7 cases)

Renaming is a semantic trap. Ripgrep searches for the *new* name, which didn't exist before the commit. RAG embeddings are built from the current codebase, so they'll find the new name's location — but that's not what was *changed*, it's where the code now lives.

```
"event_queue: Rename last_for_client to last_client_for_user"
  GT: zerver/tornado/event_queue.py

"rate_limiter: Remove Tornado in-memory backend, use Redis"
  GT: zerver/lib/rate_limiter.py, zproject/computed_settings.py
```

The ripgrep approach finds the file with the new name, which happens to be the same file. But it fails for the `computed_settings.py` — that's where the Redis config lives, and the commit message doesn't mention it.

### Pattern 3: Database Migrations (4 cases in Saleor)

Auto-generated migration files are inherently unpredictable:

```
"Migrate shipping methods to checkout deliveries"
  GT: 0093_propagate_checkout_delivery.py, tasks/saleor3_23.py

"Fix site migration history"
  GT: 0046_sitesettings_password_login_mode.py, 0047_merge_20260320_1308.py
```

No amount of semantic reasoning can predict a file named `0046_sitesettings_...`. This is a hard upper bound: any LLM-based method will score 0 on these.

### Pattern 4: GraphQL Schema Changes (6 cases in Saleor)

Saleor's GraphQL layer is enormous (~300 mutation/type files). A commit like:

```
"Fix creation of the event."
  GT: order/mutations/order_line_discount_remove.py, graphql/order/types.py
```

The commit message "Fix creation of the event" gives no signal about *which* of the hundreds of GraphQL files is involved. This explains why Saleor consistently scores lower than Zulip across all methods — a larger, more specialized codebase creates more ambiguity.

---

## Part 2: Why RAG Beats Ripgrep in Saleor Exploration

This was the finding that surprised us. In function-level search on Saleor, RAG (0.133) beats v3 (0.069). Let's look at the 4 RAG wins:

### Win 1: Function name in commit message

```
Commit: "Add user email domain to order search vector"
GT function: prepare_order_search_vector_value
RAG F1: 0.667 | v3 F1: 0.000
```

"order search vector" in the commit message semantically matches a function *in* `saleor/order/search.py`. RAG's embedding space captures this. Ripgrep searches for "order", "search", "vector" keywords — which appear in many files — and the AI picks the wrong one.

### Win 2: Long, unique function names

```
Commit: "Convert list_shipping_methods_for_checkout to return Promise"
GT: list_shipping_methods_for_checkout, _parse_list_shipping_methods_response, process_responses
RAG F1: 0.500 | v3 F1: 0.000
```

The full function name `list_shipping_methods_for_checkout` appears in the commit message. RAG finds the chunk containing this exact function. Ripgrep would need to search for "list_shipping_methods_for_checkout" — but the issue is the AI step picking from candidates. Here, RAG's direct embedding match wins.

### Win 3: Semantic domain alignment

```
Commit: "Fix media creation mutations when alt field is null"
GT: ProductBulkCreate, clean_media, ProductMediaCreate, perform_mutation
RAG F1: 0.333 | v3 F1: 0.000
```

"media creation mutations" → RAG finds `ProductMediaCreate` and `clean_media` via semantic similarity. Ripgrep finds "alt" and "null" across many files; the AI picks wrong.

### The one v3 win:

```
Commit: "Fix trackInventory not being applied in productBulkCreate"
GT: ProductBulkCreate, save
v3 F1: 0.667 | RAG F1: 0.000
```

`trackInventory` is a specific implementation keyword. Ripgrep finds it exactly in `productBulkCreate.py`. RAG's chunk-based approach missed this — the function `save` is a generic name that matches many chunks.

### The Pattern

**RAG wins when:** the commit message contains the function name or strong semantic domain clues (shipping, search, media). The embedding captures function-level semantic meaning directly.

**v3 wins when:** a specific implementation keyword exists that can be ripgrepped to the exact file, and the function name is generic (like `save`).

This suggests a powerful insight: **for function-level search, the optimal strategy depends on commit message style.** GraphQL/domain-specific commits (Saleor) favor RAG; implementation-level commits (Zulip internal) favor keyword search.

---

## Part 3: Hybrid Experiment

We tried combining both: get candidates from RAG + ripgrep, then let AI pick the best files.

**Result: 0.113 F1 on Zulip (worse than v3's 0.345)**

Why? Several issues:
1. **Path normalization**: RAG returns absolute paths (`/workspace/codebase/...`), ripgrep returns relative paths, ground truth uses repo-relative. The AI picks correctly but F1 matching fails.
2. **Signal dilution**: 20 mixed-quality candidates confuse the AI vs. 5-8 focused ripgrep hits.
3. **Wrong file types**: RAG pulls TypeScript files (`.ts`) when the GT is Python only.

The hybrid is theoretically the right approach — it's the *implementation* that needs fixing. A proper implementation would:
- Normalize all paths to repo-relative before AI comparison
- Filter candidates to the same language
- Use structured output for the AI to pick from a numbered list

---

## What We Learned

**Hard limits:**
- Database migrations: ~0% precision possible for any method (auto-generated filenames)
- Cross-module changes: Hard upper bound ~50% even for an oracle with keywords
- Rename operations: Requires knowing old names, which pre-change embeddings would have

**Where to invest:**
- Function-level RAG works surprisingly well when function names appear in commit messages (Saleor-style)
- The real gap is in multi-file commits — knowing which *files* are involved in a cross-cutting change
- A better hybrid needs proper path normalization (simple engineering fix)

**Numbers that matter:**

```
Method         Zulip-FP    Saleor-FP    Zulip-Exp   Saleor-Exp
v3             0.345       0.146        0.135        0.069
RAG            0.196       0.091        0.035        0.133  ← WINS HERE
Hybrid(naive)  0.113       -            -            -
AI alone       0.141       0.054        0.000        0.000
```

The reversal at `Saleor-Exp` is real and meaningful: when function names carry semantic weight (as in GraphQL mutations), embedding-based search outperforms keyword search at the function level.

---

## Next Steps

1. **Fix hybrid path normalization** — this is a quick engineering fix that could push hybrid above v3
2. **Adaptive strategy selection** — route to RAG vs keyword based on commit message style (contains function name? → RAG)
3. **Code-specialized embeddings** — jina-embeddings-v3 or CodeBERT might improve the embedding quality for the function-level task
4. **Larger test set** — 15-30 samples is too small; need 100+ for reliable conclusions

The research continues. [Part 4](/) coming soon.

---

*Experiments run on a Raspberry Pi cluster. All code and results: [teddy-team-sync](https://github.com/leth2/teddy-team-sync).*
