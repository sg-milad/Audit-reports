# Last Man Standing - Findings Report

- ## Medium Risk Findings
  - ### [M-01. Critical Logic Flaw in claimThrone(): Throne Cannot Be Claimed Due to Inverted Check](#M-01)
  - ### [M-02. Missing Previous King Payout Breaks Reward Mechanism in `claimThrone()`](#M-02)
- ## Low Risk Findings
  - ### [L-01. Incorrect Event Emission in `declareWinner()` Causes Misleading Prize Data](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #45

### Dates: Jul 31st, 2025 - Aug 7th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-07-last-man-standing)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 2
- Low: 1

# Medium Risk Findings

## <a id='M-01'></a>M-01. Critical Logic Flaw in claimThrone(): Throne Cannot Be Claimed Due to Inverted Check

# Root + Impact

## Description

- Normally, the `claimThrone()` function should allow any user who is **not** the current king to claim the throne by meeting the required conditions (e.g., sending enough ETH, etc.).

- However, the condition `require(msg.sender == currentKing)` is incorrectly implemented. This logic only allows the current king to reclaim the throne, defeating the purpose of a game mechanic where others compete to become the new king. The check should instead prevent the current king from reclaiming their own throne.

```Solidity
require(
@>     msg.sender == currentKing,
        "Game: You are already the king. No need to re-claim."
);
```

## Risk

**Likelihood**:

- This will always occur when any user tries to claim the throne and they are **not** the current king.

- The contract logic enforces a false precondition that contradicts the intended functionality.

**Impact**:

- No new user can ever claim the throne, rendering the core mechanic of the game useless.

- The contract becomes entirely non-functional in its intended context, potentially locking up funds and making it unusable.

## Proof of Concept

The following Foundry test demonstrates that new players are unable to claim the throne due to the incorrect logic in the `require` statement.

```Solidity
  function testClaimThrone_CannotBeUsedByNewPlayers() public {
        // Player1 tries to claim the throne but fails because they're not the current king
        vm.startPrank(player1);
        vm.expectRevert("Game: You are already the king. No need to re-claim.");
        game.claimThrone{value: INITIAL_CLAIM_FEE}();
        vm.stopPrank();
    }
```

## Recommended Mitigation

To fix the logic error and restore the correct behavior of the game, update the `require` condition to **reject** the current king and allow new players to claim the throne.

```diff
- require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
+ require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
```

## <a id='M-02'></a>M-02. Missing Previous King Payout Breaks Reward Mechanism in `claimThrone()`

# Root + Impact

## Description

The function comment states: "If there's a previous king, a small portion of the new claim fee is sent to them." However, the implementation doesn't actually pay out to the previous king.

## Risk

**Likelihood**:

- This will always happen whenever a new player claims the throne after someone else — the previous king is never rewarded.
- The issue is systemic and tied directly to how the claim logic is structured, meaning it affects every throne transition.

**Impact**:

- Breaks the core reward system of the contract — players are not incentivized to become king if they don’t earn anything after being dethroned.
- Damages user trust and game dynamics; may cause players to abandon the game or never participate at all.

## Proof of Concept

The following Foundry test demonstrates that the previous king does not receive any payout when a new player claims the throne:

```Solidity
  function testMissingPreviousKingPayout() public {
        // First player claims the throne
        vm.startPrank(player1);
        uint256 player1InitialBalance = player1.balance;
        game.claimThrone{value: INITIAL_CLAIM_FEE}();
        uint256 player1BalanceAfterClaim = player1.balance;
        vm.stopPrank();

        // Verify player1's balance decreased by the claim fee
        assertEq(
            player1InitialBalance - player1BalanceAfterClaim,
            INITIAL_CLAIM_FEE,
            "Player1 balance should decrease by claim fee"
        );

        // Get the updated claim fee for the next player
        uint256 nextClaimFee = game.claimFee();

        // Second player claims the throne
        vm.startPrank(player2);
        uint256 player1BalanceBeforeSecondClaim = player1.balance;
        game.claimThrone{value: nextClaimFee}();
        uint256 player1BalanceAfterSecondClaim = player1.balance;
        vm.stopPrank();

        // Previous king (player1) should have received a portion of player2's claim fee
        // But in reality, no payout happens
        assertEq(
            player1BalanceAfterSecondClaim,
            player1BalanceBeforeSecondClaim,
            "Previous king should not have received any payout"
        );

        // If previous king payout was implemented correctly, player1's balance would have increased
        // But it remains unchanged, proving the payout logic is missing
    }
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect Event Emission in `declareWinner()` Causes Misleading Prize Data

# Root + Impact

## Description

- The `declareWinner()` function should emit a `GameEnded` event that accurately reflects the final pot amount distributed to the winner. Off-chain systems, analytics tools, and UIs rely on this event for displaying accurate game results.

- However, the contract resets the `pot` to zero before emitting the `GameEnded` event. As a result, the event logs a prize amount of `0`, which does not match the actual payout that occurred internally via `pendingWinnings[currentKing] += pot`.

```Solidity
  pot = 0; // Reset pot after assigning to winner's pending winnings

  emit GameEnded(currentKing, pot, block.timestamp, gameRound);// @> Event emits 0 as the prize
```

## Risk

**Likelihood**:

- This will always occur when `declareWinner()` is called , the event will report the pot as zero every time.

- Developers and users relying on event logs for prize tracking or frontend display will consistently receive incorrect information.

**Impact**:

- Event logs provide misleading data about the amount won, breaking trust with users and developers.

- Off-chain services such as subgraphs, dashboards, or game histories will show incorrect winnings for each round.

## Recommended Mitigation

Capture the pot amount in a temporary variable before resetting it, and use that for emitting the event.

```diff
-       pendingWinnings[currentKing] = pendingWinnings[currentKing] + pot;
-       pot = 0; // Reset pot after assigning to winner's pending winnings
-       emit GameEnded(currentKing, pot, block.timestamp, gameRound);

+       pendingWinnings[currentKing] = pendingWinnings[currentKing] + pot;
+       emit GameEnded(currentKing, pot, block.timestamp, gameRound);
+       pot = 0; // Reset pot after assigning to winner's pending winnings

```

It preserves the accurate prize value for the event while still resetting the pot afterward.

External systems consuming the GameEnded event will now receive correct and consistent information.
