# ETF Sitch — Rust Backtest Spec

Port of `etf_sitch_cpp.cpp` into the `strat-scaffold` framework for historical backtesting
and iterative alpha research.

---

## Overview

Intraday mean-reversion strategy on ETFs. Detects when an ETF's price has "switched"
(deviated significantly from short-term fair value) and fades the move. Both long
("down sitch") and short ("up sitch") directions. Universe: ETFs and ETNs excluding
oil/gas, healthcare, biotech sectors and a manually maintained exclusion list.

---

## Data Inputs

| Data | Source | How loaded |
|---|---|---|
| Daily OHLCV + adjusted prices | `s3://zoa-factors/tiingo_eod.parquet` | Read once at backtest startup; indexed as `HashMap<(Symbol, Date), DailyBar>` |
| Intraday trades | `s3://zoa-daily-per-symbol-mbo-{year}/{exchange}/{date}/{symbol}.mbo.dbn.zst` | Streamed via `TradeReceiver` — filter `Action::Trade` from MBO events |
| ETF/ETN universe | `s3://zoa-factors/symbols.parquet` | Read once at startup; filter `type IN (Etf, Etn)` |
| Excluded ETFs | `shit_etfs.csv` (local config file) | Loaded into `HashSet<String>` at startup |
| Sector exclusion lists | Hardcoded UUIDs in C++ → replace with a static lookup table | Loaded as `HashSet<String>` of excluded symbols at startup |
| Tier 1 ETF list | Hardcoded lookup (S&P 500 / Russell 1000 ETFs + major broad-market ETFs) | Used for LULD band width selection |

### Daily bars needed per symbol (pre-loaded at day start)
- `prev_close` = `adjClose` from prior trading day
- `atr` = 14-day ATR computed from daily `high`, `low`, `adjClose`
- `daily_ma[period]` = SMA of `adjClose` for each period in `moving_average_periods`
- `open_price` = set from the first eligible trade of the day

### Simplifications vs live
- **NBBO bid/ask**: use last trade price as proxy. Affects only the `not_sitching_` stop
  refinement path — acceptable for backtesting.
- **ETB/HTB**: ignored. All symbols treated as shortable.
- **Exchange routing**: abstracted away. Backtest fills at the limit price.

---

## Config Struct

```rust
pub struct EtfSitchConfig {
    // Position sizing
    pub max_size: f64,           // max shares per position
    pub per_trade_risk: f64,     // max dollars risked per trade (drives size calculation)
    pub max_daily_drawdown: f64, // negative dollars — stop trading symbol if unrealized PL hits this
    pub ff_max_capital: f64,     // max notional capital per position

    // Signal thresholds
    pub pct_atr: f64,            // deviation required to trigger (in ATR multiples)
    pub min_dist: f64,           // minimum absolute price distance for entry
    pub min_trades: u32,         // minimum trades seen before entry allowed
    pub min_volume: u32,         // minimum cumulative volume while extended before entry

    // Timing
    pub give_up_secs: f64,       // seconds to wait for fill before cancelling entry
    pub start_offset_mins: u32,  // minutes after open before entering (default 15)
    pub end_offset_mins: u32,    // minutes before close to stop entering (default 15)

    // Moving averages (daily timeframe, computed at day start)
    pub moving_average_periods: Vec<u32>,

    // Universe
    pub shit_etfs: HashSet<String>,
    pub sector_exclusions: HashSet<String>,  // replaces the UUID-based sector lists
    pub tier1_etfs: HashSet<String>,         // for LULD band width
}
```

---

## LULD Band Computation (in-strategy, per symbol)

Kept inside `EtfSitchSymbolState` — not a separate engine component.

### Reference price
- Rolling 5-minute volume-weighted average of eligible trades
- Eligible: `trade.size >= 100` (exclude odd lots) + exclude known erroneous conditions
- Recomputed every 30 seconds (on a 30s timer callback, or lazily on each trade check)
- Maintained as a sliding deque of `(timestamp, price, size)` entries; drop entries
  older than 5 minutes

