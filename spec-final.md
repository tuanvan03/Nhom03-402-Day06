# SPEC — AI Product Hackathon

**Nhóm:** Group_3_E402

**Track:**  Vinmec 

---

## Problem statement
Lễ tân và tổng đài hiện phải thủ công hỏi triệu chứng, tra cứu và tư vấn chuyên khoa cho từng bệnh nhân — mỗi ca mất 3–5 phút, dễ gây hàng chờ kéo dài khi lượng người tăng cao, ảnh hưởng đến trải nghiệm bệnh nhân và tạo áp lực lớn cho nhân viên tiếp nhận. Hệ thống AI được đề xuất để đảm nhận bước sàng lọc ban đầu — hỏi triệu chứng, gợi ý chuyên khoa phù hợp và đề xuất xếp phòng khám — giúp rút ngắn thời gian xử lý và tăng độ chính xác trong điều hướng, trong khi lễ tân vẫn là người xác nhận và quyết định cuối cùng.

---

## 1. AI Product Canvas

|             | Value                                      | Trust                                                                                                                                                                                                                                    | Feasibility                                                                         |
| ----------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **Câu hỏi** | User nào? Pain gì? AI giải gì?             | Khi AI sai thì sao? User sửa bằng cách nào?                                                                                                                                                                                              | Cost/latency bao nhiêu? Risk chính?                                                 |
| **Trả lời** | Lễ tân mất 5–10 phút/lần sắp xếp bệnh nhân | AI gắn sai nhãn → Thời gian phải chờ lễ tân chọn chuyên khoa, xếp phòng phù hợp từ mô tả triệu chứng của người dùng. User (lễ tân) phải xác nhận và chọn lại phòng phù hợp trong ticket mẫu được AI xuất ra, hệ thống học từ correction. | ~0,01 USD/API call, latency <10s. Risk: Triệu chứng mô tả mơ hồ, phân vào sai khoa. |

**Automation hay augmentation?** ☐ Automation · ☑ Augmentation
Justify: Augmentation — lễ tân luôn nhìn thấy gợi ý và xác nhận trước khi điều hướng bệnh nhân. AI sai thì lễ tân sửa trong 1 giây, không ảnh hưởng flow.

**Learning signal:**

1. User correction đi vào đâu? -> Log cặp (chuỗi triệu chứng -> chuyên khoa lễ tân đã chọn thay thế) vào correction database. Dùng làm fine-tuning data hoặc few-shot examples cho prompt sau.
2. Product thu signal gì để biết tốt lên hay tệ đi? -> (a) Tỷ lệ lễ tân giữ nguyên gợi ý AI (acceptance rate); (b) Tỷ lệ bệnh nhân phải chuyển khoa sau khi đã vào khám (downstream error rate); (c) Thời gian xử lý trung bình mỗi lượt.
3. Data thuộc loại nào? Domain-specific và Human-judgment.
Có marginal value không? **Có** — dữ liệu triệu chứng -> chuyên khoa theo ngữ cảnh Vinmec (tên khoa, quy trình nội bộ, phân loại bệnh nhân VIP/thường) là domain-specific, model nền chưa biết. Mỗi correction của lễ tân là human-judgment label có giá trị cao.

---

## 2. User Stories — 4 paths

Mỗi feature chính = 1 bảng. AI trả lời xong → chuyện gì xảy ra?

### Feature 1: *AI gợi ý chuyên khoa và xếp phòng*

**Trigger:** *Bệnh nhân nhập triệu chứng → AI phân tích → gợi ý chuyên khoa + xếp phòng*

| Path                           | Câu hỏi thiết kế                                           | Mô tả                                                                                  |
| ------------------------------ | ---------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Happy — AI đúng, tự tin        | User thấy gì? Flow kết thúc ra sao?                        | *AI gợi ý “Nội tổng quát”, lễ tân thấy hợp lý → chọn → điều hướng bệnh nhân*           |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | *Hiện 2–3 chuyên khoa + confidence %, lễ tân chọn*                                     |
| Failure — AI sai               | User biết AI sai bằng cách nào? Recover ra sao?            | *AI gợi ý “Da liễu” nhưng triệu chứng đau ngực → lễ tân nhận ra → đổi sang “Tim mạch”* |
| Correction — user sửa          | User sửa bằng cách nào? Data đó đi vào đâu?                | *Lễ tân chọn lại chuyên khoa → lưu log → dùng để improve model*                        |

### Feature 2: *Cảnh báo triệu chứng nguy hiểm*
(Tuy nhiên nhóm thống nhất chỉ tập trung vào một feature chính là AI gợi ý chuyên khoa)
**Trigger:** *AI phát hiện dấu hiệu nghiêm trọng từ triệu chứng*

