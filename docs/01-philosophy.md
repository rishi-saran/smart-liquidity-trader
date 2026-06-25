# Smart Liquidity Trader V1 — Project Context & Strategy Philosophy

## What We Are Building

We are building a TradingView Pine Script v5 trading system called **Smart Liquidity Trader V1**.

The objective is not to create another indicator that produces dozens of signals every day. The objective is to create a highly selective trading system that identifies only the highest-probability liquidity-based reversals on Gold (XAUUSD) and NAS100.

The system must behave like a disciplined trader who waits for institutional liquidity events rather than chasing price.

This project is intended to evolve in stages:

**V1:**
- Visual Indicator
- Strategy Backtesting
- Trade Management
- Statistical Validation

**V2:**
- Alerts
- Webhooks
- Broker Automation
- Live Execution

**V3:**
- Advanced Filters
- Volume Profile
- Market Regime Detection
- Portfolio Management

> Note: Volume Profile key levels (POC/VAH/VAL) were deliberately pulled forward into V1 scope as an exception — see `02-technical-spec.md` and `03-decision-log.md` for details on this and other confirmed deviations from this original roadmap.

## Core Trading Philosophy

The strategy is built on a simple belief: **markets move because liquidity is being sought.**

Retail traders often buy breakouts and sell breakdowns. Institutions often use these liquidity pools to enter large positions. Because of this, price frequently:

1. Moves toward liquidity.
2. Sweeps liquidity.
3. Rejects the liquidity area.
4. Creates displacement.
5. Retraces.
6. Continues in the new direction.

The strategy is designed to capture this sequence.

## Why This Strategy Exists

Many Smart Money Concept strategies become difficult to automate because they rely heavily on subjective interpretation. Examples: Market Structure Shift, Order Blocks, Breakers, Premium/Discount Arrays, Liquidity Pools.

Most traders can visually identify these concepts, but computers struggle because the definitions are often subjective.

The goal of this project is different. **Every rule must be objective. Every condition must evaluate to TRUE or FALSE. No discretionary interpretation is allowed. The strategy must be fully deterministic.**

## How The Strategy Was Designed

After reviewing many ICT-style concepts, the strategy was simplified into a small number of highly important components.

Instead of using MSS, BOS, Order Blocks, Breakers, OTE, Multiple FVG Types, Volume Profile, and numerous confirmations, we reduced the system to:

**Liquidity Level → Liquidity Sweep → CISD → Displacement → FVG → Retracement → Entry**

This sequence forms the core edge of the strategy.

## Why We Focus On Specific Levels

The strategy intentionally focuses on only a few liquidity locations. These levels consistently attract price.

Current V1 levels:

**Tier 1:**
- Previous Day High
- Previous Day Low

**Tier 2:**
- Asia High
- Asia Low

These levels were chosen because they repeatedly act as liquidity targets during London and New York sessions. The system is intentionally selective. More levels do not necessarily produce better results.

## Why Session Timing Matters

Not every hour of the trading day is equal. Liquidity events tend to occur during specific windows.

The strategy only trades during the **London Killzone** and **New York AM Killzone**.

The purpose is to avoid dead market conditions, random consolidations, and low participation environments. The system should prioritize quality over quantity.

## CISD Philosophy

CISD (Change in State of Delivery) is the mechanism used to identify that a liquidity sweep may have completed.

The CISD is not intended to be a traditional market structure break. Instead, it represents a strong shift in delivery after liquidity has been taken.

**Important:** the CISD must be a displacement candle. A weak candle is not sufficient. The displacement itself is a critical part of the edge.

## Why FVG Is Used

The Fair Value Gap is not the signal. The Fair Value Gap is the entry mechanism. The actual signal is: **Liquidity Sweep + CISD.**

The FVG simply provides a retracement location for efficient entry. This distinction is important. Without a valid sweep and valid CISD, an FVG alone has no value.

## Risk Philosophy

The strategy is not designed to trade frequently. The objective is to find a small number of A+ setups.

Current constraints:
- Maximum 2 trades per day.
- Maximum 2 losses per day.
- Fixed risk per trade.
- Hard daily loss limit.

The goal is consistency rather than activity.

## Development Philosophy

The system should be developed in stages.

**Phase 1: Visual validation.** The strategy should first prove that it correctly identifies levels, sweeps, CISDs, and FVGs before any execution logic is trusted.

**Phase 2: Backtesting.** The strategy must be statistically validated.

**Phase 3: Optimization.** Only after collecting data should parameters be adjusted.

**Phase 4: Automation.** Only after proving profitability should broker execution be introduced.

## Non-Repainting Requirement

This is a hard requirement. The system must never repaint. Historical signals and real-time signals must be identical. Any logic requiring future candles must wait for confirmation before becoming valid. No lookahead behavior is allowed.

## What Success Looks Like

Success is not: producing the most trades, producing the most signals, or looking visually impressive.

Success means: objective rules, non-repainting behavior, reliable backtesting, consistent execution, high-quality setups, clean architecture, and future automation readiness.

The end goal is a professional-grade systematic trading framework that can eventually power a fully automated execution system.
