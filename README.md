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

### How the consensus is built

Not every book deserves an equal vote. Sharp books make the number; retail books copy it and shade
toward public money. A flat median lets seven retail books outvote the two that actually know
something, dragging the "consensus" toward retail pricing — the very thing you're trying to beat.

So the consensus is a **weighted median of what each book's price implies**, in margin (or total) space
rather than probability space. Two consequences:

- **Every book contributes**, even on a fragmented board. Books aren't discarded for sitting off the
  modal line — their price is inverted through the discrete model to a common footing first.
- **Sharp books carry 4× weight** (tunable in Settings; 1 gives a flat median). At the default, two
  sharp books outweigh seven retail ones, so the consensus is effectively "what the market makers
  think," degrading gracefully to the wider market when no sharp book is present.

| Sharp book | Region key | Notes |
|---|---|---|
| LowVig | `us` | already in a default US fetch |
| BetOnline | `us` | already in a default US fetch |
| Pinnacle | `eu` | the sharpest signal; doubles credit cost |
| Novig / ProphetX | `us_ex` | exchanges |

Pinnacle is worth fetching as a *price signal* even if you can't bet there — use the book filter to
keep it out of recommendations while it still anchors the consensus.

### Prediction markets

Kalshi, Polymarket, Novig and ProphetX arrive through the same API under the `us_ex` region — no second
integration. They're structurally different from sportsbooks in ways that matter:

- **No vig.** You trade against other users, so de-vigging an exchange quote finds almost no hold —
  because there isn't any.
- **The cost is a separate trading fee.** Kalshi charges roughly `k × P × (1−P)` per contract with
  k = 0.07, peaking near 1.75¢ at even money — well inside a typical 4–5% sportsbook hold.
- **Binary settlement.** No pushes. Contracts are worded "by N or more", so mass landing exactly on the
  number is treated as a loss, which errs toward understating an edge rather than inventing one.
- **CFTC-regulated**, so they operate in states where sportsbooks can't.

**The fee model is not optional.** Left out, every exchange quote looks like a free 2–3% edge and swamps
the board — measured at 3.18% of phantom edge on a test quote. The coefficient is editable in Settings;
Polymarket and Novig default to zero.

### Cross-market gaps

Book-versus-exchange disagreements run wider and last longer than book-versus-book, because the crowds
are different — recreational money on one side, traders on the other. When the best price on each side
implies under 100% combined *after fees*, both sides can be backed for a locked profit.

Fees change these materially: a test gap prices at 1.84% with the fee model and 3.57% without, so a
marginal "2% arb" is really a loss. Liquidity is the binding constraint the API can't show you — a price
good for 20 contracts often isn't good for 500, and a partial fill on one side leaves a directional bet.

### Line movement and staleness

Fetches already cost credits, so the app keeps snapshots of where the consensus sat and reports how far
it has drifted since. A book still on an old number while the market has moved toward your side is a
much stronger signal than a static outlier — that's the difference between a stale line and a book that
simply disagrees. The board shows drift toward the bet side and how long ago each book last updated.

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

### Track record

Answers the one question the rest of the app can't: were the board's picks any good?

Every time you fetch a slate, the top Best Board picks are written to a log at the price and line
showing at that moment. Once the games finish, **Settle finished games** pulls the finals and grades
them. Everything is scored **flat 1 unit per pick** — this measures whether the picks were right, not
whether the sizing was, which is the Slate Planner's job.

Three rules keep the record from flattering itself:

- **A pick is written once.** Re-fetching the same slate bumps a "seen" counter instead of appending a
  second row, so you cannot pad a record by reloading. A pick's identity is game + market + side, not
  the line — a number drifting −3 → −3.5 is one opinion expressed twice, and counting it twice
  double-weights it.
- **Nothing is logged after kickoff.** Logging a game already under way would be choosing a bet with
  information the board never had.
- **A result you set by hand is never overwritten.** Correct a bad grade in the table and it sticks
  through every later settle, marked `hand`.

Weeks are anchored to Tuesday in **Eastern time**, so a Monday night game files with the Thursday it
belongs to rather than opening a week of its own — and it does that regardless of where the viewer's
own clock is set.

**Settling costs 2 credits per league** with an open pick, and only ever calls the API when something
is actually unsettled and already kicked off. A game still in progress reports `completed: false` and
is skipped, so a half-time score can never land on the board. The scores endpoint looks back three
days; anything older than that is only gradeable by hand, and the status line says so rather than
failing quietly.

**Read the numbers honestly.** At the 1–3% edge this board finds, hundreds of bets are needed before a
win rate separates from noise, so the summary leads with expected-versus-actual: the model gave each
pick a win probability at the time, and those sum to the wins it expected across exactly those bets.
That is a far better-powered read than a W–L record, and it still wants a full season. The record also
shows the hit rate against break-even at the prices actually taken, which is the number a win rate has
to clear to mean anything.

Exchange fees are already netted out of a winning payout, so a Kalshi win pays what it really pays.
**Export CSV** dumps the whole log — one row per pick, with the score, result and units — if you would
rather do your own analysis on it. Auto-logging can be switched off with the checkbox next to the
settle button.

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

**Settling the track record costs 2 credits per league**, and only when that league has a pick that
has kicked off and is still ungraded. One settle after each slate finishes is enough — roughly 4
credits a week for both leagues, or about 16 a month. It never calls the API with nothing to grade,
and the button quotes the exact cost before you press it.

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

