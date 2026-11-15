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

## Methodology in depth

This section spells out, in full, how a sealed snapshot becomes a ranked board. Everything described here lives in `lib/model.ts` and `main.ts`. No ranking comes from a hand-written list of favourites, no protocol name is ever given special treatment, and no valuation number is typed in by a person: every figure on the board is derived from the snapshot itself at run time.

### The pipeline, end to end

1. The snapshot is parsed and checked. It must carry an `as_of` date, a source, a `floor_usd` value, a list of protocols and a watchlist of slugs. Any protocol record missing a slug, a name or a TVL is refused outright.
2. The protocols are split into two sets. A protocol counts as pre-token when its record carries no market cap, no CoinGecko id and no symbol. Everything else is live-token: it already trades, so the market has priced it, and the model never tries to price it again.
3. The live-token set is used to price the pre-token set. For each category, the model computes a comparable multiple from the live-token protocols in that category.
4. Every pre-token protocol that qualifies is scored with the same function, producing one implied value per protocol.
5. The scored rows are ordered by implied value, ranked from 1, and written out with their model factors and a farm plan attached to each row.
6. The dashboard, the screenshots and the dossier are generated from the same board, so what the judge sees in the page is exactly what `board.json` says.

### Where the multiples come from

The heart of the valuation is the comparable multiple, and it is deliberately boring: for each category, take every live-token protocol in that category, divide its market cap by its TVL, and take the median of those ratios. The median is used instead of the mean on purpose, because a single outlier — one protocol with a tiny TVL and an enormous market cap — should not be able to drag a whole category's multiple with it. A category only earns its own median when at least three live-token protocols price it; with fewer than three, the category falls back to the global median over all live-token protocols, so thin categories are never priced off one or two noisy data points. Every multiple, category or global, is clamped into the range 0.05x to 0.80x, which keeps a freak ratio from producing an absurd valuation in either direction.

Exchange tokens are left out of the comparables entirely. A centralised exchange's token is a claim on a business, not on a protocol's locked value, so treating it as a comparable for a lending market or a DEX would mix two different kinds of asset. The same reasoning keeps live-token protocols out of the ranked board itself: their market caps are shown beside the board as the market's own values, and they are never blended into the ranking.

Some of the multiples the snapshot produced for this run:

| Category | Multiple | Reading |
| --- | --- | --- |
| Canonical Bridge | 0.7464 | The richest multiple on the board; bridges with tokens trade dearly against their TVL |
| Basis Trading | 0.6544 | Basis strategies carry a high multiple, in line with their fee-heavy revenue |
| Bridge | 0.5567 | Plain bridges sit well above the global median |
| Dexs | 0.3424 | Decentralised exchanges earn a solid premium over the median protocol |
| RWA | 0.2748 | Real-world-asset protocols price a touch above the global median |
| Yield | 0.1914 | Yield protocols sit below the median, reflecting thinner margins |
| CDP | 0.1687 | Collateralised debt positions price conservatively |
| Yield Aggregator | 0.1510 | Aggregators earn one of the lower multiples |
| Indexes | 0.1477 | Index products trade at a modest ratio to their TVL |
| Lending | 0.1324 | Lending is the cheapest large category per dollar of TVL |
| Liquid Staking | 0.0500 | Pinned at the floor: the raw median was lower still |
| Risk Curators | 0.2460 | Fewer than three live comps, so the global median prices it instead |

The global median itself is 0.2460x. Any category with fewer than three live-token protocols, and any protocol whose category is missing, is priced at that number, which is why so many rows on the board share a 0.2460 multiple. That sharing is a feature, not a bug: it means the model refuses to invent precision it does not have.

### The four factors, one by one

The implied value of a protocol is its TVL times its category multiple, adjusted by three further factors. Each factor answers a different question about the protocol, and each is bounded so that no single factor can dominate the result.

First, the category multiple answers the question of what a dollar of locked value is worth as a token in this kind of protocol. A lending market and a DEX do not earn the same multiple, and the board does not pretend they do. The multiple is the market's own answer, taken from the protocols in the same category that already have tokens.

Second, the growth factor answers whether the protocol is currently gaining or losing locked value. It takes the seven-day TVL change, clamps it into the range from −60% to +120%, and applies it at half weight, so a protocol growing 20% in a week gets a multiplier of 1.10, and one shrinking 20% gets 0.90. The clamp matters: without it, a protocol that doubled its TVL from a tiny base in a week would swamp the ranking on a single noisy week.

