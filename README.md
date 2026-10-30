# elysium-ropes-market

One-minute UP/DOWN market on the Hyperliquid HYPE mark, played on **Elysium testnet** (chain ID `99801`). ROPES: each round lasts sixty seconds; a climber’s height is the live value of that position.

Part of [ElysiumForge Labs](https://github.com/ElysiumForge-Labs).

> **Independent research repository. Testnet only.** Not affiliated with, endorsed by, or sponsored by Kinetiq, Hyperliquid, Ascend, dreamDEX, or Somnia. Test USDC (tUSDC) and test HYPE have no monetary value. Contracts and the house wallet are testnet research surfaces.

Public game reference: [playropes.xyz](https://playropes.xyz). This repository is the Elysium port of the Somnia × dreamDEX Event Contracts hackathon entry, adapted for the Ascend Builder Program.

## Overview

The keeper opens, prices, and settles rounds. In this version trading is operator-only: the executor submits player orders. The contract is the market maker and quotes both sides from the relayed Hyperliquid mark.

| Time | Action |
| --- | --- |
| 00:00 (~2 s) | `openRound`, struck at the open of Hyperliquid’s 1-minute candle |
| 00:01–00:35 | Entries: `buy(side, budget)` with a fresh mark |
| Until the bell | Bail: `sell(side, shares)` at the live bid |
| 01:00 (~3 s) | `settle` on the next candle open; calls `ArenaRegistry.onRoundSettled` |
| After settlement | `redeem` — a winning share redeems for exactly 1 |

**Pricing** (continuous quotes; no waiting for a counterparty):

```
fair(UP) = Φ( ln(S / K) / (σ · √τ) )
```

The contract sells UP at fair + 1% and buys back at fair − 1%. DOWN is the mirror. Each round may risk at most 25% of the vault’s free balance.

**Stack**

| Layer | Technology |
| --- | --- |
| Contracts | Solidity 0.8.28, Foundry, Solady FixedPointMathLib |
| App | Next.js 15, React 19, Tailwind 4, TypeScript, viem 2 |
| Data | Prisma 6, Postgres |
| Oracle path | Hyperliquid info API → keeper relay (until HyperCore read precompile) |

## Deployed contracts (Elysium testnet)

Sources are verified on the Elysium testnet explorer. House vault seed: 100,000 tUSDC (per-round risk cap 25,000).

| Contract | Address |
| --- | --- |
| RopeMarket (HYPE) | [`0xB425550958863472D351F17575B6b8F0C3a525D6`](https://elysium.kinetiq.xyz/testnet-explorer/address/0xB425550958863472D351F17575B6b8F0C3a525D6) |
| ArenaRegistry | [`0x0D8b61252d631D3483DD10C5725ecf9F7f314262`](https://elysium.kinetiq.xyz/testnet-explorer/address/0x0D8b61252d631D3483DD10C5725ecf9F7f314262) |
| tUSDC | [`0x46397dA50c05AfA43cBfcA712215eD1E02A9F57d`](https://elysium.kinetiq.xyz/testnet-explorer/address/0x46397dA50c05AfA43cBfcA712215eD1E02A9F57d) |

## Getting started

### Prerequisites

- Node.js (web app, executor, scripts)
- Postgres (`DATABASE_URL`)
- [Foundry](https://getfoundry.sh/) (contract tests / deploy)

Use a throwaway executor key funded with a small amount of Elysium testnet HYPE for gas. The web app does not use `PRIVATE_KEY`.

### Install and run

```sh
npm install
cp .env.example .env
npx prisma db push
npm run dev
```

Wall UI: [http://localhost:3000](http://localhost:3000).

| Command | Purpose |
| --- | --- |
| `npm run dev` | Next.js development server |
| `npm run build` / `npm start` | Production build and server |
| `npm run executor` | Supervisor loop around the game executor |
| `npm run executor:once` | One executor process, no supervisor |
| `npm run doctor` | Check ledger balances |
| `npm run contracts:test` | `forge test` |
| `npx tsx scripts/rehearse.ts` | One end-to-end round |
| `npx tsx scripts/audit.ts` | Strike / close vs Hyperliquid candle opens |

Deploy:

```sh
DEPLOY_PK=0x… SIGMA_WAD=… forge script contracts/script/Deploy.s.sol \
  --rpc-url elysium_testnet --broadcast
```

`elysium_testnet` is defined in `foundry.toml` as `https://testnet-rpc.elysium.kinetiq.xyz`.

## Trust boundaries

Until Elysium’s HyperCore read precompile is available, the keeper relays the mark from Hyperliquid’s public info API. The contract enforces sample windows, void-on-missed-bell, stale-mark trading stops, a 25% risk cap, and operator-only trading.

Whoever controls `PRIVATE_KEY` can move the house tUSDC balance. Keep owner keys offline; rotate the operator with `setOperator` when needed.

## Layout

| Path | Role |
| --- | --- |
| `contracts/src/RopeMarket.sol` | Pricing, rounds, caps, settlement, redemption, vault |
| `contracts/src/ArenaRegistry.sol` | Public scoreboard |
| `contracts/src/MockUSDC.sol` | Test dollars (open mint) |
| `src/lib/chain/` | Elysium client, HL prices, ABIs |
| `src/lib/ledger/` | Postgres ledger |
| `src/lib/game/` | Rounds, orders, boards, points |
| `src/executor/` | Tick loop |
| `src/app/` | Wall, play, wallet, docs, API |
| `scripts/` | Audit, rehearsal, money doctor |

## Disclaimer

Independent research for ElysiumForge Labs. Operator-relayed marks are not a substitute for sequencer-attested HyperCore reads. Do not use funded mainnet keys. Verify contract addresses and explorer links before interacting.
