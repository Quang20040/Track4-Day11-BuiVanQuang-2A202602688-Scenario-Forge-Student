MSSV: 202602688

# Lab 11 · Scenario Forge — báo cáo

Thị trấn của bạn (`meta.json: _run.town`): D

---

## 1. Phát lại log có trung thực không? (bước A)

### Bảng số liệu đối chiếu (`replay.json`)

| Chỉ số | Giá trị | Trích xuất từ `evidence/replay.json` |
|---|---|---|
| Tổng kịch bản khảo sát ($n$) | 400 | `replay.json: n = 400` |
| Kịch bản người lái gây va chạm bị loại | 0 | `replay.json: dropped_human_crash_logs = 0` |
| Kịch bản đưa vào kiểm thử ($n_{\text{used}}$) | 400 | `replay.json: n_used = 400` |
| Va chạm khi chạy lại trên log phát lại (`replay`) | 82 | `replay.json: replay.crashes = 82` |
| – Tiếp xúc va chạm sau (`rear`) | 79 | `replay.json: replay.contact.rear = 79` |
| – Tiếp xúc va chạm hông (`side`) | 3 | `replay.json: replay.contact.side = 3` |
| – Tiếp xúc va chạm trước (`front`) | 0 | `replay.json: replay.contact.front = 0` |
| Va chạm trong thế giới phản ứng (`reactive`) | 0 | `replay.json: reactive.crashes = 0` |
| Va chạm xuất hiện ở cả hai môi trường (`both`) | 0 | `replay.json: crash_in_both = 0` |

### Giải thích kỹ thuật

1. **Vì sao phần lớn va chạm của "phát lại" là `rear` và là va chạm giả (simulation artifact)?**
   - Trong chế độ phát lại log (`others = "replay"`), các xe xung quanh (đặc biệt là xe bám đuôi TG) di chuyển theo quỹ đạo định sẵn đã ghi nhận từ hành trình của người lái con người trong quá khứ.
   - Planner tự lái của xe ta (ego) có phong cách điều khiển khác người lái thật: nó giảm tốc sớm hơn và thận trọng hơn khi tiếp cận vùng vạch đi bộ (`cautionV`) hoặc khi nhận diện xe cắt làn C2.
   - Tuy nhiên, trong log phát lại, xe bám đuôi TG là tác nhân **không phản ứng** (non-reactive): nó không "biết" xe ta đang phanh sớm hơn mà vẫn tiếp tục lao tới theo đúng gia tốc và vận tốc ghi trong log của con người, dẫn đến việc húc thẳng vào đuôi xe ta (`rear`).
   - Đây là va chạm **giả** (artifact) vì trong thế giới mô phỏng phản ứng (`reactive`), xe TG lái bằng mô hình IDM thực thụ: khi thấy khoảng cách tới xe ta giảm, TG lập tức tự động phanh theo IDM, giữ khoảng cách an toàn và không hề xảy ra va chạm nào (`reactive.crashes = 0`).

2. **Phân tích một kịch bản trong `replay_only` (`scenario = 1`):**
   - Trích xuất: `replay.json: replay_only[0]` (kịch bản số 1, `contact: "rear"`, `min_ttc_reactive = 2.3704` s, thông số: `cutT0 = 2.61`, `cutDur = 4.85`, `cutV = 12.05`, `tgT = 1.39`).
   - Diễn biến: Ở kịch bản này, xe ta nhận diện xe C2 nhập làn từ làn ngoài E2 vào làn E1 nên giảm tốc để nhường đường một cách an toàn. Trong thế giới phản ứng, xe TG phía sau phản ứng bằng cách phanh tương ứng, TTC nhỏ nhất đo được duy trì ở mức an toàn $2.37$ s. Ngược lại, trên log phát lại, xe TG giữ nguyên vận tốc $12.2$ m/s của người lái cũ và đâm thẳng vào đuôi xe ta tại khung hình số 296 ($t = 14.8$ s).

---

## 2. Kịch bản bất khả thi (bước B)

### Bốn luật trong `is_feasible` và căn cứ vật lý

1. **Gia tốc ngang khi chuyển làn:** $a_{\text{lat}} = \left(\frac{\pi}{\text{cutDur}}\right)^2 \cdot \frac{3.5}{2} \le \mu g$.
   - *Căn cứ vật lý:* Quỹ đạo chuyển làn ngang $3.5$ m (chiều rộng một làn xe) theo hàm cosin trong thời gian `cutDur` đạt gia tốc ngang cực đại $a_{\text{lat}} = \frac{\pi^2 \cdot 1.75}{\text{cutDur}^2}$. Theo định luật ma sát Coulomb và giới hạn vòng tròn ma sát của lốp (friction circle), lực bám ngang tối đa mặt đường cung cấp là $F_{\text{lat}} \le \mu m g$. Nếu $a_{\text{lat}} > \mu g$, xe sẽ mất lực bám ngang, văng đuôi hoặc mất lái (spin-out).
