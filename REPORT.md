MSSV: 2A202602755
Họ và tên: Đặng Quốc Hiệp (Mã lớp/HV: 2A202602755)

# Lab 11 · Scenario Forge — báo cáo

Thị trấn của bạn (`meta.json: _run.town`): **D** (p_bin: 0.15, 0.90, 0.20; cut_gap_median: 14.0m)

---

## 1. Phát lại log có trung thực không? (bước A)

Trích số từ `evidence/replay.json`:
- Số kịch bản bị loại do người lái gây va chạm: `dropped_human_crash_logs = 0` (trên tổng số `n = 400`, toàn bộ `n_used = 400` kịch bản đều hợp lệ).
- Va chạm khi planner chạy trên **log phát lại (replay)**: `crashes = 110`, phân bố loại tiếp xúc: `rear = 100`, `side = 10`, `front = 0`.
- Va chạm khi planner chạy trong **thế giới phản ứng (reactive)**: `crashes = 0`, phân bố tiếp xúc: `{}` (không có va chạm nào).
- Số kịch bản va chạm ở cả hai: `crash_in_both = 0`.

**Giải thích:**
Phần lớn va chạm của phương pháp phát lại là va chạm từ phía sau (`rear = 100/110` vụ, chiếm 90.9%). Đây là **va chạm giả (sim artifact)**. 
Nguyên nhân là log phát lại là hệ thống hở (open-loop replay): các xe khác di chuyển y hệt theo quỹ đạo cố định đã ghi lại khi người lái thật cầm lái. Khi xe tự lái (planner) phanh sớm hơn hoặc phanh thận trọng hơn người lái thật, xe bám đuôi phía sau (TG) trong log phát lại không có cơ chế phản xạ để giảm tốc mà tiếp tục lao tới theo dữ liệu cũ, dẫn đến việc húc vào đuôi xe ta. Trong thế giới phản ứng (`reactive`), các xe phía sau tuân theo mô hình IDM nên khi xe ta phanh, xe sau cũng chủ động đạp phanh giữ khoảng cách an toàn, triệt tiêu hoàn toàn 100 vụ va chạm phía sau này.

**Kịch bản ví dụ (`replay_only`):**
Trong kịch bản số 5 (`scenario: 5` trong `replay_only`): trên log phát lại, xe ta bị tông đuôi (`contact: rear`). Nhưng cũng kịch bản đó trong thế giới phản ứng, thời gian tới va chạm nhỏ nhất là `min_ttc_reactive = 7.46 s` — xe sau phản ứng kịp thời và giữ cự ly hoàn toàn an toàn, không có tai nạn.

---

## 2. Kịch bản bất khả thi (bước B)

**Các luật trong `is_feasible` và căn cứ vật lý:**
1. **Gia tốc ngang khi chuyển làn:** $a_{\text{lat}} = \left(\frac{\pi}{\text{cutDur}}\right)^2 \frac{3.5}{2} \le \mu g$.
   *Căn cứ vật lý:* Định luật ma sát Coulomb. Lực ma sát ngang tối đa giữa mặt đường và lốp xe là $F_{\text{max}} = \mu m g$. Một cú chuyển làn hình cosine cự ly 3.5 m trong thời gian `cutDur` giây có gia tốc ngang đỉnh là $a_{\text{lat}}$. Nếu $a_{\text{lat}} > \mu g$, xe sẽ bị mất độ bám ngang, trượt văng khỏi mặt đường.
2. **Gia tốc phanh sau nhập làn:** $\text{cutBrake} \le \mu g$.
   *Căn cứ vật lý:* Lực bám dọc cực đại của lốp xe. Xe không thể giảm tốc vượt quá giới hạn ma sát của lốp với mặt đường mà không bị bó cứng bánh xe/trượt lết vô định hướng.
3. **Tốc độ người đi bộ:** $\text{pedV} \le 2.5\text{ m/s}$.
   *Căn cứ vật lý:* Giới hạn động học sinh học người đi bộ đô thị. Tốc độ đi bộ thông thường là 1.2–1.5 m/s; 2.5 m/s (~9 km/h) là ngưỡng chạy nhanh. Vượt quá ngưỡng này là phi thực tế với người đi bộ băng qua đường bình thường.
