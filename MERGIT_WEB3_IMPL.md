# Mergit — Web3 Implementation Guide

> A Guide to what we're building on the blockchain, why, and what it gets us.

---

## The Big Idea

AI agents do real work, but normally you have to *trust* the company that runs them. Mergit removes
that trust requirement. **Every agent's identity, work, and reputation is written to the Monad
blockchain — permanently and publicly.** Anyone can verify it. Not even we can fake or edit it.

We do this with **4 smart contracts** (programs on the blockchain), **1 oracle** (a service that
writes data to them), and **1 indexer** (a service that reads data back).

---

## Why Monad?

Monad is a blockchain that runs many transactions **at the same time** (parallel execution). Most
blockchains process one transaction at a time, like a single checkout line.

**Why it matters for us:** When 50 agents finish tasks at once, we need to record 50 proofs at once.

| | Normal blockchain | Monad |
|--|--|--|
| 50 proofs at once | Queued, takes minutes | Parallel, ~0.8 seconds |
| Speed | ~15 per second | ~10,000 per second |
| Our Solidity code | — | Runs unchanged (EVM-compatible) |

**Benefit:** Recording proof of work never slows our agents down. The blockchain keeps up with them.

```
Network : Monad Testnet          Chain ID : 10143
RPC     : https://testnet-rpc.monad.xyz
```

### Three Monad rules we must follow (they're different from other chains)

1. **Give every record its own slot.** Monad runs transactions in parallel *only if they don't touch
   the same data.* So we store proofs by `goalId`, scores by `agentId` — each in its own spot.
   ❌ A single shared counter that every transaction updates would force them back into one line and
   kill the speed advantage.

2. **You pay for the gas limit you set, not what you use.** So we measure the cost of a transaction
   once, then reuse that number — we don't over-estimate and waste money.

3. **The blockchain confirms transactions slightly after accepting them.** So "it was accepted"
   doesn't mean "it's final." Our oracle tracks each transaction until it's truly confirmed.

---

## The 4 Smart Contracts

All are written in Solidity and are **non-upgradeable** — once deployed, the code can never change.
That permanence *is* the trust guarantee.

### 1. AgentPassport — the agent's ID card

**What:** A unique token (NFT) given to each agent when it's created. It holds the agent's identity
and stats.

**The key trick — it's "soulbound" (non-transferable).** It can never be sold or moved to another
agent. We enforce this using the **ERC-5192 standard**: the token is permanently `locked`.

**Why:** If reputation could be transferred, a weak agent could just *buy* a strong agent's identity.
Soulbound means an agent's reputation must be **earned, not bought.**

```solidity
// The whole soulbound rule in one place: allow creating and destroying, block all transfers.
function _update(address to, uint256 tokenId, address auth) internal override returns (address from) {
    from = super._update(to, tokenId, auth);
    if (from != address(0) && to != address(0)) revert Soulbound();  // a real transfer → blocked
}
```

### 2. ProofOfWork — the permanent work log

**What:** For each completed goal, it stores one "fingerprint" (a Merkle root) that represents all
the tasks in that goal.

**What's a Merkle root?** Think of it as a single fingerprint for a whole stack of documents. From
that one fingerprint, anyone can prove a specific document was in the stack — without needing the
whole stack. So we store *one* small value on-chain but can still prove *any* individual task.

**Why:** Cheap (one record per goal, not per task) and tamper-proof (change any task and the
fingerprint no longer matches).

```solidity
function recordProof(bytes32 goalId, bytes32 merkleRoot, uint64 taskCount, address agent)
    external onlyRole(ORACLE_ROLE)
{
    if (proofs[goalId].timestamp != 0) revert AlreadyRecorded();   // can't record the same goal twice
    proofs[goalId] = GoalProof(merkleRoot, taskCount, uint64(block.timestamp), agent);
    emit ProofRecorded(goalId, merkleRoot, agent);
}
```

> **Important:** We build the Merkle fingerprint using OpenZeppelin's `@openzeppelin/merkle-tree`
> library — never by hand. The library guards against a known forgery trick (passing off fake data
> as real). Hand-rolling this is a security bug waiting to happen.

### 3. ReputationRegistry — the agent's score

**What:** A score from 0 to 10,000 for each agent, updated every 10 minutes.

**Safety rail:** A score can change by **at most 20% per update.** This stops anyone from suddenly
spiking or tanking a reputation.

**Why:** Reputation is the whole point of an accountability platform — it has to be stable, gradual,
and verifiable. The score is also linked to an off-chain breakdown so anyone can check the math.

```
score = (task_completion×0.35 + pr_merge_rate×0.30 + human_approval×0.25 + quality×0.10) × 10000
```

### 4. AuditTrail — the cheap activity log

**What:** Logs every agent action as a blockchain "event" (a log entry), not as stored data.

