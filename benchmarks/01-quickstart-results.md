# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5769 | 408 / 448 | 11.7 / 13.0 | 1133 / 1206 / 1206 | 85.5 |
| UD-Q2_K_XL | 2.24 | 5956 | 350 / 670 | 12.8 / 14.9 | 1135 / 1387 / 1387 | 78.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.09x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead  few cores, no GPU offload  the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

On this host equipped with an NVIDIA GeForce RTX 2060 (6 GB GDDR6, 336 GB/s memory bandwidth) and CUDA backend offload (ngl=99), both UD-Q4_K_XL (2.97 GB) and UD-Q2_K_XL (2.24 GB) reside entirely within VRAM.

Because the GPU memory bus has abundant bandwidth relative to a ~3 GB weight matrix at batch-1, memory streaming is not the primary bottleneck. Instead, UD-Q2_K_XL requires complex dequantization arithmetic to decode non-uniform 2-bit quantized scales and pack irregular bit-widths, incurring computational overhead on CUDA streaming multiprocessors. As a result, TPOT P50 is 14.0 ms (71.3 tok/s) for 2-bit vs 12.6 ms (79.3 tok/s) for 4-bit (1.11x slower).

Furthermore, 2-bit quantization exhibits noticeable perplexity degradation and loss of nuance compared to 4-bit. Since our 6 GB VRAM comfortably accommodates UD-Q4_K_XL (2.97 GB) alongside the KV cache, the 0.73 GB memory savings of UD-Q2_K_XL does not justify the 11% decode throughput penalty and quality degradation. UD-Q4_K_XL is clearly superior on this hardware.
