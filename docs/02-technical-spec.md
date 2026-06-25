# Smart Liquidity Trader V1 — Technical Specification

**Status:** Draft v1 — derived from locked strategy decisions
**Scope:** Phase 1 (Visual Indicator) → Phase 2 (Strategy Backtest) for Pine Script v5
**Instruments:** XAUUSD (Gold), NAS100
**Timeframes:** 1m / 5m execution

---

## 1. Locked Strategy Rules (source of truth)

| # | Component | Rule |
|---|---|---|
| 1 | Key Levels | Tier 1: Previous Day High/Low · Tier 2: Asia Session High/Low · Tier 3: Volume Profile POC/VAH/VAL (prior day, always active; earlier days carried forward indefinitely while untested — see Section 4.1a) |
| 2 | Sweep | Price trades through (wick beyond) a key level |
| 3 | CISD | ≥3 consecutive same-direction-close candles → confirming candle closes back through the **open of the first candle in that run** → confirming candle body ≥ 1.5× average body of prior 14 candles |
| 4 | FVG | 3-candle gap where candle 2 = the CISD-confirming candle; confirmed only after candle 3 closes |
| 5 | Entry | Price retraces into the FVG zone |
| 6 | SL | The swept swing level; capped at 10 pts (Gold) / 50 pts (NAS100) — if structural SL exceeds cap, **skip the trade** |
| 7 | TP | Fixed R:R = 1:2 (no Volume Profile in V1) |
| 8 | Trailing Stop | None in V1 |
| 9 | Risk | Fixed $250 per trade, position size derived from SL distance |
| 10 | Daily Limits | Max 2 trades/day · stop after 2 losing trades (count-based, not PnL-based) |
| 11 | Session Filter | London Killzone (06:00–09:00 UTC) + NY AM Killzone (12:00–13:30 UTC) + NY PM Killzone (13:30–20:00 UTC) — *expanded from the original philosophy doc's "London + NY AM only" rule* |
| 12 | Repainting | Strictly forbidden — every value must be confirmed-bar-only |

---

## 2. System Architecture — Pipeline Order

```
┌─────────────────┐
│ Session Engine   │  (is current bar inside a tradeable killzone?)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ Level Engine     │  (PDH/PDL, Asia H/L — locked, non-repainting)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ Sweep Detector   │  (has price traded through an unswept level?)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ CISD Engine      │  (consecutive-close run + displacement confirm)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ FVG Engine       │  (3-candle gap around CISD candle, confirmed bar+1)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ Trade Builder    │  (entry trigger, SL/TP calc, SL cap filter)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ Trade Manager    │  (daily trade/loss counters, position sizing)
└────────┬─────────┘
         ▼
┌─────────────────┐
│ Visualization    │  (Phase 1 plotting — levels, sweeps, CISD, FVG, trades)
└──────────────────┘
```

Each block is a **separate logical module** in code (not necessarily separate files, since Pine v5 is single-file per script, but separated by clearly commented sections / functions). This separation matters for Phase 1: each module must be independently visually verifiable before the next one is trusted.

---

## 3. Core Data Structures (Pine v5 UDTs)

Pine Script v5 supports user-defined types (`type`), which we'll use to keep state organized instead of scattering loose `var` variables.

```pinescript
type KeyLevel
    float price
    string label        // "PDH", "PDL", "AsiaH", "AsiaL"
    bool   swept
    int    sweepBarIndex

type CISDCandidate
    int    runStartBar
    int    runEndBar
    float  cisdLevel        // open of first candle in the run
    string direction        // "bullish" / "bearish"
    bool   confirmed
    int    confirmBarIndex

type FVGZone
    float  top
    float  bottom
    string direction
    int    candle1Bar
    int    candle2Bar       // = CISD confirm bar
    int    candle3Bar
    bool   active           // not yet filled/invalidated

type TradeSetup
    string direction
    float  entryPrice
    float  slPrice
    float  tpPrice
    float  slDistance
    float  positionSize
    bool   valid             // false if SL cap exceeded → trade skipped
```

