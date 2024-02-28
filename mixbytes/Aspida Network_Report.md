## [H-1] Staking deposit can be stolen by a 3rd party

## Description

The specification of ETH2.0 staking allows for two types of deposits: the initial deposit and the top-up deposit, which increases the balance of a previously made initial deposit. Unfortunately, the current implementation of the mainnet deposit contract does not sufficiently distinguish between these types of deposits. This oversight allows an attacker to front-run Aspida's deposit with their own initial deposit, causing Aspida's deposit to be treated as a top-up of the attacker's deposit. Consequently, Aspida's withdrawal credentials will be ignored, and the assets will be accounted for on behalf of the attacker.

This issue is classified as `critial` since it can lead to permanent loss of project liquidity.

## Recommendation

Despite this being an architectural issue of ETH2.0 itself, we recommend applying a workaround fix for this issue.

```
        bytes32 actualRoot = depositContract.get_deposit_root();
        if (expectedDepositRoot != actualRoot) {
            revert InvalidDepositRoot(actualRoot);
        }
```

The `expected deposit root` can be calculated using appropriate offchain mechanics, ensuring that no initial deposit was made to the given pubkey at the time of the `expected deposit root` calculation.

## [H-2] Excessive minting `aETH` permissions for strategies

## Description

The issue is identified in the `CorePrimary.strategyMinting` function.
Currently, the function, which can only be invoked by a `strategy`, permits the minting of an unrestricted amounts of `AETH` to the arbitrary address. It presents a significant risk of overinflating the circulating supply of `AETH`, potentially leading to a scenario where `AETH` is no longer backed on a one-to-one basis with the `ETH` locked in the project.

The issue is classified as `high` due to the excessive authority granted to the `strategy` in regulating the circulating supply of `AETH`.

Related code - strategyMinting: https://github.com/aspidanet/aspida-contract/blob/a94928e27a72baabb8c73b42634350ca934b3353/contracts/CorePrimary.sol#L257

## Recommendation

We recommend removing this function or adding the constraints to ensure that the minted `AETH` amounts are backed by an equivalent `ETH` collateral on a one-to-one basis.

## [H-3] Actions limits manipulation

## Description

An attacker is able to manipulate submit/withdraw limits by doing cycles of submit
https://github.com/aspidanet/aspida-contract/blob/a94928e27a72baabb8c73b42634350ca934b3353/contracts/CorePrimary.sol#L275
and withdraw
https://github.com/aspidanet/aspida-contract/blob/a94928e27a72baabb8c73b42634350ca934b3353/contracts/CorePrimary.sol#L291
which leads to disabling these functions for the next users.

## Recommendation

We recommend redesigning the logic to mitigate the risk of action lock-ups.
