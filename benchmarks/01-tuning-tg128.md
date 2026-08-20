# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **24 physical � 32 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 170.4 | 98% |
| 12 | 166.4 | 96% |
| 24 | 169.9 | 98% |
| 32 | 172.0 | 99% |
| 64 | 173.1 | 100% |

**Best**: `-t 64` at 173.1 tok/s
**Slowest tested**: `-t 12` at 166.4 tok/s (1.04x spread)
**Against the physical-core default** (`-t 24`, 169.9 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=64 make bench
```

## Your explanation

**The curve is nearly flat (1.04x spread) and still climbing at 64 threads.** This contradicts the expected "peak at physical cores, drop after" pattern. The peak sits at `-t 64` (logical cores), not at `-t 24` (physical cores), and the curve does not drop at oversubscription.

**Why this happens:** With `ngl=99`, all model layers run on GPU (NVIDIA RTX 4080), so the actual compute is GPU-bound. CPU threads are not doing matrix multiplication — they are mainly doing scheduling, KV cache management, and data transfer between CPU and GPU. These tasks are more embarrassingly parallel than compute, so extra threads continue to help even past the physical core count. The curve may still be climbing at 64 because the scheduling overhead is not saturated yet on this workload.

**Key insight:** On a GPU-bound workload, thread tuning on the CPU side has diminishing returns. The physical-core "knee" pattern is for CPU-bound workloads where threads compete for compute resources. Here, threads compete only for scheduling bandwidth, which is much lighter. The 1.02x improvement over the default (-t 24) is small but consistent.
