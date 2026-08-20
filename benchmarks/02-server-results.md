# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=24` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 129 | 2.34 | 3400 | 6200 | 7300 | 7.7 | 0.0% |
| 50 | 279 | 4.71 | 8800 | 11000 | 11000 | 41.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **2.02x** (40% of linear) |
| P95 latency | **1.77x** |
| Effective concurrency at 50 users | 41.6 vs `--parallel 4` slots (occupancy/slot ratio 10.40) |

**Saturated.** Throughput delivered only 2.02x for 5x the offered load, and effective concurrency (41.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.77x vs 2.02x), so this server still has headroom at 50 users.

## Your reading

**Server saturates between 10 and 50 users.** Evidence: effective concurrency at 50 users is 41.6, which is 10x the --parallel 4 slots. This means 37.6 requests were queued, not being processed. The throughput only scaled 2.02x for 5x offered load, confirming saturation. To raise goodput@SLO, I would increase --parallel first (e.g., to 8 or 16) because it directly increases the number of concurrent decode slots without requiring model changes or recompilation. The next knob would be quantization (Q2 is faster) or GPU offload configuration.