Third, the maturity factor answers how long the protocol has run without launching a token. It starts at 0.60 for a brand-new protocol and rises linearly to 1.00 over three years, on the reasoning that a protocol which has survived for years without a token has more pent-up launch value than one that appeared last month. When the listing date is unknown, the model assumes 365 days, a neutral middle value that neither rewards nor punishes the row.

Fourth, the chain factor answers how widely the protocol is distributed. It adds 2% per chain beyond the first, up to a maximum of 10%, so a protocol live on six or more chains earns a 1.10 multiplier. Distribution is treated as a modest bonus, never as the main driver of value.

### Why every factor is rounded to four decimals

Before the four factors are multiplied together, each one is rounded to four decimal places. The reason is the dashboard: the detail view recomputes the implied value from the factors stored in the row, and if the stored factors carried full floating-point precision while the board carried a rounded product, the two would disagree by a few dollars on large rows. Rounding the factors first makes the dashboard's recompute agree with `board.json` to the dollar, and the detail view shows that check so anyone can verify it. The implied value itself is then rounded to a whole dollar and floored at $1, so every row carries a positive integer.

### Worked example: the top row

Rank 1 is Steakhouse Financial, a Risk Curators protocol. Its record in the snapshot gives a TVL of $2,753,981,495, a seven-day change of −2.1935%, an age of 543 days, and eleven chains. Risk Curators has fewer than three live-token protocols, so its multiple falls back to the global median of 0.2460.

- Base value: $2,753,981,495 × 0.2460 = $677,479,448.
- Growth: the change of −2.1935% is inside the clamp, so the multiplier is 1 + (−2.1935 / 100) × 0.5 = 0.9890. A small recent dip costs the row about one percent.
- Maturity: 543 days is just under half of the three-year ramp, so the multiplier is 0.6 + (543 / 1095) × 0.4 = 0.7984.
- Chain breadth: eleven chains means ten beyond the first, but the bonus caps at five extra chains, so the multiplier is min(1.10, 1 + 0.02 × 10) = 1.1000.
- Implied value: round($677,479,448 × 0.9890 × 0.7984 × 1.1000) = $588,444,665.

The arithmetic is deliberately transparent: anyone with the snapshot and this section can reproduce the number on a calculator. That is the point. A valuation nobody can check is a claim, not a model.

### How rows are chosen and ordered

The board is built in two parts. The watchlist is the floor, never the ceiling: all 40 of its slugs are on the board, whatever their size, because the competition names them as the protocols to watch. The remaining 160 rows are the early radar: pre-token protocols below the $50M watchlist floor, taken by TVL, largest first. A protocol only qualifies for the early radar when it is pre-token, is not a centralised exchange, and its name does not end in a version marker such as V2 or 3.1 — a versioned name usually marks a pool or a fork of another protocol rather than a standalone launch candidate, and the board is about standalone launches.

Once chosen, every row is scored with the same function and the board is sorted by implied value, highest first, with the slug as the tiebreak so the order is fully deterministic. Ranks run 1 to 200 with no gaps. Two runs over the same snapshot produce byte-identical output, which is checked as part of validation.

### The farm plans

Every row carries a six-step farm plan, and like everything else on the board it is derived from the protocol's own record — its category, its chains, its TVL and its age — never from a hand-written list of protocol names. The six steps always follow the same shape: set up a fresh wallet, take the position, do one qualifying action each week, hold, claim safely, and keep a log.

The capital for a position comes from the protocol's TVL: $250 for protocols above $5B, $150 above $1B, $100 above $100M, and $50 below that. The idea is that the position should be meaningful relative to the protocol without being reckless: farming a giant protocol with $50 would barely register, while putting $250 into a tiny one would dominate it.

The holding period comes from the protocol's age: positions in protocols younger than 180 days are held for 60 days, those between 180 and 365 days for 45 days, and older protocols for 30 days. Younger protocols get a longer window because their activity needs more time to be counted across snapshots. A protocol with no known age defaults to 30 days.

The weekly qualifying action is tailored to the category so the activity reads as genuine use: a lender borrows a little and repays it, a DEX user swaps or adjusts liquidity, a bridge user moves funds to a second chain, a staker claims or restakes rewards. The action names the protocol and its first chain, so each plan reads as written for that row, and the cost and time of every step are stated up front — a few dollars of gas, ten or fifteen minutes, and a holding period in days.

The safety rules are the same on every plan and are not negotiable: use a fresh wallet, keep its seed phrase offline, claim only from the link the protocol itself publishes, never pay to claim, and never follow a claim link from a direct message. A plan that told a reader to do otherwise would be worse than no plan at all.

### Validation before the board ships