2. **Gia tốc phanh sau nhập làn:** $\text{cutBrake} \le \mu g$.
   - *Căn cứ vật lý:* Lực phanh dọc tối đa do ma sát giữa lốp xe và mặt đường tạo ra bị giới hạn bởi $\mu m g$. Gia tốc hãm phanh không thể vượt quá giới hạn ma sát trượt $\mu g$ nếu không có va chạm cản trở bên ngoài.
3. **Giới hạn tốc độ người đi bộ:** $\text{pedV} \le 2.5\text{ m/s}$ ($9\text{ km/h}$).
   - *Căn cứ vật lý:* Theo tiêu chuẩn công thái học và cơ sinh học người đi bộ (FHWA/HCM), vận tốc đi bộ bình thường của con người dao động từ $1.0$ đến $1.5$ m/s, đi bộ nhanh đạt $1.8$ m/s. Vận tốc $> 2.5$ m/s tương đương chạy nước rút (sprint), không phản ánh hành vi người đi bộ đô thị bình thường.
4. **Giới hạn vận tốc xe cắt làn:** $\text{cutV} \in (0, 22]\text{ m/s}$ ($0 < v \le 79.2\text{ km/h}$).
   - *Căn cứ vật lý:* Xe không thể lùi hoặc đứng yên trên làn chuyển tốc ($\text{cutV} \le 0$), và không thể chạy vượt quá vận tốc thiết kế tối đa cho phép của đường đô thị ($22$ m/s $\approx 80$ km/h).

### Cuộc săn sinh nhiều kịch bản bất khả thi nhất (`hunt.json`)

- Số kịch bản bất khả thi bị loại bỏ trong từng cuộc săn:
  - `hunt.json: bus_bypass.infeasible = 2`
  - `hunt.json: cut_in.infeasible = 2`
  - `hunt.json: crosswalk_block.infeasible = 7`
  - `hunt.json: rear_ended.infeasible = 21` (nhiều nhất)
- **Vì sao CEM bị kéo về phía bất khả thi?**
  - Điểm nguy hiểm của `rear_ended` được tính bằng $-\text{min\_ttc}(\text{TG}, \text{ego})$. Để tối đa hóa điểm nguy hiểm (TTC tiệm cận 0), thuật toán Cross-Entropy tự nhiên tối ưu hóa phân phối $q(u)$ vào các vùng tham số cực đoan nhất: xe chuyển làn C2 vừa vào làn đã phanh gấp vượt ngưỡng ma sát (`cutBrake` $> \mu g$) hoặc chuyển làn chớp nhoáng với thời gian phi lý (`cutDur` $< 1.6$ s trên đường ướt).
- **Điều gì xảy ra nếu đếm chúng là va chạm?**
  - Nếu không lọc bỏ bằng `is_feasible`, thuật toán sẽ ghi nhận những vụ tai nạn "ảo" do vi phạm định luật vật lý (lốp xe có độ bám vô hạn). Việc này dẫn đến ước lượng sai lệch nghiêm trọng xác suất rủi ro (overestimation), tạo ra báo động giả (false alarm) và khiến kỹ sư tối ưu hóa hệ thống vào những tình huống không bao giờ tồn tại trên đường thực tế.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

### Số liệu thực nghiệm Monte Carlo (`mc.json`)

- Số lần thử nghiệm ($n$): `mc.json: n = 3000`
- Số vụ va chạm ghi nhận ($k$): `mc.json: crashes = 5`
- Tần suất mẫu ($\hat{p}$): `mc.json: p = 0.00166667` ($1.67 \times 10^{-3}$)
- Khoảng tin cậy Wilson 95 %: `mc.json: [lo, hi] = [0.00071209, 0.00389587]` ($[7.12 \times 10^{-4}, 3.90 \times 10^{-3}]$)
- Số kịch bản bất khả thi bị bỏ qua: `mc.json: infeasible = 9`

### Số lần chạy để khoảng tin cậy hẹp còn $\pm 20\%$ của $p$

- Bán kính khoảng tin cậy 95 % ở mức lỗi tương đối $\epsilon = 0.20$ yêu cầu:
  $$1.96 \cdot \sqrt{\frac{p(1-p)}{n}} \le \epsilon \cdot p \implies n \ge \frac{1.96^2}{\epsilon^2 \cdot p} \approx \frac{3.8416}{0.04 \cdot p} \approx \frac{96.04}{p}$$
- Với $\hat{p} \approx 0.0016667$:
  $$n \ge \frac{96.04}{0.0016667} \approx 57\,624\text{ lần chạy}$$
  Cần khoảng **58 000 lần chạy mô phỏng** để độ chính xác đạt $\pm 20\%$.

