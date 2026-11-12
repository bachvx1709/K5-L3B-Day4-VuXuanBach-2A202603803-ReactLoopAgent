# Ghi chú phân tích · Day 04

Họ và tên: Vũ Xuân Bách
MSSV: 2A202603803

## 1. Case được chọn

- ID case: H2
- Câu trả lời của agent:
Dựa trên thông tin từ snapshot: Incident ID: t89g2q1800fr. Tên sự cố: "Incident with Actions". Trạng thái sự cố: Đã được giải quyết (resolved). Thời gian xảy ra: Bắt đầu vào ngày 5 tháng 11 năm 2026 lúc 16:50 UTC. Thời gian khắc phục: Kết thúc trạng thái giám sát lúc 17:42 UTC, ngày 5 tháng 11 năm 2026. Liên quan đến: Thành phần GitHub Actions. Nguồn chính thức: Chi tiết sự cố trên GitHub Status. Như vậy, sự cố đã được giải quyết và hiện tại không có tình trạng outage liên quan đến GitHub Actions. Khẳng định bạn đưa ra về việc GitHub Actions đang bị outage không còn đúng thời điểm hiện tại, dựa trên snapshot được cung cấp.

- Điểm ở lần chạy 1: 50

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
[
          {
            "event": "tool",
            "step": 1,
            "name": "get_incident_details",
            "arguments": {
              "incident_id": "t89g2q1800fr",
              "provider": "github"
            },
            "result": {
              "ok": true,
              "provider": "github",
              "captured_at": "2026-11-09T03:50:47.065020+00:00",
              "id": "t89g2q1800fr",
              "name": "Incident with Actions",
              "status": "resolved",
              "created_at": "2026-11-05T16:50:11.086Z",
              "updated_at": "2026-11-05T17:42:35.455Z",
              "components": [
                "Actions"
              ],
              "updates": [
                {
                  "status": "resolved",
                  "created_at": "2026-11-05T17:42:35.425Z",
                  "body": "This incident has been resolved. Thank you for your patience and understanding as we addressed this issue. A detailed root cause analysis will be shared as soon as it is available.",
                  "truncated": false
                },
                {
                  "status": "monitoring",
                  "created_at": "2026-11-05T17:04:46.807Z",
                  "body": "The degradation affecting Actions has been mitigated. We are monitoring to ensure stability.",
                  "truncated": false
                },
                {
                  "status": "investigating",
                  "created_at": "2026-11-05T16:50:11.189Z",
                  "body": "We are investigating reports of degraded performance for Actions",
                  "truncated": false
                }
              ],
              "updates_total": 4,
              "url": "https://stspg.io/lv4q3xm9r0rk"
            }
          }
        ],

```
- Lỗi: chưa gọi tool cần thiết(get_status)
- Nguyên nhân: Agent bị thiên kiến, chỉ tra cứu đúng incident_id có trong prompt input, bỏ quên yêu cầu kiểm tra trạng thái hiện tại.
- Đề xuất cải tiến: Tinh chỉnh promt yêu cầu agent luôn kiểm tra trạng thái hiện tại của tất cả các service liên quan, không chỉ tập trung vào incident_id được cung cấp.
   


## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Cập nhật `SYSTEM_PROMPT` trong file `agent.py`:
  + Bổ sung chỉ dẫn: Khi điều tra về trạng thái dịch vụ (outage, sự cố), luôn phải gọi `get_status` để kiểm tra trạng thái snapshot hiện tại; nếu có `incident_id` cụ thể thì tra cứu thêm `get_incident` để đối chiếu,không bỏ qua `get_status`.
- Vì sao chọn thay đổi này:
  + Agent chỉ gọi `get_incident` vì bị cuốn theo mã sự cố trong prompt của người dùng, dẫn đến vi phạm `Chưa gọi tool cần thiết: get_status`.
  + Cải tiến system prompt can thiệp vào hành vi ra quyết định của LLM mà không phá vỡ logic vòng lặp cơ bản của agent loop.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 89.45 | 91.95 |
| Điểm case đã chọn | 50 | 100 |
| Số model calls | 47 | 48 |
| Token, chi phí nếu có | 34 225 | 38 054 |

- Lỗi đã giải quyết: Lỗi ở H2 đã biến mất, trace cho thấy agent đã gọi cả get_status lẫn get_incident
- Lỗi còn lại hoặc mới xuất hiện: H7 - agent bị thực hiện theo prompt kết luận của user, k trả về đúng tình trạng thực tế và H1,H6 - agent gộp nhiều tool gọi thành 1 lượt.

## 5. Giới hạn

- Dùng chính model làm agent để đáng giá dẫn tới điểm không phản ảnh đúng, có thể cao hơn thực tể
- Chỉ chạy một lần thiếu tổng quát hoá, khi chạy lại có thể sẽ ra kết quả khác cũng như điểm số khác
- Giới hạn của snapshot là dữ liệu tĩnh, không phản ánh tình trạng thực tế, không kiểm tra được khả năng agent xử lý sự cố real-time.
