# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Phương Nam - MSV: 2A202602869  **Commit:** 76f9422a6ca59e42be8039ceb98a05b73cd9c850  **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4đ dataset ẩn = 88.0 / 88)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL (avail, retry-after, ampl) |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL (avail, ampl, recov) |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL (retry_ratio_max) |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL (latency_p95_s_max) |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL (client_error_retries, avail) |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL (avail) |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0s | G6 (provider A outage, alpha-eu treo). Nếu đặt timeout 30s theo turn deadline, lượt treo ở alpha-eu sẽ ngốn sạch 30s của cả session khiến availability tụt xuống 94.88% (FAIL). Giá trị 8.0s lớn hơn p99 lúc bình thường (~1.5–2s) nhưng đủ nhỏ để cho phép 2 attempt và failover kịp trước khi hết 30s. |
| max_attempts_per_provider | 2 | G1, G2, G3. Cho phép tối đa 1 lần retry trên cùng provider rồi failover ngay. NaiveClient retry tới 36 lần khiến amplification vọt lên 9.904 ở G2 và 17.030 ở G5. Giá trị 2 chặn đứng retry storm cục bộ. |
| backoff_base_s / cap_s | 0.25s / 4.0s | G1, G2. Thuật toán Full Jitter: delay = uniform(0, min(cap, base * 2^(n-1))). Giúp giải đồng pha (desynchronize) các đợt retry đồng thời, tránh thundering herd đập vào provider. Cap 4.0s giữ độ trễ không vượt quá turn deadline. |
| retry_budget_ratio | 0.1 | G2, G3. Token bucket nạp 10% token trên mỗi request ban đầu. Trong G3, budget giữ retry_ratio ở 0.034 (chuẩn <= 0.10). Ngay cả khi breaker bị hỏng (Thí nghiệm 1), budget vẫn chặn retry_ratio ở 0.058–0.068 thay vì 0.491. |
| breaker.window_size / min_calls | 20 / 10 | G2, G3, G4. Cần tối thiểu 10 kết quả để tính tỷ lệ lỗi, tránh mở nhầm mạch (false trip) do 1-2 request lỗi ngẫu nhiên lúc traffic thấp. Cửa sổ 20 kết quả (~2.5s ở tải 8 calls/s) đủ nhạy để phát hiện sự cố nhanh chóng. |
| breaker.failure_rate_threshold | 0.5 | G2, G3. Khi tỷ lệ lỗi >= 50% trên 20 request gần nhất, provider chắc chắn đang gặp sự cố nghiêm trọng (529/500/timeout). Mở mạch ngay giúp cứu 100% availability của G2 và G3. |
| breaker.slow_call_s | 5.0s | G4 (slow region). Alpha bị chậm thêm 5.0s nhưng không trả mã lỗi nào. Nhờ slow_call_s = 5.0s và slow_rate_threshold = 0.6, breaker phát hiện suy thoái độ trễ và ngắt mạch, kéo p95 từ 13.764s (naive) xuống 5.562s. |
| breaker.open_duration_s | 10.0s | G2, G4, G6. Giữ mạch OPEN 10 giây để provider có thời gian xả sạch hàng đợi quá tải và hồi phục tài nguyên trước khi client thử gửi probe thăm dò. |
| breaker.half_open_max_probes | 2 | G2. Ở trạng thái HALF-OPEN, chỉ cho phép tối đa 2 request probe đang bay đồng thời (kèm half_open_successes = 2), loại bỏ hoàn toàn hiện tượng thundering herd làm sập lại provider vừa mới hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                           [Attempt Failed: on_error]
                                       |
                               [Classify Error]
                                       |
       +-------------------------------+-------------------------------+
       |                               |                               |
 [CLIENT_ERROR]           [QUOTA_EXHAUSTED / RATE_LIMITED]   [OVERLOADED / SERVER_ERROR
 (400, 401, 403, 422)     (429 insufficient / 429 limit)      / TIMEOUT (529, 503, 500, 0)]
       |                               |                               |
 Breaker: release()              Breaker: release()              Breaker: on_failure()
       |                               |                               |
 GiveUp("client_error")    Cooldown provider:                  Có thể retry cùng provider?
 (KHÔNG retry,              - Quota: now + 300s                (tries < max_attempts &&
  KHÔNG failover)           - RateLimit: now + retry_after      budget.acquire() &&
                                       |                        fits deadline?)
                               Failover ngay lập tức                   |
                               (route provider kế tiếp)          +-----+-----+
                                                                 |           |
                                                               [CÓ]        [KHÔNG]
                                                                 |           |
                                                           Retry cùng    Failover ngay
                                                           provider sau  sang provider
                                                           Full Jitter   kế tiếp
                                                           Backoff
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**
   - Khi giả lập breaker bị vô hiệu hóa (`breaker.min_calls: 1000000000`) và tăng `max_attempts_per_provider: 5`:
     - Bật retry budget: Trên G2 đạt availability 99.87%, amplification 1.383, retry_ratio 0.058, recovery 0.0s; trên G3 đạt availability 100%, amplification 1.136, retry_ratio 0.068.
     - Tắt retry budget: Trên G2 availability giảm còn 96.49%, amplification tăng lên 1.672, retry_ratio vọt lên 0.491 (gần 50% traffic là retry!), recovery kéo dài đến 30.0s; trên G3 retry_ratio vọt lên 0.162 (vi phạm ngưỡng 0.10).
   - Khi circuit breaker hoạt động bình thường, nó ngắt mạch theo tỷ lệ lỗi của từng provider cụ thể. Tuy nhiên, retry budget vẫn tối quan trọng vì đóng vai trò "chốt chặn an toàn cuối cùng" (defense in depth): nó bảo vệ toàn bộ hạ tầng trước những lỗi cấu hình breaker, trong giai đoạn warm-up khi số cuộc gọi chưa đủ `min_calls`, hoặc khi lỗi phân tán trên nhiều provider mà chưa breaker nào kịp mở.

