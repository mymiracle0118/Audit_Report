# Report (2nd Day)

## Checking Result for Other Contracts

### Scope of Checking

- `v2-periphery.contracts`
- `v3-periphery.contracts`
- `v3-staker.contracts`

### Conclusion

I checked above contracts today comparing with original Uniswap V3 codebase. These codebase just forked the Uni V3 well completely except replacing the protocol name.
So, I think there is no special issue to be a crucial weakpoint.

## Personal Opinion for Forward Developing

Uniswap has several potential issues when it's interacted with external DeFi or DeX protocol.

For example, `pool.slot0` of `UniswapV3Pool` has potential risk to may occur price manipulation by sandwitch attacking during swap. Of course, it depends on how contruct and develop the logic and codebase according to your roadmap and plan. That's why I requested you the roadmap and whitepaper of the project.

To avoid and fix these potential risks and possible issues in forward developing, should work with code by considering known potential vulnerabilities of Uniswap V3, and typical DEX issues and perform the pre/post-auditing code completely.

Thank you!
