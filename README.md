# defi-data-validator

A Claude Code skill that validates DeFi/crypto enriched datasets by cross-referencing them against multiple authoritative sources: Dune Analytics, DefiLlama, DexScreener, DexPaprika, GeckoTerminal, The Graph, CoinGecko, Covalent/GoldRush, and Moralis.

## What it does

Use this skill when you want to test, validate, verify, or cross-check a DeFi dataset — token volumes, DEX trades, lending metrics, liquidity pools, TVL, yields, or any curated crypto dataset.

The skill runs a structured 4-phase validation workflow:

1. **Understand your dataset** — reads a sample, identifies columns, granularity, chains, and time range
2. **Discover reference data** — maps your columns to the best available reference source(s) and presents a validation plan
3. **Execute validation** — runs 5 checks: row count coverage, aggregate totals, time-series shape, spot-check records, and distribution sanity
4. **Report results** — delivers a structured report with pass/warn/fail status per check and root-cause recommendations

### Reference sources used

| Domain | Free Sources | Keyed Sources |
|--------|-------------|---------------|
| DEX trades / swaps | DexPaprika, GeckoTerminal, Dune | — |
| TVL / protocol fundamentals | DefiLlama | — |
| Token prices | DexScreener | CoinGecko |
| Yields / APY | DefiLlama | — |
| Lending | Dune (MCP) | The Graph |
| LP / pool analytics | GeckoTerminal, DexPaprika | — |
| Fees / revenue | DefiLlama | — |
| Wallet / portfolio | — | Covalent, Moralis |

## Installation

```bash
npx skill install iankressin/defi-data-validator
```

This installs the skill into `~/.claude/skills/defi-data-validator/` and registers it with Claude Code.

## Usage

Once installed, trigger the skill in any Claude Code session by describing your validation goal:

```
Validate my DEX trades dataset against Dune dex.trades
```

```
Cross-check my TVL dataset with DefiLlama for Ethereum protocols
```

```
Verify my lending data — I have a CSV with borrow events from Aave
```

Claude will automatically pick up the skill and walk you through the workflow.

## Requirements

- **Claude Code** CLI ([install](https://claude.ai/code))
- **Dune Analytics** account for MCP-integrated SQL validation (free tier: 2,500 credits/month)
- Optional: API keys for CoinGecko, Covalent/GoldRush, Moralis, The Graph (for extended coverage)

No keys are required to use DefiLlama, DexScreener, DexPaprika, or GeckoTerminal — these work out of the box.

## Skill files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill prompt — workflow phases, SQL patterns, mismatch handling |
| `REFERENCE.md` | Full source catalog — API endpoints, Dune Spellbook tables, schema mappings, validation thresholds |

## Validation thresholds

| Metric | Pass | Warn | Fail |
|--------|------|------|------|
| Row count diff | < 5% | 5–10% | > 10% |
| Aggregate total diff | < 2% | 2–5% | > 5% |
| Missing time periods | 0 | 1–2 gaps | > 2 gaps |
| Spot-check match rate | 100% | > 90% | < 90% |
| Distribution correlation | > 0.95 | 0.85–0.95 | < 0.85 |
