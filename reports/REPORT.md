# Lab 25B — Report

**Họ tên / nhóm:** Đỗ Đức Đại (MSV: 2A202602725)  **Commit:** d28f1b333439cea4fd234e32a4baa420b5e3e509  **Điểm tự động (`make grade`):** 84.0 / 88 (Chưa tính 4đ dữ liệu ẩn của giảng viên)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 s | Tránh nghẽn hàng đợi và kịp failover trước deadline 30s. Trong G6, alpha-eu bị treo, timeout 8s giúp client từ bỏ sớm sau 8s để failover sang beta kịp hoàn tất lượt user (p95 đạt 5.961s < deadline 30s). Nếu để 30s thì lượt user hết hạn ngay tại provider đầu tiên (availability tụt còn 94.88%). |
| `max_attempts_per_provider` | 2 | Tối đa 1 lần retry trên cùng provider (tổng 2 lần gửi). Trong G1/G2, việc giới hạn attempts giúp cắt đứt retry storm, giữ amplification ở mức 1.041 - 1.062 thay vì 7.7 - 9.9 như bản naive. |
| `min_time_left_s` | 1.0 s | Tránh lãng phí request khi thời gian còn lại không đủ chạy một LLM call hoàn chỉnh (median service time là 1.2s - 1.8s). |
| `backoff_base_s` / `cap_s` | 0.25 s / 4.0 s | Jitter giúp phân tán các lượt retry, tránh thundering herd khi provider vừa hết nghẽn. Cap 4s đảm bảo retry đủ nhanh trong cửa sổ deadline 30s. |
| `retry_budget_ratio` | 0.1 | Đảm bảo lượng retry không vượt quá 10% tổng lưu lượng request mới. Trong G3 brownout, giữ retry_ratio ở mức 0.034 (yêu cầu <= 0.10), triệt tiêu hoàn toàn khuếch đại tải. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Đủ mẫu để ước lượng tỷ lệ lỗi tin cậy, tránh ngắt mạch nhầm khi tải thấp vào ban đêm (min_calls=10). |
| `breaker.failure_rate_threshold` | 0.5 (50%) | Ngắt mạch nhanh khi provider gặp sự cố diện rộng. Trong G2 và G6, breaker ngắt sau 5 lỗi liên tiếp hoặc tỷ lệ lỗi >= 50%, bảo vệ provider và chuyển hướng ngay sang alpha-eu / beta. |
| `breaker.slow_call_s` | 5.0 s | Nhận diện provider bị suy thoái dù không trả lỗi HTTP. Trong G4, alpha phản hồi chậm thêm 5s (latency ~6.2s > 5s), kích hoạt trip theo slow call rate giúp hạ p95 từ 13.764s (naive) xuống 5.562s (yours). |
| `breaker.slow_rate_threshold` | 0.6 (60%) | Cho phép một vài call ngoại lệ (p99) chậm mà không ngắt mạch nhầm, chỉ ngắt khi đa số call bị suy thoái hiệu năng. |
| `breaker.open_duration_s` | 10.0 s | Khoảng nghỉ 10s cho provider nghẽn tự giải phóng hàng đợi trước khi gửi probe thăm dò. Trong G2, thời gian này giúp alpha xả bớt queue và đạt recovery_s = 0s ngay khi GPU phục hồi. |
| `breaker.half_open_max_probes` | 2 | Giới hạn tối đa 2 request thăm dò đồng thời ở trạng thái HALF_OPEN, ngăn chặn triệt để thundering herd dồn vào provider vừa tỉnh dậy. |

### Sơ đồ quyết định của `on_error`

