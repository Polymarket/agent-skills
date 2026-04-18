---
name: web3-polymarket
description: Polymarket integration for prediction market trading on Polygon. Covers V2 CLOB authentication (L1 EIP-712, L2 HMAC-SHA256), order placement (GTC/GTD/FOK/FAK, batch, post-only, heartbeat), market data (Gamma API, Data API, orderbook, keyset pagination), WebSocket streaming (market/user/sports channels), CTF operations (split, merge, redeem, negative risk), bridge flows, and gasless relayer transactions. Use when building AI agents, autonomous market makers, prediction market UIs, or any application integrating with Polymarket on Polygon.
compatibility: Requires network access to Polymarket APIs (clob.polymarket.com, gamma-api.polymarket.com, relayer-v2.polymarket.com) and Polygon RPC
---

# Polymarket Skill

## When to use this skill

Use this skill when the user asks about or needs to build:
- Polymarket API authentication (L1/L2, API keys, HMAC signing)
- Placing or managing orders on **CLOB V2**
- Reading orderbook data (prices, spreads, midpoints, CLOB market info)
- Market data fetching (events, markets, by slug, by tag, keyset pagination)
- WebSocket subscriptions (market channel, user channel, sports)
- CTF operations (split, merge, redeem positions)
- Negative risk markets (multi-outcome, conversion, augmented neg risk)
- Bridge operations (deposits, withdrawals, supported assets, status)
- Gasless transactions (relayer client, wallet deployment)
- Builder program integration (`builderCode` attribution)
- Polymarket SDK usage (TypeScript `@polymarket/clob-client-v2`, Python `py-clob-client-v2`)

## V2 Migration Notes

- **Go-live:** April 22, 2026 (~11:00 UTC)
- **Preprod host:** `https://clob-v2.polymarket.com`
- **Prod host after cutover:** `https://clob.polymarket.com`
- **Open orders are wiped during cutover**
- **L1/L2 auth is unchanged**
- **Builder HMAC headers are removed for order attribution**; use `builderCode`
- **Collateral is now pUSD** (not USDC.e)

## API Configuration

| API | Base URL | Auth | Purpose |
|-----|----------|------|---------|
| CLOB V2 | `https://clob.polymarket.com` | L2 for trade endpoints | Orderbook, prices, order submission |
| CLOB V2 Test | `https://clob-v2.polymarket.com` | L2 for trade endpoints | Pre-cutover V2 testing |
| Gamma | `https://gamma-api.polymarket.com` | None | Events, markets, search |
| Data API | `https://data-api.polymarket.com` | None | Trades, positions, user data |
| WebSocket (Market) | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | None | Real-time orderbook |
| WebSocket (User) | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | API creds in message | Trade/order updates |
| WebSocket (Sports) | `wss://sports-api.polymarket.com/ws` | None | Live scores |
| Relayer | `https://relayer-v2.polymarket.com/` | Relayer API key or Builder API key | Gasless transactions |
| Bridge | `https://bridge.polymarket.com` | None | Deposits/withdrawals |

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

## Client Setup

### TypeScript
```typescript
import { ClobClient, Side, OrderType } from "@polymarket/clob-client-v2";
import { Wallet } from "ethers";

const host = "https://clob.polymarket.com";
const chain = 137;
const signer = new Wallet(process.env.PRIVATE_KEY!);

const tempClient = new ClobClient({ host, chain, signer });
const apiCreds = await tempClient.createOrDeriveApiKey();

const client = new ClobClient({
  host,
  chain,
  signer,
  creds: apiCreds,
  signatureType: 2,
  funderAddress: process.env.FUNDER_ADDRESS!,
  builderConfig: process.env.POLY_BUILDER_CODE
    ? { builderCode: process.env.POLY_BUILDER_CODE }
    : undefined,
});
```

