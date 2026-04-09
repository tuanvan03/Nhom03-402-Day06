# Prototype — AI triage Vinmec

## Mô tả
Chatbot hỏi bệnh nhân 3-5 câu về triệu chứng, gợi ý top 3 chuyên khoa phù hợp
kèm confidence score. Bệnh nhân chọn hoặc gặp lễ tân.

## Level: Mock prototype
- UI build bằng Claude Artifacts (HTML/CSS/JS)
- 1 flow chính chạy thật với Gemini API: nhập triệu chứng → nhận gợi ý khoa

## Links
- Prototype: https://claude.site/artifacts/xxx
- Prompt test log: xem file `prototype/prompt-tests.md`
- Video demo (backup): https://drive.google.com/xxx

## Tools
- UI: Claude Artifacts
- AI: Google Gemini 2.0 Flash (via Google AI Studio)
- Prompt: system prompt + few-shot examples cho 10 triệu chứng phổ biến

## Phân công
| Thành viên | Phần                                      | Output                                               |
| ---------- | ----------------------------------------- | ---------------------------------------------------- |
| An         | Canvas + failure modes                    | spec/spec-final.md phần 1, 4                         |
| Bình       | User stories 4 paths + prompt engineering | spec/spec-final.md phần 2, prototype/prompt-tests.md |
| Châu       | Eval metrics + ROI + demo slides          | spec/spec-final.md phần 3, 5, demo/slides.pdf        |
| Dũng       | UI prototype + demo script                | prototype/, demo/demo-script.md                      |