```text
                                [Provider Error]
                                       |
                     +-----------------+-----------------+
                     |                                   |
              [CLIENT_ERROR]                      [Other Errors]
              (400/401/422...)                           |
                     |                        +----------+----------+
            +--------+--------+               |                     |
            | Breaker: release|        [RATE_LIMITED]       [QUOTA_EXHAUSTED]
            | Action: GiveUp  |        (429 + retry-after)  (429 insufficient_quota)
            +-----------------+               |                     |
                                      +-------+-------+     +-------+-------+
                                      |Breaker:release|     |Breaker:release|
                                      |Cooldown:      |     |Cooldown: 300s |
                                      | now+retry_after     |Failover now   |
                                      |Failover now   |     +---------------+
                                      +---------------+
                                              |
                           [OVERLOADED / SERVER_ERROR / TIMEOUT]
                                (500, 502, 503, 529, timeout)
                                              |
                                      +-------+-------+
                                      |Breaker:       |
                                      | on_failure    |
                                      +-------+-------+
                                              |
                             tries < max_attempts AND budget token available?
                                             / \
                                       YES  /   \  NO
                                           /     \
                        +-----------------+       +-----------------+
                        | Retry SAME      |       | Failover to     |
                        | with jittered   |       | next provider   |
                        | backoff delay   |       | in ladder (0s)  |
                        +-----------------+       +-----------------+
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   - **Số liệu:** Khi vô hiệu hóa Circuit Breaker (`min_calls: 1_000_000_000`) và tăng `max_attempts: 5`:
     * Có Retry Budget (exp1a): Băng G2 đạt `availability = 0.9987`, `amplification = 1.380`, `retry_ratio = 0.058`, `recovery_s = 0.0s` (PASS). Băng G3 đạt `retry_ratio = 0.068` (PASS).
     * Tắt Retry Budget (exp1b): Băng G2 có `amplification = 1.678` (vượt ngưỡng 1.6), `retry_ratio = 0.494` (tăng gấp 8.5 lần!), `recovery_s = 20.0s` (FAIL). Băng G3 có `retry_ratio = 0.159` (vượt ngưỡng 0.10, FAIL).
   - **Giải thích:** Khi Circuit Breaker bị cấu hình sai hoặc không phản ứng kịp (ví dụ threshold quá cao, cửa sổ trượt quá rộng), hệ thống rất dễ rơi vào retry storm lặp lại. Retry Budget đóng vai trò chốt chặn cuối cùng (token bucket) đảm bảo lượng retry toàn cục không bao giờ vượt quá tỷ lệ an toàn (`ratio: 0.1` tức tối đa ~10% lưu lượng). Dù breaker hoạt động tốt, ta vẫn bắt buộc phải có cả hai vì breaker chỉ bảo vệ từng node đơn lẻ, còn retry budget kiểm soát tổng áp lực tải của toàn bộ fleet client.

2. **`attempt_timeout_s: 30` (Survivorship bias & deadline exhaustion):**
   - **Số liệu:** Chạy G6 với timeout 30s: `availability` sụt từ 1.0000 xuống **0.9488** (FAIL tiêu chí >= 0.95), xuất hiện 109 lượt chết vì hết deadline (`give_up_reasons: {'deadline': 109, 'no_provider_available': 13}`). Trái lại, `latency_p95_s` trên lượt thành công đo được là **5.315s** (trông có vẻ "nhanh hơn" mức 5.961s của client chuẩn). Ở băng G4, availability vẫn giữ 1.0 và p95 đạt 5.608s (gần như không đổi so với 5.562s).
   - **Giải thích:** G4 không đổi vì provider alpha chỉ phản hồi chậm thêm 5s (latency ~6.2s < 30s), và breaker theo slow call (`slow_call_s: 5.0`) đã trip kịp thời để chuyển tải sang alpha-eu. Ở G6, alpha-eu bị treo hoàn toàn: khi timeout = 30s, client kiên nhẫn chờ đến hết hạn 30s mới chịu từ bỏ, khiến toàn bộ turn của user bị tiêu hao hết quỹ thời gian (30s) mà không kịp failover sang beta hay gamma-mini. Con số p95 giảm là ảo do **survivorship bias**: p95 chỉ đo trên các request thành công, những request bị treo 30s đã chết hoàn toàn và bị loại khỏi mẫu tính p95. Timeout đúng phải dựa trên `p99 bình thường + thời gian cho failover` (~8s), tuyệt đối không để bằng deadline của user.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   - **Số liệu:** Khi bỏ điều kiện `supports_tools` trong hàm `ladder("tool")`, chạy G6 ghi nhận: `incompatible_routes = 234`, `availability` tụt từ 1.0000 xuống **0.9019** (90.19%), vi phạm nghiêm trọng cả 2 tiêu chí kiểm tra (`availability_min: False`, `incompatible_routes_max: False`).
   - **Giải thích (Góc nhìn Product Manager):** 234 khách hàng yêu cầu hoàn tiền đã gặp lỗi sụp đổ dịch vụ ngay lập tức vì hệ thống điều phối đưa yêu cầu cần xử lý nghiệp vụ (cần gọi tool kiểm tra đơn hàng/thanh toán) sang cho `gamma-mini` – một model giá rẻ chỉ biết nói chuyện phiếm chứ không có năng lực thực thi thao tác hệ thống. Đối với người dùng, đây không phải là lỗi mạng chậm mà là sản phẩm bị "ngáo ngơ" hoặc tê liệt, trực tiếp phá hủy trải nghiệm hoàn tiền và gây mất uy tín dịch vụ.

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (cents) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700 | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

* **Vì sao chỉ idempotency key là chưa đủ?** Idempotency key bảo vệ hệ thống thanh toán không bị charge/refund 2 lần nếu cùng một thao tác được gọi lại. Tuy nhiên, nếu không có Checkpoint, mỗi khi worker crash, worker mới sẽ phải chạy lại toàn bộ từ bước 1 (`classify_request`, `lookup_order`), gây lãng phí chi phí token LLM cực lớn (`wasted_llm_per_crash = 1.123`) và làm chậm tiến trình xử lý.
* **Vì sao chỉ checkpoint cũng chưa đủ?** Nếu chỉ có checkpoint mà không có idempotency key, khi worker gặp sự cố mạng (mất response - `ServiceTimeout`) hoặc crash ngay sau khi gọi thanh toán nhưng *trước khi kịp lưu checkpoint*, worker mới phục hồi sẽ tưởng bước thanh toán chưa chạy và gọi lại payment API lần nữa, gây hoàn tiền trùng lặp (như bản naive làm mất 1,496,700 cents).
* **Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?** Nếu ghi checkpoint hoàn tiền thành công trước khi thực sự gọi API payment, worker có thể bị crash ngay tại thời điểm đó. Khi worker mới khởi động lại, nó đọc thấy checkpoint đã xong nên bỏ qua bước hoàn tiền và chuyển sang gửi email, dẫn đến **Lost Refund** (khách nhận email báo đã hoàn tiền nhưng thực tế tiền chưa bao giờ được chuyển).

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page | page | 27 min |
| `provider_outage` | page | page | page | 4 min |
| `deploy_blip` | none | **page** (False Page) | none | - |
| `slow_leak` | ticket | none (Bỏ sót) | ticket | 1337 min |
| `degraded_evening` | page | page | page | 291 min |
| `night_spike` | none | **page** (False Page) | none | - |
| `small_bump` | none | **page** (False Page) | none | - |

* **Error budget 30 ngày:**
  * Tổng requests: **25,901,755**
  * Lỗi cho phép (99.5% SLO): **129,508.8**
  * Số lỗi thực tế: **55,618**
  * Availability thực tế: **0.997853** (99.785%)
  * Budget đã tiêu: **42.95%**
  * Budget còn lại: **57.05%**
  * Quyết định freeze: **False** (Không đóng băng release vì budget còn hơn 57%).
* **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?** Sự cố kéo dài 20 phút với 12% lỗi chỉ tiêu tốn khoảng `(20 * rate * 0.12) / Budget` ~ 0.5% ngân sách lỗi của cả tháng. Nó không gây nguy cơ cạn kiệt error budget và chỉ cần được ghi nhận hoặc xử lý trong giờ làm việc bình thường. Đánh thức kỹ sư on-call cho những biến động nhỏ như vậy gây ra "alert fatigue" (mệt mỏi vì báo động), làm giảm sự nhạy bén khi có sự cố nghiêm trọng thực sự.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](/reports/gameday/G2_overload_storm.png)

### Tóm tắt (3 câu)
Vào lúc $t=120\text{s}$, provider chính (`alpha`) bị sụt giảm 75% năng lực GPU trong 180 giây, dẫn đến tỷ lệ lỗi 529 và timeout tăng vọt. Phiên bản `NaiveClient` với cơ chế retry 3 tầng chồng chéo không có jitter, không breaker và không fallback đã tạo ra một trận bão retry storm lên tới 254 attempts/s dồn vào `alpha`. Hệ thống rơi vào trạng thái thất bại siêu ổn định (metastable failure), khiến sự cố kéo dài thêm 155 giây sau khi provider đã hoàn toàn hồi phục năng lực phần cứng.

### Ảnh hưởng
* **Số lượt người dùng thất bại:** 1,285 / 2,362 lượt (Availability sụt giảm nghiêm trọng xuống còn **45.60%**).
* **Thời gian gián đoạn:** 335 giây (từ $t=120$ đến $t=455$).
* **Tiêu hao Error Budget:** Gây ra 1,285 failed turns trong 10 phút, tương đương tiêu thụ hơn **100%** error budget cho phép của cả tháng nếu chỉ xét riêng sự cố này.

### Timeline (Dữ liệu thực tế trích xuất từ `naive_G2_overload_storm_timeseries.csv`)

| $t$ (giây) | Sự kiện | Bằng chứng định lượng (Số liệu CSV) |
|---:|---|---|
| **0 – 119** | Trạng thái bình thường | `attempts_alpha`: 5–14 req/s, `calls_ok`: 6–13 req/s, tỷ lệ thành công 100%. |
| **120** | **Trigger:** alpha mất 75% GPU | `attempts_alpha`: 5 req/s, `calls_ok` lập tức tụt từ 10 xuống 7, rồi xuống 2 ở $t=121$. |
| **122 – 135** | Hàng đợi đầy, bắt đầu tích tụ retry | `calls_ok` rớt về 0–2 req/s; `attempts_alpha` tăng từ 3 lên 24 req/s do các tầng retry kích hoạt. |
| **136 – 144** | **Bùng phát Retry Storm** | `attempts_alpha` bùng nổ phi mã: $t=136$: 53 req/s $\to$ $t=138$: 122 req/s $\to$ $t=140$: 216 req/s $\to$ $t=143$: **254 req/s**. `calls_ok` = 0. |
| **145 – 299** | Quá tải toàn diện & bão hòa | `attempts_alpha` duy trì ở mức cực đoan 80–190 req/s, `calls_ok` duy trì bằng 0 liên tục trong suốt 155s tiếp theo. |
| **300** | **Phần cứng hồi phục:** GPU khôi phục 100% | Năng lực xử lý của alpha trở lại 18 slots, nhưng `calls_ok` vẫn là 0 req/s; `attempts_alpha` vẫn ở mức 115 req/s. |
| **301 – 452** | **Metastable Failure:** Tự duy trì quá tải | Dù alpha đã khỏe, hàng đợi server vẫn bị tắc nghẽn bởi hàng ngàn retry và request mà client đã abandon; `calls_ok` = 0. Đến $t=372$, attempts vẫn dội ngược lên 176 req/s. |
| **455** | Hệ thống bắt đầu thoát khỏi nghẽn | Lượng retry lắng xuống, hàng đợi được xả sạch, `calls_ok` phục hồi lên 4 req/s, và đạt 12 req/s ở $t=461$. |

### Trigger vs Root Cause
* **Trigger:** Sự cố hạ tầng tại nhà cung cấp A khiến cluster `alpha` mất 75% GPU trong 3 phút (từ $t=120$ đến $t=300$).
* **Root Cause:** Thiết kế client v1 (`NaiveClient`) thiếu năng lực bảo vệ phân tán:
  1. Ba tầng retry chồng chéo (App $\times$ Gateway $\times$ SDK = tối đa 36 attempts/call) với fixed delay.
  2. Không có Circuit Breaker để ngắt mạch khi provider quá tải.
  3. Không có Fallback ladder để phân tải sang `alpha-eu` hay `beta`.
  4. Không có Retry Budget để chặn khuếch đại tải.
* **Cơ chế Metastable Failure (Vòng lặp tự duy trì):**
  Server của provider xử lý theo hàng đợi FIFO và vẫn tiếp tục xử lý các request cũ ngay cả khi client đã timeout ở phía trước. Khi client gửi retry ồ ạt, hàng đợi bị lấp đầy bởi các request "rác/vô nghĩa". Khi provider phục hồi ở $t=300$, server phải mất hơn 2.5 phút chỉ để dọn dẹp các request cũ đã bị hủy, trong khi client tiếp tục retry mới, giữ tải luôn cao hơn năng lực phục vụ cho tới tận $t=455$.

### Điều gì giúp / Điều gì làm tệ hơn
* **Làm tệ hơn:** Cơ chế retry mù quáng với delay cố định và không giới hạn số attempt; cố chấp bám lấy provider đang chết thay vì chuyển hướng fallback.
* **Giúp ích:** Provider có cơ chế shed load (529) khi hàng đợi vượt quá 10s giúp ngăn chặn crash vật lý hoàn toàn của máy chủ.

### Action Items

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient` có Circuit Breaker (sliding-window) và token-bucket Retry Budget (`ratio: 0.1`). | **Mitigate** | Core SRE Team | Chạy CI simulation G2: `amplification <= 1.2`, `recovery_s <= 5s`. |
| 2 | Cấu hình Fallback Ladder tự động chuyển sang `alpha-eu` và `beta` khi `alpha` gặp lỗi 529 hoặc ngắt mạch. | **Prevent** | Client Platform | Băng G2 đạt `availability >= 99%`, traffic tự động tái cân bằng khi khôi phục. |
| 3 | Tích hợp cảnh báo Burn-rate đa cửa sổ phát hiện suy thoái hiệu năng trong vòng 5 phút thay vì chỉ alert tĩnh. | **Detect** | Monitoring Team | Test cảnh báo trên dataset sự cố 30 ngày: `false_pages = 0`, phát hiện outage trong < 5 phút. |