**Why:** Events cost ~50× less gas than stored data but are still permanent and searchable. Perfect
for high-frequency logging where we record a lot but rarely need to read it back on-chain.

---

## Who's Allowed to Do What (Access Control)

Two levels of permission on every contract:

| Role | Who holds it | Can do |
|------|-------------|--------|
| **Oracle** | Our backend service | Write proofs, scores, logs, mint passports |
| **Admin** | A **2-of-3 multisig** (Gnosis Safe) | Change settings, pause in emergencies |

**What's a 2-of-3 multisig?** A shared account where **any 2 of 3 people must approve** an action.
No single person — not even a founder — can act alone.

**Why:** It protects against a stolen key or a rogue insider. And we use a **timelocked handover** so
even transferring admin rights takes a built-in waiting period. **Benefit:** there's no single point
of failure for the most powerful actions.

---

## Agent Identity (DIDs) — Free and Self-Owned

**What:** Each agent gets a W3C Decentralized Identifier (DID) like `did:ethr:10143:<address>`.

**How we do it cheaply:** We use the **ERC-1056 standard (`did:ethr`)**, where an agent's blockchain
address *is* its identity. No data to store, no transaction needed — it exists the moment the agent
has a key. Identity changes (new keys, endpoints) are recorded as events.

**Why this is better than storing identity documents on-chain:**
- **Free** — creating millions of agent identities costs nothing.
- **Self-sovereign** — the identity lives on the blockchain, not in our database. If Mergit shut
  down tomorrow, every agent's identity would still exist and be verifiable.
- **Keys can expire and rotate** without losing the identity.

---

## The Oracle — Writing to the Blockchain (Rust)

**What:** A background service that watches agents finish work and writes the proofs to Monad.

**The golden rule: it never makes agents wait.** It fires off a transaction and immediately lets the
agent move on. A separate part of the service follows up to confirm the transaction landed, and
retries it (with a higher fee) if it gets stuck.

**Three things it must handle (because of how Monad works):**
- **Track its own transaction numbers (nonces) locally** — asking the blockchain each time is too
  slow and causes collisions under load.
- **Use a pool of signing keys** — so many proofs can be submitted in parallel without queuing.
- **Confirm before trusting** — "accepted" isn't "final"; it waits for real confirmation.

**Benefit:** Agents run at full speed; the blockchain writes happen reliably in the background.

---

## The Indexer — Reading from the Blockchain (Rust)

**What:** A service that reads blockchain events back out (e.g. to power the reputation leaderboard
and dashboards).

**How it stays correct:**
- It reads events in safe, confirmed batches — never the very latest, unconfirmed blocks.
- It can safely re-read the same data without creating duplicates (each event has a unique key).
- It remembers exactly where it left off, so a restart never loses or repeats data.

**Benefit:** Our app always shows accurate, up-to-date, duplicate-free data — even if the network
briefly hiccups.

---

## How Anyone Can Audit Mergit (the payoff)

This is the whole point. A judge, user, or auditor can verify our claims **without trusting us:**

```
1. Read the proof for a goal from the blockchain.
2. Get that goal's task outputs from Mergit.
3. Re-build the fingerprint themselves.
4. Compare it to the one on the blockchain.
   ✓ Match    → the work is genuine and untampered.
   ✗ No match → tampering detected.
```

**The math is the verifier. Mergit is never in the trust path.**

---

## Quick Reference: What We Use and Why

| We use | For | Benefit |
|--------|-----|---------|
| **Monad** | The blockchain | Parallel = fast enough to keep up with many agents |
| **ERC-5192 soulbound NFT** | Agent identity | Reputation is earned, not bought or sold |
| **Merkle trees** | Bundling task proofs | One cheap record proves thousands of tasks |
| **20% score cap** | Reputation updates | No sudden manipulation |
| **2-of-3 multisig** | Admin control | No single point of failure |
| **ERC-1056 DIDs** | Agent identity | Free, self-owned, survives even if Mergit shuts down |
| **Oracle + Indexer** | Writing/reading chain data | Agents never wait; data is always accurate |
| **Non-upgradeable contracts** | The whole system | The code — and the history — can never be secretly changed |

---

## For the Demo — One Paragraph

> "Most AI agents are black boxes you have to trust. Mergit puts every agent's identity, work, and
> reputation on the Monad blockchain — permanently and publicly. Each agent has a non-transferable
> identity, so reputation is earned, not bought. Every completed task is fingerprinted on-chain, so
> anyone can prove the work is genuine. We chose Monad because it runs transactions in parallel —
> when many agents finish at once, all their proofs are recorded in under a second. And because our
> contracts can never be changed and admin control needs multiple approvals, not even we can fake the
> record. Don't trust us — verify it yourself on-chain."

---

*Technical details, deployed addresses, and ABIs live in `deployments/monad-testnet.json` after launch.*
