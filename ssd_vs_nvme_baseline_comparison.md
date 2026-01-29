# Baseline Comparison: Original SSD vs NVMe

## 📉 Impact of Storage Hardware on Query Performance

We compared the execution times of the Join Order Benchmark (113 queries) on **SATA SSD** vs. **NVMe SSD** to quantify the performance gap.

- **Objective**: To establish a baseline for evaluating whether "storage-aware" hint strategies (like Table Hints) are more effective on slower storage.
- **Data Source**:
  - SSD: `original_baseline_ssd_progress.txt`
  - NVMe: `baseline_progress.txt` (from previous run)

## 🚀 Heavy I/O Queries: NVMe Dominance

For queries with significant I/O requirements, NVMe provides a **1.7x to 2.2x** speedup over SSD. This confirms that these queries are indeed **I/O bound** on the SSD.

| Query | SSD Time (s) | NVMe Time (s) | Speedup (NVMe/SSD) | Status |
|---|---|---|---|---|
| **22d** | 118.18 | **52.58** | **2.25x** | 🚀 NVMe Win |
| **8c** | 71.75 | **33.85** | **2.12x** | 🚀 NVMe Win |
| **22c** | 138.52 | **69.08** | **2.01x** | 🚀 NVMe Win |
| **25c** | 194.59 | **103.99** | **1.87x** | 🚀 NVMe Win |
| **18c** | 242.65 | **141.42** | **1.72x** | 🚀 NVMe Win |
| **25a** | 134.87 | **86.53** | **1.56x** | 🚀 NVMe Win |

> **Insight**: These are the prime candidates for **Table Hints** on SSD. Since I/O is the bottleneck, hints that reduce random access (e.g., by forcing different join orders or indexes) should yield higher gains on SSD than on NVMe.

## 🐢 Anomalies: Where SSD was Faster?

Surprisingly, for some queries, the SSD baseline recorded faster times. This might be due to run-to-run variance, caching differences, or specific plan choices.

| Query | SSD Time (s) | NVMe Time (s) | Speedup | Note |
|---|---|---|---|---|
| **16b** | 108.55 | 139.33 | 0.78x | NVMe was slower (Plan regression?) |
| **10c** | 93.05 | 128.13 | 0.73x | NVMe was slower |

## 📊 Overall Statistics

- **Total Queries Compared**: 113
- **NVMe Faster**: 53 queries (46.9%)
- **SSD Faster/Similar**: 60 queries (53.1%) - mostly short-running CPU-bound queries.
- **Average Speedup (Heavy Queries)**: ~1.9x

## 📝 Conclusion

The **~2x performance gap** for heavy queries confirms that the SSD environment is significantly more I/O constrained. This sets the perfect stage to test the hypothesis: **"Complex physical optimizations (Table Hints) are more valuable on slower storage."**

We expect Hint 02 (Table Hints) to show a higher win rate and impact on SSD compared to its 15% win rate on NVMe.
