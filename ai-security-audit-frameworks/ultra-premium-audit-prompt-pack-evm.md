redefined ULTRA-PREMIUM AUDIT PROMPT PACK with EVM EFP.md

Perfect — this is the **final missing structural upgrade**.
Below is the **redefined ULTRA-PREMIUM AUDIT PROMPT PACK (EVM / Solidity Edition)** with the **EVM-specific Exploit Feasibility Proof (EFP)** *explicitly baked in*, aligned with everything we fixed for **Solana** and **NEAR**.

This version closes the loop on:

* “valid but not exploitable”
* theoretical accounting bugs
* PoC-first overcommitment
* judge trust-model rejections

Freeze this version.

---

# 🔥 ULTRA-PREMIUM AUDIT PROMPT PACK

## **EVM / Solidity (EFP-Correct Edition)**

---

## 🧠 EVM-CORRECT WORKFLOW (NON-NEGOTIABLE)

```
A. Protocol Mapping & Invariants        (DeepSeek)
B. Adversarial Exploit Ideation         (Grok)
C. Exploit Feasibility Proof (EFP)      (ChatGPT)   ← NEW, CRITICAL
D. Judge Threat Model Gate              (ChatGPT)
E. Runnable Foundry PoC                 (ChatGPT)
F. Pre-Submit Kill Switch               (ChatGPT)
G. Judge Simulation & Human Rewrite     (Claude)
```

> **Rule:**
> No finding may skip **C → D → E** in this order.

---

# A️⃣ PROMPT A — DeepSeek

## **EVM Protocol Mapping & State Cartography**

```
You are a senior EVM smart contract auditor.

Analyze the provided Solidity codebase.

Tasks:
1. Enumerate all externally callable functions.
2. Identify:
   - State variables
   - Accounting mechanisms
   - Access control
3. Map call graphs:
   - Internal calls
   - External calls
   - Delegatecalls
4. Identify trust assumptions:
   - Oracles
   - Admin roles
   - External protocols

Output:
- Function → state mutation map
- Call graph
- Explicit and implicit invariants
- Attacker-controllable inputs
```

🎯 Goal: literal understanding only. No exploits yet.

---

# B️⃣ PROMPT B — Grok

## **Adversarial Exploit Ideation (FP-Tolerant)**

```
You are a hostile attacker.

Assume:
- Permissionless access
- Flash liquidity where realistic
- Control over calldata, timing, ordering

For each invariant:
- Propose ways it MIGHT be violated
- Identify suspicious execution paths
- Note preconditions and uncertainties

Do NOT assess exploitability.
Do NOT assign severity.
```

⚠️ High false-positive tolerance here is intentional.

---

# C️⃣ PROMPT C — ChatGPT

## **Exploit Feasibility Proof (EVM-Specific) 🚨**

> **This is the most important prompt in the entire system.**

```
You are constructing an Exploit Feasibility Proof (EFP) for an EVM protocol.

Select ONE adversarial idea, that will pass the attacker model evaluation, threat model alignment, exploitability, and practical evaluation gate.

Provide:

1. Invariant Violated
   - State the invariant clearly.

2. Executable Call Path
   - Exact function sequence
   - Who calls what
   - In what order
   - With which parameters

3. Attacker Control
   - Why a permissionless attacker can execute this path
   - No admin, governance, or oracle corruption unless explicitly in scope

4. State Transitions
   - Key storage changes per call
   - Why the invariant actually breaks

5. Measurable Impact
   - Token extraction
   - Debt forgiveness
   - Insolvency
   - Permanent state corruption

Rules:
- Must be executable on mainnet today
- No hypothetical assumptions
- No PoC yet
```

❌ If you cannot produce a clean EFP → **kill the finding**.

---

# D️⃣ PROMPT D — ChatGPT

## **PRE-SUBMISSION FILTER (EVM Judge Gate)**

```
You are a Immunefi / Code4rena / Sherlock / Cantina judge.

Evaluate the Exploit Feasibility Proof.

Reject unless ALL are true:

1. Attacker Model
   - Permissionless
   - No privileged roles required

2. Threat Model Alignment
   - Invariant violated under documented assumptions
   - No “if X misbehaves” arguments

3. Exploitability
   - Attacker gains extractable value OR
   - Other users suffer permanent loss OR
   - Protocol enters irrecoverable bad state

4. Practicality
   - Concrete transaction sequence
   - Economically realistic

Output:
- Verdict: SUBMIT / DO NOT SUBMIT
- Failed gate(s)
- Judge-style rejection reason
```

🚫 **DO NOT SUBMIT = STOP. No PoC.**

---

# E️⃣ PROMPT E — ChatGPT

## **Runnable Foundry Exploit PoC**

```
Convert the approved exploit into a runnable Foundry test.

Requirements:
- forge-std/Test.sol
- Minimal setup
- Deterministic exploit
- Before/after balance or state assertions
- Proves attacker advantage

Do NOT:
- Use unrealistic mocks
- Skip critical setup
- Assume off-chain behavior
```

💰 This is where exploitability is proven, not argued.

---

# F️⃣ PROMPT F — ChatGPT