### Kiểu thất bại chưa từng xuất hiện và con số cần báo

- Các kiểu va chạm ghi nhận trong `mc.json: modes`:
  - `bus_bypass`: 5 vụ (`mc.json: per_mode.bus_bypass.k = 5`)
  - `cut_in`: 0 vụ (`mc.json: per_mode.cut_in.k = 0`)
  - `crosswalk_block`: 0 vụ (`mc.json: per_mode.crosswalk_block.k = 0`)
  - `rear_ended`: 0 vụ (`mc.json: per_mode.rear_ended.k = 0`)
  - `other`: 0 vụ (`mc.json: per_mode.other.k = 0`)
- **Kiểu chưa xuất hiện:** Có 4 kiểu hoàn toàn chưa xuất hiện trong 3 000 lần chạy: `cut_in`, `crosswalk_block`, `rear_ended` và `other`.
- **Con số đúng cần báo:** Tuyệt đối **không báo xác suất bằng 0**. Theo chuẩn thống kê và quy tắc Wilson / Rule of Three:
  - Cận trên khoảng Wilson 95 % cho $k = 0, n = 3000$ là:
    $$\hat{p}_{\text{upper}} = \frac{1.96^2}{3000 + 1.96^2} = 0.0012789 \approx 1.28 \times 10^{-3}$$
  - Hoặc quy tắc số 3 (Rule of Three): $\text{upper} = \frac{3}{n} = \frac{3}{3000} = 1.00 \times 10^{-3}$.
  - Con số đúng cần báo là: **$P(\text{thất bại}) \le 1.28 \times 10^{-3}$** (95% CI $[0, 1.28 \times 10^{-3}]$).

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

### Bảng tổng hợp Importance Sampling qua 3 seed (`estimate.json`)

| Kiểu thất bại | Seed | Xác suất ước lượng ($p$) | Số va chạm (`hits`) | Cỡ mẫu hiệu dụng (`ess`) | Tỷ trọng mẫu lớn nhất (`top_share`) |
|---|---|---|---|---|---|
| **bus_bypass** | 0 | $4.69 \times 10^{-4}$ | 31 | 122.0 | 0.284 |
| | 1 | $2.75 \times 10^{-4}$ | 24 | 122.6 | 0.383 |
| | 2 | $3.15 \times 10^{-4}$ | 31 | 129.0 | 0.324 |
| **crosswalk_block** | 0 | $1.28 \times 10^{-6}$ | 110 | 111.1 | 0.747 |
| | 1 | $5.58 \times 10^{-7}$ | 94 | 132.0 | 0.461 |
| | 2 | $1.85 \times 10^{-5}$ | 113 | 126.1 | 0.819 |
| **cut_in** | 0 | $0.00$ ($< 2.50 \times 10^{-2}$) | 0 | 137.9 | 0.000 |
| | 1 | $0.00$ ($< 2.50 \times 10^{-2}$) | 0 | 122.0 | 0.000 |
| | 2 | $0.00$ ($< 2.50 \times 10^{-2}$) | 0 | 114.2 | 0.000 |
| **rear_ended** | 0 | $0.00$ ($< 2.50 \times 10^{-2}$) | 0 | 131.0 | 0.000 |
| | 1 | $0.00$ ($< 2.50 \times 10^{-2}$) | 0 | 127.0 | 0.000 |
| | 2 | $0.00$ ($< 2.50 \times 10^{-2}$) | 0 | 127.7 | 0.000 |
| **TỔNG (`total`)** | 0 | $4.70 \times 10^{-4}$ | — | — | 95% CI: $[1.17 \times 10^{-4}, 2.10 \times 10^{-3}]$ |
| | 1 | $2.76 \times 10^{-4}$ | — | — | 95% CI: $[3.26 \times 10^{-5}, 1.80 \times 10^{-3}]$ |
| | 2 | $3.34 \times 10^{-4}$ | — | — | 95% CI: $[5.66 \times 10^{-5}, 1.89 \times 10^{-3}]$ |

*(Ghi chú: với kiểu 0 hits, `upper_zero_hits` $= 3 / (0.2 \times 600) = 0.025$. Phần `other` lấy từ MC: $p=0$, cận trên $1.28 \times 10^{-3}$).*

### Đánh giá kỹ thuật

