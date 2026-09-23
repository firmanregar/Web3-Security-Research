# Zero-Value Split Spam Can Exhaust Merkle Capacity

**Program:** Zendex  
**Platform:** HackenProof  
**Report ID:** ZNDXSCDD-73  
**Outcome:** Valid, High

## Summary

Repeated zero-value branches provide a route to grow shared Merkle state; underlying issue was known.

## Portfolio Note

Technical validity and bounty eligibility are separate dimensions. This entry preserves the platform's recorded outcome.

## Key Takeaway

Analysis centers on reachable execution paths, root cause, violated invariants, impact boundaries, and remediation reasoning.

## Final Disposition

**Severity:** High  
**Status:** Valid — Previously Unreported  
**Reward:** Not rewarded  
**Reason:** The program only rewards Critical findings  
**Reputation impact:** None

### Triage Correction

The report was initially closed as a duplicate of F-2026-16602.

On September 16, 2026, the triage team corrected that assessment and confirmed that this report is **not a duplicate** and was **not previously reported**.

The distinction is between two separate mechanisms:

- F-2026-16602 concerns the snapshot loop at `TreeOperator` L138–144.
- This finding concerns the fixed-capacity guard `require(leafCount < MAX_LEAVES)` at L104.

The triage team also confirmed that the remediation prescribed for F-2026-16602 does not address the capacity exhaustion path described here.

The final severity was assessed as **High**, rather than Critical, because the full-tree condition causes several trustless entry paths to revert but does not permanently lock the underlying tokens. `withdraw` and `removeLiquidity` remain available, while `createOrder` continues to operate because it consumes without inserting.

The protocol retains an administrative recovery path through `ZendexVault.release()`, so the impact was characterized as permanent loss of the **trustless exit**, rather than permanent loss of the underlying assets.

The finding was not eligible for a bounty because the program's rules only reward Critical issues.

No reputation penalty was applied.
