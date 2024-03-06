# [H-1] Multipool#getAmountOut has a hardcoded feeTier which can cause the function to revert
https://github.com/sherlock-audit/2023-06-real-wagmi-judging/issues/96

## Summary
In the Multipool#getAmountOut function, the feeTier has been hardcoded as 500. If a poolAddress for that specific feeTier doesn't exist, the whole function will revert.

## Vulnerability Detail
The code in question below has the feeTier hardcoded to 500, which could lead to a scenario where the pool at index 500 does not exist, leading to an attempted read of an empty address. This causes the function to revert. It could also limit the function's usefulness, as it can't interact with other pools in the underlyingTrustedPools array.

## Impact
The hardcoding of the feeTier limits the flexibility and robustness of the getAmountOut function. If the pool at index 500 does not exist or is not the desired pool for interaction, the function will fail or give inappropriate results. This could affect any functionality in the contract that relies on this function.

## Code Snippet
https://github.com/sherlock-audit/2023-06-real-wagmi/blob/82a234a5c2c1fc1921c63265a9349b71d84675c4/concentrator/contracts/Multipool.sol#L823

```javascript
(int56[] memory tickCumulatives, ) = IUniswapV3Pool(underlyingTrustedPools[500].poolAddress).observe(secondsAgo);
```
## Tool used
Manual Review

## Recommendation
We recommend removing the hardcoding of the feeTier value. Instead, this value should be passed as a parameter to the getAmountOut function. This would allow the function to interact with any pool in the underlyingTrustedPools array, increasing the flexibility of the function and ensuring it won't fail if the pool at index 500 does not exist. It is also important to ensure proper validation of the feeTier input to avoid invalid index access.

# [H-2] Insufficient lpAmount Validation Could Lead to Stuck Funds During Significant Price Movements.
https://github.com/sherlock-audit/2023-06-real-wagmi-judging/issues/100


Insufficient lpAmount Validation Could Lead to Stuck Funds During Significant Price Movements.
Summary
The vulnerability is caused by the possibility of the estimated lpAmount being greater than or equal to the user's shares due to significant price movements, which could lead to users' funds being stuck and inaccessible.

## Vulnerability Detail
The vulnerability arises when estimating the lpAmount using the _estimateWithdrawalLp function in both the withdraw and deposit functions. The calculated lpAmount is then used to update the user's shares. However, if a significant price movement occurs, there is no guarantee that the lpAmount will always be less than the user's shares.

If the lpAmount is greater than or equal to the user's shares, users' funds may become stuck and inaccessible. This issue is especially concerning during extreme market fluctuations when users might need to withdraw their funds urgently.

Impact
The impact of this vulnerability is high, as it could prevent users from withdrawing or depositing their funds during significant price movements. Moreover, it could lead to a loss of funds for affected users.

## Code Snippet
In the withdraw function:
https://github.com/sherlock-audit/2023-06-real-wagmi/blob/82a234a5c2c1fc1921c63265a9349b71d84675c4/concentrator/contracts/Dispatcher.sol#L234

```javascript
{
    (uint256 fee0, uint256 fee1) = _calcFees(feesGrow, user);
    lpAmount = _estimateWithdrawalLp(reserve0, reserve1, _totalSupply, fee0, fee1);
}
user.shares -= lpAmount;
```
In the deposit function:
https://github.com/sherlock-audit/2023-06-real-wagmi/blob/82a234a5c2c1fc1921c63265a9349b71d84675c4/concentrator/contracts/Dispatcher.sol#L193
```javascript
{
    (uint256 fee0, uint256 fee1) = _calcFees(feesGrow, user);
    lpAmount = _estimateWithdrawalLp(reserve0, reserve1, _totalSupply, fee0, fee1);
}
user.shares -= lpAmount;
```
## Tool used
Manual Review

