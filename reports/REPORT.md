# Lab 25B — Báo cáo Game Day

**Họ tên / nhóm:** Nguyễn Hải Long (2A202602471)  **Commit:** `ef05e56`  **Điểm tự động (`make grade`):** 84.0 / 88
**Điểm tự động công khai:** 84/84 điểm có thể kiểm chứng tại máy (84/88 theo thang điểm tự động; 4 điểm SLO ẩn do giảng viên chấm). **Chấm tay:** tối đa 12 điểm, chưa có kết quả.
**Cách tái lập:** `python scripts/grade.py --seeds 25,26,27`; các bảng chi tiết dùng seed 25 từ `run_gameday.py --client both --seed 25`, `run_workflows.py --executor both`, `run_slo.py --alerter both` và băng C1.

## 1. Kết quả sáu băng Game Day

| Băng | Client | Availability | p95 (s) | Amplification | Retry ratio | Cost/good turn | Recovery (s) | Kết quả |
|---|---|---:|---:|---:|---:|---:|---:|---|
| G1 · rate limit | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4 | FAIL |
| G1 | resilient | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0 | PASS |
| G2 · overload | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155 | FAIL |
| G2 | resilient | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0 | PASS |
| G3 · brownout | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0 | FAIL |
| G3 | resilient | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0 | PASS |
| G4 · chậm | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0 | FAIL |
| G4 | resilient | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0 | PASS |
| G5 · bad deploy/quota | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | — | FAIL |
| G5 | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | — | PASS |
| G6 · outage | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0 | FAIL |
| G6 | resilient | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0 | PASS |

G1 có **0** lần vi phạm `Retry-After` với resilient, so với **36.027** ở naive. G2 trở lại thành công ngay sau mốc phục hồi theo định nghĩa metric (`recovery_s=0`) và `primary_share_after_restore=1.0`. G5 có **0** retry lỗi client; G6 có **0** lần định tuyến tool call sang model không hỗ trợ tool. Sáu băng đều PASS trên seed 25, 26, 27; băng ẩn có thời điểm và cường độ khác nên kết quả đó chưa bảo đảm điểm tuyệt đối.

Nguồn: `reports/gameday/naive.json`, `resilient.json` và `reports/scoreboard.json`.

## 2. Cấu hình client và quyết định khi lỗi

| Tham số | Giá trị | Cơ sở chọn |
|---|---:|---|
| `attempt_timeout_s` | 8 s | G6 với 30 s giảm availability từ 1.0000 xuống 0.9488; 8 s còn thời gian thử provider khác trong deadline 30 s. |
| `max_attempts_per_provider` | 2 | Cho phép một retry lỗi hạ tầng; G2 vẫn chỉ có amplification 1.041 nhờ breaker và budget. |
| `backoff_base_s / cap_s` | 0.25 / 4 s | Full jitter cho lỗi tạm thời; `Retry-After` được tôn trọng. G1: 0 vi phạm, so với 36.027 ở naive. |
| `retry_budget_ratio / min_per_s / max_tokens` | 0.1 / 0.5 / 20 | Giới hạn retry khi breaker chưa kịp mở. Trong thí nghiệm breaker bị vô hiệu, G2 amplification 1.383 với budget so với 1.672 khi tắt. Bucket có token ban đầu và nạp theo thời gian nên **không phải trần cứng 10%** trong mọi khoảng ngắn. |
| `breaker.window_size / min_calls` | 20 / 10 | Cần ít nhất 10 kết quả trước khi trip, tránh một vài lỗi lẻ; G2 failover và phục hồi trong ngưỡng chấm. |
| `failure_rate_threshold` | 0.5 | Bắt overload G2 nhưng không mở vì lỗi 400 ở G5; G5 có 0 failover không cần thiết. |
| `slow_call_s / slow_rate_threshold` | 5 s / 0.6 | Bắt G4 dù call vẫn thành công; p95 giảm từ 13.764 s xuống 5.562 s. |
| `open_duration_s` | 10 s | Giảm tải provider trong sự cố và thử lại định kỳ; G2 có recovery 0 s, primary share sau phục hồi 1.0. |
| `half_open_max_probes / successes` | 2 / 2 | Giới hạn hai probe đang bay, đóng breaker khi có hai probe thành công; tránh dồn lưu lượng khi vừa phục hồi. |
| `quota_cooldown_s` | 300 s | Không tiếp tục gửi vào tài khoản hết quota; G5 vẫn đạt availability 0.8179. |

