# Chính sách Versioning — Access Gate Service

Tài liệu này quy định cách quản lý phiên bản và lộ trình thay đổi hợp đồng API giữa **Access Gate** (Provider) và **Core Business** (Consumer).

## 1. Nguyên tắc chung

Dịch vụ áp dụng tiêu chuẩn **Semantic Versioning (SemVer) 2.0.0**:
- **MAJOR (X.0.0)**: Khi có thay đổi không tương thích (Breaking Changes).
- **MINOR (0.X.0)**: Khi thêm tính năng mới nhưng vẫn tương thích ngược (Backward-compatible).
- **PATCH (0.0.X)**: Khi sửa lỗi nhỏ (Bug fixes) không làm thay đổi cấu trúc API.

## 2. Thay đổi tương thích ngược (Backward-compatible)

Provider có thể thực hiện các thay đổi sau mà không cần nâng phiên bản MAJOR:
- **Thêm Endpoint mới**: Ví dụ thêm `GET /gates/statistics`.
- **Thêm trường tùy chọn (Optional) vào Response**: Ví dụ thêm trường `batteryLevel` vào schema `GateStatus`.
- **Thêm giá trị vào Enum**: Ví dụ thêm trạng thái `LOCKED` vào `CardStatus` (Consumer cần có logic xử lý mặc định cho giá trị lạ).
- **Thêm tham số Query tùy chọn**: Ví dụ thêm `sortBy` vào `/access/logs/recent`.

## 3. Thay đổi gây lỗi (Breaking Changes)

Các thay đổi sau bắt buộc phải nâng phiên bản MAJOR và thông báo trước cho Consumer:
- **Xóa hoặc đổi tên Endpoint**: Ví dụ đổi `/cards/{cardId}` thành `/identity/cards/{cardId}`.
- **Xóa hoặc đổi tên trường trong Response**: Ví dụ đổi `holderName` thành `ownerName`.
- **Thay đổi kiểu dữ liệu**: Ví dụ đổi `gateId` từ `string` sang `integer`.
- **Thêm ràng buộc mới cho Request**: Ví dụ biến một trường `optional` thành `required`.
- **Thay đổi logic mã lỗi**: Ví dụ thay đổi mã lỗi từ `404 Not Found` sang `410 Gone`.

## 4. Quy trình Deprecation

Khi một tính năng cũ sắp được thay thế, Provider sẽ:
1. **Đánh dấu trong OpenAPI**: Sử dụng thuộc tính `deprecated: true` cho operation hoặc schema đó.
   - *Ví dụ:* Đánh dấu endpoint `/access/logs/{logId}` là deprecated nếu chuyển sang hệ thống log mới.
2. **Thông báo qua Header**: Gửi kèm Header `Deprecation: <date>` trong các phản hồi API để cảnh báo hệ thống của Consumer.

## 5. Quy trình gỡ bỏ (Sunset)

Provider cam kết duy trì phiên bản cũ ít nhất **3 tháng** kể từ ngày thông báo Deprecation.
- **Sunset Header**: Provider sẽ trả về Header `Sunset: <date>` để chỉ định thời điểm chính xác API cũ sẽ ngừng hoạt động hoàn toàn.
- Sau ngày Sunset, API cũ sẽ trả về lỗi `410 Gone` hoặc `301 Moved Permanently`.

---
*Cập nhật lần cuối: 2026-05-13*
*Đội ngũ phát triển: Access Gate Team*