# prd-hackathon.md — Launch Constraints and Revert Plan

**Companion to:** `PRD.md`  
**Scope:** What is DIFFERENT during Phases 1–4 from the production architecture described in PRD.md, and what changes when Phase 5 runs.

---

## 1. Purpose

This file captures deliberate simplifications accepted for the hackathon launch (Phases 1–4). The production-target architecture is defined in PRD.md. Nothing in this file overrides PRD.md for Phase 5+. Reviewers reading PRD.md should treat it as the authoritative design; this document records where the running system deviates from it during the launch window and the exact steps to close each gap.

---

## 2. Internal Authentication

**PRD.md reference:** §6.1 (diagram), §7.1 (api-gateway), §13.2 (security).  
PRD.md describes mTLS between all internal services as the internal auth mechanism. That is the production target. For launch it is **replaced** by a shared-secret HMAC header on the private Docker network.

| | Detail |
|---|---|
| **Phases 1–4** | All internal service-to-service requests carry an `X-Internal-Auth` header containing `HMAC-SHA256(request_body, INTERNAL_SECRET)`. `INTERNAL_SECRET` is a shared env var injected via Docker Compose. Services reject requests missing or failing verification. This is acceptable only because all internal traffic stays on a non-routable Docker bridge network with no external exposure. |
| **Phase 5** | Replace with mTLS. Each service gets a cert-manager-issued certificate. The api-gateway and all Rust/Python services validate peer certificates. `X-Internal-Auth` header removed from all code paths. |
| **Revert trigger** | Before K8s rollout. cert-manager is a prerequisite. Feature flag: `INTERNAL_AUTH_MODE=mtls` (default `hmac` in Phase 1–4). |

---

## 3. Service Deployment

| | Detail |
|---|---|
| **Phases 1–4** | Docker Compose with `deploy.replicas` for stateless services (api-gateway, orchestration, tool servers, identity, reputation). Single-instance only for blockchain-indexer — see note below. |
| **Phase 5** | Kubernetes with HorizontalPodAutoscaler: CPU 70% threshold, min 2 / max 10 replicas per service. PodDisruptionBudget minAvailable: 1. ResourceQuotas per namespace. |
| **Revert trigger** | K8s manifests in `infra/k8s/` land and staging cluster is validated with k6 load tests. Docker Compose remains for local dev. |

**blockchain-indexer exception (both phases):** Only one active indexer instance may submit transactions at any time to avoid duplicate TX submissions. Phase 1–4: single replica. Phase 5: Redis leader-election lock (`SET indexer:leader {instance_id} NX EX 30` + heartbeat); non-leader runs in standby mode.

---

## 4. Database Connection

| | Detail |
|---|---|
| **Phases 1–4** | pgBouncer in **transaction mode** sits in front of PostgreSQL. All `sqlx` pools bounded to 10 connections per service. Prevents connection exhaustion under spike load at the cost of disallowing advisory locks and session-level statements. |
| **Phase 5** | Evaluate per-service: if any service requires `pg_advisory_lock`, `SET LOCAL`, or `LISTEN`/`NOTIFY`, switch that service to pgBouncer **session mode** or a direct pool bypassing pgBouncer. Default remains transaction mode for services that do not need session semantics. |
| **Revert trigger** | Needs assessment at Phase 5 start. Switch only if a blocking operational issue is observed or a feature explicitly requires session-level semantics. No change otherwise. |

---

## 5. Observability

**Note on compose file:** The Jaeger (`jaegertracing/all-in-one:1.57`) and Prometheus/Grafana containers are present in the Docker Compose from Phase 0. The distinction below is about what is **wired** in service code, not what infrastructure is running.

| | Detail |
|---|---|
| **Phases 1–4** | Prometheus scrape endpoints in all Rust services (`metrics-exporter-prometheus`). Basic Grafana dashboards for request latency, error rate, SSE client count. OpenTelemetry SDK dependency present in service code but trace export **not wired** — spans are no-ops. Jaeger container runs but receives no traffic. |
| **Phase 5** | Wire OTel SDK in all Rust services and Python orchestration. Export traces to Jaeger (compose) or Grafana Tempo (K8s — PRD §14.3 targets Tempo for K8s; Jaeger for compose dev is consistent with the existing compose setup). Distributed trace context propagated via W3C `traceparent` header through the full request path. |
| **Revert trigger** | Phase 5 hardening sprint. Requires setting `OTEL_EXPORTER_OTLP_ENDPOINT` in service env vars and enabling the exporter in code. |

