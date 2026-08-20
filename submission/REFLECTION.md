# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Trần Hội Thắng
**Cohort:** A20-K2
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** 13th Gen Intel Core i9-13980HX
- **Cores:** 24 physical / 32 logical
- **CPU extensions:** AVX2 (CUDA capable machine)
- **RAM:** 31.6 GB
- **Accelerator:** NVIDIA GeForce RTX 4080 Laptop GPU (12282 MiB) + Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Gemma 4 E2B (LAB_MODEL=gemma4-e2b)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Lab chạy hoàn hảo trên máy này. RAM 31.6 GB đủ cho Gemma 4 E2B default. CUDA build của llama.cpp được tự động chọn vì máy có NVIDIA GPU. Không có bước nào fail hay cần workaround.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2535 | 168 / 252 | 6.2 / 6.7 | 556 / 659 / 659 | 161.8 |
| UD-Q2_K_XL | 2.24 | 2458 | 169 / 298 | 5.3 / 5.7 | 500 / 657 / 657 | 188.4 |

**Quan sát** (≤ 60 chữ): UD-Q2_K_XL nhỏ hơn 0.73 GB (24% giảm) và decode nhanh hơn 15.4 tok/s (1.16x). Model chạy trên GPU (ngl=99), nên TTFT không khác nhiều giữa hai quantization. Q2 đáng dùng nếu cần tiết kiệm VRAM hoặc cần throughput cao hơn một chút.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|--:|
| 10 | 4.55 | 1300 | 2400 | 7300 | 7.7 | 0.0% |
| 50 | 4.86 | 7900 | 13000 | 11000 | 41.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.07× (21% of linear)
- **P95 tăng:** 5.42×
- **Effective concurrency ở 50 users:** 41.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà ở mức 50 users. Bằng chứng: effective concurrency 41.6 gấp 10x số slots (4), và throughput chỉ tăng 5x cho 5x offered load. Để tăng goodput@SLO, tôi sẽ tăng `--parallel` trước vì đây là cách nhanh nhất để xử lý queue mà không cần build lại hay đổi model.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | | stub |
| N17 Data pipeline | | stub |
| N18 Lakehouse | | stub |
| N19 Vector + features | | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms (stub — keyword fallback, không có embedding model)
- retrieve: 0.0 ms (stub — vector store không có thật)
- llm: 2824.3 ms (real — llama-server xử lý)
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): LLM là bottleneck rõ ràng (100% latency). Embed và retrieve = 0 vì dùng stub. Trong production thực, embed và retrieve sẽ chiếm phần đáng kể (10-30% total), nhưng ở đây LLM dominate.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Thread count sweep trên GPU-bound workload (ngl=99)

```
before:  -t 1   →  170.4 tok/s
after:   -t 64  →  173.1 tok/s
speedup: 1.02×
best:    -t 64   (still climbing at logical core count)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Đây là kết quả bất ngờ — curve nearly flat và vẫn tăng ở 64 threads, không phải peak ở physical cores (24) rồi drop. Điều này xảy ra vì model chạy hoàn toàn trên GPU (`ngl=99`) với NVIDIA RTX 4080. Với GPU-bound workload, CPU threads không làm matrix multiplication — chúng chỉ làm scheduling và data transfer. Những task này là embarrassingly parallel, nên thêm threads vẫn giúp dù đã oversubscribe. Physical-core "knee" chỉ xuất hiện ở CPU-bound workloads nơi threads competition cho compute resources. Ở đây, spread chỉ 1.04x (166.4→173.1) vì CPU-side tuning có giới hạn khi GPU là bottleneck thật sự.

Kết luận: Thread tuning có ít tác dụng trên GPU-bound workload. Để cải thiện thật sự, cần tối ưu phía GPU: quantization (UD-Q2_K_XL nhanh hơn 1.16x), batch size, hoặc precision.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B5 (C8 semantic cache) — demo offline

**Numbers:**
```
Change: semantic cache demo (offline mode)
Scenario: Cache hit (same prompt) → skip inference
Scenario: Cache miss (paraphrase) → full inference
```

**Điều này nói lên gì mà deck chưa nói:**

Semantic cache hoạt động ở tầng cao hơn KV cache — nó cache theo nghĩa, không theo prefix. Khi một prompt được paraphrase nhưng giữ nguyên ý, semantic cache có thể trả lời ngay mà không cần inference. Tuy nhiên, embedding model dùng để compute similarity cần phải mạnh — nếu dùng pooling mode từ chat model (như lab này), paraphrase thật có thể bị coi là "không liên quan" vì decoder không được train để làm sentence encoder.

Bài học: semantic cache chỉ hiệu quả khi có embedding model chuyên dụng (BGE-M3, Qwen3-Embedding). Trên laptop, việc này đòi hỏi thêm model hoặc dùng API bên ngoài.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Throttle CPU chỉ cho 1.06x speedup trên GPU-bound workload là bất ngờ — tôi kỳ vọng nó nhiều hơn. Điều này dạy rằng khi model offload hoàn toàn sang GPU, CPU chỉ còn vai trò như scheduler, không phải compute engine.

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
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