4. **Vận tốc xe nhập làn:** $0 < \text{cutV} \le 22.0\text{ m/s}$ (~80 km/h).
   *Căn cứ vật lý:* Xe chuyển làn trên đường phố nội đô không thể đi lùi/đứng yên ($\le 0$) và không thể vượt quá 80 km/h trong khu dân cư có giới hạn tốc độ.

**Hiện tượng trong `evidence/hunt.json`:**
Cuộc săn sinh ra nhiều kịch bản bất khả thi nhất là **`crosswalk_block`** với **11 kịch bản bất khả thi** bị loại (`infeasible = 11`), tiếp theo là `rear_ended` (7), `cut_in` (6) và `bus_bypass` (2).
*Lý do CEM bị kéo về phía đó:* Hàm tính độ nguy hiểm của `crosswalk_block` là $-\text{ped\_room}(R)$, với $\text{ped\_room} = \text{ahead} - \frac{v^2}{2\mu g}$. Để tối đa hóa độ nguy hiểm (khoảng cách dừng âm nhất có thể), CEM bị cám dỗ đẩy các thông số về cực trị: chọn đường ướt ma sát thấp ($\mu = 0.5$), người đi bộ chạy nhanh vượt ngưỡng ($\text{pedV} > 2.5$) và chuyển làn quá gắt. Các kịch bản này đạt điểm nguy hiểm cao nhất trong mắt thuật toán nên nó tập trung tìm kiếm vào đó.
*Nếu đếm chúng là va chạm:* Ta sẽ thổi phồng xác suất thất bại của xe tự lái một cách sai lệch, quy kết lỗi cho planner trên những tình huống vi phạm định luật vật lý mà ngoài đời thực không bao giờ xảy ra.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

Trích số từ `evidence/mc.json`:
- Số lần thử: $n = 3000$
- Số vụ va chạm: $k = 1$
- Tỉ lệ va chạm điểm: $\hat{p} = \frac{1}{3000} \approx 0.000333$ ($3.33 \times 10^{-4}$)
- Khoảng tin cậy Wilson 95%: $[5.88 \times 10^{-5}, 1.89 \times 10^{-3}]$
- Va chạm duy nhất ghi nhận thuộc kiểu `bus_bypass` (`modes: {'bus_bypass': 1}`).

**Số lần chạy cần thiết để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:**
Nửa độ rộng khoảng tin cậy 95% của phân phối nhị thức xấp xỉ $1.96 \sqrt{\frac{p(1-p)}{n}} \approx 1.96 \sqrt{\frac{p}{n}}$ (khi $p \ll 1$).
Yêu cầu: $1.96 \sqrt{\frac{p}{n}} \le 0.20 p \implies \sqrt{n} \ge \frac{1.96}{0.20 \sqrt{p}} \implies n \ge \frac{1.96^2}{0.04 p} \approx \frac{96.04}{p}$.
Với $\hat{p} \approx 3.33 \times 10^{-4}$:
$$n \ge \frac{96.04}{0.0003333} \approx 288,120\text{ lần chạy.}$$

**Các kiểu thất bại chưa xuất hiện:**
Các kiểu `cut_in`, `crosswalk_block`, `rear_ended` và `other` đều có $k = 0$ va chạm trong 3 000 lần chạy (`mc.json: per_mode.<mode>.k = 0`).
**Con số đúng cần báo:** Không bao giờ báo bằng 0 (vì 0 vụ trong 3 000 lần không có nghĩa là xác suất bằng 0). Cần báo **cận trên 95% khoảng Wilson** (hoặc Rule of Three: $3/n = 3/3000 = 1.0 \times 10^{-3}$):
Theo `mc.json`, cận trên đúng là **$1.28 \times 10^{-3}$** (`hi = 0.0012789`).

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

**Bảng kết quả từ `evidence/estimate.json`:**