| Path                           | Câu hỏi thiết kế                           | Mô tả                                                                                    |
| ------------------------------ | ------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Happy — AI đúng, tự tin        | Alert hiển thị thế nào để không bị bỏ qua? | *AI detect “đau ngực, khó thở” → hiện cảnh báo đỏ “Có thể cấp cứu” → ưu tiên xử lý ngay* |
| Low-confidence — AI không chắc | Khi chưa chắc là emergency?                | *Hiện cảnh báo vàng + khuyến nghị kiểm tra thêm*                                         |
| Failure — AI sai               | Nếu AI không nhận ra?                      | *Lễ tân phát hiện dấu hiệu nguy hiểm → chuyển khẩn cấp → đánh dấu missed case*           |
| Correction — user sửa          | Data đi đâu?                               | *Case được log → update rule-based + training để giảm false negative*                    |

---

## 3. Eval metrics + threshold

**Optimize precision hay recall?** ☐ Precision · ☑ Recall
Tại sao? 

Vì AI đảm nhận bước sàng lọc ban đầu (hỏi triệu chứng → gợi ý chuyên khoa), mục tiêu quan trọng nhất là không bỏ sót chuyên khoa phù hợp, đặc biệt với ca nghiêm trọng. Recall cao giúp giảm under-triage, tránh tình trạng bệnh nhân bị điều hướng sai hoặc chậm trễ do AI miss gợi ý đúng.

NNếu ưu tiên Precision mà Recall thấp → AI hay bỏ sót chuyên khoa đúng → lễ tân/tổng đài không nhận được gợi ý phù hợp → bệnh nhân vẫn phải chờ lâu hoặc đi sai hướng, không giải quyết được bài toán hàng chờ và áp lực nhân viên.

| Metric                                                                   | Threshold                     | Red flag (dừng khi)                                                                |
| ------------------------------------------------------------------------ | ----------------------------- | ---------------------------------------------------------------------------------- |
| Recall@3                                                                 | ≥ 95%                         | < 90% trong 7 ngày liên tục                                                        |
| Precision@3 (để kiểm soát gợi ý thừa)                                    | ≥ 60%                         | < 45% (quá nhiều gợi ý rác)                                                        |
| Coverage (tỷ lệ có gợi ý)                                                | ≥ 95%                         | < 90% (AI từ chối trả lời hoặc “Tôi không chắc”)                                   |
| Average Processing Time Reduction (thời gian lễ tân xử lý mỗi bệnh nhân) | ≥ 60%                         | < 40% (không cải thiện bottleneck “đọc, quyết định và thao tác nhập máy thủ công”) |
| CSAT (User Satisfaction - lễ tân)                                        | ≥ 4.0/5                       | < 3.5/5 (lễ tân phàn nàn nhiều về gợi ý sai hoặc không hữu ích)                    |
| Safety Red Flag (bổ sung)                                                | Miss rate nghiêm trọng ≤ 1.5% | > 3% ca nghiêm trọng (Đau ngực, khó thở, xuất huyết…)                              |

Recall@3: Metric cốt lõi, đo trực tiếp success criteria “phân loại đúng trong đa số trường hợp”.

Precision@3: Tránh việc gợi ý sai quá nhiều

Coverage: Đảm bảo AI đưa ra gợi ý ngay cả triệu chứng dài/không rõ ràng.

Average Processing Time Reduction: Đo lường trực tiếp bottleneck lớn nhất (xử lý nhanh hơn khi lượng bệnh nhân đông).

CSAT: Khảo sát: “Bạn hài lòng với gợi ý chuyên khoa, mức độ khẩn cấp và bác sĩ của AI không?” (1–5 sao).

Safety Red Flag: Monitor riêng (rule-based + human review) – ưu tiên under-triage vì rủi ro sức khỏe cao nhất.

---

## 4. Top 3 failure modes

*Liệt kê cách product có thể fail — không phải list features.*
*"Failure mode nào user KHÔNG BIẾT bị sai? Đó là cái nguy hiểm nhất."*

