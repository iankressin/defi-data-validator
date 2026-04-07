---
name: defi-data-validator
description: Validate DeFi/crypto enriched datasets against reference sources (Dune Analytics, DefiLlama, DexScreener, DexPaprika, GeckoTerminal, The Graph, CoinGecko, Covalent, Moralis). Use when user wants to test, validate, verify, or cross-check a DeFi dataset — token volumes, DEX trades, lending metrics, liquidity pools, TVL, yields, or any curated crypto dataset.
---

# DeFi Data Validator

Validates user-provided enriched DeFi/crypto datasets by cross-referencing against multiple sources: Dune Analytics (MCP-integrated), DefiLlama, DexScreener, DexPaprika, GeckoTerminal, The Graph, CoinGecko, Covalent/GoldRush, and Moralis. See [REFERENCE.md](REFERENCE.md) for the full source catalog, API patterns, and when to use each.

## Workflow

Run these phases sequentially. Present the plan after Phase 2 and wait for user confirmation before executing Phase 3+.

### Phase 1 — Understand the User's Dataset

1. Ask the user to provide or describe their dataset:
   - File path (CSV/JSON/Parquet), database table, or inline sample
   - What it represents (DEX trades, lending positions, token transfers, LP activity, etc.)
   - Chain(s) covered
   - Time range
   - Key columns and their meaning

2. Read a sample of the data (first 20-50 rows) to understand:
   - Column names, types, and value ranges
   - Granularity (per-trade, hourly, daily aggregates)
   - Whether amounts are in raw units, decimals-adjusted, or USD-denominated
   - Any protocol-specific fields

3. **Ask the user about the block/time range to use for validation**:

   > "Should the validation use a specific block range or time window, or should I detect the available range directly from your dataset?"

   - **Option A — Specific range**: User provides explicit bounds (e.g., block 19000000–19500000, or 2024-01-01 to 2024-01-31). Use these exact bounds when querying reference sources.
   - **Option B — Dataset range**: Inspect the dataset to find `MIN` and `MAX` of the block number or timestamp column and use those as the validation window.

   Once the range is confirmed, state it explicitly before proceeding to Phase 2 (e.g., "Validating over blocks 19000000–19500000 / 2024-01-01 to 2024-01-31").

4. **Check if the dataset has data for the validated range**:

   Query (or inspect) the dataset for the confirmed range. If the result is empty or has insufficient rows:

   **Phase 1.5 — Collect Missing Data**

   a. **Understand the codebase** — explore the project directory to identify:
      - The indexer/pipeline entrypoint (look for `package.json` scripts, `Makefile`, `README`, `docker-compose.yml`, CLI entrypoints)
      - How to configure the block/time range (env vars, config files, CLI flags)
      - Where output data is written (database connection string, output path)

   b. **Confirm before running** — tell the user what command will be executed and ask for approval:
      > "No data found for the requested range. I'll run `<command>` to collect it. Proceed?"

   c. **Run the project** to collect data for the required range using the Bash tool. Stream or tail output to confirm progress. Wait for completion.

   d. **Re-verify** that data now exists for the range. If still empty after collection, stop and report the failure to the user with the relevant logs before proceeding.

   e. Continue to Phase 2 once data is confirmed present.

### Phase 2 — Discover & Map Reference Data

1. **Pick the best reference source(s)** based on the data domain (see [REFERENCE.md — Source Selection Guide](REFERENCE.md)):

   | Data Domain | Primary Source | Secondary Source |
   |-------------|---------------|-----------------|
   | DEX trades / swaps | Dune `dex.trades` (MCP) | DexPaprika or GeckoTerminal (free REST) |
   | TVL / protocol fundamentals | DefiLlama (free, no key) | Dune spells |
   | Yields / APY | DefiLlama `/yields` | Dune |
   | Token prices | DexScreener (free, 300 req/min) | CoinGecko or `prices.usd` on Dune |
   | Lending (borrow/repay) | Dune `lending.*` (MCP) | The Graph subgraphs |
   | LP / pool analytics | GeckoTerminal or DexPaprika | Dune decoded tables |
   | Protocol-specific deep data | The Graph subgraphs | Dune decoded tables |
   | Fees / revenue | DefiLlama `/fees` | Dune spells |
   | Stablecoin supply | DefiLlama `/stablecoins` | Dune |
   | Historical wallet/portfolio | Covalent/GoldRush | Moralis |

   Use **multiple sources** when possible — agreement across 2+ independent sources is a stronger signal than matching just one.

2. **Search for matching datasets**:
   - **Dune (MCP)**: Use `searchTables` with keywords, `categories: ["spell"]` first, then `["decoded"]`. Filter by chain. Use `searchTablesByContractAddress` for specific contracts. Enable `includeSchema: true` and `includeMetadata: true`.
   - **REST APIs**: Use `WebFetch` to hit free API endpoints (see REFERENCE.md for URL patterns). No auth needed for DefiLlama, DexScreener, DexPaprika, GeckoTerminal.
   - **Signup-required APIs**: If the user has API keys for CoinGecko, Covalent, Moralis, or The Graph, use those too.