| Kiểu thất bại | Seed | Ước lượng $p$ | Số va chạm (hits) | ESS | Top share |
|---|---|---|---|---|---|
| `bus_bypass` | 0 | $4.95 \times 10^{-4}$ | 51 | 114.2 | 0.304 |
| `bus_bypass` | 1 | $3.27 \times 10^{-4}$ | 44 | 135.5 | 0.297 |
| `bus_bypass` | 2 | $2.56 \times 10^{-4}$ | 58 | 123.3 | 0.133 |
| `cut_in` | 0 | 0.0 (hi: 0.025) | 0 | 125.4 | 0.0 |
| `cut_in` | 1 | 0.0 (hi: 0.025) | 0 | 121.2 | 0.0 |
| `cut_in` | 2 | 0.0 (hi: 0.025) | 0 | 110.4 | 0.0 |
| `crosswalk_block` | 0 | $1.44 \times 10^{-6}$ | 100 | 126.3 | 0.401 |
| `crosswalk_block` | 1 | $1.87 \times 10^{-6}$ | 88 | 115.0 | 0.877 |
| `crosswalk_block` | 2 | $1.31 \times 10^{-6}$ | 95 | 114.1 | 0.654 |
| `rear_ended` | 0 | 0.0 (hi: 0.025) | 0 | 117.2 | 0.0 |
| `rear_ended` | 1 | 0.0 (hi: 0.025) | 0 | 119.1 | 0.0 |
| `rear_ended` | 2 | 0.0 (hi: 0.025) | 0 | 132.3 | 0.0 |

**Nhận xét:**
1. **Sự đồng thuận:**
   - Với `bus_bypass`, cả 3 seed cho $p$ trong khoảng $[2.56 \times 10^{-4}, 4.95 \times 10^{-4}]$, trung vị $\approx 3.27 \times 10^{-4}$, hoàn toàn nằm trong khoảng tin cậy của Monte Carlo ($[5.88 \times 10^{-5}, 1.89 \times 10^{-3}]$).
   - Với `crosswalk_block`, cả 3 seed đều chứng minh đây là biến cố siêu hiếm ($p \approx 1.31 \times 10^{-6}$ đến $1.87 \times 10^{-6}$). Điều này giải thích tại sao 3 000 lần chạy Monte Carlo không thấy vụ nào ($k=0$).
2. **Độ suy biến (`top_share`):**
   `top_share` lớn ($> 0.5$) xuất hiện ở kiểu `crosswalk_block` tại Seed 1 (đạt **0.877**) và Seed 2 (đạt **0.654**). Khi `top_share` lớn, điều đó có nghĩa là một mẫu đơn lẻ có trọng số $w$ vượt trội áp đảo toàn bộ tổng $\sum w \cdot y$. Khi đó ước lượng bị chi phối bởi độ may rủi của đúng mẫu nặng ký đó thay vì phản ánh phân phối trung bình.
3. **So sánh ngân sách:**
   Tổng ngân sách IS là `estimate.json: budget = 16800` lần chạy. Để đo được `crosswalk_block` ($p \approx 1.5 \times 10^{-6}$) bằng Monte Carlo với độ chính xác $\pm 20\%$, cần tới $n \approx 96 / (1.5 \times 10^{-6}) \approx 64,000,000$ (64 triệu) lần chạy. IS tiết kiệm chi phí tính toán gần 4 000 lần.
4. **Phần `other`:** Lấy từ `mc.json` (`estimate.json: other.source = "mc.json"`, $k=0, n=3000$, cận trên $1.28 \times 10^{-3}$). Ta không săn `other` vì đây là tập hợp tất cả các lỗi ngoại lai không theo một hình thái cụ thể, không thể định nghĩa một hàm điểm nguy hiểm định hướng để CEM tối ưu.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

### Các kiểu tìm thấy:

1. **`bus_bypass`** (Lách qua xe buýt):
   ![Va chạm lách xe buýt](evidence/shots/ch9-lab-bus-bypass.jpg)
   *Chuỗi sự kiện:* Xe buýt dừng đỗ đón/trả khách ở làn E2 che khuất tầm nhìn. Xe C2 đi sau xe buýt quyết định lách sang làn E1 với tốc độ chậm ($< 6$ m/s). Xe ta đang tiến đến với tốc độ cao, bị thân xe buýt chắn tầm mắt nên nhận diện C2 muộn; khi nhận ra thì khoảng cách quá gần không kịp phanh, đâm vào C2.

