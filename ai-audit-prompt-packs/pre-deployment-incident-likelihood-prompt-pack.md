complete ULTRA-PREMIUM PRE-DEPLOYMENT INCIDENT LIKELIHOOD PROMPT PACK.md

Perfect. What you’re asking for now is **not an audit prompt pack** — it’s a **failure-forecasting prompt system**.

Below is a **complete ULTRA-PREMIUM PRE-DEPLOYMENT INCIDENT LIKELIHOOD PROMPT PACK**, designed to **predict real-world protocol failure modes** *before* launch, across:

* **EVM / Solidity**
* **Solana / Anchor / Rust**
* **NEAR Protocol**

This pack is **orthogonal** to your Ultra-Premium Audit + EFP system:

* Audit = *Can an attacker steal funds?*
* Incident Map = *How will this system actually fail in production?*

Senior firms do this quietly. You’re formalizing it.

---

# 🔥 ULTRA-PREMIUM

## PRE-DEPLOYMENT INCIDENT LIKELIHOOD PROMPT PACK

### *(Failure Forecasting, Not Bug Finding)*

---

## 🧠 GLOBAL PRINCIPLE (NON-NEGOTIABLE)

> **Most real incidents are not exploits.**
> They are **assumption collapses under real-world conditions**.

Your goal is **NOT correctness**.
Your goal is **anticipation**.

---

# 🧩 WORKFLOW OVERVIEW (ALL CHAINS)

```
A. Assumption Extraction            (DeepSeek)
B. Failure Mode Enumeration         (Grok)
C. Incident Feasibility Proof (IFP) (ChatGPT)
D. Severity × Likelihood Scoring    (ChatGPT)
E. Operator / Judge Sanity Check    (Claude)
```

> 🔴 Note: This uses **IFP (Incident Feasibility Proof)**,
> not EFP. There is no attacker requirement.

---

# A️⃣ PROMPT A — DeepSeek

## **Protocol Assumption Extraction**

```
You are a senior protocol engineer performing pre-deployment risk analysis.

Extract ALL assumptions the protocol relies on.

Include assumptions about:
- Admin / governance behavior
- Keepers / operators
- External protocols
- Economic parameters
- Timing and liveness
- Gas availability
- User behavior

Do NOT assess exploits.
Do NOT assess correctness.

Output:
- Assumption list
- Where each assumption lives (code / off-chain / governance)
- What breaks if the assumption fails
```

🎯 Goal: make invisible dependencies visible.

---

# B️⃣ PROMPT B — Grok

## **Adversarial Failure Mode Exploration**

```
You are a hostile universe, not an attacker.

Assume:
- Humans make mistakes
- Bots go offline
- Gas spikes
- Markets behave adversarially
- External protocols degrade
- Governance reacts late

For each assumption:
- Describe realistic ways it could fail
- Describe how the system behaves when it fails
- Identify whether failure is sudden or gradual

Do NOT propose exploits.
Focus on real-world breakdowns.
```

🎯 Goal: enumerate *how the system dies*, not how it’s hacked.

---

# C️⃣ PROMPT C — ChatGPT

## **Incident Feasibility Proof (IFP) — CRITICAL**

> This is the equivalent of EFP, but for incidents.

```
Select ONE high-risk failure mode.

Produce an Incident Feasibility Proof (IFP):

1. Broken assumption
2. Concrete execution or operational sequence
3. Actors involved (users, keepers, admins, markets)
4. State evolution over time
5. Observable impact:
   - Funds locked
   - Funds slowly drained
   - Insolvency
   - User exits blocked
   - System halted

Rules:
- No attacker required
- Must be realistic on mainnet
- Must not rely on protocol misbehavior outside assumptions
```

🚨 If you cannot produce a concrete failure sequence → discard.

---

# D️⃣ PROMPT D — ChatGPT

## **Severity × Likelihood Scoring (Incident Map Core)**

```
Evaluate the incident feasibility proof.

Score qualitatively:

- Likelihood:
  Low / Medium / High

- Impact:
  Low / Medium / High / Catastrophic

- Detectability:
  Easy / Delayed / Hard

Classify into:
A. Exploitable Bug
B. Trust Boundary Failure
C. Economic Design Collapse
D. Operational Failure
E. External Dependency Failure

Output:
- Category
- Score
- Short rationale
```

🎯 Goal: prioritize what will hurt first.

---

# E️⃣ PROMPT E — Claude

## **Operator / Judge Sanity Check**

```
You are a protocol operator and post-mortem author.

Answer:
- Would this incident surprise the team?
- Would this be blamed on a bug, ops, or governance?
- Would this cause loss of trust?

Rewrite the incident description in:
- Plain language
- Post-mortem style
- No technical defensiveness
```

🎯 Goal: force realism and accountability.

---

# 🔗 CHAIN-SPECIFIC ADAPTATIONS (MANDATORY)

---

## 🔹 EVM / Solidity

### Failure modes to emphasize:

* Admin misconfiguration
* Parameter drift
* Oracle latency (even if trusted)
* Gas griefing
* Upgrade errors
* Incentive misalignment

### Replace attacker language with:

> “Any user / market condition can cause…”

---

## 🔹 Solana / Anchor / Rust

### Failure modes to emphasize:

* Keeper liveness
* CPI failure propagation
* Rent changes
* Account exhaustion
* Writable privilege assumptions
* RPC inconsistency

### Focus on:

> **Authority + liveness**, not logic.

---

## 🔹 NEAR Protocol

### Failure modes to emphasize:

* Async callback assumptions
* Promise resolution ordering
* Gas exhaustion mid-promise
* Partial state persistence
* Callback replay logic

### Focus on:

> **Time and ordering**, not atomic execution.

---

# 🧠 OUTPUT: INCIDENT LIKELIHOOD MAP (FINAL ARTIFACT)

Your final output should look like this:

```
MOST LIKELY FIRST INCIDENTS:
1. Keeper downtime causes redemption backlog (Operational, High)
2. Economic leakage via parameter drift (Economic, Medium)

LOW PROBABILITY / HIGH IMPACT:
3. Accounting exploit (Exploit, Low but Catastrophic)

OUT-OF-SCOPE BUT REAL:
4. Admin upgrade error (Trust)
5. External protocol failure (Ecosystem)
```

This is **not a vulnerability list**.
It is a **failure forecast**.

---

# 🚫 IMPORTANT: WHAT THIS PACK IS NOT

❌ Not a bug bounty workflow
❌ Not judge-submittable
❌ Not about “finding issues”

Trying to submit these as bugs is a category error.

---

# 🧠 FINAL GOLDEN LAW (INCIDENT EDITION)

> **Audits reduce exploit risk.
> Incident maps reduce surprise.
> Only teams that do both survive.**

---

## If You Want Next

We can:

* Combine **Audit + Incident Map** into a **single pre-launch playbook**
* Turn this into a **Notion decision system**
* Apply this to a live protocol you’re auditing
* Simulate a **future post-mortem before it happens**

This is now **protocol resilience engineering**, not just security research.
