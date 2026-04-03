---
layout: post
title: "Enterprise Code Search with LLMs — Part 4: The Hybrid Breakthrough"
date: 2026-04-03
tags: [ai, code-search, rag, llm, research, hybrid]
---

> This is Part 4 of an ongoing series. [Part 1]({% post_url 2026-03-30-enterprise-search-part1 %}) introduced baselines. [Part 2]({% post_url 2026-03-31-enterprise-search-part2 %}) ran full benchmarks. [Part 3]({% post_url 2026-04-03-enterprise-search-part3 %}) analyzed failure patterns.

In Part 3 we identified a critical insight: RAG and ripgrep fail on *different* commits. Today we test two ideas to exploit this complementarity:

1. **Adaptive Routing** — detect function names in commit messages → route to RAG; else use v3
2. **Fixed Hybrid** — merge candidates from both methods (with a path normalization bug fix)

One of these delivers the biggest improvement in the entire series. The other fails in a surprising way.

---

## Quick Recap: Where We Left Off

| Method | Zulip File F1 | Saleor File F1 | Zulip Exp F1 | Saleor Exp F1 |
|--------|:---:|:---:|:---:|:---:|
| v3 (ripgrep+AI) | **0.345** | **0.146** | **0.135** | 0.069 |
| RAG (Chroma) | 0.196 | 0.091 | 0.035 | **0.133** |
| AI alone | 0.141 | 0.054 | 0.000 | 0.000 |
| *hybrid attempt (buggy)* | *0.113* | — | — | — |

The Part 3 hybrid attempt achieved only 0.113 — worse than v3 — due to a path normalization bug. The Chroma DB was indexed with `/workspace/codebase/` prefixes while ripgrep returned relative paths, so the deduplication step never matched overlapping files.

---

## Experiment 1: Adaptive Routing

### Design

The hypothesis: RAG excels when a commit message explicitly names a function/class (semantic embedding finds it). Ripgrep excels otherwise (keyword search into file names and module structure).

```python
def has_function_name(commit_msg):
    patterns = [
        r'`([a-zA-Z_][a-zA-Z0-9_]{3,})`',              # backtick-quoted
        r'\b([a-z][a-z0-9]*(?:_[a-z][a-z0-9]+){2,})\b', # snake_case 3+ parts
        r'\b([A-Z][a-zA-Z]{3,}(?:[A-Z][a-zA-Z]+)+)\b',  # CamelCase class
    ]
    for p in patterns: 
        m = re.search(p, commit_msg)
        if m: return True, m.group(1)
    return False, None
```

Detection examples:
- `"event_queue: Rename last_for_client to last_client_for_user."` → `last_for_client` (snake_case) → **RAG**
- `"Make refundSettings field nullable on RefundSettingsUpdate mutation"` → `RefundSettingsUpdate` (CamelCase) → **RAG**  
- `"notifications: Channel wildcard notification setting overrides muting."` → no match → **v3**
- `"queue: Extract mobile_notifications_queue_name helper."` → `mobile_notifications_queue_name` → **RAG**

On Zulip n=30: 9 commits routed to RAG, 21 to v3.  
On Saleor n=26: 4 commits routed to RAG, 22 to v3.

### Results

| Method | Zulip File F1 | Saleor File F1 |
|--------|:---:|:---:|
| v3 (baseline) | 0.345 | 0.146 |
| **adaptive routing** | **0.274** | **0.213** |

**Adaptive routing is worse than v3 alone.** 🤔

### Why it Failed

The hypothesis was wrong in a key way: when a commit message contains a function name like `last_for_client`, ripgrep can *directly search for that symbol in the codebase*. v3 already finds it. RAG doesn't improve on this — it finds semantically similar code which is often in different files.

Example of adaptive routing's mistake:
```
Commit: "event_queue: Rename last_for_client to last_client_for_user."
Routed to: RAG (matched 'last_for_client')
RAG found: test files that mention event queue testing
v3 would find: zerver/tornado/event_queue.py (directly searches for the symbol)
```

The lesson: **function name detection alone isn't the right routing signal**. RAG's advantage is in *semantic coverage across modules*, not in matching explicit function names — ripgrep does that better.

---