---

## 7. Một điểm yếu còn lại

* **Điểm yếu:** Chi phí và áp lực tải khi failover sang provider đắt hơn (`beta`). Provider `beta` có chi phí gọi cao hơn 40% (1.4 so với 1.0) và giới hạn rate limit chỉ 16 rps. Khi toàn bộ cụm `alpha` gặp sự cố diện rộng, việc toàn bộ fleet client đồng loạt đổ dồn về `beta` có thể nhanh chóng làm kiệt quệ rate limit của `beta` và làm tăng đột biến chi phí vận hành hóa đơn LLM.
* **Giải pháp khắc phục:**
  1. Bổ sung cơ chế **Hedging / Dynamic Routing based on Cost-Capacity**: Chỉ failover sang `beta` cho các request VIP hoặc các request sắp hết deadline; đối với các request thông thường, ưu tiên sử dụng `gamma-mini` (nếu không cần tool) hoặc áp dụng kỹ thuật graceful degradation (trả lời phản hồi mặc định lịch sự).
  2. Thiết lập client-side concurrency limiter / rate limiter cho từng provider đích trong ladder để tránh đánh sập secondary provider.

---

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.592** | **192** | **0** | **2** | **0.955** | **PASS** |

### Cấu hình tham số `cache:`

| Tham số `cache:` | Giá trị | Vì sao (dựa trên số liệu thực nghiệm C1) |
|---|---:|---|
| `similarity_threshold` | **0.92** | Đảm bảo phân biệt chính xác các câu hỏi FAQ tương tự ngữ nghĩa nhưng khác bản chất. Ở mức 0.80, có tới 181 câu bị trả lời sai (`wrong_answers: 181`). Ở mức 0.92, `wrong_answers` triệt để về 0 mà vẫn giữ được `cache_hit_ratio` đạt 59.2% (vượt tiêu chí >= 50%). |
| `ttl_s` | **30 s** | Giới hạn số câu trả lời lỗi thời khi có thay đổi âm thầm không webhook (`silent_updates` tại t=480s). Với TTL=30s, số câu trả lời cũ chỉ là 2 (`outdated_answers: 2` <= 15). Nếu để TTL=300s, `outdated_answers` vọt lên 95 (vi phạm nặng nề). |
| `max_stale_s` | **300 s** | Cung cấp cửa sổ phục vụ tạm (stale-if-error) trong suốt đợt sập toàn diện của mọi provider kéo dài 120s (t=240 đến t=360). Phục vụ thành công 192 câu stale, kéo `faq_availability` từ 84.24% lên 96.45% (vượt tiêu chí >= 93%). |

