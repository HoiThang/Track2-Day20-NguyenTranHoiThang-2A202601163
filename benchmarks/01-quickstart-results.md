# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=24` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2874 | 1443 / 2154 | 10.6 / 13.4 | 2080 / 2979 / 2979 | 94.3 |
| UD-Q2_K_XL | 2.24 | 3347 | 1471 / 2197 | 9.1 / 12.7 | 1956 / 2994 / 2994 | 109.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.16x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Yes, UD-Q2_K_XL is worth it on this machine.** UD-Q2_K_XL is 0.73 GB smaller (24% reduction) but decodes 1.16x faster (109.7 vs 94.3 tok/s). Since the model runs on GPU (ngl=99), TTFT is similar between both (1443ms vs 1471ms) because prefill is compute-bound. The TPOT improvement (9.1ms vs 10.6ms) directly translates to throughput gains. I tested asking the same question on both and found output quality acceptable for Q2 on simple prompts; Q4 is noticeably better on complex reasoning tasks.
