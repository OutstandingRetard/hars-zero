# HARS-Zero Architecture

## System diagram

```
                    ┌─────────────────────────────────────────┐
                    │            EXTERNAL CLIENTS             │
                    │   (LLM agents, traders, analysts, bots)  │
                    └────────────────┬────────────────────────┘
                                     │
                                     │ HTTPS (HTTP 402)
                                     ▼
        ┌────────────────────────────────────────────────────────────┐
        │                  FastAPI :8090 (Cloudflare tunnel)         │
        │  ┌────────────┐  ┌────────────┐  ┌──────────────┐         │
        │  │ Discovery  │  │ Free tier  │  │ x402 paid    │         │
        │  │ llms.txt   │  │ /free/*    │  │ /paid/*      │         │
        │  │ tools.json │  │ /v1/*      │  │ middleware   │         │
        │  │ sitemap    │  │            │  │ EVM + Solana │         │
        │  └────────────┘  └────────────┘  └──────┬───────┘         │
        └─────────────────────────────────────────┼─────────────────┘
                                                  │
                ┌─────────────────────────────────┼──────────────────────┐
                │                                 │                      │
                ▼                                 ▼                      ▼
    ┌──────────────────────┐      ┌──────────────────────────┐   ┌──────────────┐
    │   PAYMENT VERIFY     │      │    DATA SOURCES          │   │  STORAGE     │
    │ ┌──────────────────┐ │      │  ┌──────────────────┐   │   │  SQLite WAL  │
    │ │ Base EVM         │ │      │  │ Polymarket       │   │   │              │
    │ │ EIP-3009 sig     │ │      │  │ Gamma (public)   │   │   │ receipts     │
    │ │ verification     │ │      │  └──────────────────┘   │   │ revenue_log  │
    │ └──────────────────┘ │      │  ┌──────────────────┐   │   │ product_cache│
    │ ┌──────────────────┐ │      │  │ Polymarket       │   │   │ nonce_reg    │
    │ │ Solana           │ │      │  │ CLOB (read-only) │   │   │ session_log  │
    │ │ on-chain RPC     │ │      │  └──────────────────┘   │   │              │
    │ │ (blockchain is   │ │      │  ┌──────────────────┐   │   └──────────────┘
    │ │ source of truth) │ │      │  │ News + Reddit +  │   │           │
    │ └──────────────────┘ │      │  │ RSS feeds        │   │           │
    └──────────────────────┘      │  └──────────────────┘   │           │
                                 └──────────────────────────┘           │
                                                                          │
            ┌─────────────────────────────────────────────────────────────┘
            │
            ▼
   ┌─────────────────────────┐
   │   BACKGROUND PROCESSES  │
   │  (cron jobs)            │
   │  - reconciler (5min)    │
   │  - policy (30min)       │
   │  - webhook_trigger (5m) │
   │  - revenue_alert (hour) │
   │  - daily_report (11am)  │
   │  - solana_monitor (5m)  │
   │  - solana_sweep (5am)   │
   │  - registry_ping (6h)   │
   │  - backup (4am)         │
   │  - audit (Sun 9am)      │
   └─────────────────────────┘
```

## Module map

```
src/
├── vending/          # Public HTTP API + x402 payment layer
│   ├── api.py        # FastAPI app, all routes
│   ├── x402_routes.py # x402 middleware (EVM + Solana)
│   ├── quotas.py     # Per-IP rate limits
│   ├── nonce_guard.py # Replay protection
│   ├── webhooks.py   # Webhook subscriptions
│   └── verify_eip3009.py # EIP-3009 signature verification
│
├── products/         # Product implementations
│   ├── premium.py    # $0.50-$1.00 endpoints
│   ├── reseller.py   # Bundled endpoints
│   └── historical.py # Time-series endpoints
│
├── scoring/          # Market scoring + packaging
│   ├── packaging.py  # market_brief, mispricing, etc.
│   └── ranking.py    # Cross-product ranking
│
├── collectors/       # External data ingestion
│   ├── polymarket_gamma.py
│   ├── polymarket_clob.py
│   ├── rss_news.py
│   └── scheduler.py  # Consolidated polling
│
├── refinery/         # Data normalization + feature extraction
│   ├── normalize.py
│   ├── dedupe.py
│   ├── feature_builder.py
│   └── topic_cluster.py
│
├── treasury/         # Financial accounting
│   ├── ledger.py
│   ├── reconciliation.py
│   ├── daily_report.py
│   ├── policy.py
│   └── solana_sweep.py # Bridge Solana → Base
│
├── health/           # System health monitoring
│   ├── revenue_alert.py
│   ├── payment_audit.py
│   ├── solana_monitor.py
│   └── discovery_guardian.py
│
├── evolution/        # Self-improvement loop
│   ├── metrics.py
│   └── experiment_runner.py
│
├── coordination/     # Cross-engine coordination
│   └── router.py
│
└── core/             # Foundation
    ├── db.py         # SQLite WAL helpers
    ├── logging.py
    ├── retry.py
    ├── rate_limit.py
    └── watchdog.py
```

## Key invariants

1. **Block-chain is source of truth for payments.** The CDP facilitator was bypassed for Solana because facilitator says-so != actually settled. On-chain RPC is the only ground truth.

2. **Stats separate REAL vs INTERNAL.** `/v1/stats.json` distinguishes settled on-chain revenue (tx_hash stored) from internal test traffic (tx_hash NULL). The two never mix.

3. **Free tier has no signup.** Anyone can hit `/free/*` and `/v1/*` without payment. This is the discovery surface for organic traffic.

4. **Discovery files are first-class routes.** `llms.txt`, `tools.json`, `sitemap.xml`, `robots.txt`, `x402-payment.json`, `health.json`, `stats.json` — all live, all machine-readable.

5. **No money in environment variables.** Wallet keys are on disk, chmod 600, in `keys/`. `~/.hermes/.env` for boot config only.

## What runs where

| Component | Where | Why |
|---|---|---|
| FastAPI API | systemd user service `hars-api` | Survives disconnect, auto-restart |
| Background jobs | cron (15 jobs) | Idempotent, recoverable, observable |
| Tunnel | `cloudflared` (trycloudflare) | Public HTTPS, no port forwarding |
| DB | SQLite WAL on disk | Fast, transactional, no external dep |
| Secrets | `keys/` chmod 600 + cold backup | Never in env vars |
| Cold backup | `~/hars-cold/` | Disaster recovery |

## Scaling story

Current: single-instance, single-tunnel, SQLite.
Next: read replicas for paid endpoints, dedicated `paid-tunnel` vs `free-tunnel`, Postgres if SQLite WAL locks under >100 RPS.

The x402 spec doesn't care about backend — clients just need the URL + correct 402 response. So horizontal scaling is just "more tunnels pointing to more replicas."