| #   | Trigger                                                                     | Hậu quả                                                                                                                                                                | Mitigation                                                                                                                                                                                                                                                                             |
| --- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Bệnh nhân kể lan man, triệu chứng chồng chéo                                | AI miss chuyên khoa đúng hoặc đưa ra gợi ý kém → Recall thấp. Lễ tân nhận gợi ý không hữu ích → vẫn mất thời gian tra cứu thủ công hoặc điều hướng sai.                | - Ưu tiên high-recall (luôn trả top-3 + confidence score rõ ràng)<br>- Nếu confidence thấp (<60%) → tự động gợi ý hỏi thêm 1–2 triệu chứng then chốt<br>- Active learning: thu thập correction từ lễ tân để retrain định kỳ<br>- Tối ưu prompt với few-shot examples từ dữ liệu Vinmec |
| 2   | Triệu chứng nguy hiểm được mô tả nhẹ nhàng, gián tiếp, hoặc không điển hình | AI không phát hiện mức độ khẩn cấp → không gợi ý cấp cứu hoặc chuyên khoa phù hợp. Lễ tân quá tải không để ý hoặc không biết → chậm trễ điều trị, rủi ro sức khỏe cao. | - Hard-coded red-flag rules (danh sách triệu chứng nguy hiểm cố định, ưu tiên cực cao) <br>- Tất cả red-flag cases → bắt buộc gợi ý Cấp cứu + cảnh báo đỏ nổi bật, yêu cầu lễ tân xác nhận<br>- Daily human review + log tất cả red-flag cases                                         |
| 3   | Model quá conservative hoặc bias trên case hiếm/không điển hình             | Recall thấp trên một số nhóm triệu chứng → bỏ sót gợi ý đúng. Lễ tân dựa vào AI nhưng miss option quan trọng.                                                          | - Tập trung training data trên case khó + rare diseases <br>- Active learning trên case lễ tân/bác sĩ sửa <br>- Weekly retrain với trọng số recall<br>                                                                                                                                 |

---
## 5. ROI – 3 kịch bản

|                | **Conservative**                                                                 | **Realistic**                                                                    | **Optimistic**                                                                                    |
| -------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Assumption** | 100 bệnh nhân/ngày<br>lễ tân sử dụng 40% thời gian<br>Giảm thời gian xử lý > 24% | 500 bệnh nhân/ngày<br>lễ tân sử dụng 60% thời gian<br>Giảm thời gian xử lý > 36% | 2000 bệnh nhân/ngày (bệnh viện lớn)<br>lễ tân sử dụng 90% thời gian<br>Giảm thời gian xử lý > 54% |
| **Cost**       | 0,4 USD/ngày (~10.000 VND) inference + maintenance                               | 2 USD/ngày (~50.000 VND)                                                         | 10 USD/ngày (~250.000 VND)                                                                        |
| **Benefit**    | Giảm ~2 – 3 giờ làm việc lễ tân/ngày<br>                                         | Giảm ~3 - 4,5 giờ làm việc lễ tân/ngày<br>                                       | Giảm ~4,5 - 6,5 giờ làm việc lễ tân/ngày<br> Giảm khiếu nại từ bệnh nhân                          |
| **Net**        | +0,6 - 3 triệu VND/ngày                                                          | +0,8 - 4,5 triệu VND/ngày                                                        | +3,3 – 18 triệu VND/ngày                                                                          |

**Kill criteria (tiêu chí dừng dự án):**
- Net âm (cost > benefit) trong **2 tháng liên tục**.
- Safety Red Flag (under-triage nghiêm trọng) > 3% trong 2 tuần liên tục.
- CSAT của lễ tân < 3.5/5 trong 4 tuần liên tục.
- Lễ tân không chấp nhận sử dụng (adoption rate < 40% sau 1 tháng).

---

## 6. Mini AI spec (1 trang)

- Hiện tại, lễ tân và tổng đài phải tự hỏi triệu chứng, tra cứu và quyết định chuyên khoa cho từng bệnh nhân, mỗi ca mất khoảng 5–10 phút. Khi lượng bệnh nhân tăng, quy trình này nhanh chóng trở thành nút thắt cổ chai, gây ùn tắc, kéo dài thời gian chờ và tạo áp lực lớn lên nhân viên, đồng thời làm giảm trải nghiệm chung của bệnh nhân.

- Sản phẩm đề xuất giải quyết vấn đề này bằng cách đưa AI vào bước sàng lọc ban đầu. AI sẽ tự động hỏi các triệu chứng cơ bản, phân tích và gợi ý chuyên khoa phù hợp, thậm chí đề xuất phòng khám tương ứng. Tuy nhiên, hệ thống chỉ đóng vai trò hỗ trợ **augmentation**, không thay thế con người — lễ tân vẫn là người xác nhận và đưa ra quyết định cuối cùng.

- Về chất lượng, hệ thống được thiết kế ưu tiên **recall** cao để tránh bỏ sót các ca bệnh quan trọng, chấp nhận việc đôi khi đưa ra nhiều gợi ý khi không chắc chắn. Kèm theo đó là hiển thị mức độ confidence và cơ chế hỏi thêm hoặc cảnh báo rule-based trong các trường hợp có dấu hiệu nguy hiểm.

- Rủi ro chính nằm ở việc AI có thể gợi ý sai trong các ca mơ hồ hoặc nghiêm trọng, nên cần luôn giữ lễ tân trong vòng kiểm soát, đồng thời thiết kế các cơ chế fallback rõ ràng để giảm thiểu sai sót.

![alt text](image.png)

## Phân công
- Khải: Canvas + Problem statement
- Tuấn: User stories 4 paths
- Trí: Eval metrics + ROI + threshold
- Bình: Top 3 failures test + Mini AI spec
