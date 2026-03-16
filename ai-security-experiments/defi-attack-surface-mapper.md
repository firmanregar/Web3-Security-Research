# AI-Assisted DeFi Attack Surface Mapper

## Overview

This experiment explores how AI can assist security researchers in mapping the **attack surface of DeFi protocols** during the early stages of a smart contract audit.

Rather than attempting to automatically detect vulnerabilities, the goal is to help auditors **structure their reasoning** by identifying:

* critical protocol components
* state transition points
* privilege boundaries
* economic attack surfaces

The output serves as a **threat modeling assistant** that helps researchers quickly identify areas that require deeper manual analysis.

---

## Motivation

DeFi protocols often consist of multiple interacting components such as:

* vaults
* lending pools
* oracles
* liquidation engines
* strategy contracts
* governance modules

The complexity of these interactions makes it difficult to immediately identify where vulnerabilities may exist.

This experiment investigates whether AI can help generate a **structured attack surface map** that guides auditors toward potential risk areas.

---

## Example Protocol Architecture

For demonstration purposes, consider a simplified DeFi lending protocol consisting of:

* **Vault**
* **Lending Pool**
* **Oracle**
* **Liquidation Engine**
* **Flashloan Module**

### Component Responsibilities

Vault
Stores user deposits and issues shares representing ownership.

Lending Pool
Allows users to borrow assets against collateral.

Oracle
Provides asset price data used for collateralization checks.

Liquidation Engine
Handles liquidation of undercollateralized positions.

Flashloan Module
Allows atomic borrowing and repayment within a single transaction.

---

## Step 1 — Identify Critical State Transitions

AI can assist in identifying points where **important state changes occur**.

Example transitions:

Deposit → vault balance increases
Borrow → user debt index updated
Repay → borrower debt reduced
Liquidation → collateral transferred to liquidator

These transitions represent **high-value attack targets** because they directly affect protocol accounting.

---

## Step 2 — Map Privilege Boundaries

Another key step is identifying actors with special permissions.

Typical roles include:

* protocol admin
* governance
* liquidation bots
* external users
* strategy contracts

AI can help enumerate which functions are callable by each role and highlight where **privileged operations occur**.

---

## Step 3 — Identify Economic Attack Surfaces

AI can assist in generating potential economic attack vectors based on protocol design.

Examples include:

Flashloan price manipulation
Oracle update latency exploitation
Collateral share inflation
Borrow index accounting drift
Liquidation incentive manipulation

These are not confirmed vulnerabilities but **hypotheses for further investigation**.

---

## Step 4 — Attack Surface Summary

After analyzing the protocol structure, AI can generate a structured map of potential attack surfaces.

Example categories:

Accounting Layer
Debt index updates, share calculations, rounding behavior

Price Layer
Oracle updates, TWAP delays, external price feeds

Liquidity Layer
Flashloans, pool balance changes, deposit/withdraw logic

Liquidation Layer
Collateral seizure logic, liquidation incentives

Governance Layer
Admin privileges, upgrade mechanisms, parameter changes

---

## Research Goal

The goal of this experiment is to explore how AI can help auditors move faster from:

"reading code"

to

"reasoning about attack surfaces"

By structuring protocol components and highlighting potential risk areas, AI can act as a **threat modeling assistant** that improves audit efficiency without replacing human judgment.
