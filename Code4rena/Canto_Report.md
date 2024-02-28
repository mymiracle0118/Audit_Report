# [H-1] update_market() nextEpoch calculation incorrect

A very important logic of `update_market()` is to update `accCantoPerShare`. When updating, if it crosses the epoch boundary, it needs to use the corresponding epoch's `cantoPerBlock[epoch]`. For example: cantoPerBlock[100000] = 100 cantoPerBlock[200000] = 0 (No reward) lastRewardBlock = 199999 block.number = 200002

At this time, `accCantoPerShare` needs to be increased: = cantoPerBlock[100000] * 1 Delta + cantoPerBlock[200000] *2 Delta = 100 _ (200000 - 199999) + 0 _ (200002 - 200000) = 100

The code is as follows:

```
    function update_market(address _market) public {
        require(lendingMarketWhitelist[_market], "Market not whitelisted");
        MarketInfo storage market = marketInfo[_market];
        if (block.number > market.lastRewardBlock) {
            uint256 marketSupply = lendingMarketTotalBalance[_market];
            if (marketSupply > 0) {
                uint256 i = market.lastRewardBlock;
                while (i < block.number) {
                    uint256 epoch = (i / BLOCK_EPOCH) * BLOCK_EPOCH; // Rewards and voting weights are aligned on a weekly basis
@>                  uint256 nextEpoch = i + BLOCK_EPOCH;
                    uint256 blockDelta = Math.min(nextEpoch, block.number) - i;
                    uint256 cantoReward = (blockDelta *
                        cantoPerBlock[epoch] *
                        gaugeController.gauge_relative_weight_write(_market, epoch)) / 1e18;
                    market.accCantoPerShare += uint128((cantoReward * 1e18) / marketSupply);
                    market.secRewardsPerShare += uint128((blockDelta * 1e18) / marketSupply); // TODO: Scaling
                    i += blockDelta;
                }
            }
            market.lastRewardBlock = uint64(block.number);
        }
    }
```

The above code, the calculation of `nextEpoch` is wrong

`uint256 nextEpoch = i + BLOCK_EPOCH;`

The correct one should be

`uint256 nextEpoch = epoch + BLOCK_EPOCH;`

As a result, the increase of `accCantoPerShare` becomes: = cantoPerBlock[100000] _ 3 Delta = 100 _ (200002 - 199999) = 300

## Impact

The calculation of `update_market()` is incorrect when crossing the epoch boundary, causing the calculation of `accCantoPerShare` to be incorrect, which may increase the allocated rewards.

## Proof of Concept

The following code demonstrates the above example, when added to LendingLedgerTest.t.sol:

```

    function test_ErrorNextEpoch() external {
        vm.roll(fromEpoch);
        whiteListMarket();
        //only set first , second cantoPerBlock = 0
        vm.prank(goverance);
        ledger.setRewards(fromEpoch, fromEpoch, amountPerBlock);
        vm.prank(lendingMarket);
        ledger.sync_ledger(lender, 1e18);

        //set lastRewardBlock = nextEpoch - 10
        vm.roll(fromEpoch + BLOCK_EPOCH - 10);
        ledger.update_market(lendingMarket);

        //cross-border
        vm.roll(fromEpoch + BLOCK_EPOCH + 10);
        ledger.update_market(lendingMarket);

        //show more
        (uint128 accCantoPerShare,,) = ledger.marketInfo(lendingMarket);
        console.log("accCantoPerShare:",accCantoPerShare);
        console.log("more:",accCantoPerShare - amountPerEpoch);

    }
```

```
$ forge test -vvv --match-test test_ErrorNextEpoch

Running 1 test for src/test/LendingLedger.t.sol:LendingLedgerTest
[PASS] test_ErrorNextEpoch() (gas: 185201)
Logs:
  accCantoPerShare: 1000100000000000000
  more: 100000000000000

```

## Recommended Mitigation

```
    function update_market(address _market) public {
        require(lendingMarketWhitelist[_market], "Market not whitelisted");
        MarketInfo storage market = marketInfo[_market];
        if (block.number > market.lastRewardBlock) {
            uint256 marketSupply = lendingMarketTotalBalance[_market];
            if (marketSupply > 0) {
                uint256 i = market.lastRewardBlock;
                while (i < block.number) {
                    uint256 epoch = (i / BLOCK_EPOCH) * BLOCK_EPOCH; // Rewards and voting weights are aligned on a weekly basis
-                   uint256 nextEpoch = i + BLOCK_EPOCH;
+                   uint256 nextEpoch = epoch + BLOCK_EPOCH;
```

# [M-1] secRewardsPerShare Insufficient precision

We also introduced the field secRewardDebt. The idea of this field is to enable any lending platforms that are integrated with Neofinance Coordinator to send their own rewards based on this value (or rather the difference of this value since the last time secondary rewards were sent) and their own emission schedule for the tokens.

The current calculation formula for `secRewardsPerShare` is as follows:

```
market.secRewardsPerShare += uint128((blockDelta * 1e18) / marketSupply);
```

`marketSupply` is cNOTE, with a precision of `1e18` So as long as the supply is greater than `1` cNote, `secRewardsPerShare` is easily `rounded down` to `0` Example: marketSupply = 10e18 blockDelta = 1 secRewardsPerShare=1 \* 1e18 / 10e18 = 0

## Impact

Due to insufficient precision, `secRewardsPerShare` will basically be 0.

## Recommended Mitigation

It is recommended to use 1e27 for `secRewardsPerShare`:

```
    market.secRewardsPerShare += uint128((blockDelta * 1e27) / marketSupply);
```

