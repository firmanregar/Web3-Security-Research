## Quadratic Duplicate Check Enables DoS on Raffle Entry
Matched: [M-01] `PuppyRaffle: enterRaffle` Use of gas extensive duplicate check leads to Denial of Service, making subsequent participants to spend much more gas than prev ones to enter

## Root + Impact Description
The duplicate address check in `enterRaffle()` uses nested loops that scale as O(n²), where n is the number of players. As the player array grows, the gas cost of entering eventually exceeds block limits, making it impossible for anyone to join.

The duplicate check iterates over the entire players array with nested loops:
```solidity
for (uint256 i = 0; i < players.length - 1; i++) {
    for (uint256 j = i + 1; j < players.length; j++) {
        require(players[i] != players[j], "PuppyRaffle: Duplicate player");
    }
}
```
This performs `n * (n-1) / 2 comparisons`. As players accumulate, the gas cost grows quadratically until it hits the block gas limit.

## Risk
Medium — Temporary but predictable DoS that degrades protocol functionality.

**Impact:**
Once the array reaches the threshold (typically a few hundred entries depending on gas limits), new players can't enter. The raffle becomes stuck — it can't grow to completion, and the protocol effectively stops functioning for that round.

## Proof of Concept
While `selectWinner()` eventually resets the array after `raffleDuration` expires, this still represents a significant denial of service window where the raffle can't accept new participants.
```solidity
for (uint256 i = 0; i < players.length - 1; i++) {
    for (uint256 j = i + 1; j < players.length; j++) {
        require(players[i] != players[j], "Duplicate player");
    }
}
```
​
## Recommended Mitigation
Replace the O(n²) loop with an O(1) mapping check:
```solidity
mapping(address => uint256) public addressToRaffleId;
uint256 public raffleId = 0;
​
function enterRaffle(address[] memory newPlayers) public payable {
    require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
    
    for (uint256 i = 0; i < newPlayers.length; i++) {
        require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
        addressToRaffleId[newPlayers[i]] = raffleId;
        players.push(newPlayers[i]);
    }
    
    emit RaffleEnter(newPlayers);
}
​
function selectWinner() external {
    // ... existing code ...
    
    raffleId++;  // Increment after selecting winner
    delete players;
}
```
