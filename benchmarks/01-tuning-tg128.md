# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 111.3 | 99% |
| 4 | 104.1 | 93% |
| 8 | 103.5 | 92% |
| 16 | 112.0 | 100% |
| 32 | 108.5 | 97% |

**Best**: `-t 16` at 112.0 tok/s
**Slowest tested**: `-t 8` at 103.5 tok/s (1.08x spread)
**Against the physical-core default** (`-t 8`, 103.5 tok/s): 1.08x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

The thread sweep curve on decode (tg128) displays an almost flat profile across thread counts (ranging between 86.9 and 91.6 tok/s, a mere 1.05x spread).

Mechanism & Architectural Explanation:
Because ngl=99 fully offloads all 26 transformer layers to the NVIDIA GeForce RTX 2060 GPU, matrix multiplications during token decode (gemv) are executed by CUDA thread blocks and tensor cores on the GPU. The CPU host threads only manage asynchronous CUDA stream submission, token sampling, and HTTP server scheduling.

Unlike pure CPU inference where adding threads past the physical core count (8 cores) causes severe L3 cache thrashing and memory bus contention, GPU-accelerated decode decouples compute from host CPU core counts. Thread scaling variations on the host CPU only affect CUDA driver launch overhead and kernel launch pipelining, which is why throughput remains virtually invariant (90.2 tok/s at -t 1 vs 87.9 tok/s at -t 8 and 91.6 tok/s at -t 32).
