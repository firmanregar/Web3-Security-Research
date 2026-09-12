# Unprotected Internal Swap Enables LP Sandwiching

**Program:** Zendex  
**Platform:** HackenProof  
**Report ID:** ZNDXSCDD-60  
**Outcome:** Medium — Confirmed OOS

## Summary

`amountOutMin: 0` leaves the internal swap leg of single-sided LP deposits sandwichable.

## Portfolio Note

Technical validity and bounty eligibility are separate dimensions. This entry preserves the platform's recorded outcome.

## Key Takeaway

Analysis centers on reachable execution paths, root cause, violated invariants, impact boundaries, and remediation reasoning.
