# DopeWars — Functional Specification

[Written by Claude, prompted and cleaned up by Beej. All mistakes are
Claude's 😊]

**Version:** 1.0 (canonical ruleset + optional variants)

**Date:** 2026-11-20

**Baseline:** *Drug Wars* (John E. Dell, 1984), as commonly cloned

**Audience:** Implementers (students, clone authors) and game designers

**Format of requirements:** Numbered, testable statements using RFC 2119
keywords (**MUST**, **SHOULD**, **MAY**)

---

## 0. About This Document

### 0.1 Provenance and honesty note

DopeWars exists in dozens of ports (Apple II, DOS, TI-83, Palm, Unix
`dopewars`, mobile). Their rules and numbers differ in details, and the
original source is not a normative reference. This spec therefore:

- Keeps the **structure and feel** of the 1984 original: 30 days, a
  loan-shark debt, six New York locations, price arbitrage, random
  encounters.
- Treats every number as a **default design parameter** (named constants
  in §3). Values are reconstructed from common ports and tuned using the
  reference simulation in §14. They are **not** byte-accurate to any
  single original.
- Marks every tunable constant so an implementation can change it
  without violating the spec.

### 0.2 How to read this document

- **Part I (§1–§13)** is the *canonical game*. A conforming
  implementation MUST satisfy every MUST in Part I.
- **§14** is balance rationale and tuning guidance. It is informative,
  not normative.
- **§15** is the acceptance test suite.
- **Appendices A–L** are *optional variants*. Each is self-contained,
  lists the sections of Part I it modifies, and is off by default. An
  implementation claims conformance to the canonical game unless it
  explicitly enables a variant.
- Requirement IDs: `SET` setup, `FLOW` game flow, `LOC` locations, `MKT`
  market, `TRD` trading, `EVT` events, `CMB` combat, `FIN` finance,
  `END` ending, `UI` interface, `PER` persistence, `RNG` randomness,
  `NFR` non-functional.

### 0.3 Content note

The theme is illicit drug trading. Some courses or platforms will prefer
a neutral skin. Appendix K defines a content-neutral reskin that changes
names only and leaves every rule untouched.

---

# PART I — CANONICAL GAME

## 1. Overview

### 1.1 Premise

The player is a small-time street dealer in New York City who owes a
loan shark money. The player has **30 days** to trade drugs between
boroughs, buying where prices are low and selling where they are high,
while repaying the debt, dodging police and muggers, and staying alive.
At the end of day 30 the player's net worth is scored and may enter the
high-score table.

### 1.2 Design pillars

1. **Arbitrage under uncertainty.** Prices are re-rolled every day in
   every location. The player's skill is reading price positions within
   each drug's range and sizing risk.
2. **Compound pressure.** Debt grows exponentially. Every day of delay
   has a measurable cost.
3. **Capital vs. capacity.** Early on, cash limits purchases; later,
   carrying capacity does. The optimal strategy changes mid-game.
4. **Short sessions.** A full game takes roughly 5–15 minutes and is
   replayable.
5. **Simple rules, deep tension.** All rules fit in this document; no
   hidden systems.

### 1.3 Scope

In scope: single-player, turn-based, one human player, local persistence
of high scores. Out of scope for canonical play (see appendices):
multiplayer, hospitals, hired help, dynamic markets, networking.

---

## 2. Glossary

| Term | Meaning |
|---|---|
| **Day** | One turn. Days run 1 to `GAME_DAYS`. |
| **Location** | One of six boroughs/areas the player can be in. |
| **Drug** | A tradable commodity with a fixed global price range. |
| **Market** | The set of drug prices currently offered at the player's location. |
| **In stock** | A drug has a price in the current market. Only in-stock drugs can be bought or sold. |
| **Capacity** | Total inventory slots ("pockets"). |
| **Unit** | One item of a drug. Occupies one slot. |
| **Jet** | The travel action. Advances the day. |
| **Net worth / Score** | `cash + bank − debt` at game end (see §11). |
| **Hostile encounter** | A police or mugging event on arrival. |
| **Windfall** | A beneficial random event on arrival (found drugs, coat offer). |

---

## 3. Constants and Default Parameters

All values are **tunable defaults** (requirement SET-1). They MUST be
defined in one configuration location (a constants file or table), not
scattered through code.

### 3.1 Player and game constants

| Constant | Default | Notes |
|---|---:|---|
| `GAME_DAYS` | 30 | Total days |
| `START_CASH` | $2,000 | |
| `START_DEBT` | $5,500 | Owed to the loan shark |
| `START_BANK` | $0 | |
| `START_HEALTH` / `MAX_HEALTH` | 100 | |
| `START_CAPACITY` | 100 | Slots |
| `START_GUNS` | 0 | |
| `START_LOCATION` | Bronx | |
| `HEALTH_REGEN` | 5 per day | Applied on day advance, capped at `MAX_HEALTH` |

### 3.2 Finance constants

| Constant | Default | Notes |
|---|---:|---|
| `DEBT_RATE` | 10% per day | Compounding; rounded **up** |
| `BANK_RATE` | 5% per day | Compounding; rounded **down** |
| `MAX_DEBT` | $20,000 | Cap on total outstanding debt after any borrowing |

### 3.3 Equipment constants

| Constant | Default |
|---|---:|
| `GUN_PRICE` | $500 |
| `MAX_GUNS` | 3 |
| `GUN_SPACE` | 5 slots each |
| `GUN_HIT_CHANCE` | 50% |
| `GUN_DAMAGE` | 5–10 (inclusive, uniform) |
| `COAT_PRICE` | $100–$250 (uniform, per offer) |
| `COAT_BONUS` | +40 slots |

### 3.4 Event constants

| Constant | Default | Meaning |
|---|---:|---|
| `P_STOCK` | 80% | Chance each drug is in stock at a location on a given day |
| `MIN_STOCK` | 4 | Minimum number of in-stock drugs per market |
| `P_CRASH` | 8% | Market event: one drug's price ÷ 4 |
| `P_SPIKE` | 8% | Market event: one drug's price × 4 |
| `P_COP_BASE` | 5% | Police encounter chance, base |
| `P_COP_LOAD` | 10% | Added police chance scaled by load factor (carried units ÷ capacity, clamped to 1) |
| `P_MUG` | 6% | Mugging chance, if no police encounter |
| `MUG_LOSS` | 10–30% of cash on hand | Uniform |
| `P_FIND` | 5% | Found-drugs windfall |
| `FIND_QTY` | 3–8 units | Uniform, clamped to free capacity |
| `P_COAT` | 4% | Coat offer windfall |
| `COP_COUNT_WEIGHTS` | 1 cop: 50%, 2: 30%, 3: 20% | |
| `COP_HP` | 15 + current day | Per cop |
| `COP_HIT_CHANCE` | 50% | |
| `COP_DAMAGE` | 5–10 | Inclusive, uniform |
| `RUN_SUCCESS` | 50% | Per attempt |
| `COP_LOOT` | $50–$150 per cop killed | Uniform |

### 3.5 Drug table

| # | Drug | Min price | Max price |
|---:|---|---:|---:|
| 1 | Cocaine | $15,000 | $29,000 |
| 2 | Heroin | $5,000 | $14,000 |
| 3 | Acid | $1,000 | $4,400 |
| 4 | PCP | $1,000 | $3,500 |
| 5 | Hashish | $450 | $1,500 |
| 6 | Weed | $300 | $900 |
| 7 | Speed | $70 | $250 |
| 8 | Ludes | $10 | $60 |

The order above is the **canonical drug order**, used for display and
for RNG consumption (§12).

### 3.6 Location table

| # | Location | Services |
|---:|---|---|
| 1 | Bronx | Market, **Loan Shark**, **Bank**, **Gun Shop** |
| 2 | Ghetto | Market |
| 3 | Central Park | Market |
| 4 | Manhattan | Market |
| 5 | Coney Island | Market |
| 6 | Brooklyn | Market |

---

## 4. Game State

**SET-2.** The implementation MUST maintain at least the following
state.

```
Player:
  cash        : integer ≥ 0     (dollars)
  bank        : integer ≥ 0
  debt        : integer ≥ 0
  health      : integer 0..MAX_HEALTH
  capacity    : integer         (slots, grows only via coat)
  guns        : integer 0..MAX_GUNS
  inventory   : map<Drug, integer ≥ 0>
World:
  day         : integer 1..GAME_DAYS
  location    : Location
  market      : map<Drug, integer price> (drugs absent = out of stock)
  rng         : seedable random source (§12)
Derived:
  usedSpace   = sum(inventory) + guns × GUN_SPACE
  freeSpace   = capacity − usedSpace            (MUST NOT go below 0)
  netWorth    = cash + bank − debt
```

**SET-3.** All money values MUST be integers (whole dollars). No
floating-point currency.

**SET-4.** New game start state: `cash=START_CASH`, `debt=START_DEBT`,
`bank=START_BANK`, `health=START_HEALTH`, `capacity=START_CAPACITY`,
`guns=START_GUNS`, empty inventory, `day=1`, `location=START_LOCATION`.

**SET-5.** On a new game the Day-1 market MUST be generated using §6
with **market events disabled** and with **no arrival encounters** (a
safe first day).

---

## 5. Game Flow

### 5.1 State machine

```
TITLE ──► NEW_GAME ──► DAY_ACTIVE ◄─────────────────────────────┐
                          │  (trade / bank / loan / gun shop)   │
                          │                                     │
                          ├─ Jet to a different location ──► TRAVEL_RESOLUTION
                          │                                     │
                          │                              ENCOUNTER (cops) ─► DAY_ACTIVE
                          │                                     │            or GAME_OVER (death)
                          │                                     │
                          └─ Finish (day = GAME_DAYS) ──► GAME_OVER ──► HIGH_SCORES ──► TITLE
```

### 5.2 Requirements

**FLOW-1.** During DAY_ACTIVE the player MAY perform any number of free
actions before traveling: buy, sell, drop, and (in the Bronx) borrow,
repay, deposit, withdraw, buy guns. Free actions MUST NOT advance the
day.

**FLOW-2.** The player MUST NOT be able to travel to the current
location. Travel MUST be to one of the other five locations.

**FLOW-3.** Travel on days 1 to `GAME_DAYS − 1` MUST execute **Arrival
Resolution** (§5.3), then return to DAY_ACTIVE.

**FLOW-4.** On day `GAME_DAYS` the Jet action MUST be replaced by
**Finish Game**. Finish Game requires confirmation and then transitions
to GAME_OVER with no arrival resolution.

**FLOW-5.** If health reaches 0 at any time, the game ends immediately
with cause "killed" (§11.3).

### 5.3 Arrival Resolution (normative order)

```
travel(dest):
  assert dest != location and day < GAME_DAYS
  1. day      += 1
  2. location  = dest
  3. debt      = ceil(debt × (1 + DEBT_RATE))          // if debt > 0
  4. bank      = floor(bank × (1 + BANK_RATE))
  5. health    = min(MAX_HEALTH, health + HEALTH_REGEN)
  6. generate_market()                                  // §6, with market events enabled
  7. resolve_hostile_encounter()                        // §8: cops, else mugging
  8. if alive: resolve_windfalls()                      // §9: found drugs, coat offer
  9. return to DAY_ACTIVE (show market and event log)
```

**FLOW-6.** Steps MUST execute in exactly this order. Interest is
charged *on arrival*; therefore a full game incurs `GAME_DAYS − 1`
interest periods (29 by default).

---

## 6. Locations and Markets

### 6.1 Market generation

**MKT-1.** `generate_market()` MUST produce a fresh market on every
arrival, independent of the previous visit to that location (no memory
in canonical play; see Appendix G for dynamic markets).

**MKT-2.** Stock: for each drug in canonical order, the drug is in stock
with probability `P_STOCK`. If fewer than `MIN_STOCK` drugs are in
stock, the implementation MUST repeatedly pick a uniformly random
out-of-stock drug and mark it in stock until `MIN_STOCK` is reached.

**MKT-3.** Price: for each in-stock drug in canonical order, price =
uniform random integer in `[min, max]` inclusive (§3.5).

**MKT-4.** Market event: with probability `P_CRASH + P_SPIKE`, one
market event occurs (crash with relative weight `P_CRASH`, spike with
`P_SPIKE`). The affected drug is chosen uniformly from **all** eight
drugs. If it was not in stock, it MUST be put in stock with a normally
generated price first, then modified.

- **Crash:** `price = max(1, floor(price / 4))`
- **Spike:** `price = price × 4`

**MKT-5.** Market events MUST be disabled for the Day-1 market (SET-5).

**MKT-6.** Event announcements MUST be shown to the player, e.g.:

- Crash: *"The market has been flooded with cheap {drug}!"*
- Spike: *"Cops made a big {drug} bust! Prices are outrageous!"* or
  *"Addicts are buying {drug} at outrageous prices!"* (implementations
  MAY pick either spike text).

**MKT-7.** Within a single day at a single location, prices MUST be
constant. Buying and selling do **not** change prices (canonical rule).

---

## 7. Trading

### 7.1 Buying

**TRD-1.** The player MAY buy a drug only if it is in stock.

**TRD-2.** Maximum purchasable quantity:
```
maxBuy = min( floor(cash / price), freeSpace )
```

**TRD-3.** The buy dialog MUST accept an integer quantity `1 … maxBuy`,
and SHOULD provide a MAX shortcut. Quantities outside this range MUST be
rejected with a message and no state change. If `maxBuy = 0`, the dialog
MUST explain why (not enough cash or no space).

**TRD-4.** A successful buy: `cash −= qty × price`, `inventory[drug] +=
qty`.

### 7.2 Selling

**TRD-5.** The player MAY sell a drug only if it is **in stock** at the
current location and the player holds ≥1 unit.

**TRD-6.** The sell dialog MUST accept `1 … inventory[drug]`, with an
ALL shortcut. A successful sell: `cash += qty × price`, `inventory[drug]
−= qty`.

**TRD-7.** The sale price equals the displayed price (no spread). The
canonical game has no buy/sell spread.

### 7.3 Dropping

**TRD-8.** The player MAY drop (discard) any quantity of any held drug
at no cost, at any time during DAY_ACTIVE. Dropped units are destroyed.
Drop requires confirmation.

### 7.4 Capacity

**TRD-9.** `usedSpace` MUST NEVER exceed `capacity`. All operations that
add items (buy, find drugs, buy gun) MUST check `freeSpace`.

---

## 8. Hostile Encounters

Resolved at step 7 of arrival. At most **one** hostile encounter per
arrival.

### 8.1 Selection

```
load      = min(1, sum(inventory) / capacity)
pCop      = P_COP_BASE + P_COP_LOAD × load
if rand() < pCop:           → police encounter (§8.2)
else if rand() < P_MUG:     → mugging (§8.4)
else: none
```

### 8.2 Police encounter

**EVT-1.** Number of cops is drawn per `COP_COUNT_WEIGHTS`. The first
cop is named **Officer Hardass**; any others are **Deputies**. Each has
`COP_HP` hit points.

**EVT-2.** The encounter is a round-based dialogue. Each round the
player chooses:

- **Run** — always available.
- **Fight** — available only if `guns ≥ 1`.

**EVT-3. Run.** With probability `RUN_SUCCESS` the player escapes; the
encounter ends with no penalty. On failure, every living cop takes a
shot (EVT-5) and a new round begins.

**EVT-4. Fight.** Each owned gun fires once: hit with probability
`GUN_HIT_CHANCE`; on hit, deal `GUN_DAMAGE` to the **first living cop**
in order (damage does not overflow to the next cop). A cop at 0 HP or
less is killed. If all cops are dead, the player wins (§8.5). Otherwise
every living cop takes a shot.

**EVT-5. Cop attack.** Each living cop hits with probability
`COP_HIT_CHANCE`; on hit, deals `COP_DAMAGE` to the player. If `health ≤
0` the player dies and the game ends (§11.3).

**EVT-6.** The UI MUST show after each round: cops remaining with their
HP (or a health descriptor), player health, and a log of what happened.


### 8.3 Cop attack ordering

Per-round resolution order MUST be: player action → cop attacks (if the
encounter continues). Player death is checked after each individual cop
shot.

### 8.4 Mugging

**EVT-7.** If `cash > 0`, the player loses `floor(cash × u)` where `u`
is uniform in `[0.10, 0.30]`. Message: *"You were mugged in the subway
and lost ${n}!"* If `cash = 0`, the mugging does nothing and MUST NOT be
announced. Banked money is never lost.

### 8.5 Winning a fight

**EVT-8.** On killing all cops, the player receives `COP_LOOT` per cop
killed. The encounter ends.

---

## 9. Windfalls

Resolved at step 8, only if the player is alive. Each is rolled independently.

**EVT-9. Found drugs.** With probability `P_FIND`: choose a uniformly
random drug; quantity = uniform `FIND_QTY`, clamped to `freeSpace`. If
`freeSpace = 0` nothing happens and nothing is announced. Message: *"You
find {n} units of {drug} on a dead dude in the subway."*

**EVT-10. Coat offer.** With probability `P_COAT`: offer a coat for a
price drawn from `COAT_PRICE`. Accepting costs that price and adds
`COAT_BONUS` to `capacity`. The player MAY decline. If `cash < price`
the accept option MUST be disabled with an explanation.

---

## 10. Finance and Services (Bronx only)

**LOC-1.** The Loan Shark, Bank, and Gun Shop are available **only in
the Bronx**. The UI MUST hide or disable them elsewhere.

### 10.1 Loan Shark

**FIN-1.** *Repay:* the player may pay any integer `1 … min(cash,
debt)`. A "pay all" shortcut SHOULD pay `min(cash, debt)`.

**FIN-2.** *Borrow:* the player may borrow any integer `1 … (MAX_DEBT −
debt)`. Borrowed money is added to both `cash` and `debt`. If `debt ≥
MAX_DEBT` the shark refuses.

**FIN-3.** Debt interest is applied only at arrival (§5.3 step 3): `debt
= ceil(debt × 1.10)`. There is no interest on the day the loan was taken
until the next arrival.

### 10.2 Bank

**FIN-4.** *Deposit:* `1 … cash`. *Withdraw:* `1 … bank`. No fees, no
limits, instant.

**FIN-5.** Bank interest is applied at arrival: `bank = floor(bank ×
1.05)`.

**FIN-6.** Banked money is safe from mugging and from the death
forfeiture rule (§11.3).

### 10.3 Gun Shop

**FIN-7.** Buying a gun costs `GUN_PRICE`, requires `guns < MAX_GUNS`,
`cash ≥ GUN_PRICE`, and `freeSpace ≥ GUN_SPACE`. On success `guns += 1`,
`cash −= GUN_PRICE`.

**FIN-8.** Guns cannot be sold back, dropped, or lost in canonical play.

---

## 11. Ending and Scoring

### 11.1 Normal ending

**END-1.** After the player confirms Finish Game on day `GAME_DAYS`, the
game ends. Unsold drugs are **worth $0** (they are not auto-sold); the
player is expected to sell before finishing. The UI MUST warn if
inventory is non-empty when the player attempts to finish. (See Appendix
D for auto-liquidation.)

### 11.2 Score

**END-2.** `score = cash + bank − debt`. The score MAY be negative.

### 11.3 Death

**END-3.** If the player is killed, the game ends immediately. The score
is computed as `bank − debt` (cash on hand and drugs are forfeited). The
result MUST be recorded as "killed" in the high-score table.

### 11.4 Rank titles (informative, tunable)

| Score | Title |
|---|---|
| < $0 | In the Hole |
| $0 – $49,999 | Street Rat |
| $50,000 – $249,999 | Small-Time Dealer |
| $250,000 – $999,999 | Kingpin in the Making |
| ≥ $1,000,000 | Kingpin |

### 11.5 High scores

**END-4.** The game MUST maintain a local top-10 table: player name (≤
16 characters), score, date, and outcome (finished/killed). A new score
qualifies if the table has fewer than 10 entries or the score exceeds
the lowest entry. Ties keep the earlier entry ranked higher.

**END-5.** The table MUST persist across sessions (PER-1).

---

## 12. Randomness

**RNG-1.** All randomness MUST come from a single seedable generator
exposed through two functions:

- `rand()` → float in `[0, 1)`
- `randInt(lo, hi)` → integer in `[lo, hi]` inclusive, uniform

**RNG-2.** A new game MUST accept an optional seed. With the same seed
and the same sequence of player inputs, the game MUST be fully
deterministic (identical prices, events, outcomes). Without a seed, one
MUST be generated and SHOULD be displayed on the game-over screen.

**RNG-3. Consumption order.** To make seeded replays portable across
implementations, RNG draws within arrival resolution MUST occur in this
order:

1. Stock rolls: one `rand()` per drug in canonical order; then fill-in
   picks if needed.
2. Prices: one `randInt` per in-stock drug in canonical order.
3. Market event: one `rand()` roll (event/no-event and crash-vs-spike);
   if an event occurs, one drug pick, plus a price roll if the drug was
   out of stock.
4. Hostile: `rand()` for police; if police, the cop-count draw, then
   combat rolls in the order they occur (player gun rolls in gun order,
   then cop shots in cop order); otherwise `rand()` for mugging, then
   `rand()` for the loss fraction.
5. Windfalls: found-drugs roll (+ drug pick + quantity), then coat roll
   (+ price).

> Exact bit-for-bit cross-implementation equality also requires a
> specified PRNG algorithm. Implementations that want cross-language
> replays SHOULD use a documented algorithm such as `mulberry32` or
> `xoshiro128**` and record it in the build info.

---

## 13. User Interface Requirements

The spec is platform-agnostic (terminal, desktop, web, mobile). Required
information and behaviors:

### 13.1 Main (DAY_ACTIVE) screen

**UI-1.** MUST display at all times: day (`Day d of 30`), location,
cash, bank (if in Bronx or > 0), debt, health, guns, free space /
capacity.

**UI-2.** MUST display the market as a list of all eight drugs in
canonical order with current price (or "—" if out of stock) and the
quantity held.

**UI-3.** MUST offer actions: Buy, Sell, Drop, Jet (or Finish on the
last day), and in the Bronx: Loan Shark, Bank, Gun Shop.

**UI-4.** SHOULD display, for each drug, an indicator of where today's
price falls in that drug's range (e.g., low / mid / high or a bar),
since this is the primary decision aid.

**UI-5.** MAY display average purchase price per held drug (informative;
not part of game state in canonical play).

### 13.2 Dialogs and feedback

**UI-6.** Every state-changing action MUST give visible feedback
(message log entry). Invalid input MUST never crash or silently clamp;
it MUST show an error.

**UI-7.** The event log for each arrival MUST list, in order: market
events, encounters and their outcomes, windfalls.

**UI-8.** Destructive actions (Drop, Finish Game, abandon game) MUST
require confirmation.

**UI-9.** The Jet screen MUST list the five other locations. It SHOULD
show the number of days remaining and current debt.

### 13.3 End screens

**UI-10.** The game-over screen MUST show: cause (finished/killed),
final cash, bank, debt, score, rank title, and the seed.

**UI-11.** If the score qualifies for the high-score table, the player
MUST be prompted for a name (empty input defaults to a placeholder).

### 13.4 Accessibility and input

**UI-12.** The game MUST be fully playable by keyboard alone (or by
touch alone on mobile). Information MUST NOT be conveyed by color alone.

**UI-13.** Quantity prompts MUST accept only non-negative integers, MUST
trim whitespace, and MUST cope with thousands separators if input is
free-form text.

---

## 13.5 Persistence and Non-Functional Requirements

**PER-1.** The high-score table MUST persist across sessions in local
storage appropriate to the platform.

**PER-2.** Mid-game save/resume MAY be supported. If supported, a save
MUST contain the full state of §4 including the RNG state, and resuming
MUST not change outcomes.

**PER-3.** Corrupt or missing persistence data MUST NOT crash the game;
it MUST fall back to an empty high-score table.

**NFR-1.** All game logic MUST be separable from presentation (a pure
"engine" module testable without any UI). This is the basis for the
acceptance tests in §15.

**NFR-2.** Every action and arrival MUST complete in under 100 ms on
commodity hardware.

**NFR-3.** The engine MUST be free of global mutable state apart from
the explicit game state object and RNG.

**NFR-4.** The engine SHOULD expose an action log (list of player
commands) sufficient to replay a game from its seed.

---

# 14. Balance Rationale and Tuning Guide (Informative)

### 14.1 The core economic loop

- **Price ranges are wide** (e.g., Cocaine $15k–$29k, a ~1.9× spread).
  Because prices are independent per location per day, the expected
  profit from buying at the cheap end and selling at the expensive end
  of a range is large, and finding the right city is luck plus judgment.
- **Cheap drugs carry the early game.** With $2,000, a player cannot
  afford a single unit of Cocaine, so early capital growth happens in
  Weed, Speed, Hashish and Ludes. The price ladder across six orders of
  magnitude (Ludes $10 to Cocaine $29k) makes capital growth a *phase
  structure*: cheap goods first, then Acid/PCP, then Heroin/Cocaine.
- **Market events are the jackpots.** A 4× crash is a one-day
  opportunity worth far more than ordinary spread, and a 4× spike is the
  matching sell opportunity. At 16% combined per arrival, the player
  sees roughly one event every six days: enough to feel like a recurring
  decision, rare enough to feel lucky.

### 14.2 Debt and compounding

- Debt of $5,500 at 10%/day for 29 periods grows to **$87,333** (5,500 ×
  1.1^29 ≈ 87,247 before ceiling rounding). Doing nothing therefore
  guarantees a score near **−$85,900**.
- Bank interest of 5% over the same period multiplies by **4.12×**; debt
  multiplies by **15.86×**.
- The tuning insight, **verified by simulation (§14.3)**, is that
  repaying debt immediately is *wrong*. Early in the game, trading
  returns on capital exceed 10% per day, so cash is better used buying
  drugs than retiring debt. Repay once capital is saturated by capacity.
  This is the central "aha" of the game and the reason for `MAX_DEBT`:
  unbounded borrowing would let a skilled player leverage the same logic
  indefinitely.
- Because `DEBT_RATE` is higher than `BANK_RATE`, the player always
  prefers retiring debt over banking once capital is saturated.

### 14.3 Reference simulation

A bot played the canonical rules (3,000 games per row, same price/event
model as this spec; seeds 0–2999). The bot never borrows, never buys
guns or coats, never banks, and always runs from cops. Results are an
*indication of relative difficulty*, not a claim about optimal play.

| Policy / setting | Median score | 10th pct | 90th pct | Finished with score > 0 | Killed |
|---|---:|---:|---:|---:|---:|
| Do nothing (never trade) | −$85,877 | −$86,423 | −$85,333 | 0% | 0.2% |
| Repay debt as soon as cash allows | −$15,892 | −$55,603 | $1,016,682 | 45% | 0.3% |
| Repay debt only when cash ≥ $10k | $519,287 | $14,190 | $1,578,117 | 94% | 0.6% |
| Repay debt only when cash ≥ $30k | $601,641 | $117,494 | $1,695,004 | 98% | 0.6% |
| Same, `DEBT_RATE` = 5% | $636,465 | $162,731 | $1,743,036 | 99% | 0.6% |

Interpretation:

- "Repay immediately" fails more than half the time, which demonstrates
  the capital-vs-debt tension is real, not cosmetic.
- A competent but simple capital-efficient strategy wins about 98% of
  the time. That is generous; the difficulty presets in Appendix E
  tighten it. The 90th percentile exceeding $1.5M also means the rank
  "Kingpin" (≥ $1M) is attainable with luck.
- Death is rare for a runner (~0.6%). Danger comes mainly from the
  player choosing to fight, or from future variants with harsher police.
- Capacity matters: with the $30k-repay policy, median score was about
  $388k at 60 slots, $602k at 100 and $835k at 150 (90th percentile
  $1.09M / $1.70M / $2.41M). Capacity scales outcomes roughly in
  proportion once cash is no longer the constraint.

### 14.4 Tuning knobs and expected effects

| Knob | Increase makes the game… | Notes |
|---|---|---|
| `DEBT_RATE` | harder, with sharper early pressure | 5% is forgiving; 12%+ is punishing |
| `START_DEBT` | harder | Interacts with rate |
| `START_CASH` | easier (faster phase-1) | |
| `START_CAPACITY` | easier, with higher ceilings | Largest effect on 90th-percentile score |
| `P_STOCK` | easier (more options per day) | `MIN_STOCK` guards against empty markets |
| `P_CRASH`, `P_SPIKE` | higher variance, easier | |
| `P_COP_*`, `COP_HP` | harder, forces gun purchases | |
| `MAX_DEBT` | adds leverage options for experts | The bot did not model borrowing; expect skilled players to borrow early |
| `BANK_RATE` | rewards idle cash, reduces pressure to trade | >5% risks dominating late-game strategy |

### 14.5 Design notes on specific rules

- **Day-1 safe start (SET-5):** prevents the unrecoverable scenario of a
  mugging or cop fight before the player understands the game.
- **Cash-only forfeit on death (END-3):** makes banking meaningfully
  protective and makes death costly without being a zero.
- **Spike/crash applied to a random drug of all eight:** occasionally
  affecting an out-of-stock drug keeps events visible and prevents
  "dead" events.
- **No spread and static prices within a day (MKT-7, TRD-7):** the game
  is *perfect information within a day, uncertainty across days*. This
  is the property that makes it a clean teaching object for
  expected-value reasoning.

---

# 15. Acceptance Tests

Tests assume an engine exposing the state of §4 and a seedable RNG that
can be **stubbed** (forced return values) per NFR-1. "Given/When/Then"
format.

| ID | Requirement | Test |
|---|---|---|
| T-01 | SET-4 | **Given** a new game **Then** cash=2000, debt=5500, bank=0, health=100, capacity=100, guns=0, day=1, location=Bronx, inventory empty. |
| T-02 | SET-5 | **Given** a new game **Then** the Day-1 market has ≥4 in-stock drugs, no market event is announced, and no encounter has occurred, for 1,000 different seeds. |
| T-03 | MKT-2 | **Given** stock rolls that fail for all drugs **Then** exactly 4 drugs end up in stock. |
| T-04 | MKT-3 | **Given** 10,000 generated markets **Then** every price lies within its drug's `[min, max]` (before event modifiers). |
| T-05 | MKT-4 | **Given** a forced crash on Cocaine with base price 20,000 **Then** the price is 5,000. **Given** a forced crash with base price 70 on Speed **Then** 17. **Given** price 3 and crash **Then** price is 1. |
| T-06 | MKT-4 | **Given** a forced spike on Weed with base price 500 **Then** price is 2,000. |
| T-07 | MKT-4 | **Given** a forced event on an out-of-stock drug **Then** it appears in the market, modified. |
| T-08 | TRD-2 | **Given** cash 2,000, price 500, freeSpace 100 **Then** maxBuy=4. **Given** cash 100,000, price 20, freeSpace 90 **Then** maxBuy=90. |
| T-09 | TRD-3 | **Given** a buy of quantity maxBuy+1 or 0 or −3 **Then** rejected, state unchanged. |
| T-10 | TRD-4 | **Given** buy of 3 units at $500 with cash 2,000 **Then** cash=500 and inventory +3. |
| T-11 | TRD-5 | **Given** a held drug that is out of stock here **Then** it cannot be sold. |
| T-12 | TRD-6 | **Given** 10 units, price $100, sell 4 **Then** cash +400 and inventory 6. |
| T-13 | TRD-9 / FIN-7 | **Given** guns=2 (10 slots) and capacity 100 **Then** freeSpace with empty inventory = 90. |
| T-14 | FLOW-2 | **Given** current location Bronx **Then** Jet to Bronx is rejected. |
| T-15 | FLOW-6 / FIN-3 | **Given** debt 5,500 and three consecutive jets **Then** debt = 6,050, 6,655, 7,321 (the third is 7,320.5 rounded up). |
| T-16 | FIN-5 | **Given** bank 1,000 and two jets **Then** bank = 1,050, then 1,102. |
| T-17 | FLOW-6 | **Given** health 98 **Then** after a jet health=100 (cap); **Given** health 50 **Then** 55. |
| T-18 | EVT-7 | **Given** a forced mugging, cash 2,000, loss fraction 0.25 **Then** cash=1,500. **Given** cash 0 **Then** no change and no announcement. |
| T-19 | FIN-6 | **Given** bank 5,000 and a mugging **Then** bank unchanged. |
| T-20 | EVT-9 | **Given** a found-drugs roll of 8 units and freeSpace 3 **Then** 3 units are added. **Given** freeSpace 0 **Then** nothing is added or announced. |
| T-21 | EVT-10 | **Given** a coat offer at $150 and cash 100 **Then** accept is disabled. **Given** cash 500 and accept **Then** cash=350 and capacity=140. |
| T-22 | EVT-2 | **Given** guns=0 **Then** Fight is unavailable. |
| T-23 | EVT-3 | **Given** a forced run success **Then** the encounter ends with no state change except the log. **Given** forced failure with 2 cops both hitting for 7 **Then** health −14 and a new round begins. |
| T-24 | EVT-4 | **Given** 2 guns, both forced to hit for 10, against 3 cops stubbed to 15 HP each **Then** gun 1 takes cop 1 to 5 HP, gun 2 (cop 1 is still the first living cop) takes it to −5 and kills it; cops 2 and 3 are untouched and the excess damage is discarded. |
| T-25 | EVT-8 | **Given** all cops killed **Then** loot is awarded per cop killed within $50–$150 each. |
| T-26 | FLOW-5 / END-3 | **Given** health ≤ 0 with cash 10,000, bank 5,000, debt 2,000 **Then** game over with score 3,000 and outcome "killed". |
| T-27 | FIN-1 | **Given** cash 1,000 and debt 5,500 **Then** repay of 1,001 is rejected; pay-all pays 1,000. |
| T-28 | FIN-2 | **Given** debt 5,500 **Then** maximum borrow = 14,500. **Given** debt 20,000 **Then** borrowing is refused. |
| T-29 | LOC-1 | **Given** location Ghetto **Then** Loan Shark, Bank and Gun Shop are unavailable. |
| T-30 | FIN-7 | **Given** guns=3 **Then** buying a fourth is rejected. **Given** freeSpace 4 **Then** buying a gun is rejected. |
| T-31 | FLOW-4 / END-1 | **Given** day 30 **Then** Jet is replaced by Finish, and finishing with inventory prompts a warning and counts inventory as $0. |
| T-32 | END-2 | **Given** cash 10,000, bank 5,000, debt 2,000 on finish **Then** score=13,000. Negative scores are allowed. |
| T-33 | END-4 | **Given** a full table of 10 scores **Then** a lower-than-lowest score does not qualify; a tie with the lowest does not displace it. |
| T-34 | RNG-2 | **Given** the same seed and the same inputs run twice **Then** state hashes after every action are identical. |
| T-35 | PER-3 | **Given** a corrupt high-score file **Then** the game starts with an empty table without crashing. |
| T-36 | NFR-1 | The engine test suite runs headless, with no UI dependencies. |
| T-37 | Soak | **Given** 10,000 random-action games **Then** no invariant is violated: money ≥ 0 and integer, freeSpace ≥ 0, health ≤ 100, day ≤ 30, debt ≤ MAX_DEBT unless interest pushes it above (interest MAY exceed the cap; borrowing may not). |

> **Note on T-37 / MAX_DEBT:** `MAX_DEBT` constrains *borrowing*, not
> interest accrual. Debt can exceed 20,000 through compounding.

---

# PART II — APPENDICES: OPTIONAL VARIANTS

Each appendix states **Modifies** (sections of Part I) and **Default:
Off**. Variants may be combined unless a conflict is noted.

---

## Appendix A — Extended Drug Roster and Regional Prices

**Modifies:** §3.5, §6.

A.1 Expand the roster to 12 drugs by adding:

| Drug | Min | Max |
|---|---:|---:|
| Opium | $4,500 | $8,000 |
| MDA | $1,500 | $4,500 |
| Shrooms | $600 | $1,350 |
| Peyote | $220 | $700 |

A.2 `MIN_STOCK` becomes 6. `P_STOCK` becomes 70%.

A.3 **Regional bias (optional sub-variant).** Each location has a
per-drug multiplier in `{0.8, 1.0, 1.25}` applied to the rolled price
(before events), fixed for the game. Example: Coney Island ×0.8 on Weed,
Manhattan ×1.25 on Cocaine. This creates learnable routes, which suits a
"strategy" focus but reduces the pure-randomness of canonical play.

A.4 Capacity tuning: raise `START_CAPACITY` to 120 so the larger roster
doesn't squeeze the player.

---

## Appendix B — Hired Help ("Runners")

**Modifies:** §3, §8, §10. (Some ports call these helpers by a crude
name; this spec uses "Runners".)

B.1 In the Bronx, the player may hire up to `MAX_RUNNERS = 5` runners at
`RUNNER_PRICE = $1,000` each. Each runner adds `RUNNER_SPACE = +20`
capacity.

B.2 Each runner takes a daily cut: on arrival, `RUNNER_UPKEEP = $50` per
runner is deducted (down to cash 0; unpaid upkeep makes a runner quit
with probability 50%).

B.3 Each runner adds a combat participant: in a fight, each runner
provides one extra attack with `GUN_HIT_CHANCE` but is killed (removed)
with probability `COP_HIT_CHANCE / 2` per cop volley instead of the
player taking damage first. Runners absorb hits before the player.

B.4 Rationale: B turns capacity from an uncontrolled bonus (the coat)
into a decision with ongoing cost, a classic investment-vs-return
trade-off.

---

## Appendix C — Hospital and Health

**Modifies:** §3.1, §5.3, §10.

C.1 Remove natural regeneration (`HEALTH_REGEN = 0`).

C.2 Add a **Hospital** in the Bronx. Healing costs `HEAL_COST = $25` per
health point, any amount `1 … (MAX_HEALTH − health)`.

C.3 Optional: low health (<30) halves `RUN_SUCCESS` and reduces gun hit
chance by 10 points.

C.4 Optional: when the player is killed *and* has `bank ≥ HOSPITAL_FEE =
$5,000`, the player is instead revived: the fee is deducted from the
bank, health is set to 25, inventory is lost, and the game continues.
This softens death for beginners at a real price.

---

## Appendix D — Game Length and End Conditions

**Modifies:** §3.1, §5.2 (FLOW-4), §11.

| Mode | `GAME_DAYS` | Notes |
|---|---:|---|
| Sprint | 15 | Pair with `START_DEBT = 3,000` |
| Classic | 30 | Canonical |
| Marathon | 60 | Reduce `DEBT_RATE` to 5% to keep the debt survivable; high-score tables are kept per mode |
| Endless | ∞ | The game ends only by death, by paying off the debt and reaching a goal net worth (default $1,000,000), or by player retirement |

D.1 **Auto-liquidation (optional).** On Finish, unsold inventory is sold
at that day's local prices (out-of-stock drugs are worth 0). This
removes the "forgot to sell" failure mode that frustrates newcomers.

D.2 **Early win.** If the player holds `debt = 0` and `cash + bank ≥
WIN_TARGET` at any time, they may retire early and score immediately.

D.3 High-score tables MUST be separate per mode and per enabled-variant
combination.

---

## Appendix E — Difficulty Presets

**Modifies:** §3. Presets are bundles of constants. Simulation (same bot
as §14.3, repay threshold $30k, 3,000 games) gives the following
indicative outcomes.

| Preset | Changes from canonical | Median | Score > 0 | Killed |
|---|---|---:|---:|---:|
| **Easy** | cash $3,000; debt $3,000; `DEBT_RATE` 5%; `P_CRASH` = `P_SPIKE` = 10% | $741,958 | 99.5% | 0.5% |
| **Normal** | (canonical) | $601,641 | 98.2% | 0.6% |
| **Hard** | `DEBT_RATE` 12%; `P_STOCK` 65%; `P_CRASH` = `P_SPIKE` = 5%; capacity 60; `P_MUG` 10%; `P_COP_BASE` 8% | $150,772 | 82.3% | 0.5% |

E.1 Presets MUST be selectable at game start and recorded on the
high-score entry. Scores from different presets MUST NOT be mixed in one
table.

E.2 A designer's note: Hard still leaves 82% of simple bots finishing
positive, so a "Brutal" preset (additional: `START_CASH` $1,000,
`MAX_DEBT` $10,000, cops with `COP_HP` = 25 + 2×day) is recommended if
the audience is expert.

---

## Appendix F — Loan Shark Enforcement

**Modifies:** §10.1, §8.

F.1 Add a **Goon Threat** system. At arrival, if `debt > GOON_THRESHOLD`
(default: `3 × START_DEBT`), a goon encounter replaces the normal
hostile roll with probability `P_GOON = 20%`.

F.2 A goon encounter forces a choice: **Pay** (hand over `GOON_DEMAND =
10%` of debt in cash if available; if not, lose health), **Fight**, or
**Run** using the §8 combat mechanics with 2 goons whose HP is `30 +
day`.

F.3 If the player repays the debt completely, goon encounters stop for
the rest of the game.

F.4 Rationale: introduces a second cost to deferring debt beyond
interest and rewards the knee-jerk "repay at once" strategy, shifting
the optimal policy of §14.2 toward earlier partial repayment.

---

## Appendix G — Dynamic Economy (Player Impact and Price Memory)

**Modifies:** §6.1 (MKT-1, MKT-7), §7.

G.1 **Price impact.** Selling `q` units of a drug at a location raises
its supply: the *next* time the player visits that location, the base
price for that drug is multiplied by `0.97^(q/10)` (floor 0.5). Buying
`q` units raises the next visit's price by `1.03^(q/10)` (cap 2.0). The
effect decays by 50% per day.

G.2 **Within-day impact (harder).** Prices change after every
transaction: each unit sold lowers the price by `IMPACT = 0.2%`; each
unit bought raises it by the same. This makes mass dumping costly and
makes `MAX` trades fundamentally different from the canonical game; the
"price is constant" invariant (MKT-7) is dropped.

G.3 **Persistent demand shocks.** Each location carries a hidden
per-drug demand multiplier that does a mean-reverting random walk (daily
σ = 5%). This lets alert players detect patterns.

G.4 Testing note: G.2 invalidates T-08–T-12 expected values;
implementations must supply variant-specific tests.

---

## Appendix H — Information Brokers and Rumors

**Modifies:** §6, §10 (adds a service).

H.1 Add an **Info Broker** in Central Park, who sells, for `RUMOR_PRICE
= $200`, a hint about **tomorrow's** destination: for a chosen location,
the broker reveals which drug(s) will crash or spike there (accuracy
80%; 20% of the time the rumor is false).

H.2 Alternatively, rumors arrive free as random windfalls (`P_RUMOR` =
5%) with 70% accuracy.

H.3 Rationale: converts luck into a purchasable, imperfect-information
decision, which is good for teaching expected value of information.

---

## Appendix I — Daily Challenge and Replay

**Modifies:** §12, §11.5.

I.1 **Daily seed:** the seed is a hash of the date (UTC) and the game
version. All players that day face the identical sequence of markets and
events (given identical inputs).

I.2 A separate "Daily" high-score table (one entry per player per day)
is kept.

I.3 **Replay:** the engine exports `{seed, variants, constants,
commands[]}`. A replay viewer re-executes it deterministically. Any
divergence is a bug (RNG-2).

I.4 **Anti-cheat (online only):** the server re-simulates submitted
command logs and rejects scores that do not match.

---

## Appendix J — Multiplayer

**Modifies:** §1.3, §4, §5, §11.5.

### J.1 Hot-seat (2–4 players)

- All players share the same seed and *identical markets each day* (a
  fair-play race), but have separate states.
- Turn order is fixed; a day completes when every player has traveled.
  Player order within a day rotates.
- The winner is the highest score; ties are resolved by lower debt, then
  by earlier finish.

### J.2 Networked competitive

- Each player has independent state but a shared seed per day (as in
  Appendix I).
- Optional **PvP robbery:** when two players arrive at the same location
  on the same day, a "showdown" may be offered; the winner of a fight
  takes `ROB_PCT = 20%` of the loser's cash.
- Disconnection policy: AI autopilot (simple bot of §14.3 style) takes
  over after a 60 s timeout.

### J.3 Co-op

- Players share a single debt. Cash, bank, and inventories are separate;
  players may transfer cash at the same location.

---

## Appendix K — Reskins and Content-Neutral Themes

**Modifies:** §2, §3.5, §3.6, UI text. **No rules change.**

K.1 A **theme file** maps names only: drugs → commodities, locations →
locations, events → flavor text, titles → rank titles. All numeric
constants stay as in §3.

K.2 Example neutral themes:

| Theme | Commodities (high → low value) | Premise |
|---|---|---|
| **Space Freight** | Antimatter, Quantum chips, Helium-3, Ore, Water, Rations, Scrap, Filters | Pay off a station-loan shark; police become customs |
| **Sneaker Reseller** | Rare dunks, Limited collabs, … | Arbitrage between city stores |
| **Spice Route** | Saffron, Silk, … | Historical trade caravan with bandits |
| **Stock Floor** | Eight fictional tickers | Margin loan = debt; compliance officers = cops |

K.3 Themes MUST NOT alter requirement IDs; the acceptance tests of §15
apply unchanged (tests should reference drugs by *index*, not name).

K.4 Rationale: lets an instructor or platform use the same engine and
test suite with audience-appropriate content.

---

## Appendix L — Engineering and Teaching Extensions

**Modifies:** §13.5 (NFR). Informative; useful for courses using
DopeWars as a project.

L.1 **Bot API.** The engine exposes a read-only `observe()` (full state
+ legal actions) and `act(command)`. This enables student-written bots
and tournament harnesses (cf. §14.3).

L.2 **Telemetry.** Optional JSONL logging per action `{day, action,
args, stateHash}` for analysis, grading, or debugging.

L.3 **Property-based tests.** Suggested invariants (extend T-37): money
integer-valued and non-negative; `freeSpace ≥ 0`; replay determinism;
score identity `score = cash + bank − debt` on every terminal state; no
action sequence advances the day more than once per Jet.

---

## Appendix M — Known Ambiguities and Folklore Differences (Informative)

For implementers comparing against other versions:

1. **Drug rosters** differ (6, 8, 12+ drugs); some ports include only
   cheaper or only dated drug names.
2. **Debt/bank rates**: classic debt interest of 10%/day is widely
   reported; bank rate varies by port. Treat 5% here as a design
   default.
3. **End-of-game inventory:** some ports auto-sell, some don't, some
   force a final market. This spec chooses "$0 unless sold" with a
   warning (Appendix D.1 offers auto-sell).
4. **Mid-game borrowing:** some ports allow extra loans only if the debt
   is zero; this spec uses a `MAX_DEBT` cap.
5. **Services:** the gun shop, hospital, and bank appear in different
   locations across ports.
6. **Combat:** the original had simple "Run/Fight" flavor; the numeric
   combat model here (HP, hit chance) is a spec design, not a historical
   reconstruction.

---

*End of specification.*
