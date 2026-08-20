# PLAN — Day 20 Lab: Model Serving & Inference Optimization

## Mục tiêu
Hoàn thành **base track (100 điểm)** + **bonus track (tối đa 20 điểm)** với chiến lược tối ưu điểm.

---

## Phase 0: Setup & Hardware Probe (~20 phút)

### Task 0.1: Kiểm tra phần cứng
```bash
make probe
```
**Output cần có:** `hardware.json` trong repo

**Checkpoint:** Kiểm tra RAM:
- ≥8GB → dùng Gemma 4 E2B (mặc định)
- 4-8GB → `LAB_MODEL=qwen35-0.8b make setup`
- <4GB → Cloud notebook

### Task 0.2: Setup runtime + models (~5-15 phút)
```bash
make setup
```
**Output cần có:** `models/active.json`, `runtime/`, `models/*.gguf`

---

## Phase 1: Base Track (100 điểm)

### Task 1.1: Benchmark baseline — TTFT/TPOT/Percentiles (10 điểm)
```bash
make bench
```
**Rubric đo:**
| # | Được điểm khi |
|---|---|
| 3 | Bảng latency cho **cả hai** quantization, đủ percentile |
| 4 | TTFT và TPOT báo **riêng**, không gộp |
| 5 | Nhận xét về 2-bit vs 4-bit — nhanh hơn + **có đáng không** |

**Output:** `benchmarks/01-quickstart-results.md`

**💡 Chiến lược điểm cao:**
- Đo cả 2 quantization thật sự (UD-Q4_K_XL + UD-Q2_K_XL)
- Nhận xét phải nói: (1) kích thước khác nhau bao nhiêu, (2) tốc độ khác nhau bao nhiêu, (3) **chất lượng** khác nhau thế nào (đã test thật sự chưa?)

### Task 1.2: Tune thread count (10 điểm)
```bash
make tune
```
**Rubric đo:**
| # | Được điểm khi |
|---|---|
| 11 | Before/after thật + giải thích **cơ chế** (10 điểm — NẶNG NHẤT!) |

**Output:** `benchmarks/01-tuning-tg128.md`

**💡 Chiến lược điểm cao:**
- Xác định **knee** (điểm tốt nhất)
- Giải thích **cơ chế**: memory bandwidth? cache residency? vector width?
- Nếu kết quả khác kỳ vọng → **nói rõ**, đó là điểm cộng chứ không phải điểm trừ!

### Task 1.3: Serve + Smoke test (15 điểm)
```bash
# Terminal 1:
make serve

# Terminal 2:
make smoke
```
**Rubric đo:**
| # | Được điểm khi |
|---|---|
| 6 | `/v1/chat/completions` hoạt động |
| 7 | `/metrics` có `tokens_predicted_total` ≠ 0 |

**Screenshot:** `03-serve-and-smoke.png` (cả server + smoke output)

### Task 1.4: Load test (10 điểm)
```bash
# Terminal 1: keep server running
# Terminal 2:
make load-10
make load-50

# Terminal 3 (CHẠY CÙNG LÚC với load-50):
make metrics
```
**Rubric đo:**
| # | Được điểm khi |
|---|---|
| 8 | Cả 10 và 50 users |
| 9 | Continuous batching: peak `n_busy_slots_per_decode` < số slots |

**⚠️ LỖI HAY GẶP:** `make metrics` phải chạy **chồng thời gian** với `load-50`, không phải sau!

**Screenshots:** `04-locust-10.png`, `05-locust-50.png`

### Task 1.5: Load report & saturation analysis (10 điểm)
```bash
make load-report
```
**Rubric đo:**
| # | Được điểm khi |
|---|---|
| 10 | Saturation reading: server bão hoà ở đâu? RPS plateau? P95 tăng bao nhiêu? |

**Output:** `benchmarks/02-server-results.md`

