# LLM Caching — Mergit Implementation Reference

> Internal engineering reference. The goal is **cost reduction without quality regression**.
> Caching makes the right model cheaper — it is not a reason to use a weaker model.
> Prompt caching alone yields up to 90% reduction on cached input tokens (0.1× base price on hits).

---

## Model Assignment by Agent Type

Caching does not change which model runs a task. These assignments are fixed regardless of cache state.

| Agent / Task | Model | Why |
|---|---|---|
| Goal decomposition, task DAG planning | `groq/llama-4-maverick` | Primary workhorse — fast, capable, cheap |
| Multi-step orchestration reasoning | `groq/llama-4-maverick` | Handles complex routing without Anthropic cost |
| **Code generation, code review** | `anthropic/claude-sonnet-4-6` | Sonnet is the best coding model; never downgrade this |
| Security gate evaluation (HitL approval) | `anthropic/claude-sonnet-4-6` | Accuracy-critical — wrong approval = security incident |
| Credential / permission decisions | `anthropic/claude-sonnet-4-6` | Same reason — stakes too high for a lighter model |
| Reputation score explanation | `groq/llama-4-maverick` | Formatting task, not deep reasoning |
| Simple agent routing (yes/no decisions) | `groq/llama-3.3-70b` | Classification task, overkill for anything heavier |
| Async cache judge (promotion decisions) | `groq/llama-3.3-70b` | Off critical path, low-stakes binary verdict |
| FAQ / status responses | `groq/llama-4-maverick` → cache | Most hits come from L1/L2 anyway |
| GPT-4o | Fallback only | Only reached when Groq + Anthropic both unavailable |

**LiteLLM fallback chain:** `groq/llama-4-maverick` → `groq/llama-3.3-70b` → `anthropic/claude-sonnet-4-6` → `openai/gpt-4o`

The fallback order is for availability, not for cost optimisation. Do not reorder it to save money.

---

## Cache Eligibility by Task

Not every task should be cached at every layer. The wrong cache hit is worse than a miss.

| Task | L1 Exact | L2 Semantic | L3 Prompt | L4 Tool | Notes |
|---|---|---|---|---|---|
| FAQ / status queries | ✓ | ✓ (0.95) | ✓ | ✓ | Aggressive caching is safe here |
| Agent routing decisions | ✓ | ✓ (0.97) | ✓ | — | Default threshold |
| Reputation score fetch | ✓ | ✓ (0.97) | ✓ | ✓ (60s TTL) | Tool result changes — short TTL |
| Blockchain reads / proofs | ✓ | — | ✓ | ✓ (300s TTL) | Tool cache safe; semantic unsafe |
| **Code generation** | ✓ | **Never** | ✓ | — | Semantic cache cannot safely generalise code |
| **Security gate evaluation** | **Never** | **Never** | ✓ | — | Must run fresh; stale approval = vulnerability |
| **Credential / permission decisions** | **Never** | **Never** | ✓ | — | Same — every decision is context-specific |
| Multi-turn conversations | — | — | ✓ (history) | — | Prompt cache on conversation history only |
| Human-in-the-loop summaries | ✓ | ✓ (0.99) | ✓ | — | Very high threshold — near-identical only |

**Key rules:**
- Code generation: `prompt cache = yes` (saves on the system prompt), `semantic cache = no` (a small code difference is a meaningful difference)
- Security/permission: never return a cached yes/no decision — always recompute against current state
- Blockchain tool results: cache the RPC output (L4), not the LLM interpretation

---

## Our Caching Stack

Four layers, cheapest check first. A hit at any layer stops execution for that request.

```
Request arrives
      │
      ▼
[L1] Exact Match          ──HIT──▶  Return in <1ms  (zero LLM tokens, zero tool calls)
      │ MISS                        Key: SHA-256(prompt)  Store: Redis
      ▼
[L2] Semantic Match       ──HIT──▶  Return stored response  (zero LLM tokens)
      │ MISS                        Key: pgvector cosine  Store: PostgreSQL + Redis
      ▼
[L3] Prompt Cache         ──HIT──▶  LLM call at 0.1× input token cost  (90% savings)
      │ MISS                        Key: cache_control marker  Store: Anthropic / OpenAI
      ▼
[L4] Tool Output Cache    ──HIT──▶  Skip tool re-execution, reuse result
      │ MISS                        Key: tool:{name}:{SHA-256(args)}  Store: Redis
      ▼
      Full LLM call + tool execution  ◀──── reach here as rarely as possible
```

