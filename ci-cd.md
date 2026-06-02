# CI/CD Implementation Plan

**Project:** Mergit  
**Status:** Planning  
**Last Updated:** 2026-06-02  
**Budget constraint:** Zero — all tooling must use free tiers

---

## Table of Contents

1. [What is CI/CD?](#1-what-is-cicd)
2. [Recommended Free Stack](#2-recommended-free-stack)
3. [Hosting: Oracle Cloud Always Free](#3-hosting-oracle-cloud-always-free)
4. [Runners: GitHub-Hosted with Caching](#4-runners-github-hosted-with-caching)
5. [Path Filtering](#5-path-filtering)
6. [Secret Management](#6-secret-management)
7. [GitHub Environments Setup](#7-github-environments-setup)
8. [Workflow Overview](#8-workflow-overview)
9. [CI Workflows](#9-ci-workflows)
10. [CD Workflows](#10-cd-workflows)
11. [Manual Workflows](#11-manual-workflows)
12. [Caching Strategy](#12-caching-strategy)
13. [Smoke Test Script](#13-smoke-test-script)
14. [Setup Checklist](#14-setup-checklist)
15. [Phase 5 Path: Kubernetes](#15-phase-5-path-kubernetes)

---

## 1. What is CI/CD?

**CI (Continuous Integration)** means: every time someone pushes code or opens a pull request, an automated system runs checks — format checks, linting, tests, security audits — and tells you if something is broken *before* it gets merged. This catches bugs early when they are cheap to fix.

**CD (Continuous Deployment/Delivery)** means: when code lands on certain branches (like `develop` or tagged releases), the system automatically builds Docker images, pushes them to a registry, and deploys them to a server. No one has to manually SSH in and copy files.

For Mergit specifically:
- CI makes sure Rust services compile and clippy passes, Python orchestration types check and tests pass, Solidity contracts build and Slither finds no critical issues, and the frontend builds without errors — on every PR, before any merge.
- CD makes sure the staging server always reflects the latest `develop` branch, and production deploys happen predictably when you push a version tag.

---

## 2. Recommended Free Stack

| Component | Tool | Why |
|-----------|------|-----|
| CI/CD engine | **GitHub Actions** | Native GitHub integration, free for public repos (unlimited minutes), 2,000 free minutes/month for private repos |
| Docker image registry | **GitHub Container Registry (ghcr.io)** | Free for public repos, no rate limits, uses the same `GITHUB_TOKEN` (no extra secret needed) |
| Staging + production hosting | **Oracle Cloud Always Free** | The only cloud provider with a genuinely free *forever* tier powerful enough to run all Mergit services |
| Secret management | **GitHub Environments + GitHub Secrets** | Built into GitHub Actions, zero cost, supports per-environment secrets with protection rules |
| Runners | **GitHub-hosted + aggressive caching** | Zero maintenance, free minutes sufficient, Rust build times kept under 5 minutes with `actions/cache` |

### Why not the alternatives?

**Hosting alternatives:**
- **Render / Railway free tier:** services sleep after 15 minutes of inactivity, unsuitable for a persistent backend
- **Fly.io free tier:** 3 shared-CPU VMs with 256MB RAM each — not enough to run PostgreSQL + Redis + 5 services simultaneously
- **Heroku:** no free tier since 2022
- **Self-hosted VPS (e.g. Hetzner CX11):** ~€4/month — not free, and Oracle gives more resources for free

**Runner alternatives:**
- **Self-hosted runners:** faster Rust builds (full CPU, persistent cache), but you need a machine running 24/7 to host them. Since we're on Oracle Always Free anyway, you could self-host a runner later for speed — documented in [Phase 5 Path](#15-phase-5-path-kubernetes).
- **GitHub-hosted only without caching:** Rust compiles from scratch every run (~12–15 minutes), burning through your minutes allowance fast.

---

## 3. Hosting: Oracle Cloud Always Free

### What you get for free (forever)

Oracle Cloud's Always Free tier includes:
- **Ampere A1 ARM pool:** 4 OCPUs + 24 GB RAM total, which you can split across up to 4 VMs
- **2 AMD x86 VMs:** 1 OCPU + 1 GB RAM each (too small for most Mergit services — use ARM)
- **200 GB block storage total**
- **10 TB outbound bandwidth/month**
- No credit card required to continue after trial ends

For Mergit, the recommended split:

| VM | OCPUs | RAM | Purpose |
|----|-------|-----|---------|
| `mergit-staging` (ARM) | 2 | 12 GB | Staging: full Docker Compose stack |
| `mergit-prod` (ARM) | 2 | 12 GB | Production: full Docker Compose stack |

This gives both environments 2 ARM cores and 12 GB RAM — enough to run PostgreSQL, pgBouncer, Redis, api-gateway, orchestration, identity, blockchain-indexer, reputation, all 3 tool servers, and the frontend simultaneously.

### VM Setup Steps

These are one-time manual steps. After this, all deploys are automated.

```bash
# --- On your local machine ---

# 1. Create an Oracle Cloud account at cloud.oracle.com
#    Select "Always Free" during signup

# 2. Provision the ARM VM (do this twice: once for staging, once for prod)
#    In Oracle Console → Compute → Instances → Create Instance
#    - Image: Ubuntu 22.04 (recommended for Docker compatibility)
#    - Shape: Ampere A1 Flex → 2 OCPUs, 12 GB RAM
#    - Upload your SSH public key (cat ~/.ssh/id_rsa.pub)
#    - Note the public IP address

# 3. Open required ports in the Security List (or use Security Groups)
#    Oracle Console → VCN → Security Lists → Default → Add Ingress Rules:
#    Port 22  (SSH)   — source 0.0.0.0/0
#    Port 80  (HTTP)  — source 0.0.0.0/0
#    Port 443 (HTTPS) — source 0.0.0.0/0
#    Port 8000 (api-gateway, if no reverse proxy yet) — source 0.0.0.0/0

# 4. SSH in
ssh ubuntu@<your-oracle-vm-ip>

# --- On the Oracle VM ---

# 5. Install Docker + Docker Compose plugin
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker ubuntu   # allow ubuntu user to run docker without sudo
newgrp docker                    # apply group change immediately

# 6. Log in to GitHub Container Registry
echo <GITHUB_PAT_WITH_PACKAGES_READ> | docker login ghcr.io -u <your-github-username> --password-stdin
# Note: the deploy workflow will do this automatically, but manual login is needed for the first test

# 7. Clone the repo
git clone https://github.com/mergit-io/mergit.git ~/mergit
cd ~/mergit

# 8. Create the .env file (copy from .env.example and fill in secrets)
cp .env.example .env
nano .env   # fill in all required values
```

### GitHub Actions SSH Key Setup

The deploy workflow needs to SSH into the Oracle VMs. You need a **dedicated deploy SSH key** (not your personal key).

```bash
# On your local machine
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/mergit_deploy -N ""

# Copy the PUBLIC key to each Oracle VM
ssh-copy-id -i ~/.ssh/mergit_deploy.pub ubuntu@<staging-vm-ip>
ssh-copy-id -i ~/.ssh/mergit_deploy.pub ubuntu@<prod-vm-ip>

# Print the PRIVATE key — you'll paste this into GitHub Secrets
cat ~/.ssh/mergit_deploy
```

Store the private key content as the `STAGING_SSH_KEY` and `PROD_SSH_KEY` GitHub secrets (see Section 6).

---

## 4. Runners: GitHub-Hosted with Caching

**Recommendation: GitHub-hosted `ubuntu-latest` runners with `actions/cache`.**

### Comparison

| Runner type | Rust build time | Cost | Maintenance | Verdict |
|-------------|----------------|------|-------------|---------|
| GitHub-hosted, no cache | ~12–15 min | Free (public repo) | Zero | Too slow — burns minutes |
| **GitHub-hosted + cache** | **~3–5 min** | **Free (public repo)** | **Zero** | **✓ Use this** |
| Self-hosted on Oracle VM | ~2–4 min (native) | Free (VM already allocated) | You manage runner daemon | Good upgrade once Oracle VM is set up (Phase 5) |

With `actions/cache` caching `~/.cargo/registry`, `~/.cargo/git`, and the `target/` directory keyed on `Cargo.lock`, Rust incremental builds after the first run stay under 5 minutes. If the project grows and builds creep back up, migrating to a self-hosted runner on the Oracle VM is a straightforward upgrade — same YAML, just change `runs-on: ubuntu-latest` to `runs-on: self-hosted`.

### Adding a self-hosted runner (optional, for later)

```bash
# On the Oracle VM (after the VM is set up from Section 3)
# In GitHub: Settings → Actions → Runners → New self-hosted runner → Linux ARM64
# Follow the instructions — it's a ~5-command setup that installs the runner agent
# Enable as a systemd service so it survives reboots:
sudo ./svc.sh install
sudo ./svc.sh start
```

---

## 5. Path Filtering

**Path filtering = CI jobs only run when the relevant files actually changed.**

Without it: a one-line frontend typo fix triggers a full Rust build + all contract tests + Python type checks. This wastes CI minutes, makes PRs slower, and adds noise.

With path filtering:

| What changed | Jobs that run |
|---|---|
| `services/api-gateway/**` | ci-rust.yml |
| `services/orchestration/**` | ci-python.yml |
| `contracts/**` | ci-contracts.yml |
| `frontend/**` | ci-frontend.yml |
| `libs/rust-common/**` or `libs/proto/**` | ci-rust.yml (shared Rust code affects all Rust services) |
| `libs/python-common/**` | ci-python.yml |
| Multiple of the above | All relevant jobs run in parallel |

This is implemented via the `paths:` key in each workflow's `on:` trigger (shown in the workflow YAML below).

**One exception:** when you push to `develop` or `main`, ALL CI jobs run regardless of what changed. This ensures the integration branch is always fully verified.

---

## 6. Secret Management

### Where secrets live

All secrets are stored in **GitHub Secrets**, scoped per environment. Go to:
`GitHub repo → Settings → Secrets and variables → Actions`

**Repository-level secrets** (shared across all environments):

| Secret name | What it holds |
|-------------|--------------|
| `GHCR_TOKEN` | GitHub PAT with `write:packages` scope — used to push Docker images to ghcr.io. Note: you can also use `GITHUB_TOKEN` (auto-provided) for pushes from the same org. |

**Staging environment secrets** (only available to staging jobs):

| Secret name | What it holds |
|-------------|--------------|
| `STAGING_HOST` | IP address of the staging Oracle VM |
| `STAGING_USER` | SSH username (usually `ubuntu`) |
| `STAGING_SSH_KEY` | Private SSH key for deploy access (content of `~/.ssh/mergit_deploy`) |
| `STAGING_POSTGRES_PASSWORD` | PostgreSQL password for staging |
| `STAGING_JWT_SECRET` | JWT secret for staging |
| `STAGING_GROQ_API_KEY` | Groq API key |
| `STAGING_ANTHROPIC_API_KEY` | Anthropic API key |
| `STAGING_GITHUB_TOKEN` | GitHub token for tool-server-github |
| `STAGING_BRAVE_API_KEY` | Brave Search API key |
| `STAGING_MONAD_RPC_HTTP` | `https://testnet-rpc.monad.xyz` |
| `STAGING_MONAD_RPC_WSS` | `wss://testnet-rpc.monad.xyz` |

**Production environment secrets** (only available to production jobs, require manual approval):

| Secret name | What it holds |
|-------------|--------------|
| `PROD_HOST` | IP address of the production Oracle VM |
| `PROD_USER` | SSH username |
| `PROD_SSH_KEY` | Private SSH key for deploy access |
| `PROD_POSTGRES_PASSWORD` | PostgreSQL password for production |
| `PROD_JWT_SECRET` | JWT secret for production |
| `PROD_GROQ_API_KEY` | Groq API key |
| `PROD_ANTHROPIC_API_KEY` | Anthropic API key |
| `PROD_GITHUB_TOKEN` | GitHub token |
| `PROD_BRAVE_API_KEY` | Brave Search API key |
| `PROD_MONAD_RPC_HTTP` | Monad RPC URL |
| `PROD_MONAD_RPC_WSS` | Monad WSS URL |
| `PROD_DEPLOYER_PRIVATE_KEY` | Deployer wallet private key (for contract deployments — handle with extreme care) |
| `PROD_ORACLE_KEYSTORE_PASSWORD` | Password for the encrypted oracle keystore file |

> **Security note:** The deployer private key and oracle keystore password are the most sensitive secrets in the entire system. If you use a Gnosis Safe multi-sig for admin roles (as the PRD recommends), the deployer key only needs MON for gas — compromising it lets an attacker pay gas, not take over contracts.

### What about Vault (Phase 5)?

HashiCorp Vault is listed in the PRD for Phase 5 hardening. For now, GitHub Secrets is the right choice — it's free, well-integrated, and sufficient for a Phase 0–4 deployment. When Vault is set up in Phase 5, the workflow `env:` blocks will pull from Vault instead of GitHub Secrets, but the workflow structure stays the same.

---

## 7. GitHub Environments Setup

GitHub Environments add a **required manual approval gate** before a workflow can access that environment's secrets and run.

**Why this matters:** The production deploy workflow triggers on tag push, but you don't want it to SSH into prod and restart everything without someone reviewing it first. GitHub Environments let you require a specific person to click "Approve" in the GitHub UI before the deployment step runs.

### Setup steps

1. Go to `GitHub repo → Settings → Environments`
2. Create environment: `staging`
   - No protection rules needed (staging auto-deploys are fine)
   - Add all `STAGING_*` secrets here
3. Create environment: `production`
   - Enable **"Required reviewers"** → add yourself (and any teammates)
   - Enable **"Wait timer"** (optional, e.g., 5 minutes, to give you time to abort if you pushed the wrong tag)
   - Add all `PROD_*` secrets here

Now the `cd-production.yml` workflow will pause at the deploy step and send you a GitHub notification asking for approval. Only after you approve does it SSH into the production VM.

---

## 8. Workflow Overview

Workflows live in `.github/workflows/` in the monorepo root.

```
.github/workflows/
├── ci-rust.yml           # Rust services: fmt, clippy, test, audit
├── ci-python.yml         # Orchestration: ruff, mypy, pytest
├── ci-contracts.yml      # Solidity: forge fmt, build, test, slither
├── ci-frontend.yml       # React: lint, type-check, test, build
├── cd-staging.yml        # Auto-deploy to staging on push to develop
├── cd-production.yml     # Deploy to prod on tag push (requires approval)
└── deploy-contracts.yml  # Manual contract deployment workflow
```

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `ci-rust.yml` | PR and push to `main`/`develop` (if Rust files changed) | Checks format, lint, tests, security audit |
| `ci-python.yml` | PR and push to `main`/`develop` (if Python files changed) | Checks format, types, runs tests with coverage |
| `ci-contracts.yml` | PR and push to `main`/`develop` (if `contracts/` changed) | Checks format, builds, tests with gas report, Slither |
| `ci-frontend.yml` | PR and push to `main`/`develop` (if `frontend/` changed) | Lint, type-check, test, production build |
| `cd-staging.yml` | Push to `develop` | Builds + pushes all Docker images, deploys to staging VM |
| `cd-production.yml` | Push tag `v*.*.*` | Builds + pushes versioned images, requires approval, deploys to prod |
| `deploy-contracts.yml` | Manual (`workflow_dispatch`) | Deploys Solidity contracts to Monad, updates `deployments/{chain_id}.json` |

---

## 9. CI Workflows

### 9.1 ci-rust.yml

Covers all four Rust services via a single Cargo workspace run.

```yaml
# .github/workflows/ci-rust.yml
name: CI — Rust

on:
  pull_request:
    paths:
      - 'services/api-gateway/**'
      - 'services/identity/**'
      - 'services/blockchain-indexer/**'
      - 'services/reputation/**'
      - 'libs/rust-common/**'
      - 'libs/proto/**'
      - 'Cargo.toml'
      - 'Cargo.lock'
  push:
    branches: [main, develop]

jobs:
  check:
    name: Format, Lint, Test, Audit
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      # Cache Cargo registry, git deps, and compiled artifacts
      # Key is the OS + Cargo.lock hash so cache invalidates when deps change
      - name: Cache Cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          restore-keys: |
            ${{ runner.os }}-cargo-

      - name: Format check
        run: cargo fmt --all -- --check

      - name: Clippy (treat warnings as errors)
        run: cargo clippy --workspace --all-targets --all-features -- -D warnings

      - name: Tests
        run: cargo test --workspace --locked

      - name: Security audit
        run: |
          cargo install cargo-audit --quiet
          cargo audit

  docker-build-check:
    name: Docker Build Check (no push)
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api-gateway, identity, blockchain-indexer, reputation]

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image (no push — just verify it compiles)
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: false
          tags: mergit/${{ matrix.service }}:ci-check
```

### 9.2 ci-python.yml

```yaml
# .github/workflows/ci-python.yml
name: CI — Python / Orchestration

on:
  pull_request:
    paths:
      - 'services/orchestration/**'
      - 'libs/python-common/**'
  push:
    branches: [main, develop]

jobs:
  check:
    name: Lint, Types, Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install uv
        run: pip install uv

      - name: Cache uv packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/uv
          key: ${{ runner.os }}-uv-${{ hashFiles('services/orchestration/pyproject.toml') }}
          restore-keys: |
            ${{ runner.os }}-uv-

      - name: Install dependencies
        run: uv sync --all-extras
        working-directory: services/orchestration

      - name: Lint (ruff)
        run: uv run ruff check .
        working-directory: services/orchestration

      - name: Format check (ruff)
        run: uv run ruff format --check .
        working-directory: services/orchestration

      - name: Type check (mypy)
        run: uv run mypy src
        working-directory: services/orchestration

      - name: Tests with coverage
        run: uv run pytest tests --cov=src --cov-report=term-missing --cov-fail-under=80
        working-directory: services/orchestration

  docker-build-check:
    name: Docker Build Check
    runs-on: ubuntu-latest
    strategy:
      matrix:
        tool_server: [github, code, search]

    steps:
      - uses: actions/checkout@v4

      - name: Build tool-server-${{ matrix.tool_server }} (no push)
        uses: docker/build-push-action@v5
        with:
          context: ./services/orchestration
          file: ./services/orchestration/Dockerfile.tool-server
          push: false
          build-args: TOOL_SERVER=${{ matrix.tool_server }}
          tags: mergit/tool-server-${{ matrix.tool_server }}:ci-check
```

### 9.3 ci-contracts.yml

```yaml
# .github/workflows/ci-contracts.yml
name: CI — Solidity Contracts

on:
  pull_request:
    paths:
      - 'contracts/**'
  push:
    branches: [main, develop]

jobs:
  check:
    name: Format, Build, Test, Static Analysis
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive   # required for OpenZeppelin lib installed via forge install

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly   # use stable once it catches up to nightly features used

      - name: Cache Foundry artifacts
        uses: actions/cache@v4
        with:
          path: |
            ~/.foundry/cache
            contracts/cache
            contracts/out
          key: ${{ runner.os }}-foundry-${{ hashFiles('contracts/foundry.toml', 'contracts/lib/**') }}
          restore-keys: |
            ${{ runner.os }}-foundry-

      - name: Format check
        run: forge fmt --check
        working-directory: contracts

      - name: Build
        run: forge build
        working-directory: contracts

      - name: Tests with gas report
        run: forge test -vvv --gas-report
        working-directory: contracts

      - name: Install Slither
        run: pip install slither-analyzer

      - name: Slither static analysis
        # Use || true so the workflow doesn't fail on informational findings;
        # the uploaded report is reviewed manually in the PR.
        # Change to remove || true once you establish a clean baseline.
        run: slither . --json slither-report.json || true
        working-directory: contracts

      - name: Upload Slither report
        uses: actions/upload-artifact@v4
        with:
          name: slither-report-${{ github.sha }}
          path: contracts/slither-report.json
          retention-days: 30
```

> **Note on Slither:** The `|| true` flag means the step never fails even if Slither finds issues. This is intentional for the early phases — you want to see the report without blocking PRs. Once you've reviewed all existing findings and addressed the critical ones, remove `|| true` and let the job fail on high-severity findings.

### 9.4 ci-frontend.yml

```yaml
# .github/workflows/ci-frontend.yml
name: CI — Frontend

on:
  pull_request:
    paths:
      - 'frontend/**'
  push:
    branches: [main, develop]

jobs:
  check:
    name: Lint, Types, Test, Build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v3
        with:
          version: 9

      - name: Setup Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
          cache-dependency-path: frontend/pnpm-lock.yaml

      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        working-directory: frontend

      - name: Lint (ESLint)
        run: pnpm lint
        working-directory: frontend

      - name: Type check
        run: pnpm type-check   # should run: tsc --noEmit
        working-directory: frontend

      - name: Unit tests (Vitest)
        run: pnpm test --run
        working-directory: frontend

      - name: Production build
        run: pnpm build
        working-directory: frontend
        env:
          VITE_API_BASE_URL: http://localhost:8000
          VITE_MONAD_RPC_URL: https://testnet-rpc.monad.xyz
          VITE_CHAIN_ID: "10143"
```

---

## 10. CD Workflows

### 10.1 cd-staging.yml

Triggers on every push to `develop`. Builds all Docker images, pushes them to ghcr.io tagged `:staging`, then SSH's into the staging VM and rolls the stack.

```yaml
# .github/workflows/cd-staging.yml
name: CD — Staging

on:
  push:
    branches: [develop]

env:
  REGISTRY: ghcr.io
  ORG: mergit-io

jobs:
  # Build all Rust service images in parallel
  build-rust:
    name: Build ${{ matrix.service }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        service: [api-gateway, identity, blockchain-indexer, reputation]

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push ${{ matrix.service }}
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.service }}:staging
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,mode=max,scope=${{ matrix.service }}

  # Build orchestration and all tool-server images
  build-python:
    name: Build ${{ matrix.name }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        include:
          - name: orchestration
            dockerfile: Dockerfile
            build_args: ""
          - name: tool-server-github
            dockerfile: Dockerfile.tool-server
            build_args: "TOOL_SERVER=github"
          - name: tool-server-code
            dockerfile: Dockerfile.tool-server
            build_args: "TOOL_SERVER=code"
          - name: tool-server-search
            dockerfile: Dockerfile.tool-server
            build_args: "TOOL_SERVER=search"

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push ${{ matrix.name }}
        uses: docker/build-push-action@v5
        with:
          context: ./services/orchestration
          file: ./services/orchestration/${{ matrix.dockerfile }}
          push: true
          build-args: ${{ matrix.build_args }}
          tags: ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.name }}:staging
          cache-from: type=gha,scope=${{ matrix.name }}
          cache-to: type=gha,mode=max,scope=${{ matrix.name }}

  # Build frontend image
  build-frontend:
    name: Build frontend
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.ORG }}/frontend:staging
          cache-from: type=gha,scope=frontend
          cache-to: type=gha,mode=max,scope=frontend

  # Deploy to staging VM after all images are built
  deploy:
    name: Deploy to Staging
    needs: [build-rust, build-python, build-frontend]
    runs-on: ubuntu-latest
    environment: staging   # uses STAGING_* secrets

    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            set -e
            cd ~/mergit

            # Pull latest code (for docker-compose.yml changes)
            git fetch origin develop
            git checkout develop
            git reset --hard origin/develop

            # Log in to GHCR to pull updated images
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

            # Pull new images and restart services
            docker compose pull
            docker compose up -d --remove-orphans

            # Wait for api-gateway to be healthy
            echo "Waiting for services to start..."
            sleep 15

            # Health check
            curl --fail --silent http://localhost:8000/health | grep -q '"status":"ok"'
            echo "Staging deploy successful."

      - name: Run smoke tests
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            cd ~/mergit
            bash scripts/smoke-test.sh
```

### 10.2 cd-production.yml

Triggers on tag push (`v*.*.*`). Requires manual approval via the `production` GitHub Environment before executing the SSH deploy.

```yaml
# .github/workflows/cd-production.yml
name: CD — Production

on:
  push:
    tags:
      - 'v*.*.*'

env:
  REGISTRY: ghcr.io
  ORG: mergit-io

jobs:
  # Build and push versioned images (same pattern as staging)
  build-rust:
    name: Build ${{ matrix.service }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        service: [api-gateway, identity, blockchain-indexer, reputation]

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push ${{ matrix.service }}
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: true
          # Push both a version tag and :latest
          tags: |
            ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.service }}:${{ github.ref_name }}
            ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.service }}:latest
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,mode=max,scope=${{ matrix.service }}

  build-python:
    name: Build ${{ matrix.name }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    strategy:
      matrix:
        include:
          - name: orchestration
            dockerfile: Dockerfile
            build_args: ""
          - name: tool-server-github
            dockerfile: Dockerfile.tool-server
            build_args: "TOOL_SERVER=github"
          - name: tool-server-code
            dockerfile: Dockerfile.tool-server
            build_args: "TOOL_SERVER=code"
          - name: tool-server-search
            dockerfile: Dockerfile.tool-server
            build_args: "TOOL_SERVER=search"

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push ${{ matrix.name }}
        uses: docker/build-push-action@v5
        with:
          context: ./services/orchestration
          file: ./services/orchestration/${{ matrix.dockerfile }}
          push: true
          build-args: ${{ matrix.build_args }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.name }}:${{ github.ref_name }}
            ${{ env.REGISTRY }}/${{ env.ORG }}/${{ matrix.name }}:latest
          cache-from: type=gha,scope=${{ matrix.name }}
          cache-to: type=gha,mode=max,scope=${{ matrix.name }}

  build-frontend:
    name: Build frontend
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.ORG }}/frontend:${{ github.ref_name }}
            ${{ env.REGISTRY }}/${{ env.ORG }}/frontend:latest
          cache-from: type=gha,scope=frontend
          cache-to: type=gha,mode=max,scope=frontend

  # This job requires manual approval in the GitHub UI before it runs.
  # The "production" environment in GitHub Settings has required reviewers enabled.
  deploy:
    name: Deploy to Production
    needs: [build-rust, build-python, build-frontend]
    runs-on: ubuntu-latest
    environment: production   # ← PAUSES HERE until you approve in GitHub UI

    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          envs: GITHUB_ACTOR,GITHUB_REF_NAME
          script: |
            set -e
            cd ~/mergit

            # Check out the specific tagged version
            git fetch --tags
            git checkout $GITHUB_REF_NAME

            # Log in to GHCR
            echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u $GITHUB_ACTOR --password-stdin

            # Set the image tag in the environment so docker-compose uses the versioned image
            export IMAGE_TAG=$GITHUB_REF_NAME

            docker compose pull
            docker compose up -d --remove-orphans

            sleep 20

            # Health check all services
            curl --fail --silent http://localhost:8000/health | grep -q '"status":"ok"'
            echo "Production deploy $GITHUB_REF_NAME successful."

      - name: Run smoke tests
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd ~/mergit
            bash scripts/smoke-test.sh

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          tag_name: ${{ github.ref_name }}
```

---

## 11. Manual Workflows

### 11.1 deploy-contracts.yml

Contract deployment is intentionally **not automated** in the CD pipelines. Reasons:
1. Contracts are non-upgradeable through Phase 3 — a mistaken redeploy wastes MON and creates confusion about which addresses are canonical.
2. The deployer private key is the most sensitive secret in the system — minimizing how often it's used reduces risk.
3. Contract deployment requires a human to explicitly review what changed and confirm they intend to deploy.

This workflow is triggered manually from the GitHub Actions UI: `Actions → Deploy Contracts → Run workflow`.

```yaml
# .github/workflows/deploy-contracts.yml
name: Deploy Contracts (Manual)

on:
  workflow_dispatch:
    inputs:
      network:
        description: 'Target network'
        required: true
        default: 'monad_testnet'
        type: choice
        options:
          - monad_testnet
          - monad_mainnet
      dry_run:
        description: 'Dry run (simulate only, do not broadcast)'
        required: true
        default: true
        type: boolean

jobs:
  deploy:
    name: Deploy to ${{ inputs.network }}
    runs-on: ubuntu-latest
    environment: production   # requires manual approval secret access
    permissions:
      contents: write   # needed to commit updated deployments/{chain_id}.json

    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Install Foundry
        uses: foundry-rs/foundry-toolchain@v1
        with:
          version: nightly

      - name: Cache Foundry
        uses: actions/cache@v4
        with:
          path: |
            ~/.foundry/cache
            contracts/cache
            contracts/out
          key: ${{ runner.os }}-foundry-${{ hashFiles('contracts/foundry.toml') }}

      - name: Build contracts
        run: forge build
        working-directory: contracts

      - name: Run tests before deploy
        run: forge test -vvv
        working-directory: contracts

      - name: Deploy (or dry run)
        env:
          MONAD_RPC_HTTP: ${{ secrets.PROD_MONAD_RPC_HTTP }}
          DEPLOYER_PRIVATE_KEY: ${{ secrets.PROD_DEPLOYER_PRIVATE_KEY }}
          IDENTITY_SERVICE_ADDRESS: ${{ secrets.IDENTITY_SERVICE_ADDRESS }}
          INDEXER_ORACLE_ADDRESS: ${{ secrets.INDEXER_ORACLE_ADDRESS }}
          MONAD_EXPLORER_API_KEY: ${{ secrets.MONAD_EXPLORER_API_KEY }}
        run: |
          if [ "${{ inputs.dry_run }}" = "true" ]; then
            echo "DRY RUN — simulating deployment only"
            forge script script/Deploy.s.sol \
              --rpc-url $MONAD_RPC_HTTP \
              --private-key $DEPLOYER_PRIVATE_KEY
          else
            echo "BROADCASTING deployment to ${{ inputs.network }}"
            forge script script/Deploy.s.sol \
              --rpc-url $MONAD_RPC_HTTP \
              --private-key $DEPLOYER_PRIVATE_KEY \
              --broadcast --verify \
              --verifier-url https://testnet.monadexplorer.com/api
          fi
        working-directory: contracts

      - name: Commit updated deployment addresses
        if: ${{ inputs.dry_run == false }}
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add deployments/
          git diff --staged --quiet || git commit -m "chore(contracts): update deployment addresses for ${{ inputs.network }}"
          git push
```

---

## 12. Caching Strategy

Caching is what keeps CI fast. Without it, Rust compiles take 12–15 minutes and burn your GitHub Actions minutes budget. With good caching, most CI runs finish in 3–5 minutes.

### Cache keys explained

A **cache key** determines when the cache is valid or should be discarded. The pattern is:

```
${{ runner.os }}-<language>-${{ hashFiles('**/lockfile') }}
```

This means: "the cache is valid as long as the OS and the lockfile haven't changed." When you add a new dependency (which changes `Cargo.lock` or `pnpm-lock.yaml`), the hash changes, the old cache is discarded, and a fresh one is built.

The **restore-keys** are fallback patterns — if no exact match is found, GitHub Actions looks for the most recent cache matching a prefix. This is useful when the lockfile changes but most dependencies are shared with the previous version.

### Cache table

| Language | What's cached | Cache key | Invalidates when |
|----------|--------------|-----------|-----------------|
| Rust | `~/.cargo/registry`, `~/.cargo/git`, `target/` | `Cargo.lock` hash | Any dependency added/updated |
| Python (uv) | `~/.cache/uv` | `pyproject.toml` hash | Any Python dep added/updated |
| Foundry | `~/.foundry/cache`, `contracts/cache`, `contracts/out` | `foundry.toml` + `lib/` hash | Any lib installed/updated |
| Node/pnpm | pnpm store | `pnpm-lock.yaml` hash | Any JS dep added/updated |
| Docker layers | GitHub Actions cache (via Buildx `cache-from: type=gha`) | layer content hash | Any Dockerfile layer changes |

### Docker layer caching

The Docker build jobs in the CD workflows use:
```yaml
cache-from: type=gha,scope=<service>
cache-to: type=gha,mode=max,scope=<service>
```

This stores Docker layer caches in GitHub Actions cache storage. When you rebuild an image and only the application code changed (not the base image or dependencies), only the final layer rebuilds. A Rust binary rebuild from a cached layer takes ~2–3 minutes instead of ~8–10.

---

## 13. Smoke Test Script

The CD workflows call `scripts/smoke-test.sh` after each deploy. This script lives in the monorepo root.

The script must be created in Phase 0 and expanded as services are built. A basic version for Phase 1:

```bash
#!/usr/bin/env bash
# scripts/smoke-test.sh
# Run after each deploy to verify the system is healthy.
# Exits non-zero on any failure, which fails the CD workflow.

set -euo pipefail

BASE_URL="${BASE_URL:-http://localhost:8000}"
MAX_RETRIES=5
RETRY_DELAY=5

echo "Running smoke tests against $BASE_URL"

# Helper: retry a curl command up to MAX_RETRIES times
check() {
  local label="$1"
  local url="$2"
  local expected="$3"

  for i in $(seq 1 $MAX_RETRIES); do
    response=$(curl --silent --fail "$url" 2>/dev/null) && break
    echo "  Attempt $i/$MAX_RETRIES failed for $label, retrying in ${RETRY_DELAY}s..."
    sleep $RETRY_DELAY
  done

  if ! echo "$response" | grep -q "$expected"; then
    echo "FAIL: $label — expected '$expected' in response"
    echo "Response: $response"
    exit 1
  fi
  echo "PASS: $label"
}

# Phase 1: api-gateway health
check "api-gateway /health"   "$BASE_URL/health"        '"status":"ok"'

# Phase 2+: add more checks as services come online
# check "identity service"    "$BASE_URL/agents/ping"   '"ok"'
# check "reputation service"  "$BASE_URL/reputation/ping" '"ok"'

echo "All smoke tests passed."
```

---

## 14. Setup Checklist

Work through this once before Phase 0 begins. Items are one-time setup — not repeated per deploy.

### GitHub repository setup

- [ ] Create the `mergit` monorepo at `github.com/mergit-io/mergit`
- [ ] Set repo to **public** (enables free unlimited GitHub Actions minutes and free ghcr.io storage)
- [ ] Enable **branch protection** on `main`: require 2 approvals, require CI to pass, require linear history
- [ ] Enable **branch protection** on `develop`: require CI to pass
- [ ] Set up **CODEOWNERS** (see PRD Section 15.2)
- [ ] Create environments: `staging` and `production` (see Section 7)
- [ ] Add all **GitHub Secrets** (see Section 6)

### Oracle Cloud setup

- [ ] Create Oracle Cloud account at cloud.oracle.com
- [ ] Provision `mergit-staging` ARM VM (2 OCPU, 12 GB)
- [ ] Provision `mergit-prod` ARM VM (2 OCPU, 12 GB)
- [ ] Open ports 22, 80, 443, 8000 in Oracle Security Lists for both VMs
- [ ] Install Docker + Docker Compose plugin on both VMs (see Section 3)
- [ ] Generate deploy SSH key pair (see Section 3) and add to `STAGING_SSH_KEY` / `PROD_SSH_KEY`
- [ ] Clone the repo on both VMs under `~/mergit`
- [ ] Create `.env` files on both VMs from `.env.example`

### Workflow files

- [ ] Create `.github/workflows/ci-rust.yml`
- [ ] Create `.github/workflows/ci-python.yml`
- [ ] Create `.github/workflows/ci-contracts.yml`
- [ ] Create `.github/workflows/ci-frontend.yml`
- [ ] Create `.github/workflows/cd-staging.yml`
- [ ] Create `.github/workflows/cd-production.yml`
- [ ] Create `.github/workflows/deploy-contracts.yml`
- [ ] Create `scripts/smoke-test.sh` (make executable: `chmod +x scripts/smoke-test.sh`)

### First-time validation

- [ ] Open a test PR modifying a Rust file → verify `ci-rust.yml` triggers
- [ ] Open a test PR modifying a frontend file → verify only `ci-frontend.yml` triggers (path filtering works)
- [ ] Push to `develop` → verify `cd-staging.yml` triggers and deploys successfully
- [ ] Create a test tag `v0.0.1-test` → verify `cd-production.yml` triggers and pauses for approval
- [ ] Approve the production deploy → verify it deploys successfully → delete test tag

---

## 15. Phase 5 Path: Kubernetes

The Docker Compose + SSH deploy described above is appropriate for Phases 0–4. For Phase 5 (production hardening), the plan is to migrate to Kubernetes as described in PRD Section 14.3.

### When to migrate

Migrate from Docker Compose to Kubernetes when any of these become true:
- You need more than 1 replica of any service (horizontal scaling)
- You need zero-downtime rolling deploys (Docker Compose `up -d` has a brief restart gap)
- You have funding for a managed Kubernetes cluster

### What changes in CI/CD

| Concern | Docker Compose (Phases 0–4) | Kubernetes (Phase 5+) |
|---------|-----------------------------|-----------------------|
| Deploy command | `docker compose up -d` via SSH | `kubectl set image` or Helm upgrade |
| Image tags in config | Hardcoded in docker-compose.yml | Templated in Helm `values.yaml` or Kustomize patches |
| Secret management | GitHub Secrets → `.env` on VM | GitHub Secrets → HashiCorp Vault → K8s Secrets |
| Health check after deploy | `curl /health` in SSH script | Kubernetes readiness probe + `kubectl rollout status` |
| Scaling | Manual: change `replicas:` in compose | HPA auto-scales on CPU threshold |
| Runner | GitHub-hosted or self-hosted on Oracle VM | Self-hosted on K8s node, or keep GitHub-hosted |

### Free Kubernetes options

| Option | Free tier | Limitation |
|--------|-----------|------------|
| Oracle Kubernetes Engine (OKE) | Free control plane + Always Free node pool | ARM worker nodes only; setup is complex |
| k3s on Oracle Always Free VMs | Completely free (software is free, you already have the VMs) | Single-node or 2-node; no managed control plane |
| GitHub Actions → kubectl apply | No extra cost (just workflow changes) | Requires a working cluster first |

**Recommended Phase 5 path (still free):** Install **k3s** on the two Oracle Always Free ARM VMs. k3s is a lightweight Kubernetes distribution that runs on ARM. This gives you a 2-node K8s cluster at zero additional cost using the same VMs you already have. Update the CD workflows to use `kubectl` instead of SSH + Docker Compose.

---

*This document should be updated as implementation progresses. Each section marked "Phase N" becomes active when that phase begins. Completed setup items should be checked off the checklist in Section 14.*