### Band width
```
tier1_pct_normal    = 0.05   // ±5%
tier1_pct_extended  = 0.10   // ±10% during 9:30–9:45 and 3:35–4:00
tier2_pct_normal    = 0.10
tier2_pct_extended  = 0.20

// For price < $3: band = max(20%, $0.15) regardless of tier
// Applied symmetrically: upper = ref * (1 + pct), lower = ref * (1 - pct)
```

### Initialization
At session open, seed `reference_price` with `prev_close` until 5 minutes of trades
have accumulated.

---

## Per-Symbol State

```rust
pub struct EtfSitchSymbolState {
    // --- Pre-loaded at day start ---
    pub atr: f64,
    pub prev_close: f64,
    pub open_price: Option<f64>,         // set on first trade
    pub daily_mas: HashMap<u32, f64>,    // period → SMA value
    pub lowest_ma: f64,                  // min across all periods
    pub highest_ma: f64,                 // max across all periods
    pub is_tier1: bool,

    // --- Intraday rolling MA (short-term, seconds-window) ---
    pub second_deque: VecDeque<(i64, f64, i32)>,  // (second_ts, sum, count)
    pub previous_second: i64,
    pub intra_second_sum: f64,
    pub intra_second_count: i32,
    pub seconds_sum: f64,
    pub seconds_count: i32,
    pub moving_average: f64,             // current short-term MA
    pub ma_window_secs: i32,             // default 5

    // --- LULD bands ---
    pub luld_window: VecDeque<(i64, f64, u64)>,  // (ts_ns, price, size)
    pub reference_price: f64,
    pub upper_band: f64,
    pub lower_band: f64,
    pub last_band_update_ns: i64,

    // --- Trade/volume tracking ---
    pub num_trades: u32,
    pub acc_volume: u64,
    pub last_trade_ts_ns: i64,
    pub curr_ts_ns: i64,
    pub daily_vwap_px_vol: f64,          // sum(price * size) for VWAP numerator
    pub daily_vwap_vol: u64,             // sum(size) for VWAP denominator

    // --- Minute bar ---
    pub bar_high: f64,
    pub bar_low: f64,
    pub bar_open_ts_ns: i64,             // timestamp of current 1-min bar start

    // --- Signal state ---
    pub signal: f64,                     // movingAverage_ at entry decision time
    pub entry_signal: f64,               // price level entry was signalled at
    pub dcum_vol_away: u32,              // cumulative volume while below MA (down sitch)
    pub cum_vol_away: u32,               // cumulative volume while above MA (up sitch)

    // --- Position state ---
    pub los: Option<Dir>,                // Long or Short (None = flat)
    pub is_pos: bool,
    pub average_entry_price: f64,
    pub exit_price: f64,
    pub initial_exit_price: f64,
    pub entry_time_ns: i64,
    pub cum_vol: u32,                    // volume accumulator for stop logic
    pub stop_flag: bool,
    pub not_sitching: bool,
    pub final_close: bool,
    pub first_fill_flag: bool,
    pub cancel_reset_attempts: u32,
    pub max_size_remaining: f64,         // decremented as orders placed

    // --- Daily risk ---
    pub unrealized_pl: f64,
    pub daily_drawdown_hit: bool,

    // --- Timing ---
    pub start_time_ns: i64,             // market_open + 15min
    pub end_time_ns: i64,               // market_close - 15min
    pub give_up_timer_active: bool,
}
```

---

## Event Handlers

### `on_trade(symbol, trade: TradeMsg)`

1. Increment `num_trades`, `acc_volume`. Update `curr_ts_ns`, `last_trade_ts_ns`.
2. Update daily VWAP accumulator.
3. Update 1-minute bar high/low. If `curr_ts_ns` crossed a 1-min boundary, reset bar.
4. Update LULD window (push trade if eligible, pop entries > 5 min old). Every 30s,
   recompute `reference_price` and recalculate bands.
5. Update intraday MA via `calc_moving_average()`:
   - If gap from last trade > 60s, clear MA and return (re-seed).
   - Aggregate into current second bucket; update `moving_average`.
