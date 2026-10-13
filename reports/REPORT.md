# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Tú Tài — MSV 2A202602455  **Commit:** 19e701a  **Điểm tự động (`make grade`):** 84 / 88 (4 điểm còn lại là SLO dữ liệu ẩn, chỉ giảng viên chấm)

> Mọi số liệu lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`
> (seed mặc định 25), trừ khi ghi rõ là sweep nhiều seed.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0 | 4.725 | 1.064 | 0.002 | 2.099 | 0.0 | PASS |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0 | 6.454 | 1.019 | 0.003 | 2.077 | 0.0 | PASS |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL (retry_ratio) |
| G3 | yours | 1.0 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | PASS |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL (p95) |
| G4 | yours | 1.0 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.03 | 16.03 | 2.124 | – | FAIL |
| G5 | yours | 0.8179 | 5.482 | 1.003 | 0.002 | 2.504 | – | PASS |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0 | 6.093 | 1.062 | 0.028 | 2.181 | 0.0 | PASS |

Độ bền: tôi chạy client trên **20 seed (1–20)** của 6 băng công khai → 0 check thất bại; tệ nhất: G2 availability
0.9996, G3 retry_ratio 0.052, G4 p95 5.80 s, G5 availability 0.794 (ngưỡng 0.78), G6 availability 0.9996. Tôi cũng tự
viết 12 băng "ẩn" (lệch thời điểm, đổi cường độ: G1 ×3 traffic + 10 rps, G2 capacity 0.15 trong 240 s, G3 50% lỗi,
G4 +2.5/+3/+3.5/+8 s, G6 đảo vai hang/refuse, beta 8 rps…) × 10 seed: tất cả PASS, trừ một biến thể G5 với 70% request
malformed — ở đó 100% lượt thất bại là `client_error` (không client nào cứu được), nên ngưỡng 0.78 là bất khả thi.
Ở G5 mọi lượt mất đều là `give_up_reasons = {client_error: 413}`; `client_error_retries = 0`, `unneeded_failovers = 0`.

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Healthy p99 của beta (provider chậm nhất có tool) ≈ 1.8·e^(2.33·0.35) ≈ 4.1 s, cộng hàng đợi lúc tải cao → 8 s đủ rộng để không timeout oan. Đủ ngắn để một call bị treo (alpha-eu hang ở G6) vẫn còn ~20 s cho failover trong deadline 30 s. Thí nghiệm 2: timeout 30 s làm G6 tụt xuống 0.9488 (FAIL). |
| max_attempts_per_provider | 2 | 1 retry tại chỗ cho lỗi tạm thời (G3: 30% 500 → xác suất 2 lần liên tiếp lỗi 9%), rồi failover. G3 retry_ratio 0.035 ≤ 0.10. Lên 5 thì (thí nghiệm 1) amplification G2 lên 1.38–1.68. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter: retry 1 trung bình 0.125 s (không làm chậm lượt), trần 4 s vẫn nằm trong deadline. 429 luôn chờ ≥ retry-after (G1: `retry_after_violations = 0`). |
| retry_budget_ratio | 0.1 (min 0.5/s, max 20 token) | Retry tối đa ~10% request + 0.5/s: lớp chặn cuối khi breaker cấu hình sai (thí nghiệm 1: G2 retry_ratio 0.495 → 0.058 khi bật budget). |
| breaker.window_size / min_calls | 20 / 10 | Ở 4 lượt/s alpha nhận ~8 call/s → cửa sổ 20 kết quả ≈ 2.5 s dữ liệu: đủ nhanh để G2 recovery = 0 s, min_calls 10 tránh trip vì 1–2 lỗi lẻ (base_error_rate 0.2%). |
| breaker.failure_rate_threshold | 0.5 | G3 (30% lỗi 500) **không** nên mở breaker — retry tại chỗ là đủ (availability 1.0); G2/G6 (≈100% 529/503/timeout) mở ngay. 0.5 nằm giữa hai trường hợp. |
| breaker.slow_call_s | 4.0 (đã hạ từ 5.0) | Với 5.0, biến thể G4 +3.5 s (alpha ≈ 4.7 s/call) không bị coi là chậm → p95 10.8 s (FAIL). Với 4.0: p95 5.7 s. Healthy beta p99 ≈ 4.1 s và slow_rate_threshold 0.6 nên provider khỏe không bao giờ trip vì chậm (20 seed sạch). |
| breaker.open_duration_s | 10.0 | Ngắn so với sự cố (180–300 s) nhưng mỗi 10 s chỉ tốn ≤ 2 probe; sau khi alpha hồi phục, quay lại trong ≤ 10 s + 2 probe → `primary_share_after_restore` = 1.0 ở G2/G4/G6. |
| breaker.half_open_max_probes | 2 (cần 2 thành công) | Không thundering herd lên provider đang hồi phục; 2 probe chậm/lỗi → mở lại ngay (G4: alpha còn chậm thì giữ OPEN). |
| min_time_left_s / quota_cooldown_s | 1.0 / 300 | Không gửi attempt không kịp hoàn thành; quota không tự hồi trong vài giây → nghỉ 5 phút, rồi 1 request thăm dò lại. |

Sơ đồ quyết định của `on_error`:

```
error ──classify──┬─ CLIENT_ERROR (400/401/422) ─ breaker.release ─► GiveUp("client_error")   (không retry, không failover)
                  ├─ QUOTA_EXHAUSTED (429 quota) ─ breaker.release ─► cooldown[p] = now+300s ─► failover ngay
                  ├─ RATE_LIMITED (429 + retry-after) ─ release ───► cooldown[p] = now+retry_after ─► failover ngay
                  │                                                   (hết ladder → chờ cooldown ngắn ≤5 s nếu kịp deadline)
                  └─ OVERLOADED / SERVER_ERROR / TIMEOUT ─ breaker.on_failure ─►
                         tries < 2 AND breaker CLOSED AND budget có token AND kịp deadline ?
                             ├─ có  ► Send(cùng provider, delay = full-jitter backoff)
                             └─ không ► failover sang provider kế tiếp (delay 0)
