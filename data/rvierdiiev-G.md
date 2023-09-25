## G-01. 
https://github.com/code-423n4/2023-09-asymmetry/blob/main/contracts/AfEth.sol#L160
Calculate `vValue` in such way: `uint256 vValue = amount - sValue;` and avoid additional arithmetical operations.