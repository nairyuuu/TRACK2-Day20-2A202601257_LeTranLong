# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lê Trần Long
**Cohort:** 2A202601257
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** AMD Ryzen 7 5700U with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 / FMA
- **RAM:** 22.8 GB
- **Accelerator:** NVIDIA GeForce RTX 2060 (6144 MiB VRAM, CUDA 12.4 + Vulkan)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Ban đầu PowerShell trên Windows 11 gặp lỗi charmap codec do stdout mặc định là cp1252 khi in box-drawing Unicode; giải quyết bằng cách export `PYTHONIOENCODING=utf-8`. Tải runtime llama.cpp b10488 kèm CUDA DLLs và tải thành công cả 2 bản quant Gemma 4 E2B (~5.2 GB).

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 147733 | 404 / 61945 | 12.6 / 13.9 | 1180 / 62821 / 62821 | 79.3 |
| UD-Q2_K_XL | 2.24 | 4509 | 388 / 133923 | 14.0 / 16.1 | 1260 / 134794 / 134794 | 71.3 |

**Quan sát** (≤ 60 chữ): Bản 2-bit decodes chậm hơn 4-bit 1.11x (71.3 vs 79.3 tok/s) dù nhẹ hơn 0.73 GB do chi phí dequantize bit lẻ trên CUDA cores. Thử nghiệm thực tế câu trả lời của 2-bit bị mất ngữ nghĩa rõ rệt, trong khi 4-bit vừa nhanh hơn vừa giữ trọn vẹn chất lượng. Không nên dùng 2-bit trên GPU có đủ VRAM.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.39 | 3100 | 4300 | 5200 | 7.6 | 0.0% |
| 50 | 2.36 | 20000 | 22000 | 22000 | 39.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.99x (20% of linear)
- **P95 tăng:** 5.12x
- **Effective concurrency ở 50 users:** 39.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà hoàn toàn ở 50 users vì throughput đi ngang (2.39 -> 2.36 RPS) trong khi P95 phồng 5.12x (4.3s -> 22.0s). Bằng chứng Little's Law cho thấy concurrency = 39.7 vượt xa 4 compute slots; 17.7s tăng thêm là queue time (deferred = 46). Để nâng goodput@SLO, tôi sẽ tăng `--parallel` lên 8/16 kết hợp quantize KV cache (q8_0) để batch thêm request trên GPU.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local workstation environment | stub |
| N17 Data pipeline | In-memory document list | stub |
| N18 Lakehouse | In-memory toy corpus | stub |
| N19 Vector + features | Keyword overlap lexical search | stub |
| N20 Serving | `llama-server` (Gemma 4 E2B on RTX 2060) | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 621.5 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): LLM generation là bottleneck áp đảo (100% latency). Hoàn toàn khớp kỳ vọng vì toy corpus rất nhỏ trong khi auto-regressive decode cần 20+ forward passes. Để giảm 2x latency pipeline, tôi sẽ áp dụng Speculative Decoding (MTP) cho Gemma 4 và Prefix Caching cho system/context prompt.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Full GPU layer offload to RTX 2060 (`-ngl 99` vs `-ngl 0` CPU baseline)

```
before:  11.1 tok/s (CPU-only DDR4 RAM)
after:   105.1 tok/s (NVIDIA RTX 2060 GDDR6 VRAM)
speedup: 9.49x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Inference token generation (decode phase) là bài toán bị nghẽn hoàn toàn bởi **Memory Bandwidth (Memory-Bound)** do Arithmetic Intensity rất thấp ($O(1)$ FLOPs/byte khi batch size = 1). Mỗi bước sinh token buộc hệ thống phải load toàn bộ 2.97 GB trọng số của model qua bus bộ nhớ vào execution units.

Trên CPU Ryzen 7 5700U, bộ nhớ hệ thống DDR4 dual-channel chỉ đạt băng thông thực tế ~38 GB/s chia sẻ giữa 8 cores và các tiến trình OS, dẫn đến tốc độ decode kịch trần ở mức 11.1 tok/s. Khi bật `-ngl 99`, toàn bộ 26 transformer layers được chuyển hoàn toàn vào VRAM của NVIDIA GeForce RTX 2060. Với băng thông bộ nhớ GDDR6 lên tới **336 GB/s** (gấp gần 9x CPU bus) kết hợp cùng 1920 CUDA cores và Tensor Cores chuyên dụng, thời gian streaming trọng số mỗi step giảm mạnh, mang lại bước nhảy vọt **9.49x speedup** thực tế.

Đặc biệt, do GPU RTX 2060 này kết nối qua giao tiếp **PCIe 3.0 x4** (băng thông tối đa chỉ ~3.94 GB/s), việc offload một phần (Partial Offload) sẽ phải chịu chi phí trung chuyển cực lớn qua bus hẹp này. Chỉ khi **Full Offload** (`-ngl 99`), việc giao tiếp qua bus PCIe 3.0 x4 trong decode loop mới bị loại bỏ hoàn toàn, giải phóng toàn bộ băng thông 336 GB/s của GDDR6.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 GPU Offload Sweep (`bonus/sweeps/gpu-offload-sweep.py`) & B5 Semantic Caching (`bonus/serving-regimes/semantic-cache-demo.py`)

**Numbers:**

```
before:  11.1 tok/s (-ngl 0 CPU only)
after:   105.1 tok/s (-ngl 99 GPU full offload)
speedup: 9.49x
```

**Điều này nói lên gì mà deck chưa nói:**

Sweep GPU layer offload từ `-ngl 0` đến `99` làm lộ rõ chi phí "phạt" cực lớn của Partial Offload trên giao tiếp **PCIe 3.0 x4 (~3.94 GB/s)**: `-ngl 8` đạt 15.6 tok/s, `-ngl 16` đạt 19.4 tok/s, `-ngl 24` đạt 26.5 tok/s, `-ngl 32` đạt 53.5 tok/s. Khi offload dở dang, activations và tensor biên giới phải liên tục đồng bộ qua bus PCIe 3.0 x4 chỉ có ~3.94 GB/s (chậm hơn 85 lần so với GDDR6). Chỉ khi đạt mốc Full Offload (khi toàn bộ 26 layers nằm trọn trong 6 GB VRAM), latency trung chuyển PCIe biến mất hoàn toàn và hiệu năng tăng vọt phi tuyến tính lên 105.1 tok/s.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Ảnh hưởng khổng lồ của PCIe 3.0 x4: khi offload dở dang, đường truyền 3.94 GB/s kìm hãm GPU khiến tốc độ tăng rất chậm; nhưng khi toàn bộ 26 layers nằm trọn trong VRAM, GPU decode trực tiếp trên bus GDDR6 336 GB/s giúp tốc độ tăng vọt lên 105.1 tok/s mà không còn bị nghẽn bởi bus PCIe. Đồng thời, bản `UD-Q2_K_XL` (2-bit) lại decode chậm hơn bản 4-bit 11% do chi phí dequantize bit lẻ trên CUDA cores.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo GitHub ở chế độ **public**
- [x] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