_route: bỏ qua provider đang cooldown hoặc breaker từ chối; ladder("tool") = [alpha, alpha-eu, beta] (không gamma-mini)
```

Chỉ có **một** tầng retry (client này), không chồng app × gateway × SDK.

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối** (breaker hỏng = `min_calls: 1e9`, `max_attempts_per_provider: 5`):

   | Cấu hình | G2 ampl | G2 retry_ratio | G2 avail | G2 recovery | G3 ampl | G3 retry_ratio |
   |---|---:|---:|---:|---:|---:|---:|
   | breaker hỏng + budget | 1.378 | 0.058 | 0.9983 | 0 s | 1.136 | 0.068 |
   | breaker hỏng, **không** budget | 1.675 | 0.495 | 0.9666 | 25 s | 1.162 | 0.159 |
   | breaker tốt, không budget | 1.019 | 0.003 | 1.0 | 0 s | 1.060 | 0.041 |
   | breaker tốt + budget (cấu hình nộp) | 1.019 | 0.003 | 1.0 | 0 s | 1.065 | 0.035 |

   Không có breaker, budget giữ retry ≤ ~10% (retry_ratio 0.058 so với 0.495) — đó chính là thứ ngăn retry storm: tải
   gửi vào alpha đang mất 75% GPU không bị nhân lên, nên không có vòng lặp metastable (recovery 0 s so với 25 s, và
   amplification 1.68 vượt ngưỡng 1.6). Khi breaker hoạt động đúng, budget gần như không bao giờ bị chạm (hai dòng cuối
   giống nhau) vì breaker cắt tải trước. Vẫn nên có cả hai: breaker bảo vệ *từng provider* dựa trên tín hiệu sức khỏe,
   còn budget là giới hạn *toàn cục* không phụ thuộc vào việc phân loại lỗi hay tham số breaker đúng — defense in depth
   cho đúng trường hợp breaker cấu hình sai, lỗi không được tính vào breaker, hay sự cố kiểu mới.

2. **`attempt_timeout_s: 30`** — G6: availability 1.0 → **0.9488 (FAIL)**, `give_up_reasons` từ `{}` thành
   `{deadline: 107, no_provider_available: 15}`; trong khi p95 lại *giảm* 6.09 → 5.48 s (survivorship bias: các lượt
   treo 30 s rồi chết không có mặt trong p95). Cơ chế: alpha-eu "hang" không trả lỗi; với timeout 30 s, một call tool bị
   alpha từ chối (503) sẽ treo trên alpha-eu cho tới hết deadline của user, không còn thời gian để sang beta; breaker
   alpha-eu cũng chỉ học được sau 30 s/kết quả. G4 gần như không đổi (1.0 / 5.61 s) vì ở G4 alpha vẫn *trả lời* (chỉ
   chậm thêm 5 s, ~6.2 s < 8 s), timeout không bao giờ kích hoạt — slow-call detection của breaker mới là thứ xử lý G4.
   Timeout đúng nên chọn từ **p99 lúc bình thường** (cận dưới, để không cắt oan) và **deadline của user trừ thời gian
   cần cho ít nhất một lần failover** (cận trên): 30 s − (1 failover ≈ 2–5 s + call thứ hai của lượt) → ~8 s.

3. **Bỏ lọc tool calling khỏi `ladder()`** — G6: `incompatible_routes = 235`, availability 1.0 → 0.9015 (FAIL).
   Nói với PM: "Trong lúc nhà cung cấp chính sập, 235 khách đang yêu cầu hoàn tiền đã bị chuyển sang một model rẻ không
   biết dùng công cụ (tra đơn, hoàn tiền). Model đó vẫn trả lời rất nhanh nên trên dashboard trông như 'thành công',
   nhưng thực ra không làm được việc — khách nhận câu trả lời sai/vô nghĩa, tức ~10% lượt trong sự cố. Thà gửi họ sang
   beta (đắt hơn 40%) còn hơn trả lời rác."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,496,700 cents | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Thiết kế: `idempotency_key = refund-agent:{session_id}:{step}` (ổn định qua retry và restart), checkpoint
`wf:{session_id}:step:{step}` ghi **sau** khi step thành công, khi chạy lại thì nạp checkpoint và bỏ qua step đã xong;
`_retry` chỉ bắt `ServiceError`/`ServiceTimeout`, không bao giờ bắt `WorkerCrash`.

- **Chỉ idempotency key chưa đủ:** không trùng tiền, nhưng worker mới chạy lại *mọi* bước từ đầu → gọi lại LLM
  classify/summarize (wasted LLM/crash cao, tốn tiền) và phải có đúng `refund_id` cũ để gửi email; mọi bước không có
  key (LLM, đọc đơn) đều làm lại.
- **Chỉ checkpoint chưa đủ:** crash xảy ra *sau* khi payment đã chạy nhưng *trước* khi checkpoint được ghi (hoặc
  `ServiceTimeout` — tiền đã đi nhưng response mất) → không có checkpoint, worker mới hoàn tiền lần nữa → trùng.
  Checkpoint chỉ nhớ những gì *mình biết* đã xong; key mới chống được vùng "đã xảy ra nhưng mình không biết".
- **Ghi checkpoint trước khi gọi payment:** nếu worker chết giữa lúc ghi và lúc gọi, worker sau thấy checkpoint, bỏ
  qua bước hoàn tiền → khách **không bao giờ nhận tiền** (lost refund) mà hệ thống lại báo "done".

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện (burn-rate) |
|---|---|---|---|---:|
| checkout_bug (15% / 45 phút) | page | page (1 phút) | page | 27 phút (hạn 35) |
| provider_outage (100% / 10 phút) | page | page (0 phút) | page | 4 phút (hạn 8) |
| deploy_blip (30% / 4 phút) | none | **page sai** | không page | – |
| slow_leak (1.5% / 2 ngày) | ticket | **bỏ sót** | ticket | 1337 phút (hạn 1800) |
| degraded_evening | page | page (214 phút) | page (rule 6h) | 291 phút |
| night_spike | none | **page sai** | không page | – |
| small_bump (12% / 20 phút) | none | **page sai** | không page | – |

Naive: 7 page, 3 false page, bỏ sót slow_leak → FAIL. Burn-rate: 3 page, 1 ticket, 0 false page → PASS.

Error budget 30 ngày: 25,901,755 request, cho phép 129,508.8 lỗi, thực tế 55,618 lỗi (availability 99.7853%) → **đã
tiêu 42.95%, còn 57.05%**. Không đóng băng release (`freeze_releases = false`), vì vẫn còn hơn nửa budget.

`small_bump` (20 phút × 12% lỗi): burn rate tức thời 24× nhưng cửa sổ 1h trung bình chỉ 12%·20/60 = 4% → burn 8 < 14.4,
và 6h → ~0.67% → burn 1.3 < 6. Nó chỉ tiêu 20·0.12 / (0.005·43,200) ≈ **1.1% budget tháng** và tự hết — đánh thức
người lúc 11 giờ trưa vì một việc không cần hành động gì là cách nhanh nhất để on-call bắt đầu phớt lờ page. Page chỉ
khi đốt ≥ 2% budget/giờ (14.4×) hoặc 5%/6h (6×).

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình: `reports/gameday/G2_overload_storm.png`; CSV: `reports/gameday/naive_G2_overload_storm_timeseries.csv`.
Mã lỗi theo thời gian lấy bằng cách bọc NaiveClient trong một lớp đếm `on_error` (không sửa file gốc).*

**Tóm tắt:** Lúc t=120 s, deployment alpha mất 75% GPU trong 3 phút. Client ShopAgent v1 xử lý mọi lỗi bằng cách retry
cùng provider qua ba tầng chồng nhau (tối đa 36 attempt/call, delay cố định 0.2 s, không jitter/budget/breaker/fallback),
biến ~8 call/s thành 110–214 attempt/s và giữ alpha ở trạng thái quá tải. Alpha khôi phục năng lực ở t=300 s nhưng hệ
thống chỉ hồi phục ở t≈455 s (recovery 155 s) — tỷ lệ call thành công bằng 0 suốt ~5.3 phút.

**Ảnh hưởng:** 1,285 / 2,362 lượt user thất bại (availability 45.6%); 0 call thành công từ t≈135 tới t≈450 (≈315 s).
3,481 call logic sinh ra 34,475 attempt (amplification 9.9×), chi phí/lượt thành công 4.627 so với 2.077 của client mới
(gấp 2.2×). Nếu tháng chỉ có sự cố này ở tải 4 lượt/s (10,368,000 lượt/tháng, budget 0.5% = 51,840 lượt lỗi), sự cố
tiêu **2.5% error budget tháng trong 8 phút**; tính riêng cửa sổ sự cố, tỷ lệ lỗi 54.4% gấp **109 lần** mức cho phép
(0.5%).

**Timeline** (giây ảo; attempts/s = trung bình 10 s, cột "ok/done" = call thành công / call kết thúc trong 10 s):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–119 | Bình thường | ~7–9 attempts/s, ok/done 74/74, 92/92; latency trung bình 1.3 s |
| 120 | alpha mất 75% năng lực (trigger) | service time ×4; latency TB 4.0 s, xuất hiện client_timeout đầu tiên |
| 130 | Hàng đợi đầy → 529 + timeout, ba tầng retry bắt đầu nhân tải | attempts 51.9/s; 345 lỗi 429, 114 lỗi 529, 38 timeout; ok/done 19/19 |
| 140 | Retry storm đạt đỉnh, vượt rate limit 20 rps | **214 attempts/s** (~27× bình thường); 1,934 lỗi 429 trong 10 s; ok/done **0/47** |
| 150–290 | Duy trì: 110–169 attempts/s, ~92% là 429, alpha luôn bão hòa | mỗi 10 s ~1,100–1,480 lỗi 429, ~165 lỗi 529, ~35 timeout; ok/done 0/31…0/60 |
| 300 | alpha hồi phục hoàn toàn | attempts giảm còn 38.8/s nhưng ok/done vẫn **0/18** |
| 310–350 | Hệ thống **vẫn sập** dù provider đã khỏe | ~15 attempts/s (= đúng năng lực alpha ~15 rps), **100% client_timeout** (115–154 mỗi 10 s), ok/done 0/10…0/43 |
| 360–410 | Đợt retry thứ hai: lượt mới + retry dồn lại | 78–130 attempts/s; 516–1,119 lỗi 429/10 s; ok/done vẫn 0 |
| 430–440 | Tải giảm dần do lượt cũ hết deadline 30 s và hết 36 attempt | ~13 attempts/s, toàn timeout, ok/done 0/48, 0/27 |
| 450 | Bắt đầu hồi phục | ok/done 45/70 (64%) — latency TB 7.7 s |
| 455–470 | Hồi phục (≥ 95% trong 30 s ⇒ recovery_s = 155) | ok/done 115/123, 139/139 |

**Trigger vs root cause:**
- *Trigger:* alpha mất 75% GPU trong 180 s (sự kiện hạ tầng — sẽ còn xảy ra, không thể ngăn hoàn toàn).
- *Root cause:* lớp client không phân biệt loại lỗi và có ba tầng retry chồng nhau (4 × 3 × 3 = 36 attempt/call) với
  delay cố định, không jitter, không tôn trọng retry-after, không retry budget, không circuit breaker và không fallback
  sang alpha-eu/beta. Mỗi lỗi sinh thêm tải lên đúng provider đang quá tải → khuếch đại 9.9× → vòng lặp metastable.

**Vì sao provider đã hồi phục mà hệ thống vẫn sập (metastable failure):** provider là hàng đợi và *không hủy* request
khi client timeout. Lúc t=300, hàng đợi của alpha đã chứa đầy request mà client đã bỏ (timeout 8 s) — server vẫn làm
chúng. Mỗi request mới phải chờ gần 10 s sau đống "việc vô ích" đó → client timeout ở 8 s → retry → lại xếp hàng thêm
một request vô ích. Ở t=310–350 alpha nhận ~15 attempt/s, đúng bằng năng lực của nó (18 slot / 1.2 s), nhưng **goodput
= 0**: 100% công suất dành cho request mà không ai chờ nữa. Vòng lặp tự duy trì: *timeout → retry → hàng đợi dài →
timeout*. Nó chỉ tắt khi các lượt cũ hết deadline 30 s / hết 36 attempt và tải thực giảm xuống dưới năng lực (t≈450).

**Điều gì giúp / điều gì làm tệ hơn:**
- *Giúp:* deadline 30 s của user và giới hạn 36 attempt cuối cùng cũng chặn vòng lặp; rate limit 20 rps của alpha
  chặn phần lớn storm bằng 429 rẻ (0.05 s) thay vì để nó vào hàng đợi GPU.
- *Làm tệ hơn:* bỏ qua retry-after (26,298 lỗi 429 bị retry sau 0.2 s); delay cố định khiến các retry đồng bộ; retry
  cả khi đã gần hết deadline; không chuyển sang alpha-eu/beta vốn đang rảnh; timeout 8 s nhỏ hơn thời gian chờ hàng
  đợi tối đa 10 s nên client bỏ những request mà server vẫn sẽ làm.

**Action items:**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay ba tầng retry bằng **một** tầng (ResilientClient): phân loại lỗi, full jitter, tôn trọng retry-after, retry budget 10% | prevent | Nhóm platform LLM (gateway) | CI chạy `make gameday`: G2 amplification ≤ 1.2 (hiện 1.019), G1 `retry_after_violations = 0` trên 3 seed |
| 2 | Circuit breaker sliding-window (lỗi + slow call) cho từng deployment và fallback ladder alpha → alpha-eu → beta theo khả năng tool | mitigate | Nhóm platform LLM | G2 recovery_s ≤ 30 (hiện 0), availability ≥ 0.95 (hiện 1.0), primary_share_after_restore ≥ 0.8 (hiện 1.0) |
| 3 | Dashboard + alert cho **amplification** (attempts/call logic) và goodput theo provider; page khi amplification > 2 trong 5 phút | detect | Nhóm SRE/on-call | Chạy lại băng G2 với client naive trên staging: alert phải bắn trong ≤ 60 s sau t=130 |
| 4 | Gateway hủy (cancel) request phía server khi client đã timeout / truyền deadline xuống provider | prevent | Nhóm platform LLM + liên hệ provider | Trên G2 naive, khoảng t=300–350 phải có goodput > 0 (hiện 0) |
| 5 | Game Day định kỳ hằng quý với băng ẩn (lệch thời gian, đổi cường độ) | detect | SRE lead | 6/6 băng PASS trên 3 seed mới mỗi quý |

*Blameless:* sự cố không do một người hay một lần deploy; mỗi tầng retry đều hợp lý khi được thêm vào riêng lẻ. Lỗ hổng
nằm ở chỗ hệ thống không có nơi nào nhìn tổng số retry và không có test tải dưới sự cố.

## 7. Một điểm yếu còn lại

**Cả 4 provider cùng chậm/quá tải cùng lúc** (hoặc beta gánh toàn bộ khi family A sập): breaker của mọi provider đều
mở vì slow-call → `_route` hết ladder và trả `GiveUp("no_provider_available")` dù các provider vẫn trả lời được (chậm
còn hơn không). Ở G5 phần quota, beta gánh 100% traffic với cost/turn 2.504 (so với ~2.08 bình thường, +20%) — không có
cơ chế nào cảnh báo chi phí. Cách sửa: (1) khi mọi breaker đều OPEN vì *slow* (không phải lỗi), vẫn cho call đi tới
provider ít tệ nhất ("least-bad routing") với timeout = thời gian còn lại; (2) load shedding có ưu tiên: lượt FAQ trả từ
cache/stale, chỉ lượt hoàn tiền đi provider; (3) alert khi cost/turn > 1.2× baseline trong 15 phút.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`, seed 25).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.0 | 0 | 0 | 0 | 1.386 | FAIL |
| cached — cấu hình starter (0.80 / 600 / 0) | 0.8469 | 0.8587 | 0.742 | 0 | 181 (`other_question`) | 80 | 0.921 | FAIL |
| cached (của bạn) | 0.9117 | 0.9645 | 0.568 | 189 | 0 | 2 | 0.971 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Paraphrase cùng ý có cosine ~0.95–0.98, các cặp "gần mà khác" (refund policy ↔ refund time, đổi size ↔ đổi màu, ship nội thành ↔ ngoại tỉnh, thanh toán ↔ COD) ~0.83–0.86. 0.80 → 143 `other_question`; 0.92 → 0 sai, hit 0.568; 0.97 → 0 sai nhưng hit tụt còn 0.468 (FAIL ≥ 0.5). Sweep 10 seed: 0.90/0.92/0.94 đều 0 sai — 0.92 nằm giữa vùng an toàn. |
| ttl_s | 25 | Chỉ TTL giới hạn `outdated_answers` (thay đổi im lặng). Sweep 6 băng tự chế (lệch thời điểm) × 7 seed: TTL 20 → outdated tối đa 8, hit tối thiểu 0.506; **25 → 11 / 0.539**; 30 → 14 / 0.559; 35 → 19 (FAIL). Trên băng công khai 10 seed: TTL 40 → 19 outdated (FAIL). 25 s để lại biên cho cả hai ngưỡng (15 và 0.5). |
| max_stale_s | 300 | Sự cố sập toàn bộ 120 s; entry lưu ngay trước sự cố phải còn dùng được tới cuối: cần ≥ 120 s; 300 s chịu được sự cố 5 phút. `max_stale_s: 0` → faq_availability 0.8547 (FAIL), với 300 → 0.9645 (189 câu stale). Stale chỉ phục vụ khi provider đã bỏ cuộc, và webhook vẫn xóa entry nên không phục vụ nội dung đã biết là sai. |
| max_entries | 500 | ~18 intent chia sẻ × vài paraphrase ≪ 500; `store` thay thế entry cùng ý nên cache không phình. LRU chỉ là chặn trên bộ nhớ. |

