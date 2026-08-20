# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Điền Mạnh Hùng
**Cohort:** A20-K3B
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** macOS 26.6.2 (Darwin 25.6.0, arm64)
- **CPU:** Apple M4 Pro
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** NEON
- **RAM:** 24 GB
- **Accelerator:** Apple Metal
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** gemma-4-E2B-it-UD-Q4_K_XL (primary) + gemma-4-E2B-it-UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi (M4 Pro 24GB)

**Setup story:** Không cần thay đổi gì. `make setup` chạy mượt, tự detect Metal backend. Tải đủ 2 quantization (~5.2 GB) trong ~5 phút qua Hugging Face.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| UD-Q4_K_XL   |      2.97 |      3077 |          84 / 221 |       13.4 / 15.9 |    928 / 1091 / 1091 |           74.3 |
| UD-Q2_K_XL   |      2.24 |      2021 |          82 / 295 |       12.6 / 13.3 |    881 / 1099 / 1099 |           79.1 |

**Quan sát:** Q2_K_XL nhỏ hơn 25% (0.73 GB), load nhanh hơn 34%, decode nhanh hơn 6.5%. TTFT P95 có spike (295 vs 221) nhưng P50 gần như bằng nhau. Đáng dùng trên M4 Pro 24GB.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| ----: | ---: | -------: | -------: | -------: | ---------------: | -------: |
|    10 | 1.81 |     4400 |     6400 |     7500 |              8.1 |     0.0% |
|    50 | 1.78 |    27000 |    29000 |    30000 |             37.1 |     0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.99× (không tăng!)
- **P95 tăng:** 4.53×
- **Effective concurrency ở 50 users:** 37.1 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`:** 3.96 / 4 slots (99%)

**Saturation reading:** Server bão hoà ở mức dưới 50 users. Bằng chứng: RPS không tăng (0.99×) dù load tăng 5×, trong khi P95 tăng 4.53×. Effective concurrency = 37.1 nghĩa là 37 requests đang trong flight nhưng throughput vẫn ~1.78 RPS. Phần latency tăng thêm là queue time, không phải compute — vì batching đã saturate ở 4 slots. Knob đầu tiên cần đổi: `--parallel` (tăng từ 4 lên 8 slots) vì batching là bottleneck, không phải CPU threads.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day                   | Piece                   | Real hay stub?                  |
| --------------------- | ----------------------- | ------------------------------- |
| N16 Cloud/IaC         | llama-server HTTP calls | real                            |
| N17 Data pipeline     | embed()                 | stub (keyword overlap fallback) |
| N18 Lakehouse         | TOY_DOCS                | stub (hardcoded list)           |
| N19 Vector + features | retrieve()              | stub (no vector DB)             |
| N20 Serving           | llama-server            | real                            |

**Latency split** (mean của 3 query):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 566.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection:** Bottleneck là LLM (100% time). Đúng như kỳ vọng — stubbed components không cost gì. Để giảm latency 2×, cần tấn công LLM: dùng Q2_K_XL thay vì Q4, hoặc giảm context length.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** Giảm thread count từ 12 (physical cores) xuống 6

```
before:  79.1 tok/s (at -t 12)
after:   84.1 tok/s (at -t 6)
speedup: 1.06×
```

**Tại sao nó work:**

Knee ở 6 threads, không phải 12 physical cores như kỳ vọng. Lý do: decode phase (tg128) là **memory-bandwidth bound**, không phải compute-bound.

1. **Metal GPU offload:** Với `ngl=99`, model weights load lên GPU. CPU threads chỉ feed prompts và collect results — heavy computation xảy ra trên Metal GPU, không phải CPU.
2. **Memory bandwidth ceiling:** Decode token yêu cầu đọc KV cache + weights từ unified memory. Với 6 threads, bandwidth được saturate nhưng không oversubscribe. 12 threads gây contention: threads compete cho memory bandwidth thay vì compute.
3. **Cache locality:** 6 threads fit tốt hơn trong L2 cache. 12 threads gây eviction overhead.
4. **Coordination overhead:** Metal GPU chạy async. Nhiều threads = nhiều synchronization overhead khi coordinate với GPU.

**Kết quả ngược với "peak at physical core count" heuristic:** Đó là rule cho compute-bound workloads. Decode là memory-bound, nên optimal thread < physical cores.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

**Đã làm:** B1 (build-llama + compare-builds)

**Numbers:**

```
before:  31.0 tok/s (prebuilt release, -ngl 0)
after:   32.5 tok/s (source build -DGGML_NATIVE=ON, -ngl 0)
speedup: 1.05x
```

**Điều này nói lên gì mà deck chưa nói:**

1. **Compiler flag difference is modest on M4 Pro**: Both prebuilt and source build detect NEON at runtime. M4 Pro's microarchitecture is already highly optimized, so there's less room for `-DGGML_NATIVE=ON` to improve.

2. **Memory bandwidth is the real ceiling**: The tg128 decode benchmark is memory-bandwidth bound. Compiler optimizations cannot overcome the bandwidth ceiling — the 1.05x improvement is within measurement variance.

3. **GPU offload >> compiler flags**: The comparison also showed `-ngl 99` (Metal GPU offload) gives **2.32x speedup** (32.5 → 75.3 tok/s). On Apple Silicon, Metal offload is the dominant optimization, not compiler flags.

**Conclusion for M4 Pro users:** Focus on GPU offload (`-ngl`) and thread tuning (`-t`) rather than recompiling. The compiler benefit is real but small (~5%) on modern optimized CPUs.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Thread count tối ưu là 6, không phải 12 — ngược với "dùng hết cores" intuition. M4 Pro với Metal offload có kiến trúc khác laptop CPU thông thường.

---

## 8. Self-check trước khi push

- [X] `hardware.json` committed
- [X] `models/active.json` committed
- [X] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [X] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [X] `benchmarks/02-server-results.md` committed (`make load-report`)
- [X] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [X] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [X] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [X] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
  đã được thay bằng nhận xét của bạn
- [X] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [X] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)
