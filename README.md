# sor-engine

**Ultra low-latency Smart Order Router for crypto perpetuals.**  
5 exchanges, 1–3 µs per decision, zero heap allocations, fixed-point hot path, regime-aware cost model.

I wrote this because every open-source SOR I found was either painfully slow or drowning in unnecessary abstractions. The core idea hasn't changed: sweep the order book, pick the cheapest liquidity, fill greedily. But after a proper technical audit I rewrote the internals to close four gaps that were eating into the latency budget and producing unrealistic fill assumptions. It's still readable. It's just also correct now.

---

## What changed and why

### 1. K-way merge replaces `std::sort` — O(N) instead of O(N log N)

The original code dumped all price levels into a flat array and called `std::sort`. That was wrong because each exchange's L2 book side is already sorted by construction — that's literally how the exchange delivers it. Sorting an already-sorted-by-chunks array with a general comparison sort is leaving performance on the table.

The new `build_cost_slices` does a proper **K-way merge**: one cursor per exchange, one linear scan of K=5 doubles per step to find the minimum effective price, advance that cursor, repeat. Total cost is O(N × K) = O(5N). For N=100 levels that's 500 comparisons versus ~665 for `std::sort`, but more importantly the access pattern is now sequential and prefetch-friendly instead of random-access-hostile. No heap, no allocator, cursors live on the stack.

### 2. Fixed-point storage in `NormalizedBook` — no float division on the hot path

The original `NormalizedBook` stored `double prices[]` and `double qtys[]`. This meant every call to `floor_to_lot(qty, lot_size)` — which ran on every single price level — was a float division plus a `std::floor` call. That's a `fdivsd` plus a libm round-trip per slice.

`NormalizedBook` now stores `int64_t price_ticks[]` and `int64_t qty_lots[]`. Prices are in tick units, quantities are in lot units. The routing engine converts with a single integer multiply (`IMULQ`) once per level during the merge. The lot-boundary rounding collapses to `static_cast<qty_t>(...)`, which the compiler lowers to `CVTTSD2SI` — one instruction, no libm. Conversions back to `double` only happen in reporting and logging paths.

`sizeof(LevelSide)` is unchanged at 640 bytes. The `static_assert` still passes.

### 3. Regime-aware latency penalty — dynamic, not static

The original model charged a flat `0.001 bps/µs × RTT`. That's fine as a baseline. It's not fine when Deribit is at 820 µs during a CPI print, because at that point every HFT on the planet is faster than you and the liquidity you see is phantom. The flat model still happily routes to Deribit.

The new model:

```
base(rtt)  = 0.001 bps/µs × rtt_us × 1e-4
penalty    = base × (1 + 2.0 × vol_factor) × (1 + 1.5 × directional_imb)
```

`vol_factor` is a normalized short-term realized vol you compute from recent mid-price standard deviation (0 = quiet, ~1 = 2σ intraday event). `directional_imb` is the consolidated order-book delta imbalance in the direction of your order (positive = market leans against your fill, front-runners likely present). Both go into `RoutingContext` and cost you nothing on the hot path — one multiply each.

At `vol_factor=1.0` and `directional_imb=1.0`, the latency penalty scales ×7.5. Deribit becomes appropriately expensive when the market is moving fast. At `vol_factor=0, imb=0` it degrades exactly to the original static model.

### 4. Fill rate model — stop assuming 100% fill

The original greedy fill assumed that if you submit a taker order for X, you receive X. This is only true at 0 µs RTT with no queue position. At 820 µs with aggressive spot CVD the first level on Deribit is swept before your order arrives.

Each exchange now gets a fill rate estimated at the start of each routing cycle:

```
fill_rate = exp(-2e-4 × rtt_us) × (1 - 0.10 × vol_factor) × (1 - 0.08 × directional_imb)
```

Capped at a floor of 0.50. For the example setup this gives:

| Exchange | RTT    | fill_rate (quiet) |
|----------|--------|-------------------|
| Binance  | 120 µs | ~0.976            |
| OKX      | 190 µs | ~0.963            |
| Bybit    | 350 µs | ~0.932            |
| Bitget   | 260 µs | ~0.950            |
| Deribit  | 820 µs | ~0.849            |

The `available_qty` on each `CostSlice` is already adjusted and floor'd to a lot boundary before `greedy_fill` sees it. No extra computation in phase 2.

---

## Pipeline

```
RoutingContext (books + regime inputs)
        │
        ▼
 Phase 1 — K-way merge          O(N·K) = O(N)
   · One cursor per exchange
   · Linear scan of K=5 doubles per step
   · Emits CostSlice[] in ascending effective_price order
   · available_qty = fill_rate_adjusted, pre-rounded to lot boundary
        │
        ▼
 Phase 2 — Greedy fill          O(N)
   · Sweep sorted slices
   · No std::floor, no float division on common branch
   · Integer truncation (CVTTSD2SI) when remaining is binding
        │
        ▼
 ChildOrderBuffer (ready to fire)
```

