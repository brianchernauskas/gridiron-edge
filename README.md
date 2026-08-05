# Gridiron Edge

A bet calibrator and situational scorer for NFL and college football.

Single-file static HTML — no build step, no backend, no dependencies. Open `index.html` and it runs.

**Live:** https://brianchernauskas.github.io/gridiron-edge/

---

## What it does

This is a decision-support tool, not a pick generator. You supply a projection; it tells you whether any
price on the board beats that projection. Its most common correct answer is "pass."

### Calibrator

Give it one number — your projected home margin, or your projected total — and it prices every book's
offer against it:

- Strips the vig from each book's two-way price to recover the market's true implied probability
- Converts your projection into a win probability **at each book's own line**, so a board of
  −2.5 / −3 / −3.5 gets evaluated line by line rather than collapsed to one number
- Returns edge, EV per $100, a fractional-Kelly stake, and a verdict: Bet / Lean / Thin / Pass
- Flags key-number crossings and shows push probability explicitly
- A reality-check panel compares your number to the de-vigged market consensus, and warns you when
  you disagree by enough that the input is the more likely culprit than the edge

### Situational Scorer

Ten weighted factors — rest, travel, weather, injuries, motivation, line movement vs. tickets, pace,
talent gap, environment, coaching — each rated −2 to +2. Weights differ by league, because the two
markets are not the same problem:

| Factor | NFL | CFB |
|---|---|---|
| Injury edge | ×1.4 | ×1.6 |
| Motivation spot | ×0.7 | ×1.3 |
| Talent gap vs. line | ×0.4 | ×1.2 |
| Line move vs. tickets | ×1.3 | ×1.1 |

The output is a points lean with a contribution breakdown, which feeds back into the calibrator as an
adjustment to your projected margin. It is not a bet signal on its own.

---

## Setup

The app reads live lines from [the-odds-api.com](https://the-odds-api.com). You need your own key —
the free tier allows 500 requests/month.

1. Open the app → **Settings** tab
2. Paste your key, click **Save**
3. **Calibrator** → **Fetch live odds**

One fetch pulls moneyline, spreads, and totals for every game in the selected league in a single
request, and the game selector then works offline against that snapshot. Pull once per slate rather
than once per game. Switching leagues costs a second request.

There is also a manual line-entry path for when quota runs out or lines aren't posted yet.

### About the key

Your key is stored in `localStorage` under `ge_apiKey` and is sent only to `api.the-odds-api.com`.

**It is never committed to this repo, and it must not be.** Do not hardcode a key into `index.html` to
avoid re-entering it — that publishes a working key to a public URL. `localStorage` is per-origin, so
running the app from a local file and from the Pages URL each require entering the key once.

---

## Modeling notes

**Probabilities exclude pushes.** Everything displayed is conditional on the bet actually resolving, so
your win probability, the break-even rate, and de-vigged market prices are all directly comparable.
Showing a raw win probability next to break-even doesn't reconcile on whole-number spreads, where a
meaningful share of games land exactly on the line.

**Push probability comes from league-specific margin-frequency tables.** A margin of exactly 3 accounts
for roughly 14.8% of NFL results but only ~9.0% in college — which is why key numbers matter far more
in the NFL, and why the same half-point is worth different amounts in each league.

**Default standard deviations** are 13.5 (NFL margin), 10.5 (NFL total), 16.0 (CFB margin), 13.0
(CFB total). All are editable under Settings, along with the maximum situational lean. Raising the max
lean is how a discipline tool quietly becomes a rationalization tool — the defaults are deliberately
conservative.

---

## Disclaimer

For personal use. This tool measures whether a price beats your own projection; it does not predict
outcomes and offers no guarantee of profit. The vig is real and most bets are not worth making. Check
that sports wagering is legal where you are, and never stake money you need.
