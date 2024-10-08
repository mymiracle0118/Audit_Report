# Title 1: After the current reserves are updated, kLast is also updated. As a result, the _mintFee function in JalaPair is unable to mint fee tokens.

## Severity

Medium

## Summary

The _mintFee function in JalaPair.sol determines the fee amount of liquidity tokens using the difference between the previous and current reserves. However, because the value of kLast is not being correctly updated, this results in the _mintFee function being unable to mint fee-based liquidity.

## Vulnerability Details

In the `JalaPair` contract, liquidity is generated based on the growth of `sqrt(reserve0 * reserve1)` as fees during `mint` and `burn` operations.
```javascript
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo = IJalaFactory(factory).feeTo(); // get feeTo address
        feeOn = feeTo != address(0); // @audit in the case of address DEAD?
        uint256 _kLast = kLast; // gas savings
        if (feeOn) {
            if (_kLast != 0) {
                uint256 rootK = Math.sqrt(uint256(_reserve0) * _reserve1);
                uint256 rootKLast = Math.sqrt(_kLast);
@>              if (rootK > rootKLast) {
                    uint256 numerator = totalSupply * (rootK - rootKLast);
                    uint256 denominator = rootK + rootKLast;
                    uint256 liquidity = numerator / denominator;
                    // distribute LP fee
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }

    function mint(address to) external lock returns (uint256 liquidity) {
        (uint112 _reserve0, uint112 _reserve1, ) = getReserves(); // gas savings
        uint256 balance0 = IERC20(token0).balanceOf(address(this));
        uint256 balance1 = IERC20(token1).balanceOf(address(this));
        uint256 amount0 = balance0 - _reserve0;
        uint256 amount1 = balance1 - _reserve1;

@>      bool feeOn = _mintFee(_reserve0, _reserve1);

        // ...
```
Specifically, within the `_mintFee` function, the values of `rootK` (calculated from the current reserves) and `rootKLast` (computed using the most recent reserves) are used to determine the fee-based liquidity.
```javascript
      _update(balance0, balance1, _reserve0, _reserve1);
@>      if (feeOn) kLast = uint256(reserve0) * reserve1; // reserve0 and reserve1 are up-to-date
        emit Mint(msg.sender, amount0, amount1);
    }
```
However, it should be noted that the value of `kLast` is only updated after the pair's reserves have already been adjusted according to the latest `mint` or `burn` action. Consequently, when the subsequent `mint` or `burn` operation takes place, both rootK and rootKLast inside the `_mintFee` function will hold identical values. This ultimately leads to the calculation of zero fee-based liquidity by the `_mintFee` function.

Attached below is the proof-of-concept test file:
```javascript
function test_checkingMintIssue() public {
        address alice = makeAddr("alice");
        tokenA.mint(0.5 ether, alice);
        tokenB.mint(0.5 ether, alice);

        vm.prank(feeSetter);
        factory.setFeeTo(feeSetter);
        
        // add liquidity to pair of token A and B
        vm.startPrank(alice);
        tokenA.approve(address(router), 0.5 ether);
        tokenB.approve(address(router), 0.5 ether);

        router.addLiquidity(
            address(tokenA),
            address(tokenB),
            0.1 ether,
            0.1 ether,
            0.1 ether,
            0.1 ether,
            alice,
            block.timestamp
        );
        vm.stopPrank();

        address pair = factory.getPair(address(tokenA), address(tokenB));

        assertEq(IJalaPair(pair).balanceOf(alice), 0.1 ether - 1000);

        assertEq(IJalaPair(pair).balanceOf(feeSetter), 0);

        (uint256 reserve0, uint256 reserve1, ) = IJalaPair(pair).getReserves();

        assertEq(IJalaPair(pair).balanceOf(feeSetter), 0);

        vm.startPrank(alice);
        router.addLiquidity(
            address(tokenA),
            address(tokenB),
            0.1 ether,
            0.1 ether,
            0.1 ether,
            0.1 ether,
            alice,
            block.timestamp
        );
        vm.stopPrank();

        assertEq(IJalaPair(pair).balanceOf(feeSetter), 0);

        vm.startPrank(alice);
        IJalaPair(pair).approve(address(router), 0.1 ether);
        router.removeLiquidity(
            address(tokenA),
            address(tokenB),
            0.1 ether,
            0.1 ether - 1000,
            0.1 ether - 1000,
            alice,
            block.timestamp
        );
        vm.stopPrank();

        assertEq(IJalaPair(pair).balanceOf(feeSetter), 0);
    }
```
