# Ghi chú phân tích · Day 04

- Họ và tên: Vũ Lê Đức
- MSSV: 2A202603683

## 1. Case được chọn

- ID case: A5
- Câu trả lời của agent:
```md
Thông tin từ incident **h02n9572mx9y** có thể được tóm tắt như sau:

- **Tình trạng**: Vấn đề đã được giải quyết.
- **Thời gian**: Bắt đầu từ 17:40 UTC ngày 04/11/2026, được cập nhật lần cuối vào 01:10 UTC ngày 07/11/2026.
- **Nội dung cập nhật mới nhất**: Cập nhật này không nêu chi tiết nguyên nhân mà chỉ hướng dẫn người đọc tham khảo một incident liên quan khác ([xr3myzj7ms51](https://www.githubstatus.com/incidents/xr3myzj7ms51)): "Please refer to the combined summary in this related incident."

Các cập nhật liên quan đến incident **h02n9572mx9y**:
- **Đang điều tra** (17:40 UTC): Đang xem xét các báo cáo về việc giảm khả năng sẵn có cho Actions.
- **Giám sát** (18:07 UTC): Việc giảm hiệu năng đã được giảm thiểu và đang được giám sát.

### Thông tin từ incident liên quan **xr3myzj7ms51**:
- **Tình trạng**: Vấn đề đã được giải quyết.
- **Thời gian**: Bắt đầu từ 16:05 UTC ngày 04/11/2026, kết thúc vào 01:04 UTC ngày 07/11/2026.
- **Nguyên nhân**: Dưới đây là thông tin được xác nhận:
  - Vấn đề xảy ra do một thay đổi cấu hình trong một dịch vụ định tuyến sự kiện nội bộ đã ngừng giao hàng của một số luồng sự kiện.
  - Thời gian xảy ra sự cố bao gồm hai khoảng thời gian: từ 15:49 UTC đến 16:48 UTC và từ 17:20 UTC đến 17:55 UTC, với độ trễ lên đến 25 phút cho trong các workflow của Actions.

### Điều chưa được xác nhận:
- Incident **h02n9572mx9y** không cung cấp thêm thông tin chi tiết về tác động cụ thể của sự cố trên nền tảng Actions, ngoài việc xác nhận rằng vấn đề đã được giảm thiểu và tình trạng hiện tại là ổn định.

**Nguồn**: 
- Thông tin từ incident **h02n9572mx9y**: [GitHub Status](https://stspg.io/7pwv28hvkmz8)
- Thông tin từ incident **xr3myzj7ms51**: [GitHub Status](https://stspg.io/pjgptg3cfc6n)
```
- Điểm ở lần chạy 1: 33

## 2. Bằng chứng trong trace

Đây là một case tốt vì lỗi không đơn thuần nằm ở việc thiếu thông tin. Agent đã gọi tool thành công, nhận được giới hạn dữ liệu rõ ràng, nhưng vẫn đưa ra kết luận vượt quá những gì bằng chứng cho phép.

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[Step 1] Tool: get_status
Arguments:
{
    "provider": "github",
    "component": "Actions"
}

Result:
{
    "matching_incidents_in_last_50": 23,
    "note": "Chỉ tìm theo chuỗi trong 50 incident gần nhất; không tìm thấy không chứng minh không có sự cố.",
    "source": "https://www.githubstatus.com"
}

[Step 2] Tool: get_incident_details
Arguments:
{
    "provider": "github",
    "incident_id": "t89g2q1800fr"
}

Result:
{
    "ok": true,
    "id": "t89g2q1800fr",
    "name": "Incident with Actions",
    "status": "resolved",
    "created_at": "2026-11-05T16:50:11.086Z",
    "url": "https://stspg.io/lv4q3xm9r0rk"
}

[Step 3] Model call:
finish = stop

