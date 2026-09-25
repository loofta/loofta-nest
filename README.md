# Loofta Nest

**Real stocks. Real events. Your call.**

Real tokenized US stocks on Solana, built around your interests and held. An AI watches real events on the companies you own, explains them from live X posts, and suggests trades and prediction-market plays you approve in two taps. Follow other nests and see their strategy.

Built for the Stocklana hackathon. This repository is the Nest feature extracted from the Loofta monorepo with its full commit history, so the design decisions are readable in the log as well as the code.

## What it does

1. **Build.** You pick themes (AI, semis, crypto-adjacent, consumer, healthcare, finance) and a risk persona. Nest builds a diversified basket of Backed xStocks, from $1, with no brokerage account. Positions are real SPL Token-2022 tokens in a Privy embedded wallet you control, routed through Jupiter.
2. **Hold.** The basket is built once, equal-weighted, then held. There is no silent rebalancing. The daily pass only ever deploys idle cash toward target; it never sells to correct drift.
3. **Explain.** When a company you own makes a real move, an AI reads the last three days of posts about it on X and writes you one paragraph with source links.
4. **Suggest.** That becomes one concrete idea, sized to your nest: "Sell $4.92 of MicroStrategy." Two taps to act, one to pass. Nothing executes without you.
5. **Call it.** Real prediction markets from Kalshi on the companies you hold sit next to your positions with live odds. Practice bets today; the venue layer is built so the same card becomes a real on-chain position through DFlow's tokenized Kalshi markets.
6. **Flock.** Follow other nests and see their strategy: theme mix, streak, level. Themes and streaks only, never balances or returns.

## How the AI works

Three jobs, kept deliberately separate.

| Job | Mechanism | AI? |
|---|---|---|
| Detect | A held stock moves 3%+ against its last stored close (`nest_price_history`) | No. Arithmetic on real prices. |
| Explain | Elfa `event-summary` clusters real X posts about the company from the last 3 days into one paragraph with source links | Yes. Comprehension, not prediction. |
| Size | Distance from the position's equal-weight target share of the nest | No. Deterministic. |

The AI never picks a stock, never sizes a position, never predicts a price.

## Why the AI doesn't pick

Because we tested it. The first allocator (`0d19bb3`) tilted weight toward companies with unusual social attention. `apps/backend/scripts/nest-backtest/` backtests that against 26 weeks of real Elfa data over 30 names, with a 200-run placebo test that shuffles the signal to check whether the result beats noise.

Full results are in [`RESULTS.md`](apps/backend/scripts/nest-backtest/RESULTS.md) and are served to the product by `GET /nest/backtest`. The short version: equal weight won, and the attention tilt sat at the 59th percentile of the placebo distribution. Elfa's own team confirmed mention volume on single equities is a reactive attention signal, not a return predictor. Commit `39f6f02` is the pivot. The product shows the rejected result on its own "why equal weight" toggle.

## Layout

```
apps/backend/src/modules/nest/
  nest-rebalance.service.ts     equal-weight allocator, position count scaled to capital, buy-only daily pass
  nest-suggestions.service.ts   event detection (price), explanation (Elfa), sizing, per-user caps and cooldowns
  elfa.service.ts               Elfa client: mention counts, event summaries, per-call budget metering
  kalshi.service.ts             Kalshi public API: company-matched event markets, one per series
  prediction-market.types.ts    venue-agnostic PredictionMarket + PredictionMarketProvider interface
  prediction-markets.service.ts aggregates providers; UI never learns which venue answered
  prediction-bets.service.ts    practice bets: real odds snapshotted at bet time, payout = stake / price
  prestocks.service.ts          pre-IPO practice basket on Kalshi IPO series
  xstocks.service.ts            Backed xStocks universe, Jupiter routing, price history
  nest-social.service.ts        crumbs (round-ups), flock (follow, kudos), levels, streaks
  nest.controller.ts            all routes
  backtest-summary.json         committed backtest output, served to the app

apps/backend/scripts/nest-backtest/   reproducible backtest + placebo, with cached inputs
apps/backend/supabase/migrations/     every Nest table, RLS enabled, PostgREST revoked

apps/frontend/src/components/nest/    dashboard, onboarding, suggestion + market cards, hero
apps/frontend/src/app/nest-earn/      /nest-earn (home) and /nest-earn/app (dashboard)
apps/frontend/src/services/api/nest.ts
nest-splash/, nest-onboarding/        the design kits the UI was built from
```

## Extracted from a monorepo

This is a faithful extract, not a standalone build. The code imports a handful of shared modules that live elsewhere in the Loofta monorepo and are intentionally not included here:

- `@/common/guards` — `AuthGuard` (Privy JWT verification) and the `@Public()` decorator
- `@/database/supabase.service` — a thin `SupabaseClient` wrapper (`getClient()`)
- `@/common/cron-gate` — `isCronEnabled(config)`: `NODE_ENV === 'production'` and not `DISABLE_CRONS`
- `@/common/solana-cluster-env` — mainnet RPC URL from config
- `@/hooks/useAuth`, `@/services/api/client` — Privy auth state and a `fetchApi` helper

Each is a few lines of glue. Everything that makes Nest what it is lives in this repository.

## Data and integrations

- **Backed xStocks** — tokenized US equities on Solana (Token-2022), swapped via Jupiter
- **Elfa** — X-native intelligence: `keyword-mentions` for attention, `event-summary` for explanations
- **Kalshi** — CFTC-regulated prediction markets, public read API
- **DFlow** — tokenized Kalshi markets on Solana; the `tradeable` flag on each market is what a DFlow provider would flip
- **Privy** — embedded Solana wallets
- **Supabase** — Postgres with RLS on every table

## Honesty notes

- Pilot runs on devnet USDC through a separate demo ledger identity (`nest-ledger-id.ts`); it is never mixed with real money.
- Practice bets are simulated and labelled as such. Nothing is placed on any exchange.
- Every claim in the product about performance points at the committed backtest, not at a forecast.
