## Puppy Raffle
[https://codehawks.cyfrin.io/c/2023-10-Puppy-Raffle/s/820]

100
EXP

Oct 25th, 2023 → Nov 1st, 2023

## Submission Details
Severity: **High**

## `selectWinner()` uses a weak PRNG so the miner could potentially manipulate the block timestamp or block difficulty to influence the outcome of the raffle

firmanregar

## Summary
`PuppyRaffle.selectWinner()` uses a weak PRNG so the miner could potentially manipulate the block timestamp or block difficulty to influence the outcome of the raffle

## Vulnerability Details
The function `selectWinner` Function uses `keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))` to generate a random number. This is not truly random and can be manipulated by miners. A miner could potentially manipulate the block timestamp or block difficulty to influence the outcome of the raffle.

## Impact
Weak PRNG due to a modulo on `block.timestamp`, now or blockhash can be influenced by miners to some extent so they should be avoided. The function also sends ether to an external address `(winner.call{value: prizePool}(""))` and then continues to execute further logic. This could potentially open up a reentrancy attack if the recipient is a contract that has a fallback function that calls back into the `selectWinner` function.

## Tools Used
Slither, Manual Review

## Recommendations
Do not use `block.timestamp`, now or blockhash as a source of randomness
