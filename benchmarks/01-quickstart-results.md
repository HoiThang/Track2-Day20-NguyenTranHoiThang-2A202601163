# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=24` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2535 | 168 / 252 | 6.2 / 6.7 | 556 / 659 / 659 | 161.8 |
| UD-Q2_K_XL | 2.24 | 2458 | 169 / 298 | 5.3 / 5.7 | 500 / 657 / 657 | 188.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.16x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Yes, UD-Q2_K_XL is worth it on this machine.** UD-Q2_K_XL is 0.73 GB smaller (24.5% reduction, 2.24 GB vs 2.97 GB) but decodes 1.16x faster (188.4 vs 161.8 tok/s). Since the model runs on GPU (ngl=99), TTFT is nearly identical between both quantizations (168ms vs 169ms for P50) because prefill is compute-bound. The TPOT improvement (5.3ms vs 6.2ms) directly translates to throughput gains. I tested asking the same question on both and found output quality acceptable for Q2 on simple prompts; Q4 is noticeably better on complex reasoning tasks but Q2 is sufficient for most use cases.
