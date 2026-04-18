# Gasless Transactions

Polymarket's relayer enables gasless onchain actions. Users do not need POL for gas; Polymarket sponsors execution.

For trading collateral, users need **pUSD**.

## What changed in V2

- trading collateral is **pUSD**, not USDC.e
- recommended SDK auth uses **Relayer API Keys**
- CLOB order attribution moved to `builderCode`, but relayer HTTP endpoints can still support Builder API Key auth at the raw API level
- recommended relayer client setup now uses an options object

## How It Works

1. Your app prepares one or more transactions
2. The user signs through their wallet
3. Your app submits through the relayer
4. Polymarket pays gas
5. The transaction executes from the user's wallet / proxy

## Covered operations

| Operation | Description |
|-----------|-------------|
| Wallet deployment | Deploy Safe or Proxy wallets |
| Token approvals | Approve pUSD or outcome tokens |
| CTF operations | Split, merge, redeem |
| Transfers | Move assets |

## Authentication

### Recommended: Relayer API Keys

Create a Relayer API key from Polymarket settings.

Headers:

| Header | Description |
|--------|-------------|
| `RELAYER_API_KEY` | Relayer API key |
| `RELAYER_API_KEY_ADDRESS` | Address that owns the key |

### Raw API note

The relayer API reference also allows Builder API Key auth on some endpoints, but for SDK integrations the current documented path is **Relayer API Keys**.

## Installation

```bash
# TypeScript
npm install @polymarket/builder-relayer-client

# Python
pip install py-builder-relayer-client
```

## Client Setup

### TypeScript

```typescript
import { createWalletClient, http, Hex } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { polygon } from "viem/chains";
import { RelayClient } from "@polymarket/builder-relayer-client";

const account = privateKeyToAccount(process.env.PRIVATE_KEY as Hex);
const wallet = createWalletClient({
  account,
  chain: polygon,
  transport: http(process.env.RPC_URL),
});

const client = new RelayClient({
  host: "https://relayer-v2.polymarket.com/",
  chain: 137,
  signer: wallet,
  relayerApiKey: process.env.RELAYER_API_KEY!,
  relayerApiKeyAddress: process.env.RELAYER_API_KEY_ADDRESS!,
});
```

### Python

```python
import os
from py_builder_relayer_client.client import RelayClient

client = RelayClient(
    host="https://relayer-v2.polymarket.com",
    chain=137,
    signer=os.getenv("PRIVATE_KEY"),
    relayer_api_key=os.environ["RELAYER_API_KEY"],
    relayer_api_key_address=os.environ["RELAYER_API_KEY_ADDRESS"],
)
```

## Wallet Types

| Type | Deployment | Best For |
|------|------------|----------|
| **Safe** | Call `deploy()` first | Most integrations |
| **Proxy** | Auto-deploy on first tx | Proxy-wallet flows |

```typescript
import { RelayClient, RelayerTxType } from "@polymarket/builder-relayer-client";

const client = new RelayClient({
  host: "https://relayer-v2.polymarket.com/",
  chain: 137,
  signer: wallet,
  relayerApiKey: process.env.RELAYER_API_KEY!,
  relayerApiKeyAddress: process.env.RELAYER_API_KEY_ADDRESS!,
  txType: RelayerTxType.SAFE,
});

const response = await client.deploy();
const result = await response.wait();
console.log(result?.proxyAddress);
```

## Executing Transactions

```typescript
interface Transaction {
  to: string;
  data: string;
  value: string;
}

const response = await client.execute(transactions, "Description");
await response.wait();
```

### Approval example

```typescript
import { encodeFunctionData, maxUint256 } from "viem";

const pUSD = "0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB";
const CTF = "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045";

const approveTx = {
  to: pUSD,
  data: encodeFunctionData({
    abi: [{
      name: "approve",
      type: "function",
      inputs: [
        { name: "spender", type: "address" },
        { name: "amount", type: "uint256" },
      ],
      outputs: [{ type: "bool" }],
    }],
    functionName: "approve",
    args: [CTF, maxUint256],
  }),
  value: "0",
};

await client.execute([approveTx], "Approve pUSD for CTF");
```

## Transaction States

| State | Terminal | Description |
|-------|----------|-------------|
| `STATE_NEW` | No | Received |
| `STATE_EXECUTED` | No | Submitted onchain |
| `STATE_MINED` | No | Included in a block |
| `STATE_CONFIRMED` | Yes | Finalized successfully |
| `STATE_FAILED` | Yes | Permanent failure |
| `STATE_INVALID` | Yes | Rejected |

## Contract Addresses

| Contract | Address | Approval Needed |
|----------|---------|-----------------|
| pUSD | `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB` | — |
| USDC.e | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | Wrap source asset |
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | pUSD |
| CTF Exchange V2 | `0xE111180000d2663C0091e4f400237545B87B996B` | pUSD, tokens |
| Neg Risk CTF Exchange V2 | `0xe2222d279d744050d28e00520010520000310F59` | pUSD, tokens |
| Neg Risk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | tokens |
| Collateral Onramp | `0x93070a847efEf7F70739046A929D47a521F5B8ee` | approve USDC.e |
| Collateral Offramp | `0x2957922Eb93258b93368531d39fAcCA3B4dC5854` | approve pUSD |

## Notes

- keep Relayer API keys server-side
- relayer base URL remains `https://relayer-v2.polymarket.com/`
- for API-only users, wrap USDC.e to pUSD before trading if needed