Thí nghiệm:

1. **Ngưỡng** (0.80 / 0.92 / 0.97): hit 0.563 / 0.568 / 0.468; wrong 143 / 0 / 0; cost/turn 1.034 / 0.971 / 1.039.
   Ở 0.80 toàn bộ lỗi là `other_question`: "Bao lâu thì nhận được tiền hoàn?" được trả bằng câu trả lời của "Chính sách
   hoàn tiền thế nào?" (cosine ~0.84), tương tự đổi size ↔ đổi màu, phí ship nội thành ↔ ngoại tỉnh. Lưu ý 0.80 không
   cho hit cao hơn — các câu sai cũng thất bại, nên còn đắt hơn.
2. **TTL** (30 / 60 / 300, seed 25): outdated 2 / 3 / **95**; hit 0.592 / 0.652 / 0.731. Webhook chỉ báo những thay
   đổi được *thông báo* (`policy/refund` lúc t=150 → invalidate). Bảng phí ship đổi lúc t=480 **không có webhook**:
   client không có tín hiệu nào cả, nên mỗi entry cũ sẽ được phục vụ cho tới khi hết tươi — TTL là cận trên duy nhất
   cho tuổi của một câu trả lời. (Seed 25 tình cờ ít outdated ở TTL 60; trên nhiều seed/lệch thời điểm, TTL > 30 vượt 15.)
