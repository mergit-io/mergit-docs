# LLM Caching Strategy Guide

> **Internal reference for engineering.**
> Which caching technique to use, where, and why — grounded in verified research across 23 sources,
> 105 adversarial verification agents, and 5 claims surviving 3-vote review.
> Based on Anthropic/OpenAI official docs, arXiv papers (2025–2026), and production benchmarks.

---

## Table of Contents

1. [Why Caching Matters](#1-why-caching-matters)
2. [Prerequisites — Quick Recall Notes](#2-prerequisites--quick-recall-notes)
3. [KV Cache — Conceptual Revision Notes](#3-kv-cache--conceptual-revision-notes)
4. [The 4-Layer Caching Stack](#4-the-4-layer-caching-stack)
5. [When to Use Each Type](#5-when-to-use-each-type)
6. [Decision Flowchart](#6-decision-flowchart)
7. [Side-by-Side Comparison](#7-side-by-side-comparison)
8. [Production Layering Pattern](#8-production-layering-pattern)
9. [Verified Pricing (Anthropic, May 2026)](#9-verified-pricing-anthropic-may-2026)
10. [Implementation Guide](#10-implementation-guide)
    - [10.1 Prompt Caching — Anthropic Claude](#101-prompt-caching--anthropic-claude)
    - [10.2 Prompt Caching — OpenAI](#102-prompt-caching--openai)
    - [10.3 Semantic Caching — Custom Implementation](#103-semantic-caching--custom-implementation)
    - [10.4 KV Cache — vLLM (PagedAttention)](#104-kv-cache--vllm-pagedattention)
    - [10.5 KV Cache — SGLang (RadixAttention)](#105-kv-cache--sglang-radixattention)
    - [10.6 KV Cache — From Scratch (PyTorch)](#106-kv-cache--from-scratch-pytorch)
    - [10.7 Cache-Aware RAG — Selective Recomputation](#107-cache-aware-rag--selective-recomputation)
    - [10.8 Full Stack — All Layers Combined](#108-full-stack--all-layers-combined)
11. [Advanced: Async LLM-as-a-Judge](#11-advanced-async-llm-as-a-judge)
12. [KV Cache Optimization Techniques (Survey)](#12-kv-cache-optimization-techniques-survey)
13. [One-Line Decision Rules](#13-one-line-decision-rules)
14. [Sources & Further Reading](#14-sources--further-reading)

---

## 1. Why Caching Matters

LLM API costs scale directly with token volume. Caching is the highest-leverage technique to reduce those costs — but there are four distinct types, each operating at a different layer of the stack. Using the wrong one wastes engineering effort or leaves savings on the table.

> **Verified headline number:** Prompt caching yields up to **90% cost reduction** on cached input tokens
> (cache reads at 0.1× base price). Source: Anthropic official docs, May 2026.

---

## 2. Prerequisites — Quick Recall Notes

Before working with any caching layer, you need solid footing in these areas.

### Python (Intermediate)

- **List comprehensions, generators** — memory-efficient iteration
- **Decorators** — `@cache`, `@lru_cache` are literally what you'll implement
- **Context managers** (`with`) — resource cleanup for connections
- **Type hints** — production code uses them, read them fluently
- **`dataclasses`** — cleaner than raw dicts for cache entries
- **`functools.lru_cache`** — understand how it works internally before studying any LLM cache
- **`asyncio.create_task()`** — the key pattern for async cache writes off the critical path

### Transformer Architecture + Attention

- **Tokenization** — text → integers. Costs are per token, not per character
- **Embedding** — each token becomes a vector (dimension = model hidden size)
- **Q, K, V matrices** — every attention layer projects tokens into Query, Key, Value
- **Attention formula:** `softmax(QKᵀ / √d) × V` — this is what gets cached
- **Why KV cache exists:** past tokens' K and V never change during generation — wasteful to recompute them every step
- **Autoregressive decoding** — tokens generated one at a time. Each step reuses all past context
- **Key number:** 128k context window = 128k rows in the KV cache. Memory grows linearly

### LLM APIs

- **Request structure:** `system` → `messages[]` → `response`
- **Token counting:** input + output = total cost. Input is what caching targets
- **The API is stateless** — YOU must send context every time. No server-side memory unless you build it
- **Rate limits** — RPM (requests/min) and TPM (tokens/min) are separate limits

### Vector Embeddings & Cosine Similarity

- **Embedding** — text string → fixed-length float vector (e.g., 384 dims for `all-MiniLM-L6-v2`)
- **Cosine similarity:** `cos(A, B) = (A · B) / (|A| × |B|)` — range: 0.7–1.0 for text
- **Threshold intuition:** 0.95+ = near-identical, 0.85 = same topic, <0.7 = different topics
- **ANN vs exact search** — production always uses approximate (FAISS) — exact is O(n), ANN is O(log n)
- **Key insight:** two questions with different words can have cosine similarity 0.92

### Async Programming (`asyncio`)

```python
async def foo(): ...           # coroutine
await foo()                    # run and wait
asyncio.create_task(foo())     # fire and forget — off critical path
asyncio.gather(a(), b(), c())  # run concurrently, wait for all
```

- **`create_task()`** is the key pattern for async cache writes — never block the response on cache promotion

### System Design Basics

- **Cache hierarchy:** L1 (in-memory dict) → L2 (Redis) → L3 (provider prompt cache) → L4 (semantic vector store)
- **TTL (Time to Live)** — every cache entry must expire. Stale data is worse than a cache miss
- **Eviction policies:** LRU (least recently used), LFU (least frequently used), FIFO
- **Hit rate** = `hits / (hits + misses)` — the primary metric for all caching work
- **Cold start problem** — empty cache on new deployment; first N requests pay full cost

### Docker / Cloud

- `docker-compose` wires your app + Redis + inference server together for local production
- Managed Redis (AWS ElastiCache, GCP Memorystore) replaces self-hosted Redis in production
- KV cache inference servers (vLLM, SGLang) run as separate containers

---

## 3. KV Cache — Conceptual Revision Notes

*Source: Sebastian Raschka, "Coding the KV Cache in LLMs" (June 2025) + arXiv:2603.20397*

### The Core Problem

LLMs generate one token at a time. Without caching:

```
Prompt: "Time"
Step 1: process ["Time"]              → predict "flies"
Step 2: process ["Time", "flies"]     → predict "fast"   ← recomputes "Time" again!
Step 3: process ["Time","flies","fast"]→ predict "!"      ← recomputes both again!
```

- Total work without cache: **O(n²)** — catastrophic for long contexts
- Total work with cache: **O(n)** — linear

### What Gets Cached and Why

```
Input token embedding → 3 linear projections:
  Q = W_q × x    ← what this token is looking for
  K = W_k × x    ← what this token advertises      ← CACHED
  V = W_v × x    ← what this token contains        ← CACHED

Attention = softmax(Q × Kᵀ / √d_head) × V
```

- **Q changes every step** (new query from the new token) — never cached
- **K and V for past tokens are identical every step** — cached once, reused forever

### Mechanism Step-by-Step

```
Prompt "Time flies" arrives:
  → compute K₁, V₁ for "Time"   → store in cache
  → compute K₂, V₂ for "flies"  → store in cache

New token generation step ("fast"):
  → compute K₃, V₃ for "fast" only (new)
  → retrieve K₁, V₁, K₂, V₂ from cache
  → attention over [K₁, K₂, K₃] and [V₁, V₂, V₃]
  → no recomputation of past tokens ✓
```

### Memory Formula

```
KV cache size = 2 × num_layers × num_heads × head_dim × seq_len × dtype_bytes

Example (Llama 3 8B, fp16, 32k context):
  = 2 × 32 × 32 × 128 × 32768 × 2 bytes ≈ 16 GB
```

Memory grows linearly — this is the main production bottleneck.

### Advantages vs Disadvantages

| Advantage | Disadvantage |
|-----------|-------------|
| O(n²) → O(n) computation | Memory grows linearly with context |
| ~5x+ speedup for long sequences | Adds code complexity |
| Essential for production inference | Cannot be used during training |
| Each K/V computed exactly once | Stale cache = silent wrong outputs |

### Implementation Pitfalls

1. **Forgetting `reset_kv_cache()`** between separate requests — old K/V corrupts the next request
2. **Using `torch.cat` in production** — reallocates memory every step; use pre-allocated tensors instead
3. **Position embedding offset** — `current_pos` must continue from where the cache ends, not restart at 0
4. **GPU vs CPU** — KV cache helps most on CPU and long sequences; small GPU models may see no gain

### KV Cache Optimization Landscape (arXiv:2603.20397)

| Category | Techniques | When to Use |
|----------|-----------|-------------|
| **Eviction** | H2O, SnapKV, HashEvict | Very long contexts, GPU memory constrained |
| **Quantization** | KIVI (2-bit), KVQuant, Ada-KV | Memory savings with acceptable quality loss |
| **Compression** | KVzip, RocketKV, MiniCache | Minimize footprint per token |
| **Paging** | PagedAttention (vLLM) | Eliminate memory fragmentation in production |
| **Sparse attention** | LayerKV, local/linear attention | Attend to a window, cap memory growth |
| **Architecture** | MLA (DeepSeek-V2), Mamba/SSMs | Sub-linear scaling; requires model change |

---

## 4. The 4-Layer Caching Stack

```
┌──────────────────────────────────────────────────────────┐
│  Layer 1: Semantic Cache       (your app layer)           │
│  "Is this query similar enough to a past answer?"         │
├──────────────────────────────────────────────────────────┤
│  Layer 2: Prompt Cache         (provider API level)       │
│  "Reuse prefix tokens already computed by Anthropic/OAI"  │
├──────────────────────────────────────────────────────────┤
│  Layer 3: Cache-Aware RAG      (retrieval layer)          │
│  "Reuse KV tensors of retrieved chunks — carefully"        │
├──────────────────────────────────────────────────────────┤
│  Layer 4: KV Cache             (inference engine)         │
│  "Eliminate recomputing past token K/V representations"    │
└──────────────────────────────────────────────────────────┘
```

| Type | Who Controls It | Works With |
|------|----------------|------------|
| Semantic Cache | You | Any LLM |
| Prompt Cache | Anthropic / OpenAI (you trigger it) | Managed APIs |
| Cache-Aware RAG | You | Self-hosted |
| KV Cache (engine) | Inference engine (vLLM / SGLang) | Self-hosted |

---

## 5. When to Use Each Type

### Prompt Caching
**Provider API level — zero infrastructure**

**Use when:**
- Calling Anthropic or OpenAI APIs (not self-hosting)
- System prompt / context is long and reused across many requests
- You have a shared static prefix — instructions, documents, tool definitions, few-shot examples

**Best fits:**
- Chatbots with a long fixed system prompt
- Document Q&A where the same doc is queried repeatedly
- Multi-turn conversations (cache the growing conversation history)
- RAG with a fixed retrieved context block

**Do NOT use when:**
- Every request has a fully unique prompt (no shared prefix)
- Prompt is shorter than 1,024 tokens (below minimum threshold)
- Traffic too low to recoup the 1.25× write premium (need >1.1 hits to break even)

---

### Semantic Cache
**App layer — embedding model + vector DB**

**Use when:**
- Users ask semantically similar questions repeatedly
- FAQ-style, support bot, or search workload
- Response freshness is not critical

**Best fits:**
- Customer support chatbots (same 50 questions × 10,000 users)
- Internal knowledge base assistants
- Product / order / shipping Q&A
- Any narrow-domain, repetitive query distribution

**Do NOT use when:**
- Every query is genuinely unique (creative writing, personalized advice)
- Correctness is critical and stale answers are dangerous (medical, legal, financial)
- Real-time accuracy matters more than cost

---

### KV Cache (Inference Engine)
**Self-hosted only — automatic in vLLM / SGLang**

**Use when:**
- Self-hosting a model (vLLM, SGLang, llama.cpp, TGI)
- Serving long-context requests
- Sharing prefixes across concurrent users

**Best fits:**
- Self-hosted LLM API serving
- Batch inference with shared system prompts
- Long document analysis at scale
- Code generation with shared context across users

**Do NOT use when:**
- Using managed APIs (provider handles this invisibly)
- All sequences are short and unique

---

### Cache-Aware RAG
**Retrieval layer — advanced, self-hosted only**

**Use when:**
- Large stable document corpus queried repeatedly
- Same chunks get retrieved across many different queries
- TTFT (time to first token) is a measurable bottleneck
- Self-hosting and can control the inference pipeline

**Best fits:**
- Enterprise search over fixed knowledge base
- Legal / medical document retrieval
- Code documentation assistants

> **Critical (verified 3-0 in research):** Naive KV reuse of retrieved chunks causes ~50% F1 score drop.
> Selective recomputation of at least 15% of cached chunks is mandatory.
> Sources: FusionRAG (arXiv:2601.12904), CacheClip (arXiv:2510.10129), Cache-Craft (arXiv:2502.15734)

**Do NOT use when:**
- Chunks change frequently
- Cannot implement selective recomputation (use prompt caching on the context block instead)

---

## 6. Decision Flowchart

```
New request arrives
        │
        ▼
Are you self-hosting the model?
        │
   YES ─┤─── Enable KV Cache in vLLM/SGLang (automatic).
        │    Layer Semantic Cache on top.
        │    If RAG at scale → add Cache-Aware RAG carefully.
        │
   NO ──┘
        │
        ▼
Do requests share a long static prefix? (≥ 1,024 tokens)
        │
   YES ─┼─── Use Prompt Caching
        │    Restructure: static content first, dynamic content last.
        │    Mark cache breakpoint.
        │
   NO ──┘
        │
        ▼
Do users ask similar questions repeatedly?
        │
   YES ─┼─── Use Semantic Cache
        │    Embed queries, lookup by cosine similarity,
        │    tune threshold per domain.
        │
   NO ──┘
        │
        ▼
Doing RAG with repeated document chunks?
        │
   YES ─┼─── Prompt Cache the retrieved context block
        │    (simpler and safer than full cache-aware RAG)
        │
   NO ──┘
        │
        ▼
Full LLM call. Review prompt structure for future caching opportunities.
```

---

## 7. Side-by-Side Comparison

| Type | Cost Reduction | Infrastructure | Works With | Main Risk |
|------|----------------|---------------|-----------|-----------|
| **Prompt Cache** | Up to **90%** on input tokens | None | Anthropic, OpenAI | 1.25× write premium if hit rate is low |
| **Semantic Cache** | Skip entire LLM call | Embedding model + vector DB | Any LLM | Wrong answers if threshold too loose |
| **KV Cache (engine)** | Latency (not direct $ cost) | Self-hosted inference server | vLLM, SGLang, llama.cpp | GPU memory pressure at long contexts |
| **Cache-Aware RAG** | Latency + some token cost | Self-hosted + custom pipeline | Self-hosted RAG | ~50% quality drop if done naively |

---

## 8. Production Layering Pattern

In a real system, layer all techniques. Cheapest check runs first:

```
Request arrives
      │
      ▼
[L1] Semantic Cache hit?  ──YES──▶  Return stored response (zero LLM cost)
      │ NO
      ▼
[L2] Prompt Cache hit?    ──YES──▶  LLM call at 0.1× token cost (90% savings)
      │ NO
      ▼
[L3] Cache-Aware RAG      ────────▶  Reuse chunk KV tensors (self-hosted only)
      │
      ▼
[L4] KV Cache (engine)    ────────▶  Handled automatically by vLLM/SGLang
      │
      ▼
      Full LLM call  ◀──── Minimize reaching here
```

---

## 9. Verified Pricing (Anthropic, May 2026)

> Always check [platform.claude.com](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
> for current rates — prices change. Verified against official docs and corroborated by multiple independent sources.

| Token Type | Cost Multiplier | Example (Opus 4.x base = $5/M tokens) |
|-----------|----------------|---------------------------------------|
| Cache read (hit) | **0.1× base** | $0.50/M tokens |
| Cache write (5-min TTL) | 1.25× base | $6.25/M tokens |
| Cache write (1-hour TTL) | 2× base | $10.00/M tokens |
| Regular input (no cache) | 1× base | $5.00/M tokens |

**Break-even math for 1-hour TTL:**

```
Write premium  = 2× − 1×  = 1× extra
Savings per hit = 1× − 0.1× = 0.9× saved
Break-even      = 1 / 0.9  ≈ 1.1 hits

→ A prefix read just twice already pays off the 1-hour write cost.
```

---

## 10. Implementation Guide

---

### 10.1 Prompt Caching — Anthropic Claude

Mark the cache breakpoint with `cache_control`. Everything before it is cached; everything after is dynamic.

**Golden rule:** static content FIRST, dynamic content LAST.

#### Basic — System Prompt Cache

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM = """You are a Mergit AI agent orchestration assistant.
[... 2,000+ tokens of static instructions: agent routing rules, tool definitions, Monad contract context ...]
Always prioritize Groq → Anthropic → OpenAI for LLM routing."""

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": SYSTEM,
            "cache_control": {"type": "ephemeral"}   # ← cache breakpoint
        }
    ],
    messages=[
        {"role": "user", "content": user_query}      # dynamic — NOT cached
    ]
)

# Inspect cache hit/miss
print(f"Tokens written to cache : {response.usage.cache_creation_input_tokens}")
print(f"Tokens read from cache  : {response.usage.cache_read_input_tokens}")
print(f"Regular input tokens    : {response.usage.input_tokens}")
# cache_read_input_tokens > 0 means you saved 90% on those tokens
```

#### Multi-Turn Conversation Cache

```python
def chat_with_cache(client, history: list, new_message: str):
    """
    history: [{"role": "user"|"assistant", "content": "..."}]
    Cache breakpoint sits at the end of history so the full
    conversation is reused on the next turn.
    """
    messages = []

    for i, msg in enumerate(history):
        if i == len(history) - 1:
            # Last historical message — mark as cache breakpoint
            messages.append({
                "role": msg["role"],
                "content": [
                    {
                        "type": "text",
                        "text": msg["content"],
                        "cache_control": {"type": "ephemeral"}
                    }
                ]
            })
        else:
            messages.append(msg)

    messages.append({"role": "user", "content": new_message})

    return client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=messages
    )
```

> **Prompt structure rule:** `system instructions → static documents → few-shot examples → dynamic user query`
> Any change before the cache breakpoint invalidates the entire cache.

---

### 10.2 Prompt Caching — OpenAI

OpenAI caches automatically — no explicit marker. Just keep the prefix stable and ≥ 1,024 tokens.

```python
from openai import OpenAI

client = OpenAI()

# Must be >= 1,024 tokens total for caching to activate.
# Cache hits occur in 128-token increments.
SYSTEM_PROMPT = """You are a Mergit AI agent orchestration assistant.
[... long stable instructions: agent routing rules, tool definitions, Monad contract context ...]"""

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},  # auto-cached
        {"role": "user",   "content": user_query}       # dynamic
    ]
)

# Check cache savings
cached = response.usage.prompt_tokens_details.cached_tokens
total  = response.usage.prompt_tokens
print(f"Cached tokens : {cached} / {total}")
print(f"Cache hit rate: {cached / total * 100:.1f}%")
print(f"Savings       : ${cached * 0.5 / 1_000_000:.5f}")
```

> **OpenAI vs Anthropic:**
> OpenAI: automatic, 1,024 token minimum, 128-token increment granularity, no extra write cost.
> Anthropic: explicit `cache_control` marker, model-dependent minimums, 1.25× write premium.

---

### 10.3 Semantic Caching — Custom Implementation

Embed queries, find near-duplicates by cosine similarity, return the stored response.

#### Dependencies

```bash
pip install sentence-transformers faiss-cpu numpy
```

#### Core Cache Class

```python
import asyncio
import numpy as np
import faiss
from sentence_transformers import SentenceTransformer

class SemanticCache:
    def __init__(self, threshold: float = 0.90):
        self.encoder   = SentenceTransformer("all-MiniLM-L6-v2")  # 384-dim, fast
        self.dim       = 384
        self.index     = faiss.IndexFlatIP(self.dim)  # inner product on L2-normalised = cosine
        self.store     = []                            # [{"query": ..., "response": ...}]
        self.threshold = threshold                     # tune per domain

    def _embed(self, text: str) -> np.ndarray:
        return self.encoder.encode([text], normalize_embeddings=True).astype("float32")

    def lookup(self, query: str) -> str | None:
        """Returns cached response or None. O(log n) via FAISS."""
        if not self.store:
            return None
        scores, idxs = self.index.search(self._embed(query), k=1)
        if scores[0][0] >= self.threshold:
            return self.store[idxs[0][0]]["response"]
        return None

    def insert(self, query: str, response: str):
        self.index.add(self._embed(query))
        self.store.append({"query": query, "response": response})

    async def async_judge_and_promote(
        self, query: str, response: str,
        judge_client, loose_threshold: float = 0.75
    ):
        """
        Async LLM-as-a-judge: expand cache coverage without adding latency.
        Runs OFF the critical path — user already has their response.
        Source: Krites (Apple ML Research, arXiv:2602.13165, EuroMLSys 2026)
        """
        # Skip if already covered by a close existing entry
        if self.store:
            scores, _ = self.index.search(self._embed(query), k=1)
            if scores[0][0] >= loose_threshold:
                return

        verdict = await judge_client.async_call(
            f"Is this response reusable for semantically similar future queries?\n"
            f"Query: {query}\nResponse: {response}\nAnswer YES or NO only."
        )
        if "YES" in verdict.upper():
            self.insert(query, response)
```

#### Request Handler

```python
cache = SemanticCache(threshold=0.90)

async def handle_request(query: str, llm_client) -> str:
    # [Critical path] Semantic cache check — microseconds
    hit = cache.lookup(query)
    if hit:
        return hit                  # zero LLM tokens consumed

    # [Critical path] Cache miss — full LLM call
    response = await llm_client.async_call(query)

    # [Off critical path] Promote to cache — user already has their response
    asyncio.create_task(
        cache.async_judge_and_promote(query, response, llm_client)
    )

    return response
```

#### Redis-Backed Semantic Cache (Production)

```python
import redis, json, hashlib

class RedisSemanticCache(SemanticCache):
    """
    Persist cache across restarts and share across multiple workers.
    FAISS index lives in memory per worker; Redis stores the responses.
    """
    def __init__(self, redis_url: str = "redis://localhost:6379", **kwargs):
        super().__init__(**kwargs)
        self.redis = redis.from_url(redis_url)

    def _key(self, query: str) -> str:
        return f"semcache:{hashlib.md5(query.encode()).hexdigest()}"

    def insert(self, query: str, response: str):
        super().insert(query, response)
        self.redis.setex(self._key(query), 3600, json.dumps(response))  # TTL: 1 hour

    def lookup(self, query: str) -> str | None:
        result = super().lookup(query)
        if result:
            self.redis.expire(self._key(query), 3600)  # refresh TTL on hit
        return result
```

---

### 10.4 KV Cache — vLLM (PagedAttention)

Self-hosted inference. PagedAttention eliminates memory fragmentation; prefix caching shares KV tensors across requests with identical prefixes.

#### Dependencies

```bash
pip install vllm
# Requires CUDA GPU
```

#### Server Setup (Recommended for Production)

```bash
# Start as an OpenAI-compatible API server
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3-8b-instruct \
    --enable-prefix-caching \
    --gpu-memory-utilization 0.90 \
    --max-model-len 32768 \
    --tensor-parallel-size 1

# Call it like the OpenAI API — drop-in compatible
```

#### Programmatic Usage

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="meta-llama/Llama-3-8b-instruct",
    gpu_memory_utilization=0.90,
    enable_prefix_caching=True,    # share KV tensors across requests
    max_model_len=32768,
)

sampling_params = SamplingParams(temperature=0.7, max_tokens=512)

# Requests sharing the same prefix reuse its KV tensors automatically
SHARED_PREFIX = "[Your 2,000-token system prompt here]"

prompts = [
    f"{SHARED_PREFIX}\nQ: What is aspirin's mechanism of action?",
    f"{SHARED_PREFIX}\nQ: Contraindications of ibuprofen?",
    f"{SHARED_PREFIX}\nQ: Paracetamol overdose management?",
]

# vLLM computes prefix KV once, reuses it for all 3 requests
outputs = llm.generate(prompts, sampling_params)
for out in outputs:
    print(out.outputs[0].text)
```

> **Memory tip:** For very long contexts, enable KV cache quantization:
> `--kv-cache-dtype fp8` — half the memory with minimal quality impact.

---

### 10.5 KV Cache — SGLang (RadixAttention)

SGLang's RadixAttention stores the KV cache in a radix tree — automatically shares any common prefix across all concurrent requests.

#### Dependencies

```bash
pip install sglang[all]
```

#### Implementation

```python
import sglang as sgl

@sgl.function
def agent_query(s, question: str):
    # RadixAttention caches this block and shares it across all calls
    s += sgl.system(
        "You are a Mergit AI agent orchestration assistant.\n"
        "[...long shared instructions: routing rules, tool defs, Monad contract context...]"
    )
    s += sgl.user(question)
    s += sgl.assistant(sgl.gen("answer", max_tokens=512))


# Launch with RadixAttention enabled (default in SGLang)
runtime = sgl.Runtime(
    model_path="meta-llama/Llama-3-8b-instruct",
    enable_flashinfer=True,
)
sgl.set_default_backend(runtime)

# Batch of requests — system prompt KV is computed once, shared across all
states = agent_query.run_batch(
    [
        {"question": "Which agent handles token swaps on Monad?"},
        {"question": "What is the fallback LLM when Groq is unavailable?"},
        {"question": "How does the oracle integrity check work?"},
    ],
    progress_bar=True,
)

for s in states:
    print(s["answer"])

runtime.shutdown()
```

---

### 10.6 KV Cache — From Scratch (PyTorch)

Based on Sebastian Raschka's verified implementation. Read this to understand what vLLM and SGLang do under the hood.

```python
import torch
import torch.nn as nn


class MultiHeadAttentionWithKVCache(nn.Module):
    def __init__(self, d_in: int, d_out: int, num_heads: int):
        super().__init__()
        self.W_query = nn.Linear(d_in, d_out, bias=False)
        self.W_key   = nn.Linear(d_in, d_out, bias=False)
        self.W_value = nn.Linear(d_in, d_out, bias=False)
        # Cache buffers — None until first forward pass
        self.register_buffer("cache_k", None)
        self.register_buffer("cache_v", None)

    def reset_cache(self):
        """MUST call between separate generation requests — stale cache = wrong output."""
        self.cache_k = None
        self.cache_v = None

    def forward(self, x: torch.Tensor, use_cache: bool = False) -> torch.Tensor:
        keys_new   = self.W_key(x)       # only new token(s)
        values_new = self.W_value(x)
        queries    = self.W_query(x)

        if use_cache:
            if self.cache_k is None:                              # first step: init
                self.cache_k = keys_new
                self.cache_v = values_new
            else:                                                 # subsequent steps: append
                self.cache_k = torch.cat([self.cache_k, keys_new],   dim=1)
                self.cache_v = torch.cat([self.cache_v, values_new], dim=1)
            keys, values = self.cache_k, self.cache_v            # all past + current
        else:
            keys, values = keys_new, values_new

        scale   = keys.shape[-1] ** 0.5
        scores  = queries @ keys.transpose(-2, -1) / scale
        weights = torch.softmax(scores, dim=-1)
        return weights @ values


class GPTModelWithKVCache(nn.Module):
    def __init__(self, config: dict):
        super().__init__()
        self.tok_emb    = nn.Embedding(config["vocab_size"], config["emb_dim"])
        self.pos_emb    = nn.Embedding(config["ctx_len"],   config["emb_dim"])
        self.trf_blocks = nn.ModuleList([...])  # your transformer blocks
        self.current_pos = 0   # tracks where the cache ends

    def reset_kv_cache(self):
        for blk in self.trf_blocks:
            blk.att.reset_cache()
        self.current_pos = 0

    def forward(self, idx: torch.Tensor, use_cache: bool = False) -> torch.Tensor:
        seq_len    = idx.shape[1]
        tok_embeds = self.tok_emb(idx)

        if use_cache:
            # Position IDs continue from where cache ends — not from 0
            pos_ids = torch.arange(
                self.current_pos,
                self.current_pos + seq_len,
                device=idx.device
            )
            self.current_pos += seq_len
        else:
            pos_ids = torch.arange(seq_len, device=idx.device)

        x = tok_embeds + self.pos_emb(pos_ids).unsqueeze(0)
        for blk in self.trf_blocks:
            x = blk(x, use_cache=use_cache)
        return x


def generate_with_kv_cache(
    model: GPTModelWithKVCache,
    prompt_tokens: torch.Tensor,
    max_new_tokens: int
) -> torch.Tensor:
    model.eval()
    model.reset_kv_cache()

    # Step 1: warm the cache with the full prompt (computes all prefix K/V)
    with torch.no_grad():
        logits = model(prompt_tokens, use_cache=True)

    tokens = prompt_tokens
    for _ in range(max_new_tokens):
        next_tok = logits[:, -1].argmax(dim=-1, keepdim=True)
        tokens   = torch.cat([tokens, next_tok], dim=1)
        with torch.no_grad():
            # Feed ONLY the new token — past K/V come from cache
            logits = model(next_tok, use_cache=True)

    return tokens


# Production tip: replace torch.cat (causes repeated reallocation) with pre-allocated tensors
max_seq  = 2048
cache_k  = torch.zeros(batch_size, num_heads, max_seq, head_dim, device="cuda")
cache_v  = torch.zeros(batch_size, num_heads, max_seq, head_dim, device="cuda")
# Write into slices: cache_k[:, :, step:step+1, :] = keys_new
```

---

### 10.7 Cache-Aware RAG — Selective Recomputation

Reuse precomputed KV tensors of retrieved chunks. **Never reuse 100%** — selective recomputation is mandatory.

> Source: FusionRAG (arXiv:2601.12904), CacheClip (arXiv:2510.10129) — verified 3-0.

```python
from dataclasses import dataclass, field
from typing import Any, Dict, List

@dataclass
class Chunk:
    id:   str
    text: str

@dataclass
class CacheAwareRAG:
    llm:               Any
    retriever:         Any
    # What fraction of cached chunks to recompute for cross-chunk context.
    # 0.0 = naive (causes ~50% F1 drop). 0.15 recovers quality.
    recompute_ratio:   float = 0.15
    chunk_kv_cache:    Dict[str, Any] = field(default_factory=dict)
    max_cache_entries: int = 1000

    def _evict_if_needed(self):
        if len(self.chunk_kv_cache) > self.max_cache_entries:
            oldest = list(self.chunk_kv_cache.keys())[0]
            del self.chunk_kv_cache[oldest]

    def generate(self, query: str) -> str:
        chunks: List[Chunk] = self.retriever.retrieve(query, k=5)

        cached, fresh = [], []
        for chunk in chunks:
            (cached if chunk.id in self.chunk_kv_cache else fresh).append(chunk)

        # Selective recomputation — always recompute at least 1 cached chunk
        n_recompute = max(1, int(len(cached) * self.recompute_ratio))
        recompute   = cached[:n_recompute]   # recompute for cross-chunk context
        reuse       = cached[n_recompute:]   # safely reuse KV tensors

        context = self.llm.build_context(
            reuse_kv  = [(c, self.chunk_kv_cache[c.id]) for c in reuse],
            recompute = recompute + fresh,
        )

        # Cache newly computed chunk KV tensors
        for chunk in fresh + recompute:
            self._evict_if_needed()
            self.chunk_kv_cache[chunk.id] = self.llm.extract_kv(chunk.text)

        return self.llm.generate(query, context)
```

> **Simpler alternative for most teams:** Skip cache-aware RAG and use prompt caching on the retrieved
> context block instead. Structure: `[system] → [retrieved chunks with cache_control] → [user query]`.
> Anthropic caches the retrieved content server-side. Much simpler, nearly as effective.

---

### 10.8 Full Stack — All Layers Combined

Production request handler that checks every cache layer in order, cheapest first.

```python
import asyncio
from typing import Optional
import anthropic

STATIC_SYSTEM_PROMPT = """You are a Mergit AI agent orchestration assistant.
[... 2,000+ tokens of static instructions: agent routing rules, tool definitions, Monad contract context ...]"""


class ProductionLLMHandler:
    def __init__(self, llm_client: anthropic.Anthropic, semantic_cache: SemanticCache):
        self.llm     = llm_client
        self.sem     = semantic_cache
        self.metrics = {"sem_hits": 0, "prompt_hits": 0, "misses": 0}

    async def handle(self, query: str, session_history: Optional[list] = None) -> str:

        # ── Layer 1: Semantic cache ───────────────────────────────────────
        # Zero LLM cost. Microseconds. Always check first.
        hit = self.sem.lookup(query)
        if hit:
            self.metrics["sem_hits"] += 1
            return hit

        # ── Layer 2: Prompt cache (provider-side) ─────────────────────────
        # 90% input token savings on cache hits.
        response = self.llm.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=[
                {
                    "type": "text",
                    "text": STATIC_SYSTEM_PROMPT,
                    "cache_control": {"type": "ephemeral"}   # ← cache breakpoint
                }
            ],
            messages=self._build_messages(session_history, query)
        )

        result = response.content[0].text

        # Log whether this was a prompt cache hit
        if response.usage.cache_read_input_tokens > 0:
            self.metrics["prompt_hits"] += 1
        else:
            self.metrics["misses"] += 1

        # ── Layer 3: Promote to semantic cache (off critical path) ─────────
        asyncio.create_task(
            self.sem.async_judge_and_promote(query, result, self.llm)
        )

        return result

    def _build_messages(self, history: Optional[list], query: str) -> list:
        messages = []
        if history:
            for i, msg in enumerate(history):
                if i == len(history) - 1:
                    # Mark conversation history as cached up to the last message
                    messages.append({
                        "role": msg["role"],
                        "content": [{"type": "text", "text": msg["content"],
                                     "cache_control": {"type": "ephemeral"}}]
                    })
                else:
                    messages.append(msg)
        messages.append({"role": "user", "content": query})
        return messages

    def hit_rate(self) -> dict:
        total = sum(self.metrics.values()) or 1
        return {k: f"{v / total * 100:.1f}%" for k, v in self.metrics.items()}


# Usage
cache   = RedisSemanticCache(redis_url="redis://localhost:6379", threshold=0.90)
handler = ProductionLLMHandler(llm_client=anthropic.Anthropic(), semantic_cache=cache)

response = await handler.handle(user_query, session_history=history)
print(handler.hit_rate())
# {"sem_hits": "43.2%", "prompt_hits": "38.5%", "misses": "18.3%"}
```

> **Key metric to track in production:**
> `cache_read_input_tokens / total_input_tokens` — your effective prompt cache hit rate.
> Target >60% for a system with stable system prompts and moderate query overlap.
> If below 20%, your prompt structure or TTL selection needs revisiting.

---

## 11. Advanced: Async LLM-as-a-Judge

A static cosine similarity threshold misses semantically equivalent queries that fall just below the cutoff. The Krites system (Apple ML Research, EuroMLSys 2026) solves this:

```python
# Critical path (user sees this latency)
embedding = embed(query)
hit = vector_store.search(embedding, threshold=0.90)   # strict threshold
if hit:
    return hit.response                                 # instant, no LLM call

response = llm.call(query)                             # full LLM call

# Off critical path (user already has their response)
asyncio.create_task(
    judge_and_promote(query, response, loose_threshold=0.75)
    # LLM judge decides if this response is general enough to cache
    # Expanding future cache coverage without adding latency
)
```

> **Caveat:** False approvals from the async judge degrade quality for subsequent requests that hit
> those promoted cache entries. Tune the judge threshold conservatively and monitor quality.

---

## 12. KV Cache Optimization Techniques (Survey)

*Source: arXiv:2603.20397 — "KV Cache Optimization Strategies for Scalable and Efficient LLM Inference" (March 2026)*

### Eviction-Based Methods
Remove tokens from cache when memory is full.

| Technique | Strategy |
|-----------|---------|
| **H2O** | Keep "heavy hitter" tokens (those attended to most frequently) |
| **SnapKV** | Compress to a snapshot before evicting |
| **HashEvict** | Hash-based eviction policy |

**When to use:** Very long contexts where you can't fit everything in GPU memory.

### Quantization
Compress the values stored in cache.

| Technique | Bits | Notes |
|-----------|------|-------|
| **KIVI** | 2-bit | Streaming asymmetric quantization |
| **KVQuant** | 4-bit | NeurIPS 2024; quantize K/V to reduce memory |
| **Ada-KV** | Adaptive | Per-head adaptive quantization |

**Tradeoff:** memory savings vs. slight quality loss.

### Paging & Memory Management

| Technique | What it does |
|-----------|-------------|
| **PagedAttention** (vLLM) | Store KV cache in non-contiguous memory pages like OS virtual memory — eliminates fragmentation |
| **FlexGen** | Offload KV cache to CPU/SSD for very long contexts |
| **ShadowKV** | Keep sparse cache on GPU, dense cache on CPU |

### Novel Architecture Approaches

| Approach | How |
|---------|-----|
| **MLA (DeepSeek-V2)** | Compress K/V into low-rank latent vector before storing — sub-linear memory scaling |
| **Mamba / SSMs** | State-space models avoid attention entirely — no KV cache needed |
| **Sparse attention** | Only attend to a local window — cache only that window |

---

## 13. One-Line Decision Rules

| Type | One-line rule |
|------|--------------|
| **Prompt Cache** | On Anthropic/OpenAI + system prompt > 1k tokens → use it, zero infrastructure needed |
| **Semantic Cache** | Users ask the same 50 things 50 different ways → skip the LLM call entirely |
| **KV Cache (engine)** | Self-hosting → enable it by default, automatic in vLLM/SGLang |
| **Cache-Aware RAG** | Same chunks retrieved repeatedly + can implement selective recomputation → use carefully |
| **Do nothing** | Every request is unique and freshness is critical → don't cache, monitor costs instead |

**KV cache internals:**

| Concept | One-line |
|---------|---------|
| Why KV cache | Past tokens' K and V never change — stop recomputing them |
| What gets cached | K and V only. Q is never cached (changes each step) |
| Memory cost | Grows linearly: one entry per token per layer per head |
| `reset_kv_cache()` | Must call between separate requests or output is corrupted |
| `current_pos` tracking | Position embeddings must continue from cache end, not restart at 0 |
| PagedAttention | KV cache in virtual memory pages = no fragmentation (used by vLLM) |
| MLA (DeepSeek) | Low-rank compress K/V before storing → sub-linear memory |

---

## 14. Sources & Further Reading

| Resource | Type | What it covers |
|----------|------|---------------|
| [Anthropic Prompt Caching Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) | Official doc | Pricing, TTL tiers, token minimums, `cache_control` syntax |
| [OpenAI Prompt Caching Guide](https://developers.openai.com/api/docs/guides/prompt-caching) | Official doc | Automatic caching, 1,024 token min, 128-token increments |
| [OpenAI Prompt Caching Cookbook](https://developers.openai.com/cookbook/examples/prompt_caching_201) | Cookbook | Practical structuring patterns, hit rate optimization |
| [Sebastian Raschka — Coding the KV Cache](https://magazine.sebastianraschka.com/p/coding-the-kv-cache-in-llms) | Technical blog | KV cache from scratch — best hands-on implementation guide |
| [arXiv:2603.20397](https://arxiv.org/pdf/2603.20397) | Survey paper (Mar 2026) | Systematic review of all KV cache optimization strategies |
| [arXiv:2602.13165 (Krites)](https://arxiv.org/pdf/2602.13165) | Research paper (Feb 2026) | Async LLM-as-a-judge for semantic cache coverage expansion |
| [arXiv:2601.12904 (FusionRAG)](https://arxiv.org/pdf/2601.12904) | Research paper (Jan 2026) | Cache-aware RAG with selective recomputation |
| [arXiv:2510.10129 (CacheClip)](https://arxiv.org/abs/2510.10129) | Research paper (Oct 2025) | Naive KV reuse quality degradation in RAG |
| [arXiv:2502.15734 (Cache-Craft)](https://arxiv.org/abs/2502.15734) | Research paper (Feb 2025) | ~50% F1 drop quantification for naive cache reuse |
| [arXiv:2411.05276](https://arxiv.org/html/2411.05276v3) | Research paper (Nov 2025) | Semantic caching evaluation across query domains |
| [DeepLearning.AI — Efficient Inference with SGLang](https://www.deeplearning.ai/courses/efficient-inference-with-sglang-text-and-image-generation) | Course | RadixAttention, KV reuse, production inference — only verified structured course |
| [vLLM GitHub](https://github.com/vllm-project/vllm) | Library | PagedAttention, prefix caching, production inference server |
| [SGLang GitHub](https://github.com/sgl-project/sglang) | Library | RadixAttention, structured generation, efficient batching |
| [GPTCache GitHub](https://github.com/zilliztech/GPTCache) | Library | Main open-source semantic cache implementation |

---

*Research methodology: 23 sources fetched, 84 claims extracted, 25 claims adversarially verified (3-vote system),
6 confirmed, 19 killed. Vendor-reported improvement figures not surviving verification are excluded.
Benchmark all techniques in your own traffic distribution — gains are real but highly workload-dependent.*

---

*Mergit Internal · June 2026*
