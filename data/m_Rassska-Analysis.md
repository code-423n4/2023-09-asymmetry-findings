# Analysis - Asymmetry AfETH

## Summary
> [Asymmetry Finance](https://www.asymmetry.finance/) provides an opportunity for stakers to diversify their staked eth across many liquid staking derivatives. It's not a doubt that the Lido has about 80% of the liquid staking market and Asymmetry Finance introduces a great solution to make the LSM more decentralized.


- The codebase provided for this audit introduces new mechanism under AfEth, which is by design collateralized by 2 underlying "strategy tokens" in an adjustable ratio:
  - safETH, which is the token over the LSD market (has been audited in previous contest)
  - vAfEth, which utilizes the rewards by locking cvx tokens and delegating the locked positions to the Votium in order for the Votium to decide the gauges to vote for.

</br>

## Approach being taken while auditing
- Understanding the codebase through the test coverage and provided docs
- Debugging (line by line) test cases that are pretty crucial in a system
- Generating Invariants with an intention to put the system into unexpected state
- Receiving the context needed from sponsors in order to avoid the future problems
- Testing locally && Reporting

</br>

## Architectural high-level overview
### Current flow used for deposits
* if the image has not being rendered, see it [here](https://github.com/microsoft/vscode/assets/73281386/743d434f-7cb2-4a4c-9181-91cc2ff28d36)
  
![Deposit](https://github.com/microsoft/vscode/assets/73281386/743d434f-7cb2-4a4c-9181-91cc2ff28d36)


### Current flow used for withdrawals
* if the image has not being rendered, see it [here](https://github.com/microsoft/vscode/assets/73281386/b397da6f-b7d7-410a-a40f-8a8d378cfe79)
  
![Withdrawals](https://github.com/microsoft/vscode/assets/73281386/b397da6f-b7d7-410a-a40f-8a8d378cfe79)


</br>

## Technical overview
### Current flow used for deposits
* Description:
  * Any user has an ability to deposit eth into AfETH manager. In response, AfEth mints corresponding amount of shares representing the stake being made by user. AfEth shares themselves are minted above the shares minted by underlying strategies based on the ratio. The initial ratio has to be setted at 7e17, meaning 70% of deposited eth will be staked in safETH and only 30% converts into VotiumStrategy shares. 

* Potential Improvements: 
  * The current flow with ratio mechanism included looks flexible enough to expand over new strategies in the future. However, the manual adjusment of the ratio seems not to be comfortable to deal with, since every single strategy also has its own deposit limits that could be changed in any time, especially when new strategies will be introduced. 

### Current flow used for withdrawals
* Description:
  * Any user has an ability to withdraw eth from AfETH manager. In order to do that, the user first has to request withdrawal by providing his AfEth shares and second - withdraw the finalized request. In order to finalize the request, the system has to unstake the shares in each strategy currently being used in a system. Since, some strategies do not provide an instant unlock, the manager also has to wait that finalization period, before providing eth back to the user.

* Potential Improvements: 
  * The withdrawal queue being used by Lido to process withdrawals has a lot of interesting ideas to consider. I think, the current architecture behind afEth withdrawals could be improved, although it's decent enough. 

</br>



## Invariants Generated
* The following list includes some manually generated invariants failed, which have been used during an audit to report the issues:
  * `afETH should return the correct withdrawTimeBefore, when _amount in afETH has been provided` 
  * `chainlink oracle should return the fresh rates for the pair`
  * `CVX/ETH` pool manipulation should not provide any oppotunities for an adversory to exploit the system
  * `applyRewards()` has to be called right after claiming the votium rewards
  * `requestWithdraw()` should check whether the current balance can cover the request being submitted
  * upon requesting withdrawal, the system has to freeze it's balance to prevent double accounting. 

* Ofc, the list could contain some unsuccessfull scenarios being generated(>100), but at this point, it doesn't make any sense. 

</br>

## Centralization risks
* The AfEth manager itself is upgradable, which provides an ability for owner to update the logic behind the manager. 
* The percentage of the rewards could be setted up to 100%
* The withdrawals could be paused at any time by an owner.
* Many more, but users also should consider that the current configs setted by Asymmetry could also save the funds in case of any exploit and etc...
  
## Time Spent
* 70 hours, 7 days, 10 hours each!!!

## Additional notes
* Special thanks to the team behind Asymmetry for providing the context needed to understand certain conditions better. I've had the pleasure of working with you over this period!

### Time spent:
70 hours