2. **`crosswalk_block`** (Chặn lối người đi bộ):
   ![Chặn lối người đi bộ](evidence/shots/ch9-lab-crosswalk-block.jpg)
   *Chuỗi sự kiện:* Người đi bộ P1 bước ra từ phía sau đuôi xe buýt đỗ để qua đường. Xe ta nhận diện P1 muộn, phanh gấp khẩn cấp. Xe không đâm vào người (người đi bộ dừng lại trước đầu xe), nhưng mũi xe ta đã dừng đè lên vạch qua đường, chặn đứng lối đi của người đi bộ.

### Kiểu không tìm thấy:
Cuộc săn không tìm thấy va chạm thực sự nào cho `cut_in` và `rear_ended` (trong `hunt.json`, cuộc săn `cut_in` tìm được 16 vụ nhưng đều xảy ra gần xe buýt và bị gán nhãn là `bus_bypass`; cuộc săn `rear_ended` chỉ ra 7 vụ `crosswalk_block`). Kết quả này đáng tin cậy cao vì xe láng giềng nhập làn từ xa trong thị trấn D có khe chấp nhận lớn (`cut_gap_median = 14.0m`) và xe bám đuôi TG có IDM giữ khoảng cách phản ứng tốt, trừ khi bị ép vào thế tắc nghẽn quanh xe buýt.

### Nếu chỉ chạy MỘT cuộc săn chung:
Thuật toán CEM sẽ bị suy biến và co cụm (collapse) toàn bộ về kiểu va chạm dễ đạt điểm nguy hiểm cao nhất là `bus_bypass` (như thấy ở `hunt.json`), và sẽ bỏ sót hoàn toàn kiểu va chạm hiếm gặp nhưng nghiêm trọng là `crosswalk_block`.

---

## 6. Sửa planner (bước E + 3D)

Trích số từ `evidence/fix.json`:
- **Policy được chọn:** `{'cutinLat': 3.3}` (tăng ngưỡng ngang nhận diện xe nhập làn từ 2.15 lên 3.3 m).
- **Trên tập TRAIN:**
  - `fail = 1.08e-06`, `fail_hi = 2.82e-06`
  - Quãng đường di chuyển trung bình: `progress = 219.1 m` (so với baseline `219.0 m`, không bị suy giảm).
  - Đánh dấu `unseen: ['bus_bypass']`.
- **Trên tập TEST:**
  - **Policy được chọn:**
    - `fail = 9.74e-08`
    - `fail_hi_seen = 2.11e-07`
    - `fail_hi (valid) = 3.88e-02` (do cộng thêm cận `upper_zero_hits = 0.0375` của kiểu `unseen`)
    - `unseen = ['bus_bypass']`
    - Quãng đường: `progress = 217.0 m`, số khung phanh gấp: `harsh = 7.69`.
  - **Policy gốc (baseline):**
    - `fail = 3.10e-04`
    - `fail_hi_seen = 5.20e-04`
    - `fail_hi (valid) = 1.80e-03`
    - `unseen = []`
    - Quãng đường: `progress = 217.1 m`, số khung phanh gấp: `harsh = 8.15`.

**Vì sao `fail_hi` của bản sửa lại lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật?**
Vì trên tập TEST, bản sửa đã triệt tiêu hoàn toàn các va chạm `bus_bypass` (0 va chạm $\implies$ unseen). Theo chuẩn thống kê chặt chẽ 95%, khi 0 sự kiện xuất hiện trên mẫu $n=400$ với đề xuất phòng thủ $\beta=0.2$, ta không được quyền báo xác suất bằng 0 mà phải cộng cận an toàn $3 / (0.2 \times 400) = 0.0375$ vào `fail_hi`. Ngân sách thử nghiệm hiện tại chưa đủ lớn để chứng minh một kiểu va chạm đã triệt tiêu ở mức dưới $10^{-2}$.

**Núm chỉnh "nghe an toàn hơn" nhưng thực tế phản tác dụng:**
Ứng viên số 1 (`cautionV = 5.0` m/s): Giảm tốc độ thận trọng xuống 5 m/s nghe có vẻ an toàn, nhưng thực tế làm giảm nghiêm trọng quãng đường (`progress` tụt xuống 174.3 m, mất hơn 20%). Việc phanh quá sớm và quá chậm khiến xe ta nằm lại trong vùng nguy hiểm cạnh xe buýt lâu hơn, đồng thời tạo ra chênh lệch tốc độ lớn với các xe phía sau và xe làn bên cạnh, khiến xe bám đuôi TG và xe chuyển làn C2 dễ mất phán đoán và gây nguy hiểm hơn.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định ảnh hưởng lớn nhất trong `space.py`:

