# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 139 | 2.39 | 3100 | 4300 | 5200 | 7.6 | 0.0% |
| 50 | 138 | 2.36 | 20000 | 22000 | 22000 | 39.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.99x** (20% of linear) |
| P95 latency | **5.12x** |
| Effective concurrency at 50 users | 39.7 vs `--parallel 4` slots (occupancy/slot ratio 9.93) |

**Saturated.** Throughput delivered only 0.99x for 5x the offered load, and effective concurrency (39.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.99x while P95 moved 5.12x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Saturation Evidence:
The server is completely saturated at 50 users. The definitive evidence is that increasing offered user load by 5x (from 10 to 50 users) yielded 0.99x throughput (flat at 2.39 -> 2.36 RPS), while P95 latency increased 5.12x (from 4,300 ms to 22,000 ms).
According to Little\'s Law, effective concurrency reached 39.7 against --parallel 4 slots (a 9.93x occupancy ratio). The additional ~17.7 seconds of latency per request is pure queue time (requests_deferred = 46), as compute time per token remained constant.

Goodput@SLO & Tuning Strategy:
If our SLO is defined as P95 <= 5.0s, goodput at 10 users is 100% (2.39 RPS), but drops to 0% goodput at 50 users (all requests violated the 5s SLO with 20-22s response times).
To raise goodput@SLO under high concurrency, the first knob to change is increasing --parallel from 4 to 8 or 16 (paired with KV cache quantization q8_0 or q4_0 to conserve VRAM). Expanding parallel slots allows more requests to be batched concurrently in each GPU forward pass, directly raising the throughput ceiling and draining queue backlog.
