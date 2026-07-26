# Chainlink Integration Patterns

    This document covers how to use Polymarket's three active Chainlink integrations:
    1. **Price Feeds** — AggregatorV3Interface on Polygon (BTC, ETH, SOL, MATIC, LINK, USDC)
    2. **RTDS Chainlink feed** — Real-time interpolated oracle prices via WebSocket
    3. **Strike resolution** — Finding the canonical oracle round for BTC/ETH-updown markets

    ---

    ## 1. Chainlink Price Feeds (on-chain)

    Polymarket uses Chainlink oracles on Polygon mainnet to resolve crypto price markets.
    Reading them directly is free — no API key, no Polymarket auth required.

    ### Active feed addresses (Polygon mainnet, chain ID 137)

    | Pair | Address | Deviation | Heartbeat |
    |------|---------|-----------|-----------|
    | BTC/USD | `0xc907E116054Ad103354f2D350FD2514433D57F6f` | 0.5% | 3600s |
    | ETH/USD | `0xF9680D99D6C9589e2a93a78A04A279e509205945` | 0.5% | 3600s |
    | SOL/USD | `0x10C8264C0935b3B9870013e057f330Ff3e9C56dC` | 0.5% | 3600s |
    | MATIC/USD | `0xAB594600376Ec9fD91F8e885dADF0CE036862dE0` | 0.5% | 3600s |
    | LINK/USD | `0xd9FFdb71EbE7496cC440152d43986Aae0AB76665` | 0.5% | 3600s |
    | USDC/USD | `0xfE4A8cc5b5B2366C1B58Bea3858e81843581b2F7` | 0.1% | 86400s |

    Verify and discover feeds at [data.chain.link/polygon/mainnet](https://data.chain.link/polygon/mainnet).

    ### Reading the latest round (TypeScript / ethers v6)

    ```typescript
    import { ethers } from "ethers";

    const AGGREGATOR_ABI = [
    "function latestRoundData() view returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)",
    "function getRoundData(uint80) view returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)"
    ];

    const provider = new ethers.JsonRpcProvider("https://polygon-rpc.com");  // fallback: https://1rpc.io/matic
    const BTC_USD  = "0xc907E116054Ad103354f2D350FD2514433D57F6f";
    const feed     = new ethers.Contract(BTC_USD, AGGREGATOR_ABI, provider);

    const { roundId, answer, updatedAt } = await feed.latestRoundData();
    const price     = Number(answer) / 1e8;         // 8 decimal places for all crypto feeds
    const ageSeconds = Math.floor(Date.now() / 1000) - Number(updatedAt);

    console.log(`BTC/USD: $${price.toFixed(2)} (Round ${roundId}, ${ageSeconds}s ago)`);
    ```

    ### Reading the latest round (Python / web3.py)

    ```python
    from web3 import Web3

    w3 = Web3(Web3.HTTPProvider("https://polygon-rpc.com"))

    ABI = [
      {"inputs": [], "name": "latestRoundData", "outputs": [
          {"name": "roundId", "type": "uint80"}, {"name": "answer", "type": "int256"},
          {"name": "startedAt", "type": "uint256"}, {"name": "updatedAt", "type": "uint256"},
          {"name": "answeredInRound", "type": "uint80"}
      ], "stateMutability": "view", "type": "function"},
      {"inputs": [{"name": "_roundId", "type": "uint80"}], "name": "getRoundData", "outputs": [
          {"name": "roundId", "type": "uint80"}, {"name": "answer", "type": "int256"},
          {"name": "startedAt", "type": "uint256"}, {"name": "updatedAt", "type": "uint256"},
          {"name": "answeredInRound", "type": "uint80"}
      ], "stateMutability": "view", "type": "function"}
    ]

    feed = w3.eth.contract(address="0xc907E116054Ad103354f2D350FD2514433D57F6f", abi=ABI)
    latest = feed.functions.latestRoundData().call()
    price = latest[1] / 1e8
    print(f"BTC/USD: ${price:.2f}")
    ```

    ---

    ## 2. RTDS Chainlink Feed (WebSocket)

    The [real-time-data-client](https://github.com/Polymarket/real-time-data-client) provides a
    `crypto_prices_chainlink` topic that streams BTC/USD, ETH/USD, and SOL/USD prices at 1-second
    intervals via WebSocket.

    ### Critical: understand the gap model before using this feed

    Chainlink on-chain rounds only land when price moves ≥0.5% (deviation threshold) or 3600 seconds
    elapse (heartbeat). Between real rounds, the RTDS emits a hold value — the last confirmed on-chain
    `answer`, repeated every second.

    **The RTDS is NOT interpolating price between rounds — it is repeating the last oracle value.**

    This means:
    - You may see the same price for 2–10 seconds during low-volatility periods
    - Gaps in consecutive timestamps are not dropped ticks; they reflect periods where no on-chain round landed
    - The Binance feed is continuous because Binance streams every tick; Chainlink is event-driven

    ### Correct subscription format

    The `filters` field must be a **plain string**, not a JSON-encoded object or array:

    ```json
    {
    "action": "subscribe",
    "subscriptions": [
      {
        "topic": "crypto_prices_chainlink",
        "type": "*",
        "filters": "BTC/USD"
      }
    ]
    }
    ```

    For multiple symbols use a comma-separated string:

    ```json
    {
    "action": "subscribe",
    "subscriptions": [
      {
        "topic": "crypto_prices_chainlink",
        "type": "*",
        "filters": "BTC/USD,ETH/USD,SOL/USD"
      }
    ]
    }
    ```

    To unsubscribe, send the **identical** `filters` string you used on subscribe:

    ```json
    {
    "action": "unsubscribe",
    "subscriptions": [
      {
        "topic": "crypto_prices_chainlink",
        "type": "*",
        "filters": "BTC/USD"
      }
    ]
    }
    ```

    > **Known SDK issue**: rs-clob-client-v2 ≤0.7.0 serializes `filters` as a JSON object instead
    > of a plain string, causing silent subscription failures. See
    > [#341](https://github.com/Polymarket/rs-clob-client/pull/341) and
    > [#90](https://github.com/Polymarket/rs-clob-client-v2/issues/90).

    ---

    ## 3. Strike Price Resolution for BTC/ETH-updown Markets

    BTC-updown-15m and similar markets encode the window start timestamp in their slug
    (e.g. `btc-updown-15m-1777889700`). The resolution contract reads the Chainlink round
    whose `updatedAt ≤ windowStartUnix` and records its `answer / 1e8` as the strike price
    for the entire window. **The strike is fixed at window open and does not change.**

    ### Finding the strike round (TypeScript)

    ```typescript
    import { ethers } from "ethers";

    const AGGREGATOR_ABI = [
    "function latestRoundData() view returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)",
    "function getRoundData(uint80) view returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)"
    ];

    async function getStrikeRound(pair: "BTC/USD" | "ETH/USD" | "SOL/USD", windowStartUnix: number) {
    const FEED_ADDRESSES: Record<string, string> = {
      "BTC/USD": "0xc907E116054Ad103354f2D350FD2514433D57F6f",
      "ETH/USD": "0xF9680D99D6C9589e2a93a78A04A279e509205945",
      "SOL/USD": "0x10C8264C0935b3B9870013e057f330Ff3e9C56dC",
    };

    const provider = new ethers.JsonRpcProvider("https://polygon-rpc.com");
    const feed     = new ethers.Contract(FEED_ADDRESSES[pair], AGGREGATOR_ABI, provider);

    const latest   = await feed.latestRoundData();
    const phaseId  = latest.roundId >> 64n;
    const aggRound = latest.roundId & 0xFFFFFFFFFFFFFFFFn;

    // Binary search: find the most-recent round whose updatedAt <= windowStartUnix
    let lo = 0n, hi = BigInt(Math.min(2000, Number(aggRound) - 1));
    let strikeRound = null;

    while (lo <= hi) {
      const mid = (lo + hi) / 2n;
      const rid = (phaseId << 64n) | (aggRound - mid);
      const r   = await feed.getRoundData(rid);
      if (Number(r.updatedAt) <= windowStartUnix) {
        strikeRound = r;
        hi = mid - 1n;          // try a more recent round
      } else {
        lo = mid + 1n;           // go further back
      }
    }

    if (!strikeRound) throw new Error("Strike round not found within search window");

    const strikePrice = Number(strikeRound.answer) / 1e8;
    const currentPrice = Number(latest.answer) / 1e8;
    const direction   = currentPrice > strikePrice ? "UP" : currentPrice < strikePrice ? "DOWN" : "FLAT";
    const changePct   = ((currentPrice - strikePrice) / strikePrice) * 100;

    return {
      strikePrice,
      strikeRoundId: strikeRound.roundId.toString(),
      strikeUpdatedAt: Number(strikeRound.updatedAt),
      currentPrice,
      direction,
      changePct: changePct.toFixed(4),
    };
    }

    // Example: parse window start from market slug
    const slug = "btc-updown-15m-1777889700";
    const windowStart = parseInt(slug.split("-").pop()!);
    const result = await getStrikeRound("BTC/USD", windowStart);
    console.log(result);
    // { strikePrice: 95432.12, direction: "UP", changePct: "+0.8241", ... }
    ```

    ### Finding the strike round (Python)

    ```python
    def get_strike_round(feed, window_start_unix: int):
      """Binary search for the Chainlink round active at window open."""
      latest    = feed.functions.latestRoundData().call()
      phase_id  = latest[0] >> 64
      agg_round = latest[0] & 0xFFFFFFFFFFFFFFFF

      lo, hi, strike = 0, min(2000, agg_round - 1), None
      while lo <= hi:
          mid = (lo + hi) // 2
          rid = (phase_id << 64) | (agg_round - mid)
          r = feed.functions.getRoundData(rid).call()
          if r[3] <= window_start_unix:   # updatedAt <= windowStart
              strike = r
              hi = mid - 1               # try more recent
          else:
              lo = mid + 1               # go further back

      return strike

    # Usage
    import re
    slug = "btc-updown-15m-1777889700"
    window_start = int(re.search(r'(\d+)$', slug).group(1))
    strike = get_strike_round(feed, window_start)
    if strike:
      strike_price = strike[1] / 1e8
      current      = feed.functions.latestRoundData().call()[1] / 1e8
      direction    = "UP" if current > strike_price else "DOWN"
      print(f"Strike: ${strike_price:.2f} | Current: ${current:.2f} | {direction}")
    ```

    ---

    ## 4. Agent Patterns

    ### Pattern 1 — Check oracle direction before placing a resolution trade

    ```typescript
    import { ClobClient } from "@polymarket/clob-client";

    async function tradeResolution(clobClient: ClobClient, marketSlug: string) {
    // 1. Parse window start from slug
    const windowStart = parseInt(marketSlug.split("-").pop()!);
    const pair = marketSlug.startsWith("btc") ? "BTC/USD" : "ETH/USD";

    // 2. Get oracle state
    const { direction, changePct, strikePrice, currentPrice } = await getStrikeRound(pair, windowStart);

    // 3. Only trade if oracle signal is clear (>0.5% from strike = beyond deviation threshold)
    const clearSignal = Math.abs(parseFloat(changePct)) > 0.5;
    if (!clearSignal) {
      console.log("Oracle within 0.5% of strike — skip (too close to call before next round)");
      return;
    }

    // 4. Place directional trade
    const side = direction === "UP" ? "YES" : "NO";
    console.log(`Oracle: ${direction} | ${changePct}% | Placing ${side}`);
    // await clobClient.createOrder({ ... })
    }
    ```

    ### Pattern 2 — Monitor round updates with a deviation alert

    ```typescript
    async function monitorRoundUpdates(pair: string, onNewRound: (round: any) => void) {
    const feed    = /* ... ethers contract */;
    let lastRound = await feed.latestRoundData();

    setInterval(async () => {
      const latest = await feed.latestRoundData();
      if (latest.roundId !== lastRound.roundId) {
        const devPct = Math.abs(Number(latest.answer) - Number(lastRound.answer))
                       / Number(lastRound.answer) * 100;
        console.log(`New round: ${latest.roundId} | ${devPct.toFixed(3)}% deviation`);
        onNewRound(latest);
        lastRound = latest;
      }
    }, 5_000); // poll every 5s — Chainlink rounds land ~every 2–3600s
    }
    ```

    ### Pattern 3 — Jump persistence signal

    Research on Chainlink BTC/USD on Polygon shows that consecutive oracle jumps continue
    in the same direction ~65–75% of the time during trending markets. This can be used
    as a weak directional prior when combined with orderbook depth:

    ```typescript
    async function jumpPersistence(feed: ethers.Contract, lookback: number = 20): Promise<number> {
    const latest   = await feed.latestRoundData();
    const phaseId  = latest.roundId >> 64n;
    const aggRound = latest.roundId & 0xFFFFFFFFFFFFFFFFn;

    const rounds = [latest];
    for (let i = 1n; i < BigInt(lookback); i++) {
      const rid = (phaseId << 64n) | (aggRound - i);
      try { rounds.push(await feed.getRoundData(rid)); } catch { break; }
    }

    const sorted = rounds.sort((a, b) => Number(a.updatedAt) - Number(b.updatedAt));
    let same = 0, total = 0;
    for (let i = 1; i < sorted.length - 1; i++) {
      const prev = Number(sorted[i - 1].answer) - Number(sorted[i].answer);
      const curr = Number(sorted[i].answer) - Number(sorted[i + 1].answer);
      if (prev !== 0 && curr !== 0) {
        if ((prev > 0) === (curr > 0)) same++;
        total++;
      }
    }
    return total === 0 ? 0.5 : same / total;
    }
    ```

    ---

    ## 5. Resolution Verification

    To verify a resolved market's outcome against the on-chain Chainlink data:

    1. Get the `conditionId` from the Gamma API (`GET /markets/:id`)
    2. Query the UMA `Optimistic Oracle V2` or the CTF Resolution contract for the `payoutNumerators`
    3. Cross-check against the Chainlink round active at `windowStartUnix`

    The [resolution-subgraph](https://github.com/Polymarket/resolution-subgraph) indexes all
    market resolutions. For BTC/ETH-updown markets, the resolution proof is the Chainlink
    round ID and answer stored on-chain at the time of the `resolve()` call.

    ```typescript
    // Quick sanity-check: does the oracle agree with the resolved outcome?
    const { strikePrice, currentPrice, direction } = await getStrikeRound("BTC/USD", windowStart);
    const resolvedYes = payoutNumerators[0] > 0n;  // from CTF contract
    const oracleAgrees = (direction === "UP") === resolvedYes;
    console.log(`Oracle agrees with resolution: ${oracleAgrees}`);
    ```

    ---

    ## Further reading

    - [Chainlink Data Feeds docs](https://docs.chain.link/data-feeds)
    - [AggregatorV3Interface reference](https://docs.chain.link/data-feeds/api-reference)
    - [Polymarket RTDS WebSocket docs](https://docs.polymarket.com/market-data/websocket/rtds)
    - [real-time-data-client](https://github.com/Polymarket/real-time-data-client)
    - [resolution-subgraph](https://github.com/Polymarket/resolution-subgraph)
    - [ctf-exchange-v2](https://github.com/Polymarket/ctf-exchange-v2)
    