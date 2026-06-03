# Free Tier Cost Management — Production Implementation Guide

> **Internal reference for Mergit engineering.**
> How to offer free services to companies and organisations while controlling
> LLM token costs at production scale. Covers model selection, budget ledgers,
> caching strategy, context compression, batch pricing, and Claude.ai-style
> free tier UX pattern.
>
> Last updated: June 2026 · Prices verified against official provider documentation.

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [Model Pricing Reference](#2-model-pricing-reference)
3. [Tier Definitions & Limits](#3-tier-definitions--limits)
4. [Claude.ai-Style Free Tier Pattern](#4-claudeai-style-free-tier-pattern)
5. [5-Layer Cost Defense Stack](#5-5-layer-cost-defense-stack)
   - [Layer 1: Model Routing by Tier](#layer-1-model-routing-by-tier)
   - [Layer 2: Token Budget Ledger](#layer-2-token-budget-ledger)
   - [Layer 3: Cache Maximization](#layer-3-cache-maximization)
   - [Layer 4: Context Compression](#layer-4-context-compression)
   - [Layer 5: Batch API Discount](#layer-5-batch-api-discount)
6. [Full Request Flow](#6-full-request-flow)
7. [Cost Calculations with Real Numbers](#7-cost-calculations-with-real-numbers)
8. [Database Schema](#8-database-schema)
9. [Implementation Code](#9-implementation-code)
10. [Monitoring Dashboard Spec](#10-monitoring-dashboard-spec)
11. [Scaling Thresholds](#11-scaling-thresholds)

---

## 1. The Problem

```
You offer free service to orgs
          ↓
Org A uses 10M tokens/month
Org B uses 50M tokens/month
          ↓
You pay Anthropic the full bill
          ↓
You go broke before the hackathon ends
```

The solution is a **5-layer cost defense stack** that makes free tiers
economically viable by routing cheap, caching aggressively, and compressing hard.

**Target economics:**
- Cost per free org: **< $0.50 / month**
- Without optimization: ~$24 / month per org
- Required reduction: **~98% cost elimination**

---

## 2. Model Pricing Reference

> Always verify at provider pricing pages before committing to budgets.
> Figures from official documentation as of May–June 2026.

### 2.1 Anthropic Claude Models

| Model | Input ($/M tokens) | Output ($/M tokens) | Best for |
|-------|-------------------|--------------------|---------:|
| **claude-opus-4-8** | $15.00 | $75.00 | Enterprise, complex reasoning |
| **claude-sonnet-4-6** | $3.00 | $15.00 | Startup / paid tier default |
| **claude-haiku-4-5** | $0.80 | $4.00 | Free tier last resort |

### 2.2 Anthropic Prompt Caching Multipliers

| Cache operation | Multiplier | Effective price (Haiku) | TTL |
|----------------|-----------|------------------------|-----|
| Cache read (hit) | **0.1×** | **$0.08/M** | Until expiry |
| Cache write (5-min TTL) | 1.25× | $1.00/M | 5 minutes |
| Cache write (1-hour TTL) | 2.00× | $1.60/M | 1 hour |
| Regular input (no cache) | 1.00× | $0.80/M | N/A |

> Break-even = 1.1 hits per write. Almost always worth it for any repeated prefix.

### 2.3 Anthropic Batch API Discount

| Scenario | Price vs real-time |
|----------|--------------------|
| Batch API (async, 24h window) | **50% off** all input + output |
| Real-time API | 1× baseline |

Batch pricing (Haiku): Input $0.40/M · Output $2.00/M · Cache read $0.04/M

### 2.4 Groq Models (Near-Zero Cost)

| Model | Input ($/M) | Output ($/M) | Speed | Notes |
|-------|------------|-------------|-------|-------|
| **llama-4-maverick** | ~$0.20 | ~$0.60 | 🔥 Very fast | Primary free tier |
| **llama-3.3-70b** | ~$0.59 | ~$0.79 | 🔥 Fast | Free tier fallback |
| **llama-3.1-8b** | ~$0.05 | ~$0.08 | ⚡ Fastest | Summarization only |

> Groq does **NOT** support prompt caching (June 2026). No `cache_control` field.

### 2.5 Cost Per 1000 Requests (Benchmark)

Assuming avg 2,000 input + 500 output tokens per request:

| Model | Cost/1000 req | With caching | With batch |
|-------|-------------|-------------|-----------|
| Groq llama-4-maverick | **$1.70** | N/A | N/A |
| Claude Haiku 4.5 | $9.60 | **$3.20** (67% off) | **$1.60** (83% off) |
| Claude Sonnet 4.6 | $36.00 | **$10.80** (70% off) | **$5.40** (85% off) |
| Claude Opus 4.8 | $187.50 | **$56.25** (70% off) | **$28.13** (85% off) |

---

## 3. Tier Definitions & Limits

### Free Tier Limits

| Parameter | Limit | Reason |
|-----------|-------|--------|
| Messages per day | 20 | Claude.ai-style intuitive limit |
| Messages per month | 300 | Soft monthly cap |
| Tokens per message (input) | 4,096 | Prevents context abuse |
| Tokens per response (output) | 512 | Output costs 5× input |
| Requests per minute | 5 | Prevents burst abuse |
| Model access | Groq / Haiku only | No Sonnet, no Opus |
| Prompt caching | Shared pool | Cannot use private namespace |
| Semantic cache | Shared cross-org | Benefits from others' answers |
| Batch processing | Forced (5-min delay) | 50% discount applied automatically |
| Context history | Last 10 messages | Older compressed to summary |

### Startup Tier ($29–199/mo)

| Parameter | Limit |
|-----------|-------|
| Messages per day | 500 |
| Max input per message | 32,768 tokens |
| Max output per message | 2,048 tokens |
| Requests per minute | 60 |
| Model access | Groq / Haiku / Sonnet |
| Caching | Private namespace + shared pool |

### Enterprise (Custom)

| Parameter | Limit |
|-----------|-------|
| Messages | Unlimited (billed actuals) |
| Max input | 200,000 tokens |
| Max output | 8,192 tokens |
| Requests per minute | 500+ |
| Model access | All models including Opus |
| Caching | Dedicated private namespace |
| SLA | 99.9% uptime |

---

## 4. Claude.ai-Style Free Tier Pattern

Claude.ai's free tier is the gold standard for graceful degradation without
hard cutoffs. Here is exactly how they do it and how to replicate it.

### 4.1 What Claude.ai Does (observed behaviour)

```
User sends message
        ↓
If usage < 80% of daily limit:
    → Full response, no warning

If usage 80-99% of daily limit:
    → Response given + soft warning banner
    "You're approaching your daily limit. X messages remaining."

If usage hits 100%:
    → Friendly block page (not a 429 error)
    → Shows upgrade prompt with clear value prop
    → Shows exact reset time
    → Option to "Get notified when reset"

If servers are at capacity (not user limit):
    → "Claude is at capacity right now"
    → No error code shown to user
    → Retry suggestion with timer
```

**Key principle:** The user always feels respected, never punished.
They see remaining capacity, not just a wall.

### 4.2 Implementing the Claude.ai Pattern for Mergit

#### Message-Based Limits (not token-based, for UX clarity)

```python
# Instead of showing "You have 47,823 tokens left"
# Show "You have 7 goals remaining today"
# Internally still track tokens; display as messages

DISPLAY_LIMITS = {
    "free": {
        "daily_goals":    20,      # "goals" = user-facing term for requests
        "monthly_goals":  300,
        "reset_cycle":    "daily at midnight UTC"
    }
}

def tokens_to_goals(tokens_used: int, tier: str) -> int:
    """Convert internal token budget to user-facing goal count."""
    avg_tokens_per_goal = 2500  # calibrate from real usage data
    return tokens_used // avg_tokens_per_goal
```

#### Four-State Response System

```python
# responses/free_tier_states.py

from enum import Enum

class FreeTierState(Enum):
    NORMAL    = "normal"       # < 80% used — full service
    WARNING   = "warning"      # 80–99% used — warn + serve
    LIMIT_HIT = "limit_hit"    # 100% used — block with upgrade prompt
    CAPACITY  = "capacity"     # server-side issue — retry prompt

class FreeTierResponseBuilder:

    async def get_state(self, org_id: str) -> FreeTierState:
        used  = await self._get_goals_used_today(org_id)
        limit = DISPLAY_LIMITS["free"]["daily_goals"]
        pct   = used / limit * 100

        if pct < 80:   return FreeTierState.NORMAL
        if pct < 100:  return FreeTierState.WARNING
        return FreeTierState.LIMIT_HIT

    def build_response(self, state: FreeTierState,
                       org_id: str, content: str | None,
                       goals_used: int) -> dict:

        limit = DISPLAY_LIMITS["free"]["daily_goals"]
        remaining = max(0, limit - goals_used)
        reset_at = self._next_midnight_utc()

        if state == FreeTierState.NORMAL:
            return {
                "content":   content,
                "status":    "ok",
                "usage": {
                    "goals_used":      goals_used,
                    "goals_remaining": remaining,
                    "resets_at":       reset_at,
                }
            }

        if state == FreeTierState.WARNING:
            return {
                "content": content,
                "status":  "ok",
                "usage": {
                    "goals_used":      goals_used,
                    "goals_remaining": remaining,
                    "resets_at":       reset_at,
                },
                "banner": {                                  # show in frontend
                    "type":    "warning",
                    "message": f"You have {remaining} goals remaining today.",
                    "subtext": f"Resets at midnight UTC ({reset_at}).",
                    "cta": {
                        "text": "Upgrade for unlimited goals →",
                        "url":  "https://mergit.io/pricing"
                    }
                }
            }

        if state == FreeTierState.LIMIT_HIT:
            return {
                "content": None,
                "status":  "limit_reached",
                "error": {
                    "code":    "daily_goal_limit_reached",
                    # ← No raw "429" shown to user
                    "title":   "You've reached your daily goal limit",
                    "message": "Free plan includes 20 goals per day.",
                    "resets_at": reset_at,
                    "resets_in": self._human_readable_until(reset_at),
                    "upgrade": {
                        "title":   "Get 500 goals/day with Startup",
                        "price":   "$29/month",
                        "url":     "https://mergit.io/pricing",
                        "features": [
                            "500 goals/day (25× more)",
                            "Access to Claude Sonnet",
                            "Private cache namespace",
                            "Priority processing (no batch queue)",
                            "GitHub PR creation enabled"
                        ]
                    },
                    "notify": {
                        "text": "Notify me when my limit resets",
                        "url":  f"https://mergit.io/notify/{org_id}"
                    }
                }
            }

        # FreeTierState.CAPACITY — server issue, not user fault
        return {
            "content": None,
            "status":  "at_capacity",
            "error": {
                "code":    "service_at_capacity",
                "title":   "Mergit is busy right now",
                "message": "We're handling high demand. Your goal will be "
                           "processed shortly.",
                "retry_after_seconds": 30,
                "status_url": "https://status.mergit.io"
                # No mention of pricing — this is not the user's fault
            }
        }
```

#### Upgrade Prompt Timing (Claude.ai Pattern)

```python
# Show upgrade prompts at psychologically optimal moments:

UPGRADE_PROMPT_TRIGGERS = [
    # When limit is hit — obvious
    "limit_reached",

    # When approaching limit with meaningful usage
    # (user has proven value before hitting wall)
    "80pct_daily_used",

    # After 5 consecutive days using free tier
    # (habitual user → high conversion probability)
    "5_consecutive_days_active",

    # When user tries to use a paid-only feature
    "github_pr_attempted_on_free",
    "context_over_4096_attempted",
    "claude_sonnet_requested_on_free",

    # Monthly summary email (automated)
    "monthly_usage_summary"
]

async def should_show_upgrade_prompt(org_id: str,
                                     trigger: str) -> dict | None:
    """
    Returns upgrade prompt payload or None.
    Rate-limit upgrade prompts — max 1 per session.
    """
    session_key = f"upgrade_shown:{org_id}:{date.today()}"
    if r.get(session_key):
        return None  # Already shown today — don't nag

    r.setex(session_key, 86400, "1")

    prompts = {
        "github_pr_attempted_on_free": {
            "title":   "PR creation requires Startup plan",
            "message": "Your goal would create a GitHub PR, which is a "
                       "paid feature. Upgrade to enable it.",
            "highlight_feature": "GitHub PR creation",
            "price":   "$29/month",
            "url":     "https://mergit.io/pricing?ref=pr_gate"
        },
        "80pct_daily_used": {
            "title":   "You're getting real value from Mergit",
            "message": "You've used 16 of your 20 daily goals. "
                       "Startup gives you 500/day.",
            "price":   "$29/month",
            "url":     "https://mergit.io/pricing?ref=80pct"
        },
        "5_consecutive_days_active": {
            "title":   "You're a power user",
            "message": "You've been using Mergit every day this week. "
                       "Unlock full access for $29/month.",
            "price":   "$29/month",
            "url":     "https://mergit.io/pricing?ref=power_user"
        }
    }
    return prompts.get(trigger)
```

#### Frontend Usage Indicator (what orgs see)

```
┌─────────────────────────────────────────────────────┐
│  Mergit — Free Plan                                  │
│                                                      │
│  Goals today:  ████████████░░░░  16 / 20 used       │
│  Resets in:    4 hours 23 minutes                    │
│                                                      │
│  ⚡ Upgrade to Startup — 500 goals/day  →            │
└─────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────┐
│  Daily limit reached                                 │
│                                                      │
│  You've used all 20 free goals today.                │
│  Your limit resets in 4 hours 23 minutes.            │
│                                                      │
│  Want more?                                          │
│  ┌───────────────────────────────────────────────┐  │
│  │  Startup Plan — $29/month                     │  │
│  │  ✓ 500 goals/day                              │  │
│  │  ✓ Claude Sonnet access                       │  │
│  │  ✓ GitHub PR creation                         │  │
│  │  ✓ No batch queue delay                       │  │
│  │  [Upgrade now]  [Remind me at reset]          │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 4.3 At-Capacity Handling (Server-Side, Not User Fault)

```python
# Separate from user limits — this is your infrastructure issue

class CapacityManager:

    CIRCUIT_BREAKER_THRESHOLD = 0.95  # 95% of LLM budget consumed

    async def check_global_capacity(self) -> bool:
        """Returns True if system has capacity."""
        # Check your own LLM spend rate for the day
        daily_spend = float(r.get("global:spend:today") or 0)
        daily_budget = float(os.getenv("DAILY_LLM_BUDGET_USD", "100"))

        if daily_spend / daily_budget > self.CIRCUIT_BREAKER_THRESHOLD:
            # Post to status page, alert team
            await self._alert_team("LLM budget 95% consumed")
            return False

        # Check LLM provider availability
        providers_healthy = await self._check_provider_health()
        return providers_healthy

    async def handle_at_capacity(self, org_id: str,
                                  request: dict) -> dict:
        """
        Like Claude.ai's 'at capacity' — not the user's fault.
        Queue the request, don't reject it.
        """
        request_id = await self.batcher.enqueue(org_id, request)
        return {
            "status":    "queued",
            "request_id": request_id,
            "message":   "Mergit is handling high demand right now. "
                         "Your goal has been queued and will process shortly.",
            "eta_seconds": 60,
            "poll_url":  f"/v1/results/{request_id}"
        }
```

### 4.4 Model Transparency (tell users what model they get)

```python
# Claude.ai shows "Using claude-3-5-sonnet" in the UI.
# Do the same — free users see what model handled their request.

MODEL_DISPLAY_NAMES = {
    "groq/llama-4-maverick":      "Mergit Fast (Llama 4)",
    "groq/llama-3.3-70b":         "Mergit Standard (Llama 3.3)",
    "anthropic/claude-haiku-4-5": "Claude Haiku",
    "anthropic/claude-sonnet-4-6":"Claude Sonnet",
    "anthropic/claude-opus-4-8":  "Claude Opus",
}

# Add to every response:
response["model_used"] = MODEL_DISPLAY_NAMES.get(model, model)
response["model_note"] = "Upgrade to Startup for Claude Sonnet access" \
                          if tier == "free" else None
```

---

## 5. Five-Layer Cost Defense Stack

```
Request from Free Org
         │
         ▼
  ┌─────────────────────┐
  │ Layer 1             │  Model Routing     → Groq / Haiku (cheapest)
  ├─────────────────────┤
  │ Layer 2             │  Token Budget      → Hard stop at limits
  ├─────────────────────┤
  │ Layer 3             │  Cache Maximum     → Serve from cache, 0 tokens
  ├─────────────────────┤
  │ Layer 4             │  Context Compress  → Shrink what gets sent
  ├─────────────────────┤
  │ Layer 5             │  Batch API         → 50% off remaining calls
  └─────────────────────┘
```

---

### Layer 1: Model Routing by Tier

```python
# config/tier_routing.py

TIER_MODEL_PRIORITY = {
    "free": [
        "groq/llama-4-maverick",       # Primary — near-zero cost
        "groq/llama-3.3-70b",          # Fallback
        "anthropic/claude-haiku-4-5",  # Last resort
    ],
    "startup": [
        "groq/llama-4-maverick",
        "anthropic/claude-haiku-4-5",
        "anthropic/claude-sonnet-4-6",
    ],
    "enterprise": [
        "anthropic/claude-sonnet-4-6",
        "anthropic/claude-opus-4-8",
        "openai/gpt-4o",
        "groq/llama-4-maverick",
    ]
}

ROLE_MODEL_OVERRIDE = {
    "free": {
        "researcher":  "groq/llama-4-maverick",
        "writer":      "groq/llama-3.3-70b",
        "coder":       "anthropic/claude-haiku-4-5",  # Groq too weak for code
        "integrator":  "groq/llama-3.3-70b",
        "summarizer":  "groq/llama-3.1-8b",           # cheapest for summaries
    },
    "startup": {
        "researcher":  "groq/llama-4-maverick",
        "writer":      "groq/llama-3.3-70b",
        "coder":       "anthropic/claude-haiku-4-5",
        "integrator":  "anthropic/claude-haiku-4-5",
        "summarizer":  "groq/llama-3.1-8b",
    },
    "enterprise": {
        "researcher":  "groq/llama-4-maverick",
        "writer":      "anthropic/claude-sonnet-4-6",
        "coder":       "anthropic/claude-sonnet-4-6",
        "integrator":  "anthropic/claude-sonnet-4-6",
        "summarizer":  "groq/llama-3.1-8b",
    }
}

def get_model(tier: str, role: str) -> str:
    return ROLE_MODEL_OVERRIDE.get(tier, {}).get(role) \
        or TIER_MODEL_PRIORITY[tier][0]
```

**Cost impact of model routing:**

| Request type | Without routing | With routing | Saving |
|-------------|----------------|-------------|--------|
| Researcher (free) | $3.00/M (Sonnet) | $0.20/M (Groq) | **93%** |
| Coder (free) | $3.00/M (Sonnet) | $0.80/M (Haiku) | **73%** |
| Summary (free) | $0.80/M (Haiku) | $0.05/M (8B) | **94%** |

---

### Layer 2: Token Budget Ledger

```python
# middleware/token_budget.py

BILLABLE_WEIGHTS = {
    "anthropic/claude-haiku-4-5": {
        "input":        0.80,
        "output":       4.00,
        "cache_read":   0.08,   # 0.1 × 0.80
        "cache_write":  1.00,   # 1.25 × 0.80
    },
    "anthropic/claude-sonnet-4-6": {
        "input":        3.00,
        "output":       15.00,
        "cache_read":   0.30,
        "cache_write":  3.75,
    },
    "groq/llama-4-maverick": {
        "input":        0.20,
        "output":       0.60,
        "cache_read":   0.20,   # no cache discount on Groq
        "cache_write":  0.20,
    },
}

class TokenBudgetMiddleware:

    async def pre_flight(self, org_id: str,
                         estimated_input: int) -> None:
        tier   = await self._get_tier_cached(org_id)
        limits = TIER_LIMITS[tier]
        if limits["tokens_per_day"] is None:
            return  # Enterprise — unlimited

        day_key = f"budget:day:{org_id}:{date.today().isoformat()}"
        used    = float(r.get(day_key) or 0)

        if used + estimated_input > limits["tokens_per_day"]:
            goals_used = tokens_to_goals(int(used), tier)
            raise LimitReachedException(
                state=FreeTierState.LIMIT_HIT,
                goals_used=goals_used
            )

    async def post_call(self, org_id: str, model: str,
                        usage: dict) -> float:
        prices   = BILLABLE_WEIGHTS.get(model, BILLABLE_WEIGHTS["groq/llama-4-maverick"])
        raw_in   = usage.get("input_tokens", 0)
        raw_out  = usage.get("output_tokens", 0)
        c_read   = usage.get("cache_read_input_tokens", 0)
        c_write  = usage.get("cache_creation_input_tokens", 0)
        regular  = raw_in - c_read - c_write

        cost_usd = (
            regular * prices["input"]       / 1_000_000 +
            c_read  * prices["cache_read"]  / 1_000_000 +
            c_write * prices["cache_write"] / 1_000_000 +
            raw_out * prices["output"]      / 1_000_000
        )
        billable = int(cost_usd / prices["input"] * 1_000_000)

        day_key   = f"budget:day:{org_id}:{date.today().isoformat()}"
        month_key = f"budget:month:{org_id}:{date.today().strftime('%Y-%m')}"
        pipe = r.pipeline()
        pipe.incrbyfloat(day_key,   billable)
        pipe.expire(day_key,   86400 * 2)
        pipe.incrbyfloat(month_key, billable)
        pipe.expire(month_key, 86400 * 35)
        pipe.execute()

        asyncio.create_task(self._log_usage(org_id, model, usage, cost_usd))
        return cost_usd

    async def _get_tier_cached(self, org_id: str) -> str:
        key  = f"org:tier:{org_id}"
        tier = r.get(key)
        if not tier:
            tier = await db.fetchval(
                "SELECT tier FROM org_token_budgets WHERE org_id = $1", org_id
            )
            r.setex(key, 300, tier)
        return tier.decode() if isinstance(tier, bytes) else tier
```

---

### Layer 3: Cache Maximization

#### Shared Cross-Org Semantic Cache

```python
class SharedSemanticCache:
    """
    One cache pool for all free orgs.
    Org A's answer saves Org B's tokens — pay once, serve many.
    """
    SHARED_THRESHOLD  = 0.92
    PRIVATE_THRESHOLD = 0.97

    async def lookup(self, query: str, org_id: str,
                     tier: str) -> str | None:
        if await self._is_sensitive(query):
            return await self._private_lookup(
                f"{org_id}:{query}", self.PRIVATE_THRESHOLD
            )
        if tier == "free":
            return await self._shared_lookup(query, self.SHARED_THRESHOLD)
        hit = await self._private_lookup(
            f"{org_id}:{query}", self.PRIVATE_THRESHOLD
        )
        return hit or await self._shared_lookup(query, self.SHARED_THRESHOLD)
```

#### Shared System Prompt (biggest free-tier win)

```python
# All free orgs share one static prefix — written to Anthropic cache once,
# read at 0.1× cost by every subsequent free org.

FREE_TIER_SYSTEM = """You are a Mergit AI assistant on the free plan.
[...1500+ tokens of shared instructions, tool definitions, role guidelines...]
Keep responses concise (max 512 tokens). Explain clearly when paid features
are needed."""

system_with_cache = [
    {
        "type": "text",
        "text": FREE_TIER_SYSTEM,
        "cache_control": {"type": "ephemeral"}
        # First write: 1.25× cost. Every subsequent read: 0.1× cost.
    },
    {
        "type": "text",
        "text": f"Org: {org_name}\nGoal: {user_goal}"
        # Dynamic — not cached
    }
]
```

#### Embedding Cache

```python
# Key: emb:{model}:{dims}:{sha256(text)}   TTL: infinite
# Avoids text-embedding-3-small call before every pgvector lookup

async def get_or_create_embedding(text: str) -> list[float]:
    key    = f"emb:text-embedding-3-small:1536:{sha256(text)}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    vec = (await openai.embeddings.create(
        model="text-embedding-3-small", input=text, dimensions=1536
    )).data[0].embedding
    r.set(key, json.dumps(vec))   # no TTL
    return vec
```

---

### Layer 4: Context Compression

```python
class ContextCompressor:

    TIER_CTX_LIMITS = {
        "free":       4_096,
        "startup":   32_768,
        "enterprise": 200_000,
    }
    TIER_HISTORY_LIMIT = {
        "free":       10,
        "startup":    50,
        "enterprise": None,
    }

    async def compress(self, messages: list, tier: str) -> list:
        max_ctx  = self.TIER_CTX_LIMITS[tier]
        max_hist = self.TIER_HISTORY_LIMIT[tier]

        if max_hist and len(messages) > max_hist:
            old, recent = messages[:-max_hist], messages[-max_hist:]
            summary = await self._summarize(old)
            messages = [{"role": "system",
                         "content": f"[Earlier summary]: {summary}"}] + recent

        if self._count_tokens(messages) > max_ctx:
            messages = self._truncate_to_limit(messages, max_ctx)

        return messages

    async def _summarize(self, messages: list) -> str:
        key    = f"summary:{sha256(str(messages))}"
        cached = r.get(key)
        if cached:
            return cached.decode()
        resp = await litellm.acompletion(
            model="groq/llama-3.1-8b-instant",   # $0.05/M — cheapest
            messages=[{"role": "user",
                        "content": f"Summarize in 2-3 sentences:\n{messages}"}],
            max_tokens=100
        )
        summary = resp.choices[0].message.content
        r.setex(key, 86400, summary)
        return summary
```

---

### Layer 5: Batch API Discount

```python
class FreeTierBatchProcessor:
    """
    Free tier → batch queue → Anthropic Batch API every 5 minutes.
    50% cheaper. Client polls for result.
    """

    async def enqueue(self, org_id: str, request: dict) -> str:
        request_id = str(uuid4())
        window     = int(time.time()) // 300
        r.rpush(f"batch:queue:{window}",
                json.dumps({"request_id": request_id,
                            "org_id": org_id, "request": request}))
        return request_id

    async def get_result(self, request_id: str) -> dict:
        result = r.get(f"batch:result:{request_id}")
        if result:
            return {"status": "done", "response": json.loads(result)}
        return {"status": "processing", "retry_after_seconds": 30}

    async def process_batch(self):
        """Runs every 5 minutes via cron."""
        window = int(time.time()) // 300 - 1
        items  = [json.loads(i)
                  for i in r.lrange(f"batch:queue:{window}", 0, -1)]
        if not items:
            return

        batch = await anthropic_client.beta.messages.batches.create(
            requests=[{
                "custom_id": i["request_id"],
                "params": {
                    "model":      "claude-haiku-4-5",
                    "max_tokens": 512,
                    "system":     i["request"]["system"],
                    "messages":   i["request"]["messages"],
                }
            } for i in items]
        )
        await self._poll_and_store(batch.id, items)
```

---

## 6. Full Request Flow

```
Free Org Request
        │
        ▼
[Claude.ai Pattern] Get free tier state
        │ NORMAL / WARNING / LIMIT_HIT / AT_CAPACITY
        │
        ├── LIMIT_HIT → Return upgrade prompt, reset timer, no raw 429
        ├── AT_CAPACITY → Queue request, return ETA (server fault, not user)
        │
        ▼ NORMAL or WARNING
[Rate limit] 5 RPM, 100/day (Redis INCR)
        │
        ▼
[Layer 3] Shared semantic cache lookup (cosine > 0.92)
        │ HIT → return response + usage banner  0 tokens ✓
        │ MISS ↓
        ▼
[Layer 2] Token budget pre-flight (50k tokens/day)
        │
        ▼
[Layer 4] Context compression (cap 4,096 tokens, summarize old msgs)
        │
        ▼
[Layer 1] Model routing: Groq Maverick / Haiku by role
        │
        ▼
[Layer 5] Batch queue (5-min delay, 50% Batch API discount)
        │
        ▼
[Layer 0] Anthropic shared prefix cache (cache_control: ephemeral)
        │ Read: 0.1× cost   Write (first call): 1.25× cost
        ▼
LLM Call (max_tokens: 512, cheapest available model)
        │
        ▼
[Layer 2] Post-call accounting (weighted billable tokens → Redis + PG)
        │
        ▼
[Layer 3] Store in shared semantic cache (helps next org)
        │
        ▼
Response + usage indicator + optional upgrade banner
```

---

## 7. Cost Calculations with Real Numbers

### Scenario: 10 free orgs, 500,000 tokens/month each

| Without optimization | Cost |
|---------------------|------|
| Claude Sonnet input (4M tokens) | $12.00 |
| Claude Sonnet output (1M tokens) | $15.00 |
| **Total** | **$27.00/month** |

| With full 5-layer stack | Remaining cost |
|------------------------|---------------|
| Groq handles 80% | $5.40 |
| Haiku for remaining 20% | $1.46 |
| Shared semantic cache 60% hit | $0.58 |
| Prompt prefix cache on Haiku | $0.34 |
| Batch API 50% off remainder | $0.22 |
| Context compression 40% reduction | **$0.13** |

**Total for 10 orgs: $0.13–$0.40/month ($0.01–$0.04 per org)**

### Token Savings by Layer

| Layer | Tokens saved | % saving |
|-------|-------------|---------|
| Shared semantic cache (60% hit) | 3,000,000 | 60% |
| Anthropic prefix cache | 800,000 input | 90% of prefix |
| Groq routing (80% of calls) | N/A | 93% vs Sonnet |
| Batch API | N/A | 50% |
| Context compression | 400,000 | 40% input |
| **Combined** | | **95–98%** |

---

## 8. Database Schema

```sql
CREATE TABLE org_token_budgets (
    org_id              UUID PRIMARY KEY,
    org_name            TEXT NOT NULL,
    tier                TEXT NOT NULL DEFAULT 'free'
                        CHECK (tier IN ('free','startup','enterprise')),
    tokens_used_day     BIGINT  DEFAULT 0,
    tokens_used_month   BIGINT  DEFAULT 0,
    goals_used_day      INT     DEFAULT 0,   -- user-facing display
    goals_used_month    INT     DEFAULT 0,
    day_reset_at        TIMESTAMPTZ DEFAULT date_trunc('day',  NOW()),
    month_reset_at      TIMESTAMPTZ DEFAULT date_trunc('month', NOW()),
    hard_limit_day      BIGINT,
    hard_limit_month    BIGINT,
    is_suspended        BOOLEAN DEFAULT FALSE,
    suspension_reason   TEXT,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE token_usage_log (
    id                  BIGSERIAL PRIMARY KEY,
    org_id              UUID REFERENCES org_token_budgets(org_id),
    model               TEXT NOT NULL,
    model_display       TEXT,               -- human-readable name
    role                TEXT,
    input_tokens        INT  DEFAULT 0,
    output_tokens       INT  DEFAULT 0,
    cache_read_tokens   INT  DEFAULT 0,
    cache_write_tokens  INT  DEFAULT 0,
    billable_tokens     INT  DEFAULT 0,
    actual_cost_usd     NUMERIC(12, 8),
    cache_saved_usd     NUMERIC(12, 8),
    was_batch           BOOLEAN DEFAULT FALSE,
    was_cache_hit       BOOLEAN DEFAULT FALSE,
    cache_layer         TEXT,
    free_tier_state     TEXT,               -- normal/warning/limit_hit
    upgrade_shown       BOOLEAN DEFAULT FALSE,
    endpoint            TEXT,
    request_id          UUID,
    created_at          TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE TABLE org_daily_summary (
    org_id              UUID REFERENCES org_token_budgets(org_id),
    date                DATE,
    total_requests      INT     DEFAULT 0,
    cache_hits          INT     DEFAULT 0,
    cache_hit_rate      NUMERIC(5,2),
    tokens_used         BIGINT  DEFAULT 0,
    tokens_saved        BIGINT  DEFAULT 0,
    goals_used          INT     DEFAULT 0,
    cost_usd            NUMERIC(12,6),
    saved_usd           NUMERIC(12,6),
    upgrade_prompts_shown INT   DEFAULT 0,
    PRIMARY KEY (org_id, date)
);
```

---

## 9. Implementation Code

### Full Request Handler (all layers integrated)

```python
# handlers/llm_request.py

class LLMRequestHandler:

    def __init__(self):
        self.budget      = TokenBudgetMiddleware()
        self.cache       = SharedSemanticCache()
        self.compressor  = ContextCompressor()
        self.batcher     = FreeTierBatchProcessor()
        self.ft_response = FreeTierResponseBuilder()
        self.capacity    = CapacityManager()

    async def handle(self, org_id: str, role: str,
                     messages: list, goal: str) -> dict:
        tier       = await self.budget._get_tier_cached(org_id)
        goals_used = await self._get_goals_used(org_id)

        # ── Claude.ai pattern: check state first ────────────────
        state = await self.ft_response.get_state(org_id)

        if state == FreeTierState.LIMIT_HIT:
            return self.ft_response.build_response(
                state, org_id, None, goals_used
            )

        if not await self.capacity.check_global_capacity():
            return await self.capacity.handle_at_capacity(org_id, {
                "system": self._build_system(tier),
                "messages": messages
            })

        # ── Layer 3a: Shared semantic cache ─────────────────────
        hit = await self.cache.lookup(goal, org_id, tier)
        if hit:
            await self._track_cache_hit(org_id, "semantic")
            response = self.ft_response.build_response(
                state, org_id, hit, goals_used
            )
            response["cache"] = "semantic_hit"
            response["tokens"] = 0
            return response

        # ── Layer 2: Budget pre-flight ───────────────────────────
        await self.budget.pre_flight(org_id, len(goal) // 4)

        # ── Layer 4: Context compression ─────────────────────────
        messages = await self.compressor.compress(messages, tier)

        # ── Layer 1: Model routing ────────────────────────────────
        model = get_model(tier, role)

        # ── Layer 5: Batch queue for free tier ────────────────────
        if tier == "free":
            request_id = await self.batcher.enqueue(org_id, {
                "system":   self._build_system(tier),
                "messages": messages,
                "model":    model
            })
            resp = self.ft_response.build_response(
                state, org_id, None, goals_used
            )
            resp["status"]     = "queued"
            resp["request_id"] = request_id
            resp["poll_url"]   = f"/v1/results/{request_id}"
            resp["model_used"] = MODEL_DISPLAY_NAMES.get(model, model)
            return resp

        # ── Real-time call (paid tiers) ───────────────────────────
        llm_response = await litellm.acompletion(
            model      = model,
            messages   = messages,
            system     = self._build_system(tier),
            max_tokens = TIER_LIMITS[tier]["max_output_tokens"],
        )

        content = llm_response.choices[0].message.content
        usage   = llm_response.usage.__dict__

        # ── Layer 2: Post-call accounting ─────────────────────────
        await self.budget.post_call(org_id, model, usage)

        # ── Layer 3b: Store in shared cache ───────────────────────
        asyncio.create_task(self.cache.store(goal, content, org_id, tier))

        # ── Build response with Claude.ai-style wrapper ───────────
        resp = self.ft_response.build_response(state, org_id, content, goals_used + 1)
        resp["model_used"] = MODEL_DISPLAY_NAMES.get(model, model)
        resp["tokens"]     = usage

        # Upgrade prompt if triggered
        prompt = await should_show_upgrade_prompt(org_id, "80pct_daily_used") \
                 if state == FreeTierState.WARNING else None
        if prompt:
            resp["upgrade_prompt"] = prompt

        return resp

    def _build_system(self, tier: str) -> list:
        text = FREE_TIER_SYSTEM_PROMPT if tier == "free" \
               else PAID_TIER_SYSTEM_PROMPT
        return [{"type": "text", "text": text,
                 "cache_control": {"type": "ephemeral"}}]
```

### Redis Key Schema

```
# Budget
budget:day:{org_id}:{YYYY-MM-DD}        → float (billable tokens)   TTL: 48h
budget:month:{org_id}:{YYYY-MM}         → float (billable tokens)   TTL: 35d
goals:day:{org_id}:{YYYY-MM-DD}         → int (goal count)          TTL: 48h
org:tier:{org_id}                       → string                    TTL: 5m
upgrade_shown:{org_id}:{YYYY-MM-DD}     → "1"                       TTL: 24h

# Cache
emb:{model}:{dims}:{sha256}             → JSON vector               TTL: none
semcache:shared:{sha256(query)}         → JSON response             TTL: 24h
semcache:private:{org_id}:{sha256}      → JSON response             TTL: 24h
sensitive:{md5(query)}                  → "0"/"1"                   TTL: 1h
summary:{sha256(messages)}              → string                    TTL: 24h

# Batch
batch:queue:{window}                    → list of JSON requests     TTL: 15m
batch:result:{request_id}              → JSON response             TTL: 1h
batch:eta:{request_id}                 → int seconds               TTL: 10m

# Rate limiting
ratelimit:{org_id}:rpm:{minute}         → int                       TTL: 60s
ratelimit:{org_id}:rpd:{date}           → int                       TTL: 48h

# Global capacity
global:spend:today                      → float USD                 TTL: 48h
```

---

## 10. Monitoring Dashboard Spec

### What orgs see (Claude.ai-style)

```
┌──────────────────────────────────────────────────┐
│  Mergit — Free Plan                               │
│                                                   │
│  Goals today:  ████████░░░░  16 / 20  (80%)      │
│  Resets in:    4h 23m                             │
│                                                   │
│  Model used:   Mergit Fast (Llama 4)              │
│  Cache saved:  47,000 tokens today ($0.04)        │
│                                                   │
│  ⚡ Upgrade to Startup — 500 goals/day  →         │
└──────────────────────────────────────────────────┘
```

### Admin panel (your team)

```
Free Tier Cost Summary — June 2026
────────────────────────────────────────────────────
Active free orgs:          12
Total goals served:        2,400
Goals from cache:          1,440  (60% hit rate)
Goals to LLM:              960

  ├── Groq (768 goals):    $0.31
  └── Haiku (192 goals):   $0.13
      ├── Cache reads:      $0.02
      └── Regular:          $0.11

Total output cost:          $0.64
Total actual cost:          $1.08
Without optimization:       $27.00
Total saved:                $25.92  (96%)
Cost per free org:          $0.09/month
────────────────────────────────────────────────────
Upgrade funnel:
  Upgrade prompts shown:    47
  Clicks on upgrade CTA:    12  (25.5% CTR)
  Conversions to Startup:   2   (4.3% of prompts)
  MRR from free→paid:       $58/month
  Cost of free tier:        $1.08/month
  ROI of free tier:         53.7×
────────────────────────────────────────────────────
Orgs approaching limit:
  Acme Corp       → 89% monthly budget
  StartupXYZ      → 5 RPM hit 23× today
```

### Key Metrics to Track

| Metric | Alert threshold | Action |
|--------|----------------|--------|
| `cache_hit_rate` | < 50% | Review prefix / semantic threshold |
| `cost_per_free_org_month` | > $1.00 | Tighten routing or limits |
| `batch_usage_rate` (free) | < 70% | Enforce batch more aggressively |
| `upgrade_prompt_ctr` | < 10% | A/B test prompt copy |
| `free_to_paid_conversion` | < 2% | Review upgrade trigger timing |
| `at_capacity_events` | > 5/day | Scale LLM budget or add queue |

---

## 11. Scaling Thresholds

```python
# Auto upgrade nudge triggers (show prompt, not forced upgrade)
UPGRADE_TRIGGERS = {
    "free_to_startup": [
        "daily_limit_hit_3_days_running",
        "monthly_70pct_used_before_day_20",
        "github_pr_attempted_on_free",
        "claude_sonnet_requested_on_free",
        "5_consecutive_days_active",
    ]
}

# Hard suspend (abuse, not limit exhaustion)
SUSPEND_TRIGGERS = {
    "free": [
        "monthly_tokens_over_110pct_limit",
        "rate_limit_violations_over_50_per_hour",
        "prompt_injection_detected",
        "credential_scraping_pattern_detected",
    ]
}
```

---

## Summary: The Economics

| Tier | Your cost | You charge | Margin |
|------|-----------|-----------|--------|
| Free | $0.04–0.40/org/month | $0 | –$0.40 (acquisition) |
| Startup | $2–8/org/month | $29–199/month | **$21–197/month** |
| Enterprise | $50–500/org/month | Custom $500–5000/month | **$450–4500/month** |

The free tier pays for itself the moment **1 in 50 free orgs upgrades to Startup.**
The caching infrastructure + Claude.ai-style UX is what makes that conversion
rate achievable rather than burning your budget before anyone pays you.

---

*Mergit Internal · June 2026*
*Prices: Anthropic official docs May 2026, Groq pricing page, OpenAI pricing page.*
*Always verify current rates before committing to budgets.*
