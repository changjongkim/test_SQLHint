# NVMe Benchmark: Final Comprehensive Analysis with Actual Hints

## 🚀 Performance Summary

We have evaluated 3 hint strategies against the MySQL default optimizer on high-performance **NVMe storage**.

- **Total Queries**: 113 (Join Order Benchmark)
- **Improved Queries**: **94 queries (83.2%)**
- **Max Speedup**: **18.5x** (Query 19c)
- **Time Saving**: The longest query (141s) was optimized to **40s**.

## 🏆 The Champions (Best Strategies)

| Strategy | Hint Type | Win Rate | Characteristics | Best Case |
|---|---|---|---|---|
| **Hint 01** | Optimizer Switch | **75%** | **Most Reliable**. Best for CPU-bound ops on NVMe. | 9.3x (Q3a) |
| **Hint 04** | Join Order | 3%* | **Game Changer**. Fixes catastrophic plans. | **18.5x** (Q19c) |
| **Hint 02** | Table Hints | 12% | **Specialist**. Good for specific complex joins. | 6.6x (Q18b) |

## 🔑 Hint Strategy Mapping

| Hint Name | Actual Hint / Strategy | Effect |
|---|---|---|
| `optimizer_1` | `index_condition_pushdown=off` | Disables ICP, sometimes helps when ICP overhead > gain. |
| `optimizer_2` | `mrr=on` | Enables Multi-Range Read (better I/O batching). |
| `optimizer_3` | `block_nested_loop=off` | **The Star**: Disables BNL, forcing Hash Join (usually faster). |
| `optimizer_4` | `batched_key_access=on` | Enables BKA, good for varying I/O patterns. |

## 📈 Top 20 Speedups & Actual Hints Used

| Rank | Query | Baseline | Best Hint | Speedup | Strategy | Actual Hint Used |
|---|---|---|---|---|---|---|
| 🥇 | **19c** | 29.72s | **1.60s** | **18.53x** | Hint 04 | `JOIN_ORDER(n, ci, t, mc, cn)` (Fixed Join Order) |
| 🥈 | **3a** | 4.40s | **0.47s** | **9.34x** | Hint 01 | `batched_key_access=on` |
| 🥉 | **3c** | 7.29s | **0.93s** | **7.84x** | Hint 01 | `block_nested_loop=off` (Force Hash Join) |
| 4 | **24a** | 52.69s | **7.12s** | **7.40x** | Hint 04 | `JOIN_ORDER(rt, ci, t, mk, k, mc, cn)` |
| 5 | **18b** | 8.06s | **1.21s** | **6.64x** | Hint 02 | `NO_ICP(ci)` (Disable ICP for table ci) |
| 6 | **16a** | 3.73s | 0.60s | 6.19x | Hint 01 | `block_nested_loop=off` |
| 7 | **16b** | 139.33s | 23.08s | 6.04x | Hint 01 | `batched_key_access=on` |
| 8 | **21a** | 0.60s | 0.10s | 5.95x | Hint 01 | `batched_key_access=on` |
| 9 | **26a** | 11.93s | 2.14s | 5.57x | Hint 01 | `batched_key_access=on` |
| 10 | **20a** | 26.77s | 5.86s | 4.57x | Hint 01 | `batched_key_access=on` |
| 11 | **27a** | 0.37s | 0.09s | 4.19x | Hint 01 | `block_nested_loop=off` |
| 12 | **22a** | 51.26s | 13.24s | 3.87x | Hint 01 | `mrr=on` |
| 13 | **31a** | 2.14s | 0.56s | 3.79x | Hint 01 | `mrr=on` |
| 14 | **12c** | 53.46s | 14.11s | 3.79x | Hint 01 | `batched_key_access=on` |
| 15 | **14a** | 13.61s | 3.65s | 3.73x | Hint 01 | `block_nested_loop=off` |
| 16 | **18c** | 141.42s | 40.69s | 3.48x | Hint 01 | `batched_key_access=on` |
| 17 | **17b** | 24.43s | 7.33s | 3.33x | Hint 01 | `block_nested_loop=off` |
| 18 | **6d** | 24.56s | 7.75s | 3.17x | Hint 01 | `batched_key_access=on` |
| 19 | **4c** | 0.74s | 0.23s | 3.16x | Hint 01 | `batched_key_access=on` |
| 20 | **25c** | 103.99s | 33.15s | 3.14x | Hint 01 | `batched_key_access=on` |

## 🐢 Transforming the Slowest Queries

| Query | Baseline (Worst 5) | Optimized Time | Speedup | Winner | Actual Hint |
|---|---|---|---|---|---|
| **18c** | 141.42s | **40.69s** | **3.5x** | Hint 01 | `batched_key_access=on` |
| **16b** | 139.33s | **23.08s** | **6.0x** | Hint 01 | `batched_key_access=on` |
| **10c** | 128.13s | **103.70s** | 1.2x | Hint 02 | `NO_ICP(ci)` |
| **25c** | 103.99s | **33.15s** | **3.1x** | Hint 01 | `batched_key_access=on` |
| **25a** | 86.52s | **29.68s** | **2.9x** | Hint 01 | `batched_key_access=on` |

## 🔬 Key Insights for VLDB

1.  **Batched Key Access (BKA) is King**: `batched_key_access=on` (`optimizer_4`) appears most frequently in the heavy hitters (Q16b, Q18c, Q25c), significantly reducing random access overhead even on NVMe.
2.  **BNL Disable (Hash Join)**: `block_nested_loop=off` (`optimizer_3`) is effective for smaller, faster queries (Q3c, Q16a).
3.  **Join Order fixes disasters**: Q19c's 18x speedup confirms that sometimes the optimizer chooses a fundamentally wrong join order, which only `JOIN_ORDER` hint can fix.
