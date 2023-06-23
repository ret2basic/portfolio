# Code4rena Juicebox Buyback Delegate

# Table of content

- [M-1] Swap leftovers ETH are locked in the `JBXBuybackDelegate`
- [M-2] Uniswap swap may be forced to revert, and mint tokens unexpectedly
- [M-3] Partial loss of user value as a result of lack of transaction deadline implementation
- [L-1] Unnecessary computation when _data.preferClaimedTokens=false
- [L-2] No sanity checks in the constructor can lead to issues or redeployments
- [L-3] Inconsistency between documentation and code around ERC20 compliance
- [L-4] Address of tokens in pool can be different than the ones passed to the constructor
- [L-5] Unnecessary calculation of _nonReservedTokenInContract

# [M-1] Swap leftovers ETH are locked in the `JBXBuybackDelegate`

## Lines of code  
  
[https://github.com/R-E-A-C-H/blackpanda-juicebox/blob/9a36e5c8d0588f0f262a0cd1c08e34b2184d8f4d/juice-buyback/contracts/JBXBuybackDelegate.sol#L263-L275](https://github.com/R-E-A-C-H/blackpanda-juicebox/blob/9a36e5c8d0588f0f262a0cd1c08e34b2184d8f4d/juice-buyback/contracts/JBXBuybackDelegate.sol#L263-L275)  

## Vulnerability details

In case that the project JBToken address is bigger than WETH address, `_projectTokenIsZero` is set to false. The test cases of buyback delegate only cover the situation, where the JBToken is lower than WETH.  
  
```solidity
    constructor(  
        IERC20 _projectToken,  
        IWETH9 _weth,  
        IUniswapV3Pool _pool,  
        IJBPayoutRedemptionPaymentTerminal3_1 _jbxTerminal  
    ) {  
        projectToken = _projectToken;  
        pool = _pool;  
        jbxTerminal = _jbxTerminal;  
        _projectTokenIsZero = address(_projectToken) < address(_weth); // <======= sets tokens ordering for UniswapV3  
```  
  
In uniswap v3 pool swapping, whenever user provides more ETH than what ProjectToken is worth of, then the remaining ETH will be send back to swap() caller, in this case, JBXBuybackDelegate.  
Since this ETH are not sent back to original caller i.e., payer, its stuck inside JBXBuybackDelegate contract.  
  
## Impact

Users are loosing Uniswap swap leftovers  
  
## Proof of Concept

Unfortunately this test is hard to setup with test cases provided by JuiceBox team, due to specific requirements concerning project setup and block height. We came up with the test, to verify the case in which the non-WETH token is considered as project token and is bigger than WETH. Please create a new file in test folder and run `forge test` to test it:  
  
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
interface IPool {
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1);
    function token0() external view returns(address);
    function token1() external view returns(address);
}

interface IERC20 {
    function balanceOf(address) external view returns (uint256);
}

interface IWETH9 {
    function deposit() external payable;
    function transfer(address, uint256) external;
    function balanceOf(address) external view returns (uint256);
}

