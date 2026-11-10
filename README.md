# DrawdownCap

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/drawdown-cap?style=flat&logo=github)](https://github.com/nirholas/drawdown-cap/stargazers) [![License](https://img.shields.io/github/license/nirholas/drawdown-cap?style=flat)](https://github.com/nirholas/drawdown-cap/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/drawdown-cap?style=flat)](https://github.com/nirholas/drawdown-cap/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/drawdown-cap/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/drawdown-cap/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**A limit down. The pool may not fall more than a fixed distance below where the current epoch opened, and the limit resets on a schedule rather than on anyone's say-so.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://drawdown-cap.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/DrawdownCapHook.sol`](src/hooks/DrawdownCapHook.sol)
- **Licence:** Apache-2.0

## How it works

{CircuitBreakerHook} is symmetric and reactive: a violent move in either direction halts the pool, then the halt clears. This is the other shape, and it is the one commodity and equity venues actually use. It is asymmetric, because a collapse and a rally are not the same event for the people holding the asset.

It is a hard cap rather than a trigger, so the fall never happens rather than being noticed after it did. And it resets on a clock, so everyone can see in advance when selling reopens and at what level. allowed while openTick - tick <= maxFallTicks, where openTick is the tick at the start of the epoch Buying is never restricted.

A pool at its limit can still be bid up, and doing so does not raise the limit for that epoch, because the reference is the epoch's opening price and not a running high. When the epoch rolls, the pool takes its current price as the new opening and gets a fresh allowance. As with {RatchetFloorHook}, the cap is expressed as a price a router can trade into: {sqrtPriceLimitDownX96} returns the value to pass as a swap's `sqrtPriceLimitX96`, so a seller fills as far as the cap allows and stops there.

The `afterSwap` revert is the backstop for callers that pass no limit. Liquidity operations are never blocked, so nobody is trapped by a limit-down epoch. One tick is one basis point to within rounding, so `maxFallTicks = 1000` is a ten percent daily limit.

## Prior art

Trading halts and price bands are standard on regulated venues and absent on-chain, where the closest equivalents are governance pause switches and oracle-deviation guards. Hook implementations of trading hours exist. A scheduled, asymmetric, self-resetting limit down with no privileged role does not.

## Where it does not help

A limit down does not stop a decline, it defers one. If the market has genuinely repriced, the pool reopens each epoch and falls again, one limit at a time, and in the meantime the gap between the pool and the real price is an arbitrage that grows. It buys holders time to react, which is worth something, and it costs liquidity providers the trades they would rather have made, which is not free.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    DrawdownCapHook.Config({
        maxFallTicks: /* uint24 */ 0,
        epochSeconds: /* uint32 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `maxFallTicks` | `uint24` | ticks |
| `epochSeconds` | `uint32` | seconds |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `InvalidConfig()` | `maxFallTicks` or `epochSeconds` was zero. |
| `LimitDown(int24,int24)` | The swap would take the pool past this epoch's limit down. Pass `sqrtPriceLimitDownX96` as a price limit. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `beforeSwap`
- `afterSwap`

Mask: `0x10c0`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # DrawdownCap
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # risk, circuit-breaker, oracle-free, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/drawdown-cap
cd drawdown-cap
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/drawdown-cap&type=Date)](https://www.star-history.com/#nirholas/drawdown-cap&Date)

<!-- three.ws:growth -->
## Support the project

If drawdown-cap saves you time, **[star it on GitHub](https://github.com/nirholas/drawdown-cap)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=drawdown-cap%3A%20A%20limit%20down&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdrawdown-cap) · [Share on Bluesky](https://bsky.app/intent/compose?text=drawdown-cap%3A%20A%20limit%20down%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fdrawdown-cap) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdrawdown-cap) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdrawdown-cap&t=drawdown-cap%3A%20A%20limit%20down) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fdrawdown-cap&title=drawdown-cap%3A%20A%20limit%20down)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/drawdown-cap` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/drawdown-cap/issues) or [start a discussion](https://github.com/nirholas/drawdown-cap/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/drawdown-cap)](https://github.com/nirholas/drawdown-cap/graphs/contributors)

<!-- /three.ws:growth -->

<!-- nirholas:watermark -->

---

<p align="center">
  Built by <a href="https://github.com/nirholas"><b>nirholas</b></a> · <a href="https://x.com/nichxbt">@nichxbt</a> · <a href="https://github.com/nirholas">github.com/nirholas</a>
</p>

<!-- /nirholas:watermark -->
