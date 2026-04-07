# DeFi Data Validator — Reference

## Reference Data Sources

### Fully Free (No API Key)

**DefiLlama** — TVL, yields, fees, DEX volumes, stablecoin supply, bridge data.
```
Base URL: https://api.llama.fi
GET /v2/protocols                        → All protocols with TVL
GET /v2/historicalChainTvl/{chain}       → Historical TVL by chain
GET /protocol/{protocol}                 → Protocol TVL breakdown
GET /overview/dexs                       → DEX volume overview
GET /overview/dexs/{chain}               → DEX volume by chain
GET /overview/fees                       → Fees/revenue overview
GET /overview/fees/{protocol}            → Protocol fees
GET /yields/pools                        → All yield pools
GET /stablecoins                         → Stablecoin supply
GET /bridges                             → Bridge volumes
No auth. Generous rate limits. Best for: protocol fundamentals, TVL, yields, fees.
```

**DexScreener** — Real-time DEX prices and pair data across 80+ chains.
```
Base URL: https://api.dexscreener.com
GET /latest/dex/pairs/{chainId}/{pairAddress}   → Pair details
GET /latest/dex/tokens/{tokenAddress}           → Token pairs
GET /latest/dex/search?q={query}                → Search pairs
No auth. 300 req/min. Best for: real-time price lookups, pair discovery.
Note: Limited historical data.
```

**DexPaprika** — DEX prices, OHLCV, pool analytics, 29+ chains, 15M+ tokens.
```
Base URL: https://api.dexpaprika.com
GET /networks                                   → List networks
GET /networks/{network}/pools                   → Pools on a network
GET /networks/{network}/pools/{pool}/ohlcv      → OHLCV data
GET /networks/{network}/tokens/{address}        → Token details
GET /search?query={query}                       → Search
No auth. Generous limits. Free SSE streaming. Best for: OHLCV, pool analytics.
```

**GeckoTerminal** — On-chain DEX data from 260+ networks.
```
Base URL: https://api.geckoterminal.com/api/v2
GET /networks                                           → List networks
GET /networks/{network}/pools/{address}                 → Pool details
GET /networks/{network}/tokens/{address}                → Token details
GET /networks/{network}/pools/{address}/ohlcv/{period}  → OHLCV
GET /networks/{network}/tokens/{address}/pools          → Token pools
No auth. 10 req/min (free). Best for: on-chain pool analytics, DEX data.
```

### Free Tier with Signup

**Dune Analytics** — Custom SQL over 100+ chains of decoded data. MCP-integrated.
```
2,500 free credits/month. Use Dune MCP tools directly:
- searchTables → discover tables
- createDuneQuery → write SQL
- executeQueryById → run query
- getExecutionResults → fetch results
Best for: custom SQL validation, decoded event data, Spellbook aggregates.
```

**The Graph** — Protocol-specific subgraphs via GraphQL.
```
100k free queries/month (development). Production requires GRT.
Query deployed subgraphs for Uniswap, Aave, Compound, Curve, etc.
Best for: deep protocol-specific data, real-time indexing.
```

**CoinGecko** — Broad market data, prices, OHLC.
```
Base URL: https://api.coingecko.com/api/v3
GET /simple/price?ids={ids}&vs_currencies=usd
GET /coins/{id}/market_chart?vs_currency=usd&days={days}
GET /coins/{id}/ohlc?vs_currency=usd&days={days}
Free: 30 calls/min, ~10k/month. Optional Demo API key.
```

**Covalent/GoldRush** — Full chain history, wallet analytics, DEX swaps.
```
25k free credits/month, 4 req/sec. 100+ chains.
Best for: historical depth, portfolio analytics, wallet-level data.
```

**Moralis** — Multichain token prices, DeFi positions, NFTs, wallets.
```
~40k free compute units/day.
Best for: multichain coverage, batch queries, wallet positions.
```

### Source Selection Guide

| Data Domain | Primary (Free) | Secondary | Dune Table |
|-------------|----------------|-----------|------------|
| DEX trades/swaps | DexPaprika or GeckoTerminal | DexScreener | `dex.trades` |
| DEX volume aggregates | DefiLlama `/overview/dexs` | DexPaprika | `dex.trades` (SUM) |
| TVL | DefiLlama `/v2/protocols` | — | — |
| Yields/APY | DefiLlama `/yields/pools` | — | — |
| Token prices | DexScreener | CoinGecko | `prices.usd` |
| OHLCV | DexPaprika or GeckoTerminal | CoinGecko | — |
| Lending | The Graph subgraphs | Dune MCP | `lending.borrow/repay` |
| Fees/revenue | DefiLlama `/overview/fees` | — | — |
| Stablecoin supply | DefiLlama `/stablecoins` | — | — |
| Wallet/portfolio | Covalent | Moralis | — |
| Protocol-specific | The Graph | Dune decoded | varies |

---

## Common Dune Spellbook Tables for Validation

These are the most useful curated tables. Always use `searchTables` with `categories: ["spell"]` to discover the latest available tables — this list is a starting point, not exhaustive.

