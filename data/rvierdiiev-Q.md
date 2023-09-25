## QA-01. VotiumStrategyCore.withdrawStuckTokens should not allow to withdraw cvx token.
## Description
https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L215-L220
VotiumStrategyCore.withdrawStuckTokens function can be used by malicious owner to withdraw cvx tokens from contract.
## Recommendation
If token is cvx, then revert.

## QA-02. AfEth.setRatio doesn't validate input.
## Description
AfEth.setRatio doesn't validate input. Because of that it's possible that owner will provided incorrect ratio and contract will stop working.
## Recommendation
Check that ratio is between 0 and e18.