contract Swapper {
    IPool pool;
    
    IERC20 projectToken;
    IWETH9 weth = IWETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);    

    bool _projectTokenIsZero;

    uint160 internal constant MIN_SQRT_RATIO = 4295128739;
    uint160 internal constant MAX_SQRT_RATIO = 1461446703485210103287273052203988822378723970342;

    constructor(IPool p) {
        pool = p;
        address pt = pool.token0();
        if (pool.token0() == address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2)) {
            pt = pool.token1();
        }

        projectToken = IERC20(pt);

        _projectTokenIsZero = address(projectToken) < address(weth);
    }

    function check_swap(uint amountIn, uint minOutput) external payable returns(int256, int256) {
        (int256 amount0, int256 amount1) = pool.swap({
            recipient: address(this),
            zeroForOne: !_projectTokenIsZero,
            amountSpecified: int256(amountIn),
            sqrtPriceLimitX96: _projectTokenIsZero ? MAX_SQRT_RATIO - 1 : MIN_SQRT_RATIO + 1,
            data: abi.encode(minOutput)
        });

        return (amount0, amount1);
    }

    function uniswapV3SwapCallback(int256 amount0Delta, int256 amount1Delta, bytes calldata data) external {
        // Check if this is really a callback
        require(msg.sender == address(pool), "Not Authorized");

        // Unpack the data
        (uint256 _minimumAmountReceived) = abi.decode(data, (uint256));

        // Assign 0 and 1 accordingly
        uint256 _amountReceived = uint256(-(_projectTokenIsZero ? amount0Delta : amount1Delta));
        uint256 _amountToSend = uint256(_projectTokenIsZero ? amount1Delta : amount0Delta);

        // Revert if slippage is too high
        require(_amountReceived >= _minimumAmountReceived, "min amount");

        // Wrap and transfer the weth to the pool
        weth.deposit{value: _amountToSend}(); 
        weth.transfer(address(pool), _amountToSend);
    }
    receive() external payable {}
}
contract UniswapV3ETHRefundExploitTest is Test {
    // IPool pool = IPool(0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640);
    IPool pool = IPool(0x7379e81228514a1D2a6Cf7559203998E20598346);
    
    IERC20 projectToken;
    IWETH9 weth = IWETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth");

        vm.label(address(projectToken), "ProjectToken");
        vm.label(address(weth), "WETH");

        address pt = pool.token0();
        if (pool.token0() == address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2)) {
            pt = pool.token1();
        }

        projectToken = IERC20(pt);
    }

    function testExploit() public {
        console.log("Testing for a pool :", address(pool));
        console.log("================================================================");
        Swapper swapper = new Swapper(pool);

        vm.label(address(swapper), "swapper");

        address user = makeAddr("user");
        uint256 amountIn = 1000_000 ether;
        uint256 minOutput = 0;

        vm.label(user, "user");
        vm.deal(user, amountIn);

        emit log_named_decimal_uint("Before: Balance of Pool's WETH:", weth.balanceOf(address(pool)), 18);
        emit log_named_decimal_uint("Before: Balance of Pool's Project Token:", projectToken.balanceOf(address(pool)), 18);
        console2.log();

        emit log_named_decimal_uint("Before: Balance of ETH (swapper):", address(swapper).balance, 18);
        emit log_named_decimal_uint("Before: Balance of WETH (swapper):", weth.balanceOf(address(swapper)), 18);
        emit log_named_decimal_uint("Before: Balance of Project Token (swapper):", projectToken.balanceOf(address(swapper)), 18);
        console2.log();

        emit log_named_decimal_uint("Before: Balance of ETH (user):", user.balance, 18);
        emit log_named_decimal_uint("Before: Balance of WETH (user):", weth.balanceOf(user), 18);
        emit log_named_decimal_uint("Before: Balance of Project Token (user):", projectToken.balanceOf(user), 18);
        console2.log();

        vm.prank(user);
        (int256 amount0, int256 amount1) = swapper.check_swap{value: amountIn}(amountIn, minOutput);

        console2.log("After swap: Amount 0:", amount0);
        console2.log("After swap: Amount 1:", amount1);
        console2.log();

        emit log_named_decimal_uint("After: Balance of ETH (swapper):", address(swapper).balance, 18);
        emit log_named_decimal_uint("After: Balance of WETH (swapper):", weth.balanceOf(address(swapper)), 18);
        emit log_named_decimal_uint("After: Balance of Project Token (swapper):", projectToken.balanceOf(address(swapper)), 18);
        console2.log();

        emit log_named_decimal_uint("After: Balance of ETH (user):", user.balance, 18);
        emit log_named_decimal_uint("After: Balance of WETH (user):", weth.balanceOf(user), 18);
        emit log_named_decimal_uint("After: Balance of Project Token (user):", projectToken.balanceOf(user), 18);
        console2.log();

        emit log_named_decimal_uint("After: Balance of Pool's WETH:", weth.balanceOf(address(pool)), 18);
        emit log_named_decimal_uint("After: Balance of Pool's Project Token:", projectToken.balanceOf(address(pool)), 18);
    }
}
```  

## Console Output

For **High** Liquidity Pool: 0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640

```console
  Testing for a pool : 0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640
  ================================================================
  Before: Balance of Pool's WETH:: 66269.602156132605170356
  Before: Balance of Pool's Project Token:: 0.000116014157471156
  
  Before: Balance of ETH (swapper):: 0.000000000000000000
  Before: Balance of WETH (swapper):: 0.000000000000000000
  Before: Balance of Project Token (swapper):: 0.000000000000000000
  
  Before: Balance of ETH (user):: 1000000.000000000000000000
  Before: Balance of WETH (user):: 0.000000000000000000
  Before: Balance of Project Token (user):: 0.000000000000000000
  
  After swap: Amount 0: -114230315452600
  After swap: Amount 1: 1000000000000000000000000
  
  After: Balance of ETH (swapper):: 0.000000000000000000
  After: Balance of WETH (swapper):: 0.000000000000000000
  After: Balance of Project Token (swapper):: 0.000114230315452600
  
  After: Balance of ETH (user):: 0.000000000000000000
  After: Balance of WETH (user):: 0.000000000000000000
  After: Balance of Project Token (user):: 0.000000000000000000
  
  After: Balance of Pool's WETH:: 1066269.602156132605170356
  After: Balance of Pool's Project Token:: 0.000001783842018556