### Ba thí nghiệm thực nghiệm trên băng C1

1. **Ngưỡng tương đồng (`similarity_threshold`):**
   * $\text{Threshold} = 0.80$: `hit_ratio = 0.610`, `wrong_answers = 181` (`wrong_answers_by_reason: {'other_question': 181}`), `cost/turn = 1.015`. Ngưỡng 0.80 quá lỏng lẻo khiến câu hỏi "Thời gian hoàn tiền mất bao lâu?" bị nhầm sang "Chính sách hoàn tiền thế nào?".
   * $\text{Threshold} = 0.92$: `hit_ratio = 0.592`, `wrong_answers = 0`, `cost/turn = 0.955` (Đạt chuẩn hoàn hảo).
   * $\text{Threshold} = 0.97$: `hit_ratio = 0.499` (tụt dưới mức tối thiểu 0.50), `wrong_answers = 0`, `cost/turn = 1.018`. Ngưỡng quá khắt khe khiến các câu hỏi diễn đạt hơi khác một chút (paraphrase) bị cache miss vô ích.

2. **Thời gian sống (`ttl_s`):**
   * $\text{TTL} = 30\text{s}$: `outdated_answers = 2`, `hit_ratio = 0.592` (PASS).
   * $\text{TTL} = 60\text{s}$: `outdated_answers = 3`, `hit_ratio = 0.652` (PASS).
   * $\text{TTL} = 300\text{s}$: `outdated_answers = 95`, `hit_ratio = 0.731` (FAIL nặng vì vượt mức tối đa 15).
   * **Lý do webhook không giúp được gì:** Webhook chỉ bắn được khi hệ thống nội bộ (CMS) chủ động thông báo đổi chính sách. Khi đối tác thứ ba (hãng vận chuyển) âm thầm tăng giá cước tại $t=480\text{s}$ mà không thông báo, hệ thống hoàn toàn không nhận được tín hiệu invalidation nào. Lúc này, **TTL là cơ chế duy nhất** đảm bảo cache tự động làm mới và chặn đứng việc phục vụ thông tin sai lệch cho người dùng.

