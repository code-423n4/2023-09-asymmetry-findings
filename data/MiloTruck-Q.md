## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01-withdraw-in-afethsol-can-be-called-with-the-same-_withdrawid-multiple-times) | `withdraw()` in `AfEth.sol` can be called with the same `_withdrawId` multiple times | Low |
| [L-02](#l-02-depositrewards-might-make-the-safethvafeth-allocation-even-further-from-ratio) | `depositRewards()` might make the safEth:vAfEth allocation even further from `ratio` | Low |
| [L-03](#l-03-not-having-a-backup-price-oracle-could-dos-depositrewards) | Not having a backup price oracle could DOS `depositRewards()` | Low |
| [L-04](#l-04-freshness-threshold-should-not-be-a-hardcoded-value) | Freshness threshold should not be a hardcoded value | Low |
| [L-05](#l-05-users-who-hold-vafeth-instead-of-afeth-might-lose-out-on-rewards-unfairly) | Users who hold vAfEth instead of afETh might lose out on rewards unfairly | Low |


## [L-01] `withdraw()` in `AfEth.sol` can be called with the same `_withdrawId` multiple times

In `AfEth.sol`, withdrawals are a two-step process. [`requestWithdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L171-L215) is called first to initiate the withdrawal process, which calculates how much safEth and CVX should be withdrawn based on the amount of afEth the user specified and assigns a `_withdrawId`.

After a period of time, the user can then call `withdraw()` with their `_withdrawId` to withdraw their funds:

[AfEth.sol#L243-L250](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L243-L250)

```solidity
    function withdraw(
        uint256 _withdrawId,
        uint256 _minout
    ) external virtual onlyWithdrawIdOwner(_withdrawId) {
        if (pauseWithdraw) revert Paused();
        uint256 ethBalanceBefore = address(this).balance;
        WithdrawInfo memory withdrawInfo = withdrawIdInfo[_withdrawId];
        if (!canWithdraw(_withdrawId)) revert CanNotWithdraw();
```

`withdraw()` checks if the `_withdrawId` given is ready for withdrawal using `canWithdraw()`: 

[VotiumStrategy.sol#L155-L163](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L155-L163)

```solidity
    function canWithdraw(
        uint256 _withdrawId
    ) external view virtual override returns (bool) {
        uint256 currentEpoch = ILockedCvx(VLCVX_ADDRESS).findEpochId(
            block.timestamp
        );
        return
            withdrawIdToWithdrawRequestInfo[_withdrawId].epoch <= currentEpoch;
    }
```

`canWithdraw()` only checks if a sufficient amount of time has passed since `requestWithdraw()` was called, and doesn't check if `withdraw()` has already been called for the given `_withdrawId`.

Therefore, `withdraw()` can be called multiple times with the same `_withdrawId`, which would allow the user to withdraw more funds than originally allocated in `requestWithdraw()`.

This is currently unexploitable as `VotiumStrategy`'s [`withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L253) function reverts when called with the same `_withdrawId` more than once.

However, if [`setStrategyAddress()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L77-L83) is ever called to swap out the `VotiumStrategy` contract with a new strategy, and the new strategy does not check `_withdrawId`, this could become exploitable.

### Recommendation

Ensure that `withdraw()` cannot be called with the same `_withdrawId` multiple times in `AfEth.sol`.

## [L-02] `depositRewards()` might make the safEth:vAfEth allocation even further from `ratio`

In `depositRewards()`, the ETH transferred to the contract is deposited into either safEth or vAfEth based on the following logic:

[AfEth.sol#L286-L292](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L286-L292)

```solidity
        uint256 totalTvl = (safEthTvl + votiumTvl);
        uint256 safEthRatio = (safEthTvl * 1e18) / totalTvl;
        if (safEthRatio < ratio) {
            ISafEth(SAF_ETH_ADDRESS).stake{value: amount}(0);
        } else {
            votiumStrategy.depositRewards{value: amount}(amount);
        }
```

As seen from above, it calculates the `safEthRatio` using safEth's TVL divided by overall TVL, and checks if `safEthRatio` is below `ratio`. 

However, such an implementation could actually bring the safEth:vAfEth ratio further away from `ratio` if the difference between `safEthRatio` and `ratio` is small, and the amount of ETH to deposit is large.

For example, if `ratio - safEthRatio = 1`, the safEth:vAfEth ratio is effectively already equal to `ratio`. However, due to the logic above, all ETH will be deposited into safEth, which would increase `safEthRatio` to become much higher than `ratio`.

### Recommendation

Consider splitting the amount of ETH to deposit between safEth and vAfEth based on `ratio`, and deposit into both of them whenever `depositRewards()` is called (similar to [`deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L143-L169)).

This would ensure that the safEth:vAfEth ratio always averages towards `ratio` whenever `depositRewards()` is called.

## [L-03] Not having a backup price oracle could DOS `depositRewards()`

The `ethPerCvx()` is used to determine the price of CVX in terms of ETH using a Chainlink price oracle. If it is called with `_validate = true`, the values returned from Chainlink's price feed are validated as such:

[VotiumStrategyCore.sol#L173-L185](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L173-L185)

```solidity
        if (
            (!_validate ||
                (cl.success == true &&
                    cl.roundId != 0 &&
                    cl.answer >= 0 &&
                    cl.updatedAt != 0 &&
                    cl.updatedAt <= block.timestamp &&
                    block.timestamp - cl.updatedAt <= 25 hours))
        ) {
            return uint256(cl.answer);
        } else {
            revert ChainlinkFailed();
        }
```

As seen from above, if Chainlink's oracle returns a stale price, `ethPerCvx()` will simply revert as it does not have a backup price oracle.

This could DOS the `depositRewards()` function, which calls `ethPerCvx(true)`:

[AfEth.sol#L283-L284](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L283-L284)

```solidity
        uint256 votiumTvl = ((votiumStrategy.cvxPerVotium() *
            votiumStrategy.ethPerCvx(true)) *
```

Therefore, if Chainlink's price oracle ever returns stale prices, `depositRewards()` will be DOSed as `ethPerCvx()` will always revert.

### Recommendation

Consider implementing a secondary price oracle that can be queried for prices in `ethPerCvx()` should Chainlink's oracle go down or become stale. 

This ensures that `depositRewards()` can still be called even when Chainlink's oracles are down. 

## [L-04] Freshness threshold should not be a hardcoded value

The `ethPerCvx()` function checks if the price returned from Chainlink's oracle is stale as follows:

[VotiumStrategyCore.sol#L180](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L180)

```solidity
                    block.timestamp - cl.updatedAt <= 25 hours))
```

Where:
- `cl.updatedAt` is the value of `updatedAt` returned from `latestRoundData()`.

As seen from above, it checks if more than 25 hours has passed since `cl.updatedAt`, and reverts if so. This works for the current price oracle, which has a [heartbeat of 24 hours](https://data.chain.link/ethereum/mainnet/crypto-eth/cvx-eth).

However, it is possible for the oracle's heartbeat to be updated to a different time period (eg. 3 hours), or if `setChainlinkCvxEthFeed()` is called to update the oracle to a new one that has a different heartbeat:

[VotiumStrategyCore.sol#L76-L80](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L76-L80)

```solidity
    function setChainlinkCvxEthFeed(
        address _cvxEthFeedAddress
    ) public onlyOwner {
        chainlinkCvxEthFeed = AggregatorV3Interface(_cvxEthFeedAddress);
    }
```

If the oracle's heartbeat becomes shorter, such as 3 hours, checking for staleness with a freshness threshold of 25 hours is essentially useless, since the oracle prices will already be stale if more than 3 hours have passed.

On the other hand, if the oracle's heartbeat becomes longer, such as 48 hours, a freshness threshold of 25 hours would incorrectly DOS the `ethPerCvx()` function since it will revert even when the price isn't stale.

Therefore, having a hardcoded freshness threshold of 25 hours is problematic as it cannot be updated should the oracle's heartbeat change.

### Recommendation

Consider checking for stale prices using a `FRESHNESS_THRESHOLD` state variable, which can be modified by the contract's owner:

[VotiumStrategyCore.sol#L180](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L180)

```diff
-                   block.timestamp - cl.updatedAt <= 25 hours))
+                   block.timestamp - cl.updatedAt <= FRESHNESS_THRESHOLD))
```


## [L-05] Users who hold vAfEth instead of afETh might lose out on rewards unfairly 

In the `applyRewards()` function, the ETH gained by selling rewards from Votium is deposited into `depositRewards()` of the `AfEth` contract:

[VotiumStrategyCore.sol#L302-L304](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L302-L304)

```solidity
        if (address(manager) != address(0))
            IAfEth(manager).depositRewards{value: ethReceived}(ethReceived);
        else depositRewards(ethReceived);
```

`depositRewards()` in the `AfEth` contract determines deposits the ETH into either safEth or back into the `VotiumStrategy` contract based on the current `safEthRatio`:

[AfEth.sol#L286-L292](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L286-L292)

```solidity
        uint256 totalTvl = (safEthTvl + votiumTvl);
        uint256 safEthRatio = (safEthTvl * 1e18) / totalTvl;
        if (safEthRatio < ratio) {
            ISafEth(SAF_ETH_ADDRESS).stake{value: amount}(0);
        } else {
            votiumStrategy.depositRewards{value: amount}(amount);
        }
```

However, if the ETH gained from rewards are deposited into safEth, users that are holding vAfEth will not accrue any rewards. This is because vAfEth's TVL is only affected by the protocol's amount of CVX, and not the protocol's safEth amount, which means that the price of vAfEth will not change if rewards are deposited into safEth.

Therefore, if `applyRewards()` deposits rewards into safEth, only afEth holders will accrue rewards, leading to an unfair loss of yield for users that hold vAfEth.

Note that it is possible for users to hold vAfEth by calling the [`deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L35-L46) function of the `VotiumStrategy` contract directly, instead of depositing into the `AfEth` contract.

### Recommendation

In `applyRewards()`, consider depositing rewards to only the `VotiumStrategy` contract:

[VotiumStrategyCore.sol#L302-L304](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L302-L304)

```diff
-       if (address(manager) != address(0))
-           IAfEth(manager).depositRewards{value: ethReceived}(ethReceived);
-       else depositRewards(ethReceived);
+       depositReward(ethReceived);
```