2. **`attempt_timeout_s: 30`.**
   - Chạy G6 với timeout 8.0s: availability đạt 100.0%, p95 là 6.016s, `give_up_reasons` rỗng.
   - Chạy G6 với timeout 30.0s: availability tụt mạnh xuống **94.88%** (FAIL vì check yêu cầu >= 95%), xuất hiện 109 lượt GiveUp vì hết deadline (`'deadline': 109, 'no_provider_available': 13`).
   - Đáng chú ý, latency p95 trên G6 lại *giảm* từ 6.016s xuống 5.306s. Đây là hiện tượng **Survivorship Bias** (thiên kiến sống sót): metric p95 chỉ tính trên các lượt *thành công*. Những lượt bị treo ở provider alpha-eu (outage: hang) mất trọn 30s đã bị drop vì quá hạn turn deadline và tính là thất bại, loại bỏ các mẫu chậm ra khỏi tập tính toán khiến p95 "đẹp giả tạo".
   - Ngược lại, trên G4, p95 gần như không đổi (5.604s ở 8s vs 5.683s ở 30s) vì alpha chỉ bị chậm 5s (latency ~6.5s, chưa chạm trần 8s hay 30s), và circuit breaker đã phát hiện slow calls (>= 5.0s) để trip sang alpha-eu từ sớm mà không cần chờ timeout.
   - Do đó, timeout "đúng" không được đặt bằng turn deadline mà phải dựa vào: `Turn Deadline` trừ đi thời gian dự phòng cần thiết để failover sang ít nhất 1–2 provider dự phòng trong ladder, đồng thời bao phủ mức p99 phản hồi bình thường của provider (~2–3s).

