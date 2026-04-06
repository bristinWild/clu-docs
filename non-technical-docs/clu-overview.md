# CLU — The Trust Layer for AI Agents on Solana

### For the team — no technical background needed

---

## The 30-second version

AI agents are already managing millions of dollars on Solana — executing trades, managing liquidity pools, moving funds — with no human watching. When something goes wrong, nobody is accountable.

CLU fixes that. We make AI agent failure **expensive, automatic, and permanent** — with no company or committee deciding what counts as a failure. The blockchain decides.

Think of CLU as a **credit score for AI agents** — except it cannot be faked, cannot be deleted, and automatically punishes bad behavior without anyone pressing a button.

---

## Why this matters right now

There is a new standard emerging in the Solana ecosystem called **Agent Skills**. These are pre-built instruction packs that developers drop into their AI coding assistant (like Claude Code) to teach it how to use Solana protocols.

Jupiter has a skill. Kamino has a skill. Raydium, Orca, Meteora — all have skills. They all teach an AI agent *how* to do things on Solana.

**Not one of them answers: should I trust the agent doing it?**

That gap is our entire market. CLU is the trust layer those skills are missing. We are not competing with Jupiter or Kamino. We sit underneath them.

---

## The problem in plain language

Imagine you run a DeFi protocol and you want to let AI agents trade on your platform. How do you know which agents to trust?

Right now you cannot know. There is no track record. There is no accountability. A brand new agent with zero history looks identical to a battle-tested agent that has executed a thousand trades cleanly. And if an agent loses your users' money, nothing happens to it. It just keeps operating.

**This is exactly the problem CLU solves.**

---

## How CLU works — in plain language

### Step 1 — The agent gets registered

When a developer builds an AI trading agent, they register it with CLU. Registration requires a deposit (we call it "staking") — real money locked on the blockchain. The bigger the stake, the more the operator believes in their agent.

This stake is what makes bad behavior expensive. You cannot fake a stake.

### Step 2 — The agent builds a reputation

Every time the agent successfully completes a task — executes a trade, manages a position, processes a transaction — that gets recorded permanently on the blockchain. A reputation score builds up over time. This score is public, permanent, and cannot be edited by anyone, including us.

### Step 3 — DeFi protocols check the score before trusting the agent

Before a DeFi protocol lets an agent access a liquidity pool or execute a trade, it can check CLU's trust score with a single lookup. If the agent has a high score and meaningful stake behind it, access is granted. If not, the door stays closed.

### Step 4 — Bad behavior gets punished automatically

If an agent misses a deadline, calls a function it was not supposed to call, or exceeds its authorized transaction limits, anyone can file a challenge. CLU checks the evidence automatically — no committee, no vote, no human judgment. If the evidence is valid, the agent gets slashed: a chunk of its stake is destroyed and its reputation score craters. Permanently.

The person who filed the challenge gets 60% of the slashed stake as a reward. This creates an ecosystem of watchdogs who are financially motivated to catch bad agents.

### Step 5 — Community members earn by backing good agents

Anyone — not just the developer — can add their own stake to back an agent they trust. If that agent performs well, they earn proportional rewards every seven days. If the agent gets slashed, they lose proportionally too. This creates genuine skin in the game.

---

## Who uses CLU and why

### Developers building AI agents

They want their agents to be trusted with real money. Without CLU, even a perfectly reliable agent looks indistinguishable from an unproven one. With CLU, their track record is public, permanent, and backed by real economic stake. CLU gives their agents a competitive advantage just by being registered.

**How they interact:** They install the CLU skill into their AI coding tool (`npx skills add clu-protocol/skills`), and their coding assistant walks them through the entire registration process in plain language — no forms, no separate dashboard, just a conversation.

### Individual traders hiring agents

They want to use AI agents to manage their DeFi portfolio but they have no way to evaluate which agents are actually good. CLU gives them real, verifiable data — not marketing copy.

**How they interact:** They browse the CLU agent explorer, see each agent's reputation score, how much money is staked behind it, and its full task history going back to day one. They hire an agent, set a task, and if anything goes wrong they can challenge the outcome with one button.

**How does the agent pay for its own transactions?**

This is a question that comes up often, so it's worth making explicit. When a trader hires an agent and registers a task, part of what they pay automatically goes into the agent's wallet to cover its transaction fees on Solana. The breakdown looks like this:

