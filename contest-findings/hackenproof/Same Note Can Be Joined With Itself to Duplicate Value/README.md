# Same Note Can Be Joined With Itself to Duplicate Value

**Program:** Zendex  
**Platform:** HackenProof  
**Report ID:** ZNDXSCDD-95  
**Outcome:** **Valid — Critical** — Previously accepted as **Valid / Fix Shipping / OOS**

## Summary

Missing distinctness in the join proof plus validate-before-consume nullifier handling allows duplicate input use.

The same underlying note can be supplied as both inputs to `join()`, producing a new note worth twice the real deposited value. The resulting note is fully spendable, allowing an attacker to withdraw the manufactured value from the pooled vault.

The finding was ultimately accepted as a **valid Critical vulnerability** and is reward-eligible.

## Triage History

The finding was initially closed as **Out of Scope** because the missing distinctness constraint was located in `circuits/join/src/main.nr`, outside the program's listed Solidity assets.

The technical validity and impact were not disputed, and the client was already shipping a fix.

After further deliberation, the triage team **withdrew the earlier Out of Scope closure** and accepted the finding as a **valid Critical vulnerability**.

The final determination recognized that the exploit executes through the listed `ZendexVaultManager` and `TreeOperator` assets and results in direct, permissionless, repeatable theft of pooled end-user funds.

## Key Takeaway

The important invariant is:

> **A joined note must be backed by two distinct input notes.**

This case demonstrates why security analysis should follow an invariant across the complete execution path (proof circuit, nullifier handling, vault accounting, and withdrawal) rather than stopping at the component where the missing constraint originates.