**Rust service hierarchy:** in-process `moka` LRU (hot keys, ~1ms) → Redis 7 (~2ms) → provider API. moka prevents Redis saturation at high RPS.

---

## Pricing — Anthropic (May 2026)

| Token type | Multiplier | Sonnet 4.6 ($3/M base) |
|---|---|---|
| Cache read (hit) | **0.1×** | $0.30/M |
| Cache write (5-min TTL) | 1.25× | $3.75/M |
| Cache write (1-hour TTL) | 2× | $6.00/M |
| Regular input | 1× | $3.00/M |

**Break-even:** 1-hour TTL costs 1× extra to write, saves 0.9× per hit. Break-even = **1.1 hits**. Any prefix read twice pays off. For a 2,000-token system prompt read 100× per hour, the savings are ~$0.54/hour per agent type at Sonnet pricing — material at scale.

---

## Implementation

### L1 — Exact Match Cache (Redis, SHA-256)

Identical prompts hit this before any embedding or API call. Normalise query strings (lowercase, strip whitespace) before hashing to maximise hit rate.

```python
import hashlib, json, redis

r = redis.from_url("redis://localhost:6379")

def exact_cache_key(prompt: str) -> str:
    normalised = prompt.strip().lower()
    return f"llm:exact:{hashlib.sha256(normalised.encode()).hexdigest()}"

def exact_lookup(prompt: str) -> str | None:
    val = r.get(exact_cache_key(prompt))
    return json.loads(val) if val else None

def exact_store(prompt: str, response: str, ttl: int = 3600):
    r.setex(exact_cache_key(prompt), ttl, json.dumps(response))
```

---

### L2 — Semantic Cache (pgvector, per-task threshold)

Near-duplicate queries return the stored response without an LLM call. We use pgvector because PostgreSQL 16 is already in the stack — no additional infrastructure.

**Threshold is task-specific.** A wrong cache hit on a code generation or security task is a bug.

```python
import asyncpg
from sentence_transformers import SentenceTransformer
from enum import Enum

encoder = SentenceTransformer("all-MiniLM-L6-v2")  # 384-dim, ~14ms/query on CPU

class AgentType(str, Enum):
    FAQ        = "faq"         # threshold: 0.95
    ROUTING    = "routing"     # threshold: 0.97
    CODE       = "code"        # NEVER cache semantically
    SECURITY   = "security"    # NEVER cache semantically
    HIT_LOOP   = "hitl"        # threshold: 0.99

THRESHOLDS = {
    AgentType.FAQ:      0.95,
    AgentType.ROUTING:  0.97,
    AgentType.HIT_LOOP: 0.99,
}

async def semantic_lookup(pool: asyncpg.Pool, query: str, agent_type: AgentType) -> str | None:
    if agent_type in (AgentType.CODE, AgentType.SECURITY):
        return None  # never serve cached response for these task types

    threshold = THRESHOLDS[agent_type]
    vec = encoder.encode(query, normalize_embeddings=True).tolist()
    row = await pool.fetchrow(
        """
        SELECT response, 1 - (embedding <=> $1::vector) AS similarity
        FROM semantic_cache
        WHERE agent_type = $3
          AND 1 - (embedding <=> $1::vector) >= $2
          AND expires_at > NOW()
        ORDER BY embedding <=> $1::vector
        LIMIT 1
        """,
        vec, threshold, agent_type.value
    )
    return row["response"] if row else None

async def semantic_store(pool: asyncpg.Pool, query: str, response: str,
                         agent_type: AgentType, ttl_hours: int = 1):
    if agent_type in (AgentType.CODE, AgentType.SECURITY):
        return  # never store these

    vec = encoder.encode(query, normalize_embeddings=True).tolist()
    await pool.execute(
        """
        INSERT INTO semantic_cache (query, embedding, response, agent_type, expires_at)
        VALUES ($1, $2::vector, $3, $4, NOW() + $5 * interval '1 hour')
        ON CONFLICT (query, agent_type) DO UPDATE
            SET response = EXCLUDED.response, expires_at = EXCLUDED.expires_at
        """,
        query, vec, response, agent_type.value, ttl_hours
    )
```