`on_error` phân loại rồi xử lý như sau:

| Lỗi | Breaker | Hành động |
|---|---|---|
| 400/401/403/404/422 (client error) | Chỉ trả permit, không tính lỗi | `GiveUp`; không retry, không failover vì cùng request vẫn sai. |
| 429 `insufficient_quota` | Không tính lỗi | Cooldown dài cho provider, chuyển provider tương thích. |
| 429 rate limit | Không tính lỗi | Cooldown đến `retry_after`, chuyển provider; nếu hết lựa chọn, có thể đợi cooldown ngắn còn đủ deadline. |
| 529/503, 5xx, timeout | Tính lỗi; breaker có thể mở | Retry cùng provider với jitter nếu còn lượt, budget, breaker permit và deadline; nếu không thì failover. |
| Provider thành công nhưng chậm | Ghi slow call | Mở breaker khi tỷ lệ slow vượt ngưỡng; G4 là ví dụ. |

`ladder(task)` chỉ đưa `tool` tới `alpha`, `alpha-eu`, `beta`; `gamma-mini` chỉ nhận `chat`.

## 3. Ba thí nghiệm đối chứng

**Retry budget khi breaker cấu hình sai.** Đặt `breaker.min_calls=1_000_000_000` và `max_attempts_per_provider=5`. Ở G2, bật budget cho amplification **1.383**, retry ratio **0.058**, availability **0.9987**, recovery **0 s**; tắt budget lần lượt là **1.672**, **0.491**, **0.9649**, **30 s**. Ở G3, retry ratio tăng từ **0.068** lên **0.162** và amplification từ **1.136** lên **1.163** khi tắt budget. Breaker cô lập provider suy yếu sau khi đủ kết quả; budget giới hạn lượng retry trong giai đoạn đó và bảo vệ khi breaker cấu hình sai. Số của thí nghiệm dùng cùng seed 25 và được tái lập bằng `python scripts/run_experiments.py`.

**Timeout 30 s.** G4 gần như không đổi: availability **1.0000** ở cả 8 s và 30 s, p95 **5.604 → 5.683 s**. G6 giảm availability **1.0000 → 0.9488** khi timeout tăng từ 8 lên 30 s; p95 của **các lượt thành công** là **6.016 → 5.306 s**, vì các lượt chờ quá hạn bị loại khỏi mẫu p95. Timeout cần dựa trên phân bố độ trễ bình thường và chừa thời gian cho đường dự phòng trong deadline 30 s, không chỉ nhìn p95 của các lượt còn sống.

**Bỏ lọc khả năng tool.** Trên G6, `incompatible_routes=258`, có **464** attempt tới `gamma-mini`, và availability từ **1.0000** giảm xuống **0.8918**. Với người dùng sản phẩm, đây là 258 lần hệ thống giao tác vụ cần tra cứu/cập nhật đơn hàng cho model không thể gọi công cụ, khiến một phần lượt hoàn tiền thất bại. Lọc theo khả năng là điều kiện để fallback thực sự giúp khách hàng.

## 4. Workflow hoàn tiền

| Executor | Hoàn tất | Worker crash | Refund trùng | Email trùng | Refund mất | Trả dư (cent) | LLM lãng phí/crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 500/500 | 81 | 128 | 81 | 0 | 1.574.200 | 1.123 |
| yours | 500/500 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Idempotency key ổn định theo `(session, step)` khiến payment/email chỉ có một hiệu lực kể cả khi response bị mất hoặc worker chết sau side effect. Checkpoint bền vững sau mỗi bước thành công giúp worker resume mà không gọi lại các bước LLM đã xong; chỉ key thì không tránh được phần tính toán lặp. Chỉ checkpoint cũng chưa đủ: crash sau khi payment chạy nhưng trước khi checkpoint được ghi sẽ khiến worker gọi payment lần nữa. Ghi checkpoint **trước** payment còn nguy hiểm hơn: crash giữa hai việc khiến worker bỏ qua khoản hoàn tiền chưa hề thực hiện.

## 5. Alerting và error budget

| Sự cố | Kỳ vọng | Alerter naive | Burn-rate alerter | Trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page | page | 27 phút |
| `provider_outage` | page | page | page | 4 phút |
| `deploy_blip` | none | page sai | none | — |
| `slow_leak` | ticket | bỏ sót | ticket | 1.337 phút |
| `degraded_evening` | page | page | page | 291 phút |
| `night_spike` | none | page sai | none | — |
| `small_bump` | none | page sai | none | — |

