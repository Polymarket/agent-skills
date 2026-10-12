# Wallet intelligence — the decision side of Polymarket data

`authentication.md`, `order-patterns.md` and `market-data.md` cover everything a
Polymarket agent needs to **read markets and execute**. This page covers the
question those APIs do not answer, and the one most agents get wrong:

> **Who is on the other side of this trade, and is their edge real?**

Polymarket's own APIs return order books, prices, and (via the Data API) a
wallet's trades and positions. They return **no judgement** about that wallet:
no sample-size correction, no bot/human verdict, no view of which wallets act as
one coordinated group. An agent that treats a 100% win rate over eight bets as
skill will copy noise.

This page is the reference for adding that layer without building a Polygon
indexer.

---

## What the official APIs give you

| Source | Gives | Does not give |
| --- | --- | --- |
| Gamma `gamma-api.polymarket.com` | Events, markets, prices, resolution metadata | Any wallet attribute |
| Data API `data-api.polymarket.com` | A wallet's trades and positions | Whether the wallet is skilled, a bot, or part of a cluster |
| CLOB `clob.polymarket.com` | Order book, spreads, midpoints | — |
| Subgraph | Historical fills | Derived wallet statistics |

Everything below is derived from those same on-chain fills, which is why the two
sets are complements, not competitors: **use the official APIs for state, and an
intelligence layer for judgement.**

---

## The three metrics worth having

### 1. Raw win rate cannot rank anyone — use a conditional estimate

Win rate = wins / resolved bets. Two failure modes make it useless as a sort key:

- **Small samples.** 8 wins from 8 bets is 100% and means nothing. Positions
  inside one event are correlated — twenty legs of one date ladder are one
  opinion, not twenty bets.
- **Price asymmetry.** Buying at 99.6¢ wins ~99.86% of the time and can still
  finish **negative**: 0.4¢ of upside against 99.6¢ of downside.

What to ask for instead: an expected-value-per-dollar estimate with a
confidence-interval lower bound, computed with an **event-clustered effective
sample size** (positions in one event count as fully correlated). Rank on the
lower bound, never on the point estimate.

> **Rule: win rate may filter a board. It can never sort one.**

### 2. One bot / human / maker verdict per wallet

A wallet that is actually an exchange-funded market maker is not a signal. Ask
for a single verdict — `human`, `bot`, `maker`, `unknown` — **with the evidence
that produced it** (lifetime fill count, distinct markets, maker-vs-taker rebate
ratio). Treat `unknown` as unknown; rendering it as "human" turns missing data
into a claim.

### 3. Funding clusters, not just whale lists

Wallets funded from the same USDC source are one entity. But a 100+ wallet
cluster from Polymarket's own infrastructure is **plumbing, not collusion** —
clusters should score *down* for known infra funders and only raise suspicion
when a small group of unknown proxies shares real category edge. Any cluster
tool worth using distinguishes the two.

---

## Wiring it into an agent

CrowdIntel exposes this layer as an MCP server (19 read-only tools over
streamable HTTP) and a REST API, so an agent can go from "this market moved" to
"these wallets are sharp and here is the counter-case" without writing an
indexer:

```json
{
  "mcpServers": {
    "crowdintel": {
      "type": "http",
      "url": "https://mcp.crowdintel.xyz/mcp",
      "headers": { "Authorization": "Bearer cint_mcp_YOUR_TOKEN_HERE" }
    }
  }
}
```

- MCP endpoint: `https://mcp.crowdintel.xyz/mcp` (free token, no card)
- Server card: `https://mcp.crowdintel.xyz/.well-known/mcp.json`
- REST (same data): `https://crowdintel.xyz/api/v1` — OpenAPI at `/openapi.json`
- Skills, examples and the honest source comparison:
  <https://github.com/crowdintel-xyz/polymarket-agent-kit>
- Formulas: <https://github.com/crowdintel-xyz/polymarket-agent-kit/blob/main/docs/metrics.md>
- Why raw win rates lie: <https://github.com/crowdintel-xyz/polymarket-agent-kit/blob/main/docs/pitfalls.md>

The MCP server is **read-only by design** — it holds no custody and places no
orders. Keep execution in the CLOB client documented in `order-patterns.md`:
*decide with intelligence data, execute with the official client.*

## Agent pattern

```
1. get_market_movers                     -> what is moving right now
2. get_market_flow(market)               -> smart money vs crowd delta
3. get_wallet_stats(address)             -> is the wallet behind it skilled?
                                            (bayesian win rate, trader_kind,
                                             entry-timing edge, biggest loss)
4. scan_insider_signals / get_cluster    -> is this several wallets, one actor?
5. order-patterns.md (CLOB client)       -> only now place the order
```

Step 3 is the one agents skip, and the one that stops an agent from copying a
coin flip. Always state the sample size and the counter-case (largest loss,
category where the wallet loses) alongside any positive result.

## Data licensing

CrowdIntel is an independent data provider, not affiliated with or endorsed by
Polymarket. If you would rather compute all of this yourself, the raw inputs are
public: the Subgraph and the Data API give you the fills — the statistics above
are formulas, documented in the links above, and free to reimplement.
