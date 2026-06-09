# Mergit — Repository Architecture

**Version:** 0.1.0  
**Organization:** [mergit-io](https://github.com/mergit-io)  
**Last Updated:** 2026-06-09  

This document is the authoritative reference for the directory structure, service layout, and cross-repo relationships of every repository in the `mergit-io` GitHub organization. When in doubt about where something lives, this file answers it.

---

## Table of Contents

1. [Organization Overview](#1-organization-overview)
2. [mergit — Backend Monorepo](#2-mergit--backend-monorepo)
3. [mergit-contracts — Solidity Contracts](#3-mergit-contracts--solidity-contracts)
4. [mergit-docs — Documentation](#4-mergit-docs--documentation)
5. [Cross-Repo Relationships](#5-cross-repo-relationships)
6. [Service Communication Map](#6-service-communication-map)
7. [Data Flow](#7-data-flow)
8. [Deployment Architecture](#8-deployment-architecture)
9. [CI/CD Per Repo](#9-cicd-per-repo)

---

## 1. Organization Overview

Three repos. Hard boundaries. No exceptions without a written ADR.

| Repo | Language(s) | License | Purpose |
|------|-------------|---------|---------|
| `mergit-io/mergit` | Rust, Python, TypeScript | AGPL-3.0 | All services + frontend + migrations + infra + CI/CD |
| `mergit-io/mergit-contracts` | Solidity (Foundry) | MIT | Smart contracts only — deployed independently, audited separately |
| `mergit-io/mergit-docs` | Markdown | CC BY 4.0 | PRD, architecture decisions, API docs |

**Why this split:**
- Contracts are MIT (not AGPL) because auditors and integrators require it. Mixing them into an AGPL repo creates license ambiguity.
- Docs are a separate repo to allow public contributions without granting AGPL rights over the backend.
- Everything else lives in `mergit` — no per-service repos until team growth creates real friction.

---

## 2. mergit — Backend Monorepo

**License:** AGPL-3.0  
**Languages:** Rust (api-gateway, blockchain-indexer, shared libs), Python (orchestration, accounts-service, tool servers), TypeScript (frontend)  
**Workspace managers:** Cargo workspace (Rust), uv workspace (Python), pnpm (frontend)

### 2.1 Full Directory Tree

```
mergit/
│
├── .github/
│   ├── workflows/
│   │   ├── ci-rust.yml              # cargo test + clippy + fmt check
│   │   ├── ci-python.yml            # mypy + pytest --cov ≥80%
│   │   ├── ci-frontend.yml          # tsc --noEmit + vitest
│   │   └── deploy.yml               # build images → push GHCR → SSH deploy
│   ├── CODEOWNERS                   # blockchain-indexer → contracts-reviewers
│   └── pull_request_template.md
│
├── libs/                            # Shared code — never deployed independently
│   │
│   ├── rust-common/                 # Shared Rust types used by all Rust services
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── ids.rs               # AgentId, GoalId, TaskId newtypes (UUID wrappers)
│   │       ├── error.rs             # Shared MergitError enum
│   │       └── jwt.rs               # JWT decode/validate utility (no HTTP dependency)
│   │
│   ├── proto/                       # Single source of truth for all gRPC contracts
│   │   ├── Cargo.toml               # prost + tonic-build
│   │   ├── build.rs                 # compiles .proto → Rust types at build time
│   │   └── proto/
│   │       ├── identity.proto       # VerifyCapability, RegisterAgent RPCs
│   │       ├── blockchain.proto     # SubmitProof, LogAction RPCs
│   │       └── reputation.proto     # UpdateScore, GetScore RPCs
│   │
│   └── python-common/               # Shared Python types + generated gRPC stubs
│       ├── pyproject.toml
│       └── src/
│           └── mergit_common/
│               ├── __init__.py
│               ├── models.py        # Pydantic models (Agent, Goal, Task, Proof)
│               ├── ids.py           # AgentId, GoalId, TaskId wrappers
│               └── grpc/            # Generated stubs (git-ignored; scripts/gen-proto.sh writes here)
│                   ├── identity_pb2.py
│                   ├── identity_pb2_grpc.py
│                   ├── blockchain_pb2.py
│                   ├── blockchain_pb2_grpc.py
│                   ├── reputation_pb2.py
│                   └── reputation_pb2_grpc.py
│
├── services/
│   │
│   ├── api-gateway/                 # Rust / Axum 0.7 — public ingress
│   │   ├── Cargo.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       ├── main.rs              # Axum app bootstrap, tracing init
│   │       ├── config.rs            # Config from env vars
│   │       ├── auth/
│   │       │   ├── mod.rs
│   │       │   └── middleware.rs    # JWT extractor middleware
│   │       ├── routes/
│   │       │   ├── mod.rs
│   │       │   ├── goals.rs         # POST /goals, GET /goals/:id
│   │       │   ├── agents.rs        # GET /agents/:id, GET /agents/:id/passport
│   │       │   ├── reputation.rs    # GET /reputation/leaderboard, /reputation/agents/:id
│   │       │   └── health.rs        # GET /health, GET /ready
│   │       ├── sse/
│   │       │   ├── mod.rs
│   │       │   └── multiplexer.rs   # Redis Streams → SSE fan-out per goal_id
│   │       ├── rate_limit/
│   │       │   └── mod.rs           # Redis token bucket per JWT subject
│   │       └── proxy/
│   │           └── mod.rs           # HTTP reverse proxy to orchestration/accounts/reputation
│   │
│   ├── orchestration/               # Python / LangGraph — goal decomposition + execution
│   │   ├── pyproject.toml
│   │   ├── Dockerfile               # Runs the orchestration FastAPI service
│   │   ├── Dockerfile.tool-server   # Shared base image; TOOL_SERVER build arg selects entrypoint
│   │   └── src/
│   │       └── orchestration/
│   │           ├── __init__.py
│   │           ├── main.py          # FastAPI: POST /goals/execute, POST /goals/:id/resume
│   │           ├── config.py
│   │           ├── graph/
│   │           │   ├── state.py     # GoalState TypedDict — single state object through all nodes
│   │           │   ├── nodes.py     # 6 nodes: retrieve_memories, plan, dag_router,
│   │           │   │                #   execute (researcher/writer/coder/integrator),
│   │           │   │                #   submit_proofs, finalize
│   │           │   └── graph.py     # StateGraph wiring, interrupt() gates, AsyncPostgresSaver
│   │           ├── tools/
│   │           │   ├── base.py      # ToolResult, BaseTool, Redis cache decorator
│   │           │   ├── github/
│   │           │   │   ├── github_ops.py     # repo read, issue CRUD
│   │           │   │   ├── github_pr.py      # PR creation, review request
│   │           │   │   └── wait_webhook.py   # poll/block for CI status
│   │           │   ├── code/
│   │           │   │   ├── code_exec.py      # Docker sandbox execution
│   │           │   │   └── file_ops.py       # read/write/delete in agent workspace
│   │           │   └── search/
│   │           │       ├── web_search.py     # Brave API + DuckDuckGo fallback
│   │           │       └── http_request.py   # general HTTP GET/POST
│   │           └── llm/
│   │               ├── router.py    # LiteLLM 4-provider chain: groq/llama-4-maverick →
│   │               │                #   groq/llama-3.3-70b → claude-sonnet-4-6 → gpt-4o
│   │               └── cache.py     # Layer 1: exact SHA-256 | Layer 2: pgvector cosine >0.97
│   │
│   ├── accounts-service/            # Python / FastAPI — identity + reputation (merged service)
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   └── src/
│   │       └── accounts/
│   │           ├── __init__.py
│   │           ├── main.py          # FastAPI app, lifespan, router mounts
│   │           ├── config.py
│   │           ├── identity/
│   │           │   ├── router.py    # POST /agents (register), GET /agents/:id/did
│   │           │   ├── did.py       # W3C DID generation (did:mergit:<uuid>)
│   │           │   ├── passport.py  # mintPassport alloy HTTP call → blockchain-indexer
│   │           │   └── capabilities.py  # capability registry CRUD, VerifyCapability gRPC server
│   │           └── reputation/
│   │               ├── router.py    # GET /reputation/leaderboard, /reputation/agents/:id/history
│   │               ├── scorer.py    # 5-component composite score formula
│   │               ├── badges.py    # badge engine: "Verified Run" trigger after 5 consecutive proofs
│   │               ├── leaderboard.py  # Redis sorted set read/write
│   │               └── batch.py     # periodic score batch job → UpdateScore gRPC call
│   │
│   └── blockchain-indexer/          # Rust / alloy — oracle bridge + on-chain writer
│       ├── Cargo.toml
│       ├── Dockerfile
│       └── src/
│           ├── main.rs              # tokio runtime, gRPC server start, indexer loop start
│           ├── config.rs
│           ├── indexer/
│           │   ├── mod.rs
│           │   └── subscriber.rs    # alloy WS subscription for all 4 contract events
│           ├── oracle/
│           │   ├── mod.rs
│           │   ├── signer.rs        # load keystore, sign TX, nonce management
│           │   └── retry.rs         # TX submission with exponential backoff + replacement TX
│           ├── sync/
│           │   ├── mod.rs
│           │   └── pg_sync.rs       # write contract events to PostgreSQL, idempotency table
│           └── grpc/
│               ├── mod.rs
│               ├── submit_proof.rs  # SubmitProof RPC handler → ProofOfWork.sol
│               └── log_action.rs    # LogAction RPC handler → AuditTrail.sol
│
├── frontend/                        # React TypeScript / Vite + Tailwind CSS
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── vite.config.ts
│   ├── tailwind.config.ts
│   ├── tsconfig.json
│   ├── tsconfig.node.json
│   ├── Dockerfile                   # multi-stage: build → nginx serve
│   ├── public/
│   │   └── favicon.ico
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── env.d.ts                 # VITE_* env var types
│       ├── lib/
│       │   ├── api.ts               # typed REST client (all gateway endpoints)
│       │   ├── wagmi.ts             # wagmi v2 config — MetaMask/Rabby → Monad testnet (10143)
│       │   └── sse.ts               # SSE EventSource utilities
│       ├── hooks/
│       │   ├── useSSE.ts            # SSE hook with auto-reconnect + typed event parsing
│       │   ├── useGoal.ts           # goal state + SSE subscription
│       │   └── useWallet.ts         # wagmi wallet connection helpers
│       ├── components/
│       │   ├── GoalSubmitForm/      # text input → POST /goals → redirect to GoalDetail
│       │   ├── GoalDetail/          # React Flow task DAG + live SSE event feed
│       │   ├── InterruptModal/      # surfaces on interrupt.pending SSE event; POST /goals/:id/resume
│       │   ├── AgentPassportCard/   # DID, NFT tokenId, capability hash, task stats, explorer link
│       │   ├── ProofOfWorkFeed/     # live proof.submitted SSE stream with tx hashes
│       │   ├── ReputationLeaderboard/  # paginated agent list with scores + badges
│       │   └── VerifiedRunBadge/    # badge display + on-chain mint tx link
│       └── pages/
│           ├── Home.tsx
│           ├── Goal.tsx
│           └── Agent.tsx
│
├── migrations/                      # sqlx PostgreSQL migrations — all schema, one place
│   ├── 0001_agents.sql              # agents, dids, capabilities tables
│   ├── 0002_goals.sql               # goals table
│   ├── 0003_tasks.sql               # tasks table (DAG nodes)
│   ├── 0004_tool_cache.sql          # tool output cache table
│   ├── 0005_llm_cache.sql           # llm_cache + llm_cache_vectors (pgvector)
│   ├── 0006_reputation.sql          # reputation_scores, reputation_snapshots (partitioned), badges
│   ├── 0007_audit_events.sql        # audit_events (mirrored from AuditTrail.sol)
│   └── 0008_blockchain_events.sql   # raw contract events + idempotency table
│
├── infra/
│   ├── docker-compose.yml           # full dev stack (see §8.1)
│   ├── docker-compose.test.yml      # isolated integration test environment
│   ├── nginx/
│   │   ├── nginx.conf               # reverse proxy + SSL termination
│   │   └── ssl/                     # certs (gitignored; provisioned by certbot on VPS)
│   ├── prometheus/
│   │   └── prometheus.yml           # scrape configs for all Rust services
│   ├── grafana/
│   │   └── dashboards/
│   │       ├── api-gateway.json
│   │       ├── orchestration.json
│   │       ├── blockchain-indexer.json
│   │       └── reputation.json
│   └── k8s/                         # Phase 5 — Kubernetes manifests (placeholder)
│       └── .gitkeep
│
├── scripts/
│   ├── setup-dev.sh                 # check deps → docker compose up → run migrations → seed
│   ├── gen-proto.sh                 # compile .proto → Rust (via build.rs) + Python stubs
│   └── migrate.sh                   # sqlx migrate run --source migrations/
│
├── deployments/                     # Contract addresses — committed, not secret
│   └── 10143.json                   # Monad testnet addresses (populated after Phase 2 deploy)
│
├── Cargo.toml                       # Rust workspace root
│                                    #   members: services/api-gateway, services/blockchain-indexer,
│                                    #            libs/rust-common, libs/proto
├── pyproject.toml                   # uv workspace root
│                                    #   members: services/orchestration, services/accounts-service,
│                                    #            libs/python-common
├── .env.example                     # all env vars documented with descriptions
├── .gitignore
├── CLAUDE.md                        # Claude Code guidance for this repo
└── LICENSE                          # AGPL-3.0
```

### 2.2 Rust Workspace Members

```toml
# Cargo.toml (workspace root)
[workspace]
members = [
    "services/api-gateway",
    "services/blockchain-indexer",
    "libs/rust-common",
    "libs/proto",
]
resolver = "2"

[workspace.dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
alloy = { version = "0.3", features = ["full"] }
tonic = "0.11"
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-native-tls", "uuid"] }
redis = { version = "0.25", features = ["tokio-comp"] }
serde = { version = "1", features = ["derive"] }
uuid = { version = "1", features = ["v4", "serde"] }
tracing = "0.1"
```

### 2.3 Python Workspace Members

```toml
# pyproject.toml (uv workspace root)
[tool.uv.workspace]
members = [
    "services/orchestration",
    "services/accounts-service",
    "libs/python-common",
]
```

---

## 3. mergit-contracts — Solidity Contracts

**License:** MIT  
**Toolchain:** Foundry (forge, cast, anvil)  
**Target network:** Monad testnet (chain ID 10143), Monad mainnet (Phase 5+)

### 3.1 Full Directory Tree

```
mergit-contracts/
│
├── .github/
│   └── workflows/
│       ├── ci.yml               # forge build + forge test -vvv + slither + gas snapshot
│       └── deploy-testnet.yml   # manual trigger: forge script Deploy → Monad testnet
│
├── src/                         # Production contracts
│   ├── AgentPassport.sol        # ERC-721 soulbound NFT — one per agent, non-transferable
│   │                            #   stores: DID, capabilityHash, taskCount, registeredAt
│   ├── ProofOfWork.sol          # Idempotent proof ledger
│   │                            #   recordProof(taskId, agentId, resultHash) — idempotent on taskId
│   │                            #   resultHash = SHA-256(result_json)
│   ├── ReputationRegistry.sol   # Oracle-updated composite scores
│   │                            #   updateScore(agentId, score, componentHash) — 20% max delta guard
│   │                            #   componentHash = SHA-256(component_breakdown_json)
│   ├── AuditTrail.sol           # Events only — zero SSTORE, pure audit log
│   │                            #   logAction(agentId, toolName, argsHash, resultHash)
│   ├── StakeRegistry.sol        # Agent stake deposit/withdrawal (Phase 3+)
│   └── interfaces/
│       ├── IAgentPassport.sol
│       ├── IProofOfWork.sol
│       ├── IReputationRegistry.sol
│       └── IAuditTrail.sol
│
├── script/                      # Foundry deployment scripts
│   ├── Deploy.s.sol             # Deploys all contracts in dependency order:
│   │                            #   1. AgentPassport  2. AuditTrail  3. ProofOfWork
│   │                            #   4. ReputationRegistry  5. StakeRegistry
│   │                            #   Writes addresses to deployments/{chainId}.json
│   ├── GrantRoles.s.sol         # Post-deploy: grant ORACLE_ROLE to blockchain-indexer key
│   └── Verify.s.sol             # Source verify all contracts on Monad explorer
│
├── test/                        # Foundry unit tests
│   ├── AgentPassport.t.sol
│   ├── ProofOfWork.t.sol
│   ├── ReputationRegistry.t.sol
│   ├── AuditTrail.t.sol
│   └── helpers/
│       └── TestBase.sol         # shared setUp(), fixtures, mock oracle
│
├── lib/                         # Foundry dependencies (forge install)
│   ├── openzeppelin-contracts/  # ERC-721, AccessControl, ReentrancyGuard
│   └── forge-std/               # Test utilities
│
├── deployments/                 # Committed — not secrets
│   └── 10143.json               # { "AgentPassport": "0x...", "ProofOfWork": "0x...", ... }
│
├── foundry.toml                 # Foundry config: optimizer, via-ir, solc version
├── remappings.txt               # @openzeppelin → lib/openzeppelin-contracts/contracts/
├── .env.example                 # MONAD_RPC_HTTP, ORACLE_KEYSTORE_PATH, ETHERSCAN_API_KEY
├── .gitignore                   # ignores broadcast/, cache/, out/, .env
└── LICENSE                      # MIT
```

### 3.2 Contract Dependency Order

```
AgentPassport     (no deps)
AuditTrail        (no deps — events only)
ProofOfWork       (reads AgentPassport for existence check)
ReputationRegistry (reads ProofOfWork event count, writes require ORACLE_ROLE)
StakeRegistry     (reads AgentPassport for agent validity)
```

### 3.3 Role Model

| Role | Holder | Grants |
|------|--------|--------|
| `DEFAULT_ADMIN_ROLE` | Gnosis Safe 2-of-3 | add/remove all roles |
| `MINTER_ROLE` | accounts-service wallet | `AgentPassport.mint()` |
| `ORACLE_ROLE` | blockchain-indexer wallet | `ReputationRegistry.updateScore()`, `ProofOfWork.recordProof()` |
| `PAUSER_ROLE` | Gnosis Safe 2-of-3 | emergency pause on AgentPassport, ProofOfWork |

---

## 4. mergit-docs — Documentation

**License:** CC BY 4.0  
**Format:** Markdown

### 4.1 Full Directory Tree

```
mergit-docs/
│
├── docs/
│   ├── PRD.md                   # Product Requirements Document (authoritative)
│   ├── CLAUDE.md                # Claude Code guidance for all mergit-io repos
│   ├── architect.md             # ← This file — all repo directory structures
│   ├── decisions/               # Architecture Decision Records (ADRs)
│   │   ├── 001-three-repo-split.md
│   │   ├── 002-agpl-license-backend.md
│   │   ├── 003-accounts-service-merged.md   # why identity+reputation are one service
│   │   ├── 004-docker-compose-phases-1-4.md
│   │   └── 005-non-upgradeable-contracts.md
│   └── api/                     # API reference (generated or hand-written)
│       ├── gateway.md           # api-gateway REST endpoints
│       ├── orchestration.md     # orchestration internal REST endpoints
│       └── grpc.md              # gRPC service definitions
│
├── .github/
│   └── README.md
├── README.md
└── LICENSE                      # CC BY 4.0
```

---

## 5. Cross-Repo Relationships

```
mergit-contracts
      │
      │  deployments/10143.json
      │  (addresses committed to both repos after Phase 2 deploy)
      ▼
   mergit
      │
      │  libs/proto/*.proto
      │  (single source; compiled to Rust + Python stubs at build time)
      │
      ├── services/api-gateway        (Rust)
      ├── services/orchestration      (Python)  ──── gRPC ──→  services/accounts-service
      ├── services/accounts-service   (Python)  ──── gRPC ──→  services/blockchain-indexer
      ├── services/blockchain-indexer (Rust)    ──── alloy ──→  Monad chain
      └── frontend                    (TS)      ──── wagmi ──→  Monad chain (read-only)

mergit-docs
      │
      └── (read by humans + Claude Code; no build-time dependency)
```

**Key rule:** `deployments/10143.json` lives in `mergit-contracts` as the source of truth and is **copied** (not symlinked) into `mergit/deployments/10143.json` after each contract deployment. The copy in `mergit` is what the services read at runtime.

---

## 6. Service Communication Map

```
                        ┌─────────────────────────────────────────────┐
                        │                  INTERNET                    │
                        └─────────────────────────────────────────────┘
                                           │ HTTPS
                                           ▼
                                    ┌─────────────┐
                                    │    Nginx     │  reverse proxy + SSL
                                    └──────┬───────┘
                                           │
                          ┌────────────────┼─────────────────┐
                          │ HTTP           │ HTTP             │ SSE
                          ▼                ▼                  ▼
                   ┌─────────────────────────────────────────────┐
                   │                api-gateway (Rust)            │
                   │  JWT auth · rate limit · SSE multiplexer     │
                   └──────┬───────────────┬──────────────────────┘
                          │ HTTP          │ HTTP
               ┌──────────▼──┐      ┌────▼──────────────────┐
               │orchestration│      │   accounts-service     │
               │  (Python)   │      │   (Python)             │
               │  LangGraph  │      │   identity + reputation│
               └──────┬──────┘      └────────────────────────┘
                      │ gRPC                    │ gRPC
                      │              ┌──────────▼─────────────┐
                      └─────────────►│  blockchain-indexer    │
                                     │  (Rust)                │
                                     │  alloy oracle bridge   │
                                     └──────────┬─────────────┘
                                                │ alloy WSS/HTTP
                                                ▼
                                        ┌──────────────┐
                                        │ Monad chain  │
                                        │ (chain 10143)│
                                        └──────────────┘

Shared infrastructure (all services connect):
  ┌─────────────────────┐     ┌──────────────────┐
  │ PostgreSQL 16        │     │ Redis 7          │
  │ + pgvector           │     │ Streams · cache  │
  │ + pgBouncer          │     │ rate limit · pub │
  └─────────────────────┘     └──────────────────┘
```

### Protocol summary

| Caller | Callee | Protocol | Purpose |
|--------|--------|----------|---------|
| Nginx | api-gateway | HTTP/1.1 | Reverse proxy |
| api-gateway | orchestration | HTTP REST | Forward goal requests |
| api-gateway | accounts-service | HTTP REST | Forward agent/reputation requests |
| api-gateway | Redis | Redis protocol | SSE fan-out (XREAD on Streams) |
| orchestration | accounts-service | gRPC | VerifyCapability before tool execution |
| orchestration | blockchain-indexer | gRPC | SubmitProof, LogAction per tool call |
| accounts-service | blockchain-indexer | gRPC | UpdateScore (reputation batch job) |
| blockchain-indexer | Monad | alloy WSS | Event subscription (all 4 contracts) |
| blockchain-indexer | Monad | alloy HTTP | TX submission (proofs, scores) |
| frontend | api-gateway | HTTP REST + SSE | All user-facing operations |
| frontend | Monad | wagmi/viem RPC | Read-only wallet + balance |

---

## 7. Data Flow

### 7.1 Goal Execution (Happy Path)

```
User submits goal
      │
      ▼
api-gateway: POST /goals
  → validate JWT
  → write goal row (status=pending) to PostgreSQL
  → HTTP POST to orchestration /goals/execute
      │
      ▼
orchestration: LangGraph StateGraph
  → retrieve_memories node: fetch agent memory from PostgreSQL
  → plan node: LiteLLM call → task list
  → dag_router node: dispatch tasks in parallel via Send API
  → execute nodes (per task):
      - tool call → Redis cache check (SHA-256 exact match)
      - if miss → execute tool → cache result → return
      - gRPC LogAction → blockchain-indexer → AuditTrail.sol event
  → [interrupt gate]: if PR creation or cred request → pause, publish SSE interrupt.pending
  → submit_proofs node: gRPC SubmitProof per completed task → blockchain-indexer
  → finalize node: update goal status=completed in PostgreSQL
      │
      ▼
blockchain-indexer: SubmitProof
  → sign TX with oracle key
  → submit to ProofOfWork.sol: recordProof(taskId, agentId, SHA-256(result_json))
  → write event to PostgreSQL audit_events
  → publish proof.submitted to Redis Stream for goal_id
      │
      ▼
api-gateway SSE multiplexer:
  → XREAD from Redis Stream
  → push proof.submitted event to all SSE clients subscribed to goal_id
      │
      ▼
Frontend: live event feed updates
```

### 7.2 Agent Registration

```
POST /agents  (name, capabilities[])
      │
accounts-service identity module:
  → generate did:mergit:<uuid>
  → write agent + capabilities to PostgreSQL
  → HTTP POST to blockchain-indexer: mint AgentPassport NFT
      │
blockchain-indexer:
  → alloy call: AgentPassport.mint(ownerAddress, did, capabilityHash)
  → on PassportMinted event: write tokenId to PostgreSQL agents table
      │
accounts-service response: { agentId, did, tokenId, explorerUrl }
```

### 7.3 Reputation Score Update (Batch)

```
accounts-service batch job (every 10 min, tokio interval):
  → query PostgreSQL: compute 5-component score for each agent
      (successRate, proofCount, taskVolume, taskDiversity, stakeRatio)
  → compute componentHash = SHA-256(JSON component breakdown)
  → gRPC UpdateScore → blockchain-indexer
      │
blockchain-indexer:
  → alloy call: ReputationRegistry.updateScore(agentId, newScore, componentHash)
      (20% max delta guard enforced on-chain)
  → on ScoreUpdated event: write to PostgreSQL reputation_snapshots
  → update Redis sorted set: ZADD reputation:leaderboard score agentId
      │
api-gateway /reputation/leaderboard:
  → ZREVRANGE reputation:leaderboard 0 99 → return top 100
```

---

## 8. Deployment Architecture

### 8.1 Phase 0–4: Docker Compose on VPS

```
Hetzner CX31 (4 vCPU / 8 GB / 80 GB SSD, ~€10/month)
┌──────────────────────────────────────────────────────┐
│  Cloudflare DNS → api.mergit.io                      │
│                                                       │
│  Nginx                                               │
│  ├── / → frontend:3000                               │
│  ├── /api/ → api-gateway:8000                        │
│  └── SSL via Certbot (Let's Encrypt)                 │
│                                                       │
│  Docker Compose stack (infra/docker-compose.yml):    │
│  ├── postgres (pgvector/pgvector:pg16)               │
│  ├── pgbouncer (bitnami/pgbouncer:1.22)              │
│  ├── redis (redis:7-alpine)                          │
│  ├── api-gateway                                     │
│  ├── orchestration                                   │
│  ├── tool-server-github                              │
│  ├── tool-server-code                                │
│  ├── tool-server-search                              │
│  ├── accounts-service                                │
│  ├── blockchain-indexer                              │
│  ├── frontend                                        │
│  ├── jaeger (tracing UI)                             │
│  ├── prometheus                                      │
│  └── grafana                                         │
└──────────────────────────────────────────────────────┘
```

**Secrets management (Phases 0–4):** `.env` file on VPS, gitignored. Oracle signing key stored as encrypted Foundry keystore file (`/run/secrets/oracle-keystore`), mounted via Docker secrets.

### 8.2 Phase 5+: Kubernetes

```
infra/k8s/
├── namespaces/
├── deployments/        # one per service, with HPA (CPU 70%, min 2 / max 10)
├── services/
├── ingress/            # nginx ingress controller
├── configmaps/
├── secrets/            # sealed-secrets or Vault Agent injector
└── monitoring/         # Prometheus operator, Grafana, Tempo (traces)
```

---

## 9. CI/CD Per Repo

### 9.1 mergit — ci-rust.yml

```
Trigger: push to any branch, PR to main or develop
Jobs:
  test:
    - cargo fmt --check
    - cargo clippy -- -D warnings
    - cargo test --workspace --locked
  docker:
    - (on push to main only)
    - docker build services/api-gateway → push ghcr.io/mergit-io/api-gateway:sha
    - docker build services/blockchain-indexer → push ghcr.io/mergit-io/blockchain-indexer:sha
```

### 9.2 mergit — ci-python.yml

```
Trigger: push to any branch, PR to main or develop
Jobs:
  test:
    - uv run mypy services/orchestration/src
    - uv run mypy services/accounts-service/src
    - uv run pytest services/orchestration/tests --cov --cov-fail-under=80
    - uv run pytest services/accounts-service/tests --cov --cov-fail-under=80
  docker:
    - (on push to main only)
    - docker build services/orchestration → push ghcr.io/mergit-io/orchestration:sha
    - docker build services/accounts-service → push ghcr.io/mergit-io/accounts-service:sha
```

### 9.3 mergit — ci-frontend.yml

```
Trigger: push to any branch, PR to main or develop
Jobs:
  test:
    - pnpm install --frozen-lockfile
    - pnpm tsc --noEmit
    - pnpm vitest run
    - pnpm build
  docker:
    - (on push to main only)
    - docker build frontend → push ghcr.io/mergit-io/frontend:sha
```

### 9.4 mergit — deploy.yml

```
Trigger: push to main (after all CI jobs pass)
Jobs:
  deploy:
    - SSH into VPS
    - docker compose pull
    - docker compose up -d --remove-orphans
    - docker compose run --rm api-gateway ./migrate.sh  # run pending sqlx migrations
```

### 9.5 mergit-contracts — ci.yml

```
Trigger: push to any branch, PR to main
Jobs:
  test:
    - forge build
    - forge test -vvv
    - forge snapshot --check          # fail if gas increases unexpectedly
  security:
    - slither src/ --exclude-informational
  deploy-testnet:
    - (manual trigger only)
    - forge script script/Deploy.s.sol --rpc-url $MONAD_RPC_HTTP --broadcast
    - copy deployments/10143.json to mergit repo (via PR)
```

---

*This document is generated from decisions made in the PRD and in conversation. Update it whenever a structural decision changes — do not let it drift from the actual repos.*