# [M-2] Loss of precission when calculating the accumulated CANTO per share

When calculating the amount of CANTO per share in `update_market`, dividing by `1e18` in `cantoReward` and multiplying by the same value in `accCantoPerShare` rounds down the final value, making the amount of rewards users will receive be less than expected.

## Proof of Concept

It's well known that Solidity rounds down when doing an integer division, and because of that, it is always recommended to multiply before dividing to avoid that precision loss. However, if we go to:

[LendingLedger, function update_market](https://github.com/code-423n4/2024-01-canto/blob/5e0d6f1f981993f83d0db862bcf1b2a49bb6ff50/src/LendingLedger.sol#L67C1-L70C93)

```
    function update_market(address _market) public {
        require(lendingMarketWhitelist[_market], "Market not whitelisted");
        MarketInfo storage market = marketInfo[_market];
        if (block.number > market.lastRewardBlock) {
            uint256 marketSupply = lendingMarketTotalBalance[_market];
            if (marketSupply > 0) {
                uint256 i = market.lastRewardBlock;
                while (i < block.number) {
                    uint256 epoch = (i / BLOCK_EPOCH) * BLOCK_EPOCH; // Rewards and voting weights are aligned on a weekly basis
                    uint256 nextEpoch = i + BLOCK_EPOCH;
                    uint256 blockDelta = Math.min(nextEpoch, block.number) - i;
                    uint256 cantoReward = (blockDelta *
                        cantoPerBlock[epoch] *
                        gaugeController.gauge_relative_weight_write(_market, epoch)) / 1e18;
                    market.accCantoPerShare += uint128((cantoReward * 1e18) / marketSupply);
                    market.secRewardsPerShare += uint128((blockDelta * 1e18) / marketSupply); // TODO: Scaling
                    i += blockDelta;
                }
            }
            market.lastRewardBlock = uint64(block.number);
        }
    }
```

and we expand the maths behind `accCantoPerShare` and `cantoReward`:

```
                    uint256 cantoReward = (blockDelta *
                        cantoPerBlock[epoch] *
                        gaugeController.gauge_relative_weight_write(_market, epoch)) / 1e18;
                    market.accCantoPerShare += uint128((cantoReward * 1e18) / marketSupply);
```

like follows:

$$(cantoReward * 1e18) \ / \ marketSupply$$

$$(((blockDelta * cantoPerBlock[epoch] * gaugeController.gauge_relative_weight_write(_market, epoch)) \ / \ 1e18) \ * \ 1e18) \ / \ marketSupply$$

$$((stuff \ / \ 1e18) \ * \ 1e18) \ / \ marketSupply$$

we see there is a hidden division before a multiplication that is rounding down the whole expression. This is bad as the precision loss can be significant, which leads to the given market having less rewards to offer to its users. Run the next test inside a foundry project to see such a divergence in the precision if we multiply before dividing:

```
    // @audit the assumes are to avoid under/overflows errors
    function testPOC(uint256 x) external pure {
        vm.assume(x < type(uint256).max / 1e18);
        vm.assume(x > 1e18);
        console2.log("(x / 1e18) * 1e18", (x / 1e18) * 1e18);
        console2.log("(x * 1e18) / 1e18", (x * 1e18) / 1e18);
    }
```

Some examples:

```
  [7725] POC::testPOC(1164518589284217277370 [1.164e21])
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log((x / 1e18) * 1e18, 1164000000000000000000 [1.164e21]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log((x * 1e18) / 1e18, 1164518589284217277370 [1.164e21]) [staticcall]
    │   └─ ← ()
    └─ ← ()

  [7725] POC::testPOC(16826228168456047587 [1.682e19])
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log((x / 1e18) * 1e18, 16000000000000000000 [1.6e19]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log((x * 1e18) / 1e18, 16826228168456047587 [1.682e19]) [staticcall]
    │   └─ ← ()
    └─ ← ()

  [7725] POC::testPOC(5693222591586418917 [5.693e18])
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log((x / 1e18) * 1e18, 5000000000000000000 [5e18]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log((x * 1e18) / 1e18, 5693222591586418917 [5.693e18]) [staticcall]
    │   └─ ← ()
    └─ ← ()
```

## Recommended Mitigation Steps

Change the `update_market` function to:

```
    function update_market(address _market) public {
        require(lendingMarketWhitelist[_market], "Market not whitelisted");
        MarketInfo storage market = marketInfo[_market];
        if (block.number > market.lastRewardBlock) {
            uint256 marketSupply = lendingMarketTotalBalance[_market];
            if (marketSupply > 0) {
                uint256 i = market.lastRewardBlock;
                while (i < block.number) {
                    uint256 epoch = (i / BLOCK_EPOCH) * BLOCK_EPOCH; // Rewards and voting weights are aligned on a weekly basis
                    uint256 nextEpoch = i + BLOCK_EPOCH;
                    uint256 blockDelta = Math.min(nextEpoch, block.number) - i;
                    uint256 cantoReward = (blockDelta *
                        cantoPerBlock[epoch] *
-                       gaugeController.gauge_relative_weight_write(_market, epoch)) / 1e18;
-                   market.accCantoPerShare += uint128((cantoReward * 1e18) / marketSupply);
+                       gaugeController.gauge_relative_weight_write(_market, epoch)) * 1e18;
+                   market.accCantoPerShare += uint128((cantoReward / 1e18) / marketSupply);
                    market.secRewardsPerShare += uint128((blockDelta * 1e18) / marketSupply); // TODO: Scaling
                    i += blockDelta;
                }
            }
            market.lastRewardBlock = uint64(block.number);
        }
    }
```
