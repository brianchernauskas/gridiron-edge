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

### Weekly Slate Planner

Build a card from the Calibrator or the Best Board, then size the whole thing at once.

Per-bet Kelly assumes the bet is your only exposure. It isn't — every additional bet competes for the
same bankroll, so summing individually-sized stakes systematically over-bets you. The planner simulates
the full slate and solves for the stake vector that maximises expected log growth across all bets
together, typically cutting total exposure 10–30% versus sizing one game at a time.

Bets on the same game are resolved from a single simulated game — one margin, one total — so a team's
spread and its moneyline correlate exactly instead of being treated as independent. Same-game pairs get
sized roughly 30% smaller than equivalent independent bets, because they can't diversify each other.

Also reports the chance the card finishes down, the 90% outcome range, worst case, and flags correlated
exposure (multiple bets on one game, a pile of overs, a card that's all favourites).

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
the free tier allows 500 credits/month.

1. Open the app → **Settings** tab
2. Paste your key, choose your markets, click **Save**
3. **Calibrator** → **Fetch live odds**

### Making the free tier last

The `/odds` endpoint bills **credits = markets × regions**, not one credit per request. Requesting all
three markets across US books costs 3 credits per league, so a both-league scan is 6. Two things keep
that in check:

**Fetch only the markets you price.** Spreads + totals is 2 credits per league — 4 for both, or about
125 full scans a month. Adding moneyline raises it to 6 a scan and drops you to 83.

**Slates are cached in the browser.** Reopening the app, reloading the page, and switching leagues all
cost nothing; only "Fetch live odds" spends credits. One fetch pulls every game in the league and the
selector then works entirely offline against that snapshot, so pull once per slate and work the whole
card off it.

The Settings tab shows the live credit cost of your current configuration and the age of each cached
slate. There's also a manual line-entry path for when quota runs out or lines aren't posted yet.

If you genuinely outgrow 500 credits, the paid tier is $30/month for 20,000. Running multiple free
accounts to extend the quota violates their terms and risks all of the keys.

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

**Spreads and moneylines are priced off a discrete margin distribution, not a smooth curve.** A normal
model badly undervalues buying off a key number — it smooths through the lump of games that land exactly
on 3. Instead, a normal envelope is re-weighted by empirical key-number multipliers derived from actual
margin frequencies, then renormalised. A margin of exactly 3 occurs about 2.6x more often than smooth in
the NFL, but only about 1.8x in college.

The result is a half-point value curve that matches reality:

| Buying off | NFL | CFB |
|---|---|---|
| 3 | 4.24% | 2.69% |
| 7 | 3.12% | 2.26% |
| 10 | 2.22% | 1.89% |
| 5 (not a key number) | 0.59% | 0.53% |

Push probability falls out of the same distribution, so it correctly depends on where the line sits
relative to your projection rather than being a flat table lookup. Totals stay on the normal model —
there's no equivalently reliable frequency table for exact totals, and their key numbers are much weaker.

**Default standard deviations** are 13.5 (NFL margin), 10.5 (NFL total), 16.0 (CFB margin), 13.0
(CFB total). All are editable under Settings, along with the maximum situational lean. Raising the max
lean is how a discipline tool quietly becomes a rationalization tool — the defaults are deliberately
conservative.

---

## Disclaimer

For personal use. This tool measures whether a price beats your own projection; it does not predict
outcomes and offers no guarantee of profit. The vig is real and most bets are not worth making. Check
that sports wagering is legal where you are, and never stake money you need.

## Look and feel

Shares the dark navy / gold identity of the DBFFL draft-order board, so the two sites read as one
set of tools. Dark is the default; the toggle still switches to light, which is the sensible theme
for printing a slate card.

The analytical tables are deliberately left dense. This is a tool for scanning numbers, and card
styling would have cost rows per screen for no analytical gain.

## Team logos

The Best Board and the Slate Planner card show team logos, matched from the Odds API's team names.

The map lives in `TEAM_ART` inside `index.html` rather than being fetched at runtime: ESPN's team
endpoint sends no `Access-Control-Allow-Origin` header, so a browser cannot call it, and baking the
map in keeps the app single-file and usable from `file://`. Logo images themselves are served from
`a.espncdn.com`, which does allow cross-origin requests.

Odds API team names match ESPN `displayName` exactly, which is why a plain lookup works. Anything
unmatched falls back to a chip with the team's initials, so a name we don't know degrades to
something readable rather than a broken image.

To regenerate after a conference shuffle:

```bash
curl -s "https://site.api.espn.com/apis/site/v2/sports/football/college-football/teams?limit=1000" -o cfb.json
curl -s "https://site.api.espn.com/apis/site/v2/sports/football/nfl/teams?limit=100" -o nfl.json
```

Then rebuild the two objects as `displayName -> [espnId, abbreviation]` for college and
`displayName -> abbreviation` for the NFL, skipping any team with no `logos` entry.