1. **Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không?**
   - Với `bus_bypass`, 3 seed đồng thuận chặt chẽ quanh mức $(2.75 \sim 4.69) \times 10^{-4}$ (trung vị $3.15 \times 10^{-4}$).
   - Với `crosswalk_block`, 3 seed dao động mạnh từ $5.58 \times 10^{-7}$ đến $1.85 \times 10^{-5}$ (chênh nhau hơn 30 lần).
   - So với Monte Carlo: Monte Carlo ước lượng tổng là $1.67 \times 10^{-3}$ (chỉ thấy `bus_bypass`), trong khi IS ước lượng `bus_bypass` trung vị là $3.15 \times 10^{-4}$ (thấp hơn MC khoảng 4-5 lần). Nguyên nhân do phân phối đề xuất $q_{\text{CEM}}$ bị co cụm vào một phần của miền va chạm (mode collapse/tightening), dù có hỗn hợp phòng thủ $\beta = 0.2$ thì ước lượng vẫn có xu hướng bị lệch dưới khi ngân sách hữu hạn.
2. **Ở đâu `top_share` lớn (> 0.5), và vì sao khi đó con số chỉ là trọng số của một mẫu?**
   - `top_share` vượt quá 0.5 ở kiểu `crosswalk_block` tại seed 0 ($0.747$) và seed 2 ($0.819$).
   - Khi `top_share` $> 0.5$, một mẫu duy nhất trong số $n=600$ mẫu chiếm tới $75\% - 82\%$ giá trị tổng trọng số $\sum w_i y_i$. Đây là hiện tượng suy biến trọng số (weight degeneracy): đề xuất $q$ rất hẹp nên khi tình cờ bốc trúng một điểm rìa có xác suất tự nhiên $p$ tương đối lớn, tỷ số $w = p/q$ phát nổ, biến ước lượng kỳ vọng thành giá trị của riêng mẫu đơn lẻ đó.
3. **So ngân sách (`budget`) với số lần chạy Monte Carlo:**
   - Ngân sách IS: `estimate.json: budget = 16800` lần chạy (4 kiểu $\times$ (8 vòng $\times$ 300 + 3 seed $\times$ 600)).
   - Để đo được xác suất cỡ $10^{-5}$ như `crosswalk_block` bằng Monte Carlo với cùng độ chính xác $\pm 20\%$, cần $n \approx 96 / 10^{-5} \approx 9.6 \times 10^6$ lần chạy (gần 10 triệu lần). IS tiết kiệm hơn **570 lần** chi phí tính toán.
4. **Phần `other` lấy từ đâu và vì sao không săn?**
   - Lấy từ `mc.json`: `estimate.json: other.source = "mc.json"`, $k=0, n=3000, p=0.0$, cận trên Wilson là $1.28 \times 10^{-3}$.
   - Không săn `other` vì đây là phần bù không thuần nhất của các kiểu va chạm chưa được định nghĩa; không thể thiết kế một hàm điểm nguy hiểm (danger score) cụ thể cho một tập hợp hỗn tạp không có cấu trúc.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

### Mô tả chuỗi sự kiện và minh chứng 3D

1. **Kiểu `bus_bypass` (lách qua xe buýt đỗ):**
   - *Minh chứng 3D:* `evidence/shots/ch9_bus_bypass.png`
   - *Chuỗi sự kiện:* Xe buýt đỗ chắn tầm nhìn ở làn ngoài ($E2$, $x=326$ m). Xe hàng xóm C2 tìm cách lách qua sườn xe buýt ở tốc độ thấp ($v < 6$ m/s, $x < 112$ m) để nhập vào làn $E1$. C2 bị khuất sau đuôi xe buýt nên cảm biến nóc của xe ta không nhìn thấy sớm. Khi C2 thò đầu ra làn $E1$, xe ta đang di chuyển ở vận tốc hành trình và không kịp dừng, đâm vào sườn trước bên phải của C2 lúc $t = 12.8$ s.
2. **Kiểu `crosswalk_block` (chặn lối người đi bộ):**
   - *Minh chứng 3D:* `evidence/shots/ch9_crosswalk_block.png`
   - *Chuỗi sự kiện:* Người đi bộ P1 bước xuống từ vạch qua đường ($WALK\_X = 122$ m) bị xe buýt che khuất hoàn toàn. Khi P1 vượt qua mép xe buýt và lọt vào tầm quét cảm biến, xe ta đang chạy $v > cautionV$ ($7.5$ m/s). Xe ta nhận diện P1 và phanh gấp, nhưng do đà quán tính, xe dừng lại khi thân xe đã chồm lên và nằm đè trọn lên vạch kẻ đường, khiến người đi bộ phải đứng khựng lại né tránh (`kind == "block"`, sự kiện xảy ra lúc $t = 19.7$ s).

### Kiểu cuộc săn không tìm thấy và nguy cơ khi chỉ săn chung một cuộc

