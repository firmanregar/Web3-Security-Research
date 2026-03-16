# AI Audit Hypothesis Generator

## Overview

One of the most difficult parts of smart contract auditing is forming the **right hypotheses** about how a protocol might be attacked.

Experienced auditors often rely on mental models built from studying past exploits. This experiment explores whether AI can assist by generating **structured vulnerability hypotheses** based on protocol architecture.

The goal is not to produce final findings, but to help researchers ask better questions during an audit.

---

## Motivation

Security research often follows this pattern:

1. Understand protocol design
2. Identify critical state transitions
3. Form attack hypotheses
4. Attempt to validate those hypotheses

The third step — hypothesis formation — is where experience plays a significant role.

This experiment explores whether AI can help accelerate this stage by proposing **potential vulnerability scenarios** based on known exploit patterns.

---

## Input Data

The AI receives structured information about the protocol, such as:

* protocol components
* key functions
* state variables
* external dependencies
* economic assumptions

Example protocol elements:

Vault share accounting
Borrow index updates
Collateralization checks
Oracle price feeds
Liquidation incentives

---

## Hypothesis Generation Process

The AI analyzes protocol structure and compares it against known vulnerability patterns.

Common pattern categories include:

* accounting inconsistencies
* price oracle manipulation
* precision and rounding errors
* privilege escalation
* reentrancy conditions
* economic incentive misalignment

Based on this analysis, the AI generates **investigation hypotheses**.

---

## Example Hypotheses

Hypothesis 1
Borrow index updates may occur after debt calculation, potentially allowing underpayment of debt.

Hypothesis 2
Oracle price update delays may allow flashloan-based price manipulation.

Hypothesis 3
Collateral share calculations may introduce rounding errors that allow gradual value extraction.

Hypothesis 4
Liquidation incentive parameters may allow liquidators to extract excessive collateral.

These hypotheses act as **starting points for manual testing and validation**.

---

## Auditor Workflow Integration

A possible workflow could be:

1. Auditor reviews protocol architecture
2. AI generates potential vulnerability hypotheses
3. Auditor investigates each hypothesis manually
4. Validated issues become formal audit findings

This approach treats AI as a **research assistant**, not a vulnerability detector.

---

## Limitations

AI-generated hypotheses may include:

* false positives
* unrealistic attack scenarios
* incomplete reasoning

Human verification remains essential.

The value of this system lies in **expanding the auditor's thinking space**, helping them consider attack vectors they might otherwise overlook.

---

## Research Direction

Future work may explore:

* integrating exploit databases
* learning from historical DeFi incidents
* mapping hypotheses to specific code locations
* building automated threat modeling pipelines

The long-term goal is to explore how AI can augment the **cognitive process of security auditing**.
