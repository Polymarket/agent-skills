# Bridge

Polymarket trading collateral is now **pUSD**. The Bridge API handles deposits from supported chains and withdrawals back out.

Base URL: `https://bridge.polymarket.com`

## Deposit Flow

1. `POST /deposit` with your Polymarket wallet address
2. Check `/supported-assets`
3. Send supported funds to the chain-specific deposit address
4. Bridge routes funds to Polygon and credits your Polymarket balance
5. Track progress via `/status/{deposit_address}`

For end users, bridge flows result in funds becoming usable as **pUSD** on Polymarket.

```bash
curl -X POST https://bridge.polymarket.com/deposit \
  -H "Content-Type: application/json" \
  -d '{"address": "0xYOUR_POLYMARKET_WALLET"}'
```

### Deposit address types

| Address | Use For |
|---------|---------|
| `evm` | EVM chains |
| `svm` | Solana |
| `btc` | Bitcoin |
| `tvm` | Tron |

Each address is unique to your wallet.

## Supported chains

Always call `/supported-assets` for the live list.
Examples currently documented include Ethereum, Polygon, Arbitrum, Base, Optimism, BNB Smart Chain, Solana, Bitcoin, Tron, HyperEVM, Abstract, Monad, Ethereal, Katana, and Lighter.

## Withdrawal Flow

1. Check `/supported-assets`
2. Preview using `POST /quote`
3. `POST /withdraw` with destination chain, token, and recipient
4. Send funds from your Polymarket wallet to the provided withdrawal address
5. Poll `/status/{address}` until completion

```bash
curl -X POST https://bridge.polymarket.com/withdraw \
  -H "Content-Type: application/json" \
  -d '{
    "address": "0xYOUR_POLYMARKET_WALLET",
    "toChainId": "1",
    "toTokenAddress": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "recipientAddr": "0xDESTINATION_ADDRESS"
  }'
```

Do not pre-generate withdrawal addresses too early; create them only when ready to use.

## Quote

```bash
POST https://bridge.polymarket.com/quote
```

Use quotes before deposits or withdrawals to inspect fees and expected output.

## Status Tracking

```bash
curl https://bridge.polymarket.com/status/0xDEPOSIT_ADDRESS
```

### Statuses

| Status | Terminal | Description |
|--------|----------|-------------|
| `DEPOSIT_DETECTED` | No | Source funds detected |
| `PROCESSING` | No | Routing / swap in progress |
| `ORIGIN_TX_CONFIRMED` | No | Source chain tx confirmed |
| `SUBMITTED` | No | Submitted on destination side |
| `COMPLETED` | Yes | Success |
| `FAILED` | Yes | Failure |

Poll every 10–30 seconds until terminal.

## Example response

```json
{
  "transactions": [{
    "fromChainId": "1",
    "fromTokenAddress": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "fromAmountBaseUnit": "1000000000",
    "toChainId": "137",
    "toTokenAddress": "0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB",
    "status": "COMPLETED",
    "txHash": "0x...",
    "createdTimeMs": 1697875200000
  }]
}
```

Note: destination collateral on Polymarket is now **pUSD**, so expect pUSD-related destination references instead of legacy USDC.e-only assumptions.

## Recovery

If you deposit the wrong token:
- Ethereum deposits: `https://recovery.polymarket.com/`
- Polygon deposits: `https://matic-recovery.polymarket.com/`

## Caveats

- deposits below minimum are not processed
- supported assets change; always check `/supported-assets`
- large withdrawals may incur slippage depending on route and token
- for API-only workflows, remember Polymarket trading collateral is pUSD
