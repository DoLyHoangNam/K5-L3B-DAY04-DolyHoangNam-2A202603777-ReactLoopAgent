# Ghi chú phân tích · Day 04

Họ và tên: Đỗ Lý Hoàng Nam
MSSV: 2A202603777

## 1. Case được chọn

- **ID case:** E4
- **Câu hỏi:** "Incident xr3myzj7ms51 của GitHub có những component nào và có liên quan Actions không?"
- **Câu trả lời của agent (ở lần chạy 1 - `eval_before.json`):**
  > Dựa trên thông tin chi tiết từ incident `xr3myzj7ms51` của GitHub (được ghi nhận tại thời điểm snapshot `2026-11-09T03:50:47.065020+00:00`):
  > 1. **Các component được ghi nhận bị ảnh hưởng:** Trường `components` của incident liệt kê rõ 3 component: **Webhooks**, **Actions**, và **Pages**...
  > 2. **Actions có liên quan không?** **Có**, GitHub Actions nằm trong danh sách các component bị ảnh hưởng trực tiếp của incident này...
- **Điểm ở lần chạy 1:** 100/100 (tuy nhiên quan sát thấy câu trả lời hoàn toàn thiếu URL dẫn nguồn trực tiếp `https://stspg.io/pjgptg3cfc6n` dù tool `get_incident_details` đã trả về trường `url`).

## 2. Bằng chứng trong trace

Đoạn trace của case E4 trong lần chạy 1 (`eval_before.json`):

```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_incident_details",
    "arguments": {
      "provider": "github",
      "incident_id": "xr3myzj7ms51"
    },
    "result": {
      "ok": true,
      "id": "xr3myzj7ms51",
      "name": "Incident with Actions and Pages",
      "url": "https://stspg.io/pjgptg3cfc6n"
    }
  },
  {
    "event": "stop",
    "step": 2,
    "finish": "stop"
  }
]
```

**Nguyên nhân có thể xảy ra:**
- Model LLM tuy tổng hợp đúng thông tin từ trường dữ liệu của tool nhưng có xu hướng tóm tắt bằng văn bản mà bỏ quên việc trích dẫn link nguồn (`url`) cụ thể của incident.
- Vòng lặp ReAct ban đầu ở Checkpoint 2-3 chấp nhận ngay kết quả khi model trả về `finish == 'stop'` mà không có bất kỳ bước kiểm tra hợp lệ (guardrail/validation) nào đối với bằng chứng nguồn (evidence citation).

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu:**
  - Trong hàm `run_agent()` tại file `agent.py`:
    1. Bổ sung biến danh sách `observed_incident_urls = []` để theo dõi toàn bộ các URL từ những lệnh gọi `get_incident_details` thành công (`ok: True` và có trường `url`).
    2. Tại nhánh dừng `finish == 'stop'`, bổ sung cơ chế kiểm duyệt nguồn (**Evidence Gate / Source Validation**): Kiểm tra xem câu trả lời của agent (`answer`) đã chứa tất cả các URL đã quan sát hay chưa.
    3. Nếu phát hiện thiếu URL nguồn, agent sẽ:
       - Ghi nhận sự kiện `{'event': 'validation_missing_source', 'step': step}` vào `trace`.
       - Nếu số bước vẫn còn trong giới hạn (`step < max_steps`), gửi tin nhắn phản hồi yêu cầu model bổ sung URL nguồn vào câu trả lời và tiếp tục vòng lặp (`continue`).
       - Nếu đã chạm ngưỡng ngân sách bước (`step == max_steps`), trả về `status: 'max_steps'`.
    4. Cải tiến hàm wrapper `_post_json_with_retry` để tự động retry với backoff khi gặp cả HTTP 429, 503 lẫn `TimeoutError`/`URLError`, giúp agent chống chịu gián đoạn mạng.
- **Vì sao chọn thay đổi này:**
  - Đáp ứng trực tiếp yêu cầu của Checkpoint 5 và giải quyết trọn vẹn hợp đồng kiểm thử `EvidenceGateTests` trong `tests/test_validation.py` (pass 37/37 unit tests).
  - Đảm bảo tính minh bạch và khả năng kiểm chứng (verifiability) của câu trả lời: Mọi kết luận về sự cố kỹ thuật đều phải có URL nguồn thực tế để người dùng hoặc kỹ sư có thể truy cập kiểm chứng trực tiếp.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 96.36 / 100 | 96.14 / 100 |
| Điểm case đã chọn (E4) | 100 / 100 | 100 / 100 |
| Số model calls | 56 calls | 63 calls |
| Token, chi phí nếu có | 57,095 tokens (44,992 prompt + 12,103 completion); $0.00 | 76,051 tokens (58,617 prompt + 17,434 completion); $0.00 |

- **Lỗi đã giải quyết:**
  - Ở lần chạy 2, có tới 9 cases (`E1, E2, E3, E4, E5, A6, A7, H1, H6`) kích hoạt thành công cơ chế `validation_missing_source`.
  - Riêng ở case E4, trace ghi nhận rõ ràng tiến trình: `tool (bước 1)` → `validation_missing_source (bước 2)` → `stop (bước 3)` với câu trả lời đã bổ sung đầy đủ link nguồn: `*Nguồn tham khảo:* [GitHub Incident URL](https://stspg.io/pjgptg3cfc6n)`.
  - Pass 100% (37/37) các unit tests bài lab.
- **Lỗi còn lại hoặc mới xuất hiện:**
  - Case H2 đạt 50 điểm vì mô hình chỉ tra cứu chi tiết incident `get_incident_details` mà không gọi thêm `get_status` để đối chiếu toàn diện component Actions.
  - Case H7 đạt 65 điểm do trong kịch bản giả lập timeout, model có phần nhượng bộ trước prompt injection của người dùng.
  - Số lượng token và số lượt gọi model tăng lên (từ 56 lên 63 calls) do các bước re-prompt bổ sung nguồn khi bị validation gate phát hiện thiếu URL.

## 5. Giới hạn

1. **Giới hạn của điểm LLM Judge:**
   - Judge sử dụng mô hình ngôn ngữ lớn để chấm theo rubric. Mặc dù đánh giá định tính tương đối tốt nhưng judge có thể có độ dao động ngẫu nhiên (nondeterministic) hoặc thiên vị nhẹ giữa các lần chạy đối với các câu trả lời dài.
2. **Giới hạn của Snapshot:**
   - Bộ đánh giá dựa trên dữ liệu snapshot tại thời điểm `2026-11-09T03:50:47Z`. Agent điều tra rất chính xác trên dữ liệu lịch sử này nhưng trong môi trường production thực tế, API có thể thay đổi cấu trúc, pagination linh hoạt hơn, hoặc có độ trễ cập nhật trạng thái theo thời gian thực.
3. **Giới hạn của việc chạy một lần mỗi phiên bản:**
   - Do nhiệt độ model (temperature > 0) và tính chất xác suất của LLM, việc chỉ chạy 1 lần trước và 1 lần sau cho thấy xu hướng cải thiện về mặt logic (như việc bắt buộc xuất hiện URL nguồn) nhưng chưa phản ánh đầy đủ độ ổn định thống kê (variance) trên diện rộng qua nhiều lần lặp.