```
Trader pays subscription fee
        │
        ├── Gas allocation  →  tops up the agent's wallet (covers tx fees)
        ├── Fund pool share →  distributed to stakers and validators
        └── CLU platform fee → protocol treasury
```

The agent never needs to be manually funded. Every task registration automatically tops up its wallet before the task even starts. CLU also checks the agent's balance before queuing any task — if the wallet is too low, the task waits until it's sufficiently funded.

The trader funds the agent's operations without ever knowing or caring about it. It just works.

### Community stakers

They want to earn yield on their USDC without running an agent themselves. They find an agent they trust, add their stake to it, and earn proportional rewards every seven days. The more they stake and the better the agent performs, the more they earn.

**How they interact:** Browse, stake, wait, claim. No technical knowledge required.

### DeFi protocols

They want to let AI agents trade on their platform but they have no way to screen out bad actors. CLU gives them one line of code that checks whether an agent is registered, staked, and reputable. Access granted or denied — automatically, on-chain, with no ongoing management.

**How they interact:** A developer adds a single check to their existing code. That is the entire integration. No account needed, no fee to CLU, no dashboard to manage.

---

## What makes CLU different from everything else

**There are other reputation systems.** They are all run by a company. The company decides who gets a good score. The company can be bribed, pressured, or compromised. Their data can be deleted.

**CLU has no one in charge.** The reputation is on the blockchain. The slashing happens automatically. The evidence is stored permanently on Arweave (a decentralized permanent storage network). Nobody — including us — can modify an agent's score, reverse a slash, or delete the history.

That is not a feature. That is the entire product. An accountability system that can be overridden by the people running it is not an accountability system.

---

## The Skills angle — why this is a big deal

`solana.com/skills` is a directory where developers discover and install Agent Skills. Every DeFi protocol in the Solana ecosystem is listed there, and the developer community is growing fast around it.

We want CLU listed there as the first — and only — **trust and accountability** skill. Right now the directory has DeFi skills, infrastructure skills, tooling skills. There is no trust category. We create it.

The distribution story is this: any developer building a Solana AI agent with any of those DeFi skills is a potential CLU user. When they install the CLU skill alongside Jupiter or Kamino, their coding assistant automatically injects accountability into every agent they build. They do not have to think about it. It just happens.

This is how we grow without a sales team.

---

## The UX — what each person actually sees

### What the developer sees (Claude Code terminal)

```
Developer types: "Set up CLU for my trading agent"

Claude Code responds:
  "I'll register your agent with CLU. A few quick questions:

   1. What is your agent's wallet address?
   2. I see you have Jupiter and Kamino skills installed —
      I'll suggest a trading + lending capability template.
      Does that match what this agent does?
   3. How much USDC do you want to stake? Minimum is 100 USDC.

   Ready? I'll generate the registration transaction now."

→ Two signatures later, the agent is registered and live.
   No website visit. No form. No separate dashboard.
   The entire onboarding happens inside the developer's
   existing coding environment.
```

### What the trader sees (web app)

**Agent Explorer page**

A list of registered agents. Each card shows:
- Reputation score (0–100)
- Total USDC staked behind it
- Number of tasks completed successfully
- Number of times it has been slashed (if any — permanent and visible)
- Capability type (trading agent, lending agent, etc.)

The trader filters by reputation and stake, clicks an agent, reads its history, and decides to hire it.

**Task Panel page**

After hiring, the trader sees a queue of their tasks with live status updates:
- `Pending` — in the queue, not yet started
- `Executing` — agent is working on it
- `Completed` — done, verified, receipt stored permanently
- `Challenged` — someone filed a challenge, under review
- `Resolved: Slashed` — agent failed, stake taken, trader gets compensation
- `Resolved: Dismissed` — challenge found to be invalid

At the bottom of each completed task card there is a single **Challenge** button. That is all. One click, confirm the bond amount, done. CLU handles everything else automatically.

**Staker page**

A simplified version of the agent explorer. Find an agent, stake USDC, see your current earnings, claim every seven days. No technical knowledge required.

### What the DeFi protocol sees

They never visit CLU's interface at all. Their developer adds one check to their existing code. Their users — the agents trading on their platform — either pass the check or they do not. The protocol never has to think about CLU again unless they want to adjust their minimum thresholds.

