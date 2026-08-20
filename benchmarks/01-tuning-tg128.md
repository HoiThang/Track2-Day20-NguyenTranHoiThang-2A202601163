# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **24 physical · 32 logical** cores · `ngl=99` · metric `tg128`

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

## Your explanation (required -- replace this line)

_Where is the knee, and why there? If the peak sits at your physical core count
and drops above it, say what the extra threads are competing for. If your curve
does something else -- flat, or still climbing at 2x logical cores -- say that
instead and reason about why. A result that contradicts the expected shape is
worth more than one that matches it, as long as you explain it._