### DEX / Trading
| Table | Description | Key Columns |
|-------|-------------|-------------|
| `dex.trades` | Aggregated DEX trades across protocols | `block_date`, `blockchain`, `project`, `token_bought_symbol`, `token_bought_amount`, `token_sold_symbol`, `token_sold_amount`, `amount_usd`, `tx_hash` |
| `dex.aggregator_trades` | Trades via aggregators (1inch, Paraswap, etc.) | Same structure as `dex.trades` |
| `dex.prices` | DEX-derived token prices | `block_date`, `blockchain`, `token_address`, `median_price` |

### Lending
| Table | Description | Key Columns |
|-------|-------------|-------------|
| `lending.borrow` | Borrow events across lending protocols | `block_date`, `blockchain`, `project`, `token_address`, `token_symbol`, `amount`, `amount_usd`, `borrower` |
| `lending.repay` | Repayment events | Similar to `lending.borrow` |
| `lending.flashloans` | Flash loan events | `block_date`, `blockchain`, `project`, `amount`, `amount_usd` |

### Tokens & Prices
| Table | Description | Key Columns |
|-------|-------------|-------------|
| `tokens.erc20` | ERC20 token registry | `blockchain`, `contract_address`, `symbol`, `decimals` |
| `prices.usd` | Token prices (minute-level) | `minute`, `blockchain`, `contract_address`, `symbol`, `price` |
| `prices.usd_daily` | Daily token prices | `day`, `blockchain`, `contract_address`, `symbol`, `price` |

### Transfers & Balances
| Table | Description | Key Columns |
|-------|-------------|-------------|
| `tokens.transfers` | ERC20 transfer events | `block_date`, `blockchain`, `token_address`, `from`, `to`, `amount_raw` |
| `tokens.balances_daily` | Daily token balances | `day`, `blockchain`, `token_address`, `address`, `amount_raw`, `amount` |

### NFT
| Table | Description | Key Columns |
|-------|-------------|-------------|
| `nft.trades` | NFT marketplace trades | `block_date`, `blockchain`, `project`, `nft_contract_address`, `token_id`, `amount_usd`, `buyer`, `seller` |

## Schema Mapping Patterns

### Amount Fields
```
User's field          → Likely Dune equivalent
──────────────────────────────────────────────
amount                → token_bought_amount (if decimal-adjusted)
amount_raw / amount_wei → token_bought_amount_raw
volume_usd / usd_amount → amount_usd
price                 → Check prices.usd or the trade's implicit price
fee / fee_amount      → Varies by protocol — check decoded tables
```

### Address Fields
```
User's field          → Likely Dune equivalent
──────────────────────────────────────────────
token / token_address → token_bought_address or token_sold_address
pair / pool           → project_contract_address
wallet / user         → taker (for dex.trades), borrower (for lending)
sender / from         → tx_from
```

### Time Fields
```
User's field          → Likely Dune equivalent
──────────────────────────────────────────────
timestamp / time      → block_time (timestamp type)
date / day            → block_date (date type)
block / block_number  → block_number (bigint)
```

## DuneSQL Quick Reference

```sql
-- Date filtering (always use for partition pruning)
WHERE block_date >= DATE '2024-01-01'
WHERE block_date >= CURRENT_DATE - INTERVAL '30' DAY

-- Aggregation with date truncation
SELECT date_trunc('day', block_time) AS day, SUM(amount_usd) AS volume
FROM dex.trades
WHERE blockchain = 'ethereum'
  AND block_date >= CURRENT_DATE - INTERVAL '7' DAY
GROUP BY 1

-- Cross-check row counts
SELECT COUNT(*) AS row_count,
       SUM(amount_usd) AS total_volume,
       MIN(block_time) AS earliest,
       MAX(block_time) AS latest
FROM dex.trades
WHERE blockchain = 'ethereum'
  AND project = 'uniswap'
  AND block_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'

-- Spot-check by tx hash
SELECT * FROM dex.trades
WHERE tx_hash = 0xabc...
  AND block_date = DATE '2024-01-15'

-- Distribution check
SELECT token_bought_symbol,
       COUNT(*) AS trade_count,
       SUM(amount_usd) AS total_volume
FROM dex.trades
WHERE blockchain = 'ethereum'
  AND block_date >= CURRENT_DATE - INTERVAL '7' DAY
GROUP BY 1
ORDER BY 3 DESC
LIMIT 20
```

## Validation Thresholds

| Metric | Green (✅) | Yellow (⚠️) | Red (❌) |
|--------|-----------|-------------|---------|
| Row count diff | < 5% | 5-10% | > 10% |
| Aggregate total diff | < 2% | 2-5% | > 5% |
| Missing time periods | 0 | 1-2 gaps | > 2 gaps |
| Spot-check match rate | 100% | > 90% | < 90% |
| Distribution correlation | > 0.95 | 0.85-0.95 | < 0.85 |

Note: Thresholds are starting points. Adjust based on:
- Data freshness (more tolerance for recent data due to indexing lag)
- Aggregation level (daily aggregates should match tighter than per-trade)
- Protocol complexity (multi-hop swaps may split differently across sources)
