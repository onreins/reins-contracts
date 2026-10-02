# Reins contracts

**Mandate: give an AI agent a USDC budget with rules it can't break, on [Arc](https://arc.io).**

A Mandate is a smart contract that holds an agent's USDC. The agent can trade
inside the rules its owner set, and nothing else. The rules are enforced by the
contract on every trade, not trusted to the agent, to us, or to anyone's server.

These are the contracts behind [Reins](https://reins.one). The app is at
[app.reins.one](https://app.reins.one), and the agent tooling (SDK and MCP
server) is in [reins-mcp](https://github.com/ic88t/reins-mcp).

## The rules

| Rule | Enforced how |
|---|---|
| **Only allowed assets** | Trades only between tokens the owner listed, through the one exchange the owner chose |
| **Max trade size** | No single trade worth more than `maxTradeValue`, else `TradeTooLarge` |
| **Fair prices** | Every fill is checked against a Chainlink price; anything worse than `maxSlippageBps` reverts with `PriceTooLow` |
| **Max loss** | A trade that would leave equity below `baseline × (1 − maxDrawdownBps)` reverts with `DrawdownLimit`, and *anyone* can call `checkpoint()` to freeze the Mandate if the market alone pushes it past that line: a public stop-loss |
| **Fresh prices only** | No trading on a price older than `maxPriceAge` (`StalePrice`) |
| **Expiry** | No trading after `expiresAt` |
| **No exits** | The agent has no function that moves money anywhere but back into the Mandate |

The owner can `withdraw`, `withdrawAll`, swap the agent with `setAgent`, or
`unfreeze` at any time, and that exit never depends on an oracle being healthy.

**Why the price check matters:** for a dishonest agent, "trade at a terrible
price into a pool I control" is the obvious way to drain a budget. Checking
every fill against an oracle bounds that leak to `maxSlippageBps` per trade,
and the drawdown limit bounds the total. The worst case is a number the owner
chose, not the whole balance.

## Contracts

| Contract | What it does |
|---|---|
| [`Mandate.sol`](contracts/Mandate.sol) | The Mandate and its `MandateFactory`. No admin, no upgrade path, no pause; the rules are fixed at creation. |
| [`UniswapV4Venue.sol`](contracts/UniswapV4Venue.sol) | An exchange adapter that swaps through a Uniswap v4 pool. Each pair's route can be set once and never changed, so the deployer can't later point a pair at a different pool. |
| [`PinnedFeed.sol`](contracts/PinnedFeed.sol) | A Chainlink-shaped feed for testnets, where Chainlink doesn't publish. Testnet only; on mainnet a Mandate reads the real Chainlink aggregator. |
| [`MandateSim.sol`](contracts/MandateSim.sol) | The mainnet proof below: deploys, funds and trades a Mandate inside a single `eth_call`. |
| [`MandateMocks.sol`](contracts/MandateMocks.sol) | Test doubles, including hostile ones: a venue that steals, one that re-enters, one that underpays, a token with no return value, a restricted token. |

Plain Solidity 0.8.28 with no external libraries, compiled by
[`scripts/compile.js`](scripts/compile.js) with `solc` directly (EVM `cancun`,
which Arc's Osaka fork fully supports).

## Proven against Arc mainnet, without spending anything

```bash
npm install && npm run build
npm run sim:mainnet -- 10
```

One `eth_call` against live mainnet state: a Mandate is deployed inside the
call, funded, and its agent trades $10 of USDC into EURC and back through the
real Uniswap v4 pool, checked by the real Chainlink feed. Nothing is broadcast
and nothing is spent.

```
  Chainlink EURC/USD   $1.1385
  bought               8.7775 EURC for $10.00  (fill $1.1393 per EURC, 0.068% vs oracle)
  equity at the end    $19.9900  (round trip cost $0.0100)
```

## Live on Arc testnet

| Contract | Address |
|---|---|
| `MandateFactory` | [`0x09e45d5b84d9c7cf4e8cdf5d5f1ff2b7d3589f82`](https://explorer.testnet.arc.io/address/0x09e45d5b84d9c7cf4e8cdf5d5f1ff2b7d3589f82) |
| `UniswapV4Venue` | [`0x007d5ad07b7a97fefcbd4302dfeafc11d8485052`](https://explorer.testnet.arc.io/address/0x007d5ad07b7a97fefcbd4302dfeafc11d8485052) |
| `PinnedFeed` | [`0xfdde6a331c996d6fddbdce108ba18fff0b7f972e`](https://explorer.testnet.arc.io/address/0xfdde6a331c996d6fddbdce108ba18fff0b7f972e) |
| A live Mandate | [`0x991b8687aca6Acd6b92438bb4cE22866827bD632`](https://explorer.testnet.arc.io/address/0x991b8687aca6Acd6b92438bb4cE22866827bD632) |

An agent bought euros through the real Uniswap v4 pool at 0.45% from the
oracle. The Mandate then refused a trade over its size limit, a trade into an
asset it was never granted, a trade too large for the pool to price fairly, and
a trade signed by a key that isn't its agent. A stranger froze it when it fell
through its floor, and the owner withdrew mid-strategy. Every refusal was
broadcast on purpose, so it exists on-chain as a transaction you can open:
**[the full run, with transaction links →](docs/TESTNET-RUN.md)**

## Build and test

```bash
npm install
npm run build        # compile with solc into build/
npm run chain        # terminal 1: a local node mirroring Arc's 20 gwei fee floor
npm test             # terminal 2: 32 tests
```

The tests are mostly adversarial: an exchange that takes the tokens and sends
nothing, one that calls back into the Mandate mid-trade, one that tries to trip
the stop-loss mid-swap while balances look low; small losses adding up past the
limit; stale and broken price feeds (which must never block a withdrawal);
trades over each limit, outside the allowed assets or after expiry; the agent
trying to take money out; and bad configurations refused at creation.

## Security

An adversarial review ran before any deployment. Nothing critical. It found
three issues, all fixed with regression tests:

- a broken price feed could block a partial withdrawal; now a stale or broken
  feed never blocks one, and the asset can be dropped
- a venue could trip the stop-loss mid-swap, while balances looked low
- rounding on dust-sized trades

**Not formally audited.** A review and adversarial tests are not an audit. Use
small amounts until one is done.

## Arc notes

- Native USDC is 18 decimals; the ERC-20 view of the same balance is 6. One
  balance, not two. A Mandate holds the ERC-20 form.
- Block timestamps are only non-decreasing; expiries use them, but nothing here
  needs guaranteed progress between two blocks.
- The base fee floor is 20 gwei; the local test node mirrors it.
- Chainlink publishes to Arc mainnet but not testnet, hence `PinnedFeed`.

## License

[MIT](LICENSE)
