# [High - 1] An attacker can steal the StabilityPool depositors profit

## Description

The liquidation flow of the protocol is supposed to be as follows:

- users open troves and join `StabilityPool`
- anyone calls the `liquidateTroves` function that iterates the given troves and liquidates them one by one
- `StabilityPool` depositors move collateral gains to their troves and increase the `StabilityPool` ThUSD balance by getting more ThUSD.

By using a flash loan any user can bypass the provision of liquidity to the protocol for a long time and steal some of the `StabilityPool` provider's profit taking the following steps:

1. Let's wait for liquidation opportunities. The following notions are to be introduced: Lsum is the total liquidatable amount of ThUSD and FLusd is the amount of ThUSD that can be accumulated after depositing flashloaned collateral to the protocol; FLfee is the fees the attacker should pay for opening a trove, SPusd is the total ThUSD amount in `StabilityPool`.
   The conditions for an attack are:
   Lsum < SPusd + FLusd
   FLfee < Lsum \* FLusd / SPusd
2. An attacker gets a flash loan and swaps the Lsum equivalent of the collateral token to ThUSD.
3. The attacker makes a deposit of all remaining collateral tokens to `StabilityPool`.
4. The attacker calls the [`TroveManager::liquidateTroves`](https://github.com/Threshold-USD/dev/blob/800c6c19e44628dfda3cecaea6eedcb498bf0bf3/packages/contracts/contracts/TroveManager.sol#L464) function to liquidate the troves. If the CR system is lower than 150%, the amount of liqudated troves can be significantly bigger.
5. The attacker calls [`withdrawFromSP`](https://github.com/Threshold-USD/dev/blob/800c6c19e44628dfda3cecaea6eedcb498bf0bf3/packages/contracts/contracts/StabilityPool.sol#L283) then ['withdrawCollateralGainToTrove'](https://github.com/Threshold-USD/dev/blob/800c6c19e44628dfda3cecaea6eedcb498bf0bf3/packages/contracts/contracts/StabilityPool.sol#L310) of `StabilityPool` to move collateral to the attacker's trove. The CR System here has to be more than 150%.
   The attacker's trove here contains the initial collateral and collateral gain.
6. The attacker closes the trove providing the rest of ThUSD plus the Lsum equivalent from step 1.
7. The attacker returns the flash loan.

## Recommendation

We recommend that you use the time factor to prevent flash loan attacks.

The attack's impact:

- loss of profit from liquidations by `StabilityPool` providers
- decreased motivation to use the Stability Pool which may cause Threshold USD to unpeg

# [Medium - 1] The `swap` function doesn't check insufficient liquidity

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L267-L270

## Description

It's common practice to check that the protocol has enough amount of tokens to make a swap. The minCollateralReturn parameter has to be used only for preventing sandwich attacks (price manipulation) but in BAMM smart contract this parameter is also used as a liquidity check.

Let's suppose that an attacker is the biggest depositor of the BAMM contract. The attacker can always front-run swap function calls and keep the given `minCollateralReturn` collateral amount on the contract. Calling the `swap` function with `minCollateralReturn` is a way to lose money even if the price has not been changed. As a result, users will always get `minCollateralReturn` tokens.

## Recommendation

We recommend adding a lack of liquidity check.

# [Medium - 2] The `Trade` function doesn't use `minCollateralAmount`

## Description

This `function` can be used by mistake by someone and according to the M-1 it leads to losing money because the `minCollateralAmount` parameter is always zero.

## Recommendation

We recommend adding a restriction that `trade` is called only by the Kyber protocol contracts.

# [Medium -3] The `Receive` function revert

## Description

The BAMM contract supports only one collateral token. If it is set to ERC20 tokens the `receive` function has to revert all ETH transfers to the contract.

## Recommendation

We recommend adding a check for the `collateralERC20` address and the sender's address.

# [Medium - 4] Privileges granted to accounts as `system contracts` cannot be revoked

## Description

Accounts can be marked as [`isTroveManager`, `isBorrowerOperations` and `isStabilityPool`](https://github.com/Threshold-USD/dev/blob/800c6c19e44628dfda3cecaea6eedcb498bf0bf3/packages/contracts/contracts/THUSDToken.sol#L58-L61). Such accounts have privileged access to some functionality: burn and arbitrary transfer between accounts. However, once granted, this privilege can't be revoked. If one of these accounts become compromised, there is no way to revoke its privileges.

## Recommendation

It is recommended to introduce functions to pause (temporarily disable)`TroveManager`, `BorrowerOperations` and `StabilityPool` privileges for specified accounts.
