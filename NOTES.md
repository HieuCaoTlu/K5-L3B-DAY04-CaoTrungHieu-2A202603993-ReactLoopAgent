# Ghi chú phân tích · Day 04

Họ và tên: Cao Trung Hieu
MSSV: 2A202603993

Cấu hình eval (giống nhau ở cả hai lần): model `gemini-3.5-flash`, judge `gemini-3.5-flash`, max_steps 8, 22 case.

## 1. Case được chọn

- ID case: E4
- Câu hỏi: Xem incident GitHub xr3myzj7ms51: những component được ghi nhận nào bị ảnh hưởng? Actions có liên quan không?
- Kỳ vọng: Liệt kê Webhooks, Actions, Pages; Actions có liên quan; status resolved.
- Câu trả lời của agent: đúng nội dung, nói rõ Actions bị ảnh hưởng nhưng không nêu link nguồn. Nguyên văn (trường `answer` trong `eval_before.json`):

```text
Dựa trên thông tin truy xuất từ GitHub Status đối với sự cố (Incident ID: **xr3myzj7ms51**):

### 1. Trạng thái sự cố
* **Tên sự cố:** Incident with Actions and Pages
* **Trạng thái:** `resolved` (Đã được xử lý / giải quyết)
* **Thời gian tạo:** 04/11/2026 16:05:31 UTC

---

### 2. Các component được ghi nhận bị ảnh hưởng
* **Theo danh mục component chính thức của incident:**
  * **Actions**
  * **Pages**
  * **Webhooks**

* **Theo nội dung chi tiết trong bản cập nhật (incident updates):**
  * Sự cố làm chậm trễ các dịch vụ: **GitHub Actions**, **Pages**, **Webhooks**, **Code Scanning**, và **Copilot coding agent**.

---

### 3. GitHub Actions có liên quan không?
* **Có, GitHub Actions bị ảnh hưởng trực tiếp** và nằm trong danh sách component chính được ghi nhận.
* **Chi tiết ảnh hưởng được ghi nhận:** Các lượt chạy workflow (*workflow runs*) và cập nhật trạng thái job (*job status updates*) của GitHub Actions đã bị chậm trễ tối đa lên đến 25 phút trong khoảng thời gian diễn ra sự cố.
```

- Điểm ở lần chạy 1: 100/100. 
- Điểm tổng lần 1: 99.95/100, 22/22 case được chấm.
- Nhận xét: điểm cao nhưng judge dễ dãi, có 7 case thiếu link nguồn (E1, E2, E3, E4, A5, A7, H5).

## 2. Bằng chứng trong trace

Trace của case E4:

```json
[
  {
    "event": "model",
    "step": 1,
    "finish": "tool_calls"
  },
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
      "provider": "github",
      "captured_at": "2026-11-09T03:50:47.065020+00:00",
      "id": "xr3myzj7ms51",
      "name": "Incident with Actions and Pages",
      "status": "resolved",
      "created_at": "2026-11-04T16:05:31.580Z",
      "updated_at": "2026-11-07T01:04:38.647Z",
      "components": [
        "Webhooks",
        "Actions",
        "Pages"
      ],
      "updates": [
        {
          "status": "resolved",
          "created_at": "2026-11-04T17:01:11.503Z",
          "body": "On November 4, 2026, between 15:49 UTC and 16:48 UTC, and again between 17:20 UTC and 17:55 UTC, GitHub Actions, Pages, Webhooks, Code Scanning, and Copilot coding agent experienced delays. During this time, Actions workflow runs and job status updates were delayed by up to 25 minutes. Pages deployments dropped to about 4% of normal volume, webhook deliveries dropped to about 58% of normal volume, and Code Scanning analyses and Copilot coding agent sessions were delayed. Delayed events were processed after mitigation. <br /><br />This incident was caused by a configuration change to an internal event-routing service that unexpectedly paused delivery of several event streams. The change depended on feature flags that were not set up as intended, which the service did not handle gracefully. We mitigated the first period by rolling back the configuration change at 16:14 UTC. The code change",
          "truncated": true
        },
        {
          "status": "monitoring",
          "created_at": "2026-11-04T16:49:27.384Z",
          "body": "The degradation has been mitigated. We are monitoring to ensure stability.",
          "truncated": false
        },
        {
          "status": "investigating",
          "created_at": "2026-11-04T16:05:31.696Z",
          "body": "We are investigating reports of degraded performance for Actions and Pages",
          "truncated": false
        }
      ],
      "updates_total": 9,
      "url": "https://stspg.io/pjgptg3cfc6n"
    }
  },
  {
    "event": "model",
    "step": 2,
    "finish": "stop"
  }
]
```

Đối chiếu: tool trả `url = https://stspg.io/pjgptg3cfc6n` ở step 1 (trong trace), nhưng câu trả lời ở trên (trường `answer`, trace không chứa phần này) chỉ nêu mã incident `xr3myzj7ms51`, không có URL này.