- **Kiểu không tìm thấy:** Cuộc săn không tìm thấy `cut_in` và `rear_ended` (trong `hunt.json`, cuộc săn `cut_in` chỉ kích hoạt 19 vụ bypass và 1 vụ block; cuộc săn `rear_ended` chỉ ra 2 vụ bypass và 3 vụ block; không có vụ va chạm nào thuộc đúng nhãn của chúng).
- **Mức độ tin cậy:** Rất tin cậy. Trong giao thông phản ứng chuẩn, xe phía sau TG luôn giữ khoảng cách an toàn bằng IDM và chỉ tông đuôi nếu xe ta phanh tức thời vô lý. Tương tự, xe C2 chỉ cắt làn khi có khe hở chấp nhận được (`cutGap`). Các kiểu này chỉ xuất hiện khi vi phạm giới hạn vật lý (đã bị `is_feasible` chặn).
- **Nếu chỉ chạy một cuộc săn chung:** Sẽ xảy ra hiện tượng **suy biến mode (mode collapse)**. Thuật toán CEM sẽ dồn toàn bộ mật độ xác suất vào hố hấp dẫn của kiểu va chạm dễ xảy ra nhất là `bus_bypass` (chiếm 85/85 mẫu trong cuộc săn bypass), khiến các kiểu hiếm hơn nhưng cực kỳ nguy hiểm như `crosswalk_block` bị bỏ sót hoàn toàn.

---

## 6. Sửa planner (bước E + 3D)

### Bảng số liệu tối ưu hóa Safety Policy (`fix.json`)

| Chỉ số | Policy gốc (Baseline) | Policy được chọn (`cutinLat: 2.6`) | Trích xuất từ `evidence/fix.json` |
|---|---|---|---|
| **Núm điều chỉnh** | Mặc định (`cutinLat: 2.15`) | `{"cutinLat": 2.6}` | `fix.json: chosen.policy` |
| **TRAIN:** Xác suất thất bại (`fail`) | $6.15 \times 10^{-4}$ | $3.00 \times 10^{-7}$ | `fix.json: table[5].fail` |
| **TRAIN:** Cận trên 95% (`fail_hi`) | $1.29 \times 10^{-3}$ | $5.32 \times 10^{-7}$ | `fix.json: table[5].fail_hi` |
| **TRAIN:** Chế độ chưa thấy (`unseen`) | `[]` | `["bus_bypass"]` | `fix.json: table[5].unseen` |
| **TEST:** Xác suất thất bại (`fail`) | $2.34 \times 10^{-4}$ | $5.01 \times 10^{-6}$ | `fix.json: chosen_test.fail` |
| **TEST:** Cận trên các mode thấy (`fail_hi_seen`) | $4.28 \times 10^{-4}$ | $1.24 \times 10^{-5}$ | `fix.json: chosen_test.fail_hi_seen` |
| **TEST:** Cận trên hợp lệ 95% (`fail_hi`) | $1.71 \times 10^{-3}$ | $1.29 \times 10^{-3}$ | `fix.json: chosen_test.fail_hi` |
| **TEST:** Chế độ chưa thấy (`unseen`) | `[]` | `[]` | `fix.json: chosen_test.unseen` |
| **Quãng đường di chuyển trung bình** | $217.44$ m | $217.41$ m (giảm $0.01\%$) | `fix.json: chosen_test.progress` |
| **Số khung hình phanh gấp ($a < -3\text{ m/s}^2$)** | $7.88$ frames | $7.65$ frames (giảm $2.9\%$) | `fix.json: chosen_test.harsh` |

### Phân tích kỹ thuật

1. **Vì sao `fail_hi` của bản sửa ($1.29 \times 10^{-3}$) có thể lớn hơn `fail` của policy gốc ($2.34 \times 10^{-4}$)?**
   - Đây là sự khác biệt căn bản giữa **cận trên thống kê (upper confidence bound)** và **ước lượng điểm (point estimate)**:
     - `fail` của policy gốc ($2.34 \times 10^{-4}$) là tần suất trung bình quan sát được trên tập mẫu.
     - `fail_hi` của bản sửa ($1.29 \times 10^{-3}$) là cận trên 95% nghiêm ngặt, bắt buộc phải cộng thêm biên độ không chắc chắn của các va chạm ngoài tầm săn (`other_crashes.hi` $= 1.28 \times 10^{-3}$ trên $n=3000$ mẫu holdout theo Wilson/Rule of Three).
     - Khi so sánh cùng hệ quy chiếu ước lượng điểm, bản sửa đạt $5.01 \times 10^{-6}$, giảm rủi ro tới **46 lần** so với gốc ($2.34 \times 10^{-4}$). So sánh `fail_hi_seen`, bản sửa ($1.24 \times 10^{-5}$) an toàn hơn **34 lần** so với gốc ($4.28 \times 10^{-4}$).
