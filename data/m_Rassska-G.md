# Table of contents

## Gas improvement submissions
- **[[G-01] Cache the state variables into the memory](#g-01-cache-the-state-variables-into-the-memory)**
- **[[G-02] Cache only attributes needed, when caching an object](#g-02-cache-only-attributes-needed-when-caching-an-object)**
- **[[G-03] Caching global variables is more expensive than using the actual variable (use `msg.sender/msg.value` instead of caching it](#g-03-caching-global-variables-is-more-expensive-than-using-the-actual-variable-use-msgsendermsgvalue-instead-of-caching-it)**
- **[[G-04] Use nested if statements instead of `&&`](#g-04-use-nested-if-statements-instead-of)**
- **[[G-05] Use Assembly To Check For `address(0)`](#g-05-use-assembly-to-check-for-address0)**
- **[[G-06] Internal/private functions only called once can be inlined to save gas](#g-06-internalprivate-functions-only-called-once-can-be-inlined-to-save-gas)**
- **[[G-07] With assembly, .call (bool success) transfer can be done gas-optimized](#g-07-with-assembly-call-bool-success-transfer-can-be-done-gas-optimized)**
- **[[G-08] Duplicated `require()/if()` checks should be refactored to a modifier or function](#g-08-duplicated-requireif-checks-should-be-refactored-to-a-modifier-or-function)**
- **[[G-09] Use a hardcoded address instead of `address(this)`](#g-09-use-a-hardcoded-address-instead-of-addressthis)**
- **[[G-10] `>=` costs less gas than ``>``](#g-10--costs-less-gas-than)**
- **[[G-11] Use assembly to emit events](#g-11-use-assembly-to-emit-events)**
- **[[G-12] Use constants instead of `type(uintX).max`](#g-12-use-constants-instead-of-typeuintxmax)**
- **[[G-13] Use assembly for setting stare variables](#g-13-use-assembly-for-setting-stare-variables)**

## **[G-01] Cache the state variables into the memory**<a name="G-01"></a>

### ***Description:***
- `mload` costs only 3 units of gas, `sload`(warm access) is about 100 units. Therefore, cache, when it's possible, otherwise - a lot of intrinsic gas wasted for users.


### ***Example of an occurance:***

- `ratio` and `SAF_ETH_ADDRESS` could be cached [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L148-L169):
- ``latestWithdrawId`` after an increment and ``vEthAddress`` could be cached [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L175-L215):
- use ``withdrawInfo.amount`` [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L255):
- ``vEthAddress``, ``SAF_ETH_ADDRESS`` could be cached [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L272-L293):
- ``VLCVX_ADDRESS``  could be cached [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L39-L46):
- ``VLCVX_ADDDRESS``, ``latestWithdrawId``, ``cvxUnlockObligations`` could be cached outside of the loop [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L54-L103):
- ``cvxUnlockObligations + cvxAmount`` could be cached before the loop [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L187-L190):
- `VLCVX_ADDRESS`, `CVX_ADDRESS` could be cached [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L135-L149):

</br>

## **[G-02] Cache only attributes needed, when caching an object**<a name="G-02"></a>

### ***Description:***
- When caching the struct behind the ref, do not cache the whole struct, if you don't need an access to the whole struct. 


### ***Example of an occurance:***

- Only 3 of 5 attributes are being used, when caching `withdrawInfo`, it means we have 2 sloads which incur an overhead [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L243-L265): 


</br>

## **[G-03]  Caching global variables is more expensive than using the actual variable (use `msg.sender/msg.value` instead of caching it**<a name="G-03"></a>

### ***Example of an occurance:***

- There is no need to store `msg.value` [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L148-L169):


</br>

## **[G-04] Use nested if statements instead of `&&`**<a name="G-04"></a>
### ***Description:***
- If the “if” statement has a logical “AND” and is not followed by an “else” statement, it can be replaced with 2 if statements.
  
### ***Example of an occurance:***

- Could be optimized [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L173-L181):


</br>

## **[G-05] Use Assembly To Check For `address(0)`**<a name="G-05"></a>
### ***Description:***
- It’s generally more gas-efficient to use assembly to check for a zero address (address(0)) than to use the == operator.

- The reason for this is that the == operator generates additional instructions in the EVM bytecode, which can increase the gas cost of your contract. By using assembly, you can perform the zero address check more efficiently and reduce the overall gas cost of your contract.

- Here’s an example of how you can use assembly to check for a zero address:
  - ```Solidity
      contract MyContract {
        function isZeroAddress(address addr) public pure returns (bool) {
            uint256 addrInt = uint256(addr);
            
            assembly {
                // Load the zero address into memory
                let zero := mload(0x00)
                
                // Compare the address to the zero address
                let isZero := eq(addrInt, zero)
                
                // Return the result
                mstore(0x00, isZero)
                return(0, 0x20)
            }
        }
      }
    ```
- In the above example, we have a function isZeroAddress that takes an address as input and returns a boolean value, indicating whether the address is equal to the zero address. Inside the function, we convert the address to an integer using uint256(addr), and then use assembly to compare the integer to the zero address.

- By using assembly to perform the zero address check, we can make our code more gas-efficient and reduce the overall cost of our contract. It’s important to note that assembly can be more difficult to read and maintain than Solidity code, so it should be used with caution and only when necessary
  
### ***Example of an occurance:***

- Could be used [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L302):


</br>

## **[G-06] Internal/private functions only called once can be inlined to save gas**<a name="G-06"></a>
  
### ***Example of an occurance:***

- `claimVLCVxRewards()` and `claimVotiumRewards()` could be inlined during rewards claiming [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L192-L197):


</br>

## **[G-07] With assembly, .call (bool success) transfer can be done gas-optimized**<a name="G-07"></a>
### ***Description:***
- return data (bool success,) has to be stored due to EVM architecture, but in a usage like below, ‘out’ and ‘outsize’ values are given (0,0), this storage disappears and gas optimization is provided.
  
- Ref: https://twitter.com/pashovkrum/status/1607024043718316032?t=xs30iD6ORWtE2bTTYsCFIQ&s=19
  
### ***Example of an occurance:***

- Could be optimized [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L263-L264):
- Could be optimized [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L277-L278):
- Could be optimized [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategy.sol#L127-L128):


</br>

## **[G-08] Duplicated `require()/if()` checks should be refactored to a modifier or function**<a name="G-08"></a>
### ***Description:***
- Sign modifiers or functions can make your code more gas-efficient by reducing the overall number of operations that need to be executed. For example, if you have a complex validation check that involves multiple operations and you refactor it into a function, then the function can be executed with a single opcode, rather than having to execute each operation separately in multiple locations.
  
### ***Example of an occurance:***

- Could be optimized [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L177):


</br>

## **[G-09] Use a hardcoded address instead of address(this)**<a name="G-09"></a>
### ***Description:***
- It can be more gas-efficient to use a hardcoded address instead of the address(this) expression, especially if you need to use the same address multiple times in your contract.

The reason for this is that using address(this) requires an additional EXTCODESIZE operation to retrieve the contract’s address from its bytecode, which can increase the gas cost of your contract. By pre-calculating and using a hardcoded address, you can avoid this additional operation and reduce the overall gas cost of your contract.

Here’s the ref, how to procompute it: https://book.getfoundry.sh/reference/forge-std/compute-create-address
  
### ***Example of an occurance:***

- Could be optimized as an example [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L187):


</br>

## **[G-10] `>=` costs less gas than ``>``**<a name="G-10"></a>
### ***Description:***
- The compiler uses opcodes GT and ISZERO for solidity code that uses >, but only requires LT for >=, which saves 3 gas.
  
### ***Example of an occurance:***

- Could be optimize as an example by adding `if (amountToMint <= _minout + 1)`[here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L167):


</br>

## **[G-11] Use assembly to emit events**<a name="G-11"></a>
### ***Description:***
- We can use assembly to emit events efficiently by utilizing scratch space and the free memory pointer. This will allow us to potentially avoid memory expansion costs. Note: In order to do this optimization safely, we will need to cache and restore the free memory pointer.

- For example, for a generic emit event for `eventSentAmountExample`:
- ```Solidity
      // uint256 id, uint256 value, uint256 amount
      emit eventSentAmountExample(id, value, amount);
  ```
- We can use the following assembly emit events:
  - ```Solidity
       assembly {
            let memptr := mload(0x40)
            mstore(0x00, calldataload(0x44))
            mstore(0x20, calldataload(0xa4))
            mstore(0x40, amount)
            log1(
                0x00,
                0x60,
                // keccak256("eventSentAmountExample(uint256,uint256,uint256)")
                0xa622cf392588fbf2cd020ff96b2f4ebd9c76d7a4bc7f3e6b2f18012312e76bc3
            )
            mstore(0x40, memptr)
        }
    ```
  
### ***Example of an occurance:***

- Could be optimized [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L209-L214):


</br>


## **[G-12] Use constants instead of `type(uintX).max`**<a name="G-12"></a>
### ***Description:***
- It’s generally more gas-efficient to use constants instead of type(uintX).max when you need to set the maximum value of an unsigned integer type.

The reason for this, is that the type(uintX).max expression involves a computation at runtime, whereas a constant is evaluated at compile-time. This means, that using type(uintX).max can result in additional gas costs for each transaction that involves the expression.

By using a constant instead of type(uintX).max, you can avoid these additional gas costs and make your code more efficient.

  
### ***Example of an occurance:***

- Could be optimize as an example [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/strategies/votium/VotiumStrategyCore.sol#L280):


</br>

## **[G-13] Use assembly for setting stare variables**<a name="G-13"></a>

### ***Example of an occurance:***

- Could be optimize as an example [here](https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L72-L126):