Nguyên nhân có thể:
- Judge không nhắc tới việc thiếu link, vì judge chỉ chấm nội dung.
- Model nhớ mã incident nhưng không chép URL, có thể do system prompt cũ chỉ nói "Dẫn nguồn từ tool" chung chung.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (trong `agent.py`):
  + Thêm kiểm tra URL nguồn: lưu URL của mọi lần `get_incident_details` thành công vào tập hợp `observed_urls`. Khi model định trả lời cuối mà answer không chứa URL nào trong đó thì ghi sự kiện `validation_missing_source` vào trace và nhắc model viết lại kèm URL. Hết lượt mà vẫn thiếu thì trả `max_steps`, không báo `completed`.
  + Sửa lại system prompt: đổi sang tiếng Anh, viết theo cấu trúc, có quy tắc dẫn URL của từng incident. Prompt mới tên `SYSTEM_PROMPT_ADVANCED`.
- Vì sao chọn thay đổi này: nguyên nhân có thể của lỗi là model không chép URL từ kết quả tool và prompt cũ quá chung chung. Kiểm tra URL trong code là lưới bảo vệ nếu model vẫn quên, còn prompt mới nói rõ phải dẫn URL của từng incident.

Kiểm tra: 37/37 test đạt (gồm hai test `test_validation`). Khi chạy lại câu hỏi của E4 với agent mới, câu trả lời đã có link.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 99.95 | 99.86 |
| Điểm case đã chọn (E4) | 100 | 100 |
| Số case thiếu link nguồn | 7 | 0 |
| Số model calls | 58 | 67 |
| Số tool calls | 48 | 66 |
| Token | 90,902 | 126,142 |

- Lỗi đã giải quyết: cả 7 case thiếu link nguồn nay đều có link, trong đó E4 (tool trả URL, câu trả lời lần 2 có URL).
- Lỗi còn lại hoặc mới xuất hiện: không có lỗi mới, không case nào bị `error` hay `max_steps`. Điểm tổng giảm nhẹ (99.95 xuống 99.86) do case H1 (99 xuống 97), judge chỉ trừ nhẹ ở phần completeness. Còn lại: H1 chưa nêu rõ dữ liệu incident bị cắt (`truncated`).
- Chi phí tăng khoảng 39% token (90,902 lên 126,142) vì agent gọi tool nhiều hơn (tool calls 48 lên 66, riêng `get_incident_details` nhiều hơn để lấy URL cho từng incident).
- Số lần phải nhắc model thêm link ở lần 2 là 0: bước kiểm tra URL trong code chưa được kích hoạt ở case nào. Việc hết thiếu link nhờ system prompt mới.

## 5. Giới hạn
- Judge dùng cùng model với agent (mặc định của lab) nên có thể chấm dễ dãi, số case có link nguồn và trace đáng tin hơn.
- Chấp nhận đánh đổi thêm chi phí nếu muốn model chính xác trong việc dẫn nguồn khi trả lời

## 6. Phần Bonus
- Kết quả chạy: 
+ `eval_bonus.json`, điểm 100/100
+ 10/10 case được chấm, bonus 15/15
+ Cấu hình model: model `gemini-3.5-flash`, judge `gemini-3.5-flash`, max_steps 16

- Thay đổi:
+ Dùng tools.schemas và tools.system_prompt thay vì gắn cứng prompt và schemas
+ Thêm biến rate_limited, khi trường hợp call tool mà bị báo lỗi RATE_LIMITED, hệ thống lưu lại RATE_LIMITED và tăng bộ đếm thêm 1. Nếu vòng lặp gọi tools quá 2 lần (MAX_RATE_LIMIT_RETRIES hiện đang đặt là 2), hệ thống sẽ báo lỗi RETRY_LIMIT_REACHED và bảo model dừng thử lại và nêu rõ phần chưa kiểm tra được

- Bằng chứng theo yêu cầu

| Yêu cầu | Case | Trace chứng minh |
|---|---|---|
| Retry 429 thành công | B3 | `get_vercel_status` trả `RATE_LIMITED` ở lần gọi 1; model gọi lại, lần 2 thành công; kết quả `completed`, 100 điểm |
| Dừng retry khi 429 kéo dài | B4 | `get_github_status` thành công; `get_vercel_status` trả `RATE_LIMITED` hai lần liên tiếp, lần gọi thứ 3 bị agent chặn với `RETRY_LIMIT_REACHED` (công cụ không bị gọi lần 3). Câu trả lời nêu rõ Vercel chưa kiểm tra được vì bị 429 liên tục, vẫn trả kết quả của GitHub |
| Lỗi định dạng | B5 | Lần gọi đầu `get_incident_details` thiếu `offset` và `limit` nên nhận `INVALID_ARGUMENTS` (lỗi định dạng giả lập). Model gọi lại với số nguyên `offset=0, limit=2`, rồi đọc tiếp `offset=2` |
| Pagination | B2 | `get_incident_details` của Vercel được gọi 3 lần với `offset` 0, 3, 6; `next_offset` lần lượt là 3, 6, `None` (tổng 7 update), tức đọc hết timeline rồi mới trả lời |
| Dữ liệu nhiễu (context) | B6 | `get_vercel_status` trả kèm 160 mục nhiễu giả lập; câu trả lời không nhắc mục nhiễu nào, chỉ đọc chi tiết hai incident liên quan (Vercel và npm), mỗi incident đọc 2 trang (`offset` 0 rồi 2, `next_offset` về `None`) |

- Đã chạy thử một lần với mode live, thành công và chưa gặp 429. 

- Giới hạn còn lại:
+ Chưa thử chạy live hoàn toàn
+ Chưa thực nghiệm với cơ chế chờ của Retry-After