---

## 6. Secrets Management

| | Detail |
|---|---|
| **Phases 1–4** | Most secrets in `.env` (gitignored), injected by Docker Compose `environment:` blocks. The oracle signing keystore is already handled via a **Docker secret** (`secrets: [oracle-keystore]` in compose, loaded at `/run/secrets/oracle-keystore`). This is the only launch-phase secret stored outside `.env`. |
| **Phase 5** | HashiCorp Vault replaces both `.env` and Docker secrets. All service startup secrets (DB credentials, API keys, JWT secret, oracle keystore password) are fetched from Vault at container start via the Vault Agent sidecar or direct API call. `.env` is retained for local developer use only, never used in staging or production. |
| **Revert trigger** | Vault cluster provisioned, Vault Agent injector installed in K8s, secrets migrated. All K8s Deployments updated to remove `env:` direct value references and add Vault annotations. |

---

## 7. Contract Upgradeability

| | Detail |
|---|---|
| **Through Phase 3** | All four contracts (`AgentPassport.sol`, `ProofOfWork.sol`, `ReputationRegistry.sol`, `AuditTrail.sol`) are **non-upgradeable**. This is a deliberate design decision: immutability of `ProofOfWork.sol` and `AuditTrail.sol` is the auditability guarantee. On-chain proofs and audit events cannot be silently rewritten by a future upgrade. |
| **Phase 5** | PRD.md §15.4 leans toward **migration-based upgrades** (deploy new contract, migrate state, update oracle addresses, update `deployments/{chain_id}.json`). A transparent proxy pattern (OpenZeppelin `TransparentUpgradeableProxy`) is the alternative under consideration. Decision deferred pending contract audit results and team alignment on the auditability tradeoff. Gnosis Safe 2-of-3 multi-sig setup (currently deferred to Phase 5) is a prerequisite for any upgradeability model. |
| **Revert trigger** | Requires external or internal contract audit, explicit team decision on proxy vs migration, and Gnosis Safe enrollment of all three signers. |

---

## 8. Known Shortcuts

Deliberate simplifications accepted for launch that need revisiting post-hackathon.

| # | Area | Shortcut | Revisit In |
|---|------|----------|-----------|
| 1 | Redis topology | Docker Compose runs a single `redis:7-alpine` node. PRD diagram labels it "Redis 7 Cluster." No cluster, no replication, no sentinel. | Phase 5 (K8s) |
| 2 | Code execution sandbox | `code_exec` tool uses Docker-in-Docker (simpler but weaker isolation). PRD OQ #5 notes gVisor/Firecracker as the secure alternative. | Phase 5 hardening |
| 3 | Reputation batch interval | Fixed 10-minute `tokio::interval` batch. Event-driven scoring would reduce score staleness but adds complexity. PRD OQ #6 defers this decision to Phase 3 evaluation. | Phase 3 → Phase 5 |
| 4 | Oracle key management | Single oracle signing key for routine proof recording. Gnosis Safe 2-of-3 multi-sig for `DEFAULT_ADMIN_ROLE` (key rotation, emergency pause) is a Phase 5 deliverable. | Phase 5 |
| 5 | Badge contract design | Badge storage on `AgentPassport.sol` (metadata extension) vs separate `Badges.sol` not yet decided. PRD OQ #3. | Phase 3 start |
| 6 | Vector index tuning | `pgvector` ivfflat index uses `lists = 100`. PRD §8.1 notes to re-evaluate if p95 vector query exceeds 100ms at scale. | Phase 5 load testing |
| 7 | LLM cache invalidation | TTL expiry only — no active invalidation. Stale cached responses can persist until TTL. `X-Cache-Bypass: true` header provides a manual escape hatch. | Post-launch if cache correctness issues surface |
| 8 | Internal auth (mTLS) | Covered in §2 above. | Phase 5 |

---

*This document is updated as launch constraints change. It does not replace PRD.md. When a Phase 5 item is completed, the corresponding row here is marked resolved and dated.*