State arrays (using Pine's `array<type>`) will track:
- `array<KeyLevel> activeLevels` — current day's PDH/PDL + Asia H/L
- `array<CISDCandidate> pendingCISDs` — runs being tracked, not yet confirmed
- `array<FVGZone> activeFVGs` — confirmed, unfilled FVGs awaiting retracement
- Daily counters: `var int tradesToday`, `var int lossesToday`, reset on new day

---

## 4. Component Design Detail

### 4.1 Session Engine
- Determines: (a) new-day rollover, (b) Asia session boundary, (c) whether current bar is inside London Killzone or NY AM Killzone.
- Pine v5 approach: use `time()` with an explicit session string and timezone parameter, or manual hour-based comparison against `time("","tz=...")`. **Recommend explicit timezone string** (e.g. `"UTC"`) rather than exchange/chart timezone, so behavior is identical regardless of what timezone the user's TradingView account is set to.
- New day detection: `dayofweek`/`dayofmonth` change check, or `ta.change(time("D"))`.

**Open decision needed (see Section 6):** exact UTC windows for Asia session, London Killzone, NY AM/PM Killzone.

### 4.1a Volume Profile Key Levels (Tier 3) — *added beyond original V1 scope*
- **What's tracked**: Previous day's POC, VAH, VAL — computed from a per-day volume-by-price distribution, 500 row granularity, Value Area = 70% of total volume.
- **Volume source**: tick volume (count of price changes), not true traded volume — standard default for Gold/NAS100 retail CFD feeds in TradingView, which typically don't carry genuine traded-size data the way listed equities do. Flagged as an assumption, not yet broker-confirmed.
- **Lock timing**: identical mechanic to PDH/PDL — the previous day's POC/VAH/VAL is finalized the instant that day closes, then never recalculated. Non-repaint-safe by construction.
- **Carry-forward rule (Option B, confirmed)**: any day's POC/VAH/VAL — not just D-1 — remains an active, sweepable level for as long as it stays untested, with no cutoff on how many days back this goes.
- **"Tested" definition** — 🔲 **deferred by user request.** Default assumption for now (subject to revision): reuse the same sweep mechanism already used for PDH/PDL/Asia (price wicks through the level marks it tested). To be revisited explicitly *after* the model is built and initial results are seen — not before.
- **Performance safeguard** — since untested levels can accumulate indefinitely, the implementation will include an internal safety cap/pruning mechanism for extremely old, far-away levels purely to protect Pine Script array/runtime limits on long backtests. This is an engineering safeguard only, not a change to the Option B rule, and will be tuned to never trigger within any realistic backtest window.
- **Pipeline integration**: VP levels feed into the exact same Sweep Detector → CISD Engine → FVG Engine pipeline as Tier 1/2 levels — no separate logic path needed downstream of the Level Engine.

### 4.2 Level Engine
- **PDH/PDL**: Must be the *finalized* high/low of the previous calendar day — only locks in at the first bar of the new day, using `request.security(syminfo.tickerid, "D", high[1])` / `low[1]` pattern, or tracking via daily `ta.highest`/`ta.lowest` reset on day-change with a 1-day offset. Critical: must not use the *current developing* day's high/low.
- **Asia H/L**: Tracked live *during* the Asia session (running high/low updates as Asia session bars print), then **frozen** the instant the Asia session ends. After freeze, the value never changes for the rest of the day. This is what makes it valid to use during London/NY without repainting — it's fixed before those sessions begin.
- Each `KeyLevel` gets a `swept` flag, set to `true` the first time price trades through it intraday — prevents the same level from generating duplicate sweep signals.

### 4.3 Sweep Detector
- Triggered when `high >= level.price` (for a high-side level) or `low <= level.price` (for a low-side level), **and** `level.swept == false`.
- On trigger: mark `level.swept = true`, record `sweepBarIndex`, and open a "CISD search window" looking for a consecutive-close run starting from that bar.
- Sweep direction determines which CISD direction we're now watching for (sweep of a *high* → watching for *bearish* CISD; sweep of a *low* → watching for *bullish* CISD).

### 4.4 CISD Engine
- After a sweep, scan forward bar-by-bar (in real time, this just means "on each new bar close") for ≥3 consecutive candles closing in the sweep's direction.
- `cisdLevel` = `open` of the first candle in that run — **locked in once the run reaches 3 candles**, doesn't move afterward even if the run continues.
- On each subsequent bar, check: does this candle's `close` break back through `cisdLevel` in the opposite direction? If yes →
  - Check displacement: `abs(close - open) >= 1.5 * average_body(14)`, where `average_body(14)` = average of `abs(close[i]-open[i])` over the prior 14 closed bars (not including the current one, to avoid self-reference skew — confirm this is the intended interpretation, see Section 6).
  - If displacement check passes → CISD confirmed, candidate promoted to active, `confirmBarIndex` recorded.
  - If displacement check fails → the close-through is noted, but does **not** confirm CISD. (Open question: does the run simply continue/reset, or is this sweep opportunity discarded entirely? See Section 6.)

### 4.5 FVG Engine
- On CISD confirmation at bar `N` (candle 2): wait for bar `N+1` to close (candle 3).
- Compute gap:
  - Bullish: `candle1.high < candle3.low` → zone = `[candle1.high, candle3.low]`
  - Bearish: `candle1.low > candle3.high` → zone = `[candle3.high, candle1.low]`
- If no valid gap exists (candles overlap), **no FVG forms** — this sweep/CISD produces no trade. This is a real possibility and must be handled (not every CISD produces a tradeable FVG).
- Zone becomes `active = true` only after candle 3's bar is fully closed (`barstate.isconfirmed` check) — this is the critical non-repaint gate for this module.

### 4.6 Trade Builder
- Entry trigger: price (next bars after FVG forms) trades back into `[zone.bottom, zone.top]`.
- On entry trigger:
  - `slPrice` = the swept swing level price (from the original `KeyLevel` that was swept).
  - `slDistance` = `abs(entryPrice - slPrice)`.
  - **Cap check**: if `slDistance > 10` (Gold) or `> 50` (NAS100) → `valid = false`, trade is skipped, zone is marked inactive (don't retry the same zone).
  - `tpPrice` = `entryPrice + (slDistance * 2)` in trade direction (1:2 R:R).
  - `positionSize` = `250 / (slDistance * pointValue)` — `pointValue` needs an instrument-specific constant (or use `syminfo.pointvalue` / `syminfo.mintick` mapping depending on broker feed specifics — needs confirmation per instrument, see Section 6).

### 4.7 Trade Manager
- `tradesToday`, `lossesToday` — reset at start of new day (tied to Session Engine's day-rollover detection).
- Before allowing any new trade: `if tradesToday >= 2 or lossesToday >= 2: block new entries`.
- On trade close (Phase 2, strategy mode): increment `tradesToday`; if closed at loss, increment `lossesToday`.

### 4.8 Visualization Layer (Phase 1 priority)
Must visually render, non-repainting, on the chart:
- PDH/PDL and Asia H/L as horizontal lines/boxes, clearly labeled, extending only across their valid window
- Sweep markers (e.g. small triangle/X at the sweep bar)
- CISD line + confirmation marker (showing the level and the confirming candle)
- FVG zones as shaded boxes, with a visual distinction between active/filled/invalidated
- Entry/SL/TP markers once a trade triggers, including setups that were **skipped due to SL cap** (shown differently, e.g. greyed out) — this is important for visually auditing that the selectivity filter is working as intended, not just hiding rejected setups silently

---

## 5. Non-Repaint Checklist (must hold for every module)

| Module | Repaint risk | Mitigation |
|---|---|---|
| PDH/PDL | Using current day's still-forming H/L instead of prior day's finalized H/L | Only reference `[1]`-offset daily values, locked at day open |
| Asia H/L | Updating the level after Asia session ends | Freeze value the instant the session boundary is crossed; never update afterward |
| CISD level | Using a candle's close before that candle has actually closed | All "close" checks gated by `barstate.isconfirmed` |
| Displacement avg | Including the current (still-forming) bar in the 14-bar average | Average must use **only fully closed** historical bars |
| FVG candle 3 | Drawing the gap before candle 3's wick is final | FVG zone only instantiated after candle 3 closes |
| Session/killzone | Chart-timezone dependent behavior changing per-user | Use explicit fixed timezone string everywhere, not chart default |

Phase 1 deliverable is **not considered done** until every plotted signal is confirmed identical on historical replay vs. live bar-by-bar progression (standard repaint audit: load on history, then watch it form in real time bar by bar, compare).

---

## 6. Open Technical Decisions (need your input before coding starts)

These are new — they came up while designing the architecture itself, not from the original strategy debate.

1. **Timezone for sessions** — what timezone should Asia/London/NY windows be defined in? (UTC is recommended for consistency regardless of broker/user settings — confirm?)
2. **Exact session/killzone hours** — ✅ confirmed:
   - Asia session (level-calculation window only, not tradeable): 00:00–04:00 UTC
   - London Killzone (tradeable): 06:00–09:00 UTC
   - NY AM Killzone (tradeable): 12:00–13:30 UTC
   - NY PM Killzone (tradeable): 13:30–20:00 UTC ✅ confirmed — added as a 3rd tradeable session, expanding beyond the original philosophy doc's "London + NY AM only" rule (see Section 1, item 11).
3. **14-bar displacement average** — should the current (still-forming) bar be excluded from the average, using only the prior 14 *closed* bars? (Recommended yes, to avoid the bar comparing itself into its own baseline.)
4. **Failed displacement check** — if a close-through happens but fails the 1.5× body filter, does the consecutive-close run get discarded (no more searching off this sweep), or does the engine keep watching for a stronger close on a later bar?
5. **Point value / instrument specifics** — confirm contract/point-value handling for position sizing on your specific broker feed for XAUUSD and NAS100 (this affects the `$250 → lot size` math directly and varies by broker).
6. **Indicator vs Strategy mode for Phase 1** — Phase 1 is "visual validation," which can be built as a pure `indicator()` script (no backtesting yet) before becoming a `strategy()` script in Phase 2. Confirm we build Phase 1 as `indicator()` first.
7. **VP "tested" definition** — deliberately deferred, see Section 4.1a. To be revisited after the model is built and initial results are reviewed.

---

## 7. Phase 1 Definition of Done

Phase 1 is complete only when, on both XAUUSD and NAS100 historical charts:
- [ ] PDH/PDL and Asia H/L plot correctly and never shift retroactively
- [ ] Sweeps are marked only when genuinely swept, with no false positives/negatives on manual spot-check
- [ ] CISD lines and confirmations match manual chart annotation for at least 20 sample events per instrument
- [ ] FVG zones are correctly bounded and only appear after candle 3 closes
- [ ] Skipped-due-to-SL-cap setups are visibly distinguishable from valid setups
- [ ] A live bar-by-bar replay produces identical signals to the same period loaded as history (repaint audit passes)

Only after this checklist passes do we move to Phase 2 (strategy/backtest mode).