1. **Khe chấp nhận của xe chuyển làn (`cutGap`, lognormal median 14.0 m trong Town D):**
   - *Dữ liệu thay thế:* Trích xuất từ camera flycam giao thông (dữ liệu trajectory kiểu HighD / NGSIM) đo đạc tại các tuyến phố đô thị thực tế.
   - *Hướng sai lệch:* Ngoài đời tài xế tại các đô thị đông đúc thường chèn làn với khoảng cách liều lĩnh hơn nhiều (5–8 m). Giả định 14 m là quá rộng, khiến con số va chạm trong mô phỏng đang **thấp hơn và lạc quan hơn thực tế**.
2. **Tỉ lệ người đi bộ ngó qua xe buýt (`pedLook`, Bernoulli 0.90 trong Town D):**
   - *Dữ liệu thay thế:* Khảo sát thị giác quan sát hành vi người đi bộ qua camera an ninh tại các trạm dừng xe buýt.
   - *Hướng sai lệch:* Giả định 90% người đi bộ ngó qua xe buýt là rất cao. Nếu ngoài đời người đi bộ chủ quan bước thẳng ra đường mà không quan sát (tỉ lệ ngó chỉ đạt 40–50%), số vụ chặn vạch và va chạm người đi bộ sẽ tăng vọt; mô phỏng đang ước lượng **quá an toàn so với thực tế**.
3. **Hệ số ma sát đường ướt (`mu = 0.5` chiếm 15% kịch bản trong Town D):**
   - *Dữ liệu thay thế:* Thiết bị đo hệ số bám bánh xe thực địa (British Pendulum Tester / GripTester) trên mặt đường nhựa đô thị trong các điều kiện thời tiết khác nhau.
   - *Hướng sai lệch:* Mặt đường thực tế có thể dính dầu mỡ, sình lầy khi mưa với $\mu < 0.35$. Khi đó quãng đường phanh trượt thực tế dài hơn nhiều, khiến số vụ tai nạn ngoài đời thực sẽ **nhiều hơn con số báo cáo**.

---

## 8. Giới hạn

**Ba điều mà xác suất báo cáo KHÔNG phản ánh:**
1. **Hình học và cấu trúc thế giới cực kỳ đơn giản:** Mô phỏng chỉ có một đoạn phố thẳng hai làn, không có ngã tư có đèn tín hiệu, không có vòng xuyến, không có chướng ngại vật tĩnh ngẫu nhiên (công trường, ổ gà) hay xe đi ngược chiều.
2. **Hành vi tác nhân được đơn giản hóa:** Các xe chỉ lái theo mô hình IDM chuẩn, không có phản ứng bấm còi cảnh báo, không biết lách đánh lái tránh vật cản, và không có các hành vi thất thường như tài xế mất tập trung hay sử dụng điện thoại.
3. **Mô hình cảm biến lý tưởng:** Xe ta được coi là nhận diện chính xác tuyệt đối vị trí và vận tốc của mọi vật thể trong tầm nhìn (bounding box hoàn hảo, không có độ trễ nhận diện, không có điểm mù cột A/B, không bị nhiễu mưa hay ánh sáng chói).

**Nếu có thêm một tuần:**
Tôi sẽ tích hợp mô hình **nhiễu và độ trễ cảm biến (perception latency & detection noise)** cùng mô hình tác nhân **xe hai bánh (xe máy, xe đạp điện luồn lách)** — đặc trưng sống còn của giao thông đô thị Việt Nam — để kiểm thử độ bền vững của bộ điều khiển.

