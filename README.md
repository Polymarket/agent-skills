# Polymarket Integration Skill

Agent skill for building on Polymarket — the world's largest prediction market. Gives agents the knowledge to authenticate, place orders, read markets, stream real-time data, manage positions, bridge assets across chains, and execute gasless transactions.

## What's Included

```
web3-polymarket/
├── SKILL.md              # Entry point — quick reference, client setup, core patterns
├── README.md             # This file
├── authentication.md     # L1/L2 auth, builder headers, credential lifecycle
├── order-patterns.md     # Order types, tick sizes, cancel, heartbeat, errors
├── market-data.md        # Gamma API, Data API, CLOB orderbook, subgraph
├── websocket.md          # Market/user/sports channels, subscribe, heartbeat
├── ctf-operations.md     # Split, merge, redeem, negative risk, token IDs
├── bridge.md             # Deposits, withdrawals, supported chains/tokens
└── gasless.md            # Relayer client, wallet deployment, builder setup
    wallet-intelligence.md # Decision layer: is the wallet on the other side skilled?
```

## How It Works

The skill uses **progressive disclosure** to stay efficient with context:

1. **SKILL.md loads first** — contains API endpoints, contract addresses, client setup, and core code patterns. Enough for most tasks.
2. **Reference files load on demand** — when a task needs deeper detail (e.g., full error code list, bridge chain support, WebSocket event schemas), the agent reads the relevant file.

This keeps the initial context small (~200 lines) while giving access to ~1,700 lines of detailed reference material when needed.

## When Agents Use This Skill

An agent activates this skill when a user asks about:

- **Authentication** — API keys, EIP-712 signing, HMAC-SHA256, builder credentials
- **Trading** — placing limit/market orders (GTC, GTD, FOK, FAK), batch orders, cancellation, heartbeat keepalive
- **Market data** — fetching events/markets from Gamma API, reading orderbook prices/spreads/midpoints, price history
- **Real-time data** — WebSocket subscriptions for orderbook updates, trade notifications, sports scores
- **Token operations** — splitting USDC.e into Yes/No tokens, merging, redeeming after resolution
- **Bridging** — depositing from 15+ chains, withdrawing, checking status
- **Gasless transactions** — relayer client for gas-free onchain operations
- **Negative risk** — multi-outcome markets, token conversion, augmented neg risk
- **Wallet intelligence** — whether the wallet on the other side of a trade is actually skilled, a bot, or part of a coordinated cluster; sample-size-corrected win rates and funding clusters

## Quick Start for Humans

If you're a developer reading this directly (not an agent), here's the fastest path:

### 1. Install the SDK

```bash
# TypeScript
npm install @polymarket/clob-client ethers@5.8.0

# Python
pip install py-clob-client
```

### 2. Get API Credentials

```typescript
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";

const client = new ClobClient(
  "https://clob.polymarket.com",
  137,
  new Wallet(process.env.PRIVATE_KEY)
);
const creds = await client.createOrDeriveApiKey();
```

### 3. Place an Order

```typescript
const tradingClient = new ClobClient(
  "https://clob.polymarket.com",
  137,
  signer,
  creds,
  2,              // GNOSIS_SAFE (most common)
  "FUNDER_ADDR"   // from polymarket.com/settings
);

const response = await tradingClient.createAndPostOrder(
  { tokenID: "TOKEN_ID", price: 0.50, size: 10, side: "BUY" },
  { tickSize: "0.01", negRisk: false },
  "GTC"
);
```

## Key Concepts

| Concept | Description |
|---------|-------------|
| **USDC.e** | Bridged USDC on Polygon — the collateral token for all markets |
| **Condition ID** | Identifies a market (used in API as `market` or `conditionID`) |
| **Token ID** | Identifies a specific outcome token (Yes or No) within a market |
| **Funder** | The proxy wallet address that holds funds — find at polymarket.com/settings |
| **Signature Type** | `0` = EOA, `1` = POLY_PROXY (Magic Link), `2` = GNOSIS_SAFE (most common) |
| **Neg Risk** | Multi-outcome markets where outcomes are linked — set `negRisk: true` in order options |
| **Tick Size** | Minimum price increment for a market — must match or orders are rejected |

## API Endpoints

| API | Base URL | Auth Required |
|-----|----------|---------------|
| CLOB | `https://clob.polymarket.com` | L2 headers for trades, none for reads |
| Gamma | `https://gamma-api.polymarket.com` | None |
| Data | `https://data-api.polymarket.com` | None |
| Bridge | `https://bridge.polymarket.com` | None |
| Relayer | `https://relayer-v2.polymarket.com/` | Builder headers |
| WS Market | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | None |
| WS User | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | API creds in message |
| WS Sports | `wss://sports-api.polymarket.com/ws` | None |

## Contract Addresses (Polygon)

| Contract | Address |
|----------|---------|
| USDC.e (Bridged USDC) | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` |
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` |
| CTF Exchange | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` |
| Neg Risk CTF Exchange | `0xC5d563A36AE78145C45a50134d48A1215220f80a` |
| Neg Risk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` |

## File Guide

| File | Read when you need to... |
|------|--------------------------|
| [SKILL.md](SKILL.md) | Get started — has everything for basic integration |
| [authentication.md](authentication.md) | Understand L1/L2 auth flow, builder headers, or troubleshoot credential issues |
| [order-patterns.md](order-patterns.md) | Use advanced order types (GTD, post-only, batch), handle errors, or implement heartbeat |
| [market-data.md](market-data.md) | Query markets by slug/tag, paginate results, use subgraph, or estimate fill prices |
| [websocket.md](websocket.md) | Stream real-time orderbook updates, trade notifications, or sports scores |
| [ctf-operations.md](ctf-operations.md) | Split/merge/redeem tokens, work with neg risk markets, or compute token IDs |
| [bridge.md](bridge.md) | Deposit from other chains, withdraw, check supported assets, or track transaction status |
| [gasless.md](gasless.md) | Set up gas-free transactions via the relayer, deploy wallets, or configure builder credentials |
| [wallet-intelligence.md](wallet-intelligence.md) | Judge the wallet on the other side of a trade — sample-size-corrected win rates, bot/human verdicts, funding clusters |

## SDKs

- **TypeScript**: [@polymarket/clob-client](https://github.com/Polymarket/clob-client)
- **Python**: [py-clob-client](https://github.com/Polymarket/py-clob-client)
- **Rust**: [rs-clob-client](https://github.com/Polymarket/rs-clob-client)
- **Builder Relayer (TS)**: [@polymarket/builder-relayer-client](https://github.com/Polymarket/builder-relayer-client)
- **Builder Relayer (Python)**: [py-builder-relayer-client](https://github.com/Polymarket/py-builder-relayer-client)
- **Builder Signing (TS)**: [@polymarket/builder-signing-sdk](https://github.com/Polymarket/builder-signing-sdk)
- **Builder Signing (Python)**: [py-builder-signing-sdk](https://github.com/Polymarket/py-builder-signing-sdk)
