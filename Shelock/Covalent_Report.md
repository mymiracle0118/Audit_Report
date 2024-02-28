# [M-1] Validator cannot set new address if more than 300 unstakes in it's array

## Vulnerability Detail

If a validator has more than 300 accumulated unstakes associated with it, then it cannot set a new address for itself. The only way to decrease the length of the `Unstaking` array is through the `setValidatorAddress()` function, but will revert if it's array is longer than 300 entries. A malicious delegator could stake 1,000 tokens, and then unstake 301 times with small amounts to fill up the unstaking array, and there is no way to remove those entries from the array. Every time `_unstake()` is called, it pushes another entry to it's array for that validator.

A malicious validator could also set it's new address to another validator, forcing a merge of information to the victim validator. The malicious validator can do this even when it's disabled with 0 tokens. So it could get to 300 length, and then send all those unstakes into another victim validators unstaking array. This is the same kind of attack vector, but should be noted that validators can assign their address to other validators, effectively creating a forceful merging of them.

The README.md suggests there is a mechanism to counteract this: "In case if there are more than 300 unstakings, there is an option to transfer the address without unstakings." But there appears to be no function in scope that can transfer the address of the validator without unstakings, or any other function that can reduce the unstakings array at all.

## Impact

Validator can be permanently stuck with same address if there are too many entries in it's `Unstaking` array.

## Code Snippet

Validator & Unstaking Structs: https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L35-#L51 `setValidatorAddress()` length check: https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L702 deletion of array inside `setValidatorAddress()`: https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L707

## Tool used

Manual Review

## Recommendation

Consider having a way to set a new address without unstakings, or allow for smaller batches to be transferred during an address change if there are too many. Consider disallowing validators from changing their address to other validators.

# [M-2] New staking between reward epochs will dilute rewards for existing stakers. Anyone can then front-run OperationalStaking.rewardValidators() to steal rewards

## Summary

In the Covalent Network, validators perform work to earn rewards, which is distributed through `OperationalStaking`. The staking manager is expected to regularly invoke `rewardValidators()` to distribute staking rewards accordingly to the validator, and its delegators.

However, the function takes into account all existing stakes, including new ones. This makes newer stakes being counted equally to existing stakes, despite newer stakes haven't existed for a working epoch yet.

An attacker can also then front-run `rewardValidators()` to steal a share of the rewards. The attacker gains a share of the reward, despite not having fully staked for the corresponding epoch.

## Vulnerability Detail

The function `rewardValidators()` is callable only by the staking manager to distribute rewards to validators. The rewards is then immediately distributed to the validator, and all their delegators, proportional to the amount staked.

However, any new staking in-between reward epochs still counts as staking. They receive the full reward amount for the epoch, despite not having staked for the full epoch.

An attacker can abuse this by front-run the staking manager's distribution with a stake transaction. The attacker stakes a certain amount right before the staking manager distributes rewards, then the attacker is already considered to have a share of the reward, despite not having staked during the epoch they were entitled to.

This also applies to re-stakings, i.e. unstaked tokens that are re-staked into the same validator: Any stake recovers made through `recoverUnstaking()` is considered a new stake. Therefore an attacker can use the same funds to repeatedly perform the attack.

## Proof of concept

Alice is a validator. She has two delegators:
Alicia delegating $5000$ CQT.
Alisson delegating $15000$ CQT.
Alice herself stakes $35000$ CQT, for a total of $50000$.
The staking manager distributes $1000$ CQT to Alice. This is then distributed proportionally across Alice and her delegators.
Bob notices the staking manager, and front-runs with a $50000$ CQT stake into Alice.
Alice now has a total stake of $100000$, with half of it belonging to Bob.
Bob's staking tx goes through before the staking manager's tx.
Bob now owns half of Alice's shares. He is distributed half of the rewards, despite having staked into Alice for only one second.
Bob can repeat this attack for as often as they wish to, by unstaking then restaking through recoverUnstaking().

## Impact

Unfair distribution of rewards. New stakers get full rewards for epochs they didn't fully stake into.

Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L262

## Tool used

Manual Review

## Recommendation

Use a checkpoint-based shares computation. An idea can be as follow:

For new stakings, don't mint shares for them right away, but add them into a "pending stake" variable.
For unstakings, do burn their shares right away, as with the normal setting.
When reward is distributed, distribute to existing stakes first (i.e. increase the share price only for the existing shares), only then mint new shares for the pending stakes.

# [M-3] validatorMaxStake can be bypassed by using setValidatorAddress()

## Summary

`setValidatorAddress()` allows a validator to migrate to a new address of their choice. However, the current logic only stacks up the old address' stake to the new one, never checking `validatorMaxStake`.

## Vulnerability Detail

The current logic for `setValidatorAddress()` is as follow:

```
function setValidatorAddress(uint128 validatorId, address newAddress) external whenNotPaused {
    // ...
    v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
    v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
    delete v.stakings[msg.sender];
    // ...
}
```

The old address' stake is simply stacked on top of the new address' stake. There are no other checks for this amount, even though the new address may already have contained a stake.

Then the combined total of the two stakings may exceed `validatorMaxStake`. This accordingly allows the new (validator) staker's amount to bypass said threshold, breaking an important invariant of the protocol.

## Proof of concept

Bob the validator has a self-stake equal to `validatorMaxStake`.
Bob has another address, B2, with some stake delegated to Bob's validator.
Bob migrates to B2.
Bob's stake is stacked on top of B2. B2 becomes the new validator address, but their stake has exceeded `validatorMaxStake`.
B2 can then repeated this procedure to addresses B3, B4, ..., despite B2 already holding more than the max allowed amount.
Bob now holds more stake than he should be able to, allowing him to earn an unfair amount of rewards compared to other validators.

We also note that, even if the admin tries to freeze Bob, he can front-run the freeze with an unstake, since unstakes are not blocked from withdrawing (after cooldown ends).

## Impact

Breaking an important invariant of the protocol.
Allowing any validator to bypass the max stake amount. In turn allows them to earn an unfair amount of validator rewards in the process.
Allows a validator to unfairly increase their max delegator amount, as an effect of increasing ` (validator stake) * maxCapMultiplier`.

## Code Snippet

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L689-L711

## Tool used

Manual Review

## Recommendation

Check that the new address's total stake does not exceed `validatorMaxStake` before proceeding with the migration.