3. **Cơ chế Stale-if-error (`max_stale_s`):**
   * Khi đặt `max_stale_s = 0` (tắt stale-if-error): `faq_availability` sụt giảm từ 96.45% xuống còn **85.29%** (vi phạm tiêu chí >= 93%), `stale_served = 0`.
   * Trong 120s mọi provider đều sập ($t=240 - 360$), không một câu hỏi nào được trả lời dù cache đang giữ câu trả lời cũ chất lượng tốt. Nhờ có `max_stale_s = 300s`, hệ thống đã cứu được 192 lượt user (`stale_served = 192`).

### Trả lời câu hỏi lý thuyết

* **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") KHÔNG được cache, kể cả khi cache theo từng user?**
  1. **Tính biến động cao (Staleness):** Trạng thái đơn hàng, số dư hoàn tiền, điểm tích lũy thay đổi liên tục theo thời gian thực (ví dụ shipper vừa giao hàng thành công). Trả về cache cũ sẽ khiến khách hàng nhận thông tin sai lệch về tình trạng đơn hàng của họ.
  2. **Rủi ro rò rỉ dữ liệu cá nhân (PII / Privacy Leak):** Dù cố gắng phân tách theo user, việc so khớp ngữ nghĩa bằng embedding có rủi ro va chạm hoặc nhầm lẫn tham số ngữ cảnh giữa các phiên đăng nhập, dẫn đến nguy cơ hiển thị thông tin nhạy cảm của đơn hàng cũ hoặc người khác.
* **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  * Không thể chỉ nhìn vào metric HTTP status hoặc `turn_success_rate` tổng quát ở gateway vì cache đang hấp thụ và che giấu sự cố.
  * Phải thiết lập alert dựa trên:
    1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio = stale_served / total_requests`)**: Nếu `stale_served > 0` hoặc tăng đột biến trong 1 phút, lập tức cảnh báo provider upstream đang ngừng hoạt động.
    2. **Tỷ lệ lỗi ở tầng Backend / LLM Client Provider (`upstream_error_rate` / `circuit_breaker_open_state`)**: Theo dõi trực tiếp trạng thái circuit breaker của các provider (alpha, beta...) hoặc rate lỗi 5xx từ simulator/client.