```

For **Low** Liquidity Pool : 0x7379e81228514a1D2a6Cf7559203998E20598346

```console
  Testing for a pool : 0x7379e81228514a1D2a6Cf7559203998E20598346
  ================================================================
  Before: Balance of Pool's WETH:: 13972.824366210394385747
  Before: Balance of Pool's Project Token:: 29678.153610456388197439
  
  Before: Balance of ETH (swapper):: 0.000000000000000000
  Before: Balance of WETH (swapper):: 0.000000000000000000
  Before: Balance of Project Token (swapper):: 0.000000000000000000
  
  Before: Balance of ETH (user):: 1000000.000000000000000000
  Before: Balance of WETH (user):: 0.000000000000000000
  Before: Balance of Project Token (user):: 0.000000000000000000
  
  After swap: Amount 0: 29774384470067394065394
  After swap: Amount 1: -29670151004437125603226
  
  After: Balance of ETH (swapper):: 970225.615529932605934606
  After: Balance of WETH (swapper):: 0.000000000000000000
  After: Balance of Project Token (swapper):: 29670.151004437125603226
  
  After: Balance of ETH (user):: 0.000000000000000000
  After: Balance of WETH (user):: 0.000000000000000000
  After: Balance of Project Token (user):: 0.000000000000000000
  
  After: Balance of Pool's WETH:: 43747.208836277788451141
  After: Balance of Pool's Project Token:: 8.002606019262594213
```

## Tools Used  
  
Manual analysis  
  
## Recommended Mitigation Steps

We recommend either returning the funds to the caller, or minting the leftover amount to them.  

# [M-2] Uniswap swap may be forced to revert and mint tokens unfavourably to the user

## Lines of code

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L202

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L263-L275

## Vulnerability details

There are two possible options when depositing ETH into payment terminal: minting or swapping on Uniswap. If the expected rate in Uniswap is better than minting and the user preffers claimed tokens, by passing `preferClaimedTokens` flag set to true, the [didPay(...)](https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L183C20-L208) function will first try to swap the tokens:

```solidity
    function didPay(JBDidPayData calldata _data) external payable override {
        ...
        // Pick the appropriate pathway (swap vs mint), use mint if non-claimed prefered
        if (_data.preferClaimedTokens) {
            // Try swapping
            uint256 _amountReceived = _swap(_data, _minimumReceivedFromSwap, _reservedRate);

            // If swap failed, mint instead, with the original weight + add to balance the token in
            if (_amountReceived == 0) _mint(_data, _tokenCount);
        } else {
            _mint(_data, _tokenCount);
        }
    }