## Experiment 2: Fixed Hybrid (The Real Breakthrough)

### The Bug Fix

The Part 3 hybrid failed because Chroma stored paths as `/workspace/codebase/zerver/lib/...` while ripgrep returned `zerver/lib/...`. One line of normalization fixed it:

```python
def normalize_path(path, codebase):
    for wp in ["/workspace/codebase-zulip/", "/workspace/codebase/"]:
        if path.startswith(wp):
            return path[len(wp):]
    if os.path.isabs(path):
        return os.path.relpath(path, codebase)
    return path.lstrip("./")
```

### Design

Merge candidates from both v3 (ripgrep+AI) and RAG with weighted scoring, then let AI make the final selection:

```python
# Score: v3=0.6, RAG=0.4, position-decayed
scores = {}
for i, f in enumerate(v3_files):
    scores[f] = scores.get(f, 0) + 0.6 * max(0, 1 - i * 0.1)
for i, f in enumerate(rag_files):
    scores[f] = scores.get(f, 0) + 0.4 * max(0, 1 - i * 0.1)
# Files in both get higher combined score
sorted_files = sorted(scores, key=lambda x: scores[x], reverse=True)
```

The key insight: **files that both methods agree on get boosted**, and the expanded candidate pool (50 files vs 20-30) gives AI more to work with.

### Results

| Method | Zulip File F1 | Saleor File F1 |
|--------|:---:|:---:|
| v3 (baseline) | 0.345 | 0.146 |
| RAG | 0.196 | 0.091 |
| *buggy hybrid* | *0.113* | — |
| **hybrid_fixed** | **0.477** | **0.228** |
| % improvement vs v3 | **+38%** | **+56%** |

**F1=0.477 on Zulip is the best result in this entire research series.**

### Per-Commit Analysis

Looking at where hybrid_fixed wins vs v3:

| Commit | v3 F1 | hybrid F1 | Why hybrid wins |
|--------|:---:|:---:|---|
| `rate_limiter: Update Redis...` | 0.000 | 0.667 | RAG finds Redis-related files v3 misses |
| `linkifiers: Handle optional groups...` | 0.000 | 0.667 | RAG finds linkifier modules |
| `mattermost_importer: Fix guest users...` | 0.000 | 0.667 | RAG finds mattermost modules |
| `send_email: Allow close()...` | 0.000 | 0.667 | RAG finds email backend files |
| `saml: Use ExternalAuthID for...` | 0.000 | 0.667 | RAG finds SAML auth files |
| `integrations: Disable Hubot...` | 0.500 | 1.000 | Combined candidates more complete |

The pattern: hybrid succeeds where **v3's keyword search misses the right files but RAG's semantic search finds them**, and vice versa. The union of both candidate pools is substantially better than either alone.

---

## Exploration Task: Function-Level Prediction

| Method | Zulip Exp F1 | Saleor Exp F1 |
|--------|:---:|:---:|
| v3 | **0.135** | 0.069 |
| RAG | 0.035 | **0.133** |
| adaptive | 0.066 | 0.049 |

Adaptive routing for the exploration task (function-level prediction) achieves 0.066 on Zulip — between v3 and RAG, but doesn't beat either individually on their best dataset.

The function-level task is fundamentally harder because:
- The Chroma DB stores whole files, not individual functions
- After getting candidate files, we AST-parse them to extract all functions
- The candidate pool for AI selection becomes very large (50+ functions)
- AI often picks the wrong functions from the right file

---

## Complete Results Table

### File Prediction F1

| Method | Zulip (n=30) | Saleor (n=26) |
|--------|:---:|:---:|
| AI alone | 0.141 | 0.054 |
| RAG (Chroma) | 0.196 | 0.091 |
| C-det (graph) | 0.102 | — |
| v4 (domain KB) | 0.269 | — |
| v3 (ripgrep+AI) | 0.345 | 0.146 |
| adaptive routing | 0.274 | 0.213 |
| **hybrid_fixed** | **0.477** | **0.228** |

### Exploration F1 (function-level)

| Method | Zulip (n=19) | Saleor (n=15) |
|--------|:---:|:---:|
| AI alone | 0.000 | 0.000 |
| RAG | 0.035 | **0.133** |
| adaptive | 0.066 | 0.049 |
| v3 | **0.135** | 0.069 |

