## [M-1] Transferring tokens without tainting

## Description

- https://github.com/ConicFinance/protocol/blob/7a66d26ef84f93059a811a189655e17c11d95f5c/contracts/LpToken.sol#L74
- https://github.com/ConicFinance/protocol/blob/7a66d26ef84f93059a811a189655e17c11d95f5c/contracts/LpToken.sol#L81

If `_minimumTaintedTransferAmount` is large enough, then an attacker can do `deposit`/`withdraw` in a single transaction in small amounts.

For example:

- A hacker makes a deposit into `ConicPool`.
- Transfer several times with small amounts (less than `_minimumTaintedTransferAmount`) to another account.
- The hacker invokes `withdraw` from the new account.

This attack is unlikely, but if `_minimumTaintedTransferAmount` is large, it is a possibility.

## Recommendation

We recommend adding a max limit to the `_minimumTaintedTransferAmount`.

## [M-2] `Controller.updateWeights()` can set a total weight differing from one

## Description

When invoking `Controller.updateWeights()` or `BaseConicPool updateWeights()`, it's possible to pass in duplicate pools so that the sum of the weights in the provided list would equal one. However, the resulting sum of `BaseConicPool.weights` can be either greater or less than one.

The problem lies in the fact that when setting the weights, only the number of pools and the total weight of the provided list are checked, but the final weights in `BaseConicPool.weights` are not verified:

```
function updateWeights(
    PoolWeight[] memory poolWeights
    ) external onlyController {
    ...
    require(poolWeights.length == _pools.length(), "invalid pool weights");
    ...
    for (uint256 i; i < poolWeights.length; i++) {
        ...
        uint256 newWeight = poolWeights[i].weight;
        total += newWeight;
        ...
    require(total == ScaledMath.ONE, "weights do not sum to 1");
```

https://github.com/ConicFinance/protocol/blob/7a66d26ef84f93059a811a189655e17c11d95f5c/contracts/BaseConicPool.sol#L528

As a result, for example, if there are pools `A 25%, B 25%, C 25%, D 25%`, and the admin mistakenly calls function `updateWeights(A 70%, B 10%, B 10%, B 10%)`, both checks would pass:

```
require(poolWeights.length == _pools.length(), "invalid pool weights");
...
require(total == ScaledMath.ONE, "weights do not sum to 1");
```

Subsequently, the pool weights would become `(A 70%, B 10%, C 25%, D 25%)`, which totals 130%.

## Recommendation

It is recommended to check the invariant that the sum of `BaseConicPool.weights` is equal to one at the end of the function.
