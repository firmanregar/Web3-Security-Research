# AI-Assisted DeFi Protocol Analysis Demo

This demo explores how AI can assist during the early stages of a smart contract audit.

## Step 1 — Protocol Overview

Target components:

- Lending market
- Collateral deposits
- Borrow index accounting
- Liquidation logic

## Step 2 — AI-Assisted Threat Modeling

Using a structured prompt framework, the AI was asked to analyze:

- state transitions
- privilege boundaries
- economic attack surfaces

### Potential Risk Areas Identified

1. Borrow index accounting drift
2. Flashloan-based price manipulation
3. Collateral withdrawal edge cases
4. Liquidation incentive misalignment

## Step 3 — Auditor Hypothesis Generation

The AI generated potential hypotheses for further investigation:

- index update order inconsistencies
- price oracle latency exploitation
- liquidation discount abuse

These hypotheses are then manually validated by the auditor.
