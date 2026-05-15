# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: B-pair03-core-access
- Product: Product B
- Provider: Access Gate (Nhóm B3)
- Consumer: Core Business (Nhóm B6)
- Phiên: v1.0
- Ngày: 16/05/2026

---

## Issue #1

- Raised by: Consumer (B6)
- Endpoint: Toàn bộ API (Các schema `Gate`, `AccessLog`)
- Concern: Provider dự định lưu ID thiết bị là số nguyên tự tăng (Integer), nhưng Consumer dùng UUID cho toàn bộ hệ thống định danh phân tán.
- Proposal: Thống nhất dùng chuỗi UUID v4 cho mọi trường định danh như `gateId`, `userId`, `id`.
- Resolution: Accepted
- Rationale: Việc sử dụng UUID giúp tránh lộ số lượng thiết bị thực tế của hệ thống, đồng thời dễ dàng đồng bộ và ghép nối dữ liệu từ nhiều node IoT mà không sợ trùng lặp khóa chính.
- Impact: Provider phải đổi kiểu dữ liệu khóa chính trong Database sang UUID. Thêm ràng buộc định dạng `format: uuid` vào `openapi.yaml`.

---

## Issue #2

- Raised by: Provider (B3)
- Endpoint: `GET /v1/gates`
- Concern: Thuộc tính `lastMaintenanceDate` (Ngày bảo trì gần nhất). Đối với các cổng mới lắp, chưa bảo trì lần nào thì nên ẩn field này đi trong response hay trả về giá trị rỗng/null?
- Proposal: Sử dụng kỹ thuật Union type với `null` (hỗ trợ bởi OpenAPI 3.1).
- Resolution: Accepted
- Rationale: Việc luôn trả về trường dữ liệu nhưng cho phép mang giá trị `null` giúp phía Consumer (Frontend/Core) dễ dàng map vào Entity/Model cứng, thay vì phải kiểm tra xem trường đó có tồn tại trong Object hay không.
- Impact: Định nghĩa trong Schema thuộc tính `lastMaintenanceDate` có `type: ["string", "null"]`.

---

## Issue #3

- Raised by: Provider (B3)
- Endpoint: `GET /v1/gates`
- Concern: Cổng kiểm soát ra vào có nhiều loại khác nhau (ví dụ: Cửa xoay ba chạc - Turnstile, Barrier chắn xe). Mỗi loại có các thông số kỹ thuật riêng biệt, không thể gộp chung vào một class duy nhất được.
- Proposal: Sử dụng `oneOf` kết hợp với trường `discriminator` (type) để phân loại.
- Resolution: Accepted
- Rationale: Hỗ trợ tính đa hình (Polymorphism). Consumer có thể sinh ra các class kế thừa (Inheritance) tự động khi generate source code, giúp code sạch và dễ bảo trì hơn.
- Impact: Phải tách Schema thành một interface chung `Gate`, và các schema chi tiết `TurnstileGate`, `BoomBarrierGate`. Dùng trường `type` làm `discriminator`.

---

## Issue #4

- Raised by: Consumer (B6)
- Endpoint: `GET /v1/access-logs`
- Concern: Dữ liệu lịch sử quẹt thẻ sinh ra liên tục mỗi ngày với số lượng cực lớn. Nếu dùng tham số `page` và `size` (Offset pagination) thì khi query ở các trang sâu sẽ làm chậm database đáng kể.
- Proposal: Đổi sang sử dụng chiến lược Cursor-based pagination.
- Resolution: Accepted
- Rationale: Cursor-based pagination (truyền ID/chuỗi của dòng cuối cùng) có khả năng tận dụng index Database rất nhanh, không bị chậm khi tải các trang dữ liệu cũ, cực kỳ phù hợp cho dữ liệu dạng Time-series/Logs lớn.
- Impact: Xóa `page`/`size`, thêm tham số `cursor` và `limit` vào query params. Thêm schema bọc ngoài `AccessLogPage`.

---

## Issue #5

- Raised by: Consumer (B6)
- Endpoint: `POST /v1/gates/{gateId}/open`
- Concern: Phía Consumer mong muốn sau khi gọi API điều khiển mở cổng, nếu cổng mở thành công thì API trả về HTTP 200 OK ngay lập tức để báo UI.
- Proposal: Provider đề xuất trả về mã `202 Accepted` thay vì `200 OK`.
- Resolution: Modified (Chốt dùng 202 Accepted)
- Rationale: Quá trình gửi lệnh mở cổng cần truyền qua giao thức IoT (như MQTT) xuống phần cứng vật lý, độ trễ rất khó đoán (hoặc có thể kẹt cơ học). REST API của Access Gate chỉ ghi nhận lệnh thành công, không thể treo connection để chờ phần cứng phản hồi rồi mới báo 200 (sẽ dễ gây timeout cho Core Business).
- Impact: Cập nhật HTTP response code trong hợp đồng thành `202 Accepted`.

---

## Issue #6

- Raised by: Provider (B3)
- Endpoint: Toàn bộ API (Các mã lỗi 4xx, 5xx)
- Concern: Nếu API lỗi và Provider trả về JSON lỗi tự thiết kế (ví dụ `{"message": "error"}`) thì Consumer sẽ rất khó có cấu trúc chuẩn để phân tích log nội bộ.
- Proposal: Áp dụng chuẩn công nghiệp RFC 7807 (Problem Details for HTTP APIs).
- Resolution: Accepted
- Rationale: Đây vừa là tiêu chuẩn công nghiệp tốt nhất hiện nay cho việc báo lỗi, vừa là yêu cầu bắt buộc của môn học. Cấu trúc thống nhất giúp Consumer xử lý lỗi đa dịch vụ dễ dàng hơn.
- Impact: Định nghĩa một Schema `Problem` chung và trỏ toàn bộ các response lỗi (400, 401, 403, 404, 409, 422, 500) về Content-Type `application/problem+json`.

---

# Chốt hợp đồng v1.0

Provider sign-off: Nguyễn Đức Mạnh  
Consumer sign-off: Nguyễn Trương Thuận
Witness (GV/TA): Nguyễn Duy Khương 
Date: 16/05/2026

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có | File `openapi.yaml` đã pass 100% ruleset `campus-spectral.yaml` không có lỗi. | N/A |