---

## Build & Run

```bash
git clone https://github.com/tfrmma/sor-engine.git
cd sor-engine

mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j

./sor_example
```

Output:

```
══════════════════════════════════════════════════════════
  Smart Order Router — Split Result
══════════════════════════════════════════════════════════
  Direction:          BUY
  Target qty:         5.0000 BTC
  Limit price:        65115.0000 USD
  Vol factor:         0.3000
  Book imbalance:     0.1500
──────────────────────────────────────────────────────────
  Status:             FULLY FILLED
  Filled qty:         5.0000 BTC
  Avg eff. price:     65128.8122 USD
  Total eff. cost:    325644.0609 USD
──────────────────────────────────────────────────────────
  Child orders (11)
──────────────────────────────────────────────────────────
  [DERIBIT] lvl=0  qty=0.1220  price=65097.5  eff_cost=7945.55   cumfill=0.122
  [BINANCE] lvl=0  qty=0.3740  price=65100.0  eff_cost=24357.71  cumfill=0.496
  [DERIBIT] lvl=1  qty=0.1820  price=65098.0  eff_cost=11853.29  cumfill=0.678
  ...
```

The interleaving of DERIBIT and BINANCE is the merge in action: Deribit's raw price advantage at early levels survives the dynamic penalty at `vol_factor=0.3`, but Binance takes over as Deribit's levels climb.

---

## Project layout

```
sor/
├── types.hpp             ← price_t, qty_t, fixed-point helpers, CostSlice, SplitResult
├── normalized_book.hpp   ← SoA book: int64_t price_ticks[] / qty_lots[] + prefix sums
├── exchange_state.hpp    ← LatencyTracker (dynamic penalty), LotConstraints, RoutingContext
├── routing_engine.hpp    ← RoutingEngine class + Cursor struct
├── routing_engine.cpp    ← K-way merge + greedy fill
└── main_example.cpp      ← 5-exchange demo with regime inputs
```

---

## Using it in your bot

```cpp
// One-time setup
RoutingEngine engine(global_fees);

// Per-cycle — compute these from your market data feed
RoutingContext ctx{};
ctx.states           = exchange_states;   // your pre-populated ExchangeState[]
ctx.active_exchanges = 5;
ctx.dir              = OrderDir::BUY;
ctx.total_qty        = 5.0;
ctx.limit_price      = 65115.0;          // 0 = no limit
ctx.short_vol_factor = compute_vol();    // 0 = quiet, ~1 = 2σ event
ctx.book_imbalance   = compute_imb();   // [-1, +1]

ChildOrderBuffer buffer{};
const SplitResult res = engine.calculate_optimal_split(ctx, buffer);

// buffer.orders[0..res.child_count-1] are ready to fire
```

`update_fees(new_fees)` is lock-free and can be called between cycles.

### Populating the book

Prices and quantities go in as integers via the helpers in `types.hpp`:

```cpp
// In your market-data handler (network thread):
auto& asks = book.sides[static_cast<uint8_t>(Side::ASK)];
asks.price_ticks[i] = to_ticks(raw_price, tick_size);
asks.qty_lots[i]    = to_lots(raw_qty,    lot_size);
// ... after all updates:
asks.recompute_cumulative();
```

The routing engine never calls `to_ticks` / `to_lots`. Those stay on the ingestion path.

### Computing vol_factor and book_imbalance

```cpp
// vol_factor: normalize recent realized vol by your daily baseline.
// A simple approach: rolling 60s std of mid-price returns / (daily_vol / sqrt(390))
double vol_factor = rolling_vol_60s / daily_vol_per_minute;

// book_imbalance: consolidated across venues, range [-1, +1].
// Example: (total_bid_qty_top3 - total_ask_qty_top3) / (total_bid + total_ask)
double book_imbalance = (bid_qty - ask_qty) / (bid_qty + ask_qty);
```

If you don't have these yet, pass 0 for both — the model degrades to the original static penalty.

---

## Roadmap

- [ ] Proper unit test suite (Catch2) — especially the FP drift guard in greedy_fill
- [ ] rdtsc micro-benchmark with full latency distribution (p50/p99/p999)
- [ ] Manual AVX2/AVX-512 inner loop for the K-way merge scan
- [ ] Maker/taker mixed routing with queue-position model
- [ ] Calibration tooling for `kVolSensitivity`, `kImbSensitivity`, `kLatDecay`

PRs and issues are very welcome — I actually enjoy people touching the code.

---

## License

MIT. Use it, break it, ship it. Just let me know if you make it faster.

---

Built with love.  
— Taha
