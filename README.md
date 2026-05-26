# ETH Gas Watch ✦

An Aeon skill pack that checks Ethereum gas prices on a schedule. Print the current safe/standard/fast tier, flag when gas drops below your threshold — handy for batching on-chain operations during cheap windows.

## What's in the pack

| Skill | What it does |
|---|---|
| `gas-status` | Queries Etherscan's gas oracle (public, no key required for basic use), prints current Safe / Standard / Fast gwei + a 🟢 / 🟡 / 🔴 status indicator. Logs to `memory/` so you build a price-history archive. |

## Install

```bash
./install-skill-pack sparkleware/eth-gas-watch
```

Scheduled `0 */4 * * *` (every 4 hours) by default — frequent enough to catch overnight low-gas windows without spamming logs.

## License

MIT — see [LICENSE](./LICENSE).