Burn-rate dùng đồng thời hai cửa sổ cho mỗi ngưỡng: **14.4× (1 h/5 m)**, **6× (6 h/30 m)** và ticket **1× (3 ngày/6 h)**. Trên dữ liệu public: **3 page đúng, 1 ticket đúng, 0 false page**; dữ liệu ẩn chưa được cung cấp.

Trong 30 ngày có **25.901.755** request, **55.618** lỗi trên ngân sách **129.508,8** lỗi (SLO 99,5%). Availability thực tế **99,7853%**, đã dùng **42,95%** budget, còn **57,05%**: **không đóng băng release** theo tiêu chí bài. `small_bump` có **1.794** lỗi trong 20 phút, khoảng **1,39%** budget tháng; mức tăng ngắn này không thỏa đồng thời các cửa sổ page, nên không đánh thức on-call. Vẫn cần quan sát và ticket nếu rò rỉ kéo dài.

## 6. Postmortem G2 — client naive

![Attempts và success ratio trong G2](gameday/G2_overload_storm.png)

**Tóm tắt.** Từ t=120 đến t=300 s, `alpha` mất 75% năng lực GPU. Client naive tiếp tục retry ba tầng với delay cố định, không breaker, budget hay fallback, đưa amplification lên **9.904**. Dù GPU phục hồi ở t=300 s, tỷ lệ thành công chỉ trở lại ngưỡng phục hồi sau **155 s** nữa.

**Ảnh hưởng.** Trong **2.362** lượt người dùng, **1.285** lượt thất bại (availability **45,60%**); cost/good turn **4.627** so với **2.119** của resilient. Nếu quy chiếu cùng tốc độ 4 lượt/s suốt 30 ngày, ngân sách SLO 99,5% là **51.840** lượt lỗi; riêng G2 dùng khoảng **2,48%** ngân sách giả định đó. Đây là phép quy chiếu để hiểu quy mô, không phải số liệu 30 ngày thực đo.

**Timeline** từ `reports/gameday/naive_G2_overload_storm_timeseries.csv`. Tỷ lệ ở cột cuối là `calls_ok/calls_done` của đúng giây được ghi; hình sử dụng cửa sổ trượt 10 s.

| t (s) | Diễn biến | Attempts/s vào alpha | Calls ok/done |
|---:|---|---:|---:|
| 119 | Trước sự cố | 9 | 7/7 |
| 120 | Mất 75% GPU | 5 | 7/7 |
| 150 | Retry storm rõ rệt | 163 | 0/3 |
| 200 | Hàng đợi vẫn nghẽn | 135 | 0/1 |
| 250 | Đỉnh quan sát trong các mốc này | 184 | 0/6 |
| 300 | GPU phục hồi | 115 | 0/5 |
| 350 | Provider khỏe nhưng client vẫn thất bại | 17 | 0/4 |
| 450 | Chưa phục hồi | 16 | 0/3 |
| 455 | Thành công bắt đầu trở lại | 13 | 4/5 |
| 456 | Tỷ lệ trong giây đạt 100% | 15 | 8/8 |

**Trigger và nguyên nhân gốc.** Trigger là mất GPU của `alpha`. Nguyên nhân gốc của tác động kéo dài là retry chồng app × gateway × SDK, delay cố định, bỏ qua `Retry-After`, không giới hạn retry và không chuyển hướng; G2 ghi **34.475 attempts cho 3.481 logical calls**, **28.381** vi phạm `Retry-After`. Request client đã timeout vẫn tiếp tục chiếm chỗ xử lý trên server. Retry lại thêm request mới vào hàng đợi, kéo dài thời gian chờ và sinh tiếp timeout: vòng lặp tự duy trì ngay cả sau t=300, đến khi backlog giảm đủ ở khoảng t=455. Đây là lỗi thiết kế đường phục hồi và thiếu guardrail, không quy trách nhiệm cho cá nhân.

**Điều giúp ích và làm nặng thêm.** Mất năng lực GPU là sự cố ban đầu; retry cùng nhịp vào đúng provider làm nó trầm trọng hơn. Với resilient, breaker, budget, jitter và fallback đưa amplification G2 về **1.041**, availability lên **1.0000**, recovery về **0 s** theo metric của băng. Chúng hạn chế tải mới trong lúc `alpha` giải phóng hàng đợi.

