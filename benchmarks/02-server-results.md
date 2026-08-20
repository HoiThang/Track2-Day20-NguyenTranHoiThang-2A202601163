# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=24` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 265 | 4.55 | 1300 | 2400 | 2900 | 6.0 | 0.0% |
| 50 | 253 | 4.86 | 7900 | 13000 | 13000 | 41.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.07x** (21% of linear) |
| P95 latency | **5.42x** |
| Effective concurrency at 50 users | 41.8 vs `--parallel 4` slots (occupancy/slot ratio 10.44) |

**Saturated.** Throughput delivered only 1.07x for 5x the offered load, and effective concurrency (41.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.07x while P95 moved 5.42x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server saturates between 10 and 50 users.** The number that convinced me: effective concurrency of 41.8 vs only 4 parallel slots (occupancy/slot ratio = 10.44). This means approximately 37 requests were queued at any given time, waiting for a free slot — not being processed. The evidence is clear: offered load increased 5x (10→50 users) but throughput only increased 1.07x (4.55→4.86 RPS). Meanwhile, P95 latency exploded 5.42x (2400→13000ms). The latency increase is queue time, not compute time — if it were compute-bound, throughput would scale with latency.

To raise goodput@SLO, I would increase `--parallel` first (e.g., to 8 or 16). This directly increases the number of concurrent decode slots without requiring recompilation, model changes, or quantization switches. If my SLO is P95 < 5000ms, at 50 users I have 0% goodput (P95=13000ms > 5000ms). Doubling parallel slots to 8 would let the scheduler pack more concurrent requests, reducing queue time and raising the RPS threshold where saturation begins.