**💡 Chiến lược điểm cao:**
- Xác định **khi nào RPS plateau** (Little's Law: RPS × avg_latency = concurrency)
- Nếu P95 tăng nhanh hơn RPS → phần tăng đó là **queue time**, không phải compute
- Nói rõ knob nào sẽ đổi **trước** nếu phải tăng goodput@SLO

### Task 1.6: RAG pipeline (15 điểm)
```bash
make pipeline
```
**Rubric đo:**
| # | Được điểm khi |
|---|---|
| 12 | Chạy 3 query, in ra context đã retrieve |
| 13 | Khai báo cái nào **real/stub** (N16-N19) + latency chia stage |

**Output:** `benchmarks/03-integration-results.md`

**💡 Chiến lược điểm cao:**
- **Stub không mất điểm** — nhưng nói sai mới mất
- Khai báo rõ: embedding=stub, retrieve=real/hook, llm=real
- Latency split: embed / retrieve / llm

### Task 1.7: Write REFLECTION.md
Mở `submission/REFLECTION.md` và điền **tất cả** section.

**💡 Section quan trọng nhất (§5):**
- 10 điểm — bằng với 1/3 total
- Phải có: before/after numbers + **cơ chế giải thích**
- Không chỉ ghi số — phải nói **tại sao**

### Task 1.8: Verify
```bash
make verify
```
**Phải exit 0** — nếu fail, đọc output để biết còn thiếu gì.

---

## Phase 2: Bonus Track (tối đa 20 điểm)

### Chiến lược chọn bonus

| Máy của bạn | Nên làm | Lý do |
|---|---|---|
| CPU-only | **B1** (build từ source) | Compile với `-DGGML_NATIVE=ON` → đúng CPU extensions → speedup lớn nhất |
| RAM hạn chế | **B2** (sweep-quant) | Đo quantization ladder trực tiếp |
| Có GPU | **B2** (sweep-gpu) | Tìm optimal partial offload point |
| Mọi máy | **B5** (C8 semantic cache) | Không cần tải thêm, chạy được offline |

### Khuyến nghị: B1 + B5 (C8)

**B1: Build llama.cpp từ source**
```bash
make build-llama
make compare-builds
```
**Output:** `benchmarks/bonus-compare-builds.md`
**Lý do:** Máy yếu hưởng lợi nhiều nhất — prebuilt phải dùng baseline chung, bản build cho CPU thật dùng đúng extensions.

**B5: Semantic cache (C8)**
```bash
# Offline demo:
.venv/bin/python bonus/serving-regimes/semantic-cache-demo.py --offline --sweep
```
**Lý do:** Không cần server, không cần tải thêm. Demo logic và phân tích threshold.

### Task 2.1: Build from source (4 điểm)
```bash
make build-llama
```
Build mất 5-15 phút. Cần `cmake`.

**Checkpoint:** `bonus/llama.cpp/llama-server` tồn tại

### Task 2.2: Compare builds (4 điểm)
```bash
make compare-builds
```
**Output:** `benchmarks/bonus-compare-builds.md`

**💡 Điền section "required — replace this line"**

### Task 2.3: Ghi vào REFLECTION §6
```markdown
Change: rebuild llama.cpp with -DGGML_NATIVE=ON
before: <number + units>
after:  <number + units>
speedup: <X.Y>x
Why it worked: <cơ chế — không phải vibes>
```

---

## Phase 3: Screenshots & Submission

### 5 screenshots bắt buộc
| # | File | Từ lệnh |
|---|------|---------|
| 1 | `01-hardware-probe.png` | `make probe` |
| 2 | `02-bench.png` | `make bench` |
| 3 | `03-serve-and-smoke.png` | `make serve` + `make smoke` |
| 4 | `04-locust-10.png` | `make load-10` |
| 5 | `05-locust-50.png` | `make load-50` |

### Submit
1. `make verify` → exit 0
2. Fork/copy repo lên GitHub → **PUBLIC**
3. `git push`
4. Paste URL vào VinUni LMS

---

## Verification Checklist

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` — cả 2 quantization
- [ ] `benchmarks/01-tuning-tg128.md` — with mechanism explanation
- [ ] `benchmarks/02-server-results.md` — saturation analysis
- [ ] `benchmarks/02-server-batching-u50.md` — continuous batching proof
- [ ] `benchmarks/03-integration-results.md` — real/stub declared
- [ ] **Tất cả** "required — replace this line" đã thay
- [ ] `submission/REFLECTION.md` — điền đủ, không placeholder
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] Bonus: `benchmarks/bonus-*.md` + REFLECTION §6
- [ ] `make verify` → exit 0
- [ ] Repo public

---

## Chiến lược điểm tối đa

| Rubric | Điểm | Chiến lược |
|--------|------|------------|
| §5 (single change) | **10** | Tune thread count → giải thích cơ chế đầy đủ |
| §11 (before/after) | **10** | Dùng kết quả `make tune`, phải có mechanism |
| §3,4,5 (benchmark) | **20** | Đo thật + nhận xét chất lượng |
| §8,9,10 (serving) | **20** | Load test cả 10+50, metrics đúng timing |
| §12,13 (integration) | **15** | Khai báo real/stub đúng |
| §6,7 (server) | **15** | Smoke test + metrics |
| §1,2 (setup) | **10** | hardware.json + active.json |
| **Bonus B1-B5** | **+20** | B1 (build) + B5 (semantic cache) |

**Tổng tiềm năng: 120/100 + 20 bonus = 140 điểm**