3. **Bỏ lọc tool calling khỏi `ladder()`.**
   - Khi bỏ lọc `supports_tools` trong hàm `ladder("tool")` và chạy trên G6, availability tụt từ 100% xuống **89.18%**, và chỉ số `incompatible_routes` tăng vọt từ 0 lên **258 cuộc gọi**.
   - Dưới góc nhìn của một Product Manager: "258 khách hàng yêu cầu hoàn tiền đã bị hệ thống từ chối hoặc trả về kết quả lỗi nghiêm trọng, khiến tỷ lệ hoàn tất giao dịch sụt giảm hơn 10%. Nguyên nhân là do tầng định tuyến kỹ thuật đã gửi nhầm yêu cầu gọi tool sang mô hình giá rẻ `gamma-mini` – một model hoàn toàn không có khả năng gọi function/tool. Việc định tuyến sai năng lực mô hình phá vỡ luồng nghiệp vụ cốt lõi, làm hỏng trải nghiệm khách hàng và gây tổn thất uy tín dịch vụ."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,496,700 cents | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 cents | 0.303 |

- **Vì sao chỉ idempotency key là chưa đủ?**
  Idempotency key đảm bảo một lời gọi API thanh toán cụ thể không bị thực thi lặp lại ở phía cổng thanh toán. Tuy nhiên, nếu thiếu checkpoint, khi worker bị crash ở các bước phía sau (như `send_email` hay `write_summary`), worker mới khởi động lại sẽ không biết phiên làm việc đã tiến hành đến đâu và buộc phải chạy lại toàn bộ từ bước 1. Điều này gây lãng phí chi phí gọi LLM lớn (`wasted_llm_per_crash = 1.123` ở bản naive so với 0.303 ở bản có checkpoint) và làm tăng tải không cần thiết lên toàn bộ hệ thống.
- **Vì sao chỉ checkpoint cũng chưa đủ?**
  Checkpoint lưu trạng thái vào `DurableStore` sau khi bước thành công. Nhưng nếu gặp sự cố mạng (`ServiceTimeout` - tiền đã trừ nhưng phản hồi bị mất trên đường truyền) hoặc worker bị crash ngay sau khi tiền đã chuyển nhưng trước khi kịp ghi checkpoint, worker tiếp theo kiểm tra store sẽ thấy bước refund chưa hoàn thành và gọi lại API refund. Nếu không có idempotency key ổn định theo `(session_id, step)`, cổng thanh toán sẽ xem đây là giao dịch mới và tiếp tục trừ tiền, dẫn đến hoàn tiền trùng lặp nghiêm trọng (`duplicate_refunds = 128`, overpaid gần 15.000 USD ở bản naive).
- **Điều gì xảy ra nếu ghi checkpoint *trước* khi gọi payment?**
  Đây là lỗi thiết kế nguy hiểm nhất (ghost payment / lost refund): nếu worker bị crash hoặc gặp lỗi ngay sau khi vừa ghi checkpoint mà chưa kịp gọi (hoặc lời gọi thất bại) sang payment API, worker mới hồi phục đọc checkpoint sẽ tin rằng bước thanh toán đã thành công và tiếp tục nhảy sang gửi email xác nhận. Hậu quả là khách hàng nhận được email thông báo đã hoàn tiền thành công nhưng thực tế tài khoản không hề nhận được tiền (`lost_refunds > 0`), gây khiếu nại và vi phạm pháp lý nghiêm trọng.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (FALSE PAGE) | none | — |
| slow_leak | ticket | none (MISSED) | ticket | 1337 min (~22.3h) |
| degraded_evening | page | page | page | 291 min (~4.8h) |
| night_spike | none | page (FALSE PAGE) | none | — |
| small_bump | none | page (FALSE PAGE) | none | — |