```

And if it fails, then error is caugth and 0 amount is returned, and the logic changes to minting.

```solidity
    function _swap(JBDidPayData calldata _data, uint256 _minimumReceivedFromSwap, uint256 _reservedRate)
        internal
        returns (uint256 _amountReceived)
    {
        // Pass the token and min amount to receive as extra data
        try pool.swap({
            recipient: address(this),
            zeroForOne: !_projectTokenIsZero, 
            amountSpecified: int256(_data.amount.value), 
            sqrtPriceLimitX96: _projectTokenIsZero ? TickMath.MAX_SQRT_RATIO - 1 : TickMath.MIN_SQRT_RATIO + 1,
            data: abi.encode(_minimumReceivedFromSwap) 
        }) returns (int256 amount0, int256 amount1) {
            // Swap succeded, take note of the amount of projectToken received (negative as it is an exact input)
            _amountReceived = uint256(-(_projectTokenIsZero ? amount0 : amount1));
        } catch {
            // implies _amountReceived = 0 -> will later mint when back in didPay
            return _amountReceived;
        }
...
```

This design poses a risk of users receiving bad exchange rate they didn't expect, even though they choose option to swap tokens, because at the time of sending the transaction, they received favourable exchange rate from the frontend, that is unavailable at the time of transaction execution

## Impact

User lossing money through unfavourable trade without their consent

## Proof of Concept

Imagine situation that the UniswapV3 pool is being charged with JBTokens with a exchange rate that is way better than minting. Users rush to execute the PaymentTerminal to get JBTokens at discounted price, but due to demand the normal minting path is taken, even though they were shown good rate when putting the transaction, and the transaction goes through the unfavourable path. Generally there are two parties that are beneficiaries of this:

1. Project owner - they are receiving more funding for their project, as users may decide to buy tokens with discount
2. MEV bot - by sandwitching the transactions and making them go through the minting path, they are able to buy off JBTokens at good price

And the user of the protocol is loosing, as the rate at which they were supposed is not available to them without their consent. Let's go through an exemplary situations:

### Situation 1

1. Tx is submitted
2. Gas price on the network rises significantly before tx is mined, so tx is not picked up by the validator
3. projectToken price increases in the pool (user would now get less projectToken for ETH, but still more than for minting)
5. Gas price on the network comes back to its original value.
6. Since the value of _minimumAmountReceived was calculated based on an outdated projectToken price which is now lower than the current price, the swapping reverts on the condition as _minimumAmountReceived doesn't hold.
7. As a fail-safe the minting function is used. User gets less projectTokens than they could've received by selling ETH at the current price incurring a loss.

### Situation 2

1. Project owner adds JBTokens to the UniswapV3 pool
2. Users stimulated by getting good exchange rate are putting their trasnactions to support the project and earn some premium on the tokens in the process
3. Because of huge demand, the pool liquidity falls short, reverting all the trades, because _minimumAmountReceived doesn't hold
4. Users are getting worst possible exchange rate possible, using mint function without their consent.

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Consider failing a transaction when the swap fails. Under normal circumstances it should not happen, and it does protect user from getting griefed, as there will no longer be incentive to force revert on the trade. And if the price is stale, it 's probably best option for a user to submit it again with new 


# [M-3] Partial loss of user value as a result of lack of transaction deadline implementation

## Lines of code

Pool definition:

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L85

Swap call:

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L263-L268

Callback call:

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L216-L233

## Vulnerability detail

JBXBuybackDelegate provides an extention to the flow of user contribution to the project by offering a built-in buyback functionality. This feature is implemented though a direct interaction with UniswapV3's Pool contract. 

UniswapV3 implements a [deadline check in its Router contract](https://github.com/Uniswap/v3-periphery/blob/6cce88e63e176af1ddb6cc56e029110289622317/contracts/base/PeripheryValidation.sol#L7-L10) to prevent executing trades at outdated prices, in case a swap transaction gets stuck in the mempool. This can happen if gas fee provided to the transaction is too low compared to the current average gas price on the network, making it not economically reasonable to be picked up by the validators/miners.

UniswapV3's Pool contract does not implement any further deadline checks and neither does JBXBuybackDelegate in any part of the transaction flow.

To include a valid issue from a previous contest on Sherlock as a reference, I'll add this resource: https://github.com/sherlock-audit/2023-01-derby-judging/issues/323.

## Impact

Executing a buyback via JBXBuybackDelegate contract can be exploited by a sandwich attack due to lack of transaction deadline implementation. A contributor executing the transaction can receive less tokens that they would if tokens were swapped at current market price.

## Proof of Concept

1. Transaction is submitted
2. Gas price on the network rises significantly before transaction is mined, so the transaction is not picked up by the validator
3. `projectToken` price decreases in the pool (user would now get more `projectToken` for ETH)
4. Gas price on the network comes back to its original value.
5. Since the value of `_minimumAmountReceived` was calculated based on an outdated `projectToken` price which is now higher than the current price, the swap is sandwiched by a MEV bot. The bot increases the price of `projectToken` in a Uniswap pool so than the minimum output amount check still holds and earns a profit from the swapping happening at a higher price.
6. As a result of the sandwich attack, ETH is swapped at an outdated price, which is higher than the current price of the tokens. User gets less `projectToken`s than they could've received by selling ETH at the current price, incurring a partial loss.

## Tools Used

Manual review

## Recommended Mitigation Steps

Implement a deadline check within the transaction flow including a swap.

# [L-01]: Unnecessary computation when `_data.preferClaimedTokens=false`

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L195-L206

Line number from `195-197` always getting executed, even when those variables are not needed in subsequent call when `_data.preferClaimedTokens=false`.

If line number `195-197` moved inside if block at line 200, we can save some gas for the call, by avoiding computation when `_data.preferClaimedTokens = false`

```diff
diff --git a/JBXBuybackDelegate_orig.sol b/JBXBuybackDelegate_mod.sol
index 0ee751b..44717ec 100644
--- a/JBXBuybackDelegate_orig.sol
+++ b/JBXBuybackDelegate_mod.sol
@@ -192,12 +192,12 @@ contract JBXBuybackDelegate is IJBFundingCycleDataSource, IJBPayDelegate, IUnisw
         uint256 _reservedRate = reservedRate;
         reservedRate = 1;
 
-        // The minimum amount of token received if swapping
-        (,, uint256 _quote, uint256 _slippage) = abi.decode(_data.metadata, (bytes32, bytes32, uint256, uint256));
-        uint256 _minimumReceivedFromSwap = _quote - (_quote * _slippage / SLIPPAGE_DENOMINATOR);
-
         // Pick the appropriate pathway (swap vs mint), use mint if non-claimed prefered
         if (_data.preferClaimedTokens) {
+            // The minimum amount of token received if swapping
+            (,, uint256 _quote, uint256 _slippage) = abi.decode(_data.metadata, (bytes32, bytes32, uint256, uint256));
+            uint256 _minimumReceivedFromSwap = _quote - (_quote * _slippage / SLIPPAGE_DENOMINATOR);
+
             // Try swapping
             uint256 _amountReceived = _swap(_data, _minimumReceivedFromSwap, _reservedRate);
```

# [L-02] No sanity checks in the constructor can lead to issues or redeployments

## Code snippet

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L118-L129

## Vulnerability description

In the constructor no checks are done for the addresses the contract is setting (like the project token, the pool...). Even if the deployer is trusted, it is normal to have typos or man-made mistakes and those mistakes here could mean a bricked contract or the need to redeploy it.

The constructor should perform some validations to the addresses sent or at least, check they are not address zero.

## Mitigation
Include some argument validation in the constructor.

# [L-03] Inconsistency between documentation and code around ERC20 compliance

### Code snippet

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L119

### Description

In the [Juicebox documentation](https://docs.juicebox.money/dev/learn/glossary/tokens/) it states:

> By default, the protocol allocates tokens to recipients using an internal accounting mechanism in [JBTokenStore](https://docs.juicebox.money/dev/api/contracts/jbtokenstore/). These are fungible but do not conform to the ERC-20 standard, and as such cannot be composed with ecosystem ERC-20/ERC-721 marketplaces like AMMs and Opensea. Their balances can be used for voting on various platforms.

> Projects can issue their own ERC-20 token directly from the protocol to use as its token. Projects can also bring their own token as long as it conforms to the [IJBToken](https://docs.juicebox.money/dev/api/interfaces/ijbtoken/) interface and uses 18 decimal fixed point accounting. This makes it possible to use ERC-1155's or custom tokens.

This suggest that the tokens used by the project do not need to conform to ERC20 standard. In JBXBuybackDelegate however `projectToken` is casted as `IERC20`:

```solidity
constructor(
    IERC20 _projectToken,
```

# [L-04] Address of tokens in pool can be different than the ones passed to the constructor.

## Code snippet

https://github.com/code-423n4/2023-05-juicebox/blob/9d0458282511ff269b3b35b5b082b56d5cc08663/juice-buyback/contracts/JBXBuybackDelegate.sol#L118-L129

## Vulnerability description

In constructor, there is no implementation to ensure that the `projectToken` and `weth` addresses are the same as the addresses of the tokens swapped in `pool`. This can lead lack of synchronisation between pool and tokens, forcing the contract to be redeployed.

## Mitigation

Get token addresses from a trusted source or verify token addresses passed in constructor against the ones stored in a pool contract.

# [L-05] Unnecessary calculation of `_nonReservedTokenInContract`

## Code Snippet

https://github.com/code-423n4/2023-05-juicebox/blob/main/juice-buyback/contracts/JBXBuybackDelegate.sol#L312

## Description

The following line is unnecesary to calculate because it's already been calculated before so we can get the older variable.

```solidity
uint256 _nonReservedTokenInContract = _amountReceived - _reservedToken;
```

Always `_nonReservedTokenInContract` will be equal to `_nonReservedToken`, calculated here:

```solidity
// The amount to send to the beneficiary
uint256 _nonReservedToken = PRBMath.mulDiv(
    _amountReceived, JBConstants.MAX_RESERVED_RATE - _reservedRate, JBConstants.MAX_RESERVED_RATE
);

// The amount to add to the reserved token
uint256 _reservedToken = _amountReceived - _nonReservedToken;
```
