# Polymarket Integration Skill

Agent skill for building on Polymarket CLOB V2. Covers authentication, trading, market data, WebSockets, CTF token operations, bridge flows, and gasless relayer usage.

## What's Included

```
web3-polymarket/
├── SKILL.md              # Entry point — quick reference, client setup, core patterns
├── README.md             # This file
├── authentication.md     # L1/L2 auth, builderCode attribution, relayer auth notes
├── order-patterns.md     # Order types, tick sizes, cancel, heartbeat, errors
├── market-data.md        # Gamma API, Data API, CLOB orderbook, fee params, subgraph
├── websocket.md          # Market/user/sports channels, subscribe, heartbeat
├── ctf-operations.md     # Split, merge, redeem, negative risk, token IDs
├── bridge.md             # Deposits, withdrawals, supported chains/tokens
└── gasless.md            # Relayer client, wallet deployment, builder / relayer auth
```

## V2 status

Validated against:
- `https://docs.polymarket.com/llms.txt`
- `https://docs.polymarket.com/v2-migration.md`

Key V2 changes reflected here:
- `@polymarket/clob-client-v2` / `py-clob-client-v2`
- constructor uses an **options object** in TypeScript and `chain` instead of `chain_id`
- signed orders use `timestamp`, `metadata`, `builder`; not `nonce`, `feeRateBps`, `taker`
- **pUSD** replaces USDC.e as trading collateral
- builder attribution uses **`builderCode`**, not `POLY_BUILDER_*` headers
- exchange contracts moved to V2 addresses
- Gamma keyset pagination endpoints are documented

## How It Works

The skill uses **progressive disclosure** to stay efficient with context:

1. **SKILL.md loads first** — contains API endpoints, contract addresses, client setup, and core code patterns. Enough for most tasks.
2. **Reference files load on demand** — when a task needs deeper detail, the agent reads the relevant file.

This keeps the initial context small while preserving the richer reference material in the topic files.

## When Agents Use This Skill

Use it for:
- authentication and API key setup
- V2 order creation and order management
- market/event discovery and orderbook reads
- WebSocket streaming
- pUSD / CTF operations
- bridging and funding flows
- gasless relayer transactions
- builder code attribution

## Quick Start

### Install the V2 SDK

```bash
# TypeScript
npm install @polymarket/clob-client-v2 ethers@5.8.0

# Python
pip install py-clob-client-v2
```

### Create API credentials

```typescript
import { ClobClient } from "@polymarket/clob-client-v2";
import { Wallet } from "ethers";

const signer = new Wallet(process.env.PRIVATE_KEY!);
const client = new ClobClient({
  host: "https://clob.polymarket.com",
  chain: 137,
  signer,
});

const creds = await client.createOrDeriveApiKey();
```

### Place an order

```typescript
import { ClobClient, Side, OrderType } from "@polymarket/clob-client-v2";

const tradingClient = new ClobClient({
  host: "https://clob.polymarket.com",
  chain: 137,
  signer,
  creds,
  signatureType: 2,
  funderAddress: process.env.FUNDER_ADDRESS!,
  builderConfig: process.env.POLY_BUILDER_CODE
    ? { builderCode: process.env.POLY_BUILDER_CODE }
    : undefined,
});

const response = await tradingClient.createAndPostOrder(
  {
    tokenID: "TOKEN_ID",
    price: 0.50,
    size: 10,
    side: Side.BUY,
  },
  { tickSize: "0.01", negRisk: false },
  OrderType.GTC,
);
```

## Key Concepts

| Concept | Description |
|---------|-------------|
| **pUSD** | The collateral token used for trading on Polymarket |
| **USDC.e** | Source asset wrapped into pUSD for API-only flows |
| **Condition ID** | Market identifier |
| **Token ID** | Outcome token identifier |
| **Funder** | Wallet address holding funds |
| **Signature Type** | `0` EOA, `1` POLY_PROXY, `2` GNOSIS_SAFE |
| **Builder Code** | Public builder identifier attached to orders for attribution |
| **Neg Risk** | Multi-outcome markets with linked outcomes |

## API Endpoints

| API | Base URL | Auth Required |
|-----|----------|---------------|
| CLOB V2 | `https://clob.polymarket.com` | L2 for trades, none for reads |
| CLOB V2 Test | `https://clob-v2.polymarket.com` | L2 for trades, none for reads |
| Gamma | `https://gamma-api.polymarket.com` | None |
| Data | `https://data-api.polymarket.com` | None |
| Bridge | `https://bridge.polymarket.com` | None |
| Relayer | `https://relayer-v2.polymarket.com/` | Relayer API key or Builder API key |
| WS Market | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | None |
| WS User | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | API creds in message |
| WS Sports | `wss://sports-api.polymarket.com/ws` | None |

## Contract Addresses (Polygon)

| Contract | Address |
|----------|---------|
| pUSD | `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB` |
| USDC.e | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` |
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` |
| CTF Exchange V2 | `0xE111180000d2663C0091e4f400237545B87B996B` |
| Neg Risk CTF Exchange V2 | `0xe2222d279d744050d28e00520010520000310F59` |
| Neg Risk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` |
| Collateral Onramp | `0x93070a847efEf7F70739046A929D47a521F5B8ee` |
| Collateral Offramp | `0x2957922Eb93258b93368531d39fAcCA3B4dC5854` |

## File Guide

| File | Use it when you need to... |
|------|----------------------------|
| [SKILL.md](SKILL.md) | Get started — has the quick-reference V2 setup |
| [authentication.md](authentication.md) | Understand L1/L2 auth, builderCode attribution, or relayer auth details |
| [order-patterns.md](order-patterns.md) | Use advanced order types (GTD, post-only, batch), handle errors, or implement heartbeat |
| [market-data.md](market-data.md) | Query markets by slug/tag, paginate results, inspect fee params, or estimate fill prices |
| [websocket.md](websocket.md) | Stream real-time orderbook updates, trade notifications, or sports scores |
| [ctf-operations.md](ctf-operations.md) | Split/merge/redeem tokens, work with neg risk markets, or compute token IDs |
| [bridge.md](bridge.md) | Deposit from other chains, withdraw, check supported assets, or track transaction status |
| [gasless.md](gasless.md) | Set up gas-free transactions via the relayer, deploy wallets, or configure relayer auth |

## SDKs

- **TypeScript CLOB V2**: [@polymarket/clob-client-v2](https://www.npmjs.com/package/@polymarket/clob-client-v2)
- **Python CLOB V2**: [py-clob-client-v2](https://pypi.org/project/py-clob-client-v2/)
- **Builder Relayer (TS)**: [@polymarket/builder-relayer-client](https://github.com/Polymarket/builder-relayer-client)
- **Builder Relayer (Python)**: [py-builder-relayer-client](https://github.com/Polymarket/py-builder-relayer-client)