2. **Minh chứng 3D có bóng mờ (ghost vehicle):**
   - *Minh chứng 3D:* `evidence/shots/ch9_ghost_comparison.png`
   - *Phân tích hình ảnh:* Chiếc xe bóng mờ (policy gốc với `cutinLat = 2.15`) nhận diện C2 muộn nên phanh gấp không kịp và đâm vào C2 ở $t=12.8$ s. Trong khi đó, xe thật (bản sửa với `cutinLat = 2.6`) mở rộng hành lang nhận diện sang hai bên, phát hiện C2 lấn làn sớm hơn $0.45$ m, chủ động giảm ga êm ái và tránh va chạm hoàn toàn.
3. **Núm chỉnh "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại:**
   - Ứng viên số 9: tăng khoảng cách thời gian theo sau IDM `{"T": 2.0}` (nghe có vẻ giữ khoảng cách xa hơn thì an toàn hơn).
   - Tuy nhiên, trong `fix.json: table[8]`, `fail` vọt lên $9.07 \times 10^{-4}$ (cao gấp rưỡi baseline!).
   - *Cơ chế thế giới phản ứng lại:* Khi xe ta tăng khoảng cách theo sau $T$ lên $2.0$ s, xe ta tạo ra một khoảng trống mênh mông phía trước đầu xe. Khoảng trống này kích thích xe hàng xóm C2 thực hiện hành vi cắt làn gắt và đột ngột hơn, đồng thời khiến xe ta phải liên tục phanh nhường đường, gia tăng nguy cơ bị xe TG đâm đuôi và va chạm góc với C2. Tương tự, nếu hạ `cautionV` xuống $5.0$ m/s, quãng đường bị tụt xuống $170.4$ m (mất $>20\%$ tiến độ), vi phạm điều kiện ràng buộc.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định then chốt trong `space.py` và phương án thay thế bằng dữ liệu thực tế:

| Giả định mô phỏng | Nguồn dữ liệu thực tế thay thế | Hướng sai lệch nếu giả định sai |
|---|---|---|
| **1. Khoảng cách nhập làn (`cutGap`, median = 14 m ở Town D)** | Trích xuất từ tập dữ liệu quỹ đạo tự nhiên **highD / NGSIM / INTERACTION dataset** bằng camera flycam trên cao tại các tuyến đường đô thị. | Nếu tài xế thực tế liều lĩnh hơn (chấp nhận khe hở $< 10$ m), mô phỏng của ta đã đánh giá quá lạc quan (underestimate nguy cơ va chạm cắt làn). |
| **2. Tỷ lệ người đi bộ ngó qua xe buýt (`pedLook`, xác suất 90% ở Town D)** | Khảo sát bằng camera thị giác máy tính tại các trạm dừng xe buýt thực tế đo tỷ lệ quay đầu/hướng nhìn của người đi bộ trước khi qua đường. | 90% người đi bộ ngó nhìn là giả định quá cao (thực tế nhiều người cắm mặt vào điện thoại hoặc vội vã chỉ đạt ~30-50%). Nếu giả định sai, rủi ro va chạm người đi bộ trên thực tế sẽ cao hơn gấp nhiều lần mô phỏng. |
| **3. Hệ số ma sát đường ướt ($\mu = 0.5$ cố định khi mưa)** | Đo đạc trắc diện mặt đường bằng xe đo ma sát chuyên dụng (GripTester / Mu-Meter) kết hợp dữ liệu khí tượng thủy văn trạm đo mưa. | Ma sát khi mưa biến thiên từ $0.25$ đến $0.6$ tùy độ mòn lốp và độ thoát nước (nguy cơ aquaplaning). Giả định $\mu = 0.5$ cố định sẽ đánh giá thấp quãng đường phanh thực tế trong các trận mưa rào nhiệt đới. |

---

## 8. Giới hạn

### Ba điều mà xác suất báo cáo KHÔNG nói lên

1. **Thế giới mô phỏng là môi trường hình học 1 chiều đóng kín:** Chỉ có một con phố thẳng dài 550 m, không có giao lộ đa hướng, không có bùng binh, không có phương tiện quay đầu (U-turn), không có xe máy luồn lách (loại hình phổ biến nhất tại giao thông Việt Nam).
2. **Hành vi tài xế xung quanh bị giới hạn bởi mô hình IDM đơn giản:** IDM giả định mọi phương tiện tuân thủ quy tắc giữ làn và phanh hoàn hảo. Nó không mô phỏng được hành vi người lái mất tập trung, say xỉn, giật mình đánh lái bất thình lình, hoặc không gian nhường đường xã hội phức tạp.
3. **Mô hình cảm biến và nhận thức lý tưởng hóa (Ground-Truth Bounding Box):** Cảm biến chỉ bị che khuất hình học bởi hộp xe buýt (`seg_blocked`), không có trễ đường truyền (processing latency 100-200 ms), không có hiện tượng mất dấu vết (tracking flickers), và không bị ảnh hưởng bởi sương mù, hạt mưa hay chói lóa ánh nắng mặt trời.

### Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

- Xây dựng mô hình kiểm thử vòng lặp kín (closed-loop) tích hợp **độ trễ nhận thức thực tế** ($\tau \in [100, 250]\text{ ms}$) và độ nhiễu đo đạc cảm biến; bổ sung luồng tác nhân xe hai bánh (xe máy, xe đạp điện) di chuyển hỗn hợp giữa các làn đường để đo lường xác suất va chạm biên sườn xe.

---

## 9. Rắc rối của bạn (bước G + chương 10)

### Bản thiết kế rắc rối (`hazard.json`)

- **Rắc rối:** Người đi bộ bất ngờ băng ngang đường tự phát ngoài vạch kẻ (mid-block jaywalking) từ lề đường phía nam ($z = 9.2$ m) sang phía bắc tại tọa độ $x \in [185, 215]$ m, nơi xe ta đã đi qua vạch đi bộ hợp lệ ($x = 122$ m) và đang tăng tốc trở lại vận tốc hành trình $v_0 = 12.5$ m/s ($45$ km/h). Ngoài đời, tình huống này giống người đi bộ vội vã băng qua đường từ giữa phố buôn bán, hàng quán hoặc sau chướng ngại vật khuất tầm nhìn.
- **Căn cứ kỹ thuật:** Bộ điều khiển planner của xe tự lái chỉ thiết lập vùng thận trọng giảm tốc (`cautionV = 7.5`, `cautionRange = 55`) quanh khu vực vạch đi bộ số hóa tĩnh trên bản đồ ($WALK\_X = 122$ m). Khi xe vượt qua $x > 140$ m, hệ thống mặc định đường thông thoáng và hủy bỏ chế độ thận trọng. Khi người đi bộ băng ra bất ngờ ở $x = 200$ m, planner chỉ phản ứng thuần túy bằng IDM khi người đã đi vào hành lang xe chạy; gia tốc phanh êm ái $b = 2.0$ m/s² không đủ quãng đường để hãm xe dừng kịp thời.
- **Giả định (`hazard.json: why`):**
  - Miền tham số: `space = {"h0_x": [185.0, 215.0], "h0_v": [1.8, 2.4], "h0_trig": [36.0, 48.0]}`.
  - Vận tốc đi bộ $1.8 \sim 2.4$ m/s phản ánh tốc độ đi bộ nhanh/vội vàng theo nghiên cứu FHWA/HCM. Khoảng cách kích hoạt $36 \sim 48$ m khi xe đang chạy $12.5$ m/s tạo ra cửa sổ thời gian va chạm khoảng $3 \sim 4$ s.
  - Tần suất phơi nhiễm (`exposure`): $0.015$ (tức $1.5\%$ số chuyến đi đô thị, trích từ số liệu nghiên cứu lái xe tự nhiên SHRP2 NDS và cơ sở dữ liệu tai nạn người đi bộ NHTSA).

### Kết quả quét thực nghiệm (`mine.json`)

| Chỉ số thực nghiệm | Policy gốc (Baseline) | Policy được chọn (`cutinLat: 2.6`) | Trích xuất từ `evidence/mine.json` |
|---|---|---|---|
| Số kịch bản quét hợp lệ ($n$) | 1 490 / 1 500 ($99.3\%$ accepted) | 1 490 / 1 500 ($99.3\%$ accepted) | `mine.json: baseline.n`, `mine.json: accepted` |
| Tổng số vụ va chạm với xe ta | 247 | 244 | `mine.json: baseline.crashes`, `mine.json: chosen.crashes` |
| – Va chạm tránh được (`avoidable`) | **246** | **244** | `mine.json: baseline.avoidable`, `mine.json: chosen.avoidable` |
| – Không thể tránh (`unavoidable`) | 0 | 0 | `mine.json: baseline.unavoidable`, `mine.json: chosen.unavoidable` |
| – Không quy được (`unattributed`) | 1 | 0 | `mine.json: baseline.unattributed`, `mine.json: chosen.unattributed` |
| – Va chạm ở tốc độ thấp (`low_speed` $< 1$ m/s) | 2 | 2 | `mine.json: baseline.low_speed`, `mine.json: chosen.low_speed` |
| **$P(\text{lỗi planner} \mid \text{hazard})$** | **$16.51\%$** ($0.1651$) | **$16.38\%$** ($0.1638$) | `mine.json: baseline.p`, `mine.json: chosen.p` |
| Khoảng tin cậy Wilson 95 % | $[14.71\%, 18.48\%]$ | $[14.58\%, 18.34\%]$ | `mine.json: baseline.[lo, hi]`, `mine.json: chosen.[lo, hi]` |
| **Tần suất lỗi mỗi chuyến đi** | **$2.48 \times 10^{-3}$** (cận trên $2.77 \times 10^{-3}$) | **$2.46 \times 10^{-3}$** (cận trên $2.75 \times 10^{-3}$) | `mine.json: baseline.per_trip`, `mine.json: chosen.per_trip` |

