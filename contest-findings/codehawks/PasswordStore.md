# First Flight #1: PasswordStore
[https://codehawks.cyfrin.io/c/2023-10-PasswordStore/s/777]

100
EXP

Oct 18th, 2023 → Oct 25th, 2023

## Submission Details
Severity: **High**
Non-protected Setter allow the caller to change password of the contracts

firmanregar

## Summary
Function PasswordStore.setPassword(string) (src/PasswordStore.sol#26-29) is a non-protected setter s_password is written

## Vulnerability Details
Usually, all setters must be protected with access control

## Impact
Allow the caller to change the password of the contracts

## Tools Used
Slither

## Recommendations
Add access control
