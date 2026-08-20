# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 11.1 | 1.00x | 11% |
| 8 | 15.6 | 1.40x | 15% |
| 16 | 19.4 | 1.76x | 19% |
| 24 | 26.5 | 2.39x | 25% |
| 32 | 53.5 | 4.84x | 51% |
| 99 | 105.1 | 9.49x | 100% |

Best: `-ngl 99` at 105.1 tok/s
-- 9.49x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full Offload Analysis:
Full GPU offload (-ngl 99) is overwhelmingly the best configuration on this machine, achieving 105.1 tok/s compared to 11.1 tok/s on CPU-only (-ngl 0) - a massive 9.49x speedup.

Layer Offload Mechanics:
1. Partial Offload Bottleneck: In partial offload configurations (-ngl 8 at 15.6 tok/s, -ngl 16 at 19.4 tok/s, -ngl 24 at 26.5 tok/s, -ngl 32 at 53.5 tok/s), execution ping-pongs between CPU system RAM (DDR4 ~38 GB/s) and GPU VRAM (GDDR6 336 GB/s) across PCIe 3.0 x8/x16 bus on every transformer layer. The latency is dominated by PCIe transfer latency and CPU-side GEMV execution.
2. Full Offload Transition: Gemma 4 E2B has 26 layers. As soon as all 26 layers fit in the 6 GB VRAM (-ngl 32 or -ngl 99), intermediate activations remain entirely in high-bandwidth GDDR6 memory, completely eliminating PCIe memory transfers during the decode loop and unleashing full CUDA core utilization (105.1 tok/s).
