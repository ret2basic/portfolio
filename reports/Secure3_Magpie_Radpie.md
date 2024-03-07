# Secure3 Magpie Radpie, esRDNT, OFT & CCIP Bridge

# Severity Classification

| Letter | Name | Description |
|:--:|:-------:|:-------:|
| C  | Critical risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 2 |
|:--:|:--:|

# Findings Summary

| ID     | Title                                                                                 | Severity |
| ------ | -------------------------------------------------------------------------------       | -------- |
| [M-01] | RadpieCCIPBridge.tokenTransfer(): Users can't withdraw excessive ETH sent to the contract                            | Medium |
| [L-01] | RadpieOFTBridge._debitFrom(): msg.sender has to approve himself/herself before calling                    | Low   |

# [M-01] RadpieCCIPBridge.tokenTransfer(): Users can't withdraw excessive ETH sent to the contract

### Location

https://github.com/Secure3Audit/code_Magpie_CCIP/blob/cf6caf72c5b14f7de34f80a0fb2e328cd4f4a8fd/code/contracts/crosschain/RadpieCCIPBridge.sol#L113-L151

### Description

Suppose a user sent more ETH to the `RadpieCCIPBridge` contract via `tokenTransfer()`. Currently there is no way to withdraw the excessive ETH so those ETH will be stuck in the contract forever.

### Recommendation

Compute excess ETH sent to the contract:

```solidity
msg.value - actualCost
```

and store this value inside a local accounting system. Implement a separate `withdraw()` function to let users withdraw themselves. This follows the pull over push pattern.

# [L-01] RadpieOFTBridge._debitFrom(): msg.sender has to approve himself/herself before calling

### Location

https://github.com/Secure3Audit/code_Magpie_CCIP/blob/cf6caf72c5b14f7de34f80a0fb2e328cd4f4a8fd/code/contracts/crosschain/RadpieOFTBridge.sol#L54

### Description

`RadpieOFTBridge._debitFrom()` uses `burnFrom()` from OpenZeppelin `ERC20BurnableUpgradeable.sol`:

```solidity
    function _debitFrom(
        address _from,
        uint16,
        bytes memory,
        uint _amount
    ) internal virtual override returns (uint) {
        require(_from == _msgSender(), "IndirectOFT: owner is not send caller");
        RADPIE.burnFrom(_from, _amount);
        return _amount;
    }
    function burnFrom(address account, uint256 value) public virtual {
        _spendAllowance(account, _msgSender(), value);
        _burn(account, value);
    }
```

`burnFrom()` computes allowance. In other words, `msg.sender` has to approve himself/herself to get allowance before burning. This is an anti-pattern and it could lead to mistakes.

### Recommendation

Use `burn()` instead of `burnFrom()`:

```solidity
RADPIE.burnFrom(_amount);
```
