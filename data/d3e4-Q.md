## [L-01] Dust loss
Dust may be lost in `AfEth.deposit()` by calculating
[`uint256 sValue = (amount * ratio) / 1e18;`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L156)
and then
[`uint256 vValue = (amount * (1e18 - ratio)) / 1e18;`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L160).
The second should instead be calculated as
`uint256 vValue = amount - sValue;`.
This is also cheaper.

## [L-02] Division before multiplication
[`AfEth.price()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L133-L141) loses some precision in the form of `(a/k + b/k)*k <= a + b`. Refactor:
```solidity
function price() public view returns (uint256) {
    if (totalSupply() == 0) return 1e18;
    AbstractStrategy vEthStrategy = AbstractStrategy(vEthAddress);
    uint256 safEthValueInEth = (ISafEth(SAF_ETH_ADDRESS).approxPrice(true) *
-        safEthBalanceMinusPending()) / 1e18;
+        safEthBalanceMinusPending());
    uint256 vEthValueInEth = (vEthStrategy.price() *
-        vEthStrategy.balanceOf(address(this))) / 1e18;
+        vEthStrategy.balanceOf(address(this)));
-    return ((vEthValueInEth + safEthValueInEth) * 1e18) / totalSupply();
+    return (vEthValueInEth + safEthValueInEth) / totalSupply();
}
```

## [L-03] Missing zero-address check for `manager` in `VotiumStrategyCore.initialize()`
[`manager` is initialised](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategyCore.sol#L113) without any check. Unlike for other initialised values there is no function to later change/set `manager`.

## [L-04] Unsafe `setRatio()`
[`ratio > 1e18` breaks `deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L160). [`AfEth.setRatio(_newRatio)`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L90-L92) should check that `_newRatio <= 1e18`.

## [L-05] `AfEth.canWithdraw()` does not consider `pauseWithdraw`
[`AfEth.canWithdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L222-L227) is inconsistent with `pauseWithdraw` which is [checked in `AfEth.withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L247). This is the only place where it is internally used, so it may just be inlined instead. If this function still needs external visibility it should be amended to also check `pauseWithdraw` itself.

## [L-06] `VotiumStrategy.canWithdraw()` returns `true` for non-existent withdraw requests
[`VotiumStrategy.canWithdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L155-L163) should return `false` when `withdrawIdToWithdrawRequestInfo[_withdrawId].epoch == 0` which means that there is no withdraw request with that id.

## [L-07] VotiumStrategy deposits and withdrawals are not slippage protected
Rather slippage is [handled by `AfEth.deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L167) and [by `AfEth.withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L261). `VotiumStrategy()` can be deposited into and withdrawn from directly. If this is not the intention consider preventing user mistakes by restricting calls to `AfEth`, or, if `VotiumStrategy()` ever is to be accessed directly, add a slippage protection.

## [L-08] `depositRewards(_amount)` do not check that `_amount == msg.value`
This goes for both `AfEth.depositRewards()` and `VotiumStrategyCore.depositRewards()`. Any discrepancy would be lost.
`AfEth.depositRewards()` is called only [by `VotiumStrategyCore.applyRewards()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategyCore.sol#L303) (and may therefore be changed to `external`), whereas `VotiumStrategyCore.depositRewards()` is called either by `VotiumStrategyCore.applyRewards()` or by `AfEth.depositRewards()`.
`VotiumStrategyCore.applyRewards()` is not `payable` but obtains ether by itself. `VotiumStrategyCore.depositRewards()` can therefore not check `msg.value` in the case it is used internally. It can be checked when called by `AfEth.depositRewards()`, and `AfEth.depositRewards()` can be checked in any case.
Consider checking that `_amount == msg.value` when possible or restricting the call flow. Alternatively, use `msg.value` or `address(this).balance`, whichever is applicable.

## [L-09] No way to retrieve stuck ether
`VotiumStrategyCore` has a [`withdrawStuckTokens()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategyCore.sol#L215-L220), but no function to unstuck ether. Nor does `AfEth`. Both have `receive() external payable {}`. Consider implementing functions to retrieve stuck ether. 

## [N-01] Comment errors
The comment [`Sets the target ratio of safEth to votium.`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L86) is misleading. The ratio is not "safEth to votium" i.e. safEth/votium, but safEth to total balances, i.e. safEth/(safEth + votium).

`VotiumStrategy.requestWithdraw()` does not burn afEth tokens as stated in a [notice](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L50). It [burns its own vAfEth tokens](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L60). The afEth tokens are [burned in `AfEth.withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L255).

`cvxPerVotium` returns the price of CVX per vAfEth, not per afEth [as stated in the comments](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategyCore.sol#L141-L142).

## [N-02] Typos
[transfering -> transferring](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L181)
[equivilent -> equivalent](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L50)