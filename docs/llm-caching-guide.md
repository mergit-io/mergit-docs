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

**Break-even (counting the write request itself):**
- 5-min TTL: `1.25× write + 0.1× read = 1.35×` vs `2×` uncached → pays off at **2 requests**.
- 1-hour TTL: `2× write + 2×0.1× read = 2.2×` vs `3×` uncached → pays off at **3 requests**.

At our traffic (a stable system prompt hit hundreds of times per hour per agent type), every prompt-cache-eligible call after the first is at 0.1× input cost. Use 5-min TTL by default; switch to 1-hour only for prefixes that stay byte-identical across multi-minute gaps.

**Minimum cacheable prefix is model-dependent** — below it, the cache silently does nothing (`cache_creation_input_tokens: 0`, no error):

| Model | Minimum prefix |
|---|---|
| Sonnet 4.6 | **2,048 tokens** |
| Opus 4.8 / 4.7 / 4.6, Haiku 4.5 | 4,096 tokens |
| OpenAI (gpt-4o) | 1,024 tokens |

Our system prompt must clear 2,048 tokens on Sonnet 4.6 or prompt caching is inert. Verify with `response.usage.cache_creation_input_tokens > 0` on first call.

---

## Implementation

### Eligibility Constants

Defined once and enforced at every layer in code — not just documented. The wrong cache hit is a bug, so these are hard blocks the async judge cannot override.

```python
from enum import Enum

class AgentType(str, Enum):
    FAQ        = "faq"          # L2 threshold 0.95
    ROUTING    = "routing"      # L2 threshold 0.97
    HIT_LOOP   = "hitl"         # L2 threshold 0.99
    CODE       = "code"         # L1 ok, L2 never
    SECURITY   = "security"     # L1 never, L2 never
    CREDENTIAL = "credential"   # L1 never, L2 never

# Security/permission decisions must always recompute against current state.
NO_EXACT_CACHE    = {AgentType.SECURITY, AgentType.CREDENTIAL}
# Code: a small diff is a meaningful diff. Security/credential: context-specific.
NO_SEMANTIC_CACHE = {AgentType.CODE, AgentType.SECURITY, AgentType.CREDENTIAL}

L2_THRESHOLDS = {AgentType.FAQ: 0.95, AgentType.ROUTING: 0.97, AgentType.HIT_LOOP: 0.99}
```

### L1 — Exact Match Cache (Redis, SHA-256)

Identical prompts hit this before any embedding or API call. The key includes `agent_type` and `model` so the same string routed to different agents or models never collides. **Fail open:** if Redis is unreachable, treat it as a miss and continue — a cache outage must not take down request handling.

```python
import hashlib, json, redis

r = redis.from_url("redis://localhost:6379")

def exact_cache_key(prompt: str, agent_type: AgentType, model: str) -> str:
    # Normalise to maximise hit rate; scope by agent_type + model to prevent cross-task collisions.
    norm = prompt.strip().lower()
    digest = hashlib.sha256(f"{model}|{agent_type.value}|{norm}".encode()).hexdigest()
    return f"llm:exact:{digest}"

def exact_lookup(prompt: str, agent_type: AgentType, model: str) -> str | None:
    if agent_type in NO_EXACT_CACHE:
        return None
    try:
        val = r.get(exact_cache_key(prompt, agent_type, model))
        return json.loads(val) if val else None
    except redis.RedisError:
        return None  # fail open — never block on cache

def exact_store(prompt: str, response: str, agent_type: AgentType, model: str, ttl: int = 3600):
    if agent_type in NO_EXACT_CACHE:
        return
    try:
        r.setex(exact_cache_key(prompt, agent_type, model), ttl, json.dumps(response))
    except redis.RedisError:
        pass  # a failed write is acceptable; the next request recomputes
```

---

### L2 — Semantic Cache (pgvector, per-task threshold)

Near-duplicate queries return the stored response without an LLM call. We use pgvector because PostgreSQL 16 is already in the stack — no additional infrastructure.

**Threshold is task-specific** (see Eligibility Constants above). Cosine similarity is length-invariant, so differently-worded queries with the same intent score high; the threshold sets the acceptable false-positive rate per task. Embedding is CPU-bound — run it in a thread so it never blocks the event loop.

```python
import asyncio, asyncpg
from sentence_transformers import SentenceTransformer

encoder = SentenceTransformer("all-MiniLM-L6-v2")  # 384-dim, ~14ms/query on CPU

async def _embed(text: str) -> list[float]:
    # Offload the blocking encode() so the async service stays responsive.
    vec = await asyncio.to_thread(encoder.encode, text, normalize_embeddings=True)
    return vec.tolist()

async def semantic_lookup(pool: asyncpg.Pool, query: str, agent_type: AgentType) -> str | None:
    if agent_type in NO_SEMANTIC_CACHE:
        return None
    try:
        vec = await _embed(query)
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
            vec, L2_THRESHOLDS[agent_type], agent_type.value
        )
        return row["response"] if row else None
    except (asyncpg.PostgresError, OSError):
        return None  # fail open

async def semantic_store(pool: asyncpg.Pool, query: str, response: str,
                         agent_type: AgentType, ttl_hours: int = 1):
    if agent_type in NO_SEMANTIC_CACHE:
        return
    vec = await _embed(query)
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

**Schema** (pgvector extension + a cleanup job to bound table growth):
```sql
CREATE EXTENSION IF NOT EXISTS vector;

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