**Schema:**
```sql
CREATE TABLE semantic_cache (
    id         BIGSERIAL PRIMARY KEY,
    query      TEXT NOT NULL,
    agent_type TEXT NOT NULL,
    embedding  VECTOR(384) NOT NULL,
    response   TEXT NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (query, agent_type)
);
CREATE INDEX ON semantic_cache USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

**Async judge (off critical path):** after a cache miss + LLM response, a lightweight model decides if the response is generic enough to promote. Use `groq/llama-3.3-70b` here — low stakes, binary verdict, never blocks the user.

```python
async def judge_and_promote(query: str, response: str, pool, llm_client, agent_type: AgentType):
    if agent_type in (AgentType.CODE, AgentType.SECURITY):
        return  # hard block — judge cannot override eligibility rules

    verdict = await llm_client.async_call(
        f"Is this response reusable for semantically similar future queries? "
        f"Answer YES or NO only.\nQuery: {query}\nResponse: {response}"
    )
    if "YES" in verdict.upper():
        await semantic_store(pool, query, response, agent_type)
```

---

### L3 — Prompt Caching (Anthropic)

Static content must come first. Any change before the cache breakpoint invalidates the whole cache.

**Structure:** `[system instructions] → [tool definitions] → [static context] → [dynamic user query]`

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM = """You are the Mergit orchestration engine.
[... agent routing rules, tool definitions, Monad contract context — 2,000+ tokens ...]"""

def call_with_prompt_cache(user_query: str, history: list | None = None):
    response = client.messages.create(
        model="claude-sonnet-4-6",  # model choice is fixed by task — not by cache state
        max_tokens=1024,
        system=[{
            "type": "text",
            "text": SYSTEM,
            "cache_control": {"type": "ephemeral"}  # 5-min TTL default
        }],
        messages=_build_messages(history, user_query)
    )
    usage = response.usage
    return response.content[0].text, {
        "written": usage.cache_creation_input_tokens,
        "read":    usage.cache_read_input_tokens,
        "regular": usage.input_tokens,
    }

def _build_messages(history: list | None, query: str) -> list:
    messages = []
    if history:
        for i, msg in enumerate(history):
            # Cache the conversation history at the last message boundary
            content = msg["content"]
            if i == len(history) - 1:
                content = [{"type": "text", "text": content,
                            "cache_control": {"type": "ephemeral"}}]
            messages.append({"role": msg["role"], "content": content})
    messages.append({"role": "user", "content": query})
    return messages
```

**OpenAI (automatic, no marker):**
```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": SYSTEM},  # auto-cached if ≥1,024 tokens
        {"role": "user",   "content": user_query}
    ]
)
cached_tokens = response.usage.prompt_tokens_details.cached_tokens
```

---

### L4 — Tool Output Cache (Redis, SHA-256 args)

Deterministic tool calls with stable inputs skip re-execution. TTL is set per tool based on how fast the underlying data changes.

```python
import hashlib, json

def tool_cache_key(tool_name: str, args: dict) -> str:
    return f"tool:{tool_name}:{hashlib.sha256(json.dumps(args, sort_keys=True).encode()).hexdigest()}"

def cached_tool_call(tool_name: str, args: dict, fn, ttl: int = 300) -> dict:
    key = tool_cache_key(tool_name, args)
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    result = fn(**args)
    r.setex(key, ttl, json.dumps(result))
    return result

# TTL reference:
# get_reputation_score  → 60s    (scores update frequently)
# get_proof_hash        → 300s   (proofs are immutable once recorded)
# get_agent_capabilities → 600s  (capabilities change on upgrade only)
# monad_rpc_balance     → 30s    (chain state moves fast)
```

---

### Full Stack Handler

Checks every layer in order with agent type propagated through so eligibility rules are enforced at every layer.

