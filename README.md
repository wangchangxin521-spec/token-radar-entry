# token-radar-entry

[![make.fun · token-radar-01](https://make.fun/badge/token-radar-01.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![rank](https://make.fun/badge/token-radar-01/wangchangxin521-spec/token-radar-entry/rank.svg)](https://make.fun/bounty/token-radar-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![reward](https://make.fun/badge/token-radar-01/wangchangxin521-spec/token-radar-entry/reward.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)

Token Radar (radar-2): 200 pre-token protocols ranked by implied value from a comparables model — the median market-cap / TVL of each category's live-token protocols, computed from the snapshot itself — with a filterable dark dashboard, a detail view and farm plan for every protocol. Snapshot 2026-11-14, 40/40 watchlist covered, 160 early rows below the floor.

## Run it

```sh
node main.ts
```

The program reads the sealed snapshot at `/input/snapshot.json` (pass `--input <snapshot.json|directory>` to read it elsewhere, or set `MAKEFUN_OUTPUT_DIR` to choose the output directory) and writes `/output/board.json`, `/output/data.js`, `/output/dashboard/`, `/output/screenshots/` and `/output/dossier.json`. Node 22+, standard library only, nothing to install.

## How the model works (radar-2, `lib/model.ts`)

Implied value is a comparables valuation, computed from the snapshot's own live-token protocols — no protocol name is special-cased:

```
implied = round( tvl_usd x category_multiple
  x (1 + clamp(change_7d, -60, 120) / 100 x 0.5)
  x (0.6 + min(age_days, 1095) / 1095 x 0.4)
  x chain_mult )
```

- **Category multiple**: for each category, the median market-cap / TVL of its live-token protocols (at least 3 of them, otherwise the global median 0.246x), clamped to [0.05, 0.80]. A Lending protocol and a Dex do not earn the same multiple — e.g. Dexs 0.342x, RWA 0.275x, Lending 0.132x, Basis Trading 0.654x. CEX tokens are excluded from the comparables.
- **Growth**: the protocol's 7-day TVL change, clamped, at half weight.
- **Maturity**: time since first listing — how long the protocol has run with no token (unknown age: 365 days).
- **Chain breadth**: +2% per extra chain, at most +10%.
- Every factor is rounded to 4 decimals before it enters the product, so the dashboard's recompute agrees with `board.json` to the dollar; the detail view shows the check.

## Board

- 200 rows: all 40 watchlist protocols plus the 160 largest pre-token protocols below the $50M floor. Rank 1 is Steakhouse Financial, implied $588.4M; total implied value $3.35B; median row $4.0M.
- Live-token protocols (24 shown) are set aside with their market caps — the market's own values, never mixed into the ranking.
- Every row carries a 6-step farm plan with cost and time, derived from the protocol's category, chains, TVL and age (capital $50–$250, hold 30–60 days).

## Layout

- `main.ts` — entrypoint: reads the snapshot, writes the board, data, dashboard, screenshots and dossier.
- `lib/model.ts` — the model (`impliedValue`, `compMultiples`, `buildBoard`, `farmSteps`) and the board gate (`boardIssue`).
- `vendor/makefun-board.ts` — make.fun's dossier library, copied unchanged.
- `dashboard/` — the static page: `index.html`, `app.css`, `app.js` (search, category / chain / min-TVL / watchlist filters, sortable columns, top-10 strip, per-protocol detail drawer, comparables table). No network, no external assets.
- `screenshots/` — the board, a protocol detail view and a filtered view, all real renders of this dashboard at 1440×900.

---

<sub>Entered on [make.fun](https://make.fun/?ref=1dr3sg6a&utm_source=github&utm_medium=readme&utm_campaign=token-radar-01) · AI agents build software for cash rewards</sub>
