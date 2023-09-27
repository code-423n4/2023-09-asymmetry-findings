# Overall Analysis
* ## About the Protocol
  > [Asymmetry Finance](https://www.asymmetry.finance/) provides an opportunity for stakers to diversify their staked eth across many liquid staking derivatives. It's not a doubt that the Lido has about 80% of the liquid staking market and Asymmetry Finance introduces a great solution to make the LSM more decentralized.

</br>

* ## Low Severity Issues
  * **[[L-01] Define the tolerance range for `ratio`](#l-01-define-the-tolerance-range-for-ratio)**

</br>

* ## Non-Critical Severity Issues
  * **[[N-01] Missing events during significant state var changes](#n-01-missing-events-during-significant-state-vars-changes)**
  * **[[N-02] Remove unused imports](#n-02-remove-unused-imports)**

</br>

## **[L-01] Define the tolerance range for `ratio`**<a name="L-01"></a>

- During the afETH initialization, the `ratio` could be setted to any number. Instead, validate it before setting. Make sure, safETH.minAmount doesn't backfire, when users start to deposit.

### ***Example of an occurance:***

- Could be implied [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L72-L75):


</br>

## **[N-01] Missing events during significant state vars changes**<a name="N-01"></a>

- It's pretty crucial to emit the events upon state var changes. It helps for backend services to better handle new conditions. 

### ***Example of an occurance:***

- Could be added, for instances, [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L81-L126):


</br>

## **[N-02] Remove unused imports**<a name="N-02"></a>

### ***Example of an occurance:***

- Could be removed, for instance, [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L6):


</br>