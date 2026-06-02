# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the documentation repository for **Mergit** — an AI agent economy platform built for a Monad blockchain hackathon. It currently contains only the Product Requirements Document (`PRD.md`). All implementation lives (or will live) in separate repositories under the `mergit-io` GitHub organization.

## Project Architecture (from PRD.md)

Mergit is a polyglot monorepo with 5 services and a frontend:

| Service | Language | Role |
|---------|----------|------|
| `api-gateway` | Rust / Axum 0.7 | Public ingress, JWT auth, SSE multiplexer, Redis rate limiting |
| `orchestration` | Python / LangGraph | Goal decomposition, task DAG execution, LLM calls, checkpointing |
| `identity` | Rust / Axum + alloy | DID registration, ERC-721 AgentPassport minting, capability gating |
| `blockchain-indexer` | Rust / alloy | Oracle bridge: writes proofs/scores to Monad, indexes contract events |
| `reputation` | Rust / Axum | Composite score computation, badge engine, Redis leaderboard |
| `frontend` | React TypeScript / Vite + Tailwind | SSE event feed, interrupt modals, wagmi wallet (Monad testnet) |

**Data layer:** PostgreSQL 16 + pgvector (durable), Redis 7 (volatile/pub-sub/streams). pgBouncer in transaction mode sits in front of Postgres.

**Blockchain:** Monad testnet (chain ID 10143). Four contracts — `AgentPassport.sol` (ERC-721 soulbound), `ProofOfWork.sol` (idempotent proof ledger), `ReputationRegistry.sol` (oracle-updated scores), `AuditTrail.sol` (events-only, zero SSTORE). Solidity managed with Foundry.

**Internal comms:** gRPC (Protobuf) between Rust services and orchestration. HTTP REST from api-gateway to orchestration. Redis Streams for SSE fan-out.

## Key Design Decisions

- **LangGraph orchestration:** `AsyncPostgresSaver` for checkpointing. Parallel task dispatch via `Send` API. `interrupt()` primitive for human-in-the-loop approval gates (PR creation, credential requests, security gates).
- **LLM routing:** LiteLLM with a 4-provider fallback chain (`groq/llama-4-maverick` → `groq/llama-3.3-70b` → `anthropic/claude-sonnet-4-6` → `openai/gpt-4o`). Provider health tracked in Redis.
- **Caching:** Two-layer LLM cache (exact SHA-256 hash + semantic pgvector cosine similarity >0.97). Tool output caching keyed on `tool:{name}:{sha256(args)}`. In-process `moka` cache (Rust) → Redis → source.
- **On-chain proof integrity:** Every `ProofOfWork.resultHash` is `SHA-256(result_json)`. Every `ReputationRegistry.componentHash` is `SHA-256(component_breakdown_json)`. Anyone can re-derive the hash from the off-chain data and compare.
- **Oracle trust:** `ReputationRegistry` enforces 20% max delta per score update (anti-manipulation). `ProofOfWork.recordProof` is idempotent on `taskId`. Admin roles secured behind Gnosis Safe 2-of-3.
- **Contracts are non-upgradeable** through Phase 3 — immutability is the auditability guarantee.

## Delivery Phases

| Phase | Scope | Days |
|-------|-------|------|
| 0 | Monorepo scaffold, Docker Compose, DB migrations, CI skeletons | 1–3 |
| 1 | api-gateway + orchestration + 3 tool servers — end-to-end goal execution | 4–10 |
| 2 | identity service + blockchain-indexer + 4 Solidity contracts on Monad testnet | 11–17 |
| 3 | reputation service + badge engine + on-chain score updates | 18–22 |
| 4 | React frontend — SSE feed, interrupt modals, wallet integration | 23–27 |
| 5 | K8s manifests, OpenTelemetry, k6 load tests, Vault, mainnet prep | 28+ |

## Branch and Commit Conventions

Branches: `feat/<scope>/<short-desc>`, `fix/<scope>/<desc>`, `chore/<desc>`, `contracts/<desc>`, `infra/<desc>`, `release/v*`.

Conventional commit prefixes: `feat`, `fix`, `chore`, `contracts`, `infra`.

`main` is always deployable; features merge to `develop` first. Linear history on `main` (squash for features, rebase for hotfixes).

Contract changes (`contracts/` and `services/blockchain-indexer/`) require `@mergit-io/contracts-reviewers` approval. Always run `slither` and include gas diff in the PR description.

## Environment Variables

All secrets live in `.env` (gitignored). After contract deployment, addresses go into `deployments/{chain_id}.json` (committed). Key env vars: `MONAD_RPC_HTTP`, `MONAD_RPC_WSS`, `MONAD_CHAIN_ID`, contract addresses, `ORACLE_KEYSTORE_PATH`, `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, `GITHUB_TOKEN`, `POSTGRES_PASSWORD`, `JWT_SECRET`. The oracle signing key is loaded from an encrypted Foundry keystore — never stored as plaintext.
