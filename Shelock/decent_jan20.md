# Title 1: Since `performSwap` function doesn't refund left ether, users may lose their left fund.

## Impact
Since `performSwap` function doesn't refund left ether, users may lose their fund to should be received back.

## Proof of concept
`performSwap` function performs a swap with the requested swapper and calldata. When input token of swapping is native token(i.e ether), this function checks that received ether is greater than input amount of swapping. But this function doesn't refund the left eth and it makes users lose their eth to should be received back.
```javascript
        if (swapParams.tokenIn == address(0)) {
@>          require(msg.value >= swapParams.amountIn, "not enough native");
            wrapped.deposit{value: swapParams.amountIn}();
            swapParams.tokenIn = address(wrapped);
            swapInstructions.swapPayload = swapper.updateSwapParams(
                swapParams,
                swapInstructions.swapPayload
            );
        } else if (retrieveTokenIn) {
            IERC20(swapParams.tokenIn).transferFrom(
                msg.sender,
                address(this),
                swapParams.amountIn
            );
        }
```


## Tool used

Manual Review

## Recommendation