| # | Action item | Loại | Owner | Kiểm chứng |
|---|---|---|---|---|
| 1 | Dùng client tập trung có breaker theo tỷ lệ lỗi/chậm và probe giới hạn | Prevent | Platform SDK | G2/G4 PASS trên ba seed; G2 recovery ≤30 s, G4 p95 ≤10 s |
| 2 | Chặn retry ba tầng, áp dụng jitter, `Retry-After` và budget | Mitigate | Client SDK | G1 có 0 vi phạm `Retry-After`; G2 amplification ≤1.6 trên CI |
| 3 | Theo dõi attempts/logical call, backlog và burn rate đa cửa sổ | Detect | SRE | Replay G2 cảnh báo khi amplification tăng; SLO public 0 false page |

## 7. Điểm yếu còn lại

G2 của resilient vẫn có p95 **10.920 s**, cao hơn G1 **4.776 s**, dù mọi lượt đều hoàn tất; một số lượt phải đợi lỗi/timeout rồi mới failover. Hướng cải tiến là thử hedged request có giới hạn khi độ trễ vượt ngưỡng, hủy attempt thua và đo lại p95, amplification, chi phí trên nhiều seed. Đây là đề xuất chưa được đo; nếu hedge không được kiểm soát, nó có thể tái tạo thêm tải lúc provider đang nghẽn.

## 8. Semantic cache và stale-if-error

| Client trên C1 | Availability | FAQ availability | Cache hit | Stale served | Wrong | Outdated | Cost/good turn | Kết quả |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| resilient, không cache | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số | Giá trị | Cơ sở chọn |
|---|---:|---|
| `similarity_threshold` | 0.92 | Ở 0.80 có **157** wrong answers, tất cả `other_question`; 0.92 đưa về **0** mà hit vẫn **65,2%**. |
| `ttl_s` | 60 s | TTL 300 s tạo **95** outdated answers sau lần cập nhật không báo trước; 60 s còn **3**. |
| `max_stale_s` | 300 s | Không stale: FAQ availability **87,33%**; 300 s: **96,45%** và **157** lượt FAQ được phục vụ khi provider không trả lời. |
| `max_entries` | 500 | Chặn tăng bộ nhớ, LRU bỏ entry ít dùng; băng C1 vẫn đạt hit **65,2%**. |

**Ngưỡng cosine.** Với 0.80 / 0.92 / 0.97, hit ratio là **0.666 / 0.652 / 0.581**, wrong answers là **157 / 0 / 0**, cost/good turn là **0.965 / 0.915 / 0.964**. Ngưỡng 0.80 ghép các câu hỏi gần embedding nhưng khác ý (`other_question`), nên hit cao hơn không có nghĩa chất lượng cao hơn. Giá trị 0.92 giữ an toàn mà không mất nhiều hit như 0.97.

**TTL.** Với 30 / 60 / 300 s, outdated answers là **2 / 3 / 95**, hit ratio **0.592 / 0.652 / 0.731**, cost/good turn **0.955 / 0.915 / 0.863**. Ở C1, chính sách hoàn tiền có webhook nên invalidation theo `source` xóa được entry liên quan. Bảng phí ship đổi ở t=480 s **không có webhook**; cache không thể biết nguồn đã đổi, nên TTL là giới hạn tuổi của câu trả lời cũ trong điều kiện đó. Cửa sổ stale vẫn có thể giữ entry lâu hơn TTL, nhưng chỉ dùng khi upstream không trả lời.

**Stale-if-error.** `max_stale_s=0` cho FAQ availability **0.8733**, overall availability **0.8565**, `stale_served=0`; ở 300 s lần lượt là **0.9645**, **0.9124**, **157**. Stale chỉ là phương án sau khi provider bỏ cuộc và entry vẫn cùng `prompt_version`, chưa bị invalidation, còn trong cửa sổ stale. Nhờ vậy C1 qua ngưỡng FAQ ≥93% mà `wrong_answers=0`.

Câu hỏi cá nhân không được semantic cache dù phân vùng theo user: trạng thái đơn hàng/hoàn tiền thay đổi nhanh, có thông tin nhạy cảm và ít tái sử dụng; cache theo vector còn có nguy cơ ghép nhầm câu hỏi. Khi cache giữ dashboard người dùng “xanh” lúc provider sập, on-call cần xem riêng **provider error/timeout rate, breaker OPEN, số lượt stale served và tuổi entry**. Tăng đột biến `stale_served` cùng lỗi upstream là tín hiệu sự cố ngay cả khi edge availability còn cao.