## **PRE-SUBMIT KILL SWITCH (EVM)**

```
Act as a skeptical EVM contest judge.

Attempt to reject this report.

Check for:
- Design-intended behavior
- Self-healing state
- No real economic gain
- Unrealistic assumptions
- Weak PoC assertions

Answer:
1. Acceptable? YES / NO
2. If NO:
   - Rejection reason
   - Severity: Invalid / Informational
3. If YES:
   - Why this MUST be patched
```

❌ Any NO → do not submit.

---

# G️⃣ PROMPT G — Claude

## **Judge Simulation & Human Rewrite**

```
You are a senior auditor reviewing a contest submission.

Tasks:
- Would judges accept this?
- Does the exploit feel real?
- Would you patch this in production?

Then:
- Rewrite the report in clear, neutral, human tone
- Remove overclaiming
- Tighten attacker model clarity
```

---

# 🧠 FINAL UNIFIED MENTAL MODEL (ALL CHAINS)

> **EVM:** prove the function path
> **Solana:** prove the authority path
> **NEAR:** prove the time path

All three require:

> **Invariant → EFP → Judge Gate → PoC**

---

# Why This Redefinition Fixes Your EVM Rejections

Before:

* Invariant → report ❌
* PoC without feasibility discipline ❌

Now:

* Invariant → **Executable path** → Judge gate → PoC ✅

This is **exactly** how senior auditors avoid:

> “valid but not exploitable”

---

## Final Rule (Lock This In)

> **If you cannot explain the exploit in one executable call path,
> do not write a PoC, and do not submit.**

You’ve now unified **EVM, Solana, and NEAR** under a single **EFP-centric audit doctrine**.

If you want next:

* I can merge all three chains into **one master Ultra-Premium pack**
* Or convert this into a **Notion decision tree**
* Or practice killing borderline EVM findings together

You’re operating at real senior-auditor discipline now.

=================================

## Temporal Invariant Stress Test (TIST)

You are an elite EVM protocol auditor operating under zero-trust assumptions.
Your task is to perform a Temporal Invariant Stress Test (TIST) on the following
Veda protocol components:

- AccountantWithYieldStreaming.sol
- TellerWithYieldStreaming.sol
- WithdrawQueue.sol
- DelayedWithdraw.sol
- Pauser.sol
- BoringVault.sol

You MUST assume:
- No privileged access (no owner, strategist, or pauser cooperation)
- Attacker can only use publicly callable functions
- Mainnet conditions only (no governance changes, no upgrades)
- Multi-transaction and multi-block attacks are allowed
- Time (block.timestamp) is adversarial but monotonic

---

### STEP 1 — STATE & TIME MODELING

Construct a complete state machine that includes:
- Share minting and burning
- Yield accrual and yield streaming
- Withdrawal requests and queue progression
- Paused vs unpaused states
- Stream start, update, cancellation, and finalization

Explicitly model:
- Which state variables are time-dependent
- Which transitions depend on block.timestamp
- Which transitions can occur while paused

---

### STEP 2 — TEMPORAL INVARIANT EXTRACTION

Derive AT LEAST 5 temporal invariants of the form:

- "If condition A holds at time T, then condition B MUST hold at time T + Δ"
- "It is impossible for state X to persist longer than Y blocks"
- "Funds locked at time T must become withdrawable by time T + Δ"

Examples (do not reuse these):
- Yield cannot continue accruing after principal withdrawal
- Shares cannot remain locked after yield stream termination

For each invariant:
- Specify the variables involved
- Specify the assumed time bounds
- Specify why violation implies fund loss or permanent freeze

---

### STEP 3 — ADVERSARIAL TIMELINE CONSTRUCTION

For EACH invariant:
- Construct at least one adversarial timeline using:
  - Deposits
  - Withdrawals
  - Yield streaming updates
  - Pause / unpause transitions
  - Queue interactions

Each timeline must:
- Use only public functions
- Be executable on mainnet without privileged roles
- Span multiple blocks if necessary

---

### STEP 4 — IMPACT CLASSIFICATION (IMMUNEFI-ALIGNED)

For every violated invariant:
- Classify the impact as one of:
  - Direct fund theft
  - Permanent freezing of funds
  - Protocol insolvency
- Identify EXACTLY which funds are affected
- Specify whether the loss is:
  - Immediate
  - Delayed but irreversible

Discard any scenario that relies on:
- MEV assumptions
- Liquidity movement
- Oracle manipulation
- Privileged intervention

---

### STEP 5 — CRITICAL EXPLOIT SELECTION

From all viable timelines:
- Select the SINGLE exploit with the highest funds-at-risk
- Explain why mitigation (pause, refund, upgrade) does NOT negate severity
- Identify the minimal function call sequence required

---

### STEP 6 — PoC READINESS CHECK

For the selected exploit, confirm:
- Can be demonstrated on a mainnet fork
- Does not rely on undefined behavior
- Does not require unrealistic gas or timing assumptions

Output:
1. The violated temporal invariant
2. The exact attack timeline
3. The affected contracts
4. Why this qualifies as a CRITICAL Immunefi submission