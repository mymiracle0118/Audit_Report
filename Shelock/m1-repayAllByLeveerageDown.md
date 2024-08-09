

# Any full settlement made by `LeverageTransformer` is consistently reverted.

## Summary

In `AutoExit`, when `onlyFee` is true, the reward is calculated as `state.feeAmount0 * params.rewardX64 / Q64`. It's important to note that `state.feeAmount0` may include not only fees but also assets from decreased liquidity.

## Vulnerability Detail

https://github.com/code-423n4/2024-03-revert-lend/blob/main/src/automators/Automator.sol#L208

```javascript

    function _decreaseFullLiquidityAndCollect(
        uint256 tokenId,
        uint128 liquidity,
        uint256 amountRemoveMin0,
        uint256 amountRemoveMin1,
        uint256 deadline
    ) internal returns (uint256 amount0, uint256 amount1, uint256 feeAmount0, uint256 feeAmount1) {
        if (liquidity > 0) {
            // store in temporarely "misnamed" variables - see comment below
            (feeAmount0, feeAmount1) = nonfungiblePositionManager.decreaseLiquidity(
                INonfungiblePositionManager.DecreaseLiquidityParams(
                    tokenId, liquidity, amountRemoveMin0, amountRemoveMin1, deadline
                )
            );
        }
208     (amount0, amount1) = nonfungiblePositionManager.collect(
            INonfungiblePositionManager.CollectParams(tokenId, address(this), type(uint128).max, type(uint128).max)
        );

        // fee amount is what was collected additionally to liquidity amount
        feeAmount0 = amount0 - feeAmount0;
        feeAmount1 = amount1 - feeAmount1;
    }
```
As you can see in above, `feeAmount` is the amount of uncollected fees which does not contain assets from current liquidity. However, this contains the `owed` amount. The `owed` amount contains uncollected assets which are not only from `fee` but also from `decreaseLiquidity()` called before. If the owner called `decreaseLiquidity()` before, uncollected assets would contain some assets withdrawn from his liquidity, even most of them. This means that `onlyFee` configuration does not work well enough.

## Impact

The owner of NFT could pay more rewads to `AutoExit` than expected, when `onlyFee` is true.

## Proof of Concept

1. The owner calls `NPM.approve(address(autoExit), NFT)` and set `onlyFee` as true.
2. The owner calls `nonfungiblePositionManager.decreaseLiquidity()` and withdraw most of his liquidity.
3. An operator calls `autoExit.execute()` and takes more rewads than expected.

## Tool used

Manual Review

## Recommended Mitigation Steps

The `feeAmount` has to contain only the amount calculated from `feeGrowthInside` of UniswapV3 position.