[Answer]: Không có sự cố nào được ghi nhận cho GitHub Actions trong năm 2025.
```

**Nguyên nhân có thể xảy ra**:
- Agent không tuân thủ giới hạn dữ liệu được trả về từ tool. Mặc dù tool đã thông báo chỉ tìm kiếm trong 50 incident gần nhất và không tìm thấy không có nghĩa là không tồn tại sự cố, agent vẫn kết luận không có incident trong năm 2025.
- Nguyên nhân chính là nhầm lẫn giữa không có bằng chứng và bằng chứng chứng minh không tồn tại.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  + Bổ sung cơ chế kiểm tra giới hạn dữ liệu lịch sử trong `agent.py`.
  + Kiểm tra thông tin giới hạn tìm kiếm từ kết quả `get_status`.
  + Khi người dùng yêu cầu danh sách đầy đủ nhưng tool chỉ cung cấp 50 incident gần nhất, agent phải nêu rõ giới hạn dữ liệu và không được kết luận rằng không có incident.
  + Bổ sung test trong `tests/test_validation.py` để kiểm tra hành vi validation.

- Vì sao chọn thay đổi này: Agent đã nhận được thông tin về giới hạn dữ liệu nhưng không sử dụng đúng thông tin đó khi tạo câu trả lời cuối cùng. Việc bổ sung validation giúp agent tránh đưa ra kết luận không có bằng chứng, đồng thời đảm bảo câu trả lời phản ánh chính xác phạm vi dữ liệu mà tool cung cấp.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 75.50/100 | 84.68/100 |
| Điểm case đã chọn | 33/100 | 60/100 |
| Số model calls | 3 | 3 |
| Token, chi phí nếu có | 8.382 tokens - $0.00172170 | 8,479 tokens - $0.00175425 |

- Lỗi đã giải quyết:
  + Điểm A5 tăng từ 33 lên 60.
  + Điểm Factuality tăng từ 5 lên 20; Boundaries tăng từ 3 lên 10.
  + Câu trả lời after đã đề cập rằng dữ liệu hiện tại thuộc năm 2026 và không có dữ liệu cụ thể cho năm 2025.
- Lỗi còn lại hoặc mới xuất hiện:
  + Agent vẫn kết luận không có incident GitHub Actions trong năm 2025, dù snapshot chỉ tìm kiếm trong 50 incident gần nhất.
  + Agent vẫn đưa ra danh sách 5 incident năm 2026 không trực tiếp đáp ứng yêu cầu tìm kiếm dữ liệu năm 2025.
  + Agent chưa nhấn mạnh đủ rõ rằng dữ liệu hiện tại không thể chứng minh danh sách năm 2025 đầy đủ hay xác nhận không có incident.

## 5. Giới hạn
- Giới hạn của điểm Judge: Điểm Judge được đánh giá bởi một mô hình AI khác nên có thể chịu ảnh hưởng từ cách diễn giải tiêu chí, prompt và tính chủ quan của mô hình. Điểm số cao hơn không đồng nghĩa với câu trả lời luôn chính xác hơn trong mọi tình huống.
- Giới hạn của Snapshot: Kết quả chỉ phản ánh trạng thái GitHub và Cloudflare tại thời điểm snapshot được tạo. Nếu trạng thái dịch vụ thay đổi sau đó, kết quả đánh giá có thể không còn phản ánh tình hình thực tế. Vì vậy, không thể sử dụng snapshot để khẳng định trạng thái hiện tại của hệ thống.
- Giới hạn khi chỉ chạy một lần mỗi phiên bản: Mỗi phiên bản chỉ được chạy một lần nên kết quả có thể bị ảnh hưởng bởi tính không ổn định của LLM, chẳng hạn như sự khác biệt trong cách reasoning, lựa chọn tool hoặc cách sử dụng token. Do đó, chưa đủ dữ liệu để kết luận rằng sự thay đổi giữa before và after hoàn toàn đến từ những cải tiến trong implementation.