6. If `moving_average < 0.1` → return (not yet warmed up).
7. If outside `[start_time_ns, end_time_ns]` → return.
8. If `num_trades < min_trades && acc_volume < min_volume` → return.
9. Check daily drawdown. If hit and not `final_close`, set `final_close`, schedule exit.
10. **Entry logic** (only if `!is_pos` and no pending orders):
    - Compute `daily_vwap = daily_vwap_px_vol / daily_vwap_vol`
    - **Down sitch (long)**:
      - `signal - last > atr * pct_atr`
      - `daily_vwap - last > atr * pct_atr`
      - `lowest_ma - last < atr * pct_atr` (confirms price is below all daily MAs)
      - Accumulate `dcum_vol_away`; check `min_dist`
      - Set `los = Long`, compute `entry_signal`, `exit_price`, call `enter()`
    - **Up sitch (short)**:
      - `last - signal > atr * pct_atr`
      - `last - daily_vwap > atr * pct_atr`
      - `last - highest_ma < atr * pct_atr`
      - Accumulate `cum_vol_away`; check `min_dist`
      - Set `los = Short`, compute `entry_signal`, `exit_price`, call `enter()`
    - Reset cumulative accumulators if deviation disappears before entry.
11. **Stop logic** (if `is_pos`):
    - Long path: if `not_sitching` → check `last < bar_low` or `last < exit_price`
      with volume confirmation (`cum_vol * last > 10_000`); update `exit_price`.
    - Short path: symmetric for upside.
    - If `pos_shares == 0` (exit filled): reset all state.

### `enter(symbol)`

Replicates the three-path logic from C++:
- **At lower band**: single order at `last + 0.03`, full `max_size`
- **Below lower band**: single order at `last`, `min(per_trade_risk / last, max_size)`
- **Normal case (long)**: 4-level scaled buy orders at `entry_signal - [1.5, 2.5, 3.5, 4.5] * atr`,
  allocations `[0.40, 0.20, 0.20, 0.20]` of `size * average_entry_price`
- Short mirror: sell orders at `entry_signal + [1.5, 2.5, 3.5, 4.5] * atr`
- After placing: schedule `give_up_timer` at `curr_ts + give_up_secs`

### `on_fill(symbol, order_id, fill_price, fill_size, intent)`

- `EXIT` fill: reset `max_size_remaining`, return.
- `INIT/INCREASE` fill:
  - Update `average_entry_price` (weighted avg across fills).
  - On first fill (`first_fill_flag`): place exit limit order, clear flag.
  - Subsequent fills: cancel any opposing resting orders.

### `on_cancel(symbol, order_id, intent)`

- If `EXIT` cancel and position non-zero: retry `exit()` up to 9 times.
- If > 9 attempts: if `final_close`, terminate symbol; else wait for EOD.

### `on_timer_update_ma` (every 5 min, keyed `"update ma"`)

- If `num_trades != old_num_trades`: call `calc_moving_average(system_time, last_price)`.
- Update `old_num_trades`.

### `on_timer_give_up` (one-shot)

- If no position and pending orders: cancel all orders for symbol.

### `on_timer_end_of_day`

- Cancel all open orders. Place market exit if any position open.

### `on_timer_final_close` (300ms after drawdown hit)

- Force exit at current price.

---

## Backtest Runner

```rust
pub struct BacktestConfig {
    pub start_date: Date,    // 2019-03-18 (LULD permanent adoption)
    pub end_date: Date,
    pub strategy_config: EtfSitchConfig,
    pub initial_capital: f64,           // total buying power
    pub commission_per_share: f64,      // default 0.0 for gross PnL view
}
```

### Execution flow

1. Load `tiingo_eod.parquet` → build daily bar index
2. Load `symbols.parquet` → filter ETF/ETN, apply exclusions → `Vec<Symbol>`
3. For each trading day in `[start_date, end_date]`:
   a. For each symbol: compute `atr`, `prev_close`, `daily_ma[periods]` from daily bars.
      Skip if `< max(moving_average_periods)` bars of history.
      Detect splits: if `abs(gap / prev_close) > 1.0` → skip symbol for the day.
   b. Drive MBO events through strategy (multi-symbol, time-merged stream).
   c. At EOD: collect fills, compute per-symbol PnL, reset per-day state.
4. Write output files.

### MBO stream architecture

