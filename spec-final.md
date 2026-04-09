# SPEC — AI Product Hackathon

**Nhóm:** Group_3_E402

**Track:**  Vinmec 

---

## Problem statement
- Lễ tân và tổng đài hiện phải thủ công hỏi triệu chứng, tra cứu và tư vấn chuyên khoa cho từng bệnh nhân — mỗi ca mất 5–10 phút, dễ gây hàng chờ kéo dài khi lượng người tăng cao, ảnh hưởng đến trải nghiệm bệnh nhân và tạo áp lực lớn cho nhân viên tiếp nhận. Hệ thống AI được đề xuất để đảm nhận bước sàng lọc ban đầu — hỏi triệu chứng, gợi ý chuyên khoa phù hợp và đề xuất xếp phòng khám — giúp rút ngắn thời gian xử lý và tăng độ chính xác trong điều hướng, trong khi lễ tân vẫn là người xác nhận và quyết định cuối cùng.

---

## 1. AI Product Canvas

|   | Value | Trust | Feasibility |
|---|-------|-------|-------------|
| **Câu hỏi** | User nào? Pain gì? AI giải gì? | Khi AI sai thì sao? User sửa bằng cách nào? | Cost/latency bao nhiêu? Risk chính? |
| **Trả lời** | Lễ tân | Bệnh nhân và lễ tân đau. Thời gian phải chờ lễ tân chọn chuyên khoa, xếp phòng phù hợp từ mô tả triệu chứng của người dùng. AI từ danh sách các triệu chứng của bệnh nhân -> Đưa ra lựa chọn chuyên khoa và gợi ý xếp phòng phù hợp. | Chi phí API call, latency <3s. | Risk: Triệu chứng mô tả mơ hồ, nhiều khoa overlap.|

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

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | *AI gợi ý “Nội tổng quát”, lễ tân thấy hợp lý → chọn → điều hướng bệnh nhân* |
| Low-confidence — AI không chắc | System báo "không chắc" bằng cách nào? User quyết thế nào? | *Hiện 2–3 chuyên khoa + confidence %, lễ tân chọn* |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | *AI gợi ý “Da liễu” nhưng triệu chứng đau ngực → lễ tân nhận ra → đổi sang “Tim mạch”*  |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | *Lễ tân chọn lại chuyên khoa → lưu log → dùng để improve model* |

### Feature 2: *Cảnh báo triệu chứng nguy hiểm*

**Trigger:** *AI phát hiện dấu hiệu nghiêm trọng từ triệu chứng*

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | Alert hiển thị thế nào để không bị bỏ qua? | *AI detect “đau ngực, khó thở” → hiện cảnh báo đỏ “Có thể cấp cứu” → ưu tiên xử lý ngay* |
| Low-confidence — AI không chắc | Khi chưa chắc là emergency? | *Hiện cảnh báo vàng + khuyến nghị kiểm tra thêm* |
| Failure — AI sai | Nếu AI không nhận ra? | *Lễ tân phát hiện dấu hiệu nguy hiểm → chuyển khẩn cấp → đánh dấu missed case*  |
| Correction — user sửa | Data đi đâu? | *Case được log → update rule-based + training để giảm false negative* |

---

## 3. Eval metrics + threshold

**Optimize precision hay recall?** ☐ Precision · ☑ Recall
Tại sao? 

Vì AI chỉ gợi ý (không quyết định cuối cùng), và tổn thất lớn nhất trong Problem Statement là phân loại sai ca nặng dẫn đến chậm trễ xử lý, rủi ro sức khỏe bệnh nhân. Recall cao giúp giảm nguy cơ bỏ sót chuyên khoa phù hợp (under-triage), đặc biệt với triệu chứng mơ hồ, không điển hình hoặc nghiêm trọng được mô tả nhẹ.
Lễ tân có thể lọc bớt gợi ý thừa, nhưng nếu AI miss chuyên khoa đúng thì bệnh nhân rất dễ bị điều hướng sai mà lễ tân không nhận ra kịp.

Nếu sai ngược lại thì chuyện gì xảy ra? 

Nếu ưu tiên Precision cao nhưng Recall thấp → AI hay miss chuyên khoa đúng (ví dụ: triệu chứng đau ngực + khó thở chỉ gợi ý Nội thay vì Tim mạch/Cấp cứu). Lễ tân và bệnh nhân không biết có gợi ý tốt hơn → ca nặng bị chậm trễ, rủi ro sức khỏe tăng cao, trải nghiệm bệnh nhân kém, lễ tân bị trách nhiệm.

