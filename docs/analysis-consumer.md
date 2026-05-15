# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: Pair 03 (Repo: B-pair03-core-access)
- Product: Product B
- Provider service: Access Gate (B3)
- Consumer service: Core Business (B6)
- Người viết: Nguyễn Đức Mạnh
- Ngày: 16/05/2026

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `Gate` | Thông tin các cổng kiểm soát ra vào trong campus. | `id`, `name`, `location`, `status` (online, offline, maintenance) | `ipAddress`, `lastMaintenanceDate` |
| `AccessLog` | Lịch sử quẹt thẻ/khuôn mặt đi qua cổng. | `id`, `gateId`, `userId`, `timestamp`, `accessResult` (granted, denied) | `failureReason` |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/v1/gates` | Lấy danh sách các cổng kiểm soát. | Khi Dashboard trung tâm cần hiển thị bản đồ cổng. |
| POST | `/v1/gates/{id}/open` | Điều khiển mở cổng từ xa. | Khi bảo vệ/admin bấm nút mở cổng khẩn cấp trên app trung tâm. |
| GET | `/v1/access-logs` | Truy xuất lịch sử ra vào có phân trang. | Khi admin muốn xem lại báo cáo điểm danh hoặc tra cứu an ninh. |

---

## 3. Error case

Tối thiểu 5 case.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload thiếu tham số hoặc sai định dạng khi tìm kiếm. | `Problem Details` (kèm mảng thông báo lỗi chi tiết) |
| 401 | Yêu cầu không có Bearer token hoặc token hết hạn. | `Problem Details` |
| 403 | Token hợp lệ nhưng user không có quyền mở cổng này. | `Problem Details` |
| 404 | Không tìm thấy `gateId` truyền vào. | `Problem Details` |
| 409 | Xung đột: Cổng đang bảo trì hoặc mất kết nối mạng nhưng bị gọi lệnh mở. | `Problem Details` (kèm mã lỗi nghiệp vụ `GATE_OFFLINE`) |
| 422 | Dữ liệu đúng JSON nhưng thời gian từ ngày > đến ngày (khi lọc log). | `Problem Details` |

---

## 4. Giả định bổ sung

- Giả định 1: Service Core Business đã xác thực người dùng và gán quyền đúng đắn trước khi cấp Token gọi sang Access Gate.
- Giả định 2: Access Gate chỉ lưu `userId` chứ không lưu thông tin chi tiết (tên, khoa, lớp) của sinh viên. Nếu Consumer muốn hiển thị tên, Consumer phải tự join dữ liệu với Identity Service.
- Giả định 3: Số lượng Access Log rất lớn, do đó bắt buộc phải dùng Pagination (phân trang) cho endpoint lấy log.

---

## 5. Câu hỏi cho Consumer

1. Với API lấy danh sách `AccessLog`, các bạn muốn phân trang theo kiểu Offset-based (page, size) hay Cursor-based (như yêu cầu BTVN của thầy)?
2. Khi gọi API `POST /v1/gates/{id}/open`, các bạn có cần Access Gate trả về kết quả ngay lập tức không, hay chỉ cần nhận HTTP 202 Accepted (vì phần cứng cổng có thể phản hồi chậm)?
3. Các bạn có cần gửi kèm `reason` (lý do mở cổng) vào request body khi gọi lệnh mở khẩn cấp không?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field trạng thái không thống nhất | Consumer parse lỗi logic | Chốt Enum cho `status` (ONLINE, OFFLINE) trong `openapi.yaml` |
| Phản hồi thiết bị IoT chậm | Consumer bị timeout | Thống nhất timeout chung là 5s, hoặc chuyển lệnh điều khiển sang bất đồng bộ nếu cần. |