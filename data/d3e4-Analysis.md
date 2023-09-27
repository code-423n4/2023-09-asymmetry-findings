## Centralisation risks
Deposits and withdrawals to AfEth can be independently paused by the owner. Especially pausing withdrawals could be leveraged to harm users.
The protocol fee can be set up to 100 % by the owner, which combined with paused withdrawals effectively steals the users stake.
The owner could also just steal all CVX, or any other reward tokens, from VotiumStrategy by using `withdrawStuckTokens()`.
The rewarder can also exploit `applyRewards()` to give anyone unlimited allowance on any of VotiumStrategy's tokens.

## Ratio mechanism
The `ratio` can be freely set by the owner (even to > 100 % such that it causes deposits to revert). The ratio between the held safEth and vAfEth is adjusted by depositing in the set `ratio`. Withdrawals are done in proportion to whatever the balances are. Only the rewards adjusts the balances actively towards the intended `ratio` by depositing only on the side where there is a shortage. The rewards are relatively small compared to the balances however. This means that the balance ratio will converge approximately exponentially but quite slowly. Several times the balances have to be deposited and withdrawn in order to reach close to the new ratio.

## Tokenisation of underlying assets
Several issues were found related to how AfEth exchanges its afEth for its underlying safEth and vAfEth. A complicating factor is that unlike a vault with only one asset, this one has two. That means that afEth cannot simply represent a direct share of its underlying, but there must be an intermediary valuation by which they can be measured together. Here their estimated values in ETH is used. An issue arises from this when the estimated value is not the same as the effective value when actually deposited or withdrawn.

### Time spent:
40 hours