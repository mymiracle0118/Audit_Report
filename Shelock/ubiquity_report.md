# When mint/redeem Ubiquity dollar token, the amount of collateral token is calculated incorrectly.

## Summary
In `mintDollar` and `redeemDollar` functions of `LibUbiquityPool.sol` contract, the amount of collateral token is not calculated correctly based on the price of dollar token.

## Severity
High

## Vulnerability Detail
To mint dollar token, an account should send its some collateral token to the ubiquity pool. The amount of collateral token to be sent will be decided based on the value of dollar token to be minted. And the value should be decided based on the minting amount and the current price of dollar token.
But in this codebase, they calculate needed collateral amount based on only amount of dollar token.

```solidity
        // get amount of collateral for minting Dollars
        collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount);
```

```solidity
    function getDollarInCollateral(
        uint256 collateralIndex,
        uint256 dollarAmount
    ) internal view returns (uint256) {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
        return
            dollarAmount
                .mul(UBIQUITY_POOL_PRICE_PRECISION)
                .div(10 ** poolStorage.missingDecimals[collateralIndex])
                .div(poolStorage.collateralPrices[collateralIndex]);
    }
```
Therefore, needed collateral amount is calculated with non-intentional assumption that dollar price is $1. This can led to the result that account would send less collateral token than expected if `mintPriceThreshold` is higher than $1(This is usual case).
In `redeemDollar` function, there is the same issue. In this case, an account may receive more collateral token than expected.

## Impact
When mint dollar token, an account may transfer less collateral token than expected. In otherwise, when redeem dollar token, an account may receive more collateral token than expected.

## Tool used
Manual Review

## Recommendation
Calculate the amount of needed collateral based on the amount and price of dollar token
```diff
    priceOfDollar = getDollarPriceUsd();
-   collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount);
+   collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount * priceOfDollar)
```
