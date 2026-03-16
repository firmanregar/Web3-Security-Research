Perfect — now we’re closing the loop.

Below is the **EXACT ULTRA-PREMIUM AUDIT PROMPT PACK — NEAR Protocol Edition**, rewritten **end-to-end** to match **NEAR’s async execution model**, **judge expectations**, and **bounty eligibility criteria**.

This is **not** a cosmetic rewrite.
This is structurally different from EVM/Solana and aligned with **HackenProof / Immunefi / private NEAR audits**.

Freeze this and use it.

---

# 🔥 ULTRA-PREMIUM AUDIT PROMPT PACK

## **NEAR Protocol (Async-Correct Edition)**

---

## 🧠 NEAR-CORRECT WORKFLOW (LOCK THIS FIRST)

```
A. Contract & Promise Graph Mapping      (Deepseek)
B. Async Invariant Discovery              (Deepseek)
C. Adversarial Promise Abuse              (Grok)
D. Exploit Feasibility Proof (Async)      (ChatGPT)
E. Judge Threat Model Gate (NEAR)          (ChatGPT)
F. Runnable NEAR PoC                      (ChatGPT)
G. Pre-Submit Kill Switch                 (ChatGPT)
H. Judge Simulation + Human Rewrite       (Claude)
```

Everything below maps **1:1** to this flow.

---

# A️⃣ PROMPT A — Deepseek

## **NEAR Contract & Promise Graph Mapping**

```
You are a senior NEAR Protocol security auditor.

Analyze the provided NEAR smart contracts.

Tasks:
1. Enumerate all public/external methods.
2. Identify all cross-contract calls.
3. Reconstruct promise chains:
   - Initial call
   - Then callbacks
   - Failure paths
4. Identify state mutation points:
   - Before external calls
   - Inside callbacks
5. Identify balance-affecting logic:
   - Deposits
   - Withdrawals
   - Refunds

Output:
- Contract call graph
- Promise / callback graph
- State mutation timeline
- Trust assumptions per async boundary
```

🎯 **Goal:** make async execution explicit.

---

# B️⃣ PROMPT B — Deepseek

## **Async Invariant Discovery (Recall-Optimized)**

```
You are attacking the NEAR protocol at the async design level.

Define invariants that MUST hold across asynchronous execution.

Focus on:
- Callback idempotency
- Funds credited only after final resolution
- State consistency across promise boundaries
- Exactly-once execution guarantees
- Gas exhaustion safety
- Partial execution rollback assumptions

For each invariant:
- Describe the invariant
- Identify which async boundary enforces it
- Identify how it could be violated

Allow false positives.
```

⚠️ Favor recall. Precision comes later.

---

# C️⃣ PROMPT C — Grok

## **Adversarial Async Abuse Exploration**

```
You are a hostile NEAR attacker.

Assume:
- Full control over transaction timing
- Ability to trigger cross-contract calls
- Ability to manipulate promise resolution order
- Ability to cause gas exhaustion
- Ability to replay or re-trigger callbacks where logic allows

For each promise chain:
- Attempt callback replay
- Attempt double execution
- Attempt state update before promise resolution
- Attempt gas-based partial execution
- Attempt refund duplication

Describe speculative attack paths, even if incomplete.
```

🎯 **Goal:** surface async failure modes.

---

# C½️⃣ PROMPT C½ — ChatGPT

## **Exploit Feasibility Proof (Async EFP)** 🚨 **CRITICAL**

```
Convert one async vulnerability candidate into an Exploit Feasibility Proof.

Produce:
1. One async invariant violated
2. One concrete promise chain
3. One attacker-controlled execution ordering
4. One measurable impact:
   - Double credit
   - Early withdrawal
   - Permanent state corruption
   - Locked or lost funds

Rules:
- No admin or governance assumptions
- Must be feasible on NEAR mainnet today
- No hypothetical “if NEAR breaks” arguments

Output:
- Invariant
- Promise sequence
- State transitions
- Final impact
```

🧠 This is the **make-or-break step** for NEAR.

---

# D️⃣ PROMPT D — ChatGPT

## **PRE-SUBMISSION FILTER (NEAR Judge Gate)**

```
You are a NEAR audit contest judge.

Evaluate the following exploit feasibility proof.

Judge strictly:

1. Attacker Model
- Permissionless NEAR account
- Controls transaction timing & async calls
- No privileged roles assumed

2. Threat Model Alignment
- Invariant violated under NEAR async semantics
- No assumptions outside protocol scope

3. Exploitability
- Attacker gains funds OR
- Causes permanent protocol/user harm OR
- Forces unrecoverable bad state

4. Practicality
- Reproducible async execution trace
- Demonstrable via NEAR test environment

OUTPUT:
- Verdict: SUBMIT / DO NOT SUBMIT
- Failed gate(s)
- Judge-style rejection reason
```

❌ If **DO NOT SUBMIT** → stop.

---

# F️⃣ PROMPT F — ChatGPT

## **Runnable NEAR PoC Builder**

```
You are writing a professional NEAR audit report.

Convert the approved exploit into a runnable PoC.

Requirements:
- Use near-sdk-sim or near-workspaces
- At least two contracts if cross-contract
- Explicit promise chaining
- Demonstrate before/after balances or state
- Deterministic reproduction

Do NOT:
- Skip async steps
- Assume callback ordering
- Rely on theoretical execution
```

💰 This is where research becomes bounty-eligible.

---

# G️⃣ PROMPT G — ChatGPT

## **PRE-SUBMIT KILL SWITCH (NEAR)**

```
Act as a skeptical NEAR protocol judge.

Try to reject this report.

Check for:
- Async misunderstanding
- Callback executed only once in reality
- Gas exhaustion assumptions invalid
- State actually self-heals
- No persistent attacker advantage

Answer:
1. Acceptable? YES / NO
2. If NO:
   - Judge rejection reason
   - Severity: Invalid / Informational
3. If YES:
   - Why this MUST be patched
```

🚫 If **NO** → do not submit.

---

# H️⃣ PROMPT H — Claude

## **Judge Simulation & Human Rewrite (NEAR)**

```
You are a senior NEAR auditor reviewing a contest submission.

Evaluate:
- Would judges accept this?
- Is the async exploit real or theoretical?
- Would this cause real fund loss?

Then:
- Rewrite in calm, professional, human tone
- Remove exaggeration
- Clarify async attacker control
```

---

# 🧠 NEAR-SPECIFIC MENTAL MODEL (MEMORIZE)

> **EVM breaks functions**
> **Solana breaks authority**
> **NEAR breaks time**

If your bug doesn’t exploit **async ordering**, it’s probably not real.

---

# ✅ Why This Version Will Pass Where Others Failed

Before:

* Invariant → report ❌
* Async reasoning → “interesting research” ❌

Now:

* Invariant → async execution trace → judge gate → PoC ✅

This matches **exactly** what HackenProof feedback was telling you — implicitly.

---

## Final One-Sentence Rule (NEAR Edition)

> **If you cannot point to a specific promise chain and show how state is finalized incorrectly, do not submit.**

---

If you want next, we can:

* Turn this into a **NEAR audit Notion workspace**
* Build a **NEAR async PoC skeleton**
* Upgrade one of your rejected NEAR findings into a bounty-eligible submission

You’re now operating at **multi-chain senior auditor level**, Mate.
