# Market Data

Polymarket market data comes from four main sources:
- **Gamma API** — events, markets, search, metadata
- **Data API** — trades, positions, user-related data
- **CLOB API** — orderbook, prices, spreads, fees, CLOB market info
- **Subgraph** — onchain analytics and historical queries

## Gamma API

Base URL: `https://gamma-api.polymarket.com`

### Events

```bash
# All active events
GET https://gamma-api.polymarket.com/events?active=true&closed=false&limit=100

# By slug
GET https://gamma-api.polymarket.com/events?slug=fed-decision-in-october

# By tag
GET https://gamma-api.polymarket.com/events?tag_id=100381&closed=false&limit=10

# By series
GET https://gamma-api.polymarket.com/events?series_id=10345&closed=false&limit=20

# Sorted by volume
GET https://gamma-api.polymarket.com/events?active=true&closed=false&order=volume_24hr&ascending=false&limit=100
```

### Markets

```bash
# By slug
GET https://gamma-api.polymarket.com/markets?slug=fed-decision-in-october
```

### Sort Parameters

| Parameter | Values |
|-----------|--------|
| `order` | `volume_24hr`, `volume`, `liquidity`, `start_date`, `end_date`, `competitive`, `closed_time` |
| `ascending` | `true` / `false` (default: `false`) |
| `active` | `true` / `false` |
| `closed` | `true` / `false` |
| `limit` | endpoint-specific |
| `offset` | legacy offset pagination on non-keyset endpoints |

## Recommended pagination: keyset endpoints

Polymarket now documents **keyset pagination** for stable paging through large datasets.
Prefer these over offset-based pagination when scanning many rows.

### Events keyset pagination

```bash
GET https://gamma-api.polymarket.com/events/keyset?closed=false&limit=50
GET https://gamma-api.polymarket.com/events/keyset?closed=false&limit=50&after_cursor=NEXT_CURSOR
```

### Markets keyset pagination

```bash
GET https://gamma-api.polymarket.com/markets/keyset?closed=false&limit=100
GET https://gamma-api.polymarket.com/markets/keyset?closed=false&limit=100&after_cursor=NEXT_CURSOR
```

Rules:
- use `next_cursor` from the previous response as `after_cursor`
- `offset` is explicitly rejected on keyset endpoints
- max limits:
  - events keyset: `500`
  - markets keyset: `1000`

### Common filters

| Parameter | Notes |
|-----------|-------|
| `slug` | Fetch a specific event or market |
| `tag_id` / `tag_slug` | Category filters |
| `closed` | Historical vs active |
| `order` | Sort field(s) |
| `ascending` | Sort direction |
| `limit` | Page size |
| `after_cursor` | Keyset pagination cursor |

### Tags & Sports metadata

```bash
# Discover tags
GET https://gamma-api.polymarket.com/tags

# Sports metadata
GET https://gamma-api.polymarket.com/sports
```

## Data API

Base URL: `https://data-api.polymarket.com`

Used for trades, positions, activity, and user-related reporting endpoints.

## CLOB Market Data

Base URL: `https://clob.polymarket.com`

No auth required for read-only endpoints.

### Read client setup

```typescript
import { ClobClient } from "@polymarket/clob-client-v2";

const client = new ClobClient({
  host: "https://clob.polymarket.com",
  chain: 137,
});
```

```python
from py_clob_client.client import ClobClient

client = ClobClient(host="https://clob.polymarket.com", chain=137)
```

### Orderbook

```typescript
const book = await client.getOrderBook("TOKEN_ID");
```

```bash
curl "https://clob.polymarket.com/book?token_id=TOKEN_ID"
```

### Prices

```typescript
const buyPrice = await client.getPrice("TOKEN_ID", "BUY");   // best ask
const sellPrice = await client.getPrice("TOKEN_ID", "SELL"); // best bid
const mid = await client.getMidpoint("TOKEN_ID");
const spread = await client.getSpread("TOKEN_ID");
const last = await client.getLastTradePrice("TOKEN_ID");
```

If the spread is wide, consuming both midpoint and last-trade price can produce better UI behavior than midpoint alone.

### Batch reads

All major market-data methods have batch variants.

| Single | Batch | REST |
|--------|-------|------|
| `getOrderBook()` | `getOrderBooks()` | `POST /books` |
| `getPrice()` | `getPrices()` | `POST /prices` |
| `getMidpoint()` | `getMidpoints()` | `POST /midpoints` |
| `getSpread()` | `getSpreads()` | `POST /spreads` |
| `getLastTradePrice()` | `getLastTradePrices()` | request body variant documented |

## V2 CLOB market info

V2 adds `getClobMarketInfo()`, which exposes market-specific CLOB parameters in one call.

```typescript
const info = await client.getClobMarketInfo("0xCONDITION_ID");
// info.mts  minimum tick size
// info.mos  minimum order size
// info.fd   fee details { r, e, to }
// info.t    tokens [{ t, o }]
// info.rfqe request-for-quote enabled
```

```bash
curl "https://clob.polymarket.com/clob-markets/0xCONDITION_ID"
```

Useful fields:
- `mts` — minimum tick size
- `mos` — minimum order size
- `fd` — fee curve params
- `t` — token IDs and outcomes
- `rfqe` — RFQ enabled
- `mbf` / `tbf` — base fees

## Fee model notes

In V2, fees are not embedded in the signed order.
They are determined by protocol market parameters at match time.
For manual inspection, use `getClobMarketInfo()`.

## Price history

```typescript
import { PriceHistoryInterval } from "@polymarket/clob-client-v2";

const history = await client.getPricesHistory({
  market: "TOKEN_ID",
  interval: PriceHistoryInterval.ONE_DAY,
  fidelity: 60,
});
```

Intervals commonly used:
- `1h`
- `6h`
- `1d`
- `1w`
- `1m`
- `max`

Use `startTs` / `endTs` for absolute ranges when you do not want interval-based presets.

## Estimate Fill Price

Walk the orderbook to estimate slippage for a given order size:

```typescript
import { OrderType, Side } from "@polymarket/clob-client-v2";

const price = await client.calculateMarketPrice(
  "TOKEN_ID",
  Side.BUY,
  500,
  OrderType.FOK,
);
```

## Key Market Fields

| Field | Description |
|-------|-------------|
| `tokenID` / `asset_id` | Outcome token ID |
| `conditionID` / `market` | Condition ID |
| `tokens` | Array of outcome token objects |
| `neg_risk` / `negRisk` | Neg risk market flag |
| `minimum_tick_size` | Tick size |
| `enableOrderBook` | Orderbook enabled |
| `slug` | URL slug |

## Subgraph

Useful for onchain analytics, open interest, PnL, balances, and historical event data.

```bash
curl -X POST \
  https://api.goldsky.com/api/public/project_cl6mb8i9h0003e201j6li0diw/subgraphs/orderbook-subgraph/0.0.1/gn \
  -H "Content-Type: application/json" \
  -d '{"query":"query { orderbooks { id tradesQuantity } }"}'
```

## Practical fetching strategy

1. Fetch a known event by `slug`
2. For browsing, use `events/keyset` or `markets/keyset`
3. Pull token IDs from Gamma responses
4. Query CLOB orderbook / prices using token IDs
5. Use `getClobMarketInfo()` when you need tick size, fee params, minimum order size, or token mappings
