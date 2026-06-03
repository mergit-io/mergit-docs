# Mergit — Product Requirements Document

**Version:** 0.2.0-draft  
**Organization:** [mergit-io](https://github.com/mergit-io)  
**Status:** Planning  
**Last Updated:** 2026-06-03  

---

## Table of Contents

1. [Product Vision](#1-product-vision)
2. [Problem Statement](#2-problem-statement)
3. [Goals and Success Metrics](#3-goals-and-success-metrics)
4. [User Personas](#4-user-personas)
5. [Core Feature Set](#5-core-feature-set)
6. [System Architecture](#6-system-architecture)
7. [Service Specifications](#7-service-specifications)
8. [Database Design](#8-database-design)
9. [Caching Strategy](#9-caching-strategy)
10. [Smart Contract Specifications](#10-smart-contract-specifications)
11. [LangGraph Orchestration Design](#11-langgraph-orchestration-design)
12. [API Design](#12-api-design)
13. [Security and Trust Model](#13-security-and-trust-model)
14. [Infrastructure and Deployment](#14-infrastructure-and-deployment)
15. [Repository and Contribution Strategy](#15-repository-and-contribution-strategy)
16. [Monad Blockchain Setup](#16-monad-blockchain-setup)
17. [Delivery Phases and Roadmap](#17-delivery-phases-and-roadmap)
18. [Open Questions and Decisions Log](#18-open-questions-and-decisions-log)

---

## 1. Product Vision

### 1.1 What We Are Building

**Mergit** is an **Agent Economy Platform** — a production-grade AI agent workspace where agents complete real software development tasks and every action generates verifiable, tamper-proof on-chain proof of work on the **Monad blockchain**.

Mergit upgrades the traditional AI agent loop — "receive goal → plan → execute tools → return output" — into an economy of accountable, identity-bearing agents whose work history, reputation, and capabilities are cryptographically verifiable by anyone.

### 1.2 One-Line Definition

> An AI agent workspace where agents complete real dev and GitHub tasks and generate on-chain proof of their work, identity, reputation, and accountability.

### 1.3 Background

Mergit is a full-scale rebuild of an internal prototype called **omniBox** — a Python/FastAPI + SQLite agent orchestrator that proved the core execution model (multi-agent DAG execution, GitHub automation, LLM-driven task decomposition). omniBox had no blockchain layer, no identity system, no caching strategy, and no reputation mechanism. Mergit takes the validated execution model and re-architects it at production scale with Rust-primary services, a LangGraph orchestration engine, and a Monad on-chain layer.

### 1.4 Initial Deployment Context

Mergit launches on the Monad blockchain with the theme: *give agents identity, ownership, proof, coordination, and reputation — agents should be more than tool executors*. All architectural decisions are made for production scale. Launch constraints (Docker Compose instead of Kubernetes, HMAC instead of mTLS for internal auth in Phases 1–4) are documented in prd-launch-constraints.md and will be replaced in Phase 5. The launch event is the starting line, not the finish line.

---

## 2. Problem Statement

### 2.1 The Gap in Current AI Agent Platforms

Existing AI agent frameworks (AutoGPT, CrewAI, LangChain agents, omniBox) solve agent *execution* but leave four fundamental problems unsolved:

**Problem 1 — No Verifiability**  
When an agent claims it created a PR, reviewed code, or completed a task, there is no cryptographic proof. The output is only as trustworthy as the infrastructure that generated it. No third party can independently verify that specific work was done by a specific agent at a specific time.

**Problem 2 — No Persistent Identity**  
Agents are ephemeral. Every run spawns a stateless executor with no identity, no history, no credentials that follow it across runs. There is no concept of "this agent is the same agent that successfully completed 200 prior tasks."

**Problem 3 — No Reputation or Accountability**  
Because agents have no identity and their work is unverified, there is no basis for reputation. Users cannot choose "a trusted agent" over "an untested agent." Bad agents (high failure rate, incorrect outputs, hallucinated tool calls) are indistinguishable from good ones.

**Problem 4 — No Economic Accountability**  
Agents face no financial consequence for low-quality, harmful, or fraudulent work. Without skin in the game, an agent that destroys a production database costs its operator nothing beyond a restart. Bad actors can re-register under a new DID and immediately resume operating. There is no mechanism for slashing dishonest agents or rewarding consistently reliable ones with economic incentives. Reputation scores alone cannot sustain trust at scale when the cost of abandoning a tarnished identity is zero.

### 2.2 What Mergit Solves

| Problem | Mergit Solution |
|---------|----------------|
| No verifiability | Every completed task records a `resultHash` on-chain via `ProofOfWork.sol` — tamper-proof, auditable by any observer |
| No persistent identity | Every agent receives a W3C DID (`did:mergit:<uuid>`) and an ERC-721 soulbound NFT (`AgentPassport.sol`) encoding its capability hash, task stats, and registration timestamp |
| No reputation | `ReputationRegistry.sol` maintains a live on-chain composite score per agent, updated by an oracle after each proof batch; scores are verifiable against off-chain computation receipts |
| No economic accountability | StakeRegistry.sol requires agents to post MON stake before accepting high-risk tasks; verified fraud triggers automated stake slashing via ChallengeManager.sol; honest challengers are rewarded |

---

## 3. Goals and Success Metrics

### 3.1 Phase Goals

| Phase | Goal | Success Metric |
|-------|------|---------------|
| Phase 0 | Monorepo foundation and dev environment | `docker compose up` brings up all 4 services (api-gateway, orchestration, accounts-service, blockchain-indexer) + PG + Redis with no errors; CI pipelines pass on empty scaffold |
| Phase 1 | Core agent execution with LangGraph | End-to-end goal execution: submit "create a PR for X" → agents decompose, execute, return result; <30s planning latency |
| Phase 2 | On-chain identity + proof of work | Every completed goal records a proof on Monad testnet within 60 seconds of completion; identity NFT minted on agent registration |
| Phase 3 | Reputation scoring | Reputation scores update on-chain within 10 minutes of batch window; badge issued for 5 consecutive successful proofs |
| Phase 4 | Production frontend | Users can submit goals, watch live SSE feed, approve interrupt prompts, view agent passport and on-chain proof links |
| Phase 5 | Production hardening | p95 latency < 200ms on API gateway; LLM cache hit rate > 60%; system handles 100 concurrent active goals |

### 3.2 Business Metrics (Post-Launch)

- **Weekly Active Goals (WAG)**: number of goals submitted per week
- **Proof Recording Rate**: percentage of completed goals that successfully land a proof on-chain (target: >99%)
- **LLM Cost per Goal**: average USD spent on LLM API calls per completed goal (caching drives this down)
- **Agent Reputation Distribution**: shape of the score distribution across registered agents (ideally bell-curve, not all-at-top)
- **Time to First Proof**: from goal submission to first on-chain proof (target: <2 minutes for simple goals)

---

## 4. User Personas

### 4.1 Developer / Power User

**Who:** A software engineer who wants to automate repetitive GitHub/dev tasks (triage issues, draft PRs, generate documentation, run code analysis).

**Needs:**
- Submit a high-level goal in plain English and get a real result (not a simulated output)
- See exactly what the agent did at each step (tool calls, reasoning, outputs)
- Approve PRs and credential requests without leaving the Mergit UI
- Trust that the agent result is genuine (on-chain proof link as evidence)
- Structured execution traces showing tool names, inputs (redacted), outputs (truncated), timing, and approval records — not raw LLM reasoning

**Pain point with current tools:** Existing agents hallucinate tool calls and produce unverifiable outputs. No audit trail.

### 4.2 Team Lead / Technical Manager

**Who:** Manages a team where repetitive tasks can be delegated to AI agents. Needs accountability.

**Needs:**
- See a history of all goals run by agents on team repos
- Know which agents are reliable (reputation leaderboard)
- Download an audit report for any completed workflow
- Set capability restrictions on agents (which tools and repos they can access)

**Pain point with current tools:** No way to audit what the agent did or didn't do. No accountability layer.

### 4.3 Blockchain / DeFi Developer (Early Adopters)

**Who:** Evaluating Mergit as a demonstration of Monad's use cases for AI agent accountability.

**Needs:**
- See on-chain proofs recorded in real time during agent execution
- Verify that proof hashes match actual agent outputs
- Understand the oracle trust model and anti-manipulation safeguards
- Explore AgentPassport NFTs and reputation scores via block explorer

**Pain point they're evaluating:** Most AI agent demos have zero blockchain substance — vague "we log to chain" claims with no cryptographic integrity.

---

## 5. Core Feature Set

### 5.1 Agent Execution Engine

**Goal Decomposition**  
Users submit a natural-language goal. The orchestration engine (LangGraph) decomposes it into a directed acyclic graph (DAG) of tasks, each assigned to a specialist agent role with defined inputs, outputs, and dependencies.

**Specialist Agent Roles**
| Role | Responsibility | Primary LLM |
|------|---------------|-------------|
| Researcher | Web search, GitHub exploration, context gathering | LLaMA 4 Maverick (Groq) |
| Writer | Documentation, summaries, Mermaid diagrams, PR descriptions | LLaMA 3.3 70B (Groq) |
| Coder | Code generation, debugging, Python execution in sandboxed environment | Claude Sonnet (Anthropic) |
| Integrator | GitHub operations: PR creation, branch management, issue handling, commenting | LLaMA 3.3 70B (Groq) |

**Agent Context Management**
Agents are not recreated from scratch on every goal. Base context (role prompt, capability set, tool catalog) is loaded once per agent instance and cached in-process with a 5-minute TTL. Goal-specific context (goal description, task inputs, retrieved memories) is injected per execution. Agents are refreshed only when their capability set changes or the context TTL expires, eliminating cold-start overhead per goal.

**Parallel Execution**  
Tasks with no unmet dependencies execute in parallel via LangGraph's `Send` API. The orchestrator continuously identifies ready tasks as dependencies resolve.

**Dynamic Replanning**  
When a task fails, the orchestrator rewrites the DAG subgraph with failure context injected into the planner prompt. Up to 2 automatic replanning cycles before escalating to human.

**Self-Healing**  
Consecutive failures across tasks trigger an automatic issue-filing and fix-spawning workflow (inherited from omniBox, re-implemented in LangGraph as a `self_heal_node`).

### 5.2 Tool Ecosystem

| Tool | Category | Description |
|------|----------|-------------|
| `web_search` | Research | Brave/Serper search with Redis-cached results (1h TTL) |
| `github_ops` | Integration | General GitHub API: read repos, issues, files, branches |
| `github_pr` | Integration | Create PRs, post comments, review diffs |
| `file_ops` | Execution | Read/write/delete files in agent workspace |
| `code_exec` | Execution | Python subprocess in isolated sandbox container (30s timeout) |
| `http_request` | Integration | Generic HTTP client for external APIs |
| `spawn_goal` | Orchestration | Create child goals (hierarchical goal decomposition) |
| `wait_webhook` | Integration | Block task until a webhook event arrives |
| `credential_request` | Human-in-loop | Request human to provide credentials (triggers UI interrupt) |

All tools are implemented as **FastAPI tool-server sidecars** in Python. Rust services call tool servers via HTTP. LangGraph nodes call tool servers via `httpx.AsyncClient`. Every tool call is capability-verified locally by the tool server using the capability JWT provided by orchestration — no per-call network round-trip to the identity service. Tool call logs (tool name, invocation time, duration, args hash, result preview, status) are generated exclusively by the orchestration backend. The LLM issues a tool call request; the backend records the actual invocation. Raw LLM reasoning is never written to audit records.

### 5.3 Agent Identity and Passport

**DID Registration**  
On first run, every agent receives a W3C Decentralized Identifier: `did:mergit:<uuid>`. The DID is stored in the identity service's PostgreSQL database alongside:
- Agent role(s)
- Capability set (which tools are allowed, which repos are accessible, trust level)
- SHA-256 hash of the capabilities JSON (stored on-chain in the NFT)

**AgentPassport NFT (ERC-721)**  
Simultaneously with DID registration, the identity service mints an ERC-721 soulbound token on the Monad blockchain. The token:
- Is non-transferable (soulbound — `_beforeTokenTransfer` reverts on transfer)
- Stores `capabilityHash`, `tasksCompleted`, `tasksAttempted`, `registeredAt`, `active`
- One NFT per agent address (enforced by `agentToTokenId` mapping)
- Updates `tasksCompleted`/`tasksAttempted` automatically as `ProofOfWork.sol` records proofs

### 5.4 Proof of Work

**Non-Blocking Proof Submission**
Proof submission does not block goal completion. On task completion, enqueue_proofs_node performs a non-blocking Redis XADD to stream:proof_queue and immediately proceeds to finalize. A dedicated proof-worker process in blockchain-indexer drains stream:proof_queue and submits proofs on-chain. This means goal.completed fires within <1s of all tasks finishing, regardless of Monad block confirmation time.

**Proof Bundle (Per-Goal Merkle Batching)**
The proof-worker collects all task proof entries for a goal, builds a Merkle tree (each leaf = SHA-256(task_id + result_json)), and calls ProofOfWork.sol::recordBundle(bundleId, goalId, merkleRoot, taskCount) once per goal. Individual task hashes are stored in PostgreSQL. Anyone can verify by recomputing the Merkle tree from the PostgreSQL records and comparing the root to the on-chain event.

**Proof Durability (Outbox Pattern)**
Before XADDing to stream:proof_queue, enqueue_proofs_node writes a proof_outbox record (task_id, goal_id, agent_address, result_hash, status=pending, attempts=0) to PostgreSQL. The proof-worker updates status to submitted on broadcast and confirmed on receipt. Records stuck in pending for >60s are retried with exponential backoff (1s, 2s, 4s, 8s…). After 10 failed attempts: status=dead_lettered + alert fired.

**Proof Idempotency**
recordBundle is idempotent on bundleId — recording the same bundle twice is a no-op. This prevents double-counting from retry logic, indexer restarts, or Monad reorgs.

### 5.5 Reputation System

**Composite Score Formula**  
```
score = (tasks_completed_rate  * 0.35)
      + (pr_merged_rate         * 0.25)
      + (human_approval_rate    * 0.15)
      + (quality_signal         * 0.15)
      + (badge_count_score      * 0.10)
```
Scaled to 0–10000 (e.g., 9850 = 98.50%).

**Score Components**  
| Component | Definition |
|-----------|-----------|
| `tasks_completed_rate` | `tasks_completed / tasks_attempted` (rolling 30-day window) |
| `pr_merged_rate` | Fraction of PRs created by the agent that were merged (not closed) |
| `human_approval_rate` | Average human approval rate on interrupt decisions (approvals / total interrupts) |
| `quality_signal` | 1 - (failed_rate * 0.4 + rollback_rate * 0.3 + security_issue_rate * 0.2 + revocation_rate * 0.1); ranges 0–1 |
| `badge_count_score` | Logarithmically scaled count of earned badges |

**On-Chain Score Update**  
The `reputation` service runs a batch job every 10 minutes. It calculates the new composite score, computes `componentHash = SHA-256(component_breakdown_json)`, and calls `blockchain-indexer` gRPC to submit `ReputationRegistry.sol::updateScore(agentAddress, newScore, componentHash)`.

The `componentHash` binds the on-chain integer to an off-chain JSON breakdown. Anyone can fetch the breakdown from the Mergit API, re-hash it, and verify it matches the on-chain hash.

**Verified Run Badge**  
After 5 consecutive successful task proofs (no failures, no replanning), the reputation service triggers badge issuance. Badges are metadata extensions on `AgentPassport.sol` (or a separate `Badges.sol` — TBD in Phase 3). A "Verified Run" badge is on-chain, visible in the frontend, and counted in the reputation score.

**Agent Discovery**
The leaderboard is a secondary discovery surface, accessible at /discover. The primary reputation UI is the per-agent passport card. The Redis sorted set leaderboard:global is updated in real time as scores change; the top-N view is served directly from the ZSET. The enriched/paginated view (joined with passport metadata) is cached with a 30s TTL and busted on each score write. Score history is stored in PostgreSQL (partitioned table, queryable by agent and date range).

### 5.6 Real-Time Event Streaming

Every state change in a goal or task is published to Redis Streams. The `api-gateway` consumes relevant channels and fans out to connected SSE clients. Events include:

- `task.started` — a task began executing
- `task.tool_call` — a tool was invoked (includes tool name, redacted args)
- `task.completed` — task finished with result preview
- `task.failed` — task failed with error
- `goal.completed` — all tasks done
- `proof.submitted` — on-chain proof tx hash available
- `proof.pending` — proofs enqueued to stream:proof_queue, chain submission in progress ("verifying on Monad…")
- `proof.confirmed` — transaction mined on Monad
- `reputation.updated` — agent score changed
- `badge.issued` — new badge minted on-chain
- `interrupt.pending` — agent paused, human approval required

### 5.7 Human-in-the-Loop

LangGraph's `interrupt()` primitive pauses graph execution and surfaces an approval request to the user. Four defined interrupt points:

| Trigger | Payload | User Action |
|---------|---------|------------|
| Before PR creation | PR title, diff summary, target branch | Approve / Revise (with feedback) / Reject |
| `credential_request` tool called | Credential type and description | Provide credential or Cancel |
| `replan_count > 2` | Current state, failure history | Provide guidance or Cancel goal |
| `code_exec` detects harmful pattern | Code snippet, detection reason | Allow / Block |

**Capability Gate for Supervised Tasks**
For tasks flagged human_supervised: true in the task spec, the interrupt prompt is shown regardless of whether the capability JWT permits the action. The human approval is treated as a runtime capability grant. This acts as a second capability layer for sensitive operations — JWT grants are necessary but not sufficient for supervised tasks.

The UI displays an approval modal when an `interrupt.pending` SSE event arrives. The user submits their decision via `POST /goals/{goal_id}/resume`. The gateway relays this to the orchestration service, which resumes the LangGraph thread via `Command(resume={decision})`.

### 5.8 Agent Economy

**Staking for Trust Tiering**
Before accepting high-risk tasks (github_pr writes, code_exec with external network access, credential_request), agents must post a minimum MON stake in StakeRegistry.sol. Stake amount is tiered by task risk level. Higher stake = access to higher-risk tool scopes.

**Slash Conditions**
Verified fraud (proof hash mismatch, fabricated tool results) triggers automated stake slashing via ChallengeManager.sol. Any observer can file a challenge by posting a counter-proof. If the challenge succeeds, the dishonest agent's stake is slashed and a portion rewarded to the challenger. Slash governance is reviewed by a Gnosis Safe 2-of-3 before execution.

**Economic Incentives Tied to Reputation**
Reputation score directly gates which tasks an agent can bid on. High-score agents access premium task queues. Low-score agents are restricted to supervised (human-approved) tasks until their score recovers.

**Contracts (Phase 2+)**
- StakeRegistry.sol — records agent stake, enforces minimums per risk tier
- ChallengeManager.sol — challenge filing, evidence submission, slashing execution
- TaskEscrow.sol (Phase 3+) — holds payer funds until proof verification; releases to agent on confirmed proof

---

## 6. System Architecture

### 6.1 High-Level Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              Client Layer                                 │
│              React TypeScript Frontend (Vite + Tailwind)                 │
│         REST · WebSocket · SSE · wagmi (wallet connect → Monad)          │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │ HTTPS / WSS
┌──────────────────────────────────▼───────────────────────────────────────┐
│                        api-gateway  (Rust / Axum 0.7)                    │
│         JWT Auth · Rate Limiting (Redis) · SSE Multiplex · Routing       │
│              HMAC-authed internal traffic; mTLS in Phase 5               │
└──────────┬──────────────────┬────────────────────────────────────────────┘
           │ REST/gRPC        │ HTTP REST
           ▼                  ▼
┌──────────────────┐  ┌─────────────────────────────┐
│  orchestration   │  │  accounts-service           │
│  Python/LangGraph│  │  Python / FastAPI           │
│  + tool-servers  │  │  identity module:           │
│                  │  │  DID · NFT · Caps           │
│                  │  │  reputation module:         │
│                  │  │  Scoring · Badges           │
└────────┬─────────┘  └─────────────┬───────────────┘
         │ Redis Stream             │ gRPC
         └──────────────────────────┘
                        │
        ┌───────────────▼────────────────┐
        │   blockchain-indexer  (Rust)   │
        │   alloy-rs · WS Event Sub      │
        │   Oracle Signing · TX Retry    │
        │   PostgreSQL sink · Redis cache│
        └───────────────┬────────────────┘
                        │ JSON-RPC / WSS
        ┌───────────────▼────────────────┐
        │        Monad Blockchain         │
        │  AgentPassport.sol              │
        │  ProofOfWork.sol                │
        │  ReputationRegistry.sol         │
        │  AuditTrail.sol                 │
        └─────────────────────────────────┘

                 Data Layer (shared, private network)
┌──────────────────────────────────────────────────────────────┐
│   PostgreSQL 16                  Redis 7 Cluster             │
│   + pgvector                     Streams · Hash · SortedSet  │
│   + pgBouncer (transaction mode) · pub/sub                   │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 Internal Communication Protocols

| Connection | Protocol | Justification |
|-----------|----------|--------------|
| Frontend → api-gateway | HTTPS REST + SSE + WSS | Standard web; SSE for streaming events |
| api-gateway → orchestration | HTTP REST | Goal CRUD and interrupt/resume |
| api-gateway → accounts-service | HTTP REST | Capability JWT local verification; only admin/registration calls go to accounts-service |
| orchestration → accounts-service | HTTP REST | Agent registration and JWT refresh only; capability checks are local JWT verification |
| orchestration → blockchain-indexer | Redis Stream (`stream:proof_queue`) | Non-blocking enqueue; proof-worker drains asynchronously |
| accounts-service → blockchain-indexer | gRPC | UpdateScore relay to oracle signing key (kept as gRPC: batch job, low-freq, Protobuf encoding justified) |
| tool-server → accounts-service | None (local JWT verify) | Capability JWT verified locally; no network call per tool invocation |
| blockchain-indexer → Monad | JSON-RPC (WS) | Event subscription + transaction submission |
| All services → PostgreSQL | `sqlx` async pool (TCP) | Via pgBouncer |
| All services → Redis | `redis` crate / `redis-py` (TCP) | Cache reads/writes, pub/sub |

**Internal Auth (Phases 1–4)**
All internal service-to-service calls include an `X-Internal-Auth: hex(hmac_sha256(service_secret, request_body))` header. The Docker private network prevents external spoofing; HMAC prevents in-network spoofing. mTLS replaces HMAC in Phase 5 when Kubernetes cert-manager is available.

---

## 7. Service Specifications

### 7.1 api-gateway (Rust / Axum 0.7)

**Purpose:** Single public ingress. All other services are on a private Docker network.

**Key Rust Crates**
```toml
axum = { version = "0.7", features = ["ws", "macros"] }
tokio = { version = "1", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace", "compression-gzip", "limit"] }
jsonwebtoken = "9"          # JWT HS256 (agents) and RS256 (users)
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"] }
moka = { version = "0.12", features = ["future"] }
metrics-exporter-prometheus = "0.14"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

**Responsibilities**
- JWT validation middleware on every route (excludes `/health`, `/auth/callback`)
- Sliding-window rate limiter: key = `{user_id}:{endpoint}`, stored in Redis, limit configurable per endpoint
- SSE endpoint: subscribes to Redis Streams for a given `goal_id` or `agent_id`, streams events to client; handles reconnection via `Last-Event-ID` header
- WebSocket upgrade for bidirectional agent event streams (Phase 4+)
- Request ID injection (`X-Request-ID` header, propagated to all downstream calls)
- OpenTelemetry trace context propagation (W3C `traceparent` header)
- X-Internal-Auth HMAC header on all outbound internal calls (Phases 1–4); mTLS in Phase 5
- In-process `moka` cache for user JWT public keys (5m TTL) and accounts-service signing key (5m TTL)

**Endpoints exposed**

| Method | Path | Proxies To |
|--------|------|-----------|
| POST | `/goals` | orchestration |
| GET | `/goals/{id}` | orchestration |
| GET | `/goals` | orchestration |
| DELETE | `/goals/{id}` | orchestration |
| POST | `/goals/{id}/resume` | orchestration (interrupt resume) |
| GET | `/stream/goals/{id}` | Redis Streams → SSE |
| GET | `/stream/agents/{id}` | Redis Streams → SSE |
| GET | `/agents/{id}` | accounts-service |
| POST | `/agents` | accounts-service (register) |
| GET | `/agents/{id}/passport` | accounts-service (NFT metadata) |
| GET | `/reputation/leaderboard` | accounts-service |
| GET | `/reputation/agents/{id}` | accounts-service |
| GET | `/proofs/{task_id}` | blockchain-indexer |
| GET | `/health` | local health check |
| POST | `/auth/login` | local OAuth flow |
| GET | `/auth/callback` | local OAuth callback |

---

### 7.2 orchestration (Python / LangGraph)

**Purpose:** Decomposes goals into task DAGs, manages agent execution, handles checkpointing, processes tool calls, and submits proofs.

**Key Python Packages**
```toml
langgraph = ">=0.2"
langchain-core = ">=0.3"
langgraph-checkpoint-postgres = ">=0.1"   # AsyncPostgresSaver
litellm = ">=1.40"
fastapi = ">=0.111"
uvicorn = { extras = ["standard"] }
httpx = ">=0.27"
grpcio = ">=1.64"
grpcio-tools = ">=1.64"
pydantic = ">=2.7"
redis = { extras = ["hiredis"] }
opentelemetry-sdk = ">=1.24"
```

**LangGraph StateGraph (see Section 11 for full detail)**

**Tool Servers (FastAPI sidecars)**  
Three independent FastAPI services, each in its own Docker container:

| Tool Server | Port | Tools |
|------------|------|-------|
| `tool-server-github` | 8001 | `github_ops`, `github_pr`, `wait_webhook` |
| `tool-server-code` | 8002 | `code_exec`, `file_ops` |
| `tool-server-search` | 8003 | `web_search`, `http_request` |

Each tool server:
1. Receives tool call from orchestration via `POST /execute` with `{tool, args, agent_id}`
2. Verifies the capability JWT from the request header: decodes HS256, checks `capabilities[]` claim includes `tool_name` and action scope, checks `exp`, checks `cap:revoked:{agent_id}` Redis key
3. Checks Redis tool output cache: key = `tool:{name}:{sha256(args)}` — returns cache hit if valid
4. Executes tool, writes result to Redis cache (tool-specific TTL), returns result
5. Appends action to Redis Stream `stream:audit_buffer` (non-blocking XADD): fields = `{agent_id, action_type, tool_name, args_hash, task_id, timestamp}`

**LLM Provider Strategy (LiteLLM)**  
Primary fallback chain: `groq/llama-4-maverick` → `groq/llama-3.3-70b` → `anthropic/claude-sonnet-4-6` → `openai/gpt-4o`  
Redis tracks provider health: consecutive error count, rate limit windows, last-success timestamp. LiteLLM checks Redis before routing.

**Horizontal Scaling**
Each tool server is deployed with `deploy.replicas: 3` in Docker Compose. DNS round-robin distributes requests across replicas, preventing single-replica saturation under LangGraph Send API parallel fan-out. Promoted to HPA-managed Kubernetes Deployment in Phase 5.

---

### 7.3 accounts-service (Python / FastAPI)

**Purpose:** Unified service handling agent identity (DID registration, NFT minting delegation, capability management) and reputation (score computation, badge engine, leaderboard). Two modules, one process, one Dockerfile.

**Key Python Packages**
```toml
fastapi = ">=0.111"
uvicorn = { extras = ["standard"] }
sqlalchemy = { extras = ["asyncio"] }
asyncpg = ">=0.29"
httpx = ">=0.27"
pydantic = ">=2.7"
python-jose = { extras = ["cryptography"] }  # JWT HS256
cryptography = ">=42"                         # Ed25519 keypair
redis = { extras = ["hiredis"] }
moka-py = ">=0.1"                             # in-process cache (or use functools.lru_cache + asyncio lock)
```

**Identity Module — Agent Registration Flow**
1. `POST /agents` received by api-gateway → forwarded to accounts-service
2. Generates UUIDv4, derives `did:mergit:{uuid}`
3. Generates Ed25519 keypair (stored encrypted in PostgreSQL via cryptography library; private key never leaves service)
4. Computes `capabilityHash = SHA-256(capabilities_json)`
5. Calls blockchain-indexer HTTP `POST /mint-passport {agentAddress, capabilityHash}` → gets `tokenId` (blockchain-indexer holds the alloy client; accounts-service has no chain client)
6. **Issues capability JWT** (HS256, TTL=5min, claims=`{agent_id, capabilities[], exp}`) signed with accounts-service signing key
7. Stores DID, keypair reference, capabilities, tokenId in PostgreSQL
8. Writes `agent_id → capabilities` to in-process cache (60s TTL) and Redis (5m TTL)
9. Returns `{agent_id, did, token_id, tx_hash, capability_jwt}`

Orchestration caches the `capability_jwt` and proactively refreshes it 30 seconds before expiry.

**Capability JWT Verification (Tool Servers)**
Tool servers verify the JWT locally:
1. Decode HS256 JWT using accounts-service public key (cached in moka, refreshed every 5m)
2. Check `capabilities[]` claim includes the requested `tool_name` and action scope
3. Check `exp` — reject if expired
4. Check `cap:revoked:{agent_id}` Redis key — if set, reject immediately (near-instant revocation within TTL)  
No network call to accounts-service per tool invocation.

**Reputation Module — Score Batch Job (every 10 minutes)**
```
1. SELECT agents with proofs since last_computed_at
2. For each agent:
   a. Fetch proof records from PostgreSQL (30-day window)
   b. Fetch PR merge data from PostgreSQL tool_results
   c. Compute 5 component scores (including quality_signal negative components)
   d. Compute composite = weighted sum * 10000
   e. Compute componentHash = SHA-256(component_json)
   f. Call blockchain-indexer gRPC UpdateScore(agent_address, composite, componentHash)
   g. Insert into reputation_snapshots (partitioned table)
   h. Update Redis sorted set ZADD leaderboard:global composite agent_id
3. Log batch stats (agents updated, avg score change, max delta)
```

**Badge Engine**
- "Verified Run" badge: triggered when `consecutive_successes >= 5`
- Badge issuance calls blockchain-indexer to update `AgentPassport.sol` badge metadata
- Stored in PostgreSQL `badges(agent_id, badge_type, issued_at, tx_hash)`

**Redis Read-Through Cache**
- `agent:status:{agent_id}` HASH with TTL=30s (miss path reads PostgreSQL)
- In-process cache (TTL=60s) sits in front of Redis for hottest lookups
- `cap:revoked:{agent_id}` pub/sub channel for near-instant JWT revocation

**Checkpointer Note**
AsyncPostgresSaver uses a dedicated PostgreSQL connection pool (separate from the general app pool) to prevent checkpoint burst writes from starving application queries.

---

### 7.4 blockchain-indexer (Rust / alloy)

**Purpose:** Bridges the off-chain system and Monad blockchain. Subscribes to contract events, maintains a PostgreSQL cache of on-chain state, and provides an oracle signing service for proof recording and reputation updates.

**Key Rust Crates**
```toml
alloy = { version = "1", features = [
  "providers", "signers", "sol-types", "contract",
  "rpc-types", "transports-ws", "transports-http",
  "network", "consensus",
] }
tonic = "0.12"           # gRPC server
sqlx = { version = "0.8", features = ["postgres", "uuid", "chrono"] }
redis = { version = "0.25", features = ["tokio-comp"] }
tokio = { version = "1", features = ["full"] }
```

**gRPC Service Interface**
```proto
service BlockchainService {
  rpc SubmitProof(ProofRequest) returns (ProofResponse);
  rpc LogAction(ActionRequest) returns (ActionResponse);
  rpc UpdateScore(ScoreRequest) returns (ScoreResponse);
  rpc StreamEvents(EventFilter) returns (stream BlockchainEvent);
}

message ProofRequest {
  string task_id = 1;    // idempotency key
  string goal_id = 2;
  string agent_address = 3;
  bytes  result_hash = 4;  // SHA-256 bytes
}

message ProofResponse {
  string tx_hash = 1;
  bool   was_duplicate = 2;
}
```

**Event Indexer Loop**
```
alloy WS Provider → subscribe to:
  - ProofOfWork.ProofRecorded events
  - ReputationRegistry.ScoreUpdated events
  - AgentPassport.PassportMinted events
  - AuditTrail.ActionLogged events

For each event:
  1. Decode via sol! generated bindings
  2. Check idempotency table (block_number + tx_hash)
  3. Upsert into PostgreSQL on_chain_state
  4. Publish to Redis Stream stream:onchain_events
  5. Mark idempotency record processed
```

**Oracle Transaction Management**
- Nonce management: `alloy::providers::fillers::NonceFiller` for automatic nonce tracking
- Gas estimation: `alloy::providers::fillers::GasFiller` with 20% multiplier for testnet variability
- Retry logic: exponential backoff (1s, 2s, 4s, 8s) on `REPLACEMENT_UNDERPRICED` and `NONCE_TOO_LOW`
- Receipt waiting: `provider.watch_pending_transaction(tx_hash).with_timeout(30s)`
- Idempotency: every `SubmitProof` gRPC call checks `proofs_submitted(task_id)` in PostgreSQL — returns existing tx_hash if already submitted

---

### 7.5 reputation (merged into accounts-service)
The reputation module is part of accounts-service (§7.3). This section is intentionally blank. The service will be promoted to a standalone Rust binary in Phase 5 if reputation compute starts contending with identity request handling.

---

## 8. Database Design

### 8.1 Engine Selection

**Two engines serve all persistence needs:**

| Engine | Role | Why |
|--------|------|-----|
| **PostgreSQL 16** | All durable, relational, and time-series data | ACID transactions, pgvector for embeddings, native partitioning for time-series, `sqlx` compile-time safety, single operational target |
| **Redis 7** | All volatile, ephemeral, and pub/sub data | Sub-millisecond TTL reads, Streams for event propagation, sorted sets for leaderboards, native expiry for cache entries |

**Why not additional engines:**  
- TimescaleDB: PostgreSQL native partitioning handles the reputation history time-series pattern without an additional extension
- Qdrant/Pinecone: `pgvector` co-locates embeddings with task records — enables JOIN-based semantic search; re-evaluate if p95 vector query exceeds 100ms at scale
- Cassandra/ScyllaDB: no write-heavy time-series pattern warrants distributed wide-column storage at current scale

**pgBouncer** runs in transaction-mode pooling in front of PostgreSQL. All `sqlx` pools are bounded to 10 connections per service. This prevents connection exhaustion under spike load.

### 8.2 PostgreSQL Schema

#### Goals Table
```sql
CREATE TABLE goals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES goals(id),
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
      -- 'pending' | 'planning' | 'executing' | 'awaiting_human' | 'done' | 'failed' | 'cancelled'
    agent_id        UUID REFERENCES agents(id),
    output          JSONB,
    error           TEXT,
    dag_json        JSONB,           -- serialized LangGraph task DAG
    replan_count    INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_goals_status       ON goals(status);
CREATE INDEX idx_goals_agent_id     ON goals(agent_id);
CREATE INDEX idx_goals_created_at   ON goals(created_at DESC);
```

#### Tasks Table
```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goal_id         UUID NOT NULL REFERENCES goals(id) ON DELETE CASCADE,
    agent_role      TEXT NOT NULL,   -- 'researcher' | 'writer' | 'coder' | 'integrator'
    tool_name       TEXT,
    args_json       JSONB,
    args_hash       TEXT,            -- SHA-256 of args_json (idempotency)
    status          TEXT NOT NULL DEFAULT 'pending',
      -- 'pending' | 'ready' | 'running' | 'done' | 'failed' | 'waiting_webhook' | 'waiting_credential'
    result_json     JSONB,
    result_hash     TEXT,            -- SHA-256 of result_json (stored on-chain)
    error           TEXT,
    attempt_count   INTEGER DEFAULT 0,
    max_attempts    INTEGER DEFAULT 3,
    worker_id       TEXT,
    lease_expires_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);

CREATE TABLE task_dependencies (
    task_id         UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    depends_on_id   UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    PRIMARY KEY (task_id, depends_on_id)
);

CREATE INDEX idx_tasks_goal_id     ON tasks(goal_id);
CREATE INDEX idx_tasks_status      ON tasks(status);
CREATE INDEX idx_tasks_lease       ON tasks(lease_expires_at) WHERE status = 'running';
```

#### Messages Table (LLM conversation history per task)
```sql
CREATE TABLE messages (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id     UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    role        TEXT NOT NULL,   -- 'system' | 'user' | 'assistant' | 'tool'
    content     TEXT NOT NULL,
    tool_name   TEXT,
    tool_args   JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_task_id ON messages(task_id);
```

#### Agents Table
```sql
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    did             TEXT UNIQUE NOT NULL,   -- 'did:mergit:{uuid}'
    address         TEXT UNIQUE NOT NULL,   -- Ethereum address (Monad)
    passport_token_id BIGINT,               -- ERC-721 tokenId
    capabilities    JSONB NOT NULL,
    capability_hash TEXT NOT NULL,          -- SHA-256 of capabilities JSON
    trust_level     INTEGER DEFAULT 1,      -- 1-5
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Proofs Table (on-chain proof records)
```sql
CREATE TABLE proofs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES tasks(id),
    goal_id         UUID NOT NULL REFERENCES goals(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    result_hash     TEXT NOT NULL,          -- matches on-chain ProofOfWork.resultHash
    tx_hash         TEXT,                   -- Monad transaction hash
    block_number    BIGINT,
    confirmed       BOOLEAN DEFAULT FALSE,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confirmed_at    TIMESTAMPTZ
);

CREATE UNIQUE INDEX idx_proofs_task_id ON proofs(task_id);  -- one proof per task
CREATE INDEX idx_proofs_agent_id       ON proofs(agent_id);
CREATE INDEX idx_proofs_confirmed      ON proofs(confirmed, confirmed_at);
```

#### On-Chain State Cache
```sql
CREATE TABLE on_chain_state (
    contract_address TEXT NOT NULL,
    key_name        TEXT NOT NULL,
    value_json      JSONB NOT NULL,
    block_number    BIGINT NOT NULL,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (contract_address, key_name)
);

CREATE TABLE on_chain_events_idempotency (
    tx_hash         TEXT NOT NULL,
    log_index       INTEGER NOT NULL,
    processed       BOOLEAN DEFAULT FALSE,
    processed_at    TIMESTAMPTZ,
    PRIMARY KEY (tx_hash, log_index)
);
```

#### Reputation Snapshots (time-series, partitioned by month)
```sql
CREATE TABLE reputation_snapshots (
    agent_id                UUID NOT NULL REFERENCES agents(id),
    composite_score         INTEGER NOT NULL,   -- 0-10000
    tasks_completed_rate    NUMERIC(5,4),
    pr_merged_rate          NUMERIC(5,4),
    response_time_score     NUMERIC(5,4),
    peer_review_score       NUMERIC(5,4),
    badge_count_score       NUMERIC(5,4),
    component_hash          TEXT NOT NULL,
    snapshot_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (agent_id, snapshot_at)
) PARTITION BY RANGE (snapshot_at);

-- Create monthly partitions (automated via cron job or migration)
CREATE TABLE reputation_snapshots_2026_06 
    PARTITION OF reputation_snapshots
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
```

#### Badges Table
```sql
CREATE TABLE badges (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_id    UUID NOT NULL REFERENCES agents(id),
    badge_type  TEXT NOT NULL,   -- 'verified_run' | 'streak_10' | etc.
    tx_hash     TEXT,
    issued_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_badges_agent_id ON badges(agent_id);
```

#### Vector Embeddings (pgvector)
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE goal_embeddings (
    goal_id     UUID UNIQUE NOT NULL REFERENCES goals(id) ON DELETE CASCADE,
    embedding   vector(1536) NOT NULL,    -- text-embedding-3-small
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_goal_embeddings_vector 
    ON goal_embeddings USING ivfflat (embedding vector_cosine_ops) 
    WITH (lists = 100);
```

### 8.3 Redis Key Schema

```
# Session management
session:{session_id}                    HASH  TTL=24h
  user_id, agent_id, permissions, created_at, expires_at

# Rate limiting
ratelimit:{user_id}:{endpoint}:{window} STRING (count)  TTL=window_size

# LLM response cache
llm:exact:{sha256(model+messages)}      HASH  TTL=24h
  response, model, token_count, created_at
llm:sem:{embedding_hash}                HASH  TTL=6h
  response, model, original_hash

# Tool output cache
tool:{tool_name}:{sha256(args)}         HASH  TTL=varies
  result, cached_at

# Agent plan cache
plan:{goal_embedding_hash}              HASH  TTL=12h
  dag_json, original_goal_id, created_at

# GitHub ETag cache
github:etag:{url_hash}                  HASH  TTL=5m
  etag, body_json

# On-chain read cache
onchain:{contract}:{method}:{args_hash} STRING (json)  TTL=12s

# Leaderboard
leaderboard:global                      ZSET
  score → agent_id

# Real-time event streams
stream:goal:{goal_id}                   STREAM
stream:agent:{agent_id}                 STREAM
stream:onchain_events                   STREAM  (consumer group: reputation, frontend-fanout)
stream:proof_queue                      STREAM  Consumer group: proof-worker. XADD fields: task_id, goal_id, agent_address, result_hash, bundle_id
stream:audit_buffer                     STREAM  Consumer group: audit-batcher. XADD fields: agent_id, action_type, tool_name, args_hash, task_id, timestamp

# LLM provider health
llm:provider:{provider_name}:errors     STRING (count)  TTL=60s
llm:provider:{provider_name}:ratelimit  STRING          TTL=varies

# Pub/sub channels
cap:revoked:{agent_id}                  PUBSUB  Published when an agent's capabilities are revoked. Tool servers subscribe to invalidate cached JWT trust within seconds.
```

---

## 9. Caching Strategy

### 9.1 Cache Hierarchy

Three layers, innermost wins:

| Layer | Technology | Scope | Eviction |
|-------|-----------|-------|---------|
| L1 | moka (in-process) | Per service replica | TTL + capacity |
| L2 | Redis 7 | Cross-replica shared | TTL + explicit invalidation |
| L3 | PostgreSQL (pgvector) | Semantic similarity | Never (query-time) |

### 9.2 LLM Response Cache

**Two-key strategy:**

| Cache Type | Key | TTL | When to Use |
|-----------|-----|-----|------------|
| Exact | llm:exact:v{asset_hash}:{sha256(model+messages)} | 24h | Deterministic prompts |
| Semantic | llm:semantic:{sha256(goal_embedding)} | 6h | Similar-but-not-identical goals |

**Versioned keys (critical):** The asset_hash prefix is derived at deploy time from sha256(system_prompts + tool_definitions)[:8]. This ensures a deploy that changes a prompt immediately invalidates all cached responses without waiting for TTL. Never use a manually bumped integer — those get forgotten.

**Semantic threshold:** cosine similarity > 0.97 required for a semantic hit. Roles that produce artifacts feeding real actions (Coder, Integrator) use **exact-match only, never semantic** — two goals 93% similar in embedding space can legitimately need different repos, files, or branches.

**Per-role policy (safety control, not just optimization):**
| Role | Cache Type | Max TTL | Rationale |
|------|-----------|---------|-----------|
| Researcher | exact + semantic | 24h | Idempotent prose, high reuse |
| Writer | exact + semantic | 24h | Idempotent prose |
| Coder | exact only | 12h | Artifacts feed code_exec/file_ops |
| Integrator | exact only | 1h | Artifacts feed github_pr writes |

### 9.3 Tool Output Cache

Key: `tool:{name}:{sha256(canonicalized_args)}`

**Canonicalization is required.** Serialize args with sorted keys and normalized whitespace before hashing — otherwise `{"a":1,"b":2}` and `{"b":2,"a":1}` are treated as different entries and miss each other.

**TTL policy:**
| Tool | TTL | Notes |
|------|-----|-------|
| web_search | 1h | Stale search results acceptable |
| github_ops (read) | 5m | Repo state changes frequently |
| code_exec | 12h | Only if sandbox is hermetic (no network, no clock) |
| http_request GET | 5m | Respects upstream Cache-Control/ETag |
| file_ops (read) | 2m | Workspace files change during execution |

**Never-cache set (side-effectful tools):** github_pr writes, file_ops writes, spawn_goal, wait_webhook, credential_request, http_request POST/PUT/PATCH/DELETE. These are excluded unconditionally — no TTL, no bypass.

### 9.4 Embedding Cache

Every goal is embedded for semantic LLM cache lookup. Cache the embedding itself to avoid a redundant API round-trip before the pgvector query.

Key: `emb:{model}:{dims}:{sha256(normalized_goal_text)}`  
TTL: Indefinite (embeddings are deterministic; a model swap changes the key prefix automatically)

Cost context: text-embedding-3-small costs $0.02/1M tokens — embedding cache is a latency win, not primarily a cost win.

### 9.5 Anthropic Prefix Caching

For calls routed to `anthropic/claude-sonnet-4-6`, set `cache_control: {type: ephemeral}` on the static system prompt prefix (tool definitions + role instructions).

**Requirements to make this work:**
- Token floor: Anthropic silently ignores the breakpoint if the prefix is < ~1024 tokens on Sonnet. Verify your tool definitions + role instructions clear this threshold.
- Breakpoint placement: The prefix must be byte-identical across calls. Order strictly: [static tool defs][static role instructions] | BREAKPOINT | [dynamic goal + retrieved memories]. If retrieve_memories injects into the system block, it must go after the breakpoint or it busts the cache on every call.
- Write cost is 1.25× input price; cache reads cost 0.1× input price. Net-positive only when reuse within the 5-minute provider TTL exceeds 1 call per breakpoint.
- Provider scope: Groq has no prefix cache; OpenAI auto-caches (≥1024 tokens). This optimization only fires on the Anthropic leg of the fallback chain.

**LiteLLM fallback and cache warmth:** Prefix caches are per-provider and non-portable. Every fallback to Anthropic pays the 1.25× write cost to warm a cold prefix. Implement sticky provider per task thread: Redis key `llm:thread:{task_id}:provider` (TTL = task TTL) to reduce provider flapping. Track `cache_read_input_tokens` / `cache_creation_input_tokens` in observability — these are real Anthropic API response fields that prove whether prefix caching is net-positive.

### 9.6 Stampede Protection

moka coalescing is per-process. With N service replicas, hot-key expiry (or a versioned-key deploy from §9.2) triggers up to N concurrent cache misses to the LLM simultaneously.

**Cross-instance single-flight:** On cache miss, set a Redis distributed lock:
```
SET lock:{cache_key} 1 NX PX 30000  (30s TTL for LLM calls)
```
First requester computes and writes; others retry the read after 100ms with exponential backoff.

**Stale-while-revalidate:** For LLM responses, serve the slightly-stale cached answer while one elected worker recomputes in the background. LLM answers tolerate mild staleness far better than latency spikes.

### 9.7 On-Chain Event Cache Correctness

Monad reorgs and indexer restarts re-deliver events. Two rules:

1. **Idempotency set:** Key all processed events on `(tx_hash, log_index)`. Never process the same event twice regardless of delivery count.
2. **Finality-keyed cache for write decisions:** For reads that drive a write-back (a reputation score you are about to submit on-chain), key the cache by `block_number` / finality tag, not wall-clock TTL. A 12s TTL can straddle a reorg. Wall-clock TTL is acceptable only for cosmetic reads (dashboard display, non-consequential queries).

### 9.8 Negative Caching

Cache failure responses with short TTLs to prevent retry storms:

| Failure Type | Cache TTL | Key Pattern |
|-------------|----------|------------|
| GitHub 404 | 5m | `tool:github_ops:404:{sha256(args)}` |
| LLM 429 (rate limit) | Retry-After value | `llm:ratelimit:{provider}` |
| Capability denied | 60s | `cap:denied:{agent_id}:{tool_name}` |
| Empty web_search result | 30s | `tool:web_search:empty:{sha256(args)}` |

Distinguish "empty result" (cacheable) from "error/timeout" (never cache — you would memoize a flaky upstream).

### 9.9 Cross-Cutting Landmines

**pgBouncer transaction mode:** pgBouncer in transaction mode recycles connections between transactions. This silently breaks:
- PostgreSQL advisory locks (released when connection is returned to pool before your intent completes) — use Redis SET NX locks instead
- LISTEN/NOTIFY (subscription is connection-bound, lost on pool recycle) — use Redis pub/sub instead
- asyncpg prepared statement cache — set `statement_cache_size=0` in the asyncpg connection string (or use pgbouncer-aware mode) for AsyncPostgresSaver

**LiteLLM fallback breaks prefix cache warmth:** See §9.5. Correlate provider-switch events with `cache_creation_input_tokens` spikes in observability to detect prefix cache thrashing.

**Deploy-time cold start:** Versioned key rollout (§9.2) invalidates all caches simultaneously. The first request to each prompt pattern post-deploy pays full LLM cost. Mitigate with a pre-warm step in the deploy pipeline (replay top-20 goal types against the new key prefix before routing live traffic). Tracked as a Phase 5 operational runbook item.

### 9.10 Caching Roadmap (Phase 5)

- LangGraph node memoization: short-circuit deterministic, side-effect-free nodes whose input-state slice is unchanged between DAG iterations. Memoize by `hash(node_id + input_slice)`. Only temp=0 nodes. Big win on `self_heal_node` which re-runs the whole DAG.
- Per-role TTL config table: move role→cache policy from code to a config file; enables tuning from usage data without a deploy.
- API gateway response cache: short-lived moka cache for idempotent GETs (`GET /agents/{id}/passport` 60s, `GET /reputation/leaderboard` 30s, `GET /proofs/{task_id}` 300s if confirmed). SSE streams are never cached.
- Cache observability: `llm_cache_hits_total{type}`, `llm_cache_misses_total{type}`, `llm_cache_tokens_saved_total`, `tool_cache_hits_total{tool_name}`. Target: >60% LLM cache hit rate. Alert on >80% Redis memory usage.
- LLM-as-judge semantic promotion: mine the 0.90–0.97 similarity band offline; gate promotion on observed repeat-frequency; store promotions as alias→canonical pointers (revocable). Scope to Researcher/Writer only.

---

## 10. Smart Contract Specifications

### 10.1 Contract Suite Overview

| Contract | Standard | Storage | Role |
|----------|---------|---------|------|
| `AgentPassport.sol` | ERC-721 | SSTORE | Soulbound identity NFT |
| `ProofOfWork.sol` | Custom | SSTORE | Idempotent proof ledger |
| `ReputationRegistry.sol` | Custom | SSTORE | On-chain score oracle |
| `AuditTrail.sol` | Custom | None (events only) | Immutable action log |

**Why separate contracts?**  
Monad's parallel EVM (MDBFT) executes transactions against non-overlapping storage slots concurrently. Separate contracts with distinct storage prevent read-write conflicts and maximize parallelism. A single combined contract would serialize all proof recordings, score updates, and audit logs.

### 10.2 AgentPassport.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";

contract AgentPassport is ERC721, AccessControl {
    bytes32 public constant MINTER_ROLE   = keccak256("MINTER_ROLE");
    bytes32 public constant UPDATER_ROLE  = keccak256("UPDATER_ROLE");  // ProofOfWork contract

    uint256 private _nextTokenId;

    struct PassportData {
        bytes32 capabilityHash;   // SHA-256 of capabilities JSON (packed)
        uint32  tasksCompleted;
        uint32  tasksAttempted;
        uint64  registeredAt;
        bool    active;
    }
    // Packed into 2 storage slots:
    //   slot 0: capabilityHash (32 bytes)
    //   slot 1: tasksCompleted (4) + tasksAttempted (4) + registeredAt (8) + active (1) = 17 bytes → packed

    mapping(uint256 => PassportData) public passports;
    mapping(address => uint256)      public agentToTokenId;  // enforces one-per-agent
    mapping(uint256 => string)       private _tokenURIs;     // IPFS metadata URI

    event PassportMinted(uint256 indexed tokenId, address indexed agent, bytes32 capabilityHash);
    event StatsUpdated(uint256 indexed tokenId, uint32 tasksCompleted, uint32 tasksAttempted);
    event Deactivated(uint256 indexed tokenId);

    constructor() ERC721("AgentPassport", "AGNT") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    // Soulbound: revert on transfer unless minting (from == address(0))
    function _beforeTokenTransfer(address from, address to, uint256, uint256)
        internal pure override {
        require(from == address(0), "AgentPassport: soulbound");
    }

    function mintPassport(address agent, bytes32 capabilityHash, string calldata uri)
        external onlyRole(MINTER_ROLE) returns (uint256) {
        require(agentToTokenId[agent] == 0, "AgentPassport: already registered");
        uint256 tokenId = ++_nextTokenId;
        _safeMint(agent, tokenId);
        passports[tokenId] = PassportData({
            capabilityHash: capabilityHash,
            tasksCompleted: 0,
            tasksAttempted: 0,
            registeredAt:   uint64(block.timestamp),
            active:         true
        });
        agentToTokenId[agent] = tokenId;
        _tokenURIs[tokenId] = uri;
        emit PassportMinted(tokenId, agent, capabilityHash);
        return tokenId;
    }

    function updateStats(uint256 tokenId, uint32 tasksCompleted, uint32 tasksAttempted)
        external onlyRole(UPDATER_ROLE) {
        PassportData storage p = passports[tokenId];
        p.tasksCompleted = tasksCompleted;
        p.tasksAttempted = tasksAttempted;
        emit StatsUpdated(tokenId, tasksCompleted, tasksAttempted);
    }
}
```

**Gas notes:**
- `PassportData` struct packed into 2 slots (saves 3 SSTORE on creation vs naive layout)
- Soulbound check is a `pure` revert — no storage read
- `mintPassport` costs ~80,000 gas; `updateStats` costs ~25,000 gas

### 10.3 ProofOfWork.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "./interfaces/IAgentPassport.sol";

contract ProofOfWork is AccessControl {
    bytes32 public constant RECORDER_ROLE = keccak256("RECORDER_ROLE");

    IAgentPassport public immutable agentPassport;

    struct Proof {
        bytes32 goalId;
        address agentAddress;
        bytes32 resultHash;
        uint64  timestamp;
        bool    verified;
    }
    // bytes32 taskId is the mapping key — not stored in struct (saves 1 slot)

    struct BundleRecord {
        bytes32 goalId;
        bytes32 merkleRoot;
        uint32  taskCount;
        uint64  timestamp;
        bool    recorded;
    }

    mapping(bytes32 => Proof)        public proofs;        // taskId → Proof
    mapping(bytes32 => bytes32[])    public goalProofs;    // goalId → taskIds[]
    mapping(bytes32 => BundleRecord) public bundles;       // bundleId → BundleRecord

    event ProofRecorded(
        bytes32 indexed taskId,
        bytes32 indexed goalId,
        address indexed agentAddress,
        bytes32 resultHash,
        uint64 timestamp
    );
    event ProofVerified(bytes32 indexed taskId, address verifier);

    // Bundle recording — one tx per goal (gas-efficient)
    event BundleRecorded(
        bytes32 indexed bundleId,
        bytes32 indexed goalId,
        bytes32          merkleRoot,
        uint32           taskCount,
        uint64           timestamp
    );

    constructor(address _agentPassport) {
        agentPassport = IAgentPassport(_agentPassport);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    // Idempotent: if taskId already recorded, returns without reverting
    function recordProof(
        bytes32 taskId,
        bytes32 goalId,
        address agentAddress,
        bytes32 resultHash
    ) external onlyRole(RECORDER_ROLE) {
        if (proofs[taskId].timestamp != 0) return;  // idempotency check

        proofs[taskId] = Proof({
            goalId:       goalId,
            agentAddress: agentAddress,
            resultHash:   resultHash,
            timestamp:    uint64(block.timestamp),
            verified:     false
        });
        goalProofs[goalId].push(taskId);

        // Update passport stats (via interface — stats pulled from off-chain, set atomically)
        // NOTE: passport.updateStats is called by the reputation batch job, not here,
        // to avoid per-proof passport SSTORE (expensive). This event is the source of truth.
        emit ProofRecorded(taskId, goalId, agentAddress, resultHash, uint64(block.timestamp));
    }

    function recordBundle(
        bytes32 bundleId,
        bytes32 goalId,
        bytes32 merkleRoot,
        uint32  taskCount
    ) external onlyRole(RECORDER_ROLE) {
        require(!bundles[bundleId].recorded, "Bundle already recorded");
        bundles[bundleId] = BundleRecord({
            goalId:     goalId,
            merkleRoot: merkleRoot,
            taskCount:  taskCount,
            timestamp:  uint64(block.timestamp),
            recorded:   true
        });
        emit BundleRecorded(bundleId, goalId, merkleRoot, taskCount, uint64(block.timestamp));
    }

    function verifyProof(bytes32 taskId) external onlyRole(RECORDER_ROLE) {
        require(proofs[taskId].timestamp != 0, "ProofOfWork: proof not found");
        proofs[taskId].verified = true;
        emit ProofVerified(taskId, msg.sender);
    }

    function getGoalProofCount(bytes32 goalId) external view returns (uint256) {
        return goalProofs[goalId].length;
    }
}
```

Individual task hashes stay in PostgreSQL. Anyone can verify by recomputing the Merkle tree from off-chain records and comparing the root to the on-chain `BundleRecorded` event. `recordProof` is retained as the single-task fallback for debugging and per-task idempotency guarantees.

### 10.4 ReputationRegistry.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract ReputationRegistry is AccessControl {
    bytes32 public constant ORACLE_ROLE = keccak256("ORACLE_ROLE");

    uint256 public constant MAX_DELTA_BPS = 2000;      // 20% max change per update (basis points)
    uint256 public constant HISTORY_CAP   = 1000;      // ring-buffer cap

    struct Score {
        uint64  composite;        // 0–10000 (scaled by 1e4)
        uint32  snapshotBlock;
        bytes32 componentHash;    // SHA-256 of off-chain component JSON
    }

    mapping(address => Score)    public currentScores;
    mapping(address => Score[])  public scoreHistory;
    mapping(address => uint256)  private _historyHead;  // ring-buffer head

    event ScoreUpdated(
        address indexed agent,
        uint64  oldScore,
        uint64  newScore,
        uint32  blockNum,
        bytes32 componentHash
    );

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    function updateScore(
        address agent,
        uint64  newScore,
        bytes32 componentHash
    ) external onlyRole(ORACLE_ROLE) {
        require(newScore <= 10000, "ReputationRegistry: score out of range");

        Score storage current = currentScores[agent];
        uint64 oldScore = current.composite;

        // Anti-manipulation: reject updates with delta > 20%
        if (oldScore > 0) {
            uint256 delta = oldScore > newScore
                ? (uint256(oldScore - newScore) * 10000) / oldScore
                : (uint256(newScore - oldScore) * 10000) / oldScore;
            require(delta <= MAX_DELTA_BPS, "ReputationRegistry: delta too large");
        }

        // Update current score
        current.composite      = newScore;
        current.snapshotBlock  = uint32(block.number);
        current.componentHash  = componentHash;

        // Append to history (ring-buffer)
        Score[] storage history = scoreHistory[agent];
        if (history.length < HISTORY_CAP) {
            history.push(current);
        } else {
            uint256 head = _historyHead[agent];
            history[head] = current;
            _historyHead[agent] = (head + 1) % HISTORY_CAP;
        }

        emit ScoreUpdated(agent, oldScore, newScore, uint32(block.number), componentHash);
    }

    function getScore(address agent) external view returns (uint64 score, bytes32 componentHash) {
        Score storage s = currentScores[agent];
        return (s.composite, s.componentHash);
    }
}
```

### 10.5 AuditTrail.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract AuditTrail is AccessControl {
    bytes32 public constant WRITER_ROLE = keccak256("WRITER_ROLE");

    // Action type constants (keccak256 of human-readable strings)
    bytes32 public constant ACTION_TOOL_CALL    = keccak256("TOOL_CALL");
    bytes32 public constant ACTION_PR_CREATED   = keccak256("PR_CREATED");
    bytes32 public constant ACTION_PR_MERGED    = keccak256("PR_MERGED");
    bytes32 public constant ACTION_GOAL_START   = keccak256("GOAL_START");
    bytes32 public constant ACTION_GOAL_DONE    = keccak256("GOAL_DONE");
    bytes32 public constant ACTION_INTERRUPT    = keccak256("INTERRUPT");

    // No storage — all information is in events (immutable in log trie)
    event ActionLogged(
        bytes32 indexed agentId,
        bytes32 indexed actionType,
        bytes32          payloadHash,   // SHA-256 of full payload JSON (stored off-chain)
        uint64           timestamp
    );

    // Batched write — indexer drains stream:audit_buffer every 10s
    event ActionBatchLogged(
        bytes32 indexed merkleRoot,
        uint32          count,
        uint64          timestamp
    );

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }

    // Low gas: single event emit, zero SSTORE
    function log(
        bytes32 agentId,
        bytes32 actionType,
        bytes32 payloadHash
    ) external onlyRole(WRITER_ROLE) {
        emit ActionLogged(agentId, actionType, payloadHash, uint64(block.timestamp));
    }

    function logBatch(
        bytes32 merkleRoot,
        uint32  count
    ) external onlyRole(WRITER_ROLE) {
        emit ActionBatchLogged(merkleRoot, count, uint64(block.timestamp));
    }
}
```

**Gas cost:** ~21,000 base + ~750 per indexed topic + ~8 per byte of non-indexed data ≈ ~24,000 gas per audit log entry. On Monad testnet with 10,000+ TPS, this is negligible. `logBatch` amortizes this cost across hundreds of actions per tx — the indexer drains `stream:audit_buffer` every 10 seconds, builds the Merkle tree off-chain, and emits a single `ActionBatchLogged` event. Off-chain reconstruction: client pulls the action JSON list for a batch from `/audit/batches/{merkleRoot}`, rebuilds the Merkle tree, and compares the root to the on-chain event. Tamper-evidence is preserved. The single-action `log()` / `ActionLogged` path is retained as the fallback for debugging.

### 10.6 Deployment Script

```solidity
// contracts/script/Deploy.s.sol
pragma solidity ^0.8.24;
import "forge-std/Script.sol";
import "../src/AgentPassport.sol";
import "../src/ProofOfWork.sol";
import "../src/ReputationRegistry.sol";
import "../src/AuditTrail.sol";

contract Deploy is Script {
    function run() external {
        vm.startBroadcast();

        AgentPassport passport  = new AgentPassport();
        ProofOfWork   pow       = new ProofOfWork(address(passport));
        ReputationRegistry rep  = new ReputationRegistry();
        AuditTrail    audit     = new AuditTrail();

        // Grant roles (addresses from env)
        address identityService   = vm.envAddress("IDENTITY_SERVICE_ADDRESS");
        address indexerOracle     = vm.envAddress("INDEXER_ORACLE_ADDRESS");

        passport.grantRole(passport.MINTER_ROLE(),    identityService);
        passport.grantRole(passport.UPDATER_ROLE(),   address(pow));
        pow.grantRole(pow.RECORDER_ROLE(),            indexerOracle);
        rep.grantRole(rep.ORACLE_ROLE(),              indexerOracle);
        audit.grantRole(audit.WRITER_ROLE(),          indexerOracle);

        // Renounce deployer admin on audit + pow (immutability guarantee)
        // Keep admin on passport + rep for capability updates

        vm.stopBroadcast();

        // Log addresses for deployments/10143.json
        console.log("AgentPassport:       ", address(passport));
        console.log("ProofOfWork:         ", address(pow));
        console.log("ReputationRegistry:  ", address(rep));
        console.log("AuditTrail:          ", address(audit));
    }
}
```

---

## 11. LangGraph Orchestration Design

### 11.1 Full AgentState Schema

```python
from typing import TypedDict, Annotated, Literal
from langgraph.graph.message import add_messages

class TaskRecord(TypedDict):
    id:           str
    agent_role:   str   # "researcher" | "writer" | "coder" | "integrator"
    tool_name:    str
    args:         dict
    depends_on:   list[str]   # list of task IDs
    status:       str         # "pending" | "ready" | "running" | "done" | "failed"
    result:       dict | None
    result_hash:  str | None  # SHA-256 of result JSON

class InterruptPayload(TypedDict):
    type:          str   # "pr_approval" | "credential_request" | "escalation" | "security_gate"
    description:   str
    data:          dict  # type-specific payload
    goal_id:       str

class AgentState(TypedDict):
    # Goal context
    goal_id:           str
    goal_description:  str
    goal_status:       Literal["planning", "executing", "awaiting_human", "done", "failed", "cancelled"]
    
    # Task DAG
    tasks:             list[TaskRecord]
    current_task_id:   str | None
    
    # Agent identity
    agent_id:          str
    agent_role:        str
    
    # Execution tracking
    tool_results:      dict           # task_id → result dict
    error_count:       int
    replan_count:      int
    consecutive_successes: int         # for badge tracking
    
    # Human-in-the-loop
    pending_interrupt: InterruptPayload | None
    
    # Blockchain
    proofs_submitted:  list[str]       # tx hashes
    
    # Memory and context
    relevant_memories: list[str]       # pgvector retrieval before planning
    
    # Conversation (for LLM context)
    messages: Annotated[list, add_messages]
    
    # Metadata
    created_at:        str
    updated_at:        str
    parent_goal_id:    str | None
    thread_id:         str             # "{goal_id}:{attempt_number}"
```

### 11.2 Graph Definition

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

def build_graph(checkpointer: AsyncPostgresSaver) -> CompiledGraph:
    graph = StateGraph(AgentState)

    # Add all nodes
    graph.add_node("retrieve_memories",   retrieve_memories_node)
    graph.add_node("plan",                plan_node)
    graph.add_node("dag_router",          dag_router_node)
    graph.add_node("researcher",          researcher_node)
    graph.add_node("writer",              writer_node)
    graph.add_node("coder",               coder_node)
    graph.add_node("integrator",          integrator_node)
    graph.add_node("replan",              replan_node)
    graph.add_node("self_heal",           self_heal_node)
    graph.add_node("enqueue_proofs",      enqueue_proofs_node)
    graph.add_node("finalize",            finalize_node)

    # Entry point
    graph.set_entry_point("retrieve_memories")

    # Edges
    graph.add_edge("retrieve_memories",  "plan")
    graph.add_edge("plan",               "dag_router")

    # dag_router uses Send API for parallel execution
    graph.add_conditional_edges(
        "dag_router",
        route_tasks,           # returns list of Send() objects or "enqueue_proofs"
        {"enqueue_proofs": "enqueue_proofs", "__dynamic__": None}
    )

    # All agent nodes → dag_router (check for next ready tasks)
    for role in ["researcher", "writer", "coder", "integrator"]:
        graph.add_conditional_edges(
            role,
            check_after_task,  # → "dag_router" | "replan" | "self_heal"
            {"dag_router": "dag_router", "replan": "replan", "self_heal": "self_heal"}
        )

    graph.add_edge("replan",         "dag_router")
    graph.add_edge("self_heal",      "finalize")
    graph.add_edge("enqueue_proofs", "finalize")
    graph.add_edge("finalize",       END)

    return graph.compile(checkpointer=checkpointer)
```

### 11.3 Key Node Implementations

**plan_node** — decomposes goal into task DAG
```python
async def plan_node(state: AgentState) -> dict:
    # Build prompt with memories injected
    planning_prompt = build_planning_prompt(
        goal=state["goal_description"],
        memories=state["relevant_memories"]
    )

    # Check plan cache first (semantic similarity via pgvector)
    cached_plan = await check_plan_cache(state["goal_description"])
    if cached_plan:
        return {"tasks": interpolate_plan(cached_plan, state), "goal_status": "executing"}

    # LLM decomposition
    response = await litellm_call(planning_prompt, model="groq/llama-4-maverick")
    tasks = parse_task_dag(response)

    # Cache plan for future similar goals
    await store_plan_cache(state["goal_description"], tasks)

    return {"tasks": tasks, "goal_status": "executing"}
```

**dag_router_node** — identifies ready tasks and fans out
```python
from langgraph.types import Send

def route_tasks(state: AgentState):
    # Check if all tasks done → proceed to proof enqueueing
    if all(t["status"] in ("done", "failed") for t in state["tasks"]):
        return "enqueue_proofs"

    # Find tasks with all dependencies completed
    completed_ids = {t["id"] for t in state["tasks"] if t["status"] == "done"}
    ready = [
        t for t in state["tasks"]
        if t["status"] == "pending"
        and all(dep in completed_ids for dep in t["depends_on"])
    ]

    if not ready:
        # No ready tasks but not all done → waiting (shouldn't happen in correct DAG)
        return "enqueue_proofs"

    # Fan out in parallel via Send API
    return [Send(t["agent_role"], {**state, "current_task_id": t["id"]}) for t in ready]
```

**integrator_node** — with interrupt for PR approval
```python
from langgraph.types import interrupt, Command

async def integrator_node(state: AgentState) -> dict:
    task = get_current_task(state)

    if task["tool_name"] == "github_pr":
        # Execute PR creation draft (no actual merge yet)
        pr_draft = await call_tool_server("github", "github_pr", {
            **task["args"], "draft": True
        }, state["agent_id"])

        # Interrupt for human approval
        approval = interrupt({
            "type": "pr_approval",
            "pr_url": pr_draft["url"],
            "diff_summary": pr_draft["diff_summary"],
            "goal_id": state["goal_id"],
            "description": f"Agent wants to create a PR: {pr_draft['title']}"
        })

        if approval["decision"] == "approve":
            result = await call_tool_server("github", "github_pr_publish", {
                "pr_id": pr_draft["id"]
            }, state["agent_id"])
        elif approval["decision"] == "revise":
            # Re-enter with feedback
            return {
                "goal_description": f"{state['goal_description']}\n\nFeedback: {approval['feedback']}",
                "replan_count": state["replan_count"] + 1,
                "tasks": reset_pending_tasks(state["tasks"])
            }
        else:  # reject
            return {"goal_status": "cancelled"}
    else:
        result = await call_tool_server("github", task["tool_name"], task["args"], state["agent_id"])

    result_hash = sha256_json(result)
    return update_task_result(state, task["id"], result, result_hash)
```

**enqueue_proofs_node** — non-blocking proof submission via outbox + Redis Stream
```python
async def enqueue_proofs_node(state: GraphState) -> GraphState:
    """Non-blocking: XADD to stream:proof_queue, do not wait for chain confirmation."""
    for task in state["completed_tasks"]:
        result_hash = hashlib.sha256(
            json.dumps(task["result"], sort_keys=True).encode()
        ).hexdigest()
        # Write outbox record first (durability)
        await db.execute(
            """INSERT INTO proof_outbox (task_id, goal_id, agent_address, result_hash, status)
               VALUES ($1, $2, $3, $4, 'pending')
               ON CONFLICT (task_id) DO NOTHING""",
            task["id"], state["goal_id"], task["agent_address"], result_hash
        )
        # Non-blocking enqueue
        await redis.xadd("stream:proof_queue", {
            "task_id":       task["id"],
            "goal_id":       state["goal_id"],
            "agent_address": task["agent_address"],
            "result_hash":   result_hash,
            "bundle_id":     state["goal_id"],  # one bundle per goal
        })
    # Publish proof.pending SSE event
    await redis.xadd(f"stream:goal:{state['goal_id']}", {
        "event": "proof.pending",
        "goal_id": state["goal_id"],
        "task_count": len(state["completed_tasks"]),
    })
    return state
```

The `blockchain-indexer` consumes `stream:proof_queue` independently, batches task hashes per `bundle_id` (goal), computes the Merkle root, and calls `ProofOfWork.recordBundle` — one transaction per goal. The graph node returns immediately without blocking on chain confirmation, keeping goal latency decoupled from Monad block time.

### 11.4 Checkpoint Strategy

```python
import psycopg
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async def create_checkpointer() -> AsyncPostgresSaver:
    conn = await psycopg.AsyncConnection.connect(
        os.environ["POSTGRES_URL"],
        autocommit=True
    )
    checkpointer = AsyncPostgresSaver(conn)
    await checkpointer.setup()  # creates checkpoint tables if not exists
    return checkpointer

# Thread ID scheme: goal_id:attempt_number
# This allows retrying a failed goal while preserving prior attempt history
def make_thread_id(goal_id: str, attempt: int = 0) -> str:
    return f"{goal_id}:{attempt}"

# Invoking the graph
async def run_goal(goal: GoalRequest, checkpointer: AsyncPostgresSaver):
    graph = build_graph(checkpointer)
    config = {"configurable": {"thread_id": make_thread_id(goal.id)}}
    initial_state = build_initial_state(goal)
    
    async for event in graph.astream(initial_state, config=config):
        # Publish event to Redis Stream for SSE fan-out
        await publish_event(goal.id, event)
```

---

## 12. API Design

### 12.1 Goals API

```
POST   /goals                       Submit a new goal
GET    /goals                       List goals (paginated, filterable by status/agent_id)
GET    /goals/{goal_id}             Get goal detail with full task list and proof links
DELETE /goals/{goal_id}             Cancel a running goal
POST   /goals/{goal_id}/resume      Resume from interrupt (body: {decision, ...})
GET    /goals/{goal_id}/audit       Download full audit report (proofs + task outputs)
```

### 12.2 Agents API

```
POST   /agents                      Register a new agent (returns DID + passport tx_hash)
GET    /agents/{agent_id}           Get agent info (DID, capabilities, stats)
GET    /agents/{agent_id}/passport  Get passport NFT metadata + on-chain stats
PATCH  /agents/{agent_id}/capabilities  Update capability set (identity service)
```

### 12.3 Proofs API

```
GET    /proofs/{task_id}            Get proof record (result_hash, tx_hash, block explorer link)
POST   /proofs/{task_id}/verify     Trigger verification (re-hashes result, checks chain)
GET    /goals/{goal_id}/proofs      List all proofs for a goal
```

### 12.4 Reputation API

```
GET    /reputation/leaderboard      Top-N agents (query params: limit, offset)
GET    /reputation/agents/{agent_id}         Current score + component breakdown
GET    /reputation/agents/{agent_id}/history Score history (query params: from, to, resolution)
GET    /reputation/agents/{agent_id}/badges  List earned badges
```

### 12.5 Streaming API

```
GET    /stream/goals/{goal_id}      SSE stream for goal events
GET    /stream/agents/{agent_id}    SSE stream for agent events
GET    /stream/proofs               SSE stream for all new on-chain proofs (global feed)
```

### 12.6 Auth API

```
GET    /auth/login?provider={github|google}  Initiate OAuth flow
GET    /auth/callback                        OAuth callback handler
POST   /auth/logout                          Invalidate session
GET    /auth/me                              Current user info
```

### 12.7 Standard Response Envelope

```json
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-06-02T10:00:00Z"
  },
  "error": null
}
```

Error response:
```json
{
  "data": null,
  "meta": { "request_id": "uuid", "timestamp": "..." },
  "error": {
    "code": "GOAL_NOT_FOUND",
    "message": "Goal with ID abc123 does not exist",
    "details": {}
  }
}
```

---

## 13. Security and Trust Model

### 13.1 Oracle Trust Model

The oracle is the entity that writes data on-chain (proof recording, score updates, audit events). A single compromised oracle key could fabricate proofs, inflate scores, and corrupt the audit log. Mergit defends against this at four independent layers:

**Layer 1 — Role Separation**  
Three distinct wallet addresses with three distinct roles:
- `MINTER_ROLE` → identity service (mints passport NFTs)
- `RECORDER_ROLE` + `ORACLE_ROLE` + `WRITER_ROLE` → blockchain-indexer oracle address

Compromising the identity service key allows minting fake passports but cannot modify scores or records.

**Layer 2 — On-Chain Validation Guards**  
- `ProofOfWork.recordProof`: idempotent on `taskId` — recording the same task twice is a no-op. A compromised oracle cannot inflate task counts.
- `ReputationRegistry.updateScore`: rejects updates where the score delta exceeds 20% of the current score. A compromised oracle cannot set scores to 0 or 10000 in a single transaction.

**Layer 3 — Off-Chain Verifiability**  
Every on-chain value is bound to an off-chain receipt:
- `ProofOfWork.resultHash` ↔ task output JSON in PostgreSQL (SHA-256 verifiable)
- `ReputationRegistry.componentHash` ↔ score component breakdown JSON in PostgreSQL (SHA-256 verifiable)

Any observer can download the off-chain data, re-compute the hash, and verify it matches the on-chain value. This detects silent oracle tampering without requiring on-chain storage of full data.

**Layer 4 — Multi-Signature Key Management**  
The oracle signing key used for routine proof recording is a single key held by the blockchain-indexer service. However, the `DEFAULT_ADMIN_ROLE` on all contracts (which can grant/revoke roles) is held in a **Gnosis Safe 2-of-3 multi-sig**:
- Key 1: blockchain-indexer service (automated operations)
- Key 2: human operator A
- Key 3: human operator B

Key rotation, role grants, or emergency pausing require 2-of-3 signatures. This prevents a single private key compromise from permanently corrupting the contracts.

### 13.2 API Security

- All API endpoints require valid JWT except `/health` and `/auth/*`
- JWT HS256 for agent tokens (short-lived, 1h), RS256 for user tokens (24h)
- Rate limiting per `{user_id}:{endpoint}` via Redis sliding window
- Input validation via Pydantic (Python) and serde/validator (Rust) at every service boundary
- No user-supplied data is passed to shell commands (code execution runs in isolated sandbox container with no network and no file system mount)
- **Internal Auth (Phases 1–4):** All internal service-to-service calls include `X-Internal-Auth: hex(hmac_sha256(service_secret, request_body))`. The Docker private network prevents external access; HMAC prevents in-network spoofing. Secrets rotated at each deploy via environment injection.
- **Internal Auth (Phase 5+):** mTLS via cert-manager in Kubernetes. Mechanical swap — the HMAC header is removed and replaced with mutual certificate verification. No application logic changes required.

### 13.3 Capability-Based Access Control

Tool servers verify capabilities locally using the capability JWT issued by accounts-service at agent registration:
```
1. Extract Bearer JWT from X-Capability-JWT request header
2. Verify HS256 signature using accounts-service public key (cached in moka, TTL 5m)
3. Check capabilities[] claim includes the requested tool_name and action scope
4. Check exp — reject if expired (force re-fetch from accounts-service)
5. Check cap:revoked:{agent_id} Redis key — if set, reject with 403 CAPABILITY_REVOKED
No network hop to accounts-service on the hot path. Revocation takes effect within seconds via Redis pub/sub.
```

Capabilities are set at agent registration and can be updated via authenticated `PATCH /agents/{id}/capabilities`. Changes to capabilities update the `capabilityHash` in PostgreSQL but do **not** automatically update the on-chain hash (the blockchain records the capability hash at registration time — this is intentional for auditability of what the agent was authorized to do at any point in time).

---

## 14. Infrastructure and Deployment

### 14.1 Local Development (Docker Compose)

```yaml
# infra/docker-compose.yml (abridged)
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: mergit
      POSTGRES_USER: mergit
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  pgbouncer:
    image: bitnami/pgbouncer:1.22
    environment:
      PGBOUNCER_DATABASE: mergit
      PGBOUNCER_POOL_MODE: transaction
      PGBOUNCER_MAX_CLIENT_CONN: 200
    depends_on: [postgres]

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  api-gateway:
    build: ./services/api-gateway
    ports: ["8000:8000"]
    environment:
      POSTGRES_URL: postgres://mergit:${POSTGRES_PASSWORD}@pgbouncer:5432/mergit
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
    depends_on: [pgbouncer, redis]

  orchestration:
    build: ./services/orchestration
    environment:
      POSTGRES_URL: postgres://...
      REDIS_URL: redis://...
      IDENTITY_GRPC: identity:50051
      BLOCKCHAIN_GRPC: blockchain-indexer:50052
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
    depends_on: [pgbouncer, redis, identity, blockchain-indexer]

  tool-server-github:
    build:
      context: ./services/orchestration
      dockerfile: Dockerfile.tool-server
      args: TOOL_SERVER: github
    environment:
      GITHUB_TOKEN: ${GITHUB_TOKEN}
      REDIS_URL: redis://...
      IDENTITY_GRPC: identity:50051

  tool-server-code:
    build: { context: ..., dockerfile: Dockerfile.tool-server, args: { TOOL_SERVER: code } }

  tool-server-search:
    build: { context: ..., dockerfile: Dockerfile.tool-server, args: { TOOL_SERVER: search } }
    environment:
      BRAVE_API_KEY: ${BRAVE_API_KEY}

  identity:
    build: ./services/identity
    environment:
      POSTGRES_URL: ...
      MONAD_RPC_HTTP: ${MONAD_RPC_HTTP}
      AGENT_PASSPORT_ADDRESS: ${AGENT_PASSPORT_ADDRESS}

  blockchain-indexer:
    build: ./services/blockchain-indexer
    environment:
      POSTGRES_URL: ...
      REDIS_URL: ...
      MONAD_RPC_WSS: ${MONAD_RPC_WSS}
      ORACLE_KEYSTORE_PATH: /run/secrets/oracle-keystore
    secrets: [oracle-keystore]

  reputation:
    build: ./services/reputation
    environment:
      POSTGRES_URL: ...
      REDIS_URL: ...
      BLOCKCHAIN_GRPC: blockchain-indexer:50052

  frontend:
    build: ./frontend
    ports: ["3000:3000"]

  jaeger:
    image: jaegertracing/all-in-one:1.57
    ports: ["16686:16686", "4317:4317"]   # UI + OTLP gRPC

  prometheus:
    image: prom/prometheus:v2.51
    volumes: ["./infra/prometheus.yml:/etc/prometheus/prometheus.yml"]
    ports: ["9090:9090"]

  grafana:
    image: grafana/grafana:10.4
    ports: ["3001:3000"]
    volumes: ["grafana_data:/var/lib/grafana"]
```

### 14.2 CI/CD Pipelines

**ci-rust.yml** (triggers on PR and push to `main`)
```yaml
- name: Format check   → cargo fmt --check
- name: Lint           → cargo clippy -- -D warnings
- name: Test           → cargo test --workspace --locked
- name: Security audit → cargo audit
- name: Docker build   → docker build services/api-gateway, identity, blockchain-indexer, reputation
```

**ci-python.yml**
```yaml
- name: Lint   → ruff check . && ruff format --check .
- name: Types  → mypy services/orchestration/src
- name: Test   → pytest services/orchestration/tests --cov --cov-fail-under=80
```

**ci-contracts.yml**
```yaml
- name: Format  → forge fmt --check
- name: Build   → forge build
- name: Test    → forge test -vvv --gas-report
- name: Analyze → slither contracts/src --json slither-report.json
- name: Upload  → artifact upload slither-report.json
```

**cd-staging.yml** (on merge to `main`)
```yaml
- Build + push Docker images to ghcr.io/mergit-io
- SSH to staging server → docker compose pull && docker compose up -d
- Wait for health checks → curl /health all services
- Run smoke tests → scripts/smoke-test.sh
```

**cd-production.yml** (on tag push `v*.*.*`, requires manual approval)
```yaml
- If contracts/ changed: forge script Deploy.s.sol --broadcast --verify
- Update deployments/{chain_id}.json with new addresses
- Rolling deploy of services with health-check gating
```

### 14.3 Phase 5 Kubernetes (Planned)

Each service gets a `Deployment` with:
- `HorizontalPodAutoscaler` (CPU 70% threshold, min 2 / max 10 replicas)
- `PodDisruptionBudget` (minAvailable: 1 during rolling updates)
- `ResourceQuota` (CPU/memory limits)

**Note on orchestration scaling:** LangGraph with `AsyncPostgresSaver` supports horizontal scaling — multiple orchestration pods can safely share the PostgreSQL checkpoint store. The only contention is on PostgreSQL, which pgBouncer handles.

**Note on blockchain-indexer:** Only one active indexer instance should process events (to avoid duplicate TX submissions). Use leader election via a Redis lock (`SET indexer:leader {instance_id} NX EX 30` + heartbeat) — the non-leader runs in standby mode.

---

## 15. Repository and Contribution Strategy

### 15.1 Branch Strategy

```
main          Protected. Always deployable. Requires 2 approvals + CI green.
              Linear history only (squash merge for features, rebase for hotfixes).

develop       Integration branch. Features merge here first via PR.
              Auto-deploys to staging on merge.

feat/<scope>/<short-desc>   Feature branches off develop
  e.g. feat/identity/agent-did-registration
  e.g. feat/blockchain/proof-of-work-contract
  e.g. feat/orchestration/langgraph-interrupt

fix/<scope>/<short-desc>    Bug fix branches
  e.g. fix/gateway/sse-reconnect-memory-leak

chore/<desc>                Maintenance (deps, configs, docs)
  e.g. chore/update-alloy-1.2

contracts/<desc>            Solidity changes (require contracts reviewer)
  e.g. contracts/add-badge-nft

infra/<desc>                Infrastructure changes
  e.g. infra/add-k8s-manifests

release/v<major>.<minor>.<patch>   Release preparation branches
```

### 15.2 CODEOWNERS

```
# .github/CODEOWNERS
*                   @mergit-io/core-reviewers
contracts/          @mergit-io/core-reviewers @mergit-io/contracts-reviewers
services/blockchain-indexer/  @mergit-io/core-reviewers @mergit-io/contracts-reviewers
infra/k8s/          @mergit-io/infra-team
```

### 15.3 PR Template

```markdown
## Summary
<!-- What does this change do? Link the issue: Closes #N -->

## Type of Change
- [ ] feat: new feature
- [ ] fix: bug fix
- [ ] chore: maintenance / dependencies
- [ ] contracts: Solidity changes
- [ ] infra: infrastructure

## Checklist
- [ ] Tests added or updated
- [ ] `cargo clippy` clean (for Rust changes)
- [ ] `forge test` passes (for contract changes)
- [ ] `slither` output reviewed (for contract changes)
- [ ] No secrets committed
- [ ] ARCHITECTURE.md updated if service topology changed
- [ ] `deployments/{chain_id}.json` updated if contracts redeployed

## Blockchain Impact
<!-- If contracts changed: gas diff? New deployment needed? Role changes? -->
<!-- If touching oracle key path: security review requested? -->

## Test Plan
<!-- Steps to test this change manually -->

## Screenshots / Proof (if applicable)
<!-- Monad explorer link if testing on-chain, UI screenshot if frontend -->
```

### 15.4 Versioning and Release

**Version scheme:** Semantic versioning on the monorepo as a whole.
- MAJOR: breaking API or contract changes requiring migration
- MINOR: new features, new tools, new agent roles
- PATCH: bug fixes, dependency updates

**Changelog:** Generated via `git-cliff` using conventional commits. Commit message format:
```
feat(identity): add DID rotation endpoint
fix(blockchain-indexer): handle NONCE_TOO_LOW on retry
chore(deps): update alloy to 1.2.0
contracts(AgentPassport): add badge metadata extension
```

**Contract versioning:** Tracked in `foundry.toml` NatSpec and in `deployments/{chain_id}.json`. Contracts are **non-upgradeable in Phase 1–3** — this is a deliberate design decision. Immutability of `ProofOfWork.sol` and `AuditTrail.sol` is the source of the auditability guarantee. Upgrades in Phase 5 are migration-based: deploy new contract, migrate state, update oracle addresses, update `deployments/{chain_id}.json`.

---

## 16. Monad Blockchain Setup

### 16.1 Network Configuration

| Parameter | Testnet | Mainnet (future) |
|-----------|---------|-----------------|
| Network Name | Monad Testnet | Monad Mainnet |
| Chain ID | 10143 | 143 |
| RPC HTTP | `https://testnet-rpc.monad.xyz` | TBD (QuickNode paid) |
| RPC WSS | `wss://testnet-rpc.monad.xyz` | TBD |
| Block Explorer | `https://testnet.monadexplorer.com` | `https://monadscan.com` |
| Faucet | `https://faucet.monad.xyz` | N/A |
| Avg Block Time | ~1s | ~1s |

### 16.2 Wallet Setup (Step-by-Step)

```bash
# Step 1: Generate three dedicated wallets
cast wallet new  # → deployer
cast wallet new  # → oracle (blockchain-indexer signing key)
cast wallet new  # → admin (for multi-sig enrollment later)

# Step 2: Store in encrypted Foundry keystores
cast wallet import monad-deployer  --interactive  # enter private key, set password
cast wallet import monad-oracle    --interactive
cast wallet import monad-admin     --interactive

# Step 3: Verify keystore contents
cast wallet list   # should show monad-deployer, monad-oracle, monad-admin

# NEVER commit private keys — only commit wallet addresses
```

### 16.3 Acquire Testnet MON

**Required balance:** 5+ MON on deployer (for 4 contract deployments + role grants), 1+ MON on oracle (for ongoing TX fees).

**Faucet options (priority order):**
1. `https://faucet.monad.xyz` — 10 MON if wallet holds ≥0.001 ETH Mainnet
2. `https://faucet.quicknode.com/monad` — 1 MON / 12h, no ETH requirement
3. `https://www.alchemy.com/faucets/monad-testnet` — drip faucet
4. `https://faucets.chain.link/monad-testnet` — Chainlink faucet

### 16.4 Foundry Project Setup

```bash
# From repo root
forge init --template monad-developers/foundry-monad contracts/
cd contracts/
forge install OpenZeppelin/openzeppelin-contracts

# Verify setup
forge build    # should compile with 0 errors
forge test     # should show 0 tests (until written), exit 0
```

**foundry.toml:**
```toml
[profile.default]
src              = "src"
out              = "out"
libs             = ["lib"]
optimizer        = true
optimizer_runs   = 200
via_ir           = false

[profile.default.rpc_endpoints]
monad_testnet = "https://testnet-rpc.monad.xyz"
monad_mainnet = "${MONAD_RPC_HTTP}"

[profile.default.etherscan]
monad_testnet = { key = "${MONAD_EXPLORER_API_KEY}", url = "https://testnet.monadexplorer.com/api" }

[fmt]
line_length = 100
```

### 16.5 Environment Variables Reference

```bash
# .env (gitignored — never commit)

# Monad
MONAD_RPC_HTTP=https://testnet-rpc.monad.xyz
MONAD_RPC_WSS=wss://testnet-rpc.monad.xyz
MONAD_CHAIN_ID=10143
MONAD_EXPLORER_API_KEY=...

# Contract Addresses (populated after deployment, also in deployments/10143.json)
AGENT_PASSPORT_ADDRESS=0x...
PROOF_OF_WORK_ADDRESS=0x...
REPUTATION_REGISTRY_ADDRESS=0x...
AUDIT_TRAIL_ADDRESS=0x...

# Wallet addresses (public keys — safe to share, but store consistently)
IDENTITY_SERVICE_ADDRESS=0x...   # deployer's derived address for MINTER_ROLE
INDEXER_ORACLE_ADDRESS=0x...     # oracle wallet address for RECORDER/ORACLE/WRITER roles

# Oracle signing key (loaded via encrypted keystore, never as plaintext)
ORACLE_KEYSTORE_PATH=/run/secrets/oracle-keystore
ORACLE_KEYSTORE_PASS=...         # loaded from Docker secret or vault

# AI Providers
ANTHROPIC_API_KEY=...
GROQ_API_KEY=...
OPENAI_API_KEY=...             # fallback

# GitHub
GITHUB_TOKEN=...
GITHUB_WEBHOOK_SECRET=...

# Search
BRAVE_API_KEY=...

# Database
POSTGRES_PASSWORD=...
REDIS_PASSWORD=...             # optional for local dev

# Auth
JWT_SECRET=...                 # HS256 secret for agent tokens
```

---

## 17. Delivery Phases and Roadmap

### Phase 0 — Foundation (Days 1–3)

**Goal:** Working local development environment and monorepo skeleton.

**Deliverables:**
- `Cargo.toml` workspace root with all service members and shared dependency versions
- `pyproject.toml` uv workspace root
- `libs/rust-common/`: `AgentId`, `GoalId`, `TaskId` newtype wrappers; shared error enum; JWT validation utility
- `libs/proto/`: `identity.proto`, `blockchain.proto`, `reputation.proto` with `tonic-build` `build.rs`
- `libs/python-common/`: Pydantic models, gRPC client stubs generated from protos
- `infra/docker-compose.yml`: PostgreSQL 16 + pgvector, Redis 7, pgBouncer, Jaeger, Prometheus, Grafana
- `infra/docker-compose.test.yml`: isolated integration test environment
- `sqlx` migrations: complete PostgreSQL schema (all tables defined in Section 8.2)
- GitHub repo setup: branch protection on `main`, CODEOWNERS, PR template, dependabot, CI pipeline skeletons
- `scripts/setup-dev.sh`: one-command bootstrap (checks dependencies, starts compose, runs migrations)

**Success criteria:** `./scripts/setup-dev.sh` runs to completion. All 4 services (api-gateway, orchestration, accounts-service, blockchain-indexer) `Cargo.toml` files compile clean. `docker compose up` brings up PostgreSQL, Redis, and observability stack.

**Estimated effort:** 2–3 dev-days  
**Dependencies:** None — this phase unblocks all subsequent work.

---

### Phase 1 — Core Agent Execution (Days 4–10)

**Deliverables:**
- `services/api-gateway/`: JWT middleware, Redis rate limiter, SSE multiplexer subscribing to Redis Streams, routing to orchestration/identity/reputation, health endpoint
- `services/orchestration/`: LangGraph `StateGraph` with all 6 nodes (retrieve_memories, plan, dag_router, researcher/writer/coder/integrator, submit_proofs, finalize), `AsyncPostgresSaver` checkpoint wiring, interrupt/resume REST endpoint
- `tool-server-github/`: `github_ops`, `github_pr`, `wait_webhook` — all with Redis cache
- `tool-server-code/`: `code_exec` (Docker sandbox) and `file_ops`
- `tool-server-search/`: `web_search` (Brave API + fallback) and `http_request`
- LiteLLM integration in orchestration: multi-provider routing, health tracking in Redis
- Redis LLM response cache (exact + semantic hash layers) wired into `litellm_call` wrapper
- gRPC Python clients for identity and blockchain-indexer (stubs generated, `IdentityServiceStub` used in tool servers)
- End-to-end integration test: `POST /goals` → LangGraph decompose → task execution → `GET /goals/{id}` returns completed result

**Success criteria:** Submit "Write a summary of the README for repo X" → agents complete → result returned via API. SSE stream delivers task events in real time. LLM cache hit on second identical goal.

**Estimated effort:** 5–6 dev-days
**Dependencies:** Phase 0 complete; PostgreSQL and Redis running.

---

### Phase 2 — Identity and Blockchain (Days 11–17)

**Goal:** Every agent has an on-chain identity. Every completed task generates an on-chain proof.

**Deliverables:**
- `contracts/src/`: all four Solidity contracts written, unit tests (`forge test -vvv`), Slither static analysis clean, source-verified on Monad testnet
- `contracts/script/Deploy.s.sol`: deploys in correct order, grants roles, writes to `deployments/10143.json`
- `services/identity/`: DID generation, capability registry CRUD, `mintPassport` alloy call on agent registration, `VerifyCapability` gRPC endpoint, moka in-process cache
- `services/blockchain-indexer/`: alloy WebSocket event subscription for all 4 contracts, PostgreSQL event sync, idempotency table, gRPC `SubmitProof` and `LogAction` endpoints, oracle signing with nonce management and TX retry
- `proof_node` in LangGraph: calls `blockchain-indexer gRPC SubmitProof` on goal completion for each completed task
- `AuditTrail` events: each tool server calls `blockchain-indexer gRPC LogAction` on every tool execution
- `deployments/10143.json` committed to repository

**Success criteria:** Register an agent → passport NFT visible on `testnet.monadexplorer.com`. Complete a goal → `ProofOfWork.proofs[taskId]` on-chain matches `SHA-256(result_json)`. `AuditTrail` events appear in block explorer for every tool call.

**Estimated effort:** 6–7 dev-days
**Dependencies:** Phase 1 complete; Monad testnet wallets funded with sufficient MON.

---

### Phase 3 — Reputation and Badges (Days 18–22)

**Goal:** Agents accumulate verifiable on-chain reputation scores and earn badges.

**Deliverables:**
- `services/reputation/`: composite score calculation (5 components, weighted formula), batch job (tokio interval 10min), PostgreSQL partitioned `reputation_snapshots` table, `UpdateScore` gRPC call to blockchain-indexer, Redis leaderboard sorted set update
- Badge engine: "Verified Run" badge triggers after 5 consecutive successful proofs, writes to `badges` table, calls identity service to update passport metadata
- Leaderboard REST endpoints (`/reputation/leaderboard`, `/reputation/agents/{id}`)
- Score history endpoint with date range filtering
- Reputation service gRPC client in api-gateway routing
- Score verifiability CLI: `mergit verify-score <agent_address>` — fetches PostgreSQL breakdown, re-computes `componentHash`, compares to on-chain value, prints pass/fail

**Success criteria:** Complete 5 consecutive goals → badge issued and visible on-chain. Score visible in `ReputationRegistry` on block explorer. `mergit verify-score` passes for any agent. Leaderboard updates within 10 minutes of completed goal.

**Estimated effort:** 4–5 dev-days
**Dependencies:** Phase 2 complete (requires ProofOfWork events to flow into PostgreSQL).

---

### Phase 4 — Frontend (Days 23–27)

**Goal:** A production-quality UI that surfaces agent execution, on-chain proofs, and reputation in real time.

**Note:** Custom visual design will be provided by the user. This phase scaffolds the structure and wires up all data connections. Design implementation follows design asset delivery.

**Deliverables:**
- Clean Vite + React TypeScript + Tailwind CSS scaffold (wagmi v2 + viem for wallet connection)
- Typed API client (`src/lib/api.ts`) for all gateway endpoints
- Typed SSE hook (`src/hooks/useSSE.ts`) with auto-reconnect logic and typed event parsing
- Component stubs (logic-complete, design TBD):
  - `GoalSubmitForm` — text input, submit, shows planning status
  - `GoalDetail` — task DAG visualization (React Flow), live SSE event feed
  - `InterruptModal` — surfaces when `interrupt.pending` event arrives via SSE; posts resume decision
  - `AgentPassportCard` — displays DID, NFT tokenId, capability hash, task stats, block explorer link
  - `ProofOfWorkFeed` — live SSE stream of `proof.submitted` events with tx hashes and explorer links
  - `ReputationLeaderboard` — sorted agent list with scores and badges
  - `VerifiedRunBadge` — badge display with on-chain mint tx link
- wagmi config (`src/lib/wagmi.ts`): MetaMask/Rabby → Monad testnet (Chain ID 10143)
- Environment config (`VITE_API_BASE_URL`, `VITE_MONAD_RPC_URL`, `VITE_CHAIN_ID`)

**Success criteria:** Submit goal from UI → see live task events in SSE feed → approve PR interrupt from modal → see proof tx hash with explorer link. Wallet connects to Monad testnet. Leaderboard displays agent scores.

**Estimated effort:** 4–5 dev-days (structure + wiring) + additional effort for design implementation
**Dependencies:** Phase 1 (SSE/API), Phase 3 (reputation/badge data).

---

### Phase 5 — Production Hardening (Days 28+)

**Goal:** System is observable, secure, horizontally scalable, and ready for mainnet deployment.

**Deliverables:**
- Kubernetes manifests (`infra/k8s/`): Deployments, Services, HPA (CPU 70% threshold, min 2 / max 10), PDB (minAvailable: 1), ResourceQuotas
- OpenTelemetry SDK in all Rust services and Python orchestration → Grafana Tempo
- Prometheus scrape endpoints in all Rust services → dashboards: API gateway (p50/p95/p99 latency, error rate, SSE client count), Orchestration (active goals, LLM cache hit rate, interrupt queue depth), Blockchain-indexer (chain sync lag, TX submission latency, oracle key balance), Reputation (score update batch duration, leaderboard write latency)
- HashiCorp Vault integration: oracle keystore, API keys, DB credentials (replaces Docker secrets)
- `k6` load tests: 100 concurrent goal submissions, 1000 concurrent SSE clients
- Gnosis Safe 2-of-3 multi-sig setup for oracle key rotation capability
- Mainnet deployment checklist: contract audit, gas estimation at 10x load, QuickNode paid RPC upgrade, `deployments/143.json` update, key rotation into Vault

**Estimated effort:** 8–10 dev-days (ongoing, parallelizable with user onboarding)

---

## 18. Open Questions and Decisions Log

| # | Question | Status | Decision / Notes |
|---|----------|--------|-----------------|
| 1 | Project branding name — "Mergit" or different product name? | **Decided** | "Mergit" confirmed |
| 2 | Frontend visual design | **Open** | User will provide design assets — Phase 4 scaffolds structure first |
| 3 | Badge NFT: extend `AgentPassport.sol` with badge metadata, or deploy a separate `Badges.sol`? | **Open** | Lean toward separate contract (cleaner Monad parallel EVM story); decide in Phase 3 |
| 4 | LLM semantic cache: use `text-embedding-3-small` (OpenAI) or an open-source model via Groq? | **Open** | `text-embedding-3-small` costs $0.02/1M tokens — acceptable; alternative: `nomic-embed-text` via Ollama for zero cost |
| 5 | Code execution sandbox: Docker-in-Docker (simpler) or gVisor/Firecracker (more secure)? | **Open** | Docker-in-Docker for Phase 1-4; gVisor for Phase 5 hardening |
| 6 | Reputation batch interval: 10 minutes (current plan) or event-driven? | **Open** | 10-minute batch is simpler; event-driven reduces score staleness but adds complexity — revisit in Phase 3 |
| 7 | Monad testnet → Mainnet migration timeline | **Open** | Post-launch (Phase 5+); depends on MON mainnet availability and contract audit |
| 8 | Multi-agent coordination: can multiple agent instances share the same DID? | **Decided** | No — one DID per agent instance. Parallel task execution within a single LangGraph thread does not create multiple DIDs |
| 9 | `spawn_goal` tool: should child goals have their own DID or inherit parent's? | **Open** | Lean toward inheriting parent's DID with a child-goal tag in the proof — simpler identity model |
| 10 | GitHub webhook authentication: HMAC-SHA256 signature validation | **Decided** | Required from day 1 in `tool-server-github`; reject all unverified webhooks |
| 11 | Merge identity + reputation into accounts-service (Python/FastAPI) | **Decided** | Both services are I/O-bound, no independent scaling pressure; reduces Cargo crates, Docker images, CI lanes. Promote to standalone in Phase 5 if needed. |
| 12 | Non-blocking enqueue_proofs_node via stream:proof_queue | **Decided** | Blocking on chain confirmation added 5-10s to goal completion; async enqueue + proof-worker separates concerns |
| 13 | Batch audit logging: stream:audit_buffer + 10s Merkle ticker | **Decided** | Synchronous per-call LogAction serialized on indexer nonce ordering, adding latency per tool call |
| 14 | Capability JWT for local verification at tool servers | **Decided** | Eliminates per-call RPC to identity service; cap:revoked pub/sub provides near-instant revocation |
| 15 | Keep gRPC (tonic) on blockchain-indexer UpdateScore path; drop elsewhere | **Decided** | UpdateScore is a batch job (10-min interval), gRPC justified for Protobuf encoding + HTTP/2; all other internal calls use HTTP/JSON |
| 16 | HMAC X-Internal-Auth for Phases 1–4; mTLS deferred to Phase 5 | **Decided** | mTLS cert rotation setup consumes 2-3 days of Phase 1-2 budget inside a private Docker network with equivalent security |

---

*This document is the source of truth for Mergit's architecture, features, and delivery plan. It will be updated as decisions are made and requirements evolve. Major changes require a PR with the `chore/prd-update` branch prefix and approval from project leads.*
