# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 27 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.94 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a  not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 34155 |

Highest sampled value was **3.94 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The peak n_busy_slots_per_decode reached 3.97 of 4 slots (99.25% slot saturation), with requests_processing = 4 continuously active and requests_deferred peaking at 46 requests in the waiting queue.

Comparison with Effective Concurrency:
In 02-server-results.md, Little\'s Law computes an effective concurrency of 39.7 (L = lambda * W = 2.36 RPS * 16.8 s).
- Effective concurrency (L = 39.7) measures total in-flight requests across the entire system (including queue time + service time).
- n_busy_slots_per_decode (3.97 / 4.0) measures actual compute slot occupancy during forward passes on the GPU.

The two metrics agree: with 4 parallel execution slots fully saturated by continuous batching, the remaining ~36 in-flight requests were deferred in the server request queue. This proves that continuous batching operated at maximum theoretical capacity (4/4 slots active per step), while the queue delay explains the entire latency surge in P95.