```python
import asyncio
from typing import Optional

class MergitLLMHandler:
    def __init__(self, pg_pool, anthropic_client):
        self.pool = pg_pool
        self.llm  = anthropic_client
        self.metrics = {"exact_hits": 0, "semantic_hits": 0, "prompt_hits": 0, "misses": 0}

    async def handle(self, query: str, agent_type: AgentType,
                     history: Optional[list] = None) -> str:
        # L1: exact match — always safe regardless of agent type
        hit = exact_lookup(query)
        if hit:
            self.metrics["exact_hits"] += 1
            return hit

        # L2: semantic match — gated by agent type
        hit = await semantic_lookup(self.pool, query, agent_type)
        if hit:
            self.metrics["semantic_hits"] += 1
            return hit

        # L3 + full LLM call — model is fixed by agent type, not by cache
        result, usage = call_with_prompt_cache(query, history)

        self.metrics["prompt_hits" if usage["read"] > 0 else "misses"] += 1

        # Promote off the critical path
        asyncio.create_task(self._promote(query, result, agent_type))
        return result

    async def _promote(self, query: str, response: str, agent_type: AgentType):
        exact_store(query, response)
        await judge_and_promote(query, response, self.pool, self.llm, agent_type)

    def hit_rates(self) -> dict:
        total = sum(self.metrics.values()) or 1
        return {k: f"{v / total * 100:.1f}%" for k, v in self.metrics.items()}
```

---

## Production Operations

### TTL Configuration

| Layer | TTL | Change when |
|---|---|---|
| Exact match (Redis) | 3600s | Shorten if responses change frequently |
| Semantic (pgvector) | 1 hour | Matches Anthropic 1-hour prompt cache tier |
| Anthropic prompt cache | 300s (dev) / 3600s (stable) | Drop to 5-min during active prompt iteration |
| Tool output | 30–600s | Per-tool, based on data change frequency |
| moka (in-process) | 60s | Hot keys only, LRU eviction |

Switch to 1-hour prompt cache TTL once system prompt stabilises post-launch. The 2× write premium pays off after 1.1 reads — at moderate traffic this covers the cost within minutes.

---

### Cache Invalidation

```python
async def invalidate_query(query: str, pool, agent_type: AgentType):
    r.delete(exact_cache_key(query))
    await pool.execute(
        "DELETE FROM semantic_cache WHERE query = $1 AND agent_type = $2",
        query, agent_type.value
    )

def on_deploy():
    # Flush exact match cache — prompts may have changed with new code
    # Semantic cache survives — entries are query-keyed, not code-keyed
    for key in r.scan_iter("llm:exact:*"):
        r.delete(key)

def on_blockchain_state_change(tool_name: str, args: dict):
    r.delete(tool_cache_key(tool_name, args))
```

---

### Metrics — Cost and Quality

Track both. Cost metrics without quality metrics are incomplete — a high semantic hit rate with degraded output quality is a regression, not a win.

| Metric | Target | Action if off |
|---|---|---|
| `cache_read_tokens / total_input_tokens` | >60% | Restructure prompt or extend TTL |
| Semantic hit rate (FAQ agents) | >40% | Lower threshold or expand judge coverage |
| Semantic hit rate (code/security) | 0% always | Alert if non-zero — eligibility rule broken |
| Exact hit rate (batch workloads) | >20% | Normalise query strings before hashing |
| Tool cache hit rate (blockchain reads) | >70% | Extend TTL for stable chain state |
| Semantic false-positive rate | <2% | Raise cosine threshold for that agent type |
| User-reported quality regression | 0 | Audit which cache layer the regressed response came from |

Emit `cache_layer` tag with every response so quality regressions can be traced to the specific layer that served them.

---

### Cache-Aware RAG (Phase 3+)

When we add the RAG pipeline, retrieved document chunks that recur across queries can have their computed representations cached. **Do not reuse 100% of cached chunks** — selective recomputation of ≥15% is required or output quality drops ~50% (F1 regression measured across similar systems). Simpler alternative for launch: apply `cache_control` to the retrieved context block and let Anthropic cache the token computation server-side. Same savings, far less complexity.

---

*Mergit Internal · June 2026*