## Recommendation
Modify the _estimateWithdrawalLp function to account for price movements by adding a priceFactor parameter.
```javascript
function _estimateWithdrawalLp(
    uint256 reserve0,
    uint256 reserve1,
    uint256 _totalSupply,
    uint256 amount0,
    uint256 amount1,
    uint256 priceFactor
) private pure returns (uint256 shareAmount) {
    shareAmount =
        ((amount0 * _totalSupply) / (reserve0 * priceFactor) + (amount1 * _totalSupply) / reserve1) /
        2;
}
```
Update the withdraw and deposit functions to include the priceFactor when calling _estimateWithdrawalLp.
// In the withdraw function:
```javascript
{
    (uint256 fee0, uint256 fee1) = _calcFees(feesGrow, user);
    lpAmount = _estimateWithdrawalLp(reserve0, reserve1, _totalSupply, fee0, fee1, priceFactor);
}

// In the deposit function:
{
    (uint256 fee0, uint256 fee1) = _calcFees(feesGrow, user);
    lpAmount = _estimateWithdrawalLp(reserve0, reserve1, _totalSupply, fee0, fee1, priceFactor);
}
```
In both the withdraw and deposit functions, add a loop to adjust the priceFactor until a valid lpAmount is found, ensuring that users can withdraw their funds even in cases of extreme market fluctuations.
// In the withdraw function:
```javascript
uint256 priceFactor = 1;
while (lpAmount >= user.shares) {
    priceFactor++;
    (uint256 fee0, uint256 fee1) = _calcFees(feesGrow, user);
    lpAmount = _estimateWithdrawalLp(reserve0, reserve1, _totalSupply, fee0, fee1, priceFactor);
}
 change
// In the deposit function:
uint256 priceFactor = 1;
while (lpAmount >= user.shares) {
    priceFactor++;
    (uint256 fee0, uint256 fee1) = _calcFees(feesGrow, user);
    lpAmount = _estimateWithdrawalLp(reserve0, reserve1, _totalSupply, fee0, fee1, priceFactor);
}
```
this will allow users to withdraw their funds even in cases of extreme market fluctuations.

# [H-3] Deposit transactions lose funds to front-running when multiple fee tiers are available
https://github.com/sherlock-audit/2023-06-real-wagmi-judging/issues/105 
## Summary
Deposit transactions lose funds to front-running when multiple fee tiers are available

## Vulnerability Detail
The deposit transaction takes in minimum parameters for amount0 and amount1 of tokens that the user wishes to deposit, but no parameter for the minimum number of LP tokens the user expects to receive. A malicious actor can limit the number of LP tokens that the user receives in the following way:

A user Alice submits a transaction to deposit tokens into Multipool where (amount0Desired, amount0Min) > (amount1Desired, amount1Min)

A malicious actor Bob can front-run this transaction if there are multiple feeTiers:

* by first moving the price of feeTier1 to make tokenA very cheap (lots of tokenA in the pool)
* then moving the price of feeTier2 in opposite direction to make tokenB very cheap (lots of tokenB in the pool)
  
This results in reserves being balanced accross feeTiers, and the amounts resulting from _optimizeAmounts are balanced as well:
https://github.com/sherlock-audit/2023-06-real-wagmi/blob/82a234a5c2c1fc1921c63265a9349b71d84675c4/concentrator/contracts/Multipool.sol#L780-L808

So the minimum amounts checks pass and but results as less LP tokens minted, because even though the reserves are balanced, they are also overinflated due to the large swap, and the ratio:
https://github.com/sherlock-audit/2023-06-real-wagmi/blob/82a234a5c2c1fc1921c63265a9349b71d84675c4/concentrator/contracts/Multipool.sol#L468-L470

becomes a lot smaller than before the large swap

## Impact
The user loses funds as a result of maximum slippage.

## Code Snippet
## Tool used
Manual Review

## Recommendation
Add extra parameters for the minimum number of LP tokens that the user expects, instead of just checking non-zero amount:
https://github.com/sherlock-audit/2023-06-real-wagmi/blob/82a234a5c2c1fc1921c63265a9349b71d84675c4/concentrator/contracts/Multipool.sol#L473