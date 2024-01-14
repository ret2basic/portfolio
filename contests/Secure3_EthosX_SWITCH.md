# Secure3 EthosX SWITCH

# Severity Classification

| Letter | Name | Description |
|:--:|:-------:|:-------:|
| C  | Critical risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 8 |
|:--:|:--:|

# Findings Summary

| ID     | Title                                                                                 | Severity |
| ------ | -------------------------------------------------------------------------------       | -------- |
| [C-01] | Vault.mint(): First depositor sandwich attack (aka. inflation attack)                            | Critical |
| [M-01] | SettlementManger.sol lacks stale price checks when calling latestRoundData()                    | Medium   |
| [M-02] | Unchecked return value for transfer() and transferFrom() | Medium   |
| [L-01] | SwitchToken centralization risk                          | Low      |

# [C-01] Vault.mint(): First depositor sandwich attack (aka. inflation attack)

### description

#### Impact

An attacker can sandwich the first depositor of the vault and steal depositor's asset.

#### Vulnerability Detail

In `Vault.mint()`, the first depositor's mint() call goes into the following if block:

```solidity
            if (overall_total_deposit_shares_ == 0) {
                shares_to_issue_ = deposit_amount_;
            } else {
                ...
            }
```

This is vulnerable to the famous ERC4626 inflation attack. A detailed explanation of this attack can be found here: https://tienshaoku.medium.com/eip-4626-inflation-sandwich-attack-deep-dive-and-how-to-solve-it-9e3e320cc3f1

#### PoC

Here is a summary of the attack:

1. Attacker observes first depositor's buyLongToken() or buyShortToken() call in mempool. Say the depositor's tx says the deposit amount is 100e6 tokens.
2. Attacker frontruns this tx by depositing 1 wei of token. Attacker gets 1 share here.
3. Within the same tx, attacker transfers 100e6 token to the vault contract. This "inflates" the share -> `overall_deposit_token_amount_` is supposed to be 1 wei but now it is 100e6 + 1.
4. Victim's tx gets executed. This goes into the else block:

```solidity
            if (overall_total_deposit_shares_ == 0) {
                ...
            } else {
                shares_to_issue_ =
                    (deposit_amount_ * overall_total_deposit_shares_) /
                    overall_deposit_token_amount_;
            }
```

Recall that victim is depositing 100e6 tokens. Plug that into the formula, we have `shares_to_issue = (100e6 * 1) / (100e6 + 1)`. Note that this rounds down to 0, therefore victim receives 0 share.

5. Attacker withdraws the share by calling Vault.claim(). Since there is only 1 share in the vault and the share is owned by the attacker, this withdrawal takes all the tokens out of the contract, which is around 200e18. By conducting this attack, victim's asset is stolen by the attacker.


### recommendation

When `overall_total_deposit_shares_ == 0`, consider minting 1000 shares to address(0) to "dilute" each share's value. This method is called creating "dead shares". It was first used by Uniswap V2 to mitigate the inflation attack. A more systematic mitigation is to employ OpenZeppelin ERC4626 (new version) in Vault.sol.

### locations

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Tokens/Vault.sol#L102-L108

### severity

Critical

### category

Logical

# [M-01] SettlementManger.sol lacks stale price checks when calling latestRoundData()

### description

Multiple functions in `SettlementManger.sol` call `latestRoundData()` to get price info from Chainlink price feed:

- latestCycleStartPrice()
- latestCycleStartTime()
- checkUpkeep()
- performUpkeep()

None of the above functions implemented checks for stale price. For example, in latestCycleStartPrice():

```solidity
        (, int256 last_round_px_, , uint256 last_round_time_, ) = _price_feed
            .latestRoundData();

        if (cycles_count_ == 0) {
            settlement_cycles.push(
                CycleData({
                    cycle_start_time: last_round_time_,
                    cycle_start_price: uint256(last_round_px_),
                    cycle_end_time: 0,
                    cycle_end_price: 0
                })
            );
            return uint256(last_round_px_);
        }
```

Here `last_round_px_` and `last_round_time_` are used without being validated. If stale price is used, all upcoming computations will be influenced.

### recommendation

 Check that `last_round_px_` and `last_round_time_` are in acceptable ranges.

### locations

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Utils/SettlementManager.sol#L72-L73

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Utils/SettlementManager.sol#L127-L128

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Utils/SettlementManager.sol#L195-L196

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Utils/SettlementManager.sol#L235-L242

### severity

Medium

### category

Logical

# [M-02] Unchecked return value for transfer() and transferFrom()

### description

Tokens such as USDC do not revert upon failure. Instead, they just return false. Multiple functions in the protocol calls `transfer()` and `transferFrom()` directly without checking their return values. The transfer might silently fail but the tx won't revert.

### recommendation

Use OpenZeppelin SafeERC20 library for transferring tokens.

### locations

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Controller/ControllerBase.sol#L323-L327

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Controller/ControllerBase.sol#L378-L382

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Controller/ControllerBase.sol#L738

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Controller/ControllerBase.sol#L782

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Controller/ControllerBase.sol#L914

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Tokens/Vault.sol#L157-L160

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Tokens/Vault.sol#L239-L243

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Tokens/Vault.sol#L291

### severity

Medium

### category

Logical

# [L-01] SwitchToken centralization risk

### description

Owner can call SwitchToken.transferFrom() without any allowance:

```solidity
    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        address spender = _msgSender();
        if (spender != owner()) {
            _spendAllowance(from, spender, amount);
        }
        _transfer(from, to, amount);
        return true;
    }
```

Malicious owner could steal all users's long and short tokens by calling this transferFrom() and specifying `to` as his own address.


### recommendation

Treat owner as an average user and count allowance.

### locations

https://github.com/Secure3Audit/audit_EthosX-SWITCH_ret2basic/blob/f647560a9d2dfa06b6c45d2e431e3b139c3b3290/code/packages/hardhat/contracts/Tokens/SwitchToken.sol#L25-L27

### severity

Medium

### category

Logical
