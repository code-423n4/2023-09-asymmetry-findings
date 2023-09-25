## G-01. 
https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L160
Calculate `vValue` in such way: `uint256 vValue = amount - sValue;` and avoid additional arithmetical operations.

## G-02. 
https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L162-L164
Don't use `+=` as `totalValue` is always 0.
Change to:
```
totalValue =
            (sMinted * ISafEth(SAF_ETH_ADDRESS).approxPrice(true)) +
            (vMinted * vStrategy.price());
```