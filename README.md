# CLU — Technical Architecture
**AI Agent Accountability Protocol**
*v0.3 · 2026 · Built on Solana · Arweave Storage · x402 Payments*

> CLU makes AI agent failure expensive — automatically, permanently, and without a central authority — by combining economic staking, automatic slashing, and on-chain reputation into a single composable protocol on Solana.

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [How CLU Fits the Landscape](#2-how-clu-fits-the-landscape)
3. [Who Can Register an Agent](#3-who-can-register-an-agent)
4. [Agent Ownership Verification](#4-agent-ownership-verification)
5. [Capability Hash and Predefined Templates](#5-capability-hash-and-predefined-templates)
6. [The Execution Receipt and output_hash](#6-the-execution-receipt-and-output_hash)
7. [x402 Payments and Agent Accountability](#7-x402-payments-and-agent-accountability)
8. [The Two-Pool System](#8-the-two-pool-system)
9. [On-Chain Programs](#9-on-chain-programs)
10. [Attestation: Two Paths](#10-attestation-two-paths)
11. [Core Protocol Flows](#11-core-protocol-flows)
12. [The Peer-Validation Consensus Layer](#12-the-peer-validation-consensus-layer)
13. [The Slashing System](#13-the-slashing-system)
14. [x402-Gated Registry Access](#14-x402-gated-registry-access)
15. [Arweave Storage Integration](#15-arweave-storage-integration)
16. [Off-Chain Components](#16-off-chain-components)
17. [Full Deployment Map](#17-full-deployment-map)
18. [Token Decision](#18-token-decision)
19. [Open Problems](#19-open-problems)
20. [Hackathon Demo Scenario](#20-hackathon-demo-scenario)
21. [Build Plan](#21-build-plan)

---

## 1. The Problem

AI agents on Solana are already managing real money — executing trades, managing liquidity pools, calling APIs, handling hundreds of thousands of dollars with minimal human oversight. x402 is enabling machine-to-machine payments at scale.

But there is no trust infrastructure. When an agent manages a $500k liquidity pool and something goes wrong — who is accountable? How does a DeFi protocol decide whether to trust an agent it has never worked with?

| Existing approach | What breaks |
|---|---|
| Centralized reputation systems | Gameable, not portable, controlled by a single party |
| API key access control | No behavioral history — a new key wipes the slate |
| Manual audits | Doesn't scale to thousands of agents executing millions of tasks |
| Staking without slashing | Sybil-friendly — bad actors spin up new identities after each failure |

CLU addresses all of these through one core idea: make failure expensive, automatically, with no one in charge of deciding.

---

## 2. How CLU Fits the Landscape

| Protocol | What it does well | What's missing |
|---|---|---|
| SAID Protocol | On-chain identity, reputation, public agent directory | No staking or slashing — reputation is attestation-only |
| SATI | ERC-8004 compliant identity, proof-of-participation | No fund pool, no peer-validation consensus |
| Solana Agent Registry (official) | Foundation-backed identity + reputation + third-party verification | No challenger mechanism, no slashing |
| Blueprint Agentic Staking | Native staking infra for AI agents, ~6% APY | Focused on yield, not accountability — no slash, no reputation history |
| CYNIC | 11-agent Proof of Judgment on Solana | Closed agent set — other operators cannot register into it |

**What CLU adds that no existing protocol has assembled:**
- Two-pool architecture separating reputation tracking from fund management
- Automatic on-chain slashing for provable failures — no humans in the loop
- `verify_agent()` — a single CPI call any Solana program can make to check an agent
- x402-gated registry API — protocol earns revenue on every lookup
- Permanent reputation history anchored to Arweave, CID stored on-chain
- Predefined capability templates — standardized, machine-verifiable agent behavior declarations

---

## 3. Who Can Register an Agent

**Any agent built anywhere can register on CLU.** CLU is accountability infrastructure, not an agent framework. It does not care how an agent was built, what language it runs in, or what platform it lives on. It only tracks the on-chain record of what the agent does.

The agent just needs a **Solana keypair**. Everything else — the code, runtime, framework — is invisible to CLU.

```
An agent built with ElizaOS, Rig, custom Rust, Python, or anything else
can register on CLU. The operator runs:

  clu register --agent-id <agent_solana_pubkey> --capability-hash <hash>
  clu stake    --agent-id <agent_solana_pubkey> --amount 5000

The agent's code does not change at all.
```

**The three parties:**

```
OPERATOR                    AGENT                       HIRING PROTOCOL
Built CLU or not            Built anywhere              Any Solana program
                            Runs anywhere
Registers agent             Executes tasks              Calls verify_agent()
Stakes USDC                 Signs task outputs          before trusting an agent
Withdraws rewards           Solana keypair is the
                            only CLU requirement
```

CLU records everything on-chain, permanently. The operator is accountable for what they register.

---

## 4. Agent Ownership Verification

### The Problem

If any Solana pubkey can be registered as an agent, what prevents someone from registering an agent they don't own? A malicious actor could register a legitimate agent's address before the real operator does, pollute its reputation record, or manufacture slash scenarios against it.

### The Fix: Dual Signature on Registration

Registration requires **two signers**: the operator and the agent itself. In Anchor, the agent account is declared as `Signer<'info>`. Solana's runtime enforces at the VM level that the agent keypair signed the transaction. This cannot be faked — the transaction is rejected unless the private key was used.

```rust
#[derive(Accounts)]
pub struct RegisterAgent<'info> {
    #[account(mut)]
    pub operator: Signer<'info>,    // pays fees, controls Fund PDA

    pub agent: Signer<'info>,       // must co-sign — cryptographic proof
                                    // that operator controls this keypair

    #[account(
        init,
        seeds = [b"agent_registry", agent.key().as_ref()],
        bump,
        payer = operator,
        space = 8 + ReputationRecord::SIZE,
    )]
    pub registry_pda: Account<'info, ReputationRecord>,

    pub system_program: Program<'info, System>,
}
```

No application-level check needed. Solana's own transaction verification handles it.

### What the Dual Signature Covers

| Scenario | Protected? | Why |
|---|---|---|
| Registering an agent you don't control | Yes | Agent must co-sign — impossible without private key |
| Registering a fake agent with a fresh keypair | No — but irrelevant | Reputation starts at 0, stake is your own money — only hurts yourself |
| Two operators registering the same agent | Structurally impossible | PDA is seeded by agent pubkey — only one Registry PDA can exist per agent |
| Operator lying about capability_hash | Partial | Hash is immutable on-chain — acting outside declared capabilities is a valid slash ground |

### Task Output Signing

Registration proves the operator controls the agent keypair. Task output signing proves the agent actually performed the work it is being attested for.

When an agent completes a task, it signs a hash of the execution receipt with its keypair. The hiring protocol includes that signature when submitting attestation.

```rust
pub fn submit_attestation(
    ctx: Context<SubmitAttestation>,
    task_id: [u8; 32],
    output_hash: [u8; 32],
    agent_signature: [u8; 64],  // agent signed sha256(task_id || output_hash)
    score: u8,
    stake_weight: u64,
) -> Result<()> {
    let agent_pubkey = ctx.accounts.registry_pda.agent_id;
    let message = sha256([task_id, output_hash].concat());
    require!(
        ed25519_verify(&agent_signature, &message, &agent_pubkey),
        CluError::InvalidAgentSignature
    );
    // proceed with reputation update
}
```

---

## 5. Capability Hash and Predefined Templates

### What the Capability Hash Is

`capability_hash = sha256(capability_manifest_json)`

The manifest is a structured JSON document declaring the agent's operational boundaries. The manifest itself is stored on Arweave at registration time. The hash is what goes on-chain — small, fixed size, immutable forever.

```
capability_hash   =   fingerprint of what the agent DECLARES it can do
                      set once at registration, immutable
                      declares allowed programs, instructions, token limits
```

### Predefined Template Library

Instead of operators writing manifests from scratch, CLU ships a library of standard templates. Operators drag templates into a builder UI, set constraint overrides, and the frontend computes the merged manifest and hash automatically.

```
┌──────────────────────────────────────────────────────────┐
│  Template Library              Agent Capability Builder   │
│                                                           │
│  ┌──────────────┐              ┌─────────────────────┐   │
│  │ DEX_TRADER   │──── drag ───▶│ DEX_TRADER_V1       │   │
│  │    V1        │              │   max_usdc: [50000]  │   │
│  └──────────────┘              │   ✎ override         │   │
│                                │                     │   │
│  ┌──────────────┐              │ ORACLE_READER_V1    │   │
│  │ORACLE_READER │──── drag ───▶│   (read-only)       │   │
│  │    V1        │              └─────────────────────┘   │
│  └──────────────┘                         │              │
│                               capability_hash: 0x8f3a... │
│                               [Register Agent]            │
└──────────────────────────────────────────────────────────┘
```

**Standard templates:**

| Template | What it unlocks |
|---|---|
| `DEX_TRADER_V1` | Raydium, Orca, Jupiter swaps |
| `LP_MANAGER_V1` | Add/remove liquidity on major Solana AMMs |
| `ORACLE_READER_V1` | Read Pyth + Switchboard price feeds (read-only) |
| `TOKEN_TRANSFER_V1` | Send/receive bounded USDC/SOL transfers |
| `YIELD_OPTIMIZER_V1` | Deposit/withdraw from yield vaults (Kamino, Drift) |
| `GOVERNANCE_VOTER_V1` | Vote on Realms governance proposals |
| `PORTFOLIO_MGMT_V1` | Rebalance across whitelisted tokens |
| `NFT_TRADER_V1` | Buy/list on Tensor, Magic Eden |
| `DATA_LABELER_V1` | Off-chain task execution, on-chain attestation only |

**What a template looks like:**

```json
{
  "template_id": "DEX_TRADER_V1",
  "version": "1.0.0",
  "description": "Executes token swaps on major Solana DEXes",
  "allowed_programs": [
    "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
    "whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3sBjvsm",
    "JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4"
  ],
  "allowed_instructions": ["swap", "route_swap", "exact_out_swap"],
  "constraints": {
    "max_single_transfer_usdc": 50000,
    "allowed_token_mints": [
      "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
      "So11111111111111111111111111111111111111112"
    ],
    "max_slippage_bps": 100
  }
}
```

### How Templates Enable Faster Auto-Adjudication

Each standard template hash is pre-registered on-chain in a `TemplateRegistry` PDA at protocol deployment. When `auto_adjudicate()` processes an `OutOfScopeCall` challenge, it checks whether the agent's `capability_hash` matches a known template:

```rust
// Inside auto_adjudicate()
if let Some(template) = template_registry.get(&agent.capability_hash) {
    // Known template — validate proof inline, no Arweave fetch needed
    verify_against_template(proof_data, template)
} else {
    // Custom manifest — require Arweave CID in proof_data
    verify_against_custom_manifest(proof_data)
}
```

Standard template agents resolve `OutOfScopeCall` slashes entirely on-chain. Custom manifest agents require an Arweave fetch in the proof. Most agents will use standard templates, so most objective slashes resolve with no off-chain dependency.

### Registration Flow With Templates

```
Operator selects templates in UI + sets constraint overrides
     │
     ▼
Frontend merges templates into single manifest JSON
     │
     ▼
Upload manifest to Arweave → get txid
     │
     ▼
capability_hash = sha256(manifest_json)  [computed locally]
     │
     ▼
register_agent(agent_pubkey, capability_hash)
← both operator + agent must sign this transaction
     │
     ▼
Registry PDA stores hash on-chain, permanently immutable
Anyone can retrieve manifest from Arweave and verify hash matches
```

If an agent's capabilities need to change — register a new agent with a new manifest. The old record stays permanently.

---

## 6. The Execution Receipt and output_hash

### The Three Concepts Compared

```
Solana tx signature    Generated by Solana runtime
                       Identifies one specific transaction on-chain
                       64 bytes, base58 encoded
                       You cannot control what this is

Ethereum tx receipt    Generated by EVM after execution
                       Contains status, gas used, logs, events
                       Auto-generated by the blockchain

CLU output_hash        Generated by the AGENT itself
                       sha256 of a JSON document the agent constructs
                       Summarizes what happened across a full task
                       Agent signs it to explicitly accept accountability
                       You control exactly what goes into it
```

**Key distinction:** Solana tx signature and Ethereum receipt are generated by the blockchain. `output_hash` is generated by the agent — it is the agent's own accountability statement, referencing the blockchain's records.

### Why output_hash Cannot Be Replaced by Solana Tx Signature

| Gap | Why tx signature doesn't cover it |
|---|---|
| Multi-transaction tasks | One task often spans 3–10 txs — which tx signature represents the task? |
| Explicit responsibility | Agent signing output_hash is a deliberate accountability statement, not a side effect of being the fee payer |
| Off-chain tasks | A data labeling agent produces no Solana transactions — output_hash is the only accountability record |
| Capability verification | The receipt's `programs_called` field maps directly to the capability manifest for OutOfScopeCall verification |

### The Execution Receipt Structure

```json
{
  "schema_version": "1.0",
  "task_id": "7f3a9c...",
  "agent_id": "AgentPubkeyBase58...",
  "operator": "OperatorPubkeyBase58...",
  "hiring_protocol": "ProtocolProgramIdBase58...",
  "timestamp_unix": 1743724800,

  "execution": {
    "tx_signatures": [
      "5Kj8xVmN3...",
      "9xQeWvG81..."
    ],
    "programs_called": [
      "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin",
      "whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3sBjvsm"
    ],
    "instructions_executed": ["swap", "get_price"],
    "token_transfers": [
      { "mint": "EPjFWdd5...", "amount": 1000000, "direction": "out" },
      { "mint": "So111111...", "amount": 5230000, "direction": "in"  }
    ]
  },

  "result": {
    "status": "success",
    "summary": "Swapped 1 USDC for 0.00523 SOL at 191.2 USDC/SOL"
  }
}
```

```
output_hash = sha256(canonical JSON above, keys sorted alphabetically)
agent_signature = sign(output_hash, agent_private_key)
```

### Why Each Field Exists

| Field | Purpose in protocol |
|---|---|
| `task_id` | Links output to the specific task assigned by hiring protocol |
| `tx_signatures` | Challenger looks these up on Solana to verify what actually happened |
| `programs_called` | Compared against `allowed_programs` in capability manifest for `OutOfScopeCall` |
| `instructions_executed` | Compared against `allowed_instructions` in manifest |
| `token_transfers` | Compared against `max_single_transfer_usdc` and `allowed_token_mints` constraints |
| `timestamp_unix` | Used for `MissedDeadline` slash — was this submitted before deadline? |

### Analogy

```
Solana tx signature    =    ATM receipt from the machine
                             proves the transaction happened
                             generated by the bank

output_hash            =    Expense report your accountant files
                             references the ATM receipts (tx_signatures)
                             adds context: what was it for, who authorized it
                             signed by the person taking accountability
```

### Independent Verifiability

Anyone can verify an output_hash claim in five steps:

```
1. Retrieve execution receipt from Arweave using arweave_cid
2. sha256(receipt_json) == output_hash on Registry PDA?   ← receipt is authentic
3. ed25519_verify(agent_signature, output_hash, agent_pubkey)?  ← agent signed it
4. Look up tx_signatures on Solana RPC  ← did these transactions actually happen?
5. programs_called in receipt match actual tx instructions?  ← agent reported honestly
```

A challenger walks all five steps to build a slash proof. A hiring protocol walks steps 1–4 to trust an agent's work. The Solana tx signatures are ground truth — output_hash is the structured reference to them.

---

## 7. x402 Payments and Agent Accountability

### The Interaction

In x402-enabled agentic flows, agents have their own autonomous wallets and sign their own payment transactions. This changes the accountability picture compared to traditional setups:

```
Traditional setup:
  Operator keypair signs task execution transactions
  → tx signature proves operator, not agent
  → agent accountability is indirect

x402 setup:
  Agent has its own autonomous wallet
  Agent signs its own payment transactions
  Agent IS the fee payer for services it consumes
  → tx signature directly proves agent identity for payment actions
```

When an agent makes x402 payments, it necessarily has its own keypair and signs its own transactions. So in x402 context, the tx signature does carry agent accountability — for payment transactions specifically.

### What x402 Proves vs What output_hash Proves

```
x402 payment tx signature   →   proves agent identity and that it
                                consumed a specific service at a point in time

output_hash + agent_sig     →   proves agent identity AND that it explicitly
                                accepts responsibility for a specific task outcome
```

x402 proves **"this agent paid for X"** — not **"this agent produced output Y and takes responsibility for it."** Both are needed for the full accountability picture.

### The Positive Synergy

x402 and CLU strengthen each other:

```
x402 payment history    →   proves agent was autonomously active,
                            consuming services, operating independently

CLU output_hash         →   proves agent accepts responsibility for
                            specific task outcomes

Combined                →   complete accountability picture:
                            what the agent consumed + what it produced
                            + explicit responsibility acceptance
```

**CLU's auto-attestation watcher uses x402 payment events as supporting evidence.** If an agent paid for a Pyth price feed at timestamp T and executed a swap at timestamp T+2, the payment proves the agent had the data it claimed to have used. This strengthens slash defense and attestation quality.

### Three-Signature Accountability Model

In a fully x402-enabled flow:

```
x402 payment tx signed by agent keypair      ← proves agent identity and autonomy
task execution tx signed by agent keypair    ← proves agent executed the work
output_hash signed by agent keypair          ← proves agent accepts responsibility

Three independent signatures from the same keypair = strong proof that one
autonomous agent was responsible for the full task lifecycle
```

This is a stronger accountability picture than any competing protocol provides — CLU + x402 create a chain of evidence from payment through execution through outcome, all signed by the same agent keypair.

---

## 8. The Two-Pool System

Every other agent registry conflates identity, reputation, and funds into a single account. CLU separates them into two distinct Solana PDAs with different access patterns and lifecycles.

|  | Agent Registry Pool | Agentic Fund Pool |
|---|---|---|
| Maps | `agent_id → ReputationRecord` | `(operator, agent_id) → FundAccount` |
| Type | One PDA per agent | One PDA per (operator, agent) pair |
| Access pattern | Read-heavy — queried constantly by external protocols | Write-heavy — touched only during stake, slash, rewards |
| Fields | `reputation_score`, `capability_hash`, `slash_count`, `arweave_cid` | `locked_stake`, `validator_stake`, `claimable_rewards`, `challenger_bond_pending` |
| Who writes | Only CLU program after attestation or slash | Operator (deposit/withdraw), CLU program (slash/rewards) |

**Why split them?** Reputation reads and fund writes have different access patterns. Separating them eliminates contention. Multiple stakers can also back a single agent without controlling its identity.

---

## 9. On-Chain Programs

CLU deploys three Anchor programs on Solana. All token operations use USDC (SPL token).

### Program 1: `clu_registry`

**Registry PDA** — one per agent

```rust
// seeds: ["agent_registry", agent.key()]
pub struct ReputationRecord {
    pub agent_id:           Pubkey,    // agent wallet
    pub operator:           Pubkey,    // who registered it
    pub capability_hash:    [u8; 32],  // SHA256 of capability manifest — immutable
    pub reputation_score:   u64,       // 0–10000 (represents 0–100.00%)
    pub slash_count:        u8,        // permanent — never decrements
    pub total_attestations: u64,
    pub total_tasks:        u64,
    pub arweave_cid:        String,    // max 100 chars — Arweave txid
    pub registered_at:      i64,
    pub last_updated:       i64,
    pub bump:               u8,
}
```

**Instructions:**

`register_agent(capability_hash)`
- Requires both `operator` and `agent` as `Signer` — dual signature
- Creates Registry PDA + Fund PDA
- `reputation_score = 0` — cannot buy reputation
- `capability_hash` immutable after this
- Emits: `AgentRegistered { agent_id, operator, timestamp }`

`submit_attestation(task_id, output_hash, agent_signature, score, stake_weight)`
- Verifies `ed25519_verify(agent_signature, sha256(task_id || output_hash), agent_pubkey)`
- Reputation contribution = `(stake_weight * score) / max_stake`
- Prevents reputation farming — points scale with stake at risk
- Emits: `AttestationSubmitted { agent_id, score, new_reputation }`

`verify_agent(agent_id, min_reputation, min_stake) → (bool, ReputationRecord)`
- CPI-callable by any external Solana program
- Returns `true` if `reputation_score >= min_reputation AND locked_stake >= min_stake`
- This is the composability primitive — the most important instruction in the protocol

`write_arweave_cid(agent_id, cid)`
- Authority: winning validator (after validation round) or CLU program (after slash)
- Updates `arweave_cid` in Registry PDA

---

### Program 2: `clu_fund`

**Fund PDA** — one per (operator, agent) pair

```rust
// seeds: ["fund_account", operator.key(), agent_id.key()]
pub struct FundAccount {
    pub operator:                Pubkey,
    pub agent_id:                Pubkey,
    pub locked_stake:            u64,   // USDC (6 decimals) — what gets slashed
    pub validator_stake:         u64,   // locked when participating as validator
    pub claimable_rewards:       u64,
    pub challenger_bond_pending: u64,
    pub stake_token_mint:        Pubkey, // USDC mint
    pub bump:                    u8,
}
// Token vault PDA: ["vault", fund_account.key()]
```

**Instructions:**

`stake(amount)` — transfers USDC from operator → vault PDA

`withdraw_stake(amount)` — only if no active challenges; transfers vault → operator

`claim_rewards()` — transfers `claimable_rewards` → operator

---

### Program 3: `clu_adjudication`

**Challenge PDA**

```rust
// seeds: ["challenge", agent_id.key(), nonce.to_le_bytes()]
pub enum FailureType {
    MissedDeadline,       // L2: provable on-chain — auto-adjudicated
    OutOfScopeCall,       // L2: provable on-chain — auto-adjudicated
    SubstantiveFailure,   // L3: requires peer validation round
}

pub struct Challenge {
    pub agent_id:     Pubkey,
    pub challenger:   Pubkey,
    pub bond_amount:  u64,
    pub failure_type: FailureType,
    pub proof_data:   Vec<u8>,    // max 1024 bytes
    pub status:       ChallengeStatus,
    pub filed_at:     i64,
    pub nonce:        u64,
    pub bump:         u8,
}
```

**Validation Round PDA**

```rust
// seeds: ["validation_round", challenge.key()]
pub struct ValidatorRecord {
    pub validator:  Pubkey,
    pub commitment: [u8; 32],    // hash(verdict || confidence || salt)
    pub verdict:    Option<bool>,
    pub confidence: Option<u8>,  // 0–100
}

pub struct ValidationRound {
    pub challenge_id:      Pubkey,
    pub task_output_cid:   String,
    pub validators:        Vec<ValidatorRecord>, // max 21
    pub commit_deadline:   i64,
    pub reveal_deadline:   i64,
    pub status:            RoundStatus,
    pub winning_validator: Option<Pubkey>,
    pub bump:              u8,
}
```

**Instructions:**

`file_challenge(agent_id, failure_type, proof_data, bond_amount)`
- Creates Challenge PDA, locks challenger bond in escrow
- Minimum bond enforced by program constant — filters frivolous claims

`auto_adjudicate(challenge_id)`
- Only for `MissedDeadline` / `OutOfScopeCall`
- Checks agent's `capability_hash` against `TemplateRegistry` PDA first
- If known template → validates proof inline, no Arweave fetch
- If custom manifest → requires Arweave CID in proof_data
- Valid proof → `execute_slash()`; invalid → dismiss + return bond

`execute_slash(agent_id, challenge_id)`
- 60% of `locked_stake` → challenger wallet
- 40% → CLU treasury PDA
- `reputation_score = reputation_score * 35 / 100`
- `slash_count += 1` (permanent)
- Emits: `AgentSlashed { agent_id, slash_count, new_reputation }`

`open_validation_round(challenge_id, task_output_cid)`
- Creates ValidationRound PDA
- `commit_deadline = now + 86400` (24h)
- `reveal_deadline = commit_deadline + 43200` (12h)

`commit_verdict(round_id, commitment: [u8; 32])`
- Validator posts `sha256(verdict || confidence || salt)`
- Must have minimum `validator_stake` and `reputation_score`

`reveal_verdict(round_id, verdict: bool, confidence: u8, salt: [u8; 32])`
- Verifies commitment, stores actual verdict

`finalize_round(round_id)`
- Counts majority verdict
- Winning validator = majority side member with highest confidence
- Minority validators lose `validator_stake`
- Grants winning validator write access to update Arweave CID

---

## 10. Attestation: Two Paths

Requiring every hiring protocol to integrate `submit_attestation()` is high friction — extra dev work, transaction fees, responsibility for judging output quality. CLU solves this with two parallel attestation paths.

### Path 1 — Auto-Attestation (Primary Path)

CLU runs an off-chain watcher that monitors Solana transactions. When it detects an agent completing a verifiable action, it generates an attestation automatically. No hiring protocol integration required.

```
Agent executes a Jupiter swap

CLU watcher detects on-chain:
  - agent_pubkey signed the transaction
  - programs_called ⊆ allowed_programs in capability manifest
  - transaction succeeded
  - output_hash signed by agent keypair

CLU watcher calls submit_attestation():
  - score: derived from objective execution quality
  - stake_weight: agent's current locked_stake
  - no hiring protocol involvement
```

**Auto-score derivation:**

| Outcome | Score |
|---|---|
| Task completed, all programs within allowlist | 85 |
| Completed, within slippage tolerance | 90 |
| Completed, exceeded slippage but within manifest limits | 65 |
| Completed before deadline | +10 bonus |
| Hiring protocol submitted additional quality score | Weighted average |

### Path 2 — Protocol Attestation (Quality Layer)

For subjective outcomes that cannot be derived from transaction data alone, the hiring protocol submits a quality score. This is optional and layers on top of auto-attestation.

The hiring protocol does not need to modify their on-chain program. They call a CLU API endpoint from their backend:

```typescript
// Hiring protocol backend — no Solana program changes needed
await clu.attest({
  agentId: "AgentPubkey...",
  taskId: "task_123",
  outputHash: "7f3a9c...",
  agentSignature: "base64sig...",
  score: 85,
  stakeWeight: 5000,
  signerKeypair: hiringProtocolKeypair  // they sign, CLU submits on-chain
});
```

CLU validates their signature and submits the on-chain transaction. CLU covers the transaction fee, recovering it via x402 registry access revenue.

### The verify_agent() Flywheel

Once `verify_agent()` becomes a standard that DeFi protocols call before trusting agents with their pools, hiring protocols gain a direct incentive to submit attestations. Better-attested agents build higher reputation and keep getting work. Attestation becomes self-interested — no evangelism needed.

---

## 11. Core Protocol Flows

### Registration

```
1. Operator constructs registration transaction
2. Both operator keypair AND agent keypair sign it      ← dual signature
3. Solana runtime rejects if either signature missing
4. Registry PDA created — identity immutable from this point
5. Fund PDA created — operator can now stake
6. AgentRegistered event emitted
```

### Normal Operation (Reputation Building)

```
Agent executes task across 1–N Solana transactions
     │
     ▼
Agent constructs execution receipt JSON
output_hash = sha256(receipt_json, sorted keys)
agent_signature = sign(output_hash, agent_private_key)
     │
     ▼
CLU auto-attestation watcher detects completed task on-chain
calls submit_attestation(agent_id, task_id, output_hash, agent_signature, score, stake_weight)
     │
     ▼
Program verifies ed25519 signature on-chain
reputation_score += (stake_weight * score) / max_stake
     │
     ▼
AttestationSubmitted event emitted
Receipt stored on Arweave, CID anchored on-chain
```

### Slash Flow (L2 — Automatic)

```
Agent misses deadline or makes out-of-scope call
     │
     ▼
Challenger posts bond + proof → file_challenge()
     │
     ▼
auto_adjudicate():
  checks TemplateRegistry for known template hash
  validates proof inline (no Arweave fetch for standard templates)
     │
     ├── Invalid proof → dismiss, return bond
     │
     └── Valid proof → execute_slash()
              ├── 60% locked_stake → challenger
              ├── 40% → treasury
              ├── reputation_score × 0.35
              └── slash_count += 1 (permanent)
                       │
                       ▼
              AgentSlashed event emitted
              Off-chain listener uploads snapshot to Arweave
              write_arweave_cid() anchors CID on-chain
```

### Composability (External Protocol Checking an Agent)

```rust
// Inside any Solana program — one CPI call
let (trusted, record) = cpi::verify_agent(
    agent_id,
    min_reputation: 5000,
    min_stake: 10_000_000,  // 10 USDC
)?;

if trusted {
    // grant pool access, execute trade, etc.
} else {
    return Err(ProtocolError::AgentNotTrusted);
}
```

---

## 12. The Peer-Validation Consensus Layer

For subjective failures that cannot be proven automatically on-chain, CLU uses peer validation: competing registered agents stake their own USDC on their verdict and compete for the right to write the reputation update.

**Economic equilibrium:**
- Voting with the losing minority costs `validator_stake`
- Voting with the winning majority earns nothing but avoids a loss
- Only the winning validator with the highest confidence earns a positive reward

The rational strategy is always honest evaluation. Collusion doesn't pay — honest validators outvote colluders, and colluders lose their stake.

**Sybil resistance:** A validator must have a registered CLU identity with minimum `validator_stake` and `reputation_score`. Cheap new accounts cannot flood a validation round.

**Hackathon scope:** Simplified verdict submission. Full commit-reveal if ahead of schedule. The architecture matters for judges, not the cryptographic detail.

---

## 13. The Slashing System

| Failure type | Examples | How adjudicated |
|---|---|---|
| Objective / provable | Missed deadline, out-of-scope program call, transfer over limit | L2: Auto on-chain — no humans |
| Subjective / qualitative | Output delivered but wrong, adversarial behavior | L3: Staked peer validation round |
| Genuinely disputed | L3 verdict contested | L4: DAO governance — not in hackathon scope |

**Slash economics:**

| Recipient | Share | Rationale |
|---|---|---|
| Challenger | 60% of `locked_stake` | Reward accurate claims, fund enforcement ecosystem |
| CLU treasury | 40% of `locked_stake` | Fund panelist rewards, DAO operations, protocol development |
| Challenger bond | Returned in full | Bond was only to filter frivolous claims |

**Reputation impact:** `reputation_score = reputation_score * 35 / 100` per slash.
- One slash: 100% → 35%
- Two slashes: 100% → 35% → ~12%

`slash_count` is permanent. It can never be decremented — not by the operator, not by CLU, not by anyone.

---

## 14. x402-Gated Registry Access

External protocols querying CLU's registry pay via x402 — the HTTP-native micropayment protocol settling on Solana. A free registry has no revenue model, no spam filter, and no signal on data value.

**Endpoints and pricing:**

| Endpoint | Price |
|---|---|
| `GET /agents/:id` — single agent lookup | 0.001 USDC |
| `GET /agents/batch` — up to 100 agents | 0.05 USDC |
| `GET /agents/:id/history` — full Arweave history | 0.01 USDC/minute |

**Payment flow:**
1. Client sends request with no payment header
2. Server returns `402` + `{ amount: "0.001", token: "USDC", network: "solana-devnet" }`
3. Client sends USDC tx on Solana, retries with `X-Payment` header
4. Server verifies tx on-chain → serves data. End-to-end under 500ms.

**What is always free:** Raw Solana PDA data is public. Anyone running their own RPC node can read it for free. The x402 gate applies only to the indexed, structured API layer.

---

## 15. Arweave Storage Integration

Full reputation history is too large and too permanent for Solana accounts. CLU stores the Arweave transaction ID on-chain — the chain stores the pointer, Arweave stores the data permanently.

**Trigger events:** `AgentSlashed` or `RoundFinalized`

**Write flow:**

```
Off-chain listener catches event
     ▼
Serialize full ReputationRecord + event history to JSON
     ▼
Upload to Arweave via arweave-js → get txid
     ▼
Call write_arweave_cid(agent_id, txid) on-chain
     ▼
Registry PDA now contains pointer to permanent history
```

**What lives where:**

| Data | Storage | Why |
|---|---|---|
| `reputation_score`, `slash_count`, `arweave_cid` | Solana on-chain | Must be composable — `verify_agent()` reads it via CPI |
| `locked_stake`, balances | Solana on-chain (Fund PDA) | All token transfers must be atomic and trustless |
| Full reputation history + validation verdicts | Arweave | Too large for Solana accounts. Permanent, cannot be deleted |
| Link between chain and storage | `arweave_cid` in Registry PDA | Anyone can reconstruct full agent audit history from on-chain data |

**Note on Codex/Logos:** Codex paused its testnet in August 2025. Arweave is the storage layer for the hackathon. The `arweave_cid` field is storage-layer-agnostic — switching to Codex in production requires only swapping the storage SDK, not any on-chain program changes.

---

## 16. Off-Chain Components

### x402 API Server (Node.js / TypeScript)

```
GET  /agents/:id          → single ReputationRecord   (0.001 USDC)
GET  /agents/batch        → up to 100 records         (0.05 USDC)
GET  /agents/:id/history  → full Arweave history      (0.01 USDC/min)
```

### Auto-Attestation Watcher (TypeScript)

Monitors all three CLU programs via Solana websocket. Detects completed agent task transactions. Derives objective attestation scores. Calls `submit_attestation()` automatically. No hiring protocol action required.

### Indexer (TypeScript)

Parses emitted events from all three programs. Writes to SQLite (hackathon) / PostgreSQL (production). Powers x402 API with fast indexed reads.

### CLI (TypeScript)

```bash
clu register   --agent-id <pubkey> --capability-hash <hex>
clu stake      --agent-id <pubkey> --amount <usdc>
clu attest     --agent-id <pubkey> --task-id <id> --score <0-100> --output-hash <hex> --agent-sig <base64>
clu verify     --agent-id <pubkey> --min-rep <score> --min-stake <usdc>
clu challenge  --agent-id <pubkey> --type missed-deadline --proof <base64>
clu history    --agent-id <pubkey>
```

### Frontend (Next.js)

| Page | What it shows |
|---|---|
| Operator Dashboard | Register agent (template drag-and-drop), stake/unstake USDC, reputation score + history chart |
| Agent Explorer | Browse all agents, filter by min reputation/stake, link to Arweave history |
| Challenge Interface | File challenge, upload proof, post bond, view dispute status |
| Validator Interface | Open validation rounds, commit/reveal UI, personal stake + rewards |

### Tech Stack

| Layer | Technology |
|---|---|
| On-chain programs | Rust + Anchor |
| Program client | `@coral-xyz/anchor` (TypeScript) |
| Wallet integration | `@solana/wallet-adapter-react` |
| Token transfers | `@solana/spl-token` (USDC) |
| x402 payments | `x402` npm package |
| Off-chain storage | Arweave via `arweave-js` |
| API server | Node.js + Express |
| Database | SQLite → PostgreSQL |
| Frontend | Next.js |
| Network | Solana devnet |

---

## 17. Full Deployment Map

| Component | Deployment | Reason |
|---|---|---|
| Agent Registry Pool — `agentId → ReputationRecord` PDA | Solana on-chain | Must be composable — external programs call `verify_agent()` via CPI |
| Agentic Fund Pool — `(operator, agentId) → FundAccount` PDA | Solana on-chain | All token transfers must be atomic and trustless |
| Template Registry — standard capability template hashes | Solana on-chain | Enables inline OutOfScopeCall validation without Arweave fetch |
| Adjudication Engine — slash + validation logic | Solana on-chain | Objective slash proofs verified by program with no human in the loop |
| Reputation History — snapshots + validation records | Arweave | Too large for Solana accounts. Permanent. CIDs anchored on-chain |
| Registry API — indexed, structured interface | Off-chain server + x402 gate | Raw PDA data is public but unstructured. API indexes and gates access |
| Auto-Attestation Watcher | Off-chain service | Monitors on-chain agent activity, generates objective attestations automatically |
| Agent execution | Operator's own infrastructure | CLU only cares about the on-chain record of agent outputs |

---

## 18. Token Decision

**There is no $CLU token in the hackathon build.**

All stakes, bonds, slashes, and rewards flow in USDC. $CLU becomes a post-launch governance and reward token — distributed via governance vote after mainnet. This removes an entire complexity layer. No judge will penalize this decision.

---

## 19. Open Problems

**Subjective failure adjudication**
L3 panel selects experts by reputation score — but a top DeFi agent isn't qualified to judge a data labeling task. Long-term fix: category-specific panels based on task history.

**Validator collusion risk**
A coordinated group could drain honest minority validators over time. Mitigation: reputation-weighted consensus makes collusion expensive, entry barriers limit new colluder accounts.

**Cold start**
A registry with no agents has no network effects. Initial deployment needs seeded agents and at least one live integrating protocol from day one.

**Micro-task reputation farming**
Flooding the registry with trivial successful tasks before attempting high-value work. Mitigation: stake-weighted task scoring — reputation points scale with stake at risk.

**Compute limits for large validation rounds**
Current design: maximum 21 validators per round. Larger rounds require off-chain computation with on-chain proof submission.

**Auto-attestation watcher trust**
The watcher is off-chain and controlled by CLU team. Long-term fix: decentralised watcher network where external parties run watchers and earn bounties for submitting accurate attestations.

---

## 20. Hackathon Demo Scenario

Every component serves this single demo. If this runs cleanly end-to-end, the submission wins.

```
Step 1 — Agent A registers
  Operator selects DEX_TRADER_V1 template in UI
  capability_hash = sha256(DEX_TRADER_V1_manifest)
  Both operator + agent sign registration transaction
  clu stake --agent-id <A_pubkey> --amount 5000
  ✓ Registry PDA created, Fund PDA created, 5000 USDC locked

Step 2 — Agent A completes tasks, builds reputation
  Agent executes 3 swaps, signs each output_hash
  CLU auto-attestation watcher submits attestations automatically
  ✓ reputation_score rises to 7200

Step 3 — Agent B registers and fails
  clu register --agent-id <B_pubkey> --capability-hash <hash>
  clu stake    --agent-id <B_pubkey> --amount 2000
  Agent B misses a deadline
  ✓ Registry PDA + Fund PDA created

Step 4 — Challenger files claim
  clu challenge --agent-id <B_pubkey> --type missed-deadline --proof <base64>
  ✓ Challenge PDA created, bond locked

Step 5 — Auto-adjudication fires
  auto_adjudicate() verifies proof against known template inline — no Arweave fetch
  execute_slash() runs:
    → 1,200 USDC (60%) to challenger
    → 800 USDC (40%) to treasury
    → reputation_score: 5000 → 1750
    → slash_count: 0 → 1 (permanent)
  ✓ AgentSlashed event emitted

Step 6 — Arweave snapshot written
  Off-chain listener catches AgentSlashed
  Serializes ReputationRecord to JSON, uploads to Arweave
  write_arweave_cid() anchors CID on Registry PDA
  ✓ Permanent history record created, cannot be deleted by anyone

Step 7 — External DeFi program calls verify_agent()
  verify_agent(A_pubkey, min_reputation: 5000, min_stake: 3000) → TRUE  ✓
  verify_agent(B_pubkey, min_reputation: 5000, min_stake: 3000) → FALSE ✗

  DeFi protocol grants Agent A pool access.
  DeFi protocol denies Agent B.
  No central authority made this decision. The on-chain record did.
```

---

## 21. Build Plan

### Pre-hackathon (April 3–5)
- Anchor workspace initialized, repo structured, devnet wallets funded
- Program IDs reserved on devnet
- Standard template manifests written and uploaded to Arweave
- TemplateRegistry PDA seeded with standard template hashes
- Demo script written (exact CLI commands for steps 1–7 above)
- Dependencies pinned: Anchor, SPL token, x402, arweave-js

### Week 1 (April 6–12) — Core Identity & Composability
- `clu_registry`: `register_agent` (dual signature), `submit_attestation` (ed25519 verify), `verify_agent`
- `clu_fund`: `stake`, `withdraw_stake`
- `TemplateRegistry` PDA with standard template hashes
- **Dummy DeFi program calling `verify_agent()` via CPI** — build this week
- Goal: steps 1–2 of demo scenario run cleanly

### Week 2 (April 13–19) — Slashing & Payments
- `clu_adjudication`: `file_challenge`, `auto_adjudicate` (template-aware), `execute_slash`
- x402 API server: single agent lookup with payment gate
- Arweave write triggered by slash event, CID anchored on-chain
- Auto-attestation watcher v1
- Goal: steps 3–6 of demo scenario run cleanly

### Week 3 (April 20–26) — Validation Round & Frontend
- Validation round (simplified verdict submission)
- Indexer subscribing to all three programs
- Batch API endpoint
- Frontend: Operator Dashboard with template drag-and-drop

### Week 4 (April 27 – May 3) — Polish & Integration
- Full demo scenario runs without manual intervention
- CLI complete and tested
- Frontend: Agent Explorer page
- All edge cases handled

### Week 5 (May 4–11) — Submission
- 3-minute pitch video
- Written submission materials
- Protocol outreach — 3–5 Solana DeFi teams
- Buffer for bugs — no new features this week

### Team Allocation

| Role | Owns |
|---|---|
| Rust dev 1 | `clu_registry` + `clu_fund` + `TemplateRegistry` programs |
| Rust dev 2 | `clu_adjudication` + dummy CPI demo program |
| TS dev | x402 API + indexer + Arweave + auto-attestation watcher |
| Frontend dev | Next.js dashboard with template builder (starts week 3) |
| Product | Demo script, user flows, documentation, submission materials |
| BD | Protocol outreach, Codex/Logos contact |

---

*CLU Technical Architecture · v0.3 · April 2026 · Hackathon build — Colosseum Frontier · All tokenomic parameters subject to governance finalization before mainnet.*
