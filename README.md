# HARS-Zero

> Autonomous Polymarket prediction-market intelligence. Pay-per-call in USDC. No accounts. No API keys. Just HTTP 402.

[![x402](https://img.shields.io/badge/payment-x402-blue)](https://www.x402.org/)
[![network](https://img.shields.io/badge/network-base%20%2B%20solana-green)]()
[![status](https://img.shields.io/badge/status-operational-brightgreen)](https://identify-nut-ronald-proprietary.trycloudflare.com/health)
[![llms.txt](https://img.shields.io/badge/-/llms.txt-orange)](https://identify-nut-ronald-proprietary.trycloudflare.com/llms.txt)

**Public endpoint:** `https://identify-nut-ronald-proprietary.trycloudflare.com`

**Discovery files (machine-readable):**
- [`llms.txt`](https://identify-nut-ronald-proprietary.trycloudflare.com/llms.txt) — LLM-friendly manifest
- [`/.well-known/tools.json`](https://identify-nut-ronald-proprietary.trycloudflare.com/.well-known/tools.json) — OpenAI tool-call format
- [`/.well-known/x402-payment.json`](https://identify-nut-ronald-proprietary.trycloudflare.com/.well-known/x402-payment.json) — x402 payment requirements
- [`/sitemap.xml`](https://identify-nut-ronald-proprietary.trycloudflare.com/sitemap.xml)
- [`/robots.txt`](https://identify-nut-ronald-proprietary.trycloudflare.com/robots.txt)
- [`/v1/health.json`](https://identify-nut-ronald-proprietary.trycloudflare.com/v1/health.json) — uptime + wallet info
- [`/v1/stats.json`](https://identify-nut-ronald-proprietary.trycloudflare.com/v1/stats.json) — real on-chain revenue

---

## What is this?

HARS-Zero is a **pay-per-call intelligence API** for Polymarket prediction markets. AI agents and automated trading systems get fresh market data — top movers, mispricing, sentiment, whale activity, depth snapshots, curated bundles — by paying micro-amounts of USDC.

**The whole point:** no signup, no rate-limit dance, no API key rotation. Payment IS authentication. The HTTP 402 protocol handles the rest.

```
GET /paid/market-brief
→ 402 Payment Required
   X-PAYMENT: <signed EIP-3009 transfer or Solana tx signature>
→ 200 OK + JSON payload
```

## Endpoints (17 paid + 5 free)

| Endpoint | Price | Description |
|---|---|---|
| `GET /free/market-brief` | FREE | 5 active markets, watermarked |
| `GET /free/structured-feed` | FREE | 10 markets, watermarked |
| `GET /v1/markets` | FREE | Top 10 active markets preview |
| `GET /paid/market-brief` | $0.005 | Top 20 active markets with vol/spread/tags |
| `GET /paid/event-mispricing` | $0.02 | YES+NO sum < 0.98 markets |
| `GET /paid/whale-watch` | $0.03 | Volume spikes + leaderboard changes |
| `GET /paid/sentiment-snapshot` | $0.01 | News+reddit+RSS rollup per market |
| `GET /paid/structured-feed` | $0.01 | Compact JSON for agent subscribers |
| `GET /paid/momentum-signal` | $0.02 | Biggest 24h price moves × volume |
| `GET /paid/breaking-markets` | $0.01 | Resolving within 30 days |
| `GET /paid/depth-snapshot` | $0.02 | Top-of-book depth for liquidity providers |
| `GET /paid/historical-momentum` | $0.03 | Time-series price+volume for a market |
| `GET /paid/historical-mispricing` | $0.04 | arb_gap lifecycle across markets |
| `GET /paid/historical-screener` | $0.05 | Markets that moved > threshold% in N hours |
| `GET /paid/premium/strategy-report` | $0.50 | Curated 5-trade thesis bundle |
| `GET /paid/premium/portfolio-analysis` | $0.75 | Hedge recs + correlation matrix + risk score |
| `GET /paid/premium/alpha-feed` | $1.00 | Top 3 highest-conviction opportunities |
| `GET /paid/reseller/aggregated-brief` | $0.04 | Combines 5 products into 1 ranked JSON |
| `GET /paid/reseller/curated-feed` | $0.06 | Markets passing all 5 filters simultaneously |
| `GET /paid/reseller/tradingview-export` | $0.03 | Polymarket data for TradingView watchlist import |

## Quickstart (for AI agents)

```python
import httpx

# 1. Hit free preview first — see what the data looks like
r = httpx.get("https://identify-nut-ronald-proprietary.trycloudflare.com/free/market-brief")
markets = r.json()

# 2. When you want the full paid brief, pay via x402
# See x402 docs for client libraries: https://github.com/coinbase/x402
# Flow:
#   - GET /paid/market-brief → 402 with accepts[]
#   - Sign EIP-3009 transfer (Base) or submit Solana tx
#   - Retry with X-PAYMENT header → 200 + data
```

For Solana clients (no facilitator needed, on-chain verification):

```python
# 1. Build a USDC SPL TransferChecked tx to 7Cjuc27ccoBZjtaoH1xv7ePE8W5ni4bTU5Y4uK3P5a2F
#    USDC mint: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
# 2. Sign and submit to Solana mainnet
# 3. POST with X-PAYMENT: {"payload":{"tx_signature":"<sig>"},"network":"solana:mainnet"}
# 4. Server verifies tx on-chain → returns data
```

## Architecture (high-level)

```
                ┌─────────────────┐
   AI agent ──▶ │  FastAPI :8090  │
                │   x402 layer    │
                └────────┬────────┘
                         │
        ┌────────────────┼─────────────────┐
        ▼                ▼                 ▼
   ┌────────┐    ┌──────────────┐   ┌──────────────┐
   │Polymarket│   │ Solana RPC   │   │ EIP-3009     │
   │Gamma API │   │ (on-chain    │   │ verification │
   │(public)  │   │  verification)│  │ (Base main)  │
   └────────┘    └──────────────┘   └──────────────┘
                         │
                         ▼
              ┌──────────────────┐
              │ SQLite + WAL     │
              │ (receipts,       │
              │  revenue_log,    │
              │  product_cache)  │
              └──────────────────┘
```

**Key design choices:**
- **No facilitator dependency for Solana** — on-chain RPC is the source of truth (blockchain > facilitator say-so)
- **No API keys** — payment IS authentication
- **WAL-mode SQLite** — survives crashes without losing paid-call receipts
- **Free previews** — drives organic conversion (5 free endpoints, no signup)
- **Open stats** — `/v1/stats.json` shows real on-chain revenue separated from internal test traffic

## Pricing philosophy

Cheap enough that an AI agent doesn't have to think twice. Expensive enough that the data has to be useful. The middle tier ($0.01-0.05) covers per-call operational costs; the premium tier ($0.50-$1.00) covers multi-source analysis and human-quality synthesis.

**Margin:** a 1.5× markup over Polymarket API + scoring + packaging cost.

## What this is NOT

- Not a prediction-market betting platform
- Not an opinion/news source — it's structured data + scoring
- Not a wallet/custodian — payment goes straight to receiver address
- Not a wrapper around a single API — it's a multi-source signal refinery

## Self-test the system

```bash
# Health
curl https://identify-nut-ronald-proprietary.trycloudflare.com/health

# Free preview
curl https://identify-nut-ronald-proprietary.trycloudflare.com/free/market-brief

# Pricing
curl https://identify-nut-ronald-proprietary.trycloudflare.com/v1/pricing

# Real revenue (on-chain only)
curl https://identify-nut-ronald-proprietary.trycloudflare.com/v1/stats.json
```

## Use cases (real ones, not marketing)

1. **Momentum trading** — `/paid/momentum-signal` surfaces markets with biggest 24h moves × volume
2. **Arbitrage detection** — `/paid/event-mispricing` finds YES+NO sum < 0.98 markets
3. **Pre-resolution trading** — `/paid/breaking-markets` lists markets resolving within 30 days
4. **Sentiment-driven entries** — `/paid/sentiment-snapshot` aggregates news+reddit+RSS
5. **Liquidity provision** — `/paid/depth-snapshot` shows top-of-book for market makers
6. **Strategy backtesting** — `/paid/historical-*` endpoints for time-series analysis

## License

MIT. See LICENSE.

## Contact

Built by [CryptoTavern](https://cryptotavern.xyz). Service questions → that site.

---

*This README was generated by an autonomous system. The numbers, endpoints, and pricing are real and verifiable via `/health` and `/v1/stats.json`.*