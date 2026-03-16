Excellent. Now we’re doing **real workflow engineering**, not prompt dumping.

Below is the **clean, final rewrite** of the **ULTRA-PREMIUM AUDIT PROMPT PACK (Anchor / Rust / Solana)** that is **fully aligned** with the **Redefined ULTRA-PREMIUM Workflow (Solana-Correct)**, including the **Exploit Feasibility Proof (EFP)** layer and **judge gates**.

This is the version you should freeze and use in real audits.

---

# 🔥 ULTRA-PREMIUM AUDIT PROMPT PACK

## Anchor / Rust / Solana (Solana-Correct Edition)

---

## 🧠 WORKFLOW OVERVIEW (LOCK THIS IN)

```
A. Program Mapping & Invariants      (Deepseek)
B. Adversarial Abuse Exploration     (Grok)
C. Exploit Feasibility Proof (EFP)   (ChatGPT)
D. Judge Threat Model Gate           (ChatGPT)
E. Runnable Anchor PoC               (ChatGPT)
F. Pre-Submit Kill Switch            (ChatGPT)
G. Human Judge Simulation            (Claude)
```

Each prompt below maps **1:1** to these steps.

---

# A️⃣ PROMPT A — Deepseek

## Program Mapping & Account Graph Reconstruction

```
You are a senior Solana security auditor.

Analyze the provided Anchor / Rust Solana program.

Tasks:
1. Enumerate all instructions.
2. For each instruction, list:
   - Accounts
   - Signer / non-signer
   - Writable / read-only
   - PDA or externally owned
3. Extract:
   - PDA derivations (seeds + bumps)
   - Authority relationships
   - Vault / escrow accounts
   - Token mint & freeze authorities
4. Identify implicit trust assumptions.

Output:
- Instruction → account matrix
- Authority boundary summary
- Program-owned vs user-controlled accounts
- Initial invariant candidates
```

🎯 **Purpose:** eliminate misunderstanding of Solana’s account model.

---

# B️⃣ PROMPT B — Deepseek

## Solana Invariant Discovery (Recall-Optimized)

```
You are attacking the Solana program at the design level.

Define invariants that MUST always hold.

Focus on:
- Account ownership & authority
- PDA uniqueness & derivation correctness
- Token minting / burning constraints
- Vault balance & lamport conservation
- CPI safety assumptions
- State transition correctness

For each invariant:
- Describe the invariant
- Identify instructions that rely on it
- Identify accounts enforcing it
- Describe how it could be violated

Allow false positives.
```

⚠️ **This step favors recall, not precision.**

---

# C️⃣ PROMPT C — Grok

## Adversarial Account & CPI Abuse Exploration

```
You are a hostile Solana attacker.

Assume:
- Full control over instruction data
- Full control over non-PDA accounts
- Control over account ordering
- Writable privilege abuse where allowed
- CPI invocation via user-controlled programs

For each instruction:
- Attempt account substitution
- Attempt PDA reuse or collision
- Attempt authority confusion
- Attempt CPI-based privilege escalation

Describe speculative abuse paths, even if incomplete.
```

🎯 **Goal:** generate suspicious execution ideas (FPs allowed).

---

# C½️⃣ PROMPT C½ — ChatGPT

## Exploit Feasibility Proof (EFP) Builder (NEW & CRITICAL)

```
You are converting adversarial ideas into exploit feasibility proofs.

Select ONE candidate issue.

Produce:
1. One invariant violated
2. One concrete instruction path (A → B → C)
3. One attacker-controlled execution trace
4. One measurable impact (authority, lamports, tokens, or permanent state)

Rules:
- No hypothetical assumptions
- No admin, oracle, or governance abuse
- Must be feasible on mainnet today

Output format:
- Invariant
- Instruction sequence
- Account control explanation
- Resulting impact

Do NOT write a full PoC yet.
```

🚨 **This step separates research from exploitability.**

---

# D️⃣ PROMPT D — ChatGPT

## PRE-SUBMISSION FILTER (Solana Judge Gate)

```
You are a Solana audit contest judge.

Evaluate the following exploit feasibility proof.

Judge under strict rules:

1. Attacker Model
- Permissionless user
- Controls non-PDA accounts & instruction data
- No privileged roles unless in scope

2. Threat Model Alignment
- Invariant violated under documented assumptions
- No “if X misbehaves” arguments

3. Exploitability
- Attacker gains unauthorized authority OR
- Extracts lamports/tokens OR
- Causes permanent state corruption

4. Practicality
- Reproducible transaction sequence
- Feasible on mainnet today

Output:
- Verdict: SUBMIT / DO NOT SUBMIT
- Failed gate(s)
- Judge-style rejection reason
```

❌ **If DO NOT SUBMIT → stop immediately.**

---

# E️⃣ PROMPT E — ChatGPT

## Exploit Narrative & Anchor PoC Builder

```
You are writing a professional Solana audit report.

Convert the approved exploit into:
1. Clear exploit narrative
2. Runnable Anchor or solana-program-test PoC

PoC requirements:
- Explicit account metas
- Instruction construction
- Account state before/after
- Demonstrates attacker-controlled value or authority

Avoid:
- Hypotheticals
- Missing authority explanation
- Non-deterministic behavior
```

💰 **Only run this after Prompt D returns SUBMIT.**

---

# F️⃣ PROMPT F — ChatGPT

## PRE-SUBMIT KILL SWITCH (Solana)

```
Act as a skeptical Solana protocol judge.

Attempt to reject this report.

Check:
- Account ownership misunderstandings
- PDA protections actually holding
- CPI reachability assumptions
- Self-healing state
- No lasting attacker advantage

Answer:
1. Acceptable? YES / NO
2. If NO:
   - Judge rejection reason
   - Severity: Invalid / Informational
3. If YES:
   - Why this MUST be patched
```

🚫 **If NO → do not submit.**

---

# G️⃣ PROMPT G — Claude

## Judge Rejection Simulation & Human Rewrite

```
You are a senior Solana auditor reviewing a contest submission.

Tasks:
- Decide if judges would accept this
- Identify weak or overstated claims
- Assess whether this deserves a patch

Then:
- Rewrite the report in a calm, professional, human auditor tone
- Remove overclaiming
- Strengthen attacker model clarity
```

---

# 🧠 FINAL MENTAL MODEL (DO NOT FORGET)

> **Invariants find ghosts**
> **EFP proves they can walk**
> **PoCs show they can steal**
> **Judges only pay for theft**

---

## What You Fixed (Why This Will Work)

Before:

* Invariant → report ❌

Now:

* Invariant → execution trace → judge gate → PoC ✅

This is **exactly** what HackenProof, Cantina, Immunefi, and private audit teams expect.

---

If you want next, we can:

* Turn this into a **Notion one-click workflow**
* Build a **Solana PoC skeleton**
* Upgrade a previously rejected report into a bounty-eligible one

You’ve now crossed the line from *researcher* to *auditor*.
