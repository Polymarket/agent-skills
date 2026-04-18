# Authentication

Polymarket CLOB V2 keeps the same two-layer auth model:
- **L1**: EIP-712 wallet signature for API key creation / derivation
- **L2**: HMAC-SHA256 request signing with API credentials

Important V2 clarification:
- **L1/L2 auth is unchanged** from V1
- **Builder order attribution no longer uses `POLY_BUILDER_*` headers**
- Builders now attach a **`builderCode`** to orders
- Raw relayer HTTP endpoints may still accept Builder API key auth, but normal CLOB trading attribution is now `builderCode`-based

## L1 Authentication

L1 proves wallet ownership and is used to create or derive API credentials.

### EIP-712 Domain

```typescript
const domain = {
  name: "ClobAuthDomain",
  version: "1",
  chainId: 137,
};

const types = {
  ClobAuth: [
    { name: "address", type: "address" },
    { name: "timestamp", type: "string" },
    { name: "nonce", type: "uint256" },
    { name: "message", type: "string" },
  ],
};
```

The auth domain remains version `"1"` in V2.

### L1 Headers

| Header | Description |
|--------|-------------|
| `POLY_ADDRESS` | Polygon signer address |
| `POLY_SIGNATURE` | EIP-712 signature |
| `POLY_TIMESTAMP` | Current UNIX timestamp |
| `POLY_NONCE` | Nonce used for deterministic key derivation |

### Create / Derive Credentials

```typescript
import { ClobClient } from "@polymarket/clob-client-v2";

const client = new ClobClient({
  host: "https://clob.polymarket.com",
  chain: 137,
  signer,
});

const creds = await client.createOrDeriveApiKey();
```

```python
from py_clob_client.client import ClobClient

client = ClobClient(host="https://clob.polymarket.com", chain=137, key=pk)
creds = client.create_or_derive_api_creds()
```

**REST endpoints**
- `POST {host}/auth/api-key`
- `GET {host}/auth/derive-api-key`

## L2 Authentication

L2 uses your API credentials for HMAC signing. Required for trading endpoints.

### L2 Headers

| Header | Description |
|--------|-------------|
| `POLY_ADDRESS` | Polygon signer address |
| `POLY_SIGNATURE` | HMAC signature for request |
| `POLY_TIMESTAMP` | Current UNIX timestamp |
| `POLY_API_KEY` | User API key |
| `POLY_PASSPHRASE` | User API passphrase |

### Initialize Trading Client

```typescript
import { ClobClient } from "@polymarket/clob-client-v2";

const client = new ClobClient({
  host: "https://clob.polymarket.com",
  chain: 137,
  signer,
  creds: apiCreds,
  signatureType: 2,
  funderAddress,
});
```

```python
from py_clob_client.client import ClobClient

client = ClobClient(
    host="https://clob.polymarket.com",
    chain=137,
    key=pk,
    creds=api_creds,
    signature_type=2,
    funder=funder_address,
)
```

## Signature Types

| Type | Value | When to Use |
|------|-------|-------------|
| EOA | `0` | Standard externally owned wallet |
| POLY_PROXY | `1` | Legacy Polymarket proxy wallets |
| GNOSIS_SAFE | `2` | Gnosis Safe / proxy wallet flows |

The **funder** is the address holding the funds used for trading.

## Builder Attribution in V2

V1 used Builder HMAC headers plus `builder-signing-sdk` for order attribution.
V2 replaces that with a public **builder code** attached directly to each order.

### Per-order builder code

```typescript
import { ClobClient, Side } from "@polymarket/clob-client-v2";

await client.createAndPostOrder(
  {
    tokenID: "TOKEN_ID",
    price: 0.55,
    size: 100,
    side: Side.BUY,
    builderCode: process.env.POLY_BUILDER_CODE,
  },
  { tickSize: "0.01", negRisk: false },
);
```

### Client-wide builder code

```typescript
const client = new ClobClient({
  host: "https://clob.polymarket.com",
  chain: 137,
  signer,
  creds: apiCreds,
  signatureType: 2,
  funderAddress,
  builderConfig: { builderCode: process.env.POLY_BUILDER_CODE! },
});
```

### What changed

- removed for CLOB order attribution:
  - `POLY_BUILDER_API_KEY`
  - `POLY_BUILDER_SECRET`
  - `POLY_BUILDER_PASSPHRASE`
  - `POLY_BUILDER_SIGNATURE`
  - `@polymarket/builder-signing-sdk`
- added:
  - `builderCode` on each order, or
  - `builderConfig: { builderCode }` on the client

## Raw Relayer Builder Auth

For normal CLOB order attribution, use `builderCode`.
For raw relayer HTTP endpoints, the official docs still describe Builder API key auth and Relayer API key auth.

### Builder / Relayer auth headers

| Header | Description |
|--------|-------------|
| `POLY_BUILDER_API_KEY` | Builder API key identifier |
| `POLY_BUILDER_PASSPHRASE` | Builder passphrase |
| `POLY_BUILDER_SIGNATURE` | Builder HMAC signature |
| `POLY_BUILDER_TIMESTAMP` | Current UNIX timestamp |

Use these only when you are integrating with the relayer directly.
Do not treat them as the order-attribution path for CLOB V2 orders.

## Raw Order Signing Changes

Only relevant if you sign raw order structs yourself instead of using the SDK.

### Exchange domain changes

- exchange EIP-712 domain version: `"1"` → `"2"`
- standard exchange verifying contract:
  - V1: `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E`
  - V2: `0xE111180000d2663C0091e4f400237545B87B996B`
- neg risk exchange verifying contract:
  - V1: `0xC5d563A36AE78145C45a50134d48A1215220f80a`
  - V2: `0xe2222d279d744050d28e00520010520000310F59`

### Signed order field changes

Removed from signed orders:
- `taker`
- `nonce`
- `feeRateBps`
- `expiration` from the EIP-712 struct shown in V1 docs

Added in V2:
- `timestamp` (ms)
- `metadata` (`bytes32`)
- `builder` (`bytes32`)

If you use the official V2 SDK, this is handled for you.

## Credential Lifecycle

- `createApiKey()` — create new API credentials
- `deriveApiKey(nonce)` — derive existing credentials
- `createOrDeriveApiKey()` — preferred setup flow

Your existing API keys continue to work in V2.