- **Error budget 30 ngày:**
  - Tổng số request: 25,901,755
  - Số request lỗi cho phép (SLO 99.5%): 129,508.8
  - Số request lỗi thực tế: 55,618
  - Availability đạt được: **99.7853%**
  - Ngân sách lỗi đã tiêu thụ: **42.95%**
  - Ngân sách lỗi còn lại: **57.05%**
  - Quyết định đóng băng release: **Không** (`freeze_releases = False`), vì ngân sách lỗi còn lại hơn 57%, hệ thống vẫn đáp ứng tốt cam kết SLO 99.5%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
  Trong 20 phút ở mức lỗi 12%, lượng request hỏng chiếm: $(20 \times 12\%) / (43200 \times 0.5\%) \approx 1.11\%$ tổng ngân sách lỗi của cả tháng. Sự cố này diễn ra ngắn và tự hết, tiêu hao một phần không đáng kể của error budget. Nếu page đánh thức kỹ sư on-call, nó sẽ gây hội chứng "chai lì cảnh báo" (alert fatigue), làm gián đoạn công việc và giảm khả năng phản ứng với các thảm họa thực sự. Bộ quy tắc Burn-rate Alerting đa cửa sổ (yêu cầu cả cửa sổ dài 1h và cửa sổ ngắn 5m cùng vượt ngưỡng 14.4x) loại bỏ hoàn toàn các false page dạng này mà vẫn bắt trọn các sự cố nghiêm trọng.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào thời điểm t=120s trong băng sự cố G2, provider LLM chính (`alpha`) bị sụt giảm 75% năng lực xử lý trong 3 phút do sự cố phần cứng GPU. Client naive trên production đã kích hoạt vòng lặp retry 3 tầng đồng loạt mà không có jitter hay circuit breaker, khuếch đại tải lên gấp gần 10 lần (amplification 9.904) và dội hơn 34.000 request vào provider đang nghẽn. Sự quá tải tự duy trì này đã đẩy hệ thống vào trạng thái thất bại siêu bền (metastable failure), khiến dịch vụ tiếp tục sập thêm 155 giây sau khi provider đã hồi phục hoàn toàn phần cứng, làm availability toàn đợt rơi xuống mức thảm hại 45.60%.

**Ảnh hưởng:**
- Tổng cộng 1.285 lượt yêu cầu của khách hàng bị thất bại (chỉ có 1.077 / 2.362 lượt thành công, availability chỉ đạt 45.60%).
- Sự cố kéo dài 335 giây ảo (từ t=120s đến t=455s), trong đó có 155 giây sập kéo dài sau khi provider phần cứng đã khôi phục 100% dung lượng tại t=300s.
- Nếu tính trên quy mô tháng, một đợt sập diện rộng 5.5 phút với tỷ lệ lỗi ~54.4% tiêu tốn hơn 100% error budget của cả tháng.