### Python
```python
from py_clob_client.client import ClobClient
import os

host = "https://clob.polymarket.com"
chain = 137
pk = os.getenv("PRIVATE_KEY")

temp_client = ClobClient(host=host, chain=chain, key=pk)
api_creds = temp_client.create_or_derive_api_creds()

client = ClobClient(
    host=host,
    chain=chain,
    key=pk,
    creds=api_creds,
    signature_type=2,
    funder=os.getenv("FUNDER_ADDRESS"),
)
```

## Quick Reference: Order Types

| Type | Behavior | Use Case |
|------|----------|----------|
| **GTC** | Rests on book until filled or cancelled | Default limit orders |
| **GTD** | Active until expiration (UTC seconds) | Auto-expire before events |
| **FOK** | Fill entirely immediately or cancel | All-or-nothing market orders |
| **FAK** | Fill what is available, cancel rest | Partial-fill market orders |

- FOK/FAK BUY: `amount` = dollar amount to spend
- FOK/FAK SELL: `amount` = number of shares to sell
- Market buy orders can include `userUSDCBalance` for fee-aware sizing
- Fees are **not** set in signed orders in V2

## Quick Reference: Signature Types

| Type | Value | Description |
|------|-------|-------------|
| EOA | `0` | Standard EOA wallet |
| POLY_PROXY | `1` | Legacy Polymarket proxy wallet |
| GNOSIS_SAFE | `2` | Gnosis Safe multisig proxy wallet |

## Core Pattern: Place an Order

### TypeScript
```typescript
const response = await client.createAndPostOrder(
  {
    tokenID: "TOKEN_ID",
    price: 0.50,
    size: 10,
    side: Side.BUY,
    builderCode: process.env.POLY_BUILDER_CODE,
  },
  {
    tickSize: "0.01",
    negRisk: false,
  },
  OrderType.GTC,
);
console.log(response.orderID, response.status);
```

### Python
```python
from py_clob_client.clob_types import OrderArgs, OrderType
from py_clob_client.order_builder.constants import BUY

response = client.create_and_post_order(
    OrderArgs(token_id="TOKEN_ID", price=0.50, size=10, side=BUY),
    options={"tick_size": "0.01", "neg_risk": False},
    order_type=OrderType.GTC,
)
print(response["orderID"], response["status"])
```

## Core Pattern: Read Orderbook

### TypeScript
```typescript
const readClient = new ClobClient({ host: "https://clob.polymarket.com", chain: 137 });
const book = await readClient.getOrderBook("TOKEN_ID");
const mid = await readClient.getMidpoint("TOKEN_ID");
const spread = await readClient.getSpread("TOKEN_ID");
```

### Python
```python
read_client = ClobClient(host="https://clob.polymarket.com", chain=137)
book = read_client.get_order_book("TOKEN_ID")
mid = read_client.get_midpoint("TOKEN_ID")
spread = read_client.get_spread("TOKEN_ID")
```

## Core Pattern: Query V2 Fee / Market Params

```typescript
const info = await client.getClobMarketInfo("0xCONDITION_ID");
// info.mts  minimum tick size
// info.mos  minimum order size
// info.fd   fee details { r, e, to }
// info.t    tokens
// info.rfqe RFQ enabled
```

## Core Pattern: WebSocket Subscribe

```typescript
const ws = new WebSocket("wss://ws-subscriptions-clob.polymarket.com/ws/market");

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: "market",
    assets_ids: ["TOKEN_ID"],
    custom_feature_enabled: true,
  }));
  setInterval(() => ws.send("PING"), 10_000);
};
```

## Reference files (load on demand)

- **Authentication**: [authentication.md](authentication.md)
- **Order patterns**: [order-patterns.md](order-patterns.md)
- **Market data**: [market-data.md](market-data.md)
- **WebSocket**: [websocket.md](websocket.md)
- **CTF operations**: [ctf-operations.md](ctf-operations.md)
- **Bridge**: [bridge.md](bridge.md)
- **Gasless transactions**: [gasless.md](gasless.md)
