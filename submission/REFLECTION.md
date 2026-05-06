# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** _<Điền họ tên của bạn>_
**Cohort:** _<Điền cohort của bạn>_
**Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Windows (AMD64)_
- **CPU:** _unknown (theo detect-hardware.py)_
- **Cores:** _8 physical / 8 logical_
- **CPU extensions:** _AVX2 (theo llama runtime log)_
- **RAM:** _detect-hardware.py báo 0.0 GB (cần xác minh lại script probe trên máy)_
- **Accelerator:** _CPU only_
- **llama.cpp backend đã chọn:** _CPU_
- **Recommended model tier:** _TinyLlama-1.1B (Q4_K_M)_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Máy Windows không có GPU backend phù hợp nên chạy CPU-only path. Lúc setup gặp lỗi `llama-cpp-python` vì pip bị `PIP_NO_INDEX=1`; sau khi bỏ no-index và dùng prebuilt wheel + server extras thì benchmark/server chạy được.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model                             | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
| --------------------------------- | --------: | ----------------: | ----------------: | -------------------: | ------------------: |
| qwen2.5-1.5b-instruct-q4_k_m.gguf |      1707 |        287 / 1047 |      78.1 / 105.6 |   4949 / 6341 / 6359 |                12.8 |
| qwen2.5-1.5b-instruct-q2_k.gguf   |       807 |         400 / 538 |       57.8 / 90.2 |   4060 / 6046 / 6718 |                17.3 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q2_K nhanh hơn rõ ở decode rate và load time, nhưng Q4_K_M ổn định hơn về chất lượng câu trả lời. Với máy hiện tại, Q4_K_M phù hợp cho serving demo; Q2_K hợp khi cần giảm latency/ràng buộc tài nguyên.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
| ----------: | --------: | ------------: | -----------: | -----------: | -------: |
|          10 |      0.21 |         27000 |        42000 |        42000 |        0 |
|          50 |      0.30 |         30000 |        54000 |        54000 |        0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _<0.XX>_, nghĩa là …

Run `record-metrics.py` đã ghi được file `benchmarks/02-server-metrics.csv`, nhưng để kết luận đúng cho concurrency 50 cần chạy lại locust `-u 50 -t 1m` khi server ổn định ở đúng endpoint, rồi lấy peak `llamacpp:kv_cache_usage_ratio` từ cùng phiên chạy đó.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only_
- **N17 (Data pipeline):** _stub: in-memory flow_
- **N18 (Lakehouse):** _stub: not connected (demo pipeline only)_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS keyword retrieval_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _0 ms (stub, không chạy embed riêng)_
- retrieve: _~0.1 ms_
- llama-server: _~12635–17652 ms (3 query demo)_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Điểm nghẽn nằm ở bước gọi LLM (llama-server) trên CPU; retrieve gần như không đáng kể vì chỉ là TOY_DOCS in-memory. Kết quả này khớp kỳ vọng: inference chiếm phần lớn tổng latency.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Chưa thực hiện bonus build/sweep (thiếu toolchain CMake/MSVC tại thời điểm chạy core)._

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: N/A
after:  N/A
speedup: N/A
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Tập trung hoàn thành core trước. Bonus sẽ làm sau khi cài đủ CMake + Build Tools để build `llama.cpp` source và chạy `thread-sweep.py`.

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Điểm gây bất ngờ là cùng máy CPU-only vẫn chạy được full flow benchmark + server + pipeline, nhưng throughput giảm mạnh khi concurrency tăng, nên rất cần theo dõi goodput/SLO thay vì chỉ throughput danh nghĩa.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