3. **Stale-if-error** (`max_stale_s: 0`): faq_availability 0.9645 → 0.8547, hit 0.568 → 0.458, stale_served 189 → 0.
   Không có cửa sổ stale, mọi câu FAQ chung trong 120 s sập đều thất bại thay vì nhận câu trả lời cũ vài phút.

Trả lời:

- **Vì sao không cache câu hỏi cá nhân, kể cả theo từng user?** Câu hỏi của hai user chỉ khác số đơn nên cosine ~0.99
  — cache ngữ nghĩa sẽ trả đơn #48213 của người này cho người kia (rò rỉ dữ liệu, `other_user`). Kể cả khi tách theo
  user, câu trả lời phụ thuộc trạng thái đơn/hoàn tiền/điểm thay đổi từng phút mà không có webhook nào báo, nên câu trả
  lời cache gần như luôn lỗi thời ("đơn đang giao" khi đã giao xong); tỷ lệ trùng lặp lại thấp (mỗi user hỏi đơn của
  mình vài lần) nên lợi ích gần như bằng 0 so với rủi ro.
- **Alert gì khi dashboard xanh nhờ cache?** Không alert trên availability phía user mà trên **tín hiệu phía provider**:
  (1) tỷ lệ `stale_served` / FAQ (bình thường = 0; trong sự cố C1 = 189 câu) — > 1% trong 5 phút là page; (2) tỷ lệ
  thành công của provider / số breaker OPEN / `give_up_reasons` theo phút; (3) availability của lượt **không** cache
  được (tool call, câu hỏi cá nhân — trong C1 tổng availability vẫn chỉ 0.9117 dù faq 0.9645). SLO nên tính câu trả lời
  stale là "degraded", không phải "good".
