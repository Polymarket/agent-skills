# CTF Operations

The Conditional Token Framework (CTF) powers Polymarket outcome tokens. In V2, positions are collateralized with **pUSD**.

Core operations:
- **split** — pUSD → Yes + No
- **merge** — Yes + No → pUSD
- **redeem** — winning tokens → pUSD after resolution

## Token Model

Every binary market has two tokens:

| Token | Redeems for | Condition |
|-------|-------------|-----------|
| **Yes** | $1.00 pUSD | Event occurs |
| **No** | $1.00 pUSD | Event does not occur |

Every Yes/No pair is fully collateralized by pUSD locked in the CTF system.

## Split

Convert pUSD into a full set of outcome tokens.

```text
$100 pUSD → 100 Yes tokens + 100 No tokens
```

### Prerequisites
1. pUSD balance on Polygon
2. pUSD approval for the CTF contract
3. Market condition ID

### Function: `splitPosition`

| Parameter | Type | Value |
|-----------|------|-------|
| `collateralToken` | IERC20 | `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB` (pUSD) |
| `parentCollectionId` | bytes32 | `0x0000...0000` |
| `conditionId` | bytes32 | Market condition ID |
| `partition` | uint[] | `[1, 2]` for binary markets |
| `amount` | uint256 | Amount of pUSD to split |

## Merge

Convert a full set of outcome tokens back into pUSD.

```text
100 Yes + 100 No → $100 pUSD
```

### Function: `mergePositions`

Uses the same parameters as split. One full set burns into one unit of pUSD collateral.

## Redeem

After resolution, winning tokens redeem into pUSD.

```text
Market resolves YES:
100 Yes → $100 pUSD
100 No  → $0
```

### Function: `redeemPositions`

| Parameter | Type | Value |
|-----------|------|-------|
| `collateralToken` | IERC20 | pUSD |
| `parentCollectionId` | bytes32 | `0x0000...0000` |
| `conditionId` | bytes32 | Market condition ID |
| `indexSets` | uint[] | `[1, 2]` |

Redemption burns your full balance for that condition. There is no amount parameter.

## Payout Vectors

| Outcome | Payout Vector | Redemption |
|---------|---------------|------------|
| Yes wins | `[1, 0]` | Yes = $1, No = $0 |
| No wins | `[0, 1]` | Yes = $0, No = $1 |

## Contract Addresses

| Contract | Address | Purpose |
|----------|---------|---------|
| pUSD | `0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB` | Trading collateral |
| USDC.e | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | Wrap source asset for API-only flows |
| CTF | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | Token storage and split/merge/redeem |
| CTF Exchange V2 | `0xE111180000d2663C0091e4f400237545B87B996B` | Standard market trading |
| Neg Risk CTF Exchange V2 | `0xe2222d279d744050d28e00520010520000310F59` | Neg risk market trading |
| Neg Risk Adapter | `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` | Neg risk conversions |
| Collateral Onramp | `0x93070a847efEf7F70739046A929D47a521F5B8ee` | Wrap USDC.e → pUSD |
| Collateral Offramp | `0x2957922Eb93258b93368531d39fAcCA3B4dC5854` | Unwrap pUSD → USDC.e |

## Approval Matrix

| Operation | Contract to Approve | Token |
|-----------|---------------------|-------|
| Buy order (standard) | CTF Exchange V2 | pUSD |
| Sell order (standard) | CTF Exchange V2 | Conditional tokens |
| Buy order (neg risk) | Neg Risk CTF Exchange V2 | pUSD |
| Sell order (neg risk) | Neg Risk CTF Exchange V2 | Conditional tokens |
| Split | CTF | pUSD |
| Neg risk conversion | Neg Risk Adapter | Conditional tokens |
| Wrap | Collateral Onramp | USDC.e |
| Unwrap | Collateral Offramp | pUSD |

## Negative Risk

Neg risk markets are linked multi-outcome markets where a No token can convert into baskets of Yes tokens in other outcomes.

### Identifying a neg risk market

```json
{
  "negRisk": true
}
```

When placing orders, pass `negRisk: true` in order options.

## Token ID Computation

Position IDs are computed as:
1. `getConditionId(oracle, questionId, outcomeSlotCount)`
2. `getCollectionId(parentCollectionId, conditionId, indexSet)`
3. `getPositionId(collateralToken, collectionId)`

In V2, the `collateralToken` for Polymarket positions is **pUSD**.

In practice, prefer token IDs returned by the Gamma API `tokens` array.