---

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** Người đi bộ bất ngờ băng ngang đường ngoài vạch qua đường từ sau khu vực xe buýt / xe đỗ che khuất tầm nhìn (jaywalker mid-block).
- **Căn cứ:** Planner hiện tại chỉ kích hoạt logic thận trọng (`cautionRange = 55m, cautionV = 7.5m/s`) khi tới gần vạch qua đường cố định ($x = 122$ m). Khi người đi bộ băng cắt ở đoạn giữa phố ($x = 92 - 108$ m), planner không có logic giảm tốc đón đầu, chỉ phát hiện khi người đã bước ra đường và chỉ áp dụng phanh IDM mềm ($b = 2.0\text{ m/s}^2$), dẫn đến đâm phải người đi bộ.
- **Giả định (`hazard.json`):**
  - `space`: `h0_x: [92.0, 108.0]`, `h0_v: [1.3, 2.1]`, `h0_trig: [15.0, 28.0]`.
  - `exposure`: `0.04` lần/chuyến (dựa trên dữ liệu va chạm người đi bộ ngoài giao lộ của NHTSA và SHRP 2 NDS trong khu vực đô thị sầm uất).

- **Kết quả thực nghiệm (`evidence/mine.json`):**
  - Số mẫu hợp lệ: 1 494 / 1 500 (`accepted = 99.60%`).
  - **Policy gốc (baseline):**
    - Tổng va chạm: **29** vụ = **28** `avoidable` (tránh được bằng phanh hết cỡ) + **0** `unavoidable` + **1** `unattributed` (4 vụ ở tốc độ $< 1$ m/s).
    - $P(\text{lỗi planner} \mid \text{rắc rối}) = 1.87 \times 10^{-2}$ [Khoảng Wilson 95%: $1.30 \times 10^{-2}, 2.70 \times 10^{-2}$].
    - Tần suất mỗi chuyến: **$7.50 \times 10^{-4}$** (cận trên $1.08 \times 10^{-3}$).
  - **Policy đã chọn (chosen: `cutinLat = 3.3`):**
    - Tổng va chạm: **27** vụ = **27** `avoidable` + **0** `unavoidable` + **0** `unattributed` (2 vụ ở tốc độ $< 1$ m/s).
    - $P(\text{lỗi planner} \mid \text{rắc rối}) = 1.81 \times 10^{-2}$ [Khoảng Wilson 95%: $1.24 \times 10^{-2}, 2.62 \times 10^{-2}$].
    - Tần suất mỗi chuyến: **$7.23 \times 10^{-4}$** (cận trên $1.05 \times 10^{-3}$).
  - **So sánh:** Tần suất rắc rối này ($7.50 \times 10^{-4}$) cao gấp đôi tần suất va chạm lách xe buýt ở bước C3 ($3.27 \times 10^{-4}$), và việc sửa policy ở bước E hầu như không giải quyết được rắc rối này (vẫn còn 27 vụ avoidable).
  - **Vì sao "tránh được là chuẩn lạc quan":**
    Chuẩn này giả định xe biết có người đi bộ ngay từ khung hình đầu tiên nhìn thấy và lập tức phanh tối đa ở giới hạn bám $\mu g$, nên nó đếm *nhiều hơn* số va chạm mà một planner thực tế có thể tránh được (cận trên của lỗi planner). Ngược lại, thử nghiệm chỉ kiểm tra phanh mà không cho phép đánh lái né sang làn trống, nên ở khía cạnh cơ động nó lại đếm *ít đi*.

- **Hình ảnh minh họa:**
  ![Kịch bản rắc rối người đi bộ băng cắt](evidence/shots/ch10-studio.jpg)

- **Bạn tự lái thì sao:**
  Khi tự lái trong Chương 10, người lái dễ dàng tránh được va chạm vì quan sát thấy khu vực xe buýt đỗ che khuất tầm nhìn sẽ chủ động rà phanh hạ tốc độ từ xa xuống dưới 25 km/h và chuẩn bị sẵn chân phanh (phản xạ phòng thủ), thay vì duy trì 45 km/h như planner tự động.
- **Cách sửa:**
  Không có núm chỉnh nào trong 9 núm hiện tại sửa triệt để được rắc rối này (vì `cautionV` và `cautionRange` chỉ gắn cứng với vạch qua đường ở $x = 122$ m). Planner cần bổ sung **logic nhận thức vùng che khuất (occlusion-aware planning)**: khi phát hiện có vật thể kích thước lớn (xe buýt, xe tải) che khuất tầm nhìn mép đường ở cự ly gần, planner phải tự động kích hoạt chế độ thận trọng hạ tốc độ tối đa xuống $6 - 7$ m/s cho đến khi vượt qua khỏi vùng khuất tầm nhìn.