- For each symbol × exchange pair: stream `.mbo.dbn.zst` file
- Time-merge across all symbols using a min-heap on `ts_event`
- Filter for `Action::Trade` events only (no quote events needed for v1)
- Deliver to `on_trade(symbol, trade)` in chronological order

---

## Output Schema

### `trades.parquet`

| Column | Type | Description |
|---|---|---|
| `date` | date32 | Trading date |
| `symbol` | string | Ticker |
| `los` | string | `"long"` or `"short"` |
| `entry_time_ns` | i64 | Entry fill timestamp (nanoseconds UTC) |
| `entry_price` | f64 | Volume-weighted average entry price across fills |
| `entry_type` | string | `"scaled"`, `"at_lower"`, `"below_lower"`, `"at_upper"` |
| `exit_time_ns` | i64 | Exit fill timestamp |
| `exit_price` | f64 | Exit fill price |
| `exit_reason` | string | `"limit_hit"`, `"stopped_bar"`, `"stopped_volume"`, `"give_up"`, `"eod"`, `"drawdown"` |
| `shares` | i32 | Total shares (signed: positive = long, negative = short) |
| `gross_pnl` | f64 | `(exit_price - entry_price) * shares` |
| `net_pnl` | f64 | `gross_pnl - commission` |
| `entry_signal` | f64 | The MA value at entry |
| `atr` | f64 | ATR used for sizing |
| `dist` | f64 | Risk distance (entry to band) used for sizing |
| `holding_secs` | f64 | `(exit_time_ns - entry_time_ns) / 1e9` |

### `daily_summary.parquet`

| Column | Type | Description |
|---|---|---|
| `date` | date32 | |
| `n_trades` | i32 | Number of completed round-trips |
| `n_longs` | i32 | |
| `n_shorts` | i32 | |
| `gross_pnl` | f64 | |
| `net_pnl` | f64 | |
| `avg_holding_secs` | f64 | |
| `n_stopped` | i32 | Trades exited via stop logic |
| `n_gave_up` | i32 | Entries cancelled via give_up timer |
| `n_drawdown_exits` | i32 | |

### `run_params.json`

Full copy of `EtfSitchConfig` serialized as JSON. Attached to every run so results
are always paired with the exact params that produced them.

---

## Iterative Research Plan

Once the engine is running, the following variants are worth testing in order:

1. **Baseline**: exact C++ logic port, validate logic on 2023–2024 before expanding
2. **Joint grid: `ma_window_secs` × `pct_atr`** — primary search, run together:
   - `ma_window_secs`: [3, 5, 8, 10, 15, 20, 30] seconds
   - `pct_atr`: [0.10, 0.15, 0.20, 0.25, 0.30, 0.40, 0.50]
   - Expected tradeoff: wider window → smoother signal, fewer trades, larger per-trade
     edge, lower Sharpe. Evaluate on trade count, gross PnL, Sharpe, avg holding time.
   - Note: wider `ma_window_secs` compresses the effective signal magnitude, so
     `pct_atr` needs to be re-calibrated jointly — do not scan them independently.
3. **Entry scaling**: adjust the 4 bid levels and allocations
4. **VWAP filter**: relax or tighten the daily VWAP alignment requirement
5. **Daily MA filter**: test with only `lowest_ma` (drop `highest_ma` confirmation)
6. **Sector filters**: test removing biotech/healthcare exclusion (volatile = more signal)
7. **Volume filter**: test `min_volume` sensitivity on entry confirmation
8. **Exit price**: adjust the `entry_signal - 0.03` target for longs

---

## Key Files to Create

```
research/paratrade/
└── libs/
    └── etf-sitch/
        ├── Cargo.toml
        └── src/
            ├── lib.rs           ← re-exports
            ├── config.rs        ← EtfSitchConfig, param loading
            ├── luld.rs          ← LuldState (reference price + band computation)
            ├── ma.rs            ← intraday MA (CalcMovingAverage port)
            ├── symbol_state.rs  ← EtfSitchSymbolState
            ├── strategy.rs      ← EtfSitchStrategy implements Strategy<M>
            └── backtest.rs      ← BacktestRunner, output writers
```