| Metric | Threshold | Red flag (dừng khi) |
|--------|-----------|---------------------|
| Recall@1 (chuyên khoa đúng nhất nằm trong top 1)  | ≥ 85%  | < 70% trong 7 ngày liên tục  |
| Recall@3 (chuyên khoa đúng nằm trong top 3) | ≥ 95% | < 90% trong 7 ngày liên tục |
| Precision@3 (để kiểm soát gợi ý thừa) | ≥ 60% |< 45% (quá nhiều gợi ý rác)
| Coverage (tỷ lệ có gợi ý) | ≥ 90% | < 85% (AI từ chối trả lời hoặc “Tôi không chắc”) |
| Average Processing Time Reduction (thời gian lễ tân xử lý mỗi bệnh nhân) | ≥ 40% | < 25% (không cải thiện bottleneck “đọc, quyết định và thao tác nhập máy thủ công”) |
| CSAT (User Satisfaction - lễ tân) | ≥ 4.5/5 | < 4.0/5 (lễ tân phàn nàn nhiều về gợi ý sai hoặc không hữu ích) |
| Safety Red Flag (bổ sung) | Miss rate nghiêm trọng ≤ 1.5% | > 3% ca nghiêm trọng (Đau ngực, khó thở, xuất huyết…) |

Recall@1 và Recall@3: Metric cốt lõi, đo trực tiếp success criteria “phân loại đúng trong đa số trường hợp”.

Precision@3: Tránh việcc gợi ý sai quá nhiều

Coverage: Đảm bảo AI hỗ trợ tốt ngay cả triệu chứng dài/không rõ ràng.

Average Processing Time Reduction: Đo lường trực tiếp bottleneck lớn nhất (xử lý nhanh hơn khi lượng bệnh nhân đông).

CSAT: Khảo sát: “Bạn hài lòng với gợi ý chuyên khoa, mức độ khẩn cấp và bác sĩ của AI không?” (1–5 sao).

Safety Red Flag: Monitor riêng (rule-based + human review) – ưu tiên under-triage vì rủi ro sức khỏe cao nhất.
---

## 4. Top 3 failure modes

*Liệt kê cách product có thể fail — không phải list features.*
*"Failure mode nào user KHÔNG BIẾT bị sai? Đó là cái nguy hiểm nhất."*

| # | Trigger | Hậu quả | Mitigation |
|---|---------|---------|------------|
| 1 | Triệu chứng mơ hồ, dài dòng, không điển hình hoặc kết hợp nhiều hệ cơ quan | AI miss chuyên khoa đúng → Recall thấp. Lễ tân không nhận được gợi ý phù hợp → điều hướng sai khoa mà không biết. | - Tối ưu model theo hướng high-recall (temperature cao hơn, broader retrieval)<br>- Luôn trả top-3 + confidence<br>- Nếu confidence thấp → khuyến khích lễ tân hỏi thêm triệu chứng<br> |
| 2 | Triệu chứng nghiêm trọng nhưng mô tả nhẹ hoặc dùng từ thông thường | AI miss mức độ khẩn cấp và chuyên khoa cấp cứu/Tim mạch/Thần kinh… → Under-triage nghiêm trọng. Bệnh nhân và lễ tân không biết → rủi ro sức khỏe cao. | - Hard-coded red-flag rules (ưu tiên cực cao)<br>- Tất cả red-flag cases → bắt buộc gợi ý Cấp cứu + cảnh báo nổi bật<br>- Human review daily red-flag cases<br> |
| 3 | Model quá conservative hoặc bias trên case hiếm/không điển hình | Recall thấp trên một số nhóm triệu chứng → bỏ sót gợi ý đúng. Lễ tân dựa vào AI nhưng miss option quan trọng. | - Tập trung training data trên case khó + rare diseases <br>- Active learning trên case lễ tân/bác sĩ sửa <br>- Weekly retrain với trọng số recall<br> |

---
## 5. ROI – 3 kịch bản

|   | **Conservative** | **Realistic** | **Optimistic** |
|---|------------------|---------------|----------------|
| **Assumption** | 100 bệnh nhân/ngày<br>lễ tân sử dụng 40% thời gian<br>Giảm thời gian xử lý > 16% | 500 bệnh nhân/ngày<br>lễ tân sử dụng 60% thời gian<br>Giảm thời gian xử lý > 24% | 2000 bệnh nhân/ngày (bệnh viện lớn)<br>lễ tân sử dụng 90% thời gian<br>Giảm thời gian xử lý >36% |
| **Cost** | 1 USD/ngày (~25.000 VND) inference + maintenance | 7,5 USD/ngày (~200,000 VND) | 45 USD/ngày (~1,2 triệu VND) |
| **Benefit** | Giảm ~1,2–2 giờ làm việc lễ tân/ngày<br> | Giảm ~1,8–3 giờ làm việc lễ tân/ngày<br> | Giảm ~3–4,5 giờ làm việc lễ tân/ngày<br> Giảm khiếu nại từ bệnh nhân |
| **Net** | **+0,025 – 0,05 triệu VND/ngày** | **+0,2 – 0,8 triệu VND/ngày** | **+1,2 – 8 triệu VND/ngày** |

**Kill criteria (tiêu chí dừng dự án):**
- Net âm (cost > benefit) trong **2 tháng liên tục**.
- Safety Red Flag (under-triage nghiêm trọng) > 3% trong 7 ngày liên tục.
- CSAT của lễ tân < 4.0/5 trong 4 tuần liên tục.
- Recall@1 < 70% trong 7 ngày liên tục mà không cải thiện sau retrain.
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