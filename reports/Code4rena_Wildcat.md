# Code4rena Venus Prime

# Severity Classification

| Letter | Name | Description |
|:--:|:-------:|:-------:|
| H  | High risk | Loss of funds |
| M |  Medium risk | Unexpected behavior |
| L  | Low risk | Potentially a risk |

| Total Found Issues | 2 |
|:--:|:--:|

# Findings Summary

| ID     | Title                                                                                 | Severity |
| ------ | -------------------------------------------------------------------------------       | -------- |
| [H-01] | Lack of appropriate function in WildcatMarketController to close the market                            | High |
| [H-02] | Wrong parameters used in calls to createEscrow function                            | High |

# [H-01] Lack of appropriate function in WildcatMarketController to close the market

## Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L142

## Vulnerability details

The `WildcatMarket` contract has a `closeMarket` function that should allow the borrower to close the existing market if needed and the required conditions are met:

```
File: WildcatMarket.sol
133:   /**
134:    * @dev Sets the market APR to 0% and marks market as closed.
135:    *
136:    *      Can not be called if there are any unpaid withdrawal batches.
137:    *
138:    *      Transfers remaining debts from borrower if market is not fully
139:    *      collateralized; otherwise, transfers any assets in excess of
140:    *      debts to the borrower.
141:    */
142:   function closeMarket() external onlyController nonReentrant {
...
```

The `closeMarket` function could be called only through the market controller due to it's access control modifier.

The motivation for existing such functionality is clearly described in protocol documentation:

```
In the event that a borrower has finished utilising the funds for the purpose that the market was set up to facilitate (or if lenders are choosing not to withdraw their assets and the borrower is paying too much interest on assets that have been re-deposited to the market), the borrower can close a market at will.

```

However, if we check the code of `WildcatMarketController` we can't find an appropriate function that would allow the borrower to call `closeMarket` at the market address. This would lead to the situation described earlier in docs - borrowers would need to pay interest for funds their not used after finished utilizing the funds.

### Impact

Borrowers cannot close the market if needed and would need to pay interest for funds their not used after finished utilizing the funds.

### Proof of Concept

1. Borrower creates a new market.
2. Lenders deposit their funds into the market.
3. Borrower uses provided funds and later returns them back to the market.
4. Lenders do not withdraw their funds, interests continue to accrue for their deposits.
5. Borrower can't close the market and is forced to pay interest for funds that are not utilized anymore.

This bug is missed in tests since they implemented in the way the caller is pranked as a `controller` address, while it should be an actual call through not existed `WildcatMarketController#closeMarket` function - L202:

```
File: WildcatMarket.t.sol
198:   // ===================================================================== //
199:   //                             closeMarket()                              //
200:   // ===================================================================== //
201:
202:   function test_closeMarket_TransferRemainingDebt() external asAccount(address(controller)) {
203:     // Borrow 80% of deposits then request withdrawal of 100% of deposits
204:     _depositBorrowWithdraw(alice, 1e18, 8e17, 1e18);
205:     startPrank(borrower);
206:     asset.approve(address(market), 8e17);
207:     stopPrank();
208:     vm.expectEmit(address(asset));
209:     emit Transfer(borrower, address(market), 8e17);
210:     market.closeMarket();
211:   }
```

### Recommended Mitigation Steps

Consider implementing a function in `WildcatMarketController` that would allow the borrower to call the `closeMarket` function on the market contract.

### Assessed type

Other

# [H-02] Wrong parameters used in calls to createEscrow function

## Lines of code

https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L172-L175

https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L166-L169

## Vulnerability details

When a lender address becomes sanctioned due to sanction oracle, the market could create a separate Escrow contract that would hold the lender balance, or 2 Escrow contracts if sanctioned lender call `WildcatMarketWithdrawals#executeWithdrawal`.

Escrow is created using the `WildcatSanctionsSentinel#createEscrow()` function, which expects the next order of parameters:

```
File: WildcatSanctionsSentinel.sol
87:   /**
88:    * @dev Creates a new WildcatSanctionsEscrow contract for `borrower`,
89:    *      `account`, and `asset` or returns the existing escrow contract
90:    *      if one already exists.
91:    *
92:    *      The escrow contract is added to the set of sanction override
93:    *      addresses for `borrower` so that it can not be blocked.
94:    */
95:   function createEscrow(
96:     address borrower,
97:     address account,
98:     address asset
99:   ) public override returns (address escrowContract) {
```

However, if we check how this function is called at `WildcatMarketBase#_blockAccount`, we can see that the wrong order of parameters was used - `accountAddress` and `borrower` switched their places:

```
File: WildcatMarketBase.sol
163:   function _blockAccount(MarketState memory state, address accountAddress) internal {
164:     Account memory account = _accounts[accountAddress];
165:     if (account.approval != AuthRole.Blocked) {
166:       uint104 scaledBalance = account.scaledBalance;
167:       account.approval = AuthRole.Blocked;
168:       emit AuthorizationStatusUpdated(accountAddress, AuthRole.Blocked);
169:
170:       if (scaledBalance > 0) {
171:         account.scaledBalance = 0;
172:         address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
173:           accountAddress,
174:           borrower,
175:           address(this)
176:         );
```

This issue has a second appearance at WildcatMarketWithdrawals.sol#166, in this case, 2 Escrow contracts with wrong parameters would be created - one for market balance in `_blockAccount` call and the second for asset balance:

```
File: WildcatMarketWithdrawals.sol
164:     if (IWildcatSanctionsSentinel(sentinel).isSanctioned(borrower, accountAddress)) {
165:       _blockAccount(state, accountAddress);
166:       address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
167:         accountAddress,
168:         borrower,
169:         address(asset)
170:       );
```

### Impact

An escrow contract (or 2 contracts) would be created with the borrower's address as an expected receiver of sanctioned funds, while it should be the lender's address.

### Proof of Concept

The next test is an update of the existing test `test_executeWithdrawal_Sanctioned` in `test/market/WildcatMarketWithdrawals.t.sol` and could show a scenario when an Escrow contract created with wrong parameters, allowing the borrower to instantly withdraw assets that should be available only for the sanctioned lender:

```
  function test_executeWithdrawal_Sanctioned() external {
    _deposit(alice, 1e18);
    _requestWithdrawal(alice, 1e18);
    fastForward(parameters.withdrawalBatchDuration);
    sanctionsSentinel.sanction(alice);
    address escrow = sanctionsSentinel.getEscrowAddress(alice, borrower, address(asset));
    vm.expectEmit(address(asset));
    emit Transfer(address(market), escrow, 1e18);
    vm.expectEmit(address(market));
    emit SanctionedAccountWithdrawalSentToEscrow(alice, escrow, uint32(block.timestamp), 1e18);
    market.executeWithdrawal(alice, uint32(block.timestamp));

    // This check fails since Alice is not an actual "account" in escrow
    assertEq(alice, WildcatSanctionsEscrow(escrow).account(), "Account address at escrow is not Alice");

    // This check shows that the borrower could instantly withdraw funds that should be stored for Alice
    uint256 _balance = asset.balanceOf(borrower);
    WildcatSanctionsEscrow(escrow).releaseEscrow();
    uint256 balance_ = asset.balanceOf(borrower);
    assertEq(_balance, balance_, "Borrower balance increased");
  }
```

### Recommended Mitigation Steps

Consider updating the order of parameters at affected lines in `WildcatMarketBase.sol`#L172 and `WildcatMarketWithdrawals.sol`#L166

### Assessed type

Other