**Timeline (giây ảo — trích xuất từ CSV `naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số từ CSV) |
|---:|---|---|
| 0–119 | Hoạt động bình thường trước sự cố | attempts_alpha dao động 5–9 req/s, calls_ok = 7 req/s, success_ratio = 1.0 (100%). |
| 120 | alpha mất 75% GPU (capacity_factor = 0.25) | attempts_alpha = 5 req/s, calls_ok = 7 req/s; hàng đợi alpha bắt đầu tích tụ độ trễ. |
| 150 | Retry storm bùng nổ toàn diện | attempts_alpha tăng vọt lên **163 req/s** (>18x bình thường), calls_done = 3, calls_ok = 0 (success ratio = 0.0%). |
| 180 | Quá tải đỉnh điểm, nghẽn hàng đợi | attempts_alpha duy trì mức cao 138 req/s, calls_ok = 0. Hàng đợi server tràn ngập mã lỗi 529 và 429. |
| 240 | Server xử lý lãng phí các request đã timeout | attempts_alpha = 105 req/s, calls_ok = 0. Client timeout bỏ cuộc nhưng server vẫn tiếp tục xử lý vô ích. |
| 300 | alpha hồi phục 100% GPU phần cứng | attempts_alpha = 115 req/s, calls_ok = 0 (vẫn 0% thành công!). Hệ thống không thể tự hồi phục. |
| 360 | Tồn dư bão retry tiếp tục duy trì quá tải | attempts_alpha = 50 req/s, calls_ok = 0 (60s sau khi hồi phục phần cứng, tỷ lệ thành công vẫn là 0%). |
| 420 | Hàng đợi bắt đầu giảm tải chậm chạp | attempts_alpha = 53 req/s, calls_done = 2, calls_ok = 0 (120s sau khi hồi phục, hệ thống vẫn tắc). |
| 455 | Bão retry tan, hệ thống bắt đầu phục hồi | attempts_alpha hạ xuống 13 req/s, calls_done = 5, calls_ok = 4 (success ratio = 80%, đánh dấu recovery_s = 155s). |
| 500 | Trở lại trạng thái bình thường hoàn toàn | attempts_alpha = 9 req/s, calls_done = 10, calls_ok = 10 (success ratio = 1.0). |

**Trigger vs root cause:**
- **Trigger:** Provider alpha gặp sự cố hạ tầng, tạm thời mất 75% số lượng GPU trong 3 phút (từ t=120s đến t=300s).
- **Root cause:** Khiếm khuyết kiến trúc nghiêm trọng trong client v1 (`NaiveClient`):
  1. Xếp chồng 3 tầng retry độc lập (application x gateway x SDK) tạo ra hệ số nhân tải lên tới 36 lần.
  2. Retry với khoảng cách cố định (không jitter), tạo ra các đợt sóng xung kích đồng pha đánh sập hàng đợi.
  3. Hoàn toàn không có Circuit Breaker để ngắt mạch khi tỷ lệ lỗi vượt ngưỡng.
  4. Không có Retry Budget để giới hạn tỷ lệ retry tối đa của hệ thống.
  5. Không có Fallback Ladder để chuyển hướng lưu lượng sang các cụm dự phòng sẵn có (`alpha-eu`, `beta`).
- **Vì sao provider đã hồi phục phần cứng tại t=300s mà hệ thống vẫn sập đến t=455s (Metastable Failure)?**
  Cơ chế hàng đợi FIFO của provider: khi client timeout sau 8–10s chờ đợi, server vẫn tiếp tục tính toán hoàn tất request đó (lãng phí năng lực tính toán). Trong khi đó, hàng nghìn retry mới từ client tiếp tục được đẩy vào lấp đầy hàng đợi 10s. Sau t=300s, mặc dù GPU đã có lại 100% công suất, toàn bộ năng lực này bị tiêu tốn để xử lý các request rác mà client đã từ bỏ từ trước. Vòng lặp tự duy trì: *Server xử lý request cũ -> Request mới bị nghẽn và timeout -> Client lại retry -> Hàng đợi tiếp tục đầy* khiến hệ thống không thể tự thoát ra cho đến khi bão retry tự triệt tiêu sau 155 giây.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Điều làm tệ hơn:* 3 tầng retry không phối hợp; vi phạm 28.381 lần `retry-after` do dội request ngay lập tức; không có cơ chế timeout sớm cho các request đã hết hạn; dồn 100% traffic vào duy nhất provider alpha đang hấp hối.
- *Điều giúp:* Provider phục hồi năng lực phần cứng tại t=300s; turn deadline 30s của người dùng cuối cùng cũng cắt đứt các session quá hạn giúp lưu lượng giảm dần sau t=450s.

**Action items:**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế toàn bộ NaiveClient bằng ResilientClient tích hợp Circuit Breaker sliding window và Retry Budget 10%. | Prevent | Core Agent Platform Team | Chạy CI simulation băng G2 đạt amplification <= 1.2 và recovery_s <= 5s. |
| 2 | Áp dụng thuật toán Full Jitter Backoff và cơ chế tôn trọng tuyệt đối header `retry-after` cho mọi lần thử lại. | Prevent | SDK Infrastructure Team | Kiểm tra unit test và băng G1 đạt `retry_after_violations = 0`. |
| 3 | Triển khai Fallback Ladder đa vùng tự động (alpha -> alpha-eu -> beta) khi breaker mở hoặc rate limit. | Mitigate | Routing & Reliability Team | Availability đạt >= 98% trên toàn bộ 6 băng sự cố Game Day. |
| 4 | Cấu hình hệ thống cảnh báo đa cửa sổ Multi-window Multi-burn-rate Alerting (page ở 14.4x 1h/5m và 6x 6h/30m). | Detect | SRE On-call Team | Mô phỏng sự cố đạt 0 false page trên deploy blip và phát hiện outage trong < 5 phút. |

## 7. Một điểm yếu còn lại

- **Điểm yếu còn lại của ResilientClient:**
  Trong sự cố băng G2, dù availability đạt 100%, độ trễ p95 vẫn bị kéo dài lên tới **10.920s**. Nguyên nhân là khi alpha bắt đầu chậm và lỗi, request đầu tiên vẫn phải mất thời gian chờ timeout trên alpha trước khi failover sang `alpha-eu`. Hơn nữa, khi breaker chuyển sang trạng thái HALF-OPEN sau 10s, nếu provider vẫn chưa thực sự ổn định, các probe request tiếp tục chịu độ trễ cao. Ngoài ra, việc failover dồn dập sang `beta` trong các sự cố diện rộng như G6 làm tăng chi phí vận hành (chi phí của beta là 1.4 so với 1.0 của alpha, tổng cost tăng lên 5.221,4).
- **Phương án khắc phục:**
  1. *Hedged Requests / Speculative Retry:* Thay vì chờ trọn vẹn timeout, nếu request gửi tới alpha vượt quá ngưỡng p90 thông thường (~2.0s) mà chưa phản hồi, client chủ động bắn một bản sao song song sang `alpha-eu`. Bản sao nào về trước sẽ được dùng ngay và hủy bản sao còn lại. Kỹ thuật này giúp ép p95 của G2 xuống dưới 4s mà chỉ tăng chi phí thêm ~5%.
  2. *Cost-aware Failover Balancing:* Phân bổ lưu lượng failover thông minh theo tỷ lệ giữa `alpha-eu` (cost 1.0) và `beta` (cost 1.4) kết hợp concurrency limit nội bộ trên client để tránh làm quá tải cụm dự phòng và kiểm soát chi phí API.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL (faq_avail, hit) |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.90 | Dựa trên Thí nghiệm 1: Ở ngưỡng 0.80, có 157 câu hỏi bị trả lời sai (`wrong_answers = 157`). Ở ngưỡng 0.97, hit ratio bị giảm sâu xuống 58.1%. Ngưỡng 0.90 đạt độ chính xác tuyệt đối (`wrong_answers = 0`) trong khi vẫn duy trì cache_hit_ratio cao 65.2%. |
| ttl_s | 60s | Dựa trên Thí nghiệm 2: Khi phí ship đổi thầm lặng không webhook ở t=480, TTL 300s phục vụ tới 95 câu trả lời lỗi thời (`outdated_answers = 95` > 15, FAIL). TTL 60s giới hạn số câu lỗi thời xuống chỉ còn 3 câu (PASS <= 15) mà chi phí chỉ 0.915/turn. |
| max_stale_s | 300s | Dựa trên Thí nghiệm 3: Đợt outage t=240–360 kéo dài 120s trên toàn bộ provider. Nếu `max_stale_s: 0`, faq_availability chỉ đạt 87.33% (FAIL < 93%). Cửa sổ stale 300s cho phép phục vụ 157 câu trả lời stale an toàn, nâng faq_availability lên 96.45% (PASS). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng similarity_threshold.**
   - Chạy C1 với `similarity_threshold = 0.80`: cache_hit_ratio đạt 66.6%, nhưng sinh ra **157 câu trả lời sai** (`wrong_answers: 157`, lý do: `{'other_question': 157}`), cost là 0.965.
   - Chạy với ngưỡng **0.90**: cache_hit_ratio đạt 65.2%, hoàn toàn không có câu trả lời sai (`wrong_answers: 0`), cost tối ưu nhất ở mức 0.915.
   - Chạy với ngưỡng **0.97**: `wrong_answers: 0`, nhưng do quá khắt khe nên nhiều câu hỏi diễn giải (paraphrase) bị miss cache, khiến hit ratio tụt xuống 58.1% và chi phí tăng lên 0.964.
   - Ở ngưỡng 0.80, các câu hỏi có ngữ cảnh gần nhau nhưng bản chất khác biệt (ví dụ: "Thời gian hoàn tiền trong bao lâu?" bị match nhầm vào cache của "Chính sách và điều kiện hoàn tiền là gì?") đã bị cache trả lời sai hoàn toàn.

2. **TTL.**
   - Chạy với `ttl_s = 30`: `outdated_answers: 2`, hit ratio đạt 59.2%, cost = 0.955.
   - Chạy với `ttl_s = 60`: `outdated_answers: 3` (đáp ứng xuất sắc yêu cầu <= 15), hit ratio đạt 65.2%, cost = 0.915.
   - Chạy với `ttl_s = 300`: `outdated_answers: 95` (vi phạm nghiêm trọng trần 15 của bài toán), hit ratio 73.1%, cost = 0.863.
   - Khi đơn vị vận chuyển thay đổi bảng giá cước âm thầm tại t=480 mà không gửi webhook báo trước, cache không hề nhận được tín hiệu để gọi `invalidate()`. Trong trường hợp này, webhook hoàn toàn bất lực và thời gian sống TTL chính là cơ chế bảo vệ duy nhất để ép cache phải tự hủy các câu trả lời cũ và truy vấn dữ liệu mới từ provider.

3. **Stale-if-error.**
   - Chạy với `max_stale_s: 0`: Trong 120 giây cả 4 provider sập hoàn toàn (t=240–360), client không được phép dùng dữ liệu cũ nên toàn bộ câu hỏi FAQ bị fail, khiến `faq_availability` rơi xuống mức **87.33%** (FAIL vì tiêu chuẩn đòi hỏi >= 93%), `stale_served = 0`.
   - Chạy với `max_stale_s: 300`: Khi provider trả về GiveUp, client kích hoạt cơ chế bậc 3 của fallback ladder (stale-if-error), phục vụ thành công **157 câu trả lời stale**, kéo `faq_availability` lên **96.45%** (PASS).

Trà lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Tính toàn vẹn dữ liệu:* Dữ liệu cá nhân (trạng thái đơn hàng, số dư hoàn tiền, điểm tích lũy) thay đổi liên tục theo thời gian thực dựa trên các sự kiện nghiệp vụ bên ngoài. Việc cache dù chỉ vài chục giây có thể trả về thông tin sai lệch nghiêm trọng (ví dụ khách vừa bấm hủy đơn nhưng bot vẫn trả lời "đơn đang được giao"), gây hiểu lầm và tranh chấp.
  2. *Bảo mật & Rò rỉ thông tin:* Câu hỏi cá nhân chứa dữ liệu nhạy cảm (PII). Nếu logic xác thực hoặc session ID gặp lỗi nhỏ, cache có thể trả nhầm dữ liệu đơn hàng của người này cho người khác.
  3. *Hiệu quả cache cực thấp:* Câu hỏi cá nhân mang tính ngẫu nhiên cao (mỗi người một mã đơn khác nhau), tỷ lệ cache hit gần như bằng 0 nên việc lưu trữ chỉ gây lãng phí bộ nhớ và tăng overhead tìm kiếm cosine.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  Nếu chỉ nhìn vào `faq_availability` hay `overall_availability`, hệ thống giám sát sẽ bị đánh lừa bởi cache. Để on-call phát hiện sự cố ngay lập tức, cần thiết lập cảnh báo dựa trên:
  1. **Provider Error Rate / Circuit Breaker State:** Cảnh báo ngay khi tỷ lệ lỗi từ provider thật vượt ngưỡng hoặc khi có bất kỳ circuit breaker nào chuyển sang trạng thái `OPEN`.
  2. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_rate`):** Bình thường chỉ số này bằng 0. Khi `stale_served` tăng vọt bất thường, đó là dấu hiệu chắc chắn rằng hạ tầng provider phía sau đã sụp đổ và client đang phải "cầm cự" bằng stale-if-error.
  3. **Backend Synthetics / Uncached Health Checks:** Chạy các probe định kỳ với cờ bypass cache gửi thẳng vào các provider để liên tục đo lường nhịp tim thực tế của upstream service.
