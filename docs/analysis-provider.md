# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: Pair 03 (Repo: B-pair03-core-access)
- Product: Product B
- Consumer service: Core Business (B6)
- Provider service: Access Gate (B3)
- Người viết: Nguyễn Trương Thuận
- Ngày: 16/05/2026

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `Gate` | Hiển thị trạng thái các cổng lên Dashboard giám sát của nhà trường. | `id`, `name`, `status` | `location` |
| `AccessLog` | Tổng hợp dữ liệu để điểm danh sinh viên và tính công nhân viên. | `gateId`, `userId`, `timestamp`, `accessResult` | |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| GET | `/v1/gates` | Khi admin mở trang Quản lý thiết bị Access Gate. | Trả về mảng JSON chứa thông tin các cổng hiện có. |
| POST | `/v1/gates/{id}/open` | Khi bảo vệ trực ban ấn nút "Mở khẩn cấp" trên Core Dashboard. | Trả về 204 No Content hoặc 200 OK báo hiệu lệnh đã được gửi thành công. |
| GET | `/v1/access-logs` | Khi hệ thống tự động chạy batch job đồng bộ điểm danh lúc nửa đêm. | Danh sách lịch sử phân trang. |

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Tham số phân trang/lọc ngày bị sai | Báo lỗi nội bộ, dev Core cần sửa lại logic gọi API. |
| 401 | Thiếu token hoặc token hết hạn | Tự động lấy lại token từ Identity Service và gọi lại (Retry). |
| 403 | Admin thao tác không có quyền điều khiển cổng | Hiển thị thông báo "Bạn không có quyền thực hiện thao tác này" lên UI. |
| 404 | Cổng không tồn tại (có thể đã bị gỡ bỏ) | Hiển thị trạng thái "Thiết bị không tồn tại" trên bảng điều khiển. |
| 409 | Cổng đang mất kết nối, không thể mở | Cảnh báo UI: "Không thể kết nối đến cổng vật lý, vui lòng kiểm tra lại". |
| 422 | Payload gửi lên bị vi phạm rule nghiệp vụ | Thông báo người dùng kiểm tra lại thông tin gửi đi. |

---

## 4. Giả định bổ sung

- Giả định 1: Định dạng thời gian (timestamp) mà Access Gate trả về sẽ thống nhất dùng chuẩn ISO 8601 (VD: `2026-05-12T08:00:00Z`).
- Giả định 2: Các API được gọi nội bộ giữa 2 service (Server-to-Server) trong mạng LAN của Smart Campus.

---

## 5. Câu hỏi cho Provider

1. Mảng dữ liệu của `AccessLog` có hỗ trợ truyền query params để lọc theo `userId` (tra cứu nhanh một người) không?
2. Nếu lệnh mở cổng được phát đi, làm sao Core biết được cửa đã *thực sự* mở thành công (vì có thể kẹt cơ học)? Có API nào để check trạng thái cửa đang đóng/mở realtime không?
3. Định dạng `id` của các bạn dùng kiểu UUID hay Integer tự tăng? 

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider đổi kiểu dữ liệu ID | Consumer bị lỗi runtime do parse nhầm | Chốt chặt định dạng kiểu String (UUID) ngay trong `openapi.yaml`. |
| Provider thiếu mã lỗi cụ thể | Consumer khó xử lý logic hiển thị lỗi cho user | Chuẩn hóa chuẩn Problem Details `application/problem+json` như yêu cầu môn học. |