---

## The numbers that matter

**For an agent that gets slashed:**
- Loses 65% of their reputation score in one event
- Gets slashed again: drops to about 12% of original score
- Three slashes: effectively banned from any serious DeFi protocol
- The slash record is permanent. It cannot be removed.

**For a challenger who catches a bad agent:**
- Posts a small bond (e.g. 10 USDC)
- If the challenge is valid: gets back their bond + 60% of the agent's slashed stake
- Catching a 1,000 USDC staked agent pays out 600 USDC to the challenger
- This creates professional "agent watchdogs" who monitor agents for profit

**For a community staker:**
- Proportional rewards every 7 days
- Example: agent has 100 USDC total stake, you staked 30 USDC → you earn 30% of all rewards
- If agent gets slashed, you lose 30% of the slashed amount

---

## What we are building first (hackathon scope)

We are not building everything at once. The hackathon focuses on the parts that make the core story work:

**We are building:**
- Agent registration and staking (the economic foundation)
- Automatic reputation scoring (the track record)
- Automatic slashing for provable failures (the accountability)
- Community staking and proportional rewards (the flywheel)
- The CLU skills for Claude Code (the distribution)
- A dummy DeFi protocol that calls our trust check — so judges can see the whole story end to end

**We are describing but not building yet:**
- Peer review panels for subjective failures (when an agent delivers something wrong but not provably wrong)
- Governance for protocol parameters
- Decentralized watchdog network

---

## The demo — what judges will see

The hackathon demo runs as a single script. No slides, no mock data, no switching between apps. Everything happens on the real Solana devnet with real transactions.

```
1. Developer installs CLU skill into Claude Code
   "Set up my trading agent" → registered and staked in 2 minutes

2. Agent executes three trades
   Reputation score climbs from 0 to 72 automatically

3. A second agent misses its deadline
   Anyone clicks "Challenge" → automatic slash fires in under a second
   → 1,200 USDC goes to the challenger
   → Agent's reputation craters from 50 to 17
   → The record is permanent on the blockchain

4. A DeFi protocol checks both agents
   → Agent A: trusted, access granted
   → Agent B: not trusted, access denied
   → No human made this decision. The on-chain record did.
```

That last line is the pitch. The system works without us.

---

## The pitch in one sentence

> **CLU is the accountability skill the Solana agent ecosystem didn't know it was missing — every DeFi skill teaches an agent how to interact with a protocol, CLU is the skill that makes that agent answerable for what it does.**

---

## Frequently asked questions from the team

**"Can't a developer just register a new agent after being slashed?"**

Yes — but reputation starts at zero and stake has to be deposited fresh. The slash count on the old agent is permanent and public. Over time, DeFi protocols will require agents to have sustained track records, not just a clean new account. And the community stakers who lost money on the bad agent will not back the new one.

**"What stops someone from filing fake challenges to drain agents?"**

The challenger has to post a bond. If the challenge is invalid — the evidence does not check out — the challenger loses their bond. Filing fake challenges is financially punishing. And the evidence check is automatic and on-chain, so there is no one to bribe or convince.

**"Why would DeFi protocols use CLU instead of just building their own vetting?"**

Because building their own vetting is hard, expensive, and centralized. With CLU, a single line of code gives them access to a shared reputation system built by hundreds of operators and stakers over time. The network effect is the product. The more agents that register, the more valuable the trust signal becomes for every protocol.

**"Is this only for Solana?"**

For now yes — we are built on Solana specifically. The Agent Skills standard is most active on Solana right now and that is where the agent economy is developing fastest. Cross-chain is a post-launch consideration.

**"What is our business model?"**

Two revenue streams. First, the x402 API — protocols pay a small fee (0.001 USDC) per agent lookup. Second, the platform fee on every task registration — when a trader hires an agent, a portion of their payment goes to CLU. Both are automatic and on-chain. We do not invoice anyone.

**"Who pays the gas fees when an agent executes a transaction?"**

The trader does — indirectly. Part of the subscription fee they pay when hiring an agent is automatically allocated to the agent's wallet as a gas top-up before the task starts. Operators never manually fund agent wallets, and agents never stall mid-task because they ran out of SOL. The protocol handles this automatically on every task registration.

---

*CLU · April 2026 · Colosseum Frontier Hackathon*
