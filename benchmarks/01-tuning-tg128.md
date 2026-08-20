# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **24 physical � 32 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 161.6 | 94% |
| 12 | 164.1 | 96% |
| 24 | 171.4 | 100% |
| 32 | 170.8 | 100% |
| 64 | 168.7 | 98% |

**Best**: `-t 24` at 171.4 tok/s
**Slowest tested**: `-t 1` at 161.6 tok/s (1.06x spread)
**Against the physical-core default** (`-t 24`, 171.4 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=24 make bench
```

## Your explanation

**The knee is at physical core count (-t 24), then flat/slight drop.** The curve is nearly flat (161.6 to 171.4 tok/s = 1.06x spread) because this workload is GPU-bound, not CPU-bound. With ngl=99, all model layers run on NVIDIA RTX 4080, so CPU threads mainly handle scheduling and small memory copies. The 1.06x spread is small but consistent: more threads help with prefill batching and KV cache management slightly. At -t 32 and -t 64, throughput drops slightly (170.8 and 168.7 tok/s) due to oversubscription overhead. Physical core count (-t 24) is optimal because it matches the CPU's natural parallelism without oversubscription.
