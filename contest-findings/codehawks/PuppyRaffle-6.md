## Predictable Randomness Allows Winner Manipulation
Matched: [H-03] Randomness can be gamed

## Root + Impact Description
The contract uses on-chain values to generate randomness for winner selection. Since these values are either publicly known or miner-influenced, an attacker can predict or bias the outcome.

The winner index calculation relies entirely on deterministic or manipulable inputs:
```solidity
uint256 winnerIndex = uint256(keccak256(abi.encodePacked(
    msg.sender,          // Attacker-controlled
    block.timestamp,     // Miner-influenced (±15 seconds)
    block.difficulty     // Miner-influenced
))) % players.length;
```

## Risk
**Likelihood:**
Undermines protocol integrity, though exploitation requires effort or privileged position.

**Impact:**
An attacker can influence the raffle outcome in two ways:
* Repeated attempts — Try calling `selectWinner()` from different addresses or at different times until they get a favorable outcome
* Miner collusion — A miner can manipulate `block.timestamp` and `block.difficulty` to bias results, or simply refuse to include unfavorable transactions.

This breaks the fairness guarantee that's central to a raffle system.

## Proof of Concept
```
msg.sender is chosen by the caller
block.timestamp can be manipulated by miners within tolerance bounds
block.difficulty is public and predictable
```

## Recommended Mitigation
Use a verifiable random function like Chainlink VRF. This provides cryptographically secure randomness that can't be predicted or manipulated:
```solidity
// Example integration (simplified)
function selectWinner() external {
    require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
    
    // Request random number from Chainlink VRF
    requestRandomness(keyHash, fee);
}
​
function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
    uint256 winnerIndex = randomness % players.length;
    // ... rest of winner selection logic
}
```
