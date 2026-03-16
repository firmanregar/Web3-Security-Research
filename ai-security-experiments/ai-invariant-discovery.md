# AI-Assisted Protocol Invariant Discovery

## Overview

This experiment explores whether AI can assist security researchers in identifying **protocol invariants** during smart contract audits.

Protocol invariants are rules that must always remain true for a system to remain secure and economically sound. Many smart contract vulnerabilities ultimately arise when these invariants are violated.

The goal of this experiment is to investigate whether AI can help auditors reason about protocol logic and propose candidate invariants that should be verified during security reviews.

---

## Motivation

Complex DeFi protocols often involve multiple interacting components such as:

* vaults
* lending pools
* collateral management systems
* price oracles
* liquidation engines

Each component introduces new state transitions and economic assumptions. As a result, identifying the invariants that guarantee system safety can be challenging.

Experienced auditors often develop invariants mentally while reading protocol logic. This experiment explores whether AI can help generate **candidate invariants** that guide the auditing process.

---

## Example Protocol Components

Consider a simplified lending protocol consisting of:

Vault
Stores user deposits and manages asset balances.

Borrowing Module
Allows users to borrow assets against deposited collateral.

Oracle
Provides price feeds for collateral valuation.

Liquidation Engine
Handles liquidation of undercollateralized positions.

---

## Example Invariants

AI may propose invariants such as:

Total collateral value must always exceed total outstanding debt.

Vault asset balance must equal the sum of all user share balances.

Borrow index must never decrease over time.

Liquidation rewards must not exceed the collateral being seized.

These invariants represent safety properties that auditors can test and verify.

---

## AI-Assisted Invariant Discovery

The experiment explores whether AI can help derive invariants by analyzing:

* protocol state variables
* asset flows
* accounting relationships
* economic assumptions

By understanding how protocol components interact, AI may suggest rules that must remain true for the protocol to function safely.

These candidate invariants can then be validated by human auditors or tested using automated tools such as invariant testing frameworks.

---

## Example Auditor Workflow

A possible workflow could look like this:

1. Auditor reviews protocol architecture
2. AI proposes candidate invariants
3. Auditor validates and refines the invariants
4. Invariants are tested using fuzzing or invariant testing tools
5. Violations are investigated for potential vulnerabilities

In this workflow, AI acts as a **reasoning assistant**, helping auditors identify safety conditions that should be preserved.

---

## Limitations

AI-generated invariants may include:

* incomplete assumptions
* overly broad rules
* incorrect interpretations of protocol logic

Human verification remains essential.

The purpose of this experiment is to support auditor reasoning rather than replace manual analysis.

---

## Research Direction

Future research may explore:

* linking invariant discovery with fuzz testing tools
* generating invariant-based security tests
* integrating protocol invariant databases
* learning invariants from historical exploit patterns

Ultimately, this line of research investigates how AI might assist in formalizing the reasoning process that experienced security researchers use during complex protocol audits.
