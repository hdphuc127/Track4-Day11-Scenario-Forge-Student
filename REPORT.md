MSSV: 2A202602796

# Lab 11 · Scenario Forge — báo cáo

Họ và tên: **Hồ Đăng Phúc**

Thị trấn của bạn (`meta.json: _run.town`): **C**

## 1. Phát lại log có trung thực không? (bước A)

| `replay.json` | giá trị |
|---|---|
| `n` / `n_used` | 400 / 399 |
| `dropped_human_crash_logs` (người lái gây va chạm) | 1 |
| `replay.crashes` (planner trên log phát lại) | 112 = `rear` 97 + `side` 15 |
| `reactive.crashes` (thế giới phản ứng) | 0 |
| `crash_in_both` | 0 |

Toàn bộ va chạm khi phát lại là va chạm **giả**: không kịch bản nào va chạm trong thế giới phản ứng (`crash_in_both = 0`). Phần lớn là `rear` vì planner lái khác người lái trong log (phanh sớm hơn, chậm hơn) trong khi xe phía sau vẫn chạy theo đúng quỹ đạo đã ghi và không "thấy" xe ta, nên lao vào đuôi. Ngoài đời xe đó sẽ phanh theo. `world.json` cũng cho thấy điều này: phát lại có `ego|TG` tại t = 14.8 s (`fake: true`), còn `reactive_events = []`.

Ví dụ `replay_only[0]` (kịch bản 0, `contact: rear`): xe cắt làn vào ở `cutV` ≈ 13.9 m/s với `cutGap` ≈ 9.4 m, và planner giảm tốc để giữ khoảng cách. Xe bám đuôi (`tgT` ≈ 1.18 s) phát lại đúng log nên đâm vào đuôi xe ta. Trong thế giới phản ứng, xe đó phanh theo và `min_ttc_reactive` = 2.63 s, tức không hề nguy hiểm.

## 2. Kịch bản bất khả thi (bước B)

Các luật trong `lab11/feasible.py: is_feasible`:

| Luật | Căn cứ vật lý |
|---|---|
| `(π/cutDur)² · 3.5/2 > μg` → loại | Đổi một làn 3.5 m theo hình sin trong `cutDur` giây cần gia tốc ngang đỉnh như vậy. Vượt μg (ma sát lốp–đường) thì xe trượt, không thể cắt làn. |
| `cutBrake > μg` → loại | Không xe nào phanh mạnh hơn giới hạn bám của lốp. |
| `pedV > 2.5 m/s` → loại | Người đi bộ băng qua đường khoảng 1–1.5 m/s, chạy nhanh khoảng 2.5 m/s. Nhanh hơn là "người chạy nước rút" chứ không phải người đi bộ. |
| `cutV ∉ (0, 22]` → loại | Xe cắt làn phải đang chạy, và không vượt quá tốc độ hợp lý trên phố này. |

Trong `hunt.json`, cuộc săn `crosswalk_block` sinh nhiều kịch bản bất khả thi nhất (`infeasible = 7`, so với 3 / 1 / 3 của bus_bypass, cut_in và rear_ended). CEM bị kéo về phía đó vì điểm số chỉ thưởng "gần va chạm": người đi bộ càng nhanh, hoặc xe càng phanh hay lạng gắt, thì càng dễ va chạm. Vì vậy phân phối đề xuất trôi dần ra rìa vật lý. Nếu **đếm** chúng là va chạm, ước lượng sẽ bị thổi phồng bởi những va chạm mà thế giới thật không sinh ra (chúng nằm ngoài giá đỡ của phân phối tự nhiên). Với IS, những mẫu đó có trọng số ≈ 0 nhưng vẫn làm CEM lệch hướng. Với MC, chúng làm tăng `k` (`mc.json: infeasible = 15` lần, gấp 2.5 lần số va chạm thật là 6).

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k = **6**, n = **3000**, p̂ = **2.00e-3**, Wilson 95 % = **[9.17e-4, 4.36e-3]**. Cả 6 va chạm đều là `bus_bypass` (`modes`).

Để khoảng tin cậy còn ±20 % của p, cần n ≈ 1.96² (1−p) / (0.2² · p) ≈ 3.84 / (0.04 × 0.002) ≈ **48 000 lần chạy** (khoảng 16 lần hiện tại, hay chừng 120 va chạm).