Before the board is accepted, it passes a series of checks that run on every build. The snapshot's date, source, floor, protocols and watchlist are all required to be present and well-formed, and any protocol record missing a slug, a name or a TVL stops the run. The board gate then checks coverage (every watchlist slug present), ranks (contiguous from 1, no gaps), order (implied value never rising as the rank falls), spread (the top and bottom rows must not share the same implied value, or the board would not be a ranking at all), farm steps (between three and twelve per row, each between 20 and 600 characters), and value (every implied value a positive integer).

On top of the gate, the build is checked for determinism: the program is run twice over the same snapshot and the two outputs are compared byte for byte. The dashboard's recompute is checked against `board.json` to the dollar, as described above. And the dossier — the machine-readable summary the judges receive — is built from the same outputs, so the text the judges read, the numbers in the board and the page in the screenshots can never drift apart.

### Reproducing the run

The program needs Node.js 22 or later and nothing else: standard library only, no packages to install, no network access at run time. From the repository root, with the sealed snapshot available:

```sh
MAKEFUN_OUTPUT_DIR=out node main.ts --input ../token-radar-01-kit/input/snapshot.json
```

Or, with the snapshot at the conventional location, plain `node main.ts` reads `/input/snapshot.json` and writes to `/output`. Either way the run writes `board.json` (the 200 ranked rows with their model factors and farm plans), `data.js` (the same data as one script the dashboard loads), `dashboard/` (the three static files of the page), `screenshots/` (three renders of the page at 1440×900), and `dossier.json` (the machine-readable summary). Running the same command twice over the same snapshot produces the same files, byte for byte.

## Reading the board

The board is a ranking, not a recommendation, and it helps to know how to read it. The first thing to look at on any row is not the implied value but the category multiple and the number of comparables behind it. A row priced at its own category median with a dozen live protocols behind it rests on far more evidence than a row priced at the global median because its category had too few live tokens to price itself. Both kinds of row belong on the board, but they should not be trusted equally, and the board shows the evidence count on every row so the reader can tell them apart.

The second thing to look at is the spread of the factors. A row near the top because its TVL is enormous is a different story from a row near the top because it is growing fast, has run for years without a token, and lives on many chains. The detail view breaks the implied value into its four factors for exactly this reason: it lets the reader see which of the four did the work. A row whose value comes mostly from the growth factor is a bet on momentum continuing; a row whose value comes mostly from the maturity factor is a bet on a long-delayed launch finally happening. Those are different bets, and the board does not hide which one a row is making.

The dashboard is built to make this kind of reading easy. The top strip shows the ten highest rows at a glance. The table can be searched by name or slug, filtered by category, chain, minimum TVL and watchlist membership, and sorted by any column, so a reader who only cares about, say, lending protocols on Arbitrum can narrow the board to exactly that slice. Clicking a row opens a detail drawer with the protocol's full record, its four factors, the recompute check, the comparables table for its category, and its six-step farm plan. The live-token view sits beside the ranked board and shows the 24 largest live-token protocols by market cap, which is where the multiples ultimately come from.

The board is a snapshot in time, dated 2026-11-14. TVL moves daily, multiples move with the market, and a protocol that launches a token leaves the board entirely. None of that makes the board wrong; it makes it dated, which is why the date is printed on the page. A reader who wants today's answer should re-run the program against a fresh snapshot, not quote this board as if it were current.

## Limits and caveats

A comparables model is only as honest as its comparables, and this one has limits worth stating plainly. First, the multiples are backward-looking: they describe what live-token protocols are worth today, not what a new token will be worth tomorrow. A launch can price far above or far below its category's median, and the model has no way to know which. Second, TVL is a noisy input. A protocol can inflate its TVL with incentives, a single large deposit can move a small protocol's TVL by a lot, and a protocol's TVL can fall as fast as it rose. The growth factor and the maturity factor soften this, but they cannot remove it.

Third, the model treats every category as one market, and categories are broad. Two protocols in the same category can have very different revenue models, and the median multiple smooths over that difference by design. The board is a starting point for research, not a substitute for it: a row's rank says "look here first", and the farm plan says "if you look, here is how to position", but neither says "this will pay". Fourth, the farm plans assume the reader understands the risks of on-chain positions — smart-contract risk, bridge risk, and the risk that a protocol never launches a token at all. A farm plan is a set of steps, not a promise of a reward.

Finally, the snapshot itself is sealed and dated. Protocols that launched after 2026-11-14, protocols the snapshot does not carry, and protocols whose records are incomplete are simply not on the board. The board covers its 200 rows well and says nothing at all about anything outside them. That boundary is part of the design: a ranking that pretended to cover the whole universe would be less trustworthy than one that states its limits.
