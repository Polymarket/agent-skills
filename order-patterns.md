# Order Patterns

All Polymarket CLOB V2 orders are expressed as limit-style orders. Market orders are marketable limit orders that execute immediately or cancel.

## V2 Changes That Matter

- SDK package is now `@polymarket/clob-client-v2` / `py-clob-client-v2`
- fees are **not embedded** in signed orders
- do **not** set `feeRateBps`, `nonce`, or `taker`
- builders use `builderCode`
- collateral for buying is **pUSD**
- if you manually sign raw orders, the exchange EIP-712 domain version is now `"2"`

## Order Types

| Type | Behavior | Use Case |
|------|----------|----------|
| **GTC** | Good-Til-Cancelled | Default resting limit orders |
| **GTD** | Good-Til-Date | Auto-expiring limit orders |
| **FOK** | Fill-Or-Kill | All-or-nothing marketable order |
| **FAK** | Fill-And-Kill | Partial-fill marketable order |

## Tick Sizes

| Tick Size | Precision | Example |
|-----------|-----------|---------|
| `0.1` | 1 decimal | `0.5` |
| `0.01` | 2 decimals | `0.50` |
| `0.001` | 3 decimals | `0.500` |
| `0.0001` | 4 decimals | `0.5000` |

Fetch tick size from:
- `client.getTickSize(tokenID)`
- `minimum_tick_size` on market objects
- `client.getClobMarketInfo(conditionID).mts`

## Limit Order (GTC)

```typescript
import { ClobClient, Side, OrderType } from "@polymarket/clob-client-v2";

const response = await client.createAndPostOrder(
  {
    tokenID: "TOKEN_ID",
    price: 0.50,
    size: 10,
    side: Side.BUY,
    builderCode: process.env.POLY_BUILDER_CODE,
  },
  { tickSize: "0.01", negRisk: false },
  OrderType.GTC,
);
```

```python
response = client.create_and_post_order(
    OrderArgs(token_id="TOKEN_ID", price=0.50, size=10, side=BUY),
    options={"tick_size": "0.01", "neg_risk": False},
    order_type=OrderType.GTC,
)
```

## Two-step flow

```typescript
const signedOrder = await client.createOrder(
  { tokenID: "TOKEN_ID", price: 0.50, size: 10, side: Side.BUY },
  { tickSize: "0.01", negRisk: false },
);

const response = await client.postOrder(signedOrder, OrderType.GTC);
```

## Market Orders (FOK / FAK)

- **BUY**: `amount` is the dollar amount to spend
- **SELL**: `amount` is the number of shares to sell
- optional `userUSDCBalance` helps fee-aware market-buy calculations in V2

```typescript
const response = await client.createAndPostMarketOrder(
  {
    tokenID: "TOKEN_ID",
    side: Side.BUY,
    amount: 100,
    price: 0.50,
    userUSDCBalance: 1000,
    builderCode: process.env.POLY_BUILDER_CODE,
  },
  { tickSize: "0.01", negRisk: false },
  OrderType.FOK,
);
```

## GTD Orders

Expiration remains a UNIX timestamp in seconds.

```typescript
const expiration = Math.floor(Date.now() / 1000) + 60 + 3600;

await client.createAndPostOrder(
  { tokenID: "TOKEN_ID", price: 0.50, size: 10, side: Side.BUY, expiration },
  { tickSize: "0.01", negRisk: false },
  OrderType.GTD,
);
```

## Post-only Orders

```typescript
const response = await client.postOrder(signedOrder, OrderType.GTC, true);
```

Rules:
- valid with `GTC` and `GTD`
- rejected with `FOK` / `FAK`
- rejected if it would cross the spread

## Batch Orders

Maximum: **15 orders** per request.

```typescript
const orders = [
  {
    order: await client.createOrder(
      { tokenID: "TOKEN_ID", price: 0.48, side: Side.BUY, size: 500 },
      { tickSize: "0.01", negRisk: false },
    ),
    orderType: OrderType.GTC,
  },
  {
    order: await client.createOrder(
      { tokenID: "TOKEN_ID", price: 0.52, side: Side.SELL, size: 500 },
      { tickSize: "0.01", negRisk: false },
    ),
    orderType: OrderType.GTC,
  },
];

await client.postOrders(orders);
```

## Cancel Orders

```typescript
await client.cancelOrder("0xORDER_ID");
await client.cancelOrders(["0xID_1", "0xID_2"]);
await client.cancelAll();
await client.cancelMarketOrders({ market: "0xCONDITION_ID" });
await client.cancelMarketOrders({ market: "0xCONDITION_ID", asset_id: "TOKEN_ID" });
```

## Heartbeat

If no valid heartbeat is received within roughly 10 seconds (plus small buffer), open orders are cancelled.

```typescript
let heartbeatId = "";
setInterval(async () => {
  const resp = await client.postHeartbeat(heartbeatId);
  heartbeatId = resp.heartbeat_id;
}, 5000);
```

## Trade / Insert Statuses

### Insert statuses
- `matched`
- `live`
- `delayed`
- `unmatched`

### Trade lifecycle
- `MATCHED`
- `MINED`
- `CONFIRMED`
- `RETRYING`
- `FAILED`

## Errors

Common documented errors include:
- `INVALID_ORDER_MIN_TICK_SIZE`
- `INVALID_ORDER_MIN_SIZE`
- `INVALID_ORDER_DUPLICATED`
- `INVALID_ORDER_NOT_ENOUGH_BALANCE`
- `INVALID_ORDER_EXPIRATION`
- `INVALID_POST_ONLY_ORDER_TYPE`
- `INVALID_POST_ONLY_ORDER`
- `FOK_ORDER_NOT_FILLED_ERROR`
- `MARKET_NOT_READY`

## Prerequisites

Before placing orders, the funder address must approve the exchange contract.

- **Buying**: approve **pUSD** allowance for the appropriate exchange
- **Selling**: approve conditional-token allowance for the appropriate exchange

Relevant exchange contracts:
- standard: `0xE111180000d2663C0091e4f400237545B87B996B`
- neg risk: `0xe2222d279d744050d28e00520010520000310F59`

## Raw signing checklist

If you manually build EIP-712 orders:
- use exchange domain version `"2"`
- use V2 exchange verifying contract
- include `timestamp`, `metadata`, `builder`
- remove `nonce`, `feeRateBps`, `taker`

If you use the official V2 SDK, this is automatic.