3. **Build a schema mapping** between the user's dataset and each reference source:
   - Match columns by semantic meaning, not just name (e.g., user's `amount` → Dune's `token_bought_amount` → DefiLlama's `totalVolume`)
   - Document unit differences (raw vs decimal-adjusted, wei vs ETH, cents vs USD)
   - Note any transformations needed (aggregation level, time bucketing, token symbol → address resolution)
   - Flag columns in the user's data that have NO reference equivalent (cannot be validated)
   - Flag reference columns missing from the user's data (potential gaps)

4. **Present the validation plan** to the user:
   - Which reference sources and tables/endpoints will be used
   - The column mapping table (user column → reference column → transformation needed)
   - Which validation checks will run (see Phase 3)
   - Any limitations or caveats
   - **Wait for user confirmation before proceeding**

### Phase 3 — Execute Validation

Run checks using the selected sources:
- **Dune**: `createDuneQuery` + `executeQueryById` + `getExecutionResults` (MCP tools)
- **Free REST APIs**: `WebFetch` to call DefiLlama/DexScreener/DexPaprika/GeckoTerminal endpoints
- **Keyed APIs**: `WebFetch` with user-provided API keys in headers

#### Check 1: Row Count / Coverage
Compare total record counts for the same time range and filters.
```
Expected: counts within 5% of each other
Red flag: >10% difference suggests missing data
```

#### Check 2: Aggregate Totals
Sum key numeric columns (volume, amounts, fees) over the full period.
```
Expected: totals within 2-5% (small differences from timing/indexing lag)
Red flag: >10% suggests unit errors or missing records
```

#### Check 3: Time-Series Shape
Compare daily/hourly aggregates to detect:
- Missing time periods in the user's data
- Spikes or drops that appear in one source but not the other
- Systematic offset (user data consistently higher/lower)

#### Check 4: Spot-Check Specific Records
For datasets with transaction hashes or unique IDs:
- Sample 10-20 specific records and compare field-by-field
- Verify amounts, addresses, timestamps match exactly (after applying the mapping)

#### Check 5: Distribution Sanity
Compare value distributions (top tokens, top addresses, min/max/median amounts) to catch:
- Filtering differences (e.g., one source excludes dust transactions)
- Token decimal errors (off by 10^N)

### Phase 4 — Report Results

Present a structured validation report in the conversation:

```
## Validation Report

**Dataset**: [name/description]
**Reference**: [Dune table(s) used]
**Period**: [date range]
**Chain(s)**: [chains]

### Results Summary
| Check              | Status | Details                    |
|--------------------|--------|----------------------------|
| Row Count          | ✅/⚠️/❌ | X vs Y (Z% diff)         |
| Aggregate Totals   | ✅/⚠️/❌ | ...                       |
| Time-Series Shape  | ✅/⚠️/❌ | ...                       |
| Spot-Check Records | ✅/⚠️/❌ | ...                       |
| Distribution       | ✅/⚠️/❌ | ...                       |

### Schema Mapping Used
[mapping table]

### Issues Found
[detailed list of discrepancies with possible root causes]

### Recommendations
[suggested fixes or further investigation steps]
```

Then write the same report to a file using the Write tool:
- **Filename**: `validation-report-<dataset-slug>-<YYYY-MM-DD>.md` in the current working directory
- **Content**: identical to the report shown in the conversation, plus a header line: `Generated by defi-data-validator on <ISO timestamp>`
- After writing, tell the user the file path.

## Key Dune SQL Patterns

When writing validation queries, use DuneSQL dialect:

- **Always filter on partition columns** (`block_date`, `block_time`) to reduce cost
- Common Spellbook tables: `dex.trades`, `dex.aggregator_trades`, `lending.borrow`, `lending.repay`, `lending.flashloans`, `tokens.erc20`, `prices.usd`
- Use `INTERVAL '7' DAY` syntax (not `INTERVAL 7 DAY`)
- Cast addresses: `FROM_HEX('0x...')` or `0x...` literals
- Decimal handling: `token_bought_amount` in Spellbook is already decimal-adjusted; `token_bought_amount_raw` is in wei/raw units

## Handling Common Mismatches

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Values off by 10^6/10^18 | Decimal adjustment mismatch | Check if comparing raw vs adjusted amounts |
| Consistent 1-5% difference | Timing/block finality | Acceptable — note in report |
| Missing recent data | Indexing lag | Exclude last 1-2 hours from comparison |
| Row count way off | Different filtering criteria | Check for dust filters, min amounts, token whitelists |
| USD values diverge | Different price sources | Compare underlying token amounts instead |

## See Also

- [REFERENCE.md](REFERENCE.md) — Full source catalog, API endpoints, Dune Spellbook tables, and schema mappings
