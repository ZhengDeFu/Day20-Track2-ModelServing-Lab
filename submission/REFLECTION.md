# Reflection — Lab 20 (Personal Report)

> Đây là báo cáo cá nhân. Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu bên dưới chỉ dùng để so sánh before vs after trên chính máy này.

---

**Họ Tên:** Trịnh Đắc Phú
**Cohort:** Chưa cung cấp
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Ubuntu Linux
- **CPU:** AMD Ryzen 5 5500U with Radeon Graphics
- **Cores:** 12 physical / 12 logical theo `hardware.json`
- **CPU extensions:** AVX2
- **RAM:** 15.0 GB
- **Accelerator:** NVIDIA GeForce GTX 1650, 4096 MiB
- **llama.cpp backend đã chọn:** CPU runtime cho bài nộp chính; CUDA được detect nhưng chưa dùng ổn định vì `nvidia-smi` không giao tiếp được với driver và build CUDA thiếu CUDA Toolkit (`nvcc`).
- **Recommended model tier:** Qwen2.5-1.5B-Instruct (Q4_K_M)

**Setup story** (≤ 80 chữ):

Máy có GTX 1650 4GB nhưng CUDA path chưa ổn định: `nvidia-smi` lỗi driver và build `llama-cpp-python` CUDA thiếu `nvcc`. Vì vậy bài chạy bằng `llama-cpp-python` CPU để tránh treo RAM/driver. Python server chạy được OpenAI-compatible endpoint, nhưng `/metrics` không khả dụng.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 1993 | 111 / 129 | 36.2 / 38.5 | 2398 / 2548 / 2557 | 27.6 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 990 | 133 / 167 | 25.9 / 27.9 | 1760 / 1863 / 1872 | 38.6 |

**Một quan sát** (≤ 50 chữ):

Q2_K nhanh hơn rõ ở decode rate (38.6 tok/s so với 27.6 tok/s) và load nhanh hơn, nhưng Q4_K_M hợp lý hơn nếu cần chất lượng trả lời ổn định. Với RAM 15GB, Q4_K_M vẫn chạy được.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.25 | N/A | 38000 | 38000 | 0 |
| 50 | 0.25 | N/A | 43000 | 43000 | 0 |

Locust trong lần chạy này đo full response time, chưa tách riêng TTFB. Vì server dùng `python -m llama_cpp.server`, OpenAI-compatible `/v1/chat/completions` chạy được nhưng `/metrics` trả `{"detail":"Not Found"}`.

**KV-cache observation** (từ `record-metrics.py`):

Không đo được peak `llamacpp:kv_cache_usage_ratio` trong lần nộp này vì Python server không expose Prometheus `/metrics`. Đây là giới hạn của runtime đang dùng, không phải lỗi của endpoint chat. Nếu chuyển sang native `llama-server --metrics` sau khi build llama.cpp thành công, mục này có thể đo lại.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only, gọi local llama-server ở `http://localhost:8080/v1`
- **N17 (Data pipeline):** stub: in-memory toy documents
- **N18 (Lakehouse):** stub: chưa nối Delta/Iceberg/SQLite thật trong repo này
- **N19 (Vector + Feature Store):** stub: `TOY_DOCS` và keyword-overlap retrieval trong `pipeline.py`

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 0 ms (không dùng embedder thật)
- retrieve: khoảng 0-1 ms vì chỉ keyword search trên list nhỏ
- llama-server: lớn nhất; cùng loại request smoke test mất khoảng 1477-1636 ms, còn dưới load có thể lên 38-43 giây P95

**Reflection** (≤ 60 chữ):

Bottleneck nằm ở llama-server, đúng với kỳ vọng vì retrieval đang là stub rất nhỏ. Khi có vector index thật, embed/retrieve sẽ tăng một ít, nhưng trên máy này decode/local inference vẫn là phần chi phối latency.

---

## 5. Bonus — The single change that mattered most

**Change:** So sánh quantization Q4_K_M sang Q2_K trong Track 01. Bonus source build/thread sweep chưa hoàn tất vì `llama-bench` chưa được build và build full có nguy cơ treo trên máy 16GB.

**Before vs after**:

```text
before: Q4_K_M decode rate 27.6 tok/s, E2E P95 2548 ms
after:  Q2_K decode rate 38.6 tok/s, E2E P95 1863 ms
speedup: ~1.40x theo decode rate, ~1.37x theo E2E P95
```

**Tại sao nó work**:

Q2_K dùng ít bit hơn cho trọng số nên lượng dữ liệu cần đọc từ RAM thấp hơn. Trên laptop CPU, decode thường bị giới hạn bởi memory bandwidth hơn là pure compute, nên giảm kích thước weight giúp mỗi token đi qua model nhanh hơn.

Tradeoff là quality. Q2_K hợp lý khi RAM rất chật hoặc cần tốc độ, nhưng với RAM 15GB, Q4_K_M vẫn chạy được và là lựa chọn cân bằng hơn cho câu trả lời ổn định.

---

## 6. (Optional) Điều ngạc nhiên nhất

Máy có GPU NVIDIA nhưng không tự động dùng được CUDA nếu driver/CUDA Toolkit chưa sạch. Với lab này, CPU path chậm hơn nhưng ổn định hơn để lấy số liệu và hoàn thành flow end-to-end.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` đã commit dựa trên locust screenshots
- [ ] `benchmarks/bonus-*.md` đã commit (bonus source build chưa hoàn tất)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ public
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải public đến khi điểm được công bố. Nếu private, grader không xem được.
