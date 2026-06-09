# Free Tier Cost Management — Mergit

> Target: **< $0.50/org/month** · Without optimization: ~$27/org/month · Required reduction: **98%**

---

## Model Pricing (verified June 2026)

| Model | Input $/M | Output $/M | Use for |
|-------|----------|-----------|---------|
| groq/llama-4-maverick | $0.20 | $0.60 | Free tier primary (Researcher/Writer) |
| groq/llama-3.1-8b | $0.05 | $0.08 | Summarization only |
| claude-haiku-4-5 | $0.80 | $4.00 | Free tier Coder role |
| claude-sonnet-4-6 | $3.00 | $15.00 | Startup tier |
| claude-opus-4-8 | $15.00 | $75.00 | Enterprise only |

**Anthropic prompt cache:** read = **0.1×** · write 5-min = 1.25× · write 1-hour = 2×
**Anthropic Batch API:** **50% off** all tokens (free tier always uses this)
**Groq:** no prompt caching support

---

## Tier Limits

| | Free | Startup | Enterprise |
|--|------|---------|-----------|
| Goals/day | 20 | 500 | Unlimited |
| Input tokens/request | 4,096 | 32,768 | 200,000 |
| Output tokens/request | 512 | 2,048 | 8,192 |
| RPM | 5 | 60 | 500+ |
| Models | Groq + Haiku | + Sonnet | All |
| Cache namespace | Shared | Private + shared | Dedicated |
| Batch queue | Forced | Optional | Optional |

---

## 5-Layer Cost Stack

```
Layer 0  Anthropic prefix cache   cache_control: ephemeral on system prompt
         All free orgs share one prefix → 0.1× input cost after first write

Layer 1  Model routing by role     Groq for Researcher/Writer, Haiku for Coder
         Groq is ~93% cheaper than Sonnet

Layer 2  Token budget ledger       Redis counter (pre-flight) + weighted post-call accounting
         Billable = cache_read×0.1 + cache_write×1.25 + regular×1 + output×5

Layer 3  Shared semantic cache     cosine > 0.92 across ALL free orgs (cross-org pool)
         Org A's answer serves Org B → pay once, serve many

Layer 4  Context compression       Cap at 4,096 tokens · summarize old msgs with llama-3.1-8b
         Compression saves ~40% input tokens per request

Layer 5  Batch API                 Queue free tier → process every 5 min via Anthropic Batch
         50% off all token prices automatically
```

---

## Claude.ai-Style Free Tier Pattern (4 States)

```python
class FreeTierState(Enum):
    NORMAL    = "normal"     # < 80% used  → full response, show usage bar
    WARNING   = "warning"    # 80-99% used → response + upgrade banner
    LIMIT_HIT = "limit_hit"  # 100% used   → block + upgrade prompt + reset timer
    CAPACITY  = "capacity"   # server issue → queue + ETA (never blame the user)
```

**Rules:**
- Never show raw `429` — always return friendly JSON with `resets_at` + upgrade CTA
- `CAPACITY` ≠ `LIMIT_HIT` — server overload shows retry, not paywall
- Show goals (20/day), not raw tokens — users understand goals, not token counts
- Max 1 upgrade prompt per org per day — don't nag
- Upgrade prompt triggers: `limit_hit`, `80pct_used`, `github_pr_attempted`, `5_days_active`

---

## Per-Role Model Routing

| Role | Free | Startup | Enterprise |
|------|------|---------|-----------|
| Researcher | groq/llama-4-maverick | groq/llama-4-maverick | groq/llama-4-maverick |
| Writer | groq/llama-3.3-70b | groq/llama-3.3-70b | claude-sonnet-4-6 |
| Coder | claude-haiku-4-5 | claude-haiku-4-5 | claude-sonnet-4-6 |
| Integrator | groq/llama-3.3-70b | claude-haiku-4-5 | claude-sonnet-4-6 |
| Summarizer | groq/llama-3.1-8b | groq/llama-3.1-8b | groq/llama-3.1-8b |

---

## Request Flow (Free Tier)

```
Request
  │
  ├─ FreeTierState check → LIMIT_HIT? return upgrade prompt immediately
  ├─ Global capacity check → AT_CAPACITY? queue + return ETA
  ├─ Shared semantic cache (cosine > 0.92) → HIT? return, 0 tokens
  ├─ Token budget pre-flight (Redis) → over limit? return LIMIT_HIT
  ├─ Context compression → cap 4,096 tokens, summarize history
  ├─ Model routing → Groq (Researcher/Writer) or Haiku (Coder)
  ├─ Batch queue → 5-min delay → Anthropic Batch API (50% off)
  ├─ Anthropic shared prefix cache → cache_control on system prompt
  └─ LLM call (max_tokens: 512) → store in shared semantic cache → return
```

---

## Key Redis Keys

```
budget:day:{org_id}:{date}          token counter  TTL: 48h
goals:day:{org_id}:{date}           goal counter   TTL: 48h
org:tier:{org_id}                   tier string    TTL: 5m
semcache:shared:{sha256(query)}     response JSON  TTL: 24h
emb:text-embedding-3-small:1536:{sha256(text)}     TTL: none
batch:queue:{window}                request list   TTL: 15m
batch:result:{request_id}           response JSON  TTL: 1h
upgrade_shown:{org_id}:{date}       "1"            TTL: 24h
```

---

## Shared Prompt Cache (biggest free-tier win)

```python
# One static prefix shared across ALL free orgs
# First org: pays 1.25× (cache write)
# Every other org: pays 0.1× (cache read) — 90% cheaper

system = [
    {
        "type": "text",
        "text": FREE_TIER_SYSTEM_PROMPT,        # ≥ 1024 tokens required
        "cache_control": {"type": "ephemeral"}  # Anthropic caches this
    },
    {
        "type": "text",
        "text": f"Org: {org_name}\nGoal: {user_goal}"  # dynamic, not cached
    }
]
```

**Never put dynamic content above the breakpoint** — it busts the cache every call.

---

## Cost Reality (10 free orgs, 500k tokens/month each)

| Step | Cost |
|------|------|
| Baseline (Sonnet, no optimization) | $27.00 |
| After Groq routing (80% of calls) | $5.40 |
| After Haiku for remaining 20% | $1.46 |
| After shared semantic cache (60% hit) | $0.58 |
| After shared prefix cache (90% off input) | $0.34 |
| After Batch API (50% off remainder) | $0.22 |
| After context compression (40% reduction) | **$0.13** |
| **Per org/month** | **$0.01 – $0.04** |

---

## The Economics

| Tier | Your cost | You charge | Margin |
|------|-----------|-----------|--------|
| Free | $0.04–0.40/org/mo | $0 | –$0.40 acquisition cost |
| Startup | $2–8/org/mo | $29–199/mo | **$21–197/mo** |
| Enterprise | $50–500/org/mo | $500–5000/mo | **$450–4500/mo** |

**1 in 50 free orgs upgrading to Startup pays for all 50 free orgs.**
The caching stack + Claude.ai UX pattern is what makes that conversion rate possible.

---

*June 2026 · Prices subject to change — verify at provider pricing pages before budget commitments.*