*(So sánh: tần suất lỗi của rắc rối này là $2.46 \times 10^{-3}$ mỗi chuyến, **cao gấp gần 8 lần** so với tần suất va chạm lách xe buýt $3.15 \times 10^{-4}$ đo ở bước C3).*

### Phân tích chuyên sâu

1. **Vì sao "tránh được" (`avoidable`) là một chuẩn lạc quan?**
   - Theo định nghĩa của thuật toán oracle:
     - **Ở hướng thứ nhất (chuẩn quá khắt khe với planner / đếm nhiều hơn lỗi thực tế):** Oracle được kích hoạt phanh hết cỡ với gia tốc kịch khung $\mu g \approx 7.85$ m/s² ngay tại khung hình đầu tiên vật cản vừa bắt đầu di chuyển và lọt vào tầm nhìn cảm biến. Oracle không cần thời gian trễ nhận thức (human/AI perception latency), không cần tích lũy bộ lọc vị trí (tracking convergence), và hoàn toàn không màng tới sự an toàn/êm ái của hành khách trên xe. Một planner thật trong thực tế không thể phanh gấp tối đa liên tục mỗi khi thấy bóng người ven đường, nên chuẩn `avoidable` đếm nhiều hơn số lỗi một planner thực tế có thể tránh.
     - **Ở hướng thứ hai (chuẩn quá hạn chế / đếm ít đi):** Bộ oracle trong mã nguồn chỉ thử nghiệm duy nhất một hành động là **đạp hết phanh trên đường thẳng**, hoàn toàn không thử nghiệm đánh lái né tránh (evasive swerving). Trên thực tế, khi khoảng cách phanh không đủ, tài xế hoặc planner thông minh có thể đánh lái nhẹ sang làn bên cạnh nếu làn đó đang trống để tránh va chạm.
2. **Hình ảnh minh chứng từ chương 9 & 10:**
   - *Minh chứng Chapter 9:* `evidence/shots/ch9_hazard_mine.png` (khoảnh khắc va chạm giữa xe ta và người đi bộ H0 tại $t = 22.55$ s, $x = 198.8$ m).
   - *Minh chứng Chapter 10:* `evidence/shots/ch10_studio_hazard.png` (giao diện thiết kế rắc rối trong xưởng kịch bản với thẻ rắc rối H0).
3. **Thử nghiệm tự lái:**
   - Khi chuyển sang chế độ tự cầm lái bằng bàn phím trong Chapter 10, người lái phát hiện người đi bộ từ khoảng cách $> 45$ m ở rìa vỉa hè. Thay vì tiếp tục đạp ga duy trì $12.5$ m/s như planner, người lái chủ động nhả ga phòng ngừa từ sớm, rà phanh êm ái xuống $6$ m/s và dừng hẳn cách người đi bộ hơn $5$ m mà không cần phải đạp phanh khẩn cấp giật cục.
4. **Cách khắc phục:**
   - *Núm chỉnh hiện tại:* Không có núm chỉnh nào trong 9 núm hiện tại của planner sửa được rắc rối này một cách trọn vẹn mà không gây tổn hại lớn: nếu giảm vận tốc mong muốn $v_0$ xuống $8$ m/s thì xe chạy quá chậm (mất $> 30\%$ quãng đường); nếu kéo dài `cautionRange` lên $90$ m thì xe chỉ thận trọng quanh $x = 122$ m chứ không bao phủ được các vị trí bất kỳ ngoài phố.
   - *Logic mới cần bổ sung cho planner:* Planner bắt buộc phải có **Mô-đun dự đoán quỹ đạo và ý định người đi bộ động (Dynamic Pedestrian Intent & Trajectory Prediction)** độc lập với bản đồ số:
     - Khi cảm biến phát hiện bất kỳ người đi bộ nào ở ven đường có véc-tơ vận tốc ngang hướng vào lòng đường ($v_z < 0$), hệ thống tính toán thời gian cắt ngang (Time-to-Lane-Crossing).
     - Nếu dự báo người đi bộ sẽ cắt vào hành lang xe chạy trong vòng $3.0$ s tới, planner tự động kích hoạt trạng thái "thận trọng chủ động" (proactive yielding): nhả ga sớm, chuẩn bị áp lực phanh (brake pre-fill) và giảm tốc độ về ngưỡng an toàn ($6 \sim 7$ m/s) từ trước, thay vì chỉ phản ứng thụ động bằng IDM khi người đã bước hẳn vào trước mũi xe.
