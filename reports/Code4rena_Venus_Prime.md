# Code4rena Venus Prime

# Severity Classification

| Letter | Name | Description |
|:--:|:-------:|:-------:|
| H  | High risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 1 |
|:--:|:--:|

# Findings Summary

| ID     | Title                                                                                 | Severity |
| ------ | -------------------------------------------------------------------------------       | -------- |
| [H-01] | Dos - Updatescores will always revert                            | High |

# [H-1] Dos - Updatescores will always revert

## Lines of code

https://github.com/code-423n4/2023-09-venus/blob/main/contracts/Tokens/Prime/Prime.sol#L331

https://github.com/code-423n4/2023-09-venus/blob/main/contracts/Tokens/Prime/Prime.sol#L397

## Vulnerability details

### Impact

In issue() and claim() to mint new prime token, I don't see totalScoreUpdatesRequired and pendingScoreUpdates are updated.

it might cause function performing updateScores(address[] memory users) always revert.

totalScoreUpdatesRequired and pendingScoreUpdates only updated when perform updateAlpha(), updateMultipliers() and addMarket().

### Proof of Concept

Scenario 1:

1. After calling addMarket, for example , the values are:totalScoreUpdatesRequired=2, pendingScoreUpdates=2a. totalScoreUpdatesRequired= totalIrrevocable + totalRevocable;
2. Admin calls issus() to mint 2 new revocable prime token to 2 users.
3. The 2 users withdraw 1,000 XVS each and burn their prime token.a.as now both totalScoreUpdatesRequired=0 and pendingScoreUpdates=0
4. Attempting to execute updateScores() will always result in a revert.a. if (pendingScoreUpdates == 0) revert NoScoreUpdatesRequired();

Scenario 2:

1. After calling addMarket, for example , the values are:totalScoreUpdatesRequired=2, pendingScoreUpdates=2a. totalScoreUpdatesRequired= totalIrrevocable + totalRevocable;
2. Admin calls issue() to mint 2 new revocable prime tokens for 2 users.
3. Perform updateScores() with 4 users will always revert. Due to underflow in pendingScoreUpdates--;

```solidity
    function updateScores(address[] memory users) external {
        if (pendingScoreUpdates == 0) revert NoScoreUpdatesRequired(); //@audit-issue revert
        if (nextScoreUpdateRoundId == 0) revert NoScoreUpdatesRequired();

        for (uint256 i = 0; i < users.length; ) {
            address user = users[i];

            if (!tokens[user].exists) revert UserHasNoPrimeToken();
            if (isScoreUpdated[nextScoreUpdateRoundId][user]) continue;

            address[] storage _allMarkets = allMarkets;
            for (uint256 j = 0; j < _allMarkets.length; ) {
                address market = _allMarkets[j];
                _executeBoost(user, market);
                _updateScore(user, market);

                unchecked {
                    j++;
                }
            }

            pendingScoreUpdates--; //audit-issue underflow
            isScoreUpdated[nextScoreUpdateRoundId][user] = true;

            unchecked {
                i++;
            }

            emit UserScoreUpdated(user);
        }
    }

```

### Tools Used

VSC

### Recommended Mitigation Steps

When claim and issue should keep totalScoreUpdatesRequired and pendingScoreUpdates are updated.

### Assessed type

DoS