Các kiểu **chưa hề xuất hiện** là `cut_in`, `crosswalk_block`, `rear_ended` và `other`, mỗi kiểu k = 0. Không được báo "p = 0". Con số đúng là cận trên: Wilson `hi` = **1.28e-3** (`mc.json: per_mode.*.hi`), xấp xỉ quy tắc 3/n = 1e-3.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json` (n = 600 mỗi kiểu và mỗi seed):

| seed | kiểu | p | hits | ess | top_share |
|---|---|---|---|---|---|
| 0 | bus_bypass | 1.58e-3 | 32 | 125.0 | 0.29 |
| 0 | crosswalk_block | ≈0 (<1e-6) | 56 | 128.1 | 0.17 |
| 0 | cut_in / rear_ended | 0 | 0 | 125.1 / 128.4 | 0 |
| 1 | bus_bypass | 2.27e-3 | 34 | 145.2 | 0.24 |
| 1 | crosswalk_block | 2e-6 | 79 | 132.1 | **0.55** |
| 1 | cut_in / rear_ended | 0 | 0 | 119.5 / 109.1 | 0 |
| 2 | bus_bypass | 1.35e-3 | 28 | 159.5 | 0.24 |
| 2 | crosswalk_block | 1e-6 | 69 | 119.1 | 0.27 |
| 2 | cut_in / rear_ended | 0 | 0 | 119.2 / 109.2 | 0 |

Tổng theo seed (`total`): 1.58e-3, 2.27e-3 và 1.35e-3. Ba seed đồng ý với nhau: các khoảng [lo, hi] chồng lên nhau, và cả ba nằm trong Wilson của MC [9.17e-4, 4.36e-3] quanh p̂ = 2.0e-3.

`crosswalk_block` có nhiều hits (56–79) nhưng p chỉ khoảng 1e-6. Lý do là các va chạm nằm ở vùng mà phân phối tự nhiên gần như không bao giờ sinh ra. Ở seed 1, `top_share = 0.55` nghĩa là một mẫu chiếm hơn nửa tổng trọng số, nên con số thực ra chỉ là w của mẫu đó: thêm hay bớt một mẫu là p đổi cả bậc. ESS ≈ 132 nghe lớn, nhưng ESS tính trên mọi mẫu, kể cả mẫu không va chạm. Dấu hiệu đúng của độ suy biến là `top_share` trong số các hit.

Ngân sách: `budget` = **16 800** lần chạy cho cả hunt và estimate, cho mọi kiểu. MC để đạt ±20 % cho riêng bus_bypass cần khoảng 48 000 lần (câu 3). Với `crosswalk_block` (p ~ 1e-6), MC cần khoảng 10⁸ lần mới thấy được vài va chạm. IS rẻ hơn hàng nghìn lần ở đó.

Phần `other` lấy từ `mc.json` (`estimate.json: other = {k: 0, n: 3000, hi: 1.28e-3, source: mc.json}`). Ta không săn `other` vì nó không có một cơ chế chung để CEM nhắm vào (không có điểm số "gần va chạm" riêng). Nó là phần dư. MC không thiên lệch là cách trung thực duy nhất để chặn trên nó.

## 5. Các kiểu thất bại (bước C2 + 3D)

`hunt.json: found` của từng cuộc săn:

| cuộc săn | found |
|---|---|
| bus_bypass | bus_bypass 154 |
| cut_in | bus_bypass 32 (**không** có cut_in) |
| crosswalk_block | crosswalk_block 44, bus_bypass 2 |
| rear_ended | bus_bypass 9, crosswalk_block 2 (**không** có rear_ended) |

- **bus_bypass**: xe ta lách qua xe buýt đang đỗ (x < 112, chậm). Xe buýt che tầm nhìn nên xe ta không thấy xe ngược chiều hoặc xe đang lách, và đâm vào hông. Ảnh: `evidence/shots/bus_bypass.png`.
- **crosswalk_block**: người đi bộ bước ra từ sau xe buýt tại vạch. Planner chỉ thấy người đó khi đã quá gần nên không dừng kịp. Ảnh: `evidence/shots/crosswalk_block.png`.
- **Không tìm thấy**: `cut_in` và `rear_ended`. Mỗi cuộc săn chạy 8 vòng × 300 mẫu cộng 3 × 600 mẫu ước lượng mà vẫn 0 hit (`estimate.json: upper_zero_hits = 0.025`). Tôi tin chúng hiếm ở mức vừa phải: CEM có thể kẹt cục bộ. Trong cuộc săn `cut_in`, `stopped_tightening = False` và `level` = −2.92 vẫn còn xa 0.

Nếu chỉ chạy **một** cuộc săn chung, nó sẽ bị `bus_bypass` (kiểu dễ nhất) hút hết. Ngay cả cuộc săn nhắm vào cut_in cũng chỉ trả về 32 bus_bypass. Một cuộc săn chung sẽ báo "chỉ có một kiểu thất bại" và bỏ sót crosswalk_block, kiểu chỉ tìm được khi có một cuộc săn riêng với điểm số riêng.

## 6. Sửa planner (bước E + 3D)

Policy được chọn: **`{"cutinLat": 3.0}`** (`fix.json: chosen.policy`).

| | fail | fail_hi | fail_hi_seen | progress (m) | harsh (lần phanh gấp) |
|---|---|---|---|---|---|
| gốc, TRAIN | 1.94e-3 | 4.26e-3 | 4.26e-3 | 211.5 | 8.88 |
| chọn, TRAIN | 1e-6 | 3e-6 | 3e-6 | 211.2 | 8.16 |
| gốc, TEST | 1.39e-3 | 4.08e-3 | 2.80e-3 | 213.4 | 8.34 |
| chọn, TEST | ≈0 | **3.88e-2** | 1e-6 | 213.2 | 8.05 |

Trên TEST, bản sửa có 0 hit bus_bypass (gốc có 12 hit, p = 1.38e-3). Quãng đường gần như giữ nguyên (213.2 so với 213.4 m), và số lần phanh gấp giảm.

`fail_hi` của bản sửa (3.88e-2) lớn hơn `fail` của gốc dù bản sửa tốt hơn thật. Lý do là bus_bypass rơi vào `unseen`: 0 hit trên 400 mẫu, nên cận trên phải dùng `upper_zero_hits = 0.0375` (quy tắc 3/n kiểu "chưa thấy không có nghĩa là bằng 0"). Cận trên đó phản ánh việc **thiếu dữ liệu**, không phải rủi ro thực. `fail_hi_seen` (1e-6) mới so được với gốc.

Núm "nghe thì an toàn hơn" nhưng phản tác dụng: **`T = 2.0`** (giữ khoảng cách thời gian xa hơn). `fail` tăng từ 1.94e-3 lên **2.16e-3**, với bus_bypass 23 hit so với 15, và crosswalk_block 73 hit so với 42. Thế giới phản ứng lại khoảng trống lớn hơn: xe khác chen vào khe đó (cut-in thường xuyên hơn), và xe ta bị kẹt sau xe buýt lâu hơn rồi mới lách, đúng vào lúc xe ngược chiều tới. Tương tự, `cautionV = 6.5` (chậm hơn) vẫn có 43 hit crosswalk và số lần phanh gấp tăng lên 10.9.

Ảnh có bóng mờ: `evidence/shots/fix_ghost.png` (chương 9, B-01, t = 12.55 s). Xe đặc là policy gốc lách xe buýt và va chạm. Bóng mờ phía sau là bản sửa `cutinLat: 3.0` chạy cùng kịch bản: nó chờ phía sau và không va chạm.

## 7. Từ mô phỏng ra đường thật

1. **Khe chấp nhận và tốc độ của xe lách hoặc chuyển làn** (`cutGap`, `cutV`, `tgT`). Cần thay bằng phân phối khe chấp nhận thực tế từ dữ liệu quỹ đạo như highD/inD, Waymo Open Motion, hoặc video giao thông tại chỗ ở Việt Nam. Nếu thực tế người ta chấp nhận khe ngắn hơn (đô thị Việt Nam), p bus_bypass của tôi đang **thấp hơn** thật.
2. **Hành vi người đi bộ sau xe buýt** (`pedLook`, `pedV`, `pedGap`). Tỉ lệ người nhìn trước khi bước ra đo được qua quan sát tại điểm dừng xe buýt (nghiên cứu hành vi qua đường). Nếu ít người nhìn hơn giả định, p crosswalk_block (~1e-6) bị **đánh giá thấp** nhiều bậc.
3. **Ma sát đường `mu`** (mặc định 0.8, tức đường khô). Cần tỉ lệ thời gian đường ướt theo dữ liệu khí tượng địa phương, và μ đo bằng xe thử. Mùa mưa có μ ≈ 0.4–0.5: quãng phanh gần gấp đôi, nên mọi p đều **tăng**, và luật khả thi `cutBrake ≤ μg` cũng chặt hơn.

## 8. Giới hạn

- Thế giới chỉ có **một con phố** với một xe buýt và một vạch qua đường. Xác suất này không áp dụng cho giao lộ, vòng xoay hay đường cao tốc.
- Mọi người lái khác là **IDM**, lịch sự và có thể đoán trước. Không có ai say rượu, mất tập trung hay vượt đèn đỏ. Đuôi phân phối thật dày hơn.
- Cảm biến **hoàn hảo** ngoài phần bị che: không nhiễu, không trễ, không bỏ sót vật thể. Thất bại do nhận thức không được tính.

Nếu có thêm một tuần, tôi sẽ đo độ nhạy của p theo ba giả định ở câu 7 (quét `mu` và `pedLook`). Tôi cũng sẽ chạy thêm cuộc săn `cut_in` và `rear_ended` với ngân sách lớn hơn để biết 0 hit là "hiếm" hay "CEM kẹt".

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** một người băng ngang giữa block, không có vạch (`ped`, side `S`, tức bước ra từ lề phải ngay trước xe ta), kích hoạt khi xe ta còn cách 10–40 m. Ngoài đời, đây là cảnh rất quen ở phố Việt Nam: người bước xuống lòng đường để qua bên kia, hoặc ra bắt xe ôm, giữa hai ngã tư.
- **Căn cứ:** planner chỉ giảm tốc ("caution") gần vạch qua đường và xe buýt (`cautionRange`). Giữa block nó chạy gần `v0` = 12.5 m/s và không có logic dự đoán người đi bộ ở chỗ không có vạch, nên khi người bước ra ở 10–20 m thì nó phản ứng muộn.
- **Giả định** (`hazard.json: why`):
  - `space`: `h0_x` 95–160 m (đoạn đường xe ta thực sự đi qua trong 24 s); `h0_v` 0.8–2.2 m/s (từ người già đi chậm đến chạy nhẹ; MUTCD dùng 1.07–1.2 m/s); `h0_trig` 10–40 m.
  - `exposure` = 0.02 mỗi chuyến: ước chừng 1 lần mỗi 50 chuyến ngắn có người bước ra sát đầu xe. Đây là giả định, cần dữ liệu dashcam hoặc near-miss để thay.
- **Kết quả** (`mine.json`, `accepted` = **0.997**, tức 1495/1500 khả thi):

  | | crashes | avoidable | unavoidable | unattributed | low_speed | P(lỗi \| rắc rối) [Wilson 95 %] | per_trip (upper) |
  |---|---|---|---|---|---|---|---|
  | baseline | 33 | **30** | 1 | 2 | 0 | 2.01e-2 [1.41e-2, 2.85e-2] | 4.01e-4 (5.70e-4) |
  | chosen `cutinLat: 3.0` | 31 | **30** | 1 | 0 | 0 | 2.01e-2 [1.41e-2, 2.85e-2] | 4.01e-4 (5.70e-4) |

  `avoidable_foreknown = 30 = avoidable`, nên ở đây không có chi phí do bị che tầm nhìn: người đi bộ luôn nhìn thấy được, và planner chỉ đơn giản là không phanh đủ sớm. So với lách xe buýt: `mc.json: p` = 2.0e-3 mỗi kịch bản thị trấn, còn rắc rối này là 4.0e-4 mỗi chuyến. Như vậy nó nhỏ hơn khoảng 5 lần, nhưng **bản sửa ở bước E không cứu được nó** (30 → 30). Đây là một rắc rối "hay": tránh được, nhưng planner vẫn dính.
- **Cảnh xem lại** (`mine.json: exact.planner = []`): cảnh gốc trong `hazard.json` (x = 130, v = 1.5, trig = 25) không va chạm. Va chạm chỉ xuất hiện khi quét ở trig ngắn và người đi nhanh.
- **Hình:** `evidence/shots/hazard_jaywalker.png` (chương 9, GB-01 từ `mine`, lúc va chạm t = 17.95 s). Người đi bộ bước ra từ lề phải, giữa block, và xe ta đâm thẳng vào. Policy gốc và bản sửa đều va chạm ở 17.9 s.
- **Bạn tự lái thì sao:** khi biết trước người sẽ bước ra, tôi nhả ga và giữ tốc độ khoảng 8 m/s khi đi ngang người đứng ở lề, nên phanh kịp. Planner thì giữ 12.5 m/s cho tới khi người đã ở trong làn.
- **Cách sửa:** không núm nào trong `policy_candidates.json` sửa được rắc rối này: `cutinLat` không liên quan, còn `cautionV` và `pedBuffer` chỉ có hiệu lực quanh vạch qua đường. Planner cần thêm **logic**: theo dõi mọi người đi bộ đứng ở lề trong tầm cảm biến, kể cả ngoài vạch. Nếu có người đang đứng gần mép đường hoặc hướng mặt ra đường, cần giảm tốc sao cho quãng phanh v²/(2·b) nhỏ hơn khoảng cách tới người đó (giới hạn tốc độ theo tầm nhìn), và dự đoán quỹ đạo cắt ngang làn để phanh ngay khi người đó bước xuống chứ không đợi họ vào hẳn làn.
