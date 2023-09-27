
## Use try-catch instead of retrieving check-value
In `VotiumStrategy.relock()``
```solidity
(, uint256 unlockable, , ) = ILockedCvx(VLCVX_ADDRESS).lockedBalances(
    address(this)
);
if (unlockable > 0)
    ILockedCvx(VLCVX_ADDRESS).processExpiredLocks(false);
```
`unlockable` is retrieved simply to avoid calling `processExpiredLocks`, which [reverts if there are no expired locks](https://etherscan.io/address/0x72a19342e8f1838460ebfccef09f6585e32db86e#code#L1625).
[`lockedBalances()` is not a trival function](https://etherscan.io/address/0x72a19342e8f1838460ebfccef09f6585e32db86e#code#L1388) and can be avoided here by just putting `processExpiredLocks()` in a try-catch instead.

## Use `VotiumStrategy.withdrawTime()` in `VotiumStrategy.requestWithdraw()` instead of identical inline code
[`VotiumStrategy.requestWithdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L54-L103) performs the exact same calculation as [`VotiumStrategy.withdrawTime()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L170-L195) to find the epoch at which there is enough to unlock this position. `withdrawTime()` can be reused in `requestWithdraw()` for finding the `lockedBalances[i].unlockTime`. The other calculations do not have to be performed in the loop, and `withdrawTime()` has the same revert condition.
Note that `cvxUnlockObligations += cvxAmount;` would have to be performed after `withdrawTime()`.

## Refactor if-statement
In [`VotiumStrategy.relock()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L135-L149)
```solidity
uint256 cvxAmountToRelock = cvxBalance > cvxUnlockObligations
    ? cvxBalance - cvxUnlockObligations
    : 0;
if (cvxAmountToRelock > 0) {
    IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmountToRelock);
    ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmountToRelock, 0);
}
```
can be refactored into
```solidity
if (cvxBalance > cvxUnlockObligations) {
    uint256 cvxAmountToRelock = cvxBalance - cvxUnlockObligations;
    IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmountToRelock);
    ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmountToRelock, 0);
}
```

## `ratio`, then remainder
[`uint256 vValue = (amount * (1e18 - ratio)) / 1e18;`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L160), can be calculated as
`uint256 vValue = amount - sValue;`, where
[`uint256 sValue = (amount * ratio) / 1e18;`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L156).
This is also prevents dust loss.