---
sponsor: "Asymmetry Finance"
slug: "2023-09-asymmetry"
date: "2024-02-06"
title: "Asymmetry Finance afETH Invitational"
findings: "https://github.com/code-423n4/2023-09-asymmetry-findings/issues"
contest: 286
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Asymmetry Finance afETH smart contract system written in Solidity. The audit took place between September 20—September 27 2023.

Following the C4 audit, 3 wardens ([adriro](https://code4rena.com/@adriro), [d3e4](https://code4rena.com/@d3e4), and [m\_Rassska](https://code4rena.com/@m_Rassska)) reviewed the mitigations for all identified issues; the [mitigation review report](#mitigation-review) is appended below the audit report.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 5 wardens contributed reports:

  1. [adriro](https://code4rena.com/@adriro)
  2. [d3e4](https://code4rena.com/@d3e4)
  3. [MiloTruck](https://code4rena.com/@MiloTruck)
  4. [m\_Rassska](https://code4rena.com/@m_Rassska)
  5. [rvierdiiev](https://code4rena.com/@rvierdiiev)

This audit was judged by [0xLeastwood](https://code4rena.com/@leastwood).

Final report assembled by PaperParachute.

# Summary

The C4 analysis yielded an aggregated total of 15 unique vulnerabilities. Of these vulnerabilities, 5 received a risk rating in the category of HIGH severity and 10 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 5 reports detailing issues with a risk rating of LOW severity or non-critical. There were also 4 reports recommending gas optimizations.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Asymmetry Finance afETH  repository](https://github.com/code-423n4/2023-09-asymmetry), and is composed of 4 smart contracts written in the Solidity programming language and includes 773 lines of Solidity code.

In addition to the known issues identified by the project team, an [Automated Findings report](https://gist.github.com/romeroadrian/a2045e828aa87418a66e9ab47d811292) was generated using the [4naly3er bot](https://github.com/Picodes/4naly3er) and all findings therein were classified as out of scope.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (5)
## [[H-01] Intrinsic arbitrage from price discrepancy](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62)
*Submitted by [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62)*

The up to 2 % price discrepancy from Chainlink creates an intrinsic arbitrage. Especially, it makes withdrawals worth more than deposits in the sense that one can immediately withdraw more than just deposited.

### Proof of Concept

When [depositing ETH into AfEth](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L148-L169), the ETH is split according to `ratio` and sold for safEth and vAfEth. The received share of afEth is then determined by the value in ETH of the resulting amounts of safEth and vAfEth. Note that there are two prices involved here: the true price at which ETH is traded for safEth and vAfEth (in `sellCvx()` and `buyCvx()`), and the estimated value in ETH that safEth and vAfEth is considered to have (`ISafEth.approxPrice()` and `VotiumStrategy.price()`). These are not necessarily the same.

If the ratio by which the deposited ETH is split is not the same as the ratio of the true values of the underlying assets, this implies that a deposit implicitly makes a trade between safEth and vAfEth according to the estimated price which may thus differ from the true price obtained when withdrawing. This presents an arbitrage opportunity.
Note that if all prices were the same it would not matter if safEth is "traded" for vAfEth within a deposit as the trade then makes no change in the total value deposited.

The conditions for this issue is thus that `VotiumStrategy.price()` is different from the price obtained by `sellCvx()` and `buyCvx()`, and that the deposit ratio is not the same as the withdrawal ratio.

[`VotiumStrategy.price()`](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategy.sol#L31-L33) in particular is [based on a Chainlink oracle](https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/strategies/votium/VotiumStrategyCore.sol#L156-L186) with a [2 % deviation threshold](https://data.chain.link/ethereum/mainnet/crypto-eth/cvx-eth), which means that the true price is allowed to deviate up to 2 %, within 24 hours, from the reported price. `ISafEth.approxPrice()` may perhaps be similarly inaccurate (I have not looked into this).

The ratio can skew in this way for two reasons. One is when the `ratio` is different from the ratio of the underlying balances, such as when `ratio` is changed. Deposits are made according to `ratio` but withdrawals are made proportionally from the extant balances. In this case the implicit trade between safEth and vAfEth can happen in either direction, either beneficial or detrimental to the depositor (if there is a price discrepancy).
The other is caused by the price discrepancy itself when depositing. In this case it is always beneficial to the depositor (and detrimental to the holders).

**Example 1a - reconverging ratio, vAfEth is actually more expensive**
Suppose the contract holds 100 safEth and 0 vAfEth, but that the `ratio` has now been changed to 0. Further suppose that the contract thinks all prices are 1, but that 100 ETH actually trades for 102 vAfEth.
Then depositing 100 ETH will convert it to 102 vAfEth, which will be valued as 102 ETH. This mints 102 afEth.
The balances are now 100 safEth and 102 vAfEth and the total supply is 202 afEth.
Withdrawing 102 afEth converts 102/202 of the underlying, i.e. 50.495 safEth and 51.505 vAfEth, into 50.495 + 51.505/1.02 = 100.99 ETH.

**Example 1b - reconverging ratio, vAfEth is actually cheaper**
Suppose the contract holds 100 safEth and 0 vAfEth, but that the `ratio` has now been changed to 0. Further suppose that the contract thinks all prices are 1, but that 100 ETH actually trades for 98 vAfEth.
Then depositing 100 ETH will convert it to 98 vAfEth, which will be valued as 98 ETH. This mints 98 afEth.
The balances are now 100 safEth and 98 vAfEth and the total supply is 198 afEth.
Withdrawing 98 afEth converts 98/198 of the underlying, i.e. 49.495 safEth and 48.505 vAfEth, into 49.495 + 48.505/0.98 = 98.99 ETH.

**Example 2a - stable ratio, vAfEth is actually more expensive**
Suppose the contract holds 50 safEth and 50 vAfEth and that the `ratio` is 0.5. Further suppose that the contract thinks all prices are 1 but that 50 ETH actually trades for 51 vAfEth.
Then depositing 100 ETH will convert 50 ETH to 50 safEth and 50 ETH to 51 vAfEth, which will be valued as 101 ETH. This mints 101 afEth.
The balances are now 100 safEth and 101 vAfEth and the total supply is 201 afEth.
Withdrawing 101 afEth converts 101/201 of the underlying, i.e. 50.249 safEth and 50.751 vAfEth, into 50.249 + 50.751/1.02 = 100.005 ETH.

**Example 2b - stable ratio, vAfEth is actually cheaper**
Suppose the contract holds 50 safEth and 50 vAfEth and that the `ratio` is 0.5. Further suppose that the contract thinks all prices are 1 but that 50 ETH actually trades for 49 vAfEth.
Then depositing 100 ETH will convert 50 ETH to 50 safEth and 50 ETH to 49 vAfEth, which will be valued as 99 ETH. This mints 99 afEth.
The balances are now 100 safEth and 99 vAfEth and the total supply is 199 afEth.
Withdrawing 99 afEth converts 99/199 of the underlying, i.e. 49.749 safEth and 49.251 vAfEth, into 49.749 + 49.251/0.98 = 100.005 ETH.

Thus one can make a profit by depositing and immediately withdrawing. Immediate withdrawals are possible at the moment locks expire (and before they have been relocked), but it may be enough to just immediately request a withdrawal if the true price is the same (or better) when eventually withdrawn.

The price discrepancy will appear whenever there are price fluctuations of up to 2 % within 24 hours, which seems quite likely.

Regarding the case where the underlying is reconverging after a change of `ratio` it is worth noting that convergence is quite slow. Several times the entire balances must be traded before the new ratio is approached.

### Recommended Mitigation Steps
We want a  to not diminish the value of previous deposits. That is, withdrawing $w$ shares should return at least as much if withdrawn after a deposit which mints $m$ shares as if withdrawn before.

Note that letting a share represent each underlying in equal proportions is the only way to guarantee fairness and fungibility, so we must leave the withdrawal calculation as it is.
 
Let $d_s$ and $d_v$ be the ether amounts deposited in SafEth and VotiumStrategy, respectively. Let $B_s$ and $B_v$ be the respective balances in AfEth and $T$ the total supply of afEth. Let $P_s(x)$ be the amount of safEth obtained by selling $x$ ether for safEth, and $P_s^{-1}(y)$ the amount of ether obtained by selling safEth for ether (note the abuse of notation and that $P_s^{-1}(P_s(x)) \leq x$ because of fees, spread, slippage etc.), and similarly $P_v$ and $P_v^{-1}$ for vAfEth.

Withdrawing $w$ now should return at most what it would return after a deposit of $d_s + d_v$, i.e.
 
```math
P_s^{-1}(\frac{w}{T}B_s) + P_v^{-1}(\frac{w}{T}B_v) \leq 
P_s^{-1}(\frac{w}{T+m}(B_s + P_s(d_s))) + P_v^{-1}(\frac{w}{T+m}(B_v + P_v(d_v)))
```

For small deposits and withdrawals the price functions are approximately linear and we can write e.g. $P_s(d_s)$ as $P_s d_s$, i.e. $P_s$ is just a price point, and we get
```math
\frac{1}{T}(P_s^{-1}B_s + P_v^{-1}B_v) \leq 
\frac{1}{T+m}(P_s^{-1}(B_s + P_s d_s) + P_v^{-1}(B_v + P_v d_v))$
```

Solving for $m$ we get
```math
m \leq \frac{P_s^{-1}P_s d_s + P_v^{-1}P_v d_v}{P_s^{-1}B_s + P_v^{-1}B_v}T
```

The difference from the current implementation is that instead of the true prices $P_s^{-1}$ and $P_v^{-1}$, the oracle prices, which we will denote $\hat{P}^{-1}_s$ and $\hat{P}^{-1}_v$, are used instead, i.e.
```math
m = \frac{\hat{P}^{-1}_s P_s d_s + \hat{P}^{-1}_v P_v d_v}
{\hat{P}^{-1}_s B_s + \hat{P}^{-1}_v B_v}T
```

Since we know the true prices up to within certain bounds, we can minimise $m$, as a function of $P_s^{-1}$ and $P_v^{-1}$, within these bounds.
The gradient of $m(P_s^{-1}, P_v^{-1})$ is
```math
\frac{P_s d_s B_v - P_v d_v B_s}
{(P^{-1}_s B_s + P^{-1}_v B_v)^2}
(P^{-1}_v, -P^{-1}_s)
```

so if $P_s d_s B_v - P_v d_v B_s > 0$ we pick the lower right corner (maximal $P_s^{-1}$, minimal $P_v^{-1}$), and if $P_s d_s B_v - P_v d_v B_s < 0$ we pick the upper left corner (minimal $P_s^{-1}$, maximal $P_v^{-1}$).
In the case of equality we can use any (non-zero) prices.

We minimise within the bounds, but we of course want the bounds to be as tight as possible, so that this minimum is as high as possible.

The oracle prices provide us with good bounds, namely $P_s^{-1} \in [0.98 \cdot \hat{P}^{-1}_s, 1.02 \cdot \hat{P}^{-1}_s]$ and $P_v^{-1} \in [0.98 \cdot \hat{P}^{-1}_v, 1.02 \cdot \hat{P}^{-1}_v]$.

There are a few ways to further improve the bounds. During the deposit we learn $P_s$ and $P_v$, from which we can infer $P_s^{-1} \leq 1/P_s$ and $P_v^{-1} \leq 1/P_v$ (equality in the case of no exchange losses). If we know that there is some minimum percentage lost (e.g. exchange fees, slippage, spread) we can refine these to $P_s^{-1} \leq k_s/P_s$ and $P_v^{-1} \leq k_v/P_v$, where $k_s, k_v < 1$ is some factor adjusting for these losses (e.g. $k_s = 0.99$ if there is at least (another) 1 \% lost when selling for ether).

If the trading losses are significant it may be necessary to take these into account even for the bounds from the oracle prices, such that both upper and lower bounds are slightly reduced.

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62#issuecomment-1746998418)**

**[adriro (Warden) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62#issuecomment-1752039853):**
 > @0xleastwood - I'm really sorry to do this at this stage, but did you have the chance to go through these scenarios? Seems there are lot of suppositions, some of which the same warden is confusingly invalidating in other discussions.
> 
> I think examples 1a and 1b have invalid assumptions ("the contract holds 100 safEth and 0 vAfEth, but that the ratio has now been changed to 0"). In 2a and 2b, what is "the contract thinks all prices are.."? How is the contract thinking prices?
> 
> It seems this has nothing to do with the stated impact. The chainlink response is used to price vAfEth, but the core here is a discrepancy of the ratio with the underlying assets (which again confusingly the author is trying to invalidate in other issues). Furthermore the deposit/withdraw cycle can't be executed without exposure to the underlying assets due to the locking mechanism.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62#issuecomment-1752158668):**
> @adriro - I'll look into these, I do believe it is possible to deposit and withdraw atomically as long as there is an unlocked amount of tokens in the votium strategy contract. That would be the extent at which a withdrawal could be made "instantly".
>
> I do agree that example 1a and 1b are somewhat infeasible because the protocol team has already stated that such a ratio would not exist in the first place. However, there is validity in the other examples.

**[Asymmetry commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation?tab=readme-ov-file#individual-prs):**
 > After days of research we decided that this was acceptable. See the comments in [issue 62](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62#issuecomment-1760305328) for more information.

**Status**: Unmitigated. Full details in reports from adriro ([1](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/26), [2](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45)) and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/12), and also included in the [Mitigation Review](#mitigation-review) section below.

***

## [[H-02] Zero amount withdrawals of SafEth or Votium will brick the withdraw process](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36), also found by [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/66) and [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/13)*

Withdrawals of amount zero from both SafEth and VotiumStrategy have issues downstream that will cause the transaction to revert, potentially bricking withdrawals from being executed.

### Impact

Withdrawals in AfEth undergo a process to account for any potential delay when withdrawing locked tokens in the VotiumStrategy. When a withdrawal is requested, the implementation calculates the owed amounts for each token and queues the withdrawal. SafEth tokens will be reserved in the contract, and VotiumStrategy will also queue the withdrawal of CVX tokens.

When the time arrives, the user can call `withdraw()` to execute the withdrawal. This function will unstake from SafEth and withdraw from VotiumStrategy.

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L252-L253>

```solidity
252:         ISafEth(SAF_ETH_ADDRESS).unstake(withdrawInfo.safEthWithdrawAmount, 0);
253:         AbstractStrategy(vEthAddress).withdraw(withdrawInfo.vEthWithdrawId);
```

Let's first consider the SafEth case. The current `unstake()` implementation in SafEth will revert if the unstaked amount is zero:

<https://etherscan.io/address/0x591c4abf20f61a8b0ee06a5a2d2d2337241fe970#code#F1#L124>

```solidity
119:     function unstake(
120:         uint256 _safEthAmount,
121:         uint256 _minOut
122:     ) external nonReentrant {
123:         if (pauseUnstaking) revert UnstakingPausedError();
124:         if (_safEthAmount == 0) revert AmountTooLow();
125:         if (_safEthAmount > balanceOf(msg.sender)) revert InsufficientBalance();
```

As we can see in line 124, if `_safEthAmount` is zero the function will revert, and the transaction to `withdraw()` will revert too due to the bubbled error. This means that any withdrawal that ends up with a zero amount for SafEth will be bricked.

The VotiumStrategy case has a similar issue. The implementation of `withdraw()` will call `sellCvx()` to swap the owed amount of CVX for ETH. This is executed using a Curve pool, as we can see in the following snippet of code:

```solidity
250:     function sellCvx(
251:         uint256 _cvxAmountIn
252:     ) internal returns (uint256 ethAmountOut) {
253:         address CVX_ETH_CRV_POOL_ADDRESS = 0xB576491F1E6e5E62f1d8F26062Ee822B40B0E0d4;
254:         // cvx -> eth
255:         uint256 ethBalanceBefore = address(this).balance;
256:         IERC20(CVX_ADDRESS).approve(CVX_ETH_CRV_POOL_ADDRESS, _cvxAmountIn);
257: 
258:         ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying(
259:             1,
260:             0,
261:             _cvxAmountIn,
262:             0 // this is handled at the afEth level
263:         );
264:         ethAmountOut = address(this).balance - ethBalanceBefore;
265:     }
```

If we drill down in the Curve implementation, we can see that it validates that the input amount is greater than zero:

<https://etherscan.io/address/0xB576491F1E6e5E62f1d8F26062Ee822B40B0E0d4#code#L714>

```vyper
709: def _exchange(sender: address, mvalue: uint256, i: uint256, j: uint256, dx: uint256, min_dy: uint256, use_eth: bool) -> uint256:
710:     assert not self.is_killed  # dev: the pool is killed
711:     assert i != j  # dev: coin index out of range
712:     assert i < N_COINS  # dev: coin index out of range
713:     assert j < N_COINS  # dev: coin index out of range
714:     assert dx > 0  # dev: do not exchange 0 coins
```

Again, this means that any withdrawal that ends up with a zero amount of vAfEth tokens (or the associated amount of CVX tokens) will be bricked when trying to execute the swap.

This can happen for different reasons. For example the current `ratio` may be `0` or `1e18`, meaning the split goes entirely to SafEth or to VotiumStrategy. Another reason could be rounding, for small quantities the proportion may round down values to zero.

The critical issue is that both withdrawals are executed simultaneously. A zero amount shouldn't matter, but both happen at the time, and one may affect the other. If the SafEth amount is zero, it will brick the withdrawal for a potentially non-zero vAfEth amount. Similarly, if the vAfEth amount is zero, it will brick the withdrawal for a potentially non-zero SafEth amount

### Proof of Concept

To simplify the case, let's say the current ratio is zero, meaning all goes to VotiumStrategy.

1.  A user calls `requestWithdraw()`. Since currently the SafEth ratio is zero, the contract doesn't hold a position in SafEth. This means that `safEthWithdrawAmount = 0`, and the position is entirely in vAfEth (`votiumWithdrawAmount > 0`).
2.  Time passes and the user can finally withdraw.
3.  The user calls `withdraw()`. The implementation will try to call `SafEth::unstake(0)`, which will cause an error, reverting the whole transaction.
4.  The user will never be able to call `withdraw()`. Even if the ratios are changed, the calculated amount will be already stored in the `withdrawIdInfo` mapping. The withdrawal will be bricked, causing the loss of the vAfEth tokens.

### Recommendation

For SafEth, avoid calling `SafEth::unstake()` if the calculated amount is zero:

```diff
+ if (withdrawInfo.safEthWithdrawAmount > 0) {
    ISafEth(SAF_ETH_ADDRESS).unstake(withdrawInfo.safEthWithdrawAmount, 0);
+ }
```

For VotiumStrategy, prevent requesting the withdrawal if `votiumWithdrawAmount` is zero, while also keeping track of this to also avoid executing the withdrawal when `AfEth::withdraw()` is called.

It is also recommended to add a guard in `VotiumStrategy::withdraw()` to avoid calling `sellCvx()` when `cvxWithdrawAmount = 0`.

```diff
-  uint256 ethReceived = sellCvx(cvxWithdrawAmount);
+  uint256 ethReceived = cvxWithdrawAmount > 0 ? sellCvx(cvxWithdrawAmount) : 0;
```

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36#issuecomment-1741290306)**

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36#issuecomment-1746294920):**
 > It's unclear under what circumstances, `withdrawRatio` will be zero. As it appears, `votiumWithdrawAmount` is calculated as `(withdrawRatio * votiumBalance) / 1e18` and similarly, `safEthWithdrawAmount` is calculated as `(withdrawRatio * safEthBalance) / 1e18`. So it seems the withdraw ratio is applied in the same manner to both of these amounts?
>
 > The main case where this causes issues is when `votiumBalance` is non-zero and `safEthBalance` is zero or vice-versa. I'm curious as to when this might happen @elmutt ?

**[elmutt (Asymmetry) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36#issuecomment-1746906671):**
> withdrawRatio represents the ratio of the amount being withdrawn to the total supply. So if a user owns 1% of afEth and withdraws their entire balance they will be set to receive 1% of each of the underlying assets (safEth & votiumStrategy) based on their current prices.
> 
> It should never be zero unless user is withdrawing the the last afEth from the system but we plan to solve this with an initial seed deposit

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36#issuecomment-1747441726):**
 > Okay good to know, I think I understand what you mean now. Issue appears valid and I think high severity is justified because the last staker would be unable to execute their withdrawal. 
>
 > However, can you explain why `withdrawRatio` would be zero upon the last withdrawal? It is calculated as `(_amount * 1e18) / (totalSupply() - afEthBalance)` where the denominator is equal to the `_amount`. Hence, this is equal to `1e18`. So it attempts to withdraw all votium and safEth tokens from the contract.
> 
> A better thing to understand would be, when would either of this token balances be non-zero? And your mitigation is to seed the contract with some tokens initially so the token balance is always positive?

**[elmutt (Asymmetry) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36#issuecomment-1748055065):**
 > I think we actually have a bug here. We shouldnt be subtracting afEthBalance. 
> 
> Previously we subtracted it because the afEth contract held the users afEth before finally burning it on withdraw(). Now we just burn it on requestWithdraw() so we shouldn't be subtracting anymore.
> 
> Does that make sense?

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/36#issuecomment-1748226123):**
 > Agreed, that makes sense. No need to track `afEthBalance` anymore. There might be other areas where this is being done incorrectly too.

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > Don't withdraw zero from SafEth or Votium.

**Status**: Mitigation confirmed. Full details in reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/5) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/28).

***

## [[H-03] AfEth deposits could use price data from an invalid Chainlink response](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/34)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/34), also found by [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/64), [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/28), and [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/11)*

The current price implementation for the VotiumStrategy token uses a potentially invalid Chainlink response. This price is then used to calculate the price of AfEth and, subsequently, the amount of tokens to mint while depositing.

### Impact

The price of VotiumStrategy tokens are determined by taking the amount of deposited CVX in the strategy, and multiplied by the current price of CVX in terms of ETH. This price is fetched using Chainlink in the `ethPerCvx()` function:

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L156-L186>

```solidity
156:     function ethPerCvx(bool _validate) public view returns (uint256) {
157:         ChainlinkResponse memory cl;
158:         try chainlinkCvxEthFeed.latestRoundData() returns (
159:             uint80 roundId,
160:             int256 answer,
161:             uint256 /* startedAt */,
162:             uint256 updatedAt,
163:             uint80 /* answeredInRound */
164:         ) {
165:             cl.success = true;
166:             cl.roundId = roundId;
167:             cl.answer = answer;
168:             cl.updatedAt = updatedAt;
169:         } catch {
170:             cl.success = false;
171:         }
172:         // verify chainlink response
173:         if (
174:             (!_validate ||
175:                 (cl.success == true &&
176:                     cl.roundId != 0 &&
177:                     cl.answer >= 0 &&
178:                     cl.updatedAt != 0 &&
179:                     cl.updatedAt <= block.timestamp &&
180:                     block.timestamp - cl.updatedAt <= 25 hours))
181:         ) {
182:             return uint256(cl.answer);
183:         } else {
184:             revert ChainlinkFailed();
185:         }
186:     }
```

As we can see from the previous snippet of code, if the `_validate` flag is off, then *no validation* is done, it can even return an uninitialized response from a failed call given the usage of the `try/catch` structure. This means that it can invalid price, stale price, or even zero when the call fails.

The VotiumStrategy `price()` function calls `ethPerCvx(false)`, which means it carries forward any invalid CVX/ETH price.

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L31-L33>

```solidity
31:     function price() external view override returns (uint256) {
32:         return (cvxPerVotium() * ethPerCvx(false)) / 1e18;
33:     }
```

The price of VotiumStrategy is then used in the AfEth contract to calculate its price and determine the amount of tokens to mint in `deposit()`

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L133-L169>

```solidity
133:     function price() public view returns (uint256) {
134:         if (totalSupply() == 0) return 1e18;
135:         AbstractStrategy vEthStrategy = AbstractStrategy(vEthAddress);
136:         uint256 safEthValueInEth = (ISafEth(SAF_ETH_ADDRESS).approxPrice(true) *
137:             safEthBalanceMinusPending()) / 1e18;
138:         uint256 vEthValueInEth = (vEthStrategy.price() *
139:             vEthStrategy.balanceOf(address(this))) / 1e18;
140:         return ((vEthValueInEth + safEthValueInEth) * 1e18) / totalSupply();
141:     }

148:     function deposit(uint256 _minout) external payable virtual {
149:         if (pauseDeposit) revert Paused();
150:         uint256 amount = msg.value;
151:         uint256 priceBeforeDeposit = price();
152:         uint256 totalValue;
153: 
154:         AbstractStrategy vStrategy = AbstractStrategy(vEthAddress);
155: 
156:         uint256 sValue = (amount * ratio) / 1e18;
157:         uint256 sMinted = sValue > 0
158:             ? ISafEth(SAF_ETH_ADDRESS).stake{value: sValue}(0)
159:             : 0;
160:         uint256 vValue = (amount * (1e18 - ratio)) / 1e18;
161:         uint256 vMinted = vValue > 0 ? vStrategy.deposit{value: vValue}() : 0;
162:         totalValue +=
163:             (sMinted * ISafEth(SAF_ETH_ADDRESS).approxPrice(true)) +
164:             (vMinted * vStrategy.price());
165:         if (totalValue == 0) revert FailedToDeposit();
166:         uint256 amountToMint = totalValue / priceBeforeDeposit;
167:         if (amountToMint < _minout) revert BelowMinOut();
168:         _mint(msg.sender, amountToMint);
169:     }
```

The VotiumStrategy price is first used in line 138 to calculate its TVL (`vEthValueInEth`). Any invalid price here will also mean an invalid price for AfEth.

Then both the AfEth price (line 151) and again the VotiumStrategy price (line 164) are used in `deposit()` to calculate the number of minted tokens. Depending on the direction of the wrong price, this means that the user will be minted more or less tokens than it should.

### Proof of Concept

Let's suppose the Chainlink feed is stale and the current price of CVX/ETH has increased since then.

1.  A user calls `deposit()` to create a new position in AfEth.
2.  The function calculates the current price (`priceBeforeDeposit`) in order to know how many tokens should be minted.
3.  The `price()` implementation will calculate the Votium strategy TVL using `ethPerCvx(false)`, which will successfully return the stale price.
4.  The price of AfEth will then be calculated using the old data, which will result in a lower value than the actual "real" price.
5.  The user is minted tokens based on the incorrectly calculated `priceBeforeDeposit`, since this price is lower than the expected "real" price the user will be minted more tokens than expected.

### Recommendation

Change the `ethPerCvx()` argument to `true` to make sure prices coming from Chainlink are correctly validated.

```diff
  function price() external view override returns (uint256) {
-     return (cvxPerVotium() * ethPerCvx(false)) / 1e18;
+     return (cvxPerVotium() * ethPerCvx(true)) / 1e18;
  }
```

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/34#issuecomment-1741423338):**

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/34#issuecomment-1744973191):**
 > Should we not be prioritising liveness here over validating chainlink results?
>
 > It seems important to avoid using stale price data which can be readily arbitraged. Severity seems correct.

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 >	Validate Chainlink price data.

**Status**: Mitigation confirmed. Full details in reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/6), [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/14), and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/29).

***

## [[H-04] `price()` in `AfEth.sol` doesn't take afEth held for pending withdrawals into account](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/25)
*Submitted by [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/25), also found by d3e4 ([1](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/60), [2](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/59)), [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/37), and [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/7)*

In `AfEth.sol`, the `price()` function returns the current price of afEth:

[AfEth.sol#L133-L141](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L133-L141)

```solidity
    function price() public view returns (uint256) {
        if (totalSupply() == 0) return 1e18;
        AbstractStrategy vEthStrategy = AbstractStrategy(vEthAddress);
        uint256 safEthValueInEth = (ISafEth(SAF_ETH_ADDRESS).approxPrice(true) *
            safEthBalanceMinusPending()) / 1e18;
        uint256 vEthValueInEth = (vEthStrategy.price() *
            vEthStrategy.balanceOf(address(this))) / 1e18;
        return ((vEthValueInEth + safEthValueInEth) * 1e18) / totalSupply();
    }
```

As seen from above, the price of afEth is calculated by the TVL of both safEth and vAfEth divided by `totalSupply()`. However, this calculation does not take into account afEth that is transferred to the contract when `requestWithdraw()` is called:

[AfEth.sol#L183-L187](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L183-L187)

```solidity
        uint256 afEthBalance = balanceOf(address(this));
        uint256 withdrawRatio = (_amount * 1e18) /
            (totalSupply() - afEthBalance);

        _transfer(msg.sender, address(this), _amount);
```

When a user calls `requestWithdraw()` to initiate a withdrawal, his afEth is transferred to the `AfEth` contract as shown above. Afterwards, an amount of [vAfEth proportional to his withdrawal amount is burned](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L54-L60), and [`pendingSafEthWithdraws` is increased](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L199).

When `price()` is called afterwards, `safEthBalanceMinusPending()` and `vEthStrategy.balanceOf(address(this))` will be decreased. However, since the user's afEth is only transferred and not burnt, `totalSupply()` remains the same. This causes the value returned by `price()` to be lower than what it should be, since `totalSupply()` is larger than the actual circulating supply of afEth.

This is an issue as `deposit()` relies on `price()` to determine how much afEth to mint to a depositor:

[AfEth.sol#L166-L168](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L166-L168)

```solidity
        uint256 amountToMint = totalValue / priceBeforeDeposit;
        if (amountToMint < _minout) revert BelowMinOut();
        _mint(msg.sender, amountToMint);
```

Where:

*   `totalValue` is the ETH value of the caller's deposit.
*   `priceBeforeDeposit` is the cached value of `price()`.

If anyone has initiated a withdrawal using `requestWithdraw()` but hasn't called `withdraw()` to withdraw his funds, `price()` will be lower than what it should be. Subsequently, when `deposit()` is called, the depositor will receive more afEth than he should since `priceBeforeDeposit` is smaller.

Furthermore, a first depositor can call `requestWithdraw()` with all his afEth immediately after staking to make `price()` return 0, thereby permanently DOSing all future deposits as `deposit()` will always revert with a division by zero error.

### Impact

When there are pending withdrawals, `price()` will return a value smaller than its actual value. This causes depositors to receive more afEth than intended when calling `deposit()`, resulting in a loss of funds for previous depositors.

Additionally, a first depositor can abuse this to force `deposit()` to always revert, permanently bricking the protocol forever.

### Proof of Concept

Assume that the protocol is newly deployed and Alice is the only depositor.

*   This means that Alice's afEth balance equals to `totalSupply()`.

Alice calls `requestWithdraw()` with `_amount` as all her afEth:

*   Since `_amount == totalSupply()`, `withdrawRatio` is `1e18` (100%).
*   Therefore, all of the protocol's vAfEth is burnt and `pendingSafEthWithdraws` is increased to the protocol's safEth balance.
*   Alice's afEth is transferred to the protocol.

Bob calls `deposit()` to deposit some ETH into the protocol:

*   When `price()` is called:
    *   Since `pendingSafEthWithdraws` is equal to the protocol's safEth balance, `safEthBalanceMinusPending()` is 0, therefore `safEthValueInEth` is also 0.
    *   Since `vEthStrategy.balanceOf(address(this))` (the protocol's vAfEth balance) is 0, `vEthValueInEth` is also 0.
    *   `totalSupply()` is non-zero.
    *   Therefore, `price()` returns 0 as:

```

    ((vEthValueInEth + safEthValueInEth) * 1e18) / totalSupply() = ((0 + 0) * 1e18) / x = 0
```

*   As `priceBeforeDeposit` is 0, [this line](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L166) reverts with a division by zero error.

As demonstrated above, `deposit()` will always revert as long as Alice does not call `withdraw()` to burn her afEth, thereby bricking the protocol's core functionality.

### Recommended Mitigation

In `price()`, consider subtracting the amount of afEth held in the contract from `totalSupply()`:

[AfEth.sol#L133-L141](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L133-L141)

```diff
    function price() public view returns (uint256) {
-       if (totalSupply() == 0) return 1e18;
+       uint256 circulatingSupply = totalSupply() - balanceOf(address(this));
+       if (circulatingSupply == 0) return 1e18;
        AbstractStrategy vEthStrategy = AbstractStrategy(vEthAddress);
        uint256 safEthValueInEth = (ISafEth(SAF_ETH_ADDRESS).approxPrice(true) *
            safEthBalanceMinusPending()) / 1e18;
        uint256 vEthValueInEth = (vEthStrategy.price() *
            vEthStrategy.balanceOf(address(this))) / 1e18;
-       return ((vEthValueInEth + safEthValueInEth) * 1e18) / totalSupply();
+       return ((vEthValueInEth + safEthValueInEth) * 1e18) / circulatingSupply;
    }
```

**[elmutt (Asymmetry) confirmed and commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/25#issuecomment-1741058078):**
 > @toshiSat - I think we can solve this by burning the tokens in requestWithdraw.

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > For this one we made afEth just burn on requestWithdraw.

**Status**: Mitigation confirmed. Full details in reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/7), [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/15), and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/31).

***

## [[H-05] Functions in the `VotiumStrategy` contract are susceptible to sandwich attacks](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23)
*Submitted by [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23), also found by [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/24), [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/61), adriro ([1](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/41), [2](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/39)), [m\_Rassska](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/15), and [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/8)*

In `VotiumStrategyCore.sol`, the `buyCvx()` and `sellCvx()` functions call `exchange_underlying()` of Curve's ETH / CVX pool to buy and sell CVX respectively:

[VotiumStrategyCore.sol#L233-L240](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L233-L240)

```solidity
        ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying{
            value: _ethAmountIn
        }(
            0,
            1,
            _ethAmountIn,
            0 // this is handled at the afEth level
        );
```

[VotiumStrategyCore.sol#L258-L263](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L258-L263)

```solidity
        ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying(
            1,
            0,
            _cvxAmountIn,
            0 // this is handled at the afEth level
        );
```

As seen from above, `exchange_underlying()` is called with its `_min_dy` parameter as 0, which means the minimum amount of CVX or ETH to receive from the swap is effectively 0.

This isn't an issue when users interact with the `AfEth` contract, as its [`deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L143-L169) and [`withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L238-L265) functions include a `_minOut` parameter which protects against slippage.

However, users that interact with the `VotiumStrategy` contract directly will not be protected from slippage when they call any of the following functions:

*   [`deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L35-L46), which calls `buyCvx()`
*   [`depositRewards()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L199-L208), which calls `buyCvx()`
*   [`withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L105-L129), which calls `sellCvx()`

Should users call any of the functions listed above directly, they will be susceptible to sandwich attacks by attackers, which would reduce the amount of CVX or ETH received from the swap with curve's pool.

### Impact

Due to a lack of slippage protection in `buyCvx()` and `sellCvx()`, users that interact with the `VotiumStrategy` contract will be susceptible to sandwich attacks. This results in a loss of funds for them as they will receive less CVX or ETH for the same amount of funds.

### Proof of Concept

Consider the following scenario:

*   Bob calls the `VotiumStrategy` contract's `deposit()` function directly to deposit ETH.
*   Alice sees his transaction in the mempool and front-runs his transaction. She swaps a large amount of ETH into the Curve pool and gets CVX in return.
*   Now, Bob's transaction is executed:
    *   `buyCvx()` attempts to swap Bob's ETH deposit for CVX.
    *   However, since the pool currently has a lot more ETH than CVX, Bob only gets a small amount of CVX in return.
*   Alice back-runs Bob's transaction and swaps the CVX she gained for ETH in the pool, which would result in a profit for her.

In this scenario, Alice has sandwiched Bob's `deposit()` transaction for a profit, causing Bob to receive less CVX for his deposited ETH.

### Recommended Mitigation

Consider adding a `_minOut` parameter to either `buyCvx()` and `sellCvx()`, or the following functions:

*   [`deposit()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L35-L46)
*   [`depositRewards()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L199-L208)
*   [`withdraw()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L105-L129)

This allows the caller to specify a minimum amount they expect from the swap, which would protect them from slippage.

**[elmutt (Asymmetry) confirmed and commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23#issuecomment-1738271917):**
 > @toshiSat - I think we should just lock this down so afEth can only use votium strategy.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23#issuecomment-1746323875):**
 > Marking this as primary issue and best report because it addresses all edge cases where slippage should be checked.

**[elmutt (Asymmetry) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23#issuecomment-1747218944):**
> In order fully solve it and issues marked as duplicates x([\#24](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/24), [\#61](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/61), [\#15](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/15)) we also need to pass \_minout to afEth.applyRewards() :

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23#issuecomment-1747452519):**
 > @elmutt - Agree with you on this.

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > For this one we locked down the depositRewards function and added a minout to the reward functions.

**Status**: Mitigation confirmed. Full details in reports from [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/16) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/32).

***

 
# Medium Risk Findings (10)
## [[M-01] AfEth collaterals cannot be balanced after ratio is changed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/55)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/55)*

The AfEth ratio between the collaterals can be modified but there is no direct way to balance the assets to follow the new ratio.

### Impact

The AfEth contract contains a configurable parameter `ratio` that indicates the intended balance between the two collaterals, SafEth and the Votium strategy. For example, a value of `3e17` means that 30% of the TVL should go to SafEth, and 70% should go to Votium.

This ratio is followed when new deposits are made. The deposited ETH is splitted according to the ratio and channeled in proportion to each collateral. The ratio is also checked when rewards are deposited to direct them to the proper collateral.

The ratio can also be modified by the admins of the protocol using the `setRatio()` function.

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L90-L92>

```solidity
90:     function setRatio(uint256 _newRatio) public onlyOwner {
91:         ratio = _newRatio;
92:     }
```

However, there is no way to balance the assets once a new ratio is defined. The implementation will need to rely on new deposits and reward compounding to slowly correct the offset, which may take a lot of time and may be impractical for most cases.

### Proof of Concept

Let's assume the protocol is deployed with a ratio of `3e17`.

1.  Deposits follow the configured ratio and split the TVL in 30% for SafEth and 70% for Votium.
2.  At some point, the protocol decides to switch to 50%-50% and sets a new ratio of `5e17`.
3.  New deposits will follow the new ratio and split 50% for each collateral, but these have potentially accumulated a large amount of TVL with the previous split. The existing difference will continue, new deposits can't correct this offset.

### Recommendation

Similar to how it is done in SafEth, the AfEth contract could have a rebalancing function which withdraws the proper amount from one collateral and deposits it in the other collateral, in order to correct the offset and target the new configured ratio. This function should be admin controlled, and support slippage control to correctly handle potentially large amounts of swaps.

An alternative could be to also correct a potential deviation in the ratio using new deposits. This could help speed up the correction by not only relying on rewards, but will also endure a delay in the convergence.

**[elmutt (Asymmetry) acknowledged, but disagreed with severity and commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/55#issuecomment-1747051030):**
 > It's hard to rebalance because of the locked convex. We have discussed this internally and consider it an acceptable risk for now so im acknowledging this issue instead of confirming.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/55#issuecomment-1751706361):**
 > In this instance, the implementation of the ratio mechanism with respect to rebalancing between `vEth` and `safEth` is a design decision that has flaws and potentially needs to be redesigned. However, the warden has identified that the current implementation does not enforce the ratio as what might be expected from users. For example, once the ratio has been changed, it takes some time for this to be fully in effect and as a result, users may be depositing into the protocol with the expectation of some ratio but that isn't what is currently in place. Performing instant rebalances is a design decision done by `safEth` so maybe this is the right way to be consistent? Even if users are potentially retroactively affected, provided the delay is sufficient, users should be able to exit the protocol if they do not agree with the new ratio.
> 
> For these reasons, I would still like to keep this issue as medium severity because I think this mechanism is ultimately flawed and can be realistically improved. My decision is final on this.


***

## [[M-02] Swap functionality to sell rewards is too permissive and could cause accidental or intentional loss of value](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/54)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/54), also found by [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/65)*

While the intention is to use the 0x protocol to sell rewards, the implementation doesn't provide any basic guarantee this will correctly happen and grants the rewarder arbitrary control over the tokens held by the strategy.

### Impact

Rewards earned in the VotingStrategy contract are exchanged for ETH and deposited back into the protocol. As indicated by the documentation, the intention is to swap these rewards for ETH using the 0x protocol:

> Votium rewards are claimed with claimRewards() using merkle proofs published by votium every 2 weeks. applyRewards() sells rewards on 0x and deposits them back into afEth (and ultimately back into the safEth & votium strategies), making the afEth price go up.

However, the implementation of `applyRewards()` shallowly executes a series of calls to arbitrary targets:

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L272-L305>

```solidity
272:     function applyRewards(SwapData[] calldata _swapsData) public onlyRewarder {
273:         uint256 ethBalanceBefore = address(this).balance;
274:         for (uint256 i = 0; i < _swapsData.length; i++) {
275:             // Some tokens do not allow approval if allowance already exists
276:             uint256 allowance = IERC20(_swapsData[i].sellToken).allowance(
277:                 address(this),
278:                 address(_swapsData[i].spender)
279:             );
280:             if (allowance != type(uint256).max) {
281:                 if (allowance > 0) {
282:                     IERC20(_swapsData[i].sellToken).approve(
283:                         address(_swapsData[i].spender),
284:                         0
285:                     );
286:                 }
287:                 IERC20(_swapsData[i].sellToken).approve(
288:                     address(_swapsData[i].spender),
289:                     type(uint256).max
290:                 );
291:             }
292:             (bool success, ) = _swapsData[i].swapTarget.call(
293:                 _swapsData[i].swapCallData
294:             );
295:             if (!success) {
296:                 emit FailedToSell(_swapsData[i].sellToken);
297:             }
298:         }
299:         uint256 ethBalanceAfter = address(this).balance;
300:         uint256 ethReceived = ethBalanceAfter - ethBalanceBefore;
301: 
302:         if (address(manager) != address(0))
303:             IAfEth(manager).depositRewards{value: ethReceived}(ethReceived);
304:         else depositRewards(ethReceived);
305:     }
```

This not only fails to provide any guarantee that 0x will be used (and that it will be used correctly), but grants a lot of power to the rewarder which can be used accidentally or purposely to negatively impact the protocol. The rewarded role can grant any token approval to any spender and execute arbitrary calls on behalf of the VotingStrategy.

### Recommendation

Provide better guarantees in the implementation of `applyRewards()` that 0x will be used to swap rewards, to ensure a more transparent and less error prone solution.

*   Instead of granting arbitrary allowance to any spender, set this to the 0x entrypoint.
*   Change arbitrary calls to the 0x protocol entrypoint.
*   Data sent to the 0x contract could also be validated, for example to ensure the output token is ETH.

**[Asymmetry acknowledged](https://github.com/code-423n4/2023-10-asymmetry-mitigation?tab=readme-ov-file#out-of-scope)**


***

## [[M-03] Forced relock in VotiumStrategy withdrawal causes denial of service if Convex locking contract is shutdown](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/50)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/50), also found by [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/29)*

The VotiumStrategy withdrawal process involves relocking CVX tokens, which can potentially lead to a denial of service and loss of user funds if the underlying vlCVX contract is shutdown.

### Impact

When withdrawals are executed in VotiumStrategy, the implementation of `withdraw()` will call `relock()` in order to relock any available excess (i.e. expired tokens minus the pending obligations) of CVX tokens to lock them again in the Convex protocol.

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L135-L149>

```solidity
135:     function relock() public {
136:         (, uint256 unlockable, , ) = ILockedCvx(VLCVX_ADDRESS).lockedBalances(
137:             address(this)
138:         );
139:         if (unlockable > 0)
140:             ILockedCvx(VLCVX_ADDRESS).processExpiredLocks(false);
141:         uint256 cvxBalance = IERC20(CVX_ADDRESS).balanceOf(address(this));
142:         uint256 cvxAmountToRelock = cvxBalance > cvxUnlockObligations
143:             ? cvxBalance - cvxUnlockObligations
144:             : 0;
145:         if (cvxAmountToRelock > 0) {
146:             IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmountToRelock);
147:             ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmountToRelock, 0);
148:         }
149:     }
```

This seems fine at first, but if we dig into the implementation of `lock()` we can see that the preconditions of this function requires the contract not to be shutdown:

<https://etherscan.io/address/0x72a19342e8F1838460eBFCCEf09F6585e32db86E#code#L1469>

```solidity
521:     function _lock(address _account, uint256 _amount, uint256 _spendRatio, bool _isRelock) internal {
522:         require(_amount > 0, "Cannot stake 0");
523:         require(_spendRatio <= maximumBoostPayment, "over max spend");
524:         require(!isShutdown, "shutdown");
```

Which means that any call to `lock()` after the contract is shutdown will revert. This is particularly bad because `relock()` is called as part of the withdraw process. If the vlCVX contract is shutdown, VotiumStrategy depositors won't be able to withdraw their position, causing a potential loss of funds.

Note that this also will cause a DoS while depositing rewards, since `depositRewards()` in VotiumStrategy also calls `ILockedCvx::lock()`.

### Proof of Concept

1.  vlCVX contract is shutdown.
2.  A user requests a withdrawal using `requestWithdrawal()`
3.  The user calls `withdraw()` when the withdrawal epoch is reached.
4.  The implementation will call `relock()`, which will call `ILockedCvx::lock()`.
5.  The implementation of `lock()` will throw an error because the vault is already shutdown.
6.  The transaction will be reverted.

### Recommendation

Add a condition to check if the contract is shutdown to avoid the call to `lock()` and the potential denial of service.

```diff
    function relock() public {
        (, uint256 unlockable, , ) = ILockedCvx(VLCVX_ADDRESS).lockedBalances(
            address(this)
        );
        if (unlockable > 0)
            ILockedCvx(VLCVX_ADDRESS).processExpiredLocks(false);
        uint256 cvxBalance = IERC20(CVX_ADDRESS).balanceOf(address(this));
        uint256 cvxAmountToRelock = cvxBalance > cvxUnlockObligations
            ? cvxBalance - cvxUnlockObligations
            : 0;
-       if (cvxAmountToRelock > 0) {
+       if (cvxAmountToRelock > 0 && !ILockedCvx(VLCVX_ADDRESS).isShutdown()) {
            IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmountToRelock);
            ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmountToRelock, 0);
        }
    }
```

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/50#issuecomment-1741384749)**

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > Check if vlcvx contract is shutdown before trying to relock.

**Status**: Mitigation confirmed. Full details in reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/18) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/36).

***

## [[M-04] VotiumStrategy withdrawal queue fails to consider available unlocked tokens causing different issues in the withdraw process](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/49)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/49), also found by [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/63), [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/30), [m\_Rassska](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/20), and rvierdiiev ([1](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/4), [2](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/3))*

Withdrawals in VotiumStrategy are executed in queue since CVX tokens are potentially locked in Convex. However, the implementation fails to consider the case where unlocked assets are already enough to cover the withdrawal, leading to different issues.

### Impact

VotiumStrategy withdrawals are executed in queue since the underlying CVX tokens may be locked in the Convex platform. Depositors must request a withdrawal and wait in queue until the epoch associated with their withdrawal is reached in order to exit their position. The core of this logic is present in the function `requestWithdraw()`:

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L54-L103>

```solidity
054:     function requestWithdraw(
055:         uint256 _amount
056:     ) public override returns (uint256 withdrawId) {
057:         latestWithdrawId++;
058:         uint256 _priceInCvx = cvxPerVotium();
059: 
060:         _burn(msg.sender, _amount);
061: 
062:         uint256 currentEpoch = ILockedCvx(VLCVX_ADDRESS).findEpochId(
063:             block.timestamp
064:         );
065:         (
066:             ,
067:             uint256 unlockable,
068:             ,
069:             ILockedCvx.LockedBalance[] memory lockedBalances
070:         ) = ILockedCvx(VLCVX_ADDRESS).lockedBalances(address(this));
071:         uint256 cvxAmount = (_amount * _priceInCvx) / 1e18;
072:         cvxUnlockObligations += cvxAmount;
073: 
074:         uint256 totalLockedBalancePlusUnlockable = unlockable +
075:             IERC20(CVX_ADDRESS).balanceOf(address(this));
076: 
077:         for (uint256 i = 0; i < lockedBalances.length; i++) {
078:             totalLockedBalancePlusUnlockable += lockedBalances[i].amount;
079:             // we found the epoch at which there is enough to unlock this position
080:             if (totalLockedBalancePlusUnlockable >= cvxUnlockObligations) {
081:                 (, uint32 currentEpochStartingTime) = ILockedCvx(VLCVX_ADDRESS)
082:                     .epochs(currentEpoch);
083:                 uint256 timeDifference = lockedBalances[i].unlockTime -
084:                     currentEpochStartingTime;
085:                 uint256 epochOffset = timeDifference /
086:                     ILockedCvx(VLCVX_ADDRESS).rewardsDuration();
087:                 uint256 withdrawEpoch = currentEpoch + epochOffset;
088:                 withdrawIdToWithdrawRequestInfo[
089:                     latestWithdrawId
090:                 ] = WithdrawRequestInfo({
091:                     cvxOwed: cvxAmount,
092:                     withdrawn: false,
093:                     epoch: withdrawEpoch,
094:                     owner: msg.sender
095:                 });
096: 
097:                 emit WithdrawRequest(msg.sender, cvxAmount, latestWithdrawId);
098:                 return latestWithdrawId;
099:             }
100:         }
101:         // should never get here
102:         revert InvalidLockedAmount();
103:     }
```

The implementation first considers available tokens that should be ready to be withdrawn. Line 74-75 sets `totalLockedBalancePlusUnlockable` to the amount of unlockable tokens (expired tokens in Convex that can be withdrawn) plus any available CVX balance in the contract.

However, the implementation fails to consider that **this available balance may be already enough to cover the withdrawal**, and proceeds to search within the locked balances by epoch. This will lead to different issues:

*   A user that requests a withdrawal of an amount such that there is enough available balance to cover for it, will still need to wait until the end of the next locked cycle. The implementation will start searching the locked balances array and stop at the first match, and set the withdrawal epoch as the end of the matched period. Even if there are enough tokens to cover for the withdrawal, the user is forced into an unnecessary delay.
*   If there are none locked balances, meaning everything is already in an unlockable state, the implementation will **cause a denial of service**. The for loop in line 77 won't be executed and the execution will be reverted due to the revert in line 102.

### Proof of Concept

Let's illustrate the denial of service case. We assume all deposits in VotiumStrategy were done before the 16 weeks period, which means all vlCVX should be in an unlocked state.

1.  User calls `requestWithdraw(amount)`.
2.  The implementation fetches current position from vlCVX contract using `ILockedCvx.lockedBalances()`. This will return the entire position as `unlockable` and an empty array for `lockedBalances`.
3.  The function sets `totalLockedBalancePlusUnlockable` as `unlockable + IERC20(CVX_ADDRESS).balanceOf(address(this))`. This should be enough to cover the requested amount by the user.
4.  The implementation continues to search through the `lockedBalances`, but since this array is empty the for loop is never executed.
5.  The function reaches the end and is reverted with a `InvalidLockedAmount()` error.

### Recommendation

Before searching through the `lockedBalances` array, check if there available unlocked tokens are enough to cover the withdrawal. If so, the withdrawal can be set for the current epoch. This will fix the unnecessary delay and the potential denial of service.

```diff
    function requestWithdraw(
        uint256 _amount
    ) public override returns (uint256 withdrawId) {
        latestWithdrawId++;
        uint256 _priceInCvx = cvxPerVotium();

        _burn(msg.sender, _amount);

        uint256 currentEpoch = ILockedCvx(VLCVX_ADDRESS).findEpochId(
            block.timestamp
        );
        (
            ,
            uint256 unlockable,
            ,
            ILockedCvx.LockedBalance[] memory lockedBalances
        ) = ILockedCvx(VLCVX_ADDRESS).lockedBalances(address(this));
        uint256 cvxAmount = (_amount * _priceInCvx) / 1e18;
        cvxUnlockObligations += cvxAmount;

        uint256 totalLockedBalancePlusUnlockable = unlockable +
            IERC20(CVX_ADDRESS).balanceOf(address(this));
        
+       if (totalLockedBalancePlusUnlockable >= amount) {
+           withdrawIdToWithdrawRequestInfo[
+               latestWithdrawId
+           ] = WithdrawRequestInfo({
+               cvxOwed: cvxAmount,
+               withdrawn: false,
+               epoch: currentEpoch,
+               owner: msg.sender
+           });
+           emit WithdrawRequest(msg.sender, cvxAmount, latestWithdrawId);
+           return latestWithdrawId;
+       }

        for (uint256 i = 0; i < lockedBalances.length; i++) {
            totalLockedBalancePlusUnlockable += lockedBalances[i].amount;
            // we found the epoch at which there is enough to unlock this position
            if (totalLockedBalancePlusUnlockable >= cvxUnlockObligations) {
                (, uint32 currentEpochStartingTime) = ILockedCvx(VLCVX_ADDRESS)
                    .epochs(currentEpoch);
                uint256 timeDifference = lockedBalances[i].unlockTime -
                    currentEpochStartingTime;
                uint256 epochOffset = timeDifference /
                    ILockedCvx(VLCVX_ADDRESS).rewardsDuration();
                uint256 withdrawEpoch = currentEpoch + epochOffset;
                withdrawIdToWithdrawRequestInfo[
                    latestWithdrawId
                ] = WithdrawRequestInfo({
                    cvxOwed: cvxAmount,
                    withdrawn: false,
                    epoch: withdrawEpoch,
                    owner: msg.sender
                });

                emit WithdrawRequest(msg.sender, cvxAmount, latestWithdrawId);
                return latestWithdrawId;
            }
        }
        // should never get here
        revert InvalidLockedAmount();
    }
```

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/49#issuecomment-1746416854):**
 > It seems pretty severe that `requestWithdraw()` would fail when the `lockedBalances` array is empty right?

**[elmutt (Asymmetry) confirmed and commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/49#issuecomment-1749259495):**
 > @0xleastwood - Absolutely agree. We are working on a fix to address this issue.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/49#issuecomment-1752018144):**
 > I think this DoS is recoverable. If all locks are unlocked/expired, then it will not be possible to request a withdrawal. However, if `relock()` is called, then additional requests to withdraw can be processed and the final impact is that the request is delayed more than it needs to be.
>
> It's not possible to prevent withdrawal finalisation by calling `relock()` first either because `cvxUnlockObligations` is used to reserve cvx that is owed to existing withdrawal requests.
>
> I think I will leave it as-is unless there is a way to brick funds by preventing users from finalising their withdrawals.

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 >	Check if available amount to withdraw is already in contract.

**Status**: Mitigation confirmed. Full details in reports from [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/37) and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/19).

***

## [[M-05] Reward sandwiching in VotiumStrategy](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/45)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/45), also found by [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/9)*

The reward system in VotiumStrategy can be potentially gamed by users to enter just before rewards are deposited and request an exit after that. Depending on the withdrawal queue, users may exit as early as the next epoch and avoid waiting the normal 16 weeks of vote locked CVX.

### Impact

Voting in the Convex protocol requires a commitment of at least 16 weeks. Holders of CVX tokens can lock their tokens into vlCVX, which grants them voting power in Curve gauges.

The same mechanism is applied internally in the VotiumStrategy contract. Deposited ETH is swapped to CVX and locked for vlCVX. Withdrawals are executed in a queued fashion, by reserving tokens that will eventually expire in coming epochs. A user exiting the strategy may have enough tokens to exit their position as early as the next epoch.

This means that, under the right circumstances, a user may deposit in VotiumStrategy and withdraw from it in a short period of time. The user just needs to have available expirable tokens coming from previous deposits in the platform, not necessarily related to the ones coming from their deposit. This can potentially reduce the commitment, requiring much less time than the required 16 weeks when using Convex directly.

This would allow users to game the system and enter the protocol just to collect the rewards, with a minimal commitment in the platform.

### Proof of Concept

Let's say an attacker is anticipating the claiming of rewards in VotiumStrategy, and let's assume also that there are enough tokens that will be expiring in the next epoch to sufficiently cover their position.

1.  The attacker deposits into the strategy just before rewards are claimed.
2.  Rewarder claims rewards and deposits them back into the strategy, increasing the value for holders.
3.  Right after that, the attacker requests a withdrawal. Since there are enough expirable tokens, the withdrawal is queued for the next epoch.
4.  The attacker just needs to wait for the next epoch to exit their position, along with the rewards.

### Recommendation

This is a variation of a common attack in vaults that compound rewards, present in different yield protocols. The usual mitigation is to introduce some delay or penalty to avoid bad intentionally users from depositing just to earn the rewards and leave.

In this case, two possible solutions are:

*   Introduce some kind of minimum permanency delay for depositors. This could be the 16 weeks defined by Convex, or a fraction of it to be more flexible, e.g. 4 weeks.
*   Stream rewards over a period of time. Instead of just depositing back the rewards as an immediate increase of value, have these rewards be linearly unlocked over a period of time. This will cause depositors to stay within the protocol to collect the rewards.

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/45#issuecomment-1747017488)**

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > Add a minimum epoch of 1 to not allow users to immediately withdraw.

**Status**: Unmitigated. Full details in reports from [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/21), [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/9), and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/38), and also included in the [Mitigation Review](#mitigation-review) section below.

***

## [[M-06] Missing deadline check for AfEth actions](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/43)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/43)*

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L148> 

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L243>

AfEth main actions execute on-chain swaps and lack an expiration deadline, which enables pending transactions to be maliciously executed at a later point.

### Impact

Both AfEth deposits and withdrawals include on-chain swaps in AMM protocols as part of their execution, in order to convert the deposited ETH into the different underlying assets held by SafEth and the Votium strategy.

In the case of SafEth, depending on the derivative, staking may involve swapping ETH for other LSD. For example, the RocketPool derivative implementation uses Balancer to swap between ETH and rETH during deposits:

<https://etherscan.io/address/0xb3e64c481f0fc82344a7045592284fddb9905b8b#code#F1#L157>

```solidity
function deposit() external payable onlyOwner returns (uint256) {
    uint256 rethBalanceBefore = IERC20(rethAddress()).balanceOf(
        address(this)
    );
    balancerSwap(msg.value);
    uint256 received = IERC20(rethAddress()).balanceOf(address(this)) -
        rethBalanceBefore;
    underlyingBalance = super.finalChecks(
        ethPerDerivative(true),
        msg.value,
        maxSlippage,
        received,
        true,
        underlyingBalance
    );
    return received;
}
```

In the case of Votium, deposited ETH is swapped to CVX in order to lock it in Convex. Similarly, when withdrawing, CVX tokens are swapped back to ETH. This is done using a Curve Pool in the `buyCvx()` and `sellCvx()` functions:

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L227-L265>

```solidity
227:     function buyCvx(
228:         uint256 _ethAmountIn
229:     ) internal returns (uint256 cvxAmountOut) {
230:         address CVX_ETH_CRV_POOL_ADDRESS = 0xB576491F1E6e5E62f1d8F26062Ee822B40B0E0d4;
231:         // eth -> cvx
232:         uint256 cvxBalanceBefore = IERC20(CVX_ADDRESS).balanceOf(address(this));
233:         ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying{
234:             value: _ethAmountIn
235:         }(
236:             0,
237:             1,
238:             _ethAmountIn,
239:             0 // this is handled at the afEth level
240:         );
241:         uint256 cvxBalanceAfter = IERC20(CVX_ADDRESS).balanceOf(address(this));
242:         cvxAmountOut = cvxBalanceAfter - cvxBalanceBefore;
243:     }
244: 
245:     /**
246:      * @notice - Internal utility function to sell cvx for eth
247:      * @param _cvxAmountIn - Amount of cvx to sell
248:      * @return ethAmountOut - Amount of eth received
249:      */
250:     function sellCvx(
251:         uint256 _cvxAmountIn
252:     ) internal returns (uint256 ethAmountOut) {
253:         address CVX_ETH_CRV_POOL_ADDRESS = 0xB576491F1E6e5E62f1d8F26062Ee822B40B0E0d4;
254:         // cvx -> eth
255:         uint256 ethBalanceBefore = address(this).balance;
256:         IERC20(CVX_ADDRESS).approve(CVX_ETH_CRV_POOL_ADDRESS, _cvxAmountIn);
257: 
258:         ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying(
259:             1,
260:             0,
261:             _cvxAmountIn,
262:             0 // this is handled at the afEth level
263:         );
264:         ethAmountOut = address(this).balance - ethBalanceBefore;
265:     }
```

While both actions in AfEth, `deposit()` and `withdraw()`, have a minimum output parameter to control slippage, this doesn't offer protection against when the transaction is actually executed. If the price of the underlying assets drops while the transaction is pending, then the minimum output can still be fulfilled, but the user will get a bad rate due to the stale price. The outdated slippage value now allows for a high slippage trade in detriment of the user.

This can be also attacked by MEV bots which can still sandwich the transaction to profit on the difference. See [this issue](https://github.com/code-423n4/2022-12-backed-findings/issues/64) for an excellent reference on the topic (the author runs a MEV bot).

### Proof of Concept

1.  A user submits a transaction to deposit in AfEth.
2.  The transaction sits in the mempool without being included in a block.
3.  The price of CVX/ETH drops.
4.  The transaction gets executed by the blockchain.
5.  Since the price of CVX has dropped, the user will still get the minimum output expected but this will still represent less tokens than it would expect since the transaction has been delayed and the original price is now stale.

### Recommendation

Add a deadline timestamp to the `deposit()` and `withdraw()` functions, and revert if this timestamp has passed.

Note also that the same should be applied to the VotiumStrategy contract if deposits and withdrawals are made directly there, without going through AfEth.

```diff
-   function deposit(uint256 _minout) external payable virtual {
+   function deposit(uint256 _minout, uint256 deadline) external payable virtual {
            if (pauseDeposit) revert Paused();
+           if (block.timestamp > deadline) revert StaleAction();
```

```diff
    function withdraw(
        uint256 _withdrawId,
-       uint256 _minout
+       uint256 _minout,
+       uint256 deadline
    ) external virtual onlyWithdrawIdOwner(_withdrawId) {
        if (pauseWithdraw) revert Paused();
+       if (block.timestamp > deadline) revert StaleAction();
```

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/43#issuecomment-1749768260)**

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > Add a deadline check for deposit & withdraw.

**Status**: Mitigation confirmed. Full details in reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/20) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/39).

***

## [[M-07] Lack of access control and value validation in the reward flow exposes functions to public access](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38)*

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L203>

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L272>

Some functions that are part of the Votium reward flow are left unprotected and can be accessed by anyone to spend resources held by the contract.

### Impact

Rewards coming from the Votium protocol are claimed and compounded back in AfEth. This flow consists of two parts, both controlled and initiated by the rewarder role: first, rewards are claimed in Votium and Convex using `claimRewards()`, second, those rewards are swapped to ETH and deposited back in the protocol using `applyRewards()`.

![reward](https://user-images.githubusercontent.com/131902879/275634967-8bd434e6-2c0a-47fe-aaf9-4e200561e1b0.png)

After rewards are swapped, the VotiumStrategy will call AfEth to manage the deposited rewards, which eventually calls back the VotiumStrategy. These interactions are represented in the previous diagram as steps 5 and 6.

However, both of the functions that implement these steps are publicly accessible and don't have any validation over the amount of ETH sent. Let's first see the case of `AfEth::depositRewards()`:

```solidity
272:     function depositRewards(uint256 _amount) public payable {
273:        IVotiumStrategy votiumStrategy = IVotiumStrategy(vEthAddress);
274:         uint256 feeAmount = (_amount * protocolFee) / 1e18;
275:         if (feeAmount > 0) {
276:             // solhint-disable-next-line
277:             (bool sent, ) = feeAddress.call{value: feeAmount}("");
278:             if (!sent) revert FailedToSend();
279:         }
280:         uint256 amount = _amount - feeAmount;
281:         uint256 safEthTvl = (ISafEth(SAF_ETH_ADDRESS).approxPrice(true) *
282:             safEthBalanceMinusPending()) / 1e18;
283:         uint256 votiumTvl = ((votiumStrategy.cvxPerVotium() *
284:             votiumStrategy.ethPerCvx(true)) *
285:             IERC20(vEthAddress).balanceOf(address(this))) / 1e36;
286:         uint256 totalTvl = (safEthTvl + votiumTvl);
287:         uint256 safEthRatio = (safEthTvl * 1e18) / totalTvl;
288:         if (safEthRatio < ratio) {
289:             ISafEth(SAF_ETH_ADDRESS).stake{value: amount}(0);
290:         } else {
291:             votiumStrategy.depositRewards{value: amount}(amount);
292:         }
293:     }
```

As we can see in the previous snippet of code, the function doesn't have any access control and doesn't check if the `_amount` parameter matches the amount of ETH being sent (`msg.value`). Anyone can call this function with any amount value without actually sending any ETH value.

The implementation of `depositRewards()` in `VotiumStrategyCore` has the same issue:

```solidity
203:     function depositRewards(uint256 _amount) public payable {
204:         uint256 cvxAmount = buyCvx(_amount);
205:         IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmount);
206:         ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmount, 0);
207:         emit DepositReward(cvxPerVotium(), _amount, cvxAmount);
208:     }
```

Any ETH held in these two contracts can be arbitrarily spent by any unauthorized account. The caller cannot remove value from here, unless sandwiching the trade or benefitting via a third-party call, but can use these functions to grief and unauthorizedly spend any ETH present in these contracts.

### Recommendation

If these functions are indeed intended to be publicly accessible, then add a validation to ensure that the amount argument matches the callvalue sent, i.e. `require(_amount == msg.value)`.

On the other hand, if these should only be part of the reward flow initiated by the rewarder role, then validate that `AfEth::depositRewards()` is called from the Votium Strategy (`vEthAddress`), and validate that `VotiumStrategy::depositRewards()` is called either from AfEth (`manager`) or internally through `applyRewards()`.

**[elmutt (Asymmetry) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38#issuecomment-1743662053):**
 > @toshiSat - I believe best solution is to get rid of \_amount and only use msg.value here.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38#issuecomment-1751998547):**
 > This is valid and I think should have been a standalone issue, it seems like anyone can arbitrarily spend native ether in exchange for locked cvx tokens. It is unclear how this might be abused because `applyRewards()` will only transfer out new native ether which was received from its swaps.

**[d3e4 (Warden) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38#issuecomment-1752106853):**
 > @0xleastwood - I mentioned this issue in [L-08](https://github.com/code-423n4/2023-09-asymmetry-findings/blob/main/data/d3e4-Q.md#l-08-depositrewards_amount-do-not-check-that-_amount--msgvalue).
> 
> However, note that the contracts do not hold any ETH (except [dust](https://github.com/code-423n4/2023-09-asymmetry-findings/blob/main/data/d3e4-Q.md#l-01-dust-loss) or if sent there by mistake) so calling these functions directly wouldn't really do anything. And if there is any ETH in the contract it is just deposited as rewards, which is the only way it should be spent; it cannot be made to send it elsewhere.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38#issuecomment-1752149195):**
 > @d3e4 - These are not equivalent, you talk about the fact that `_amount == msg.value` should be checked but fail to highlight the impact of not having this check. Plus this issue is more than that, the `depositRewards()` function is missing access control.
>
> Keeping this as Medium.

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38#issuecomment-1753303844):**

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > Here we did two things, check msg.value instead of passing in amount & make deposit rewards private.

**Status**: Unmitigated. Full details in reports from [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/40) and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/27), and also included in the [Mitigation Review](#mitigation-review) section below.

***

## [[M-08] Inflation attack in VotiumStrategy](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35)
*Submitted by [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35), also found by [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/26)*

The VotiumStrategy contract is susceptible to the [Inflation Attack](https://mixbytes.io/blog/overview-of-the-inflation-attack), in which the first depositor can be front-runned by an attacker to steal their deposit.

### Impact

Both AfEth and VotiumStrategy acts as vaults: accounts deposit some tokens and get back another token (share) that represents their participation in the vault.

These types of contracts are potentially vulnerable to the inflation attack: an attacker can front-run the initial deposit to the vault to inflate the value of a share and render the front-runned deposit worthless.

In AfEth, this is successfully mitigated by the slippage control. Any attack that inflates the value of a share to decrease the number of minted shares is rejected due to the validation of minimum output:

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L166-L167>

```solidity
166:         uint256 amountToMint = totalValue / priceBeforeDeposit;
167:         if (amountToMint < _minout) revert BelowMinOut();
```

However, this is not the case of VotiumStrategy. In this contract, no validation is done in the number of minted tokens. This means that an attacker can execute the attack by front-running the initial deposit, which may be from AfEth or from any other account that interacts with the contract. See *Proof of Concept* for a detailed walkthrough of the issue.

<https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L39-L46>

```solidity
39:     function deposit() public payable override returns (uint256 mintAmount) {
40:         uint256 priceBefore = cvxPerVotium();
41:         uint256 cvxAmount = buyCvx(msg.value);
42:         IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmount);
43:         ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmount, 0);
44:         mintAmount = ((cvxAmount * 1e18) / priceBefore);
45:         _mint(msg.sender, mintAmount);
46:     }
```

### Proof of Concept

Let's say a user wants to deposit in VotiumStrategy and calls `deposit()` sending an ETH amount such as it is expected to buy `X` tokens of CVX. Attacker will front-run the transaction and execute the following:

1.  Initial state is empty contract, `assets = 0` and `supply = 0`.
2.  Attacker calls deposit with an amount of ETH such as to buy `1e18` CVX tokens, this makes `assets = 1e18` and `supply = 1e18`.
3.  Attacker calls `requestWithdraw(1e18 - 1)` so that `supply = 1`, assume also `1e18 - 1` CVX tokens are withdrawn so that `cvxUnlockObligations = 1e18 - 1`.
4.  Attacker transfers (donates) X amount of CVX to VotiumStrategy contract.
5.  At this point, `priceBefore = cvxPerVotium() = (totalCvx - cvxUnlockObligations) * 1e18 / supply = (X + 1e18 - (1e18 - 1)) * 1e18 / 1 = (X + 1) * 1e18`
6.  User transaction gets through and `deposit()` buys X amount of CVX. Minted tokens will be `mintAmount = X * 1e18 / priceBefore = X * 1e18 / (X + 1) * 1e18 = X / (X + 1) = 0`.
7.  User is then minted zero VotiumStrategy tokens.
8.  Attacker calls `requestWithdraw()` again to queue withdrawal to remove all CVX balance from the contract, including the tokens deposited by the user.

### Recommendation

There are multiple ways of solving the issue:

1.  Similar to AfEth, add a minimum output check to ensure the amount of minted shares.
2.  Track asset balances internally so an attacker cannot donate assets to inflate shares.
3.  Mint an initial number of "dead shares", similar to how UniswapV2 does.

A very good discussion of these can be found [here](https://github.com/OpenZeppelin/openzeppelin-contracts/issues/3706).

**[0xleastwood (Judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35#issuecomment-1746303423):**
 > Downgrading this to medium severity because the `_minOut` parameter should actually prevent this attack as long as it's non-zero, but I agree this is of concern if users do not set this parameter. This is a stated assumption.
>
 > Upon further investigation, `AfEth.deposit()` is not vulnerable to the deposit front-running. This is only an issue if we are interacting with the votium strategy contract directly which is atypical behaviour. However, funds are still at risk even with these stated assumptions so I believe medium severity to be more correct.

**[MiloTruck (Warden) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35#issuecomment-1752132516):**
 > @0xleastwood - Apologies for commenting after post-judging QA, but isn't the inflation attack still a problem even if users only interact with the `AfEth` contract?
> 
> `AfEth.deposit()` calls VotiumStrategy's `deposit()` function, so if a user calls `AfEth.deposit()` after VotiumStrategy's state has been manipulated, `vMinted` will be 0, causing him to lose the portion of his ETH that was deposited into VotiumStrategy.
> 
> Unless I'm missing something here...

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35#issuecomment-1752133924):**
> @MiloTruck - Ultimately, I do believe `_minOut` to be sufficient in detecting such an attack.

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35#issuecomment-1753975944)**

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > Track balances instead of using `balanceOf`.

**Status**: Mitigation confirmed. Full details in reports from [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/41) and [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/24).

***

## [[M-09] Missing circuit breaker checks in `ethPerCvx()` for Chainlink's price feed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/31)
*Submitted by [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/31)*

The `ethPerCvx()` function relies on a Chainlink oracle to fetch the CVX / ETH price:

[VotiumStrategyCore.sol#L158-L169](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L158-L169)

```solidity
        try chainlinkCvxEthFeed.latestRoundData() returns (
            uint80 roundId,
            int256 answer,
            uint256 /* startedAt */,
            uint256 updatedAt,
            uint80 /* answeredInRound */
        ) {
            cl.success = true;
            cl.roundId = roundId;
            cl.answer = answer;
            cl.updatedAt = updatedAt;
        } catch {
```

The return values from `latestRoundData()` are validated as such:

[VotiumStrategyCore.sol#L173-L181](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L173-L181)

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
```

As seen from above, there is no check to ensure that `cl.answer` does not go below or above a certain price.

Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. Therefore, if CVX experiences a huge drop/rise in value, the CVX / ETH price feed will continue to return `minAnswer`/`maxAnswer` instead of the actual price of CVX.

Currently, `minAnswer` is set to `1e13` and `maxAnswer` is set to `1e18`. This can be checked by looking at the [AccessControlledOffchainAggregator](https://etherscan.io/address/0xf1F7F7BFCc5E9D6BB8D9617756beC06A5Cbe1a49) contract for the CVX / ETH price feed. Therefore, if CVX ever experiences a flash crash and its price drops to below `1e13` (eg. `100`), the `cl.answer` will still be `1e13`.

This becomes problematic as `ethPerCvx()` is used to determine the price of vAfEth:

[VotiumStrategy.sol#L31-L33](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L31-L33)

```solidity
    function price() external view override returns (uint256) {
        return (cvxPerVotium() * ethPerCvx(false)) / 1e18;
    }
```

Furthermore, vAfEth's price is used to calculate the amount of AfEth to mint to users whenever they call `deposit()`:

[AfEth.sol#L162-L166](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L162-L166)

```solidity
        totalValue +=
            (sMinted * ISafEth(SAF_ETH_ADDRESS).approxPrice(true)) +
            (vMinted * vStrategy.price());
        if (totalValue == 0) revert FailedToDeposit();
        uint256 amountToMint = totalValue / priceBeforeDeposit;
```

If CVX experiences a flash crash, `vStrategy.price()` will be `1e13`, which is much larger than the actual price of CVX. This will cause `totalValue` to become extremely large, which in turn causes `amountToMint` to be extremely large as well. Therefore, the caller will receive a huge amount of afEth.

### Impact

Due to Chainlink's in-built circuit breaker mechanism, if CVX experiences a flash crash, `ethPerCvx()` will return a price higher than the actual price of CVX. Should this occur, an attacker can call `deposit()` to receive a huge amount of afEth as it uses an incorrect CVX price.

This would lead to a loss of funds for previous depositors, as the attacker would hold a majority of afEth's total supply and can withdraw most of the protocol's TVL.

### Proof of Concept

Assume the following:

*   For convenience, assume that 1 safEth is worth 1 ETH.
*   The `AfEth` contract has the following state:
    *   `ratio = 5e17` (50%)
    *   `totalSupply() = 100e18`
    *   `safEthBalanceMinusPending() = 50e18`
    *   `vEthStrategy.balanceOf(address(this))` (vAfEth balance) is `50e18`
*   The `VotiumStrategy` contract has the following state:
    *   Only 50 vAfEth has been minted so far (`totalSupply() = 50e18`).
    *   The contract only has 50 CVX in total (`cvxInSystem() = 50e18`).
    *   This means that [`cvxPerVotium()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L140-L149) returns `1e18` as:

```solidity
((totalCvx - cvxUnlockObligations) * 1e18) / supply = ((50e18 - 0) * 1e18) / 50e18 = 1e18
```

The price of CVX flash crashes from `2e15 / 1e18` ETH per CVX to `100 / 1e18` ETH per CVX. Now, if an attacker calls `deposit()` with 10 ETH:

*   `priceBeforeDeposit`, which is equal to `price()`, is `5e17 + 5e12` as:

```solidity
safEthValueInEth = (1e18 * 50e18) / 1e18 = 50e18
vEthValueInEth = (1e13 * 50e18) / 1e18 = 5e14
((vEthValueInEth + safEthValueInEth) * 1e18) / totalSupply() = ((50e18 + 5e14) * 1e18) / 100e18 = 5e17 + 5e12
```

*   Since `ratio` is 50%, 5 ETH is staked into safEth:
    *   `sMinted = 5e18`, since the price of safEth and ETH is equal.
*   The other 5 ETH is deposited into `VotiumStrategy`:
    *   `priceBefore`, which is equal to `cvxPerVotium()`, is `1e18` as shown above.
    *   Since 1 ETH is worth `1e16` CVX (according to the price above), `cvxAmount = 5e34`.
    *   Therefore, `vMinted = 5e34` as:

```solidity
mintAmount = ((cvxAmount * 1e18) / priceBefore) = ((5e34 * 1e18) / 1e18) = 5e34 
```

*   To calculate `vStrategy.price()` after `VotiumStrategy`'s `deposit()` function is called:
    *   `ethPerCvx()` returns `1e13`, which is `minAnswer` for the CVX / ETH price fee.
    *   `cvxPerVotium()` is still `1e18` as:

```solidity
supply = totalSupply() = 5e34 + 50e18
totalCvx = cvxInSystem() = 5e34 + 50e18
((totalCvx - cvxUnlockObligations) * 1e18) / supply = ((5e34 + 50e18 - 0) * 1e18) / (5e34 + 50e18) = 1e18
```

*   Therefore `vStrategy.price()` returns `1e13` as:

```solidity
(cvxPerVotium() * ethPerCvx(false)) / 1e18 = (1e18 * 1e13) / 1e18 = 1e13
```

*   To calculate the amount of AfEth minted to the caller:

```solidity
totalValue = (5e18 * 1e18) + (5e34 * 1e13) = 5e47 + 5e36
amountToMint = totalValue / priceBeforeDeposit = (5e47 + 5e36) / (5e17 + 5e12) = ~1e30
```

As seen from above, the attacker will receive `1e30` AfEth, which is huge compared to the remaining `100e18` held by previous depositors before the flash crash.

Therefore, almost all of the protocol's TVL now belongs to the attacker as he holds most of AfEth's total supply. This results in a loss of funds for all previous depositors.

### Recommended Mitigation

Consider validating that the price returned by Chainlink's price feed does not go below/above a minimum/maximum price:

[VotiumStrategyCore.sol#L173-L181](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L173-L181)

```diff
        if (
            (!_validate ||
                (cl.success == true &&
                    cl.roundId != 0 &&
-                   cl.answer >= 0 &&
+                   cl.answer >= MIN_PRICE &&
+                   cl.answer <= MAX_PRICE &&
                    cl.updatedAt != 0 &&
                    cl.updatedAt <= block.timestamp &&
                    block.timestamp - cl.updatedAt <= 25 hours))
        ) {
```

This ensures that an incorrect price will never be used should CVX experience a flash crash, thereby protecting the assets of existing depositors.

**[Asymmetry acknowledged](https://github.com/code-423n4/2023-10-asymmetry-mitigation?tab=readme-ov-file#out-of-scope)**


***

## [[M-10] It might not be possible to `applyRewards()`, if an amount received is less than 0.05 eth](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/16)
*Submitted by [m\_Rassska](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/16)*

*   Upon claiming Votium rewards, `applyRewards()` is intended to be invoked bi-weekly in order to exchange the tokens for eth and put the eth received back into the strategies. Based on the current `ratio` it either stakes the amount into safETH or obtains some CVX by selling eth on Curve and then locks them to get vlCVX.

    *   ```Solidity
        uint256 safEthRatio = (safEthTvl * 1e18) / totalTvl;
        if (safEthRatio < ratio) {
            ISafEth(SAF_ETH_ADDRESS).stake{value: amount}(0);
        } else {
            votiumStrategy.depositRewards{value: amount}(amount);
        }

        ```

*   Let's say the `safEthRatio < ratio`, which triggers `ISafEth(SAF_ETH_ADDRESS).stake{value: amount}` being invoked. And if the `amount < ISafEth(SAF_ETH_ADDRESS).minAmount`. The whole re-investing strategy collapses.

*   As of Sep. 2023, the Votium rewards are 0,000016 eth for 1 vlCVX per round, it means, we need at least 3125 vlCVX being delegated to the Votium in order to pass the threshold.

### Impact

*   This could backfire in early stages or during certain market conditions, where the amount of vlCVX hold by afETH is not enough to generate 0.05 eth bi-weekly. Basically, that forces to flows that are inconsistent with the main re-investment flow proposed by Asymmetry, which ultimately could result in theft as demonstrated in the issue #15.

### Recommended Mitigation Steps

*   Short term:
    *   Wrapp `IAfEth(manager).depositRewards` into the try/catch block. And if one of the following conditions arises:
        *   The min safETH deposit amount is not reached
        *   Chainlink price feed could not be validated
        *   Low-level call which sends fees fails
        just simply invoke `VotiumStrategy.depositRewards()`.
        ```Solidity
          if (address(manager) == address(0)) {
              depositRewards(ethReceived);
          } else {
              try IAfEth(manager).depositRewards{value: ethReceived}(ethReceived) {}
              catch {depositRewards(ethReceived);}
          }
        ```
*   Long term: N/A

**[elmutt (Asymmetry) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/16#issuecomment-1738288823):**
 > Nice find. At first glance it doesn't seem to matter but when you pointed out out early market conditions resulting in less rewards it makes total sense that it will be a problem we will likely encounter.

**[m\_Rassska (Warden) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/16#issuecomment-1742024867):**
 > The reward threshold defined here should be changed to: 
> 
> $$ \frac{0,000016} {ratio} * \sum_{i = 0}^{lockedBalances.length - 1} lockedBalances[i] $$
> 
> Meaning that: 
>  - if the `ratio = 1e18` => the rewards > 0,05eth
>  - if the `ratio = 5e17` => the rewards > 0,1eth

**[elmutt (Asymmetry) confirmed and commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/16#issuecomment-1742076103):**
 >Thanks! After discussing internally we decided to solve this by calling setMinAmount(0) on the safEth contract.
> 
> Done here: https://etherscan.io/tx/0xb024f513adb9a8fa3bbceceeb6c11d2a1bc9c5e3545dfa76f6d6e0c8bdaf38a3

**[Asymmetry mitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation#mitigations-to-be-reviewed):**
 > No code changes needed. We removed the minimum stake amount from `SafEth`, as noted above.

**Status**: Mitigation confirmed. Full details in reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/25), [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/44), and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/33).

***

# Low Risk and Non-Critical Issues

For this audit, 5 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/56) by **adriro** received the top score from the judge.

*The following wardens also submitted reports: [MiloTruck](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/70), [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/67), [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/5), and [m\_Rassska](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/22).*

## Summary

### Low Issues

Total of **17 issues**:

|ID|Issue|
|:--:|:---|
| [L-01] | Hardcoded addresses may not be compatible with a multi-chain deployment |
| [L-02] | Use Ownable2Step instead of Ownable for access control |
| [L-03] | Missing name and symbol for AfEth token |
| [L-04] | Validate ratio argument in `setRatio()` |
| [L-05] | Price calculations may experience precision loss due to division before multiplication |
| [L-06] | Validate withdrawals in AfEth have been already executed |
| [L-07] | Prevent AfEth token transfers to its own contract |
| [L-08] | Public relock function can be used to grief withdrawal requests |
| [L-09] | Missing call to base initializers |
| [L-10] | Function `canWithdraw()` in VotiumStrategy doesn't check if withdrawal has been already executed |
| [L-11] | VotiumStrategy allows to recover ERC20 tokens but not native ETH |
| [L-12] | Missing usage of safe wrappers to handle ERC20 operations |
| [L-13] | Zero token allowance can cause denial of service in `applyRewards()` |
| [L-14] | Low level calls to account with no code will not fail |
| [L-15] | Protocol fees are not collected when rewards are not routed through AfEth |
| [L-16] | Protocol doesn't collect fees from SafEth |
| [L-17] | Potential rounding to zero issue in AfEth deposit could cause loss of value |

### Non Critical Issues

Total of **5 issues**:

|ID|Issue|
|:--:|:---|
| [N-01] | Remove debug symbols |
| [N-02] | Missing event for important parameter change |
| [N-03] | Unused constants |
| [N-04] | Use constants for literal or magic values |
| [N-05] | Missing check for zero address value in constructor or setter |

### Informational Issues

Total of **1 issue**:

|ID|Issue|
|:--:|:---|
| [I-01] | Consider using the ERC4626 standard |

## [L-01] Hardcoded addresses may not be compatible with a multi-chain deployment

The codebase is full of hardcoded addresses that refer to other parts of the protocol (SafEth, for example) or third-party protocols (e.g. Votium, Convex, Curve Pools).

Assumptions of the presence of third-party protocol, their deployment addresses and/or their arguments will prevent or complicate deployments in layer 2 chains. This is stated as a possibility in the documentation: 

> This will only be deployed to Ethereum Mainnet, with the chance of being deployed on L2's on a future date

## [L-02] Use Ownable2Step instead of Ownable for access control

Use the [Ownable2Step](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) variant of the Ownable contract to better safeguard against accidental transfers of access control.

*Instances (2)*:

- [AfEth](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L10)
- [VotiumStrategyCore](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L24)

## [L-03] Missing name and symbol for AfEth token

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L72

The AfEth contract inherits from ERC20Upgradeable but doesn't call its base initializer, which is in charge of setting the name and symbol for the ERC20 token.

## [L-04] Validate ratio argument in `setRatio()`

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L90

The `ratio` configuration parameter in AfEth measures the portion of deposited value that goes into the SafEth contract. This value is intended to range between `0` (0%) and `1e18` (100%).

The `setRatio()` function should validate that the new value is within bounds, i.e. `require(_newRatio <= 1e18)`.

## [L-05] Price calculations may experience precision loss due to division before multiplication

There are several places across the codebase in which intermediate results that depend on a division are then used as part of calculations that involve a multiple.

For example, in `requestWithdraw()` the calculation of `votiumWithdrawAmount` depends on the intermediate result of `withdrawRatio`. The expression can be simplified as:

```solidity
uint256 votiumWithdrawAmount = (_amount * votiumBalance) / (totalSupply() - afEthBalance);
```

And similarly, `safEthWithdrawAmount`:

```solidity
uint256 safEthWithdrawAmount = (_amount * safEthBalance) / (totalSupply() - afEthBalance);
```

## [L-06] Validate withdrawals in AfEth have been already executed

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L243

Withdrawals in AfEth are first enqueued when requested, and executed later when the vAfEth tokens are ready to be withdrawn. The `withdraw()` function in AfEth validates that the withdrawal is ready (by using `canWithdraw()`) but doesn't validate if the withdrawal has been already executed.

This isn't currently exploitable since the withdrawal in AfEth also depends on the withdrawal in VotiumStrategy, which correctly checks if the withdrawal associated to the `vEthWithdrawId` has been already claimed. Consider adding a similar check to `AfEth::withdraw()`.

## [L-07] Prevent AfEth token transfers to its own contract

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L183

When a user requests a withdrawal in AfEth, their tokens are transferred to the contract and "locked" until the withdrawal is made effective, at which point the tokens are burned.

Given this mechanism, AfEth tokens held by the same contract are considered as locked tokens but anyone can technically transfer tokens to the contract by calling `transfer()` or `transferFrom()`.

Consider adding a check to the base ERC20 functionality to prevent AfEth tokens from being sent to the contract by external actors.

## [L-08] Public relock function can be used to grief withdrawal requests

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L135

A malicious user can call `relock()` to front-run a transaction to `requestWithdraw()`. 

Relocking will take any available CVX in the contract and expired locks in CvxLocker and relock them. Griefed users will need to wait more, as any of the available balance that could have been used for the withdrawal has been relocked as part of the front-running.

## [L-09] Missing call to base initializers

Upgradeable contracts that inherit from other base contracts should call the corresponding base initializers during initialization.

*Instances (4)*:

- [AfEth::OwnableUpgradeable](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L72)
- [AfEth::ERC20Upgradeable](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L72)
- [AbstractStrategy::ReentrancyGuardUpgradeable](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/AbstractStrategy.sol#L10)
- [VotiumStrategyCore::OwnableUpgradeable](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L100)

## [L-10] Function `canWithdraw()` in VotiumStrategy doesn't check if withdrawal has been already executed

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L155

The implementation of `canWithdraw()` in the VotiumStrategy contract just checks if the required epoch has been reached, but doesn't validate if the request has been already fulfilled (`WithdrawRequestInfo.withdrawn`).

Consider also checking for the `withdrawn` condition as part of the implementation of `canWithdraw()`.

```solidity
function canWithdraw(
    uint256 _withdrawId
) external view virtual override returns (bool) {
    uint256 currentEpoch = ILockedCvx(VLCVX_ADDRESS).findEpochId(
        block.timestamp
    );
    return
        withdrawIdToWithdrawRequestInfo[_withdrawId].epoch <= currentEpoch && !withdrawIdToWithdrawRequestInfo[_withdrawId].withdrawn;
}
```

## [L-11] VotiumStrategy allows to recover ERC20 tokens but not native ETH

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L215

The implementation of `withdrawStuckTokens()` allows the owner of the protocol to recover any ERC20 token, but fails to consider native ETH transfers.

## [L-12] Missing usage of safe wrappers to handle ERC20 operations

- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L216
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L282
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L287

ERC20 operations on arbitrary tokens should be [safely wrapped](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20) to account for incompatible implementations.

## [L-13] Zero token allowance can cause denial of service in `applyRewards()`

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L282-L285

```solidity
IERC20(_swapsData[i].sellToken).approve(
    address(_swapsData[i].spender),
    0
);
```

During `applyRewards()`, token allowances are first reset to zero before being increased to infinity. This could cause issues with some ERC20 implementations that revert on zero value approvals, such as [BNB](https://etherscan.io/token/0xB8c77482e45F1F44dE1745F52C74426C631bDD52#code#L92).

## [L-14] Low level calls to account with no code will not fail

- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L292

Low level calls (i.e. `address.call(...)`) to account with no code will silently succeed without reverting or throwing any error. Quoting the reference for the CALL opcode in evm.codes:

> Creates a new sub context and execute the code of the given account, then resumes the current one. Note that an account with no code will return success as true.

## [L-15] Protocol fees are not collected when rewards are not routed through AfEth

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L302-L304

Protocol fees are collected in `AfEth::depositRewards()`, just before rewards are being compounded in the protocol.

The reward flow is initiated in `VotiumStrategyCore::applyRewards()`, and will deposit rewards in AfEth only if the manager address is defined:

```solidity
302:         if (address(manager) != address(0))
303:             IAfEth(manager).depositRewards{value: ethReceived}(ethReceived);
304:         else depositRewards(ethReceived);
```

It is important to note that if rewards aren't channeled through AfEth, the protocol will not receive any fees.

## [L-16] Protocol doesn't collect fees from SafEth

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L272

Protocol fees are only collected as part of the deposited rewards coming from the VotiumStrategy contract. Votium and Convex rewards are claimed and deposited back in the protocol as an explicit compound action.

On the other hand, SafEth accrues value by the passive appreciation of the underlying LSD tokens backing the protocol. There is no explicit process for claiming or compounding this increase in SafEth token value. The protocol isn't collecting fees from this side of the split.

It is not clear if this is by design or a potential oversight in the implementation.

## [L-17] Potential rounding to zero issue in AfEth deposit could cause loss of value

Deposits in the AfEth contract are split based on a configured `ratio`. One portion of the split goes to SafEth, while the other is deposited in the Votium strategy.

https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L148-L169

```solidity
148:     function deposit(uint256 _minout) external payable virtual {
149:         if (pauseDeposit) revert Paused();
150:         uint256 amount = msg.value;
151:         uint256 priceBeforeDeposit = price();
152:         uint256 totalValue;
153: 
154:         AbstractStrategy vStrategy = AbstractStrategy(vEthAddress);
155: 
156:         uint256 sValue = (amount * ratio) / 1e18;
157:         uint256 sMinted = sValue > 0
158:             ? ISafEth(SAF_ETH_ADDRESS).stake{value: sValue}(0)
159:             : 0;
160:         uint256 vValue = (amount * (1e18 - ratio)) / 1e18;
161:         uint256 vMinted = vValue > 0 ? vStrategy.deposit{value: vValue}() : 0;
162:         totalValue +=
163:             (sMinted * ISafEth(SAF_ETH_ADDRESS).approxPrice(true)) +
164:             (vMinted * vStrategy.price());
165:         if (totalValue == 0) revert FailedToDeposit();
166:         uint256 amountToMint = totalValue / priceBeforeDeposit;
167:         if (amountToMint < _minout) revert BelowMinOut();
168:         _mint(msg.sender, amountToMint);
169:     }
```

The amounts that go into each are calculated in lines 156 and 160. Both `sValue` and `vValue` calculations could be rounded down to zero if the numerator is lower than the denominator.

Even with a low `ratio` value, the amounts lost are negligible, hence the low severity.

For example, given a small `ratio` of 1% (i.e. `1e16`) we have:

```
amount * ratio < 1e18
amount < 1e18 / ratio
amount < 1e18 / 1e16
amount < 100
```

Amount should be lower than 100 wei in order to be rounded down to zero.

## [N-01] Remove debug symbols

- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L18

Remove any code related to debug functionality.

## [N-02] Missing event for important parameter change

Important parameter or configuration changes should trigger an event to allow being tracked off-chain.

*Instances (8)*:

- [AfEth::setStrategyAddress()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L81)
- [AfEth::setRatio()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L90)
- [AfEth::setFeeAddress()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L98)
- [AfEth::setProtocolFee()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L106)
- [AfEth::setPauseDeposit()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L116)
- [AfEth::setPauseWithdraw()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L124)
- [VotiumStrategyCore::setChainlinkCvxEthFeed()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L76)
- [VotiumStrategyCore::setRewarder()](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L125)

## [N-03] Unused constants

Unreferenced private constants in contracts can be removed.

*Instances (2)*:

- [AfEth::CVX_ADDRESS](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L50)
- [AfEth::VLCVX_ADDRESS](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L52)

## [N-04] Use constants for literal or magic values

Consider defining constants for literal or magic values as it improves readability and prevents duplication of config values.

*Instances (9)*:

- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L105
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L106
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L107
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L117
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L180
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L230
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L253
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L314
- https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L323

## [N-05] Missing check for zero address value in constructor or setter

Address parameters should be validated to guard against the default value `address(0)`.

*Instances (6)*:

- `vEthAddress` in [`AfEth::setStrategyAddress()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L81)
- `feeAddress` in [`AfEth::setFeeAddress()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L98)
- `rewarder` in [`VotiumStrategyCore::initialize()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L112)
- `manager` in [`VotiumStrategyCore::initialize()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L113)
- `chainlinkCvxEthFeed` in [`VotiumStrategyCore::setChainlinkCvxEthFeed()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L76)
- `rewarder` in [`VotiumStrategyCore::setRewarder()`](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L125)

## [I-01] Consider using the ERC4626 standard

Both AfEth and VotiumStrategy behave like vaults. They take deposits, mint ERC20 tokens in representation of this deposit, and have a withdraw function to recover the assets back.

This is exactly the use case of the [ERC4626 standard](https://eips.ethereum.org/EIPS/eip-4626). Consider using this standard to improve composability.

**[0xleastwood (Judge) commented](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/56#issuecomment-1746203991):**
 > L-06 & L-10 - `AbstractStrategy(vEthAddress).withdraw()` will revert if `withdrawIdToWithdrawRequestInfo[_withdrawId].withdrawn` holds true.

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/56#issuecomment-1747060202)**



***

# Gas Optimizations

For this audit, 4 reports were submitted by wardens detailing gas optimizations. The [report highlighted below](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/57) by **adriro** received the top score from the judge.

*The following wardens also submitted reports: [m\_Rassska](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/21), [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/68), and [rvierdiiev](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/6).*

## AfEth contract

- The `totalSupply()` function is called twice during the execution of `price()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L134  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L140  
  
- In `deposit()` line 162, there is no need to increase the variable `totalValue` since its previous value is zero. Consider changing this to an assignment.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L162  
  
- The variable `ratio` is read twice from storage in the function `deposit()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L156  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L160  

- In the `deposit()` function, the implementation recalculates the amount deposited by multiplying the collateral's minted tokens by their value. Instead of recalculating everything, simply use the original deposit amount (`msg.value`).  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L162-L165  
  
- The variable `latestWithdrawId` is read 7 times from storage in the function `requestWithdraw()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L175-L215  

- In `requestWithdraw()`, the implementation can simply burn the tokens from the holders, instead of first transferring them and then burning later in `withdraw()`.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L187

- In `requestWithdraw()`, the calculation of `votiumWithdrawAmount` can be simplified as `amount * votiumBalance / AfEthBalance`.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L190  
  
- In `requestWithdraw()`, the calculation of `safEthWithdrawAmount` can be simplified as `amount * safEthBalance / AfEthBalance`.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L197  
  
- The variable `vEthAddress` is read twice from storage in the function `requestWithdraw()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L221  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L225  
  
- The `withdrawTime` field from the `WithdrawInfo` struct is just used for informative purposes, but can be already derived from the Votium contract by referencing the `vEthWithdrawId` withdrawal. Consider removing this field as it is already stored elsewhere. This also allows the removal of the call to `withdrawTime()` which can be gas intensive.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L207  
  
- Avoid copying the `WithdrawInfo` struct to memory while reading `withdrawIdInfo[_withdrawId]` in the `withdraw()` function. Alias the variable to storage instead.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L249  
  
- The condition `canWithdraw()` in the `withdraw()` function is already checked by the VotiumStrategy contract when executing the withdrawal. Consider removing this duplication to save gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L288  
  
- The variable `vEthAddress` is read twice from storage in the function `depositRewards()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L273  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L285  
  
- The visibility of the `setRatio()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L90  
  
- The visibility of the `setFeeAddress()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L98  
  
- The visibility of the `setProtocolFee()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L106  
  
- The visibility of the `depositRewards()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L272  
  
## AbstractStrategy contract

- The `ReentrancyGuardUpgradeable` mixin is not used here. Consider removing it from the inheritance chain. A concrete strategy implementation can add it if needed.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/AbstractStrategy.sol#L10

## VotiumStrategy contract

- The variable `latestWithdrawId` is read 3 times from storage in the function `requestWithdraw()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L54-L103  

- In `requestWithdraw()`, the `cvxUnlockObligations` is read from storage in every iteration of the loop. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L80  
  
- In `requestWithdraw()`, the call to `ILockedCvx(VLCVX_ADDRESS).epochs(currentEpoch)` is constant in the loop and can be moved outside of it to execute it only once.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L81-L82  

- In `requestWithdraw()`, the call to `ILockedCvx(VLCVX_ADDRESS).rewardsDuration()` is constant in the loop and can be moved outside of it to execute it only once.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L86   
  
- The function `canWithdraw()` can be marked as `public` instead of `external` to allow `withdraw()` to call it internally instead of executing an external call.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L112  

- The variable `cvxUnlockObligations` is read twice from storage in the function `relock()`. Consider using a local variable as a cache to avoid multiple SLOAD to storage and reduce gas costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L142  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L143  

- The calculation in line 143 can be done using unchecked math since the overflow cannot happen due to the condition in line 142.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L143  

- The visibility of the `deposit()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L39  
  
- The visibility of the `requestWithdraw()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L54  

## VotiumStrategyCore contract

- The visibility of the `setChainlinkCvxEthFeed()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L76  

- The visibility of the `claimRewards()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L192  

- The visibility of the `withdrawStuckTokens()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L215  

- The visibility of the `applyRewards()` can be changed to `external`, as the function isn't used internally in the contract. This will save deployment costs.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L272  

- In `cvxInSystem()`, the implementation can use `ILockedCvx::lockedBalanceOf()` which just returns the total, instead of calling `lockedBalances()` which retrieves more unnecessary data, wasting more gas.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L134  
  
- In both `buyCvx()` and `sellCvx()`, the received output amount from the swap can be directly fetched from the result of the call to `exchange_underlying()`, which returns the `dy` amount. This will avoid querying the balances in the contract to calculate the difference of the received amount, saving gas.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L232-L241  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L255-L264  

- In `claimVlCvxRewards()`, using the ClaimZap will incur in a lot of overhead and wasted gas, since most the claims executed by the ClaimZap contract underneath will be skipped (we can the see the function call has lots of empty arrays/zero values). Instead of using the ClaimZap contract, simply call the `getReward()` of the vlCVX contract which would have the same effect, saving a lot of gas.  
  https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L323  

**[elmutt (Asymmetry) confirmed](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/57#issuecomment-1747066563)**



***
# Audit Analysis

For this audit, 3 analysis reports were submitted by wardens. An analysis report examines the codebase as a whole, providing observations and advice on such topics as architecture, mechanism, or approach. The [report highlighted below](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/32) by **m\_Rassska** received the top score from the judge.

*The following wardens also submitted reports: [adriro](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/69) and [d3e4](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/71).*

### Summary
> [Asymmetry Finance](https://www.asymmetry.finance/) provides an opportunity for stakers to diversify their staked eth across many liquid staking derivatives. It's not a doubt that the Lido has about 80% of the liquid staking market and Asymmetry Finance introduces a great solution to make the LSM more decentralized.

- The codebase provided for this audit introduces new mechanism under AfEth, which is by design collateralized by 2 underlying "strategy tokens" in an adjustable ratio:
  - safETH, which is the token over the LSD market (has been audited in previous audit)
  - vAfEth, which utilizes the rewards by locking cvx tokens and delegating the locked positions to the Votium in order for the Votium to decide the gauges to vote for.

### Approach being taken while auditing
- Understanding the codebase through the test coverage and provided docs
- Debugging (line by line) test cases that are pretty crucial in a system
- Generating Invariants with an intention to put the system into unexpected state
- Receiving the context needed from sponsors in order to avoid the future problems
- Testing locally && Reporting

### Architectural high-level overview
**Current flow used for deposits**
* If the image has not being rendered, see it [here](https://github.com/microsoft/vscode/assets/73281386/743d434f-7cb2-4a4c-9181-91cc2ff28d36)
  
![Deposit](https://user-images.githubusercontent.com/131902879/275635185-082468c2-3e83-413f-90bd-b064027a190b.png)

**Current flow used for withdrawals**
* If the image has not being rendered, see it [here](https://github.com/microsoft/vscode/assets/73281386/b397da6f-b7d7-410a-a40f-8a8d378cfe79)
  
![Withdrawals](https://user-images.githubusercontent.com/131902879/275635266-b2c8729a-88f0-4cab-9b7a-195bd71528c6.png)

### Technical overview
**Current flow used for deposits**
* Description:
  * Any user has an ability to deposit eth into AfETH manager. In response, AfEth mints corresponding amount of shares representing the stake being made by user. AfEth shares themselves are minted above the shares minted by underlying strategies based on the ratio. The initial ratio has to be setted at 7e17, meaning 70% of deposited eth will be staked in safETH and only 30% converts into VotiumStrategy shares. 

* Potential Improvements: 
  * The current flow with ratio mechanism included looks flexible enough to expand over new strategies in the future. However, the manual adjusment of the ratio seems not to be comfortable to deal with, since every single strategy also has its own deposit limits that could be changed in any time, especially when new strategies will be introduced. 

**Current flow used for withdrawals**
* Description:
  * Any user has an ability to withdraw eth from AfETH manager. In order to do that, the user first has to request withdrawal by providing his AfEth shares and second - withdraw the finalized request. In order to finalize the request, the system has to unstake the shares in each strategy currently being used in a system. Since, some strategies do not provide an instant unlock, the manager also has to wait that finalization period, before providing eth back to the user.

* Potential Improvements: 
  * The withdrawal queue being used by Lido to process withdrawals has a lot of interesting ideas to consider. I think, the current architecture behind afEth withdrawals could be improved, although it's decent enough. 

### Invariants Generated
* The following list includes some manually generated invariants failed, which have been used during an audit to report the issues:
  * `afETH should return the correct withdrawTimeBefore, when _amount in afETH has been provided` 
  * `chainlink oracle should return the fresh rates for the pair`
  * `CVX/ETH` pool manipulation should not provide any oppotunities for an adversory to exploit the system
  * `applyRewards()` has to be called right after claiming the votium rewards
  * `requestWithdraw()` should check whether the current balance can cover the request being submitted
  * upon requesting withdrawal, the system has to freeze it's balance to prevent double accounting. 

* Ofc, the list could contain some unsuccessfull scenarios being generated(>100), but at this point, it doesn't make any sense. 

### Centralization risks
* The AfEth manager itself is upgradable, which provides an ability for owner to update the logic behind the manager. 
* The percentage of the rewards could be setted up to 100%
* The withdrawals could be paused at any time by an owner.
* Many more, but users also should consider that the current configs setted by Asymmetry could also save the funds in case of any exploit and etc...
  
### Time Spent
* 70 hours over 7 days.



***

# [Mitigation Review](#mitigation-review)

## Introduction

Following the C4 audit, 3 wardens ([adriro](https://code4rena.com/@adriro), [d3e4](https://code4rena.com/@d3e4), and [m\_Rassska](https://code4rena.com/@m_Rassska)) reviewed the mitigations for all identified issues. Additional details can be found within the [C4 Asymmetry Mitigation Review repository](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings).

## Overview of Changes

**[Summary from the Sponsor](https://github.com/code-423n4/2023-10-asymmetry-mitigation#overview-of-changes):**

> "Most of the changes felt relatively straight forward. The biggest change we did was not burning afEth on withdraw, instead we now burn it on requestWithdraw. This is mostly in regards to H-04, but would like to have extra care taken around that to make sure nothing is broken."

## Mitigation Review Scope

### Individual PRs
| Mitigation of | Purpose | 
| ------------- | ----------- |
| H-01 | After days of research we decided that this was acceptable.  Check the link to view our response. | 
| H-02 | Don't withdraw zero from SafEth or Votium |
| H-03 | Validate Chainlink price data |
| H-04 | For this one we made afEth just burn on requestWithdraw |
| H-05 | For this one we locked down the depositRewards function and added a minout to the reward functions  |
| M-03 | Check if vlcvx contract is shutdown before trying to relock |
| M-04 | Check if available amount to withdraw is already in contract  |
| M-05 | Add a minimum epoch of 1 to not allow users to immediately withdraw |
| M-06 | Add a deadline check for deposit & withdraw |
| M-07 | Here we did two things, check msg.value instead of passing in amount & make deposit rewards private |
| M-08 | Track balances instead of using balanceOf |
| M-10 | No code changes needed, we removed the minimum stake amount from SafEth |

### Out of Scope
| URL | Mitigation of | Purpose | 
| ----------- | ------------- | ----------- |
| https://github.com/code-423n4/2023-09-asymmetry-findings/issues/55 | M-01 | Acknowledged and did not fix, plan to upgrade a fix in the future |
| https://github.com/code-423n4/2023-09-asymmetry-findings/issues/54 | M-02 | Did not fix, should have been marked acknowledged |
| https://github.com/code-423n4/2023-09-asymmetry-findings/issues/31 | M-09 | Didn't fix, should have been marked acknowledged |

## Mitigation Review Summary
| Original Issue | Status of Original Finding | Full Details |
| --- | --- | --- |
| H-01 | 🔴 Unmitigated | Reports from adriro ([1](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/26), [2](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45)) and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/12) |
| H-02 | 🟢 Mitigation Confirmed | Reports from  [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/5) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/28) |
| H-03 | 🟢 Mitigation Confirmed | Reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/6), [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/29), and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/14) |
| H-04 | 🟢 Mitigation Confirmed | Reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/7), [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/31), and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/15) |
| H-05 | 🟢 Mitigation Confirmed | Reports from [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/16) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/32)
| M-03 | 🟢 Mitigation Confirmed | Reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/18) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/36) |
| M-04 | 🟢 Mitigation Confirmed | Reports from [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/37) and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/19) |
| M-05 | 🔴 Unmitigated | Reports from [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/21), [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/9), and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/38) |
| M-06 | 🟢 Mitigation Confirmed | Reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/20) and [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/39) |
| M-07 | 🔴 Unmitigated | Reports from [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/40) and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/27) |
| M-08 | 🟢 Mitigation Confirmed | Reports from [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/41) and [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/24) |
| M-10 | 🟢 Mitigation Confirmed | Reports from [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/25), [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/44), and [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/33) |
  
**During the mitigation review, the wardens confirmed that all in-scope findings were mitigated except for H-01, M-05, & M-07. They also surfaced several new issues: 1 high severity, 9 medium severity, and [1 low severity](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/55). See below for additional details.**

## [H-01 Unmitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/26)
*Submitted by adriro ([1](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/26), [2](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45)) and also [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/12)*

**Original Issue:** https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62

**Lines of Code:** https://github.com/code-423n4/2023-09-asymmetry/blob/6b4867491350f8327d0ac4f496f263642cf3c1be/contracts/AfEth.sol#L148-L169

The sponsor has provided a detailed response in the following comment: https://github.com/code-423n4/2023-09-asymmetry-findings/issues/62#issuecomment-1760305328

In summary their analysis is:

- The conditions to expose the issue are unlikely, it needs a deviation of the intended ratio while also a deviation of the CVX/ETH Chainlink feed price.
- The sponsor was able to reproduce the issue, but for a maximum difference of 2% in the Chainlink CVX price, the difference between the target ratio and the real ratio needs to be large.

As the sponsor comments:

> Based on this analysis we think a 2% chainlink variance is an acceptable risk, even when the ratios are far apart.

However, the accepted risk is also justified by the introduction of a minimum delay in the withdrawal process:

> in another pull request we set a minimum withdraw time of 1 epoch so its impossible to instantly withdraw even if there are unlockable funds in the contract.

Given the error with the withdrawal delay in VotiumStrategy, detailed in [VotiumStrategy withdrawal can still be executed with minimal delay](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45), which still offers the possibility of depositing into the protocol with minimal exposure to CVX, the attack is still feasible and can be performed under the right circumstances. The assessment is that the issue is still present and it has not been mitigated.

***

## [M-07 Unmitigated](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/40)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/40), and also [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/27)*

**Original Issue:** https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38

The changes related to this issue are:

- Access control has been added to `AfEth::depositRewards()` using the `onlyVotiumOrRewarder` modifier. This function can now be called only by the rewarder or the VotiumStrategy.
- Access control has been added to `VotiumStrategy::depositRewards()` using the `onlyManager`. This function can now be called only by the manager role (AfEth).
- `AfEth::depositRewards()` now uses `msg.value` instead of receiving an `amount` parameter that might not match the sent `callvalue`.

This mitigates the issue, but there is still an edge related to an error introduced in the implementation of the `onlyManager` modifier.

```solidity
90:     modifier onlyManager() {
91:         if (address(manager) != address(0) && msg.sender != manager)
92:             revert NotManager();
93:         _;
94:     }
```

Access is still allowed if the `manager` address is uninitialized or has the default value of `address(0)`.

The issue is expanded in detail in [Manager authorization in VotiumStrategy still leaves room for unprotected access](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/51).

***

## [VotiumStrategy withdrawal can still be executed with minimal delay](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45), also found by [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/21) and m\_Rassska ([1](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/10), [2](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/9))*

**Severity: High**

Within the mitigation changes, the sponsor has introduced a minimum delay of one epoch for VotiumStrategy withdrawals, in order to mitigate different issues related to the exposure to CVX. The fix contains an edge case which could still be used to make deposits in AfEth with minimal exposure to CVX.

### Impact

Epochs in Convex are synchronized with Curve gauge epochs, which are weekly periods that start each Thursday at 00:00 UTC.

For example, at the time of writing the current epoch is 85. This epoch started at timestamp `1698278400`, which is Thursday Oct 18th at 00:00 UTC. The next epoch, 86, starts on Thursday Oct 25th at 00:00 UTC, which also marks the end of epoch 85.

One of the new changes in the updated code is the introduction of a minimum delay of one epoch in VotiumStrategy withdrawals. Even if the available CVX balance (CVX held by the contract plus any unlockable balance in Convex) is enough to cover the withdrawal, the request is delayed until the next epoch. Locked balances are still implemented as they were before, because any locked balance naturally implies waiting for at least the next epoch.

```solidity
78:         uint256 cvxAmount = (_amount * _priceInCvx) / 1e18;
79:         cvxUnlockObligations += cvxAmount;
80: 
81:         uint256 totalLockedBalancePlusUnlockable = unlockable +
82:             trackedCvxBalance;
83: 
84:         if (totalLockedBalancePlusUnlockable >= cvxUnlockObligations) {
85:             withdrawIdToWithdrawRequestInfo[
86:                 latestWithdrawId
87:             ] = WithdrawRequestInfo({
88:                 cvxOwed: cvxAmount,
89:                 withdrawn: false,
90:                 epoch: currentEpoch + 1,
91:                 owner: msg.sender
92:             });
93:             emit WithdrawRequest(msg.sender, cvxAmount, latestWithdrawId);
94: 
95:             return latestWithdrawId;
96:         }
```

If the current available CVX balance (`totalLockedBalancePlusUnlockable`) is enough to cover the withdrawal, the request is scheduled for the next epoch (`currentEpoch + 1`).

```solidity
098:         for (uint256 i = 0; i < lockedBalances.length; i++) {
099:             totalLockedBalancePlusUnlockable += lockedBalances[i].amount;
100:             // we found the epoch at which there is enough to unlock this position
101:             if (totalLockedBalancePlusUnlockable >= cvxUnlockObligations) {
102:                 (, uint32 currentEpochStartingTime) = ILockedCvx(VLCVX_ADDRESS)
103:                     .epochs(currentEpoch);
104:                 uint256 timeDifference = lockedBalances[i].unlockTime -
105:                     currentEpochStartingTime;
106:                 uint256 epochOffset = timeDifference /
107:                     ILockedCvx(VLCVX_ADDRESS).rewardsDuration();
108:                 uint256 withdrawEpoch = currentEpoch + epochOffset;
109:                 withdrawIdToWithdrawRequestInfo[
110:                     latestWithdrawId
111:                 ] = WithdrawRequestInfo({
112:                     cvxOwed: cvxAmount,
113:                     withdrawn: false,
114:                     epoch: withdrawEpoch,
115:                     owner: msg.sender
116:                 });
117: 
118:                 emit WithdrawRequest(msg.sender, cvxAmount, latestWithdrawId);
119:                 return latestWithdrawId;
120:             }
121:         }
```

If the first locked balance (`lockedBalances[0]`) corresponds to the next epoch, and the unlocked amount covers the requested amount (line 101), then the withdrawal will be scheduled for this next epoch (lines 109-116).

Now, as previously mentioned, epochs switch at the start of every Thursday. If we request a withdrawal at the very end of the current epoch, i.e. at most at Wednesday 11:59:59 PM UTC, then the withdrawal can potentially be scheduled for the next epoch which is only 1 second apart. This allows deposits in AfEth with minimal exposure to CVX.

A bad actor can use this to effectively deposit into AfEth, request a withdrawal, and withdraw with a minimum delay that can go as little as one second. As shown before, they would either need the funds to be unlockable (which sets the withdrawal for the next epoch) or to be unlocked in the next epoch.

As this issue still allows deposits with minimal exposure, the reward sandwiching attack and the intrinsic arbitrage due to price deviations are still feasible given the original scenarios. Given the error affects both original H-01 and M-05 issues, I'm assigning this issue a high severity.

### Proof of Concept

Let's say that the attacker makes a deposit such that `N` amount of CVX tokens are bought. To simplify the example, let's also say that the unlockable amount of tokens in Convex is greater than `N`. The attacker executes this at timestamp `1698278399`.

1.  The attacker deposits into AfEth. Their deposited share consists of `N` CVX tokens.
2.  The attack immediately requests a withdrawal. As the unlockable amount is enough to cover for the `N` tokens, the request is scheduled for the next epoch. Current epoch associated with timestamp `1698278399` is 85, which means the withdrawal is scheduled for epoch 86.
3.  The attacker waits one second, it is now `1698278400`.
4.  The attacker calls `withdraw()`, the epoch for the current timestamp `1698278400` is 86. Withdrawal is allowed and the attacker removes their share in the protocol.
5.  The attacker effectively executed the deposit and withdraw cycle with an exposure of just one second.

### Recommendation

The issue can be fixed by reconsidering potential requests to withdraw near the end of the period. For example, one potential solution will be to check if the current timestamp is after the second half of the current period. If so, then schedule the withdrawal not for the next epoch, but for the epoch after the next epoch (i.e. with a delay of two periods).

Note that this should be **considered in both scenarios**, when the current unlockable amount covers the withdrawal and when the locked balances are iterated to find the epoch that releases the needed funds.

A pseudo-code of the algorithm can summed as:

    1. Check if `totalLockedBalancePlusUnlockable >= cvxUnlockObligations`.
    2. If so, set `withdrawEpoch = currentEpoch + 1`.
    3. If not, loop through `lockedBalances`:
      3a. If `totalLockedBalancePlusUnlockable >= cvxUnlockObligations`, calculate epoch for `lockedBalances[i].unlockTime` and set that result as `withdrawEpoch`.
    4. If `withdrawEpoch` is not found, revert.
    5. If `withdrawEpoch.startTime - block.timestamp < epochDuration / 2`, then set `withdrawEpoch += 1`.
    6. Schedule withdrawal for `withdrawEpoch`.

**[toshiSat (Asymmetry) confirmed and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/45#issuecomment-1781860577):**
 > We will make the minEpoch to 2 starting out to prevent this.

***
 

## [Missing deadline for rewards](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/23)
*Submitted by [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/23)*

**Severity: Medium**

**Original Issue:** [M-06: Missing deadline check for AfEth actions](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/43)

The issue was missing deadline checks for deposits and withdrawals.

Deadline parameters have been added to `AfEth.deposit()` and `AfEth.withdraw()`. Since access to VotiumStrategy has been restricted to AfEth these checks are only needed in AfEth.
However, the same issue applies to rewards deposits, where no deadline has been added. The entry points are `AfEth.depositRewards()` and `VotiumStrategyCore.applyRewards()`, where deadlines should also be added.

**[toshiSat (Asymmetry) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/23#issuecomment-1781875941):**
 > Will add deadline checks for rewards, thank you.

***

## [Users loses their share of rewards while waiting for withdrawal](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47)*

**Severity: Medium**

Withdrawals in AfEth undergo a delay until the underlying CVX tokens can be withdrawn. Depositors need to request a withdrawal and wait until the required withdrawal epoch before making their withdrawal effective. During this period of time, they will lose their share of any compounded reward into SafEth.

### Impact

Withdrawals in AfEth are first requested using the `requestWithdraw()`. As we can see in the implementation, the function will calculate the number of SafEth tokens corresponding to the user's share (`safEthWithdrawAmount`) and store that information to be used when the withdrawal can be finally executed.

```solidity
222:         uint256 safEthBalance = safEthBalanceMinusPending();
223: 
224:         uint256 safEthWithdrawAmount = (withdrawRatio * safEthBalance) / 1e18;
225: 
226:         pendingSafEthWithdraws += safEthWithdrawAmount;
227: 
228:         withdrawIdInfo[latestWithdrawId]
229:             .safEthWithdrawAmount = safEthWithdrawAmount;
230:         withdrawIdInfo[latestWithdrawId]
231:             .votiumWithdrawAmount = votiumWithdrawAmount;
232:         withdrawIdInfo[latestWithdrawId].vEthWithdrawId = vEthWithdrawId;
233: 
234:         withdrawIdInfo[latestWithdrawId].owner = msg.sender;
235:         withdrawIdInfo[latestWithdrawId].amount = _amount;
236:         withdrawIdInfo[latestWithdrawId].withdrawTime = withdrawTimeBefore;
```

Line 224 calculates the amount based on the user's share and the total number of SafEth tokens held by the AfEth contract. This amount **gets fixed at this moment** and is stored in the `withdrawIdInfo` mapping, which is later used by `withdraw()` to send the funds to the user:

```solidity
284:         if (withdrawInfo.safEthWithdrawAmount > 0) {
285:             ISafEth(SAF_ETH_ADDRESS).unstake(
286:                 withdrawInfo.safEthWithdrawAmount,
287:                 0
288:             );
289:             trackedsafEthBalance -= withdrawInfo.safEthWithdrawAmount;
290:         }
```

Protocol rewards coming from the VotiumStrategy in AfEth are compounded back into the protocol. Depending on the current state of SafEth and vAfEth TVL, and the desired target ratio, rewards can be compounded into SafEth if the current ratio is below the target value, in order to increase the SafEth side. This can be seen in the implementation of `depositRewards()`:

```solidity
322:         uint256 safEthTvl = (ISafEth(SAF_ETH_ADDRESS).approxPrice(true) *
323:             safEthBalanceMinusPending()) / 1e18;
324:         uint256 votiumTvl = ((votiumStrategy.cvxPerVotium() *
325:             votiumStrategy.ethPerCvx(true)) * trackedvStrategyBalance) / 1e36;
326:         uint256 totalTvl = (safEthTvl + votiumTvl);
327:         uint256 safEthRatio = (safEthTvl * 1e18) / totalTvl;
328:         if (safEthRatio < ratio) {
329:             uint256 safEthReceived = ISafEth(SAF_ETH_ADDRESS).stake{
330:                 value: amount
331:             }(_safEthMinout);
332:             trackedsafEthBalance += safEthReceived;
333:         } else {
```

As we can see in lines 329-331, rewards compounded into SafEth are executed by staking into the protocol which **involves the minting of new SafEth tokens**.

Combining this with the fact that the user's share of SafEth tokens are calculated when the user requests the withdrawal, it means that any rewards earned and compounded into SafEth during the period of time the user is waiting to execute their withdrawal will be lost. As the user is still locked into the platform, and knowing that their share of CVX tokens are still generating revenue to the protocol, it is unfair that they don't receive their share of it.

This only happens with the SafEth strategy, as compounded rewards into SafEth increment the number of SafEth tokens the AfEth contract holds, but compounded rewards into the Votium strategy are directly considered by the held tokens (which behaves more like a vault).

Note that, with the new changes, even if the CVX balance is enough to cover the withdrawal, the request is delayed until the next epoch, which implies a minimum of one week. The user will be forced to wait at least one week, up to the maximum of 16 weeks imposed by Convex in the worst case, and will lose any rewards compounded into SafEth during this period.

### Proof of Concept

Let's assume a user owns 10% of the total share of AfEth. The current amount of SafEth tokens held in the AfEth contract is 1000. Let's also say that the target ratio is 50% and the current TVL of SafEth is 40% of the total.

1.  The user requests the withdrawal of all their owned AfEth. The protocol has 1000 SafEth tokens, hence their share is calculated at 100 tokens.
2.  The withdrawal is scheduled for a future epoch since it also needs to withdraw from Votium, let's say this is scheduled for 2 epochs (nearly two weeks).
3.  In between this period of time, rewards are applied in the protocol. Since the SafEth TVL is below the target ratio, rewards are compounded into the SafEth side. Let's say the total amount of new minted SafEth tokens is 100.
4.  The withdrawal period is finished and the user can finally withdraw. The user will receive 100 SafEth tokens, since this amount was calculated and fixed in step 1. The user doesn't receive the 10 tokens from their share of the applied rewards in step 3.

### Recommendation

The easiest path to solve the issue would be to delay the calculation of the user's SafEth share until the time the withdraw is made effective. This way any potential reward that is applied during the waiting time is considered in the resulting amount that is sent when `withdraw()` is called.

**[toshiSat (Asymmetry) confirmed, but disagreed with severity and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1781864765):**
 > Disagreeing with severity because 1 week of rewards in SafEth isn't really user losing their funds. I do think the issue is valid though.

**[adriro (Warden) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1781877666):**
 > @toshiSat - Note that 1 week is the minimum. Even a zero amount withdrawal will need to go through the current queue measured by `cvxUnlockObligations`. This can go as long as 16 weeks!
> 
> ```
> uint256 cvxAmount = (_amount * _priceInCvx) / 1e18;
> cvxUnlockObligations += cvxAmount; 
> ```

**[d3e4 (Warden) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1782134873):**
 > It shouldn't be taken for granted that one should get rewards past the withdrawal request. There is a cost to enter in the form of being locked in for up to 16 weeks. This cost associates well with also only receiving rewards while one is committed to the deposit, i.e. before requesting to withdraw it, rather than when the withdrawal is approved.

**[adriro (Warden) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1782234941):**
 > @d3e4 - After the request, the strategy still generates revenue using the user's share of CVX tokens:
> 
> > As the user is still locked into the platform, and knowing that their share of CVX tokens are still generating revenue to the protocol, it is unfair that they don't receive their share of it.

**[toshiSat (Asymmetry) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1789490870):**
 > @adriro - This is only true until their tokens are unlocked, once they are unlocked, theoretically, they could sit there forever gaining votium rewards and safEth rewards, this is actually more of an issue to me than the user not gaining safEth rewards.

**[adriro (Warden) commentedd](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1789734707):**
 >@toshiSat - Correct, it should be up to the unlock time. After that, the CVX portion won't be relocked which means no more revenue is coming from that share.

**[Rassska (Warden) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1806791637):**
 > I also don't think this is indeed an issue. Agree with @toshiSat and @d3e4. 
> 
> First, I recommend you to check out, how it's designed in Lido: 
> - https://hackmd.io/fVmtkj3vSkCUvVIJ1r6PcA#Withdrawal-queue-mechanics
> 
> The key moment here is that the rate is snapshotted at the time, when request is submitted, meaning that the rebase stops to occur. This is the cost taken for validator spin ups(i.e. it takes some time to pass the queue on CL). 
> 
> It's pretty clear that the users start to receive the rewards right after the moment they deposited, however, the capital efficiency of that deposit is still 0. Now, if the rate is determined at the finalization time(rebase still occurs), users are not incentivized to keep their capital locked in protocol. They can deposit/withdraw back and forth without losing anything. However, it badly impacts an APR, since the overall capital efficiency is being gamed. 
> 
> Personally, I don't recommend the following changes and it seems the sponsors are aware of drawbacks behind it.

**[adriro (Warden) commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1806827577):**
 >@Rassska - This is not about lido or Ethereum staking, it's convex.

**[0xleastwood (Judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/47#issuecomment-1819072431):**
 > Considering only rewards are impacted, I'm finding it hard to justify high severity. It is true that this is a core component of the protocol and this seems like poor capital efficiency. Instead others will simply accrue more rewards over this time frame, and they are not "lost", just diverted. 

***

## [Invalid operation in `withdrawStuckTokens()` will break CVX balance tracking in VotiumStrategy](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/49)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/49), also found by [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/48)*

**Severity: Medium**

The updated code for `withdrawStuckTokens()` contains an update to the `trackedCvxBalance` variable that will break CVX accounting in the VotiumStrategy contract, leading to multiple severe consequences.

### Impact

To mitigate a potential withdrawal of CVX tokens using `withdrawStuckTokens()`, the sponsor has updated the implementation to handle the special case of CVX.

```solidity
236:     function withdrawStuckTokens(address _token) public onlyOwner {
237:         uint256 tokenBalance = IERC20(_token).balanceOf(address(this));
238:         if (_token == CVX_ADDRESS) {
239:             if (tokenBalance <= trackedCvxBalance) revert InvalidAmount();
240:             tokenBalance -= trackedCvxBalance;
241:         }
242: 
243:         IERC20(_token).safeTransfer(msg.sender, tokenBalance);
244:         if (_token == CVX_ADDRESS) trackedCvxBalance -= tokenBalance;
245:     }
```

As we can see in the previous snippet of code, lines 238-241 handle the special case of CVX by subtracting the `trackedCvxBalance` (CVX owned by the protocol depositors) amount to the `tokenBalance` amount, to just remove the excess of CVX and avoid withdrawing protocol owned funds.

However, line 244 updates the `trackedCvxBalance` by subtracting the `tokenBalance` amount. This is wrong, as it is updating the tracked CVX balance by subtracting the excess of tokens.

This will completely break the internal CVX accounting, which tracks user deposits in VotiumStrategy. This is a core variable of the contract, which has impact in different places:

*   It is used to calculate `cvxInSystem()`, which also affects `cvxPerVotium()` **that is used to calculate deposits, withdrawals, and the price itself of vAfEth**.
*   The tracked balance is also used in `requestWithdraw()`, to calculate the withdrawal epoch based on the requested withdrawal amount.
*   In `relock()`, to calculate the amount of tokens which should be relocked in Convex.

In summary, any subtracted amount to `trackedCvxBalance` means decreasing the amount of CVX tokens owned by the depositors, which translates to loss of funds. As we can see in the next section, this can even lead to updating `trackedCvxBalance` to zero.

### Proof of Concept

Let's say that `trackedCvxBalance = 100`.

1.  An user (could be an attacker or anyone accidentally) donates 100 CVX tokens to the VotiumStrategy.
2.  The owner calls `withdrawStuckTokens(CVX)`.
3.  The balance is `tokenBalance = IERC20(_token).balanceOf(address(this)) = 200`.
4.  Because `_token == CVX_ADDRESS`, the implementation does `tokenBalance -= trackedCvxBalance` which results in `tokenBalance = 100`.
5.  100 tokens are transferred to the owner.
6.  Finally, the implementation executes `trackedCvxBalance -= tokenBalance`, leaving `trackedCvxBalance = 0`.

### Recommendation

The implementation of `withdrawStuckTokens()` should not update the `trackedCvxBalance`, only use it to calculate the excess of CVX tokens that can be transferred out of the contract.

```diff
    function withdrawStuckTokens(address _token) external onlyOwner {
        uint256 tokenBalance = IERC20(_token).balanceOf(address(this));
        if (_token == CVX_ADDRESS) {
            if (tokenBalance <= trackedCvxBalance) revert InvalidAmount();
            tokenBalance -= trackedCvxBalance;
        }

        IERC20(_token).safeTransfer(msg.sender, tokenBalance);
-       if (_token == CVX_ADDRESS) trackedCvxBalance -= tokenBalance;
    }
```

**[toshiSat (Asymmetry) confirmed](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/49#issuecomment-1781865636)**

**[0xLeastwood (Judge) decreased severity to Medium](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/49#issuecomment-1787091233)**


***

## [Price inflation by locking CVX on behalf of VotiumStrategy](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/50)
*Submitted by [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/50)*

**Severity: Medium**

The price of vAfEth can be inflated with severe rounding errors as a result.

### Proof of Concept

In VotiumStrategy the price of vAfEth is calculated by

```solidity
function cvxInSystem() public view returns (uint256) {
    uint256 total = ILockedCvx(VLCVX_ADDRESS).lockedBalanceOf(
        address(this)
    );
    return total + trackedCvxBalance;
}
```

```solidity
function cvxPerVotium() public view returns (uint256) {
    uint256 supply = totalSupply();
    uint256 totalCvx = cvxInSystem() - cvxUnlockObligations;
    if (supply == 0 || totalCvx == 0) return 1e18;
    return (totalCvx * 1e18) / supply;
}
```

```solidity
function price(bool _validate) external view override returns (uint256) {
    return (cvxPerVotium() * ethPerCvx(_validate)) / 1e18;
}
```

Making an initial deposit so that `supply == 1` and then making `total` very large will thus inflate the price.
`total` is [the amount locked in the CVX locker for VotiumStrategy](https://etherscan.io/address/0x72a19342e8f1838460ebfccef09f6585e32db86e#code#L1222). There is nothing that prevents an attacker from calling [`ILockedCvx(VLCVX_ADDRESS).lock(votiumStrategyAddress, ...)`](https://etherscan.io/address/0x72a19342e8f1838460ebfccef09f6585e32db86e#code#L1456) to lock funds on behalf of VotiumStrategy.
This is thus equivalent to donating underlying without increasing `supply`.

The price inflation in VotiumStrategy is felt by AfEth, which is where subsequent deposits would be made. The full standard attack would be to deposit in AfEth for 1 CVX locked (and around 1 safEth and 2 afEth minted), then lock e.g. 2e18 CVX on behalf of VotiumStrategy. The price of afEth would now be about 1e18, which causes rounding errors of up to about 1e18.

### Recommended mitigation steps

A possibility is to mitigate this just like [M-08: Inflation attack in VotiumStrategy](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/35), i.e. by keeping an internal accounting of locked balances. But I think the best would be to just make an initial deposit oneself on deployment. 1e9 wei should be sufficient. This prevents all kinds of inflation attacks.

**[toshiSat (Asymmetry) acknowledged, but disagreed with severity and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/50#issuecomment-1781866508):**
 > The plan was always to mint and lock afEth tokens on launch.

**[0xleastwood (Judge) decreased severity to Medium and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/50#issuecomment-1831508890):**
> The first depositor can arbitrarily set an unfair exchange rate for votium tokens. Because of this, we can get significant rounding loss that is up to the exchange rate the attacker set.
> 
> Ultimately, smaller depositors could be severely limited in participating, but I would not expect users to incorrectly set `_minout` and lose out this way. It would be from the perspective that this user is unable to participate without accepting some rounding loss which means they would set `_minout` to a value lower than what should be expected.

*Note: for full discussion, see the [original submission](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/50).*

***

## [Manager authorization in VotiumStrategy still leaves room for unprotected access](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/51)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/51), also found by [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/27) and [m\_Rassska](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/8)*

**Severity: Medium**

Access control has been added to the VotiumStrategy contract with the intention of restricting functionality only to AfEth. However, an error in the implementation still leaves the contract publicly accessible.

### Impact

In the updated codebase, the sponsor has introduced access control to the VotiumStrategy contract. This authorization is implemented in the `onlyManager` modifier.

```solidity
95:     modifier onlyManager() {
96:         if (address(manager) != address(0) && msg.sender != manager)
97:             revert NotManager();
98:         _;
99:     }
```

As we can see in line 96, the check is only executed if `manager` is different from `address(0)`. If this is not the case, then the check is not enforced and the revert in line 97 can never be triggered. This means that when `manager == address(0)` access is still granted for any caller.

This modifier has been added to `depositRewards()`, `deposit()`, `requestWithdraw()` and `withdraw()`, which means that potentially all these functions can still be publicly accessible.

This is particularly relevant in relation to issue [M-07](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/38) of the original report, as access control was introduced to mitigate this issue.

Note, additionally, that the current issue potentially affects [H-05](https://github.com/code-423n4/2023-09-asymmetry-findings/issues/23), since `depositRewards()` could still be publicly accessible and be used to purchase CVX with an arbitrary value for the `_cvxMinout` slippage parameter.

### Recommendation

Remove the condition to allow access if `manager` is `address(0)`. Additionally, check that `manager` is correctly initialized in `initialize()`.

```diff
    modifier onlyManager() {
-       if (address(manager) != address(0) && msg.sender != manager)
+       if (msg.sender != manager)
            revert NotManager();
        _;
    }
```

```diff
    function initialize(
        address _owner,
        address _rewarder,
        address _manager
    ) external initializer {
        bytes32 VotiumVoteDelegationId = 0x6376782e65746800000000000000000000000000000000000000000000000000;
        address DelegationRegistry = 0x469788fE6E9E9681C6ebF3bF78e7Fd26Fc015446;
        address votiumVoteProxyAddress = 0xde1E6A7ED0ad3F61D531a8a78E83CcDdbd6E0c49;
        ISnapshotDelegationRegistry(DelegationRegistry).setDelegate(
            VotiumVoteDelegationId,
            votiumVoteProxyAddress
        );
        rewarder = _rewarder;
+       require(_manager != address(0));
        manager = _manager;
        __ERC20_init("Votium AfEth Strategy", "vAfEth");
        _transferOwnership(_owner);
        chainlinkCvxEthFeed = AggregatorV3Interface(
            0xC9CbF687f43176B302F03f5e58470b77D07c61c6
        );
    }
```

**[toshiSat (Asymmetry) acknowledged and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/51#issuecomment-1781882582):**
 > We are going to block manager from being set to zero once set and manager will be set on launch of protocol.

***

## [AfEth withdrawals are delayed even if the vAfEth withdrawal amount is zero](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/52)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/52), also found by [d3e4](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/42)*

**Severity: Medium**

While zero amount withdrawals of SafEth have been prevented, the updated codebase still executes the withdrawal process for zero amount withdrawals of vAfEth, creating an unnecessary delay in AfEth withdrawals.

### Impact

In AfEth, the withdrawal process is initiated by requesting a withdrawal using `requestWithdraw()`. As we can see in the implementation, even if the resulting amount of vAfEth (`votiumWithdrawAmount`) is zero, the function still calls `VotiumStrategy::requestWithdraw()`

```solidity
214:         uint256 votiumWithdrawAmount = (withdrawRatio *
215:             trackedvStrategyBalance) / 1e18;
216:         uint256 withdrawTimeBefore = withdrawTime(votiumWithdrawAmount);
217:         uint256 vEthWithdrawId = AbstractStrategy(vEthAddress).requestWithdraw(
218:             votiumWithdrawAmount
219:         );
```

Drilling down into `VotiumStrategy::requestWithdraw()`, we can that even for a zero amount the request must undergo a delay

```solidity
78:         uint256 cvxAmount = (_amount * _priceInCvx) / 1e18;
79:         cvxUnlockObligations += cvxAmount;
80: 
81:         uint256 totalLockedBalancePlusUnlockable = unlockable +
82:             trackedCvxBalance;
83: 
84:         if (totalLockedBalancePlusUnlockable >= cvxUnlockObligations) {
85:             withdrawIdToWithdrawRequestInfo[
86:                 latestWithdrawId
87:             ] = WithdrawRequestInfo({
88:                 cvxOwed: cvxAmount,
89:                 withdrawn: false,
90:                 epoch: currentEpoch + 1,
91:                 owner: msg.sender
92:             });
93:             emit WithdrawRequest(msg.sender, cvxAmount, latestWithdrawId);
94: 
95:             return latestWithdrawId;
96:         }
```

The calculated `cvxAmount` in line 78 will be zero, and the rest of the algorithm is executed. This means that even if the amount is zero, the request will be delayed between 1 and 16 epochs, depending on the current queued withdrawals.

This means that a withdrawal in AfEth which consists only of SafEth will still need to be subject to an unnecessary delay due to the withdrawal request in VotiumStrategy.

### Recommendation

Avoid executing the withdrawal request in VotiumStrategy if `votiumWithdrawAmount` is zero.

**[elmutt (Asymmetry) acknowledged and commented](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/52#issuecomment-1785716886):**
 > We decided not to address this one as we never expect it to be 100% safEth and always expected that it needs to wait a full epoch anyway. Not a problem to us.

***

## [Safe approval could lead to a denial of service in VotiumStrategy](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/54)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/54)*

**Severity: Medium**

The introduction of the SafeERC20 wrapper may lead to an accidental denial of service due to how the `safeApprove()` function works internally.

### Impact

The updated codebase uses the [SafeERC20](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#SafeERC20) wrapper provided by the OpenZeppelin contracts library to handle ERC20 interaction in the VotiumStrategyCore contract. This was presumably added to provide safer support for the `applyRewards()` function, since this function needs to handle arbitrary tokens.

However, the SafeERC20 wrapper has been also applied as part of the CVX handling in the VotiumStrategyCore contract. This can be seen in the implementations of `depositRewards()` and `sellCvx()`:

```solidity
219:     function depositRewards(
220:         uint256 _amount,
221:         uint256 _cvxMinout
222:     ) public payable onlyManager {
223:         uint256 cvxAmount = buyCvx(_amount);
224:         if (cvxAmount < _cvxMinout) revert MinOut();
225:         IERC20(CVX_ADDRESS).safeApprove(VLCVX_ADDRESS, cvxAmount);
226:         ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmount, 0);
227:         trackedCvxBalance -= cvxAmount;
228:         emit DepositReward(cvxPerVotium(), _amount, cvxAmount);
229:     }
```

```solidity
276:     function sellCvx(
277:         uint256 _cvxAmountIn
278:     ) internal returns (uint256 ethAmountOut) {
279:         address CVX_ETH_CRV_POOL_ADDRESS = 0xB576491F1E6e5E62f1d8F26062Ee822B40B0E0d4;
280:         // cvx -> eth
281:         uint256 ethBalanceBefore = address(this).balance;
282:         IERC20(CVX_ADDRESS).safeApprove(CVX_ETH_CRV_POOL_ADDRESS, _cvxAmountIn);
283: 
284:         ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying(
285:             1,
286:             0,
287:             _cvxAmountIn,
288:             0 // this is handled at the afEth level
289:         );
290:         ethAmountOut = address(this).balance - ethBalanceBefore;
291:         trackedCvxBalance -= _cvxAmountIn;
292:     }
```

The implementation of `safeApprove()` has a very important detail which, if not correctly handled by the caller, may lead to an accidental denial of service.

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.9.2/contracts/token/ERC20/utils/SafeERC20.sol#L45-L54>

```solidity
function safeApprove(IERC20 token, address spender, uint256 value) internal {
    // safeApprove should only be called when setting an initial allowance,
    // or when resetting it to zero. To increase and decrease it, use
    // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
    require(
        (value == 0) || (token.allowance(address(this), spender) == 0),
        "SafeERC20: approve from non-zero to non-zero allowance"
    );
    _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
}
```

As pointed out in the comment present in the code, the function should only be called if `value` is zero or if the current allowance is zero, i.e. to reset it to zero or to assign it a new value when the current allowance is zero. Any other use case will fail the check, reverting the call.

While this is correctly handled in `applyRewards()`, as the implementation first resets the allowance to zero before assigning it the approval value, this is not the case for both `depositRewards()` and `sellCvx()`. As we can see in the previous snippets of code, lines 255 and 282 execute the call to `safeApprove()` without first resetting the value to zero.

This means that, for both cases, if the current allowance is not zero, the call to `safeApprove()` will fail, reverting the transaction and leading to a denial of service in `depositRewards()` and `sellCvx()`.

### Recommendation

Since CVX is a known token with a correct ERC20 implementation that adheres to the standard, it is not needed to use the SafeERC20 wrapper to handle approvals. Simple remove `safeApprove()` in favor of the standard `approve()`.

```diff
    function depositRewards(
        uint256 _amount,
        uint256 _cvxMinout
    ) public payable onlyManager {
        uint256 cvxAmount = buyCvx(_amount);
        if (cvxAmount < _cvxMinout) revert MinOut();
-       IERC20(CVX_ADDRESS).safeApprove(VLCVX_ADDRESS, cvxAmount);
+       IERC20(CVX_ADDRESS).approve(VLCVX_ADDRESS, cvxAmount);
        ILockedCvx(VLCVX_ADDRESS).lock(address(this), cvxAmount, 0);
        trackedCvxBalance -= cvxAmount;
        emit DepositReward(cvxPerVotium(), _amount, cvxAmount);
    }
```

```diff
    function sellCvx(
        uint256 _cvxAmountIn
    ) internal returns (uint256 ethAmountOut) {
        address CVX_ETH_CRV_POOL_ADDRESS = 0xB576491F1E6e5E62f1d8F26062Ee822B40B0E0d4;
        // cvx -> eth
        uint256 ethBalanceBefore = address(this).balance;
-       IERC20(CVX_ADDRESS).safeApprove(CVX_ETH_CRV_POOL_ADDRESS, _cvxAmountIn);
+       IERC20(CVX_ADDRESS).approve(CVX_ETH_CRV_POOL_ADDRESS, _cvxAmountIn);

        ICrvEthPool(CVX_ETH_CRV_POOL_ADDRESS).exchange_underlying(
            1,
            0,
            _cvxAmountIn,
            0 // this is handled at the afEth level
        );
        ethAmountOut = address(this).balance - ethBalanceBefore;
        trackedCvxBalance -= _cvxAmountIn;
    }
```

**[toshiSat (Asymmetry) confirmed](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/54#issuecomment-1781889214)**

***

## [CVX tracking misses to account for rewards](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/56)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/56)*

**Severity: Medium**

The updated codebase now tracks CVX balances internally. While this is correctly handled in most operations, accounting fails to consider CVX tokens coming from claimed rewards.

### Impact

CVX balances in the Votium strategy are now tracked internally. This is done by the introduction of a `trackedCvxBalance` variable that is updated whenever CVX is bought, sold or locked in Convex.

However, the implementation fails to consider potential CVX tokens coming from rewards. When claiming rewards from either Convex or Votium, CVX tokens might be transferred to the contract, and should be accounted for as part of `trackedCvxBalance`, since these are tokens owned by the protocol.

This wasn't an issue before, since CVX balance was simply queried on demand using `balanceOf()`. But with the introduction of custom tracking for CVX tokens, a failure to consider this scenario would mean not accounting these rewards as part of the owned CVX by the protocol.

### Recommendation

When claiming rewards in `claimRewards()`, account for any difference in CVX balance and add that to the `trackedCvxBalance` variable.

```diff
    function claimRewards(
        IVotiumMerkleStash.ClaimParam[] calldata _claimProofs
    ) public onlyRewarder {
+       uint256 cvxBalanceBefore = IERC20(CVX_ADDRESS).balanceOf(address(this));
        claimVotiumRewards(_claimProofs);
        claimVlCvxRewards();
+       uint256 cvxBalanceAfter = IERC20(CVX_ADDRESS).balanceOf(address(this));
+       trackedCvxBalance += cvxBalanceAfter - cvxBalanceBefore;
    }
```
**[toshiSat (Asymmetry) confirmed](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/56#issuecomment-1781892065)**

***

## [Rewarder should not be allowed to apply rewards on CVX tokens](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/58)
*Submitted by [adriro](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/58)*

**Severity: Medium**

The rewarder role should not be allowed to modify the balance of CVX tokens when applying rewards, otherwise the internal CVX balance tracking could get out of sync with major consequences for the protocol.

### Impact

The introduction of internal CVX balance tracking in the VotiumStrategy contract requires utmost care when handling token movements. Accounting should be done properly, as this essentially tracks the balance of CVX tokens that belong to depositors.

One of these sensitive areas is the `applyRewards()` function. This function is used by the rewarder role to swap arbitrary tokens from claimed rewards into ETH, in order to be compounded back into the protocol.

The current implementation of `applyRewards()` is extremely opaque. While the documentation says that the rewarder role will use the 0x protocol to process the swaps, the function executes arbitrary approvals and calls in its implementation, as can be seen in lines 320-323 and 325-327:

```solidity
301:     function applyRewards(
302:         SwapData[] calldata _swapsData,
303:         uint256 _safEthMinout,
304:         uint256 _cvxMinout
305:     ) public onlyRewarder {
306:         uint256 ethBalanceBefore = address(this).balance;
307:         for (uint256 i = 0; i < _swapsData.length; i++) {
308:             // Some tokens do not allow approval if allowance already exists
309:             uint256 allowance = IERC20(_swapsData[i].sellToken).allowance(
310:                 address(this),
311:                 address(_swapsData[i].spender)
312:             );
313:             if (allowance != type(uint256).max) {
314:                 if (allowance > 0) {
315:                     IERC20(_swapsData[i].sellToken).safeApprove(
316:                         address(_swapsData[i].spender),
317:                         0
318:                     );
319:                 }
320:                 IERC20(_swapsData[i].sellToken).safeApprove(
321:                     address(_swapsData[i].spender),
322:                     type(uint256).max
323:                 );
324:             }
325:             (bool success, ) = _swapsData[i].swapTarget.call(
326:                 _swapsData[i].swapCallData
327:             );
328:             if (!success) {
329:                 emit FailedToSell(_swapsData[i].sellToken);
330:             }
331:         }
332:         uint256 ethBalanceAfter = address(this).balance;
333:         uint256 ethReceived = ethBalanceAfter - ethBalanceBefore;
334: 
335:         if (address(manager) != address(0))
336:             IAfEth(manager).depositRewards{value: ethReceived}(
337:                 _safEthMinout,
338:                 _cvxMinout
339:             );
340:         else depositRewards(ethReceived, _cvxMinout);
341:     }
```

While this has its own problems, previously documented in issue M-02 of the original report, the introduction of the internal tracking of CVX brings a new issue vector. If CVX tokens are exchanged here, intentionally or accidentally, this would mean that the real CVX balance could potentially get out of sync with respect to the `trackedCvxBalance` variable.

The `trackedCvxBalance` variable plays a fundamental role in the new implementation of the VotiumStrategy contract, serving as an effective balance of held CVX by the contract. Any deviation of these values will cause issues to deposits, withdrawal and pricing, as this variable is involved in all these processes.

### Proof of Concept

1.  The rewarder calls `applyRewards()` by providing some swap data for 0x that swaps CVX tokens.
2.  CVX tokens are transferred out of the contract.
3.  The real balance of CVX gets out of sync with the value of `trackedCvxBalance`.

### Recommendation

In `applyRewards()`, after rewards have been swapped, verify that the current CVX balance is at least `trackedCvxBalance`. This will ensure CVX tokens are not removed from the contract.

```diff
    function applyRewards(
        SwapData[] calldata _swapsData,
        uint256 _safEthMinout,
        uint256 _cvxMinout
    ) public onlyRewarder {
        uint256 ethBalanceBefore = address(this).balance;
        for (uint256 i = 0; i < _swapsData.length; i++) {
            // Some tokens do not allow approval if allowance already exists
            uint256 allowance = IERC20(_swapsData[i].sellToken).allowance(
                address(this),
                address(_swapsData[i].spender)
            );
            if (allowance != type(uint256).max) {
                if (allowance > 0) {
                    IERC20(_swapsData[i].sellToken).safeApprove(
                        address(_swapsData[i].spender),
                        0
                    );
                }
                IERC20(_swapsData[i].sellToken).safeApprove(
                    address(_swapsData[i].spender),
                    type(uint256).max
                );
            }
            (bool success, ) = _swapsData[i].swapTarget.call(
                _swapsData[i].swapCallData
            );
            if (!success) {
                emit FailedToSell(_swapsData[i].sellToken);
            }
        }
        uint256 ethBalanceAfter = address(this).balance;
        uint256 ethReceived = ethBalanceAfter - ethBalanceBefore;

+       // Ensure CVX tokens are not removed
+       require(IERC20(CVX_ADDRESS).balanceOf(address(this)) >= trackedCvxBalance);

        if (address(manager) != address(0))
           IAfEth(manager).depositRewards{value: ethReceived}(
                _safEthMinout,
                _cvxMinout
            );
        else depositRewards(ethReceived, _cvxMinout);
    }
```

**[toshiSat (Asymmetry) confirmed](https://github.com/code-423n4/2023-10-asymmetry-mitigation-findings/issues/58#issuecomment-1781897862)**

***


# Disclosures

C4 is an open organization governed by participants in the community.

C4 Audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.