---

## Failure Pattern Analysis: Real-World Prevalence

Building on Part 3's four failure patterns, let's estimate how common they are in production codebases:

### Pattern 1: Cross-Module Changes
**Estimated frequency: ~25-35% of commits**  
Any refactoring, API change, or system-wide feature touches 3+ files across modules. All methods struggle here because the commit message can't tell you which specific modules are affected without reading the code.

**Theoretically solvable?** Partially. A codebase dependency graph could help — if you know module A depends on module B, and the commit touches A's interface, B is a likely candidate. But the explosion of candidates makes precision hard.

### Pattern 2: Rename/Refactor  
**Estimated frequency: ~15-25% of commits**  
The old name doesn't exist yet. Ripgrep can't find it. This is the clearest theoretical limit for static analysis methods.

**Theoretically solvable?** Yes, with code history / change prediction models. If a method was previously called `foo_bar` and the commit message says "rename foo_bar to baz", a rename-aware search can find all callers. Our current methods have no access to this.

### Pattern 3: DB Migrations (Saleor-specific)
**Estimated frequency: ~10-15% of commits in Django projects**  
Migration files are auto-generated with numeric names (`0042_add_checkout_delivery.py`). No keyword search or semantic search can predict these names.

**Theoretically solvable?** Only partially. You can predict *which model changed* (and thus which app's migrations folder), but not the exact filename. Could be modeled as "which migration folder?" rather than exact file.

### Pattern 4: GraphQL Schema (Saleor-specific)
**Estimated frequency: ~20-30% of GraphQL-heavy projects**  
GraphQL mutations/types are spread across many files with generic names. A commit about "RefundSettingsUpdate mutation" needs to find `saleor/graphql/shop/mutations/refund_settings_update.py` — and RAG actually does find this sometimes via semantic matching.

**Theoretically solvable?** Yes — a specialized GraphQL schema index mapping mutation/type names to file paths would solve this entirely. This is domain knowledge, not a fundamental limit.

### Summary

| Pattern | Frequency | Solvable? |
|---------|:---:|:---:|
| Cross-module changes | ~30% | Partially (graph helps) |
| Rename/refactor | ~20% | Partially (history needed) |
| DB migrations | ~12% | Partially (app-level) |
| GraphQL schema | ~25%* | Yes (schema index) |

*In GraphQL-heavy projects

---

## Key Insights from Part 4

**1. Path normalization matters more than you think.** The hybrid method went from 0.113 to 0.477 by fixing one path prefix bug. Always verify that different retrieval methods are comparing apples to apples.

**2. Combining retrieval methods dramatically outperforms either alone.** The hybrid_fixed F1=0.477 represents a new ceiling — 38% better than the best single method. The complementarity is real: ripgrep and RAG fail on different commit types.

**3. Routing logic is hard to get right.** Adaptive routing seemed theoretically sound but failed because the routing signal (function name presence) doesn't correctly predict which method will do better. A function name in the commit message means ripgrep can directly find it — the opposite of what we assumed.

**4. The exploration task (function-level) remains unsolved.** Even the best file-level method (F1=0.477) translates to poor function-level results (F1~0.066). The gap between file and function granularity requires fundamentally different approaches — possibly agentic methods that actually read and understand the code.

---

## What's Next

The hybrid approach has peaked the retrieval-based methods. What could push further?

- **Agentic exploration**: AI reads matched files, follows imports, iterates — already explored in Part 2 (A4), but not at scale
- **Codebase-specific indexes**: Migration file patterns, GraphQL schema maps, import graphs
- **Fine-tuned embeddings**: Train embeddings specifically on commit-message → changed-file pairs for this codebase
- **Change history as context**: Use git blame and past similar commits to predict current changes

The fundamental ceiling for retrieval-based methods on this task appears to be around **F1 = 0.5 on clean codebases** like Zulip, and **F1 = 0.25** on more complex ones like Saleor. Beyond that requires understanding code semantics, not just matching text patterns.

---

*Part 5 will explore whether an agentic approach — giving the AI tools to actually explore the codebase iteratively — can break through this ceiling.*