-- Run on a schedule (pg_cron or an app job). TTL filtering in queries handles
-- correctness; this reclaims space and keeps the ivfflat index healthy.
-- DELETE FROM semantic_cache WHERE expires_at < NOW();
```

> **Never cache secrets or PII.** Queries and responses are persisted in Redis and Postgres. Strip or refuse to cache anything containing credentials, tokens, private keys, or personal data before it reaches L1/L2 — the `NO_*_CACHE` blocks on security/credential agents cover the obvious cases, but content-level checks are still required for free-form queries.

**Async judge (off critical path):** after a cache miss + LLM response, a lightweight model decides if the response is generic enough to promote. Use `groq/llama-3.3-70b` here — low stakes, binary verdict, never blocks the user.

```python
async def judge_and_promote(query: str, response: str, pool, llm_client, agent_type: AgentType):
    if agent_type in NO_SEMANTIC_CACHE:
        return  # hard block — the judge cannot override eligibility rules

    verdict = await llm_client.async_call(
        f"Is this response reusable for semantically similar future queries? "
        f"Answer YES or NO only.\nQuery: {query}\nResponse: {response}"
    )
    if "YES" in verdict.upper():
        await semantic_store(pool, query, response, agent_type)
```

---

### L3 — Prompt Caching (Anthropic)

Caching is a **prefix match**: any byte change before a cache breakpoint invalidates everything after it. Render order is `tools → system → messages`, so static content must come first.

**Structure:** `[system instructions] → [tool definitions] → [static context] → [dynamic user query]`

**Rules that keep the cache alive:**
- Keep the system prompt **byte-stable** — no `datetime.now()`, request IDs, or per-user strings in it (they invalidate every request). Inject dynamic context as a later message, not in the system block.
- Serialize tool definitions deterministically (sorted keys); a reordered tool list invalidates the whole prefix.
- Max **4** breakpoints per request. 5-min TTL is `{"type": "ephemeral"}`; 1-hour is `{"type": "ephemeral", "ttl": "1h"}`.

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM = """You are the Mergit orchestration engine.
[... agent routing rules, tool definitions, Monad contract context — must exceed 2,048 tokens on Sonnet 4.6 ...]"""

def call_with_prompt_cache(user_query: str, history: list | None = None):
    response = client.messages.create(
        model="claude-sonnet-4-6",  # model is fixed by task, not by cache state
        max_tokens=1024,
        system=[{
            "type": "text",
            "text": SYSTEM,
            "cache_control": {"type": "ephemeral"}        # 5-min TTL
            # 1-hour: {"type": "ephemeral", "ttl": "1h"}  # use only for byte-stable prefixes
        }],
        messages=_build_messages(history, user_query)
    )
    usage = response.usage
    return response.content[0].text, {
        "written": usage.cache_creation_input_tokens,  # >0 confirms the prefix actually cached
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
    try:
        cached = r.get(key)
        if cached:
            return json.loads(cached)
    except redis.RedisError:
        pass  # fail open — run the tool
    result = fn(**args)
    try:
        r.setex(key, ttl, json.dumps(result))
    except redis.RedisError:
        pass
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

MODEL = "claude-sonnet-4-6"

class MergitLLMHandler:
    def __init__(self, pg_pool, anthropic_client):
        self.pool = pg_pool
        self.llm  = anthropic_client
        self.metrics = {"exact_hits": 0, "semantic_hits": 0, "prompt_hits": 0, "misses": 0}

    async def handle(self, query: str, agent_type: AgentType,
                     history: Optional[list] = None) -> str:
        # L1: exact match — gated (security/credential skip), fail-open inside
        hit = exact_lookup(query, agent_type, MODEL)
        if hit:
            self.metrics["exact_hits"] += 1
            return hit

        # L2: semantic match — gated by agent type
        hit = await semantic_lookup(self.pool, query, agent_type)
        if hit:
            self.metrics["semantic_hits"] += 1
            return hit

        # L3 + full LLM call — model is fixed by task, not by cache state
        result, usage = call_with_prompt_cache(query, history)

        self.metrics["prompt_hits" if usage["read"] > 0 else "misses"] += 1

        # Promote off the critical path; eligibility re-checked inside each call
        asyncio.create_task(self._promote(query, result, agent_type))
        return result

    async def _promote(self, query: str, response: str, agent_type: AgentType):
        exact_store(query, response, agent_type, MODEL)
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

Switch to 1-hour prompt cache TTL only once the system prompt is byte-stable across multi-minute gaps; the 2× write premium pays off at 3 requests.

---

### Reliability & Safety

- **Fail open.** Every cache read is wrapped so a Redis/Postgres outage degrades to a normal LLM call, never a request failure. A cache is an optimisation, not a dependency.
- **Stampede control (single-flight).** When a hot entry expires, many concurrent requests miss at once and all hit the LLM. Hold a short Redis lock (`SET key NX EX 10`) per cache key on miss; the winner computes and writes, the rest briefly wait and re-read. Without this, a popular query expiring under load produces a thundering herd of identical LLM calls.
- **moka → Redis → source.** In the Rust services, the in-process moka layer absorbs hot keys before Redis, preventing Redis saturation at high RPS. Same fail-open rule applies at each tier.

---

### Cache Invalidation

```python
async def invalidate_query(query: str, pool, agent_type: AgentType, model: str):
    r.delete(exact_cache_key(query, agent_type, model))
    await pool.execute(
        "DELETE FROM semantic_cache WHERE query = $1 AND agent_type = $2",
        query, agent_type.value
    )

def on_deploy():
    # Flush exact-match cache — prompts/responses may have changed with new code.
    # Semantic cache survives — entries are query-keyed, overwritten as fresh
    # responses get promoted; hit rate dips briefly post-deploy, then recovers.
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
| Semantic hit rate (code/security/credential) | 0% always | Alert if non-zero — eligibility rule broken |
| Exact hit rate (security/credential) | 0% always | Alert if non-zero — eligibility rule broken |
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
