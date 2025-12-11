# Principal-Level-Patterns

## 1) Kiến trúc tổng thể cho RetailStoreManagement
- Tách MediaService (backend .NET) chuyên ký số và upload; frontend chỉ gọi để lấy signature/preset.
- DB lưu metadata: `PublicId`, `SecureUrl`, `Format`, `Width`, `Height`, `Bytes`, `Folder`, `Version`, `CreatedAt`.
- Mapping ERD: mở rộng `Products` (thêm metadata ảnh chính); tùy chọn thêm bảng `ProductImages` cho đa ảnh; `Promotions`/`Categories` có thể dùng 1 ảnh cover.
- Không lưu binary trong DB; chỉ metadata và khóa tham chiếu.

## 2) Quy trình upload chuẩn
1. Frontend yêu cầu signature/preset.
2. Frontend upload trực tiếp Cloudinary (widget hoặc form).
3. Webhook → backend cập nhật DB (status, metadata).
4. UI hiển thị từ `SecureUrl` với transform CDN.

## 3) Delivery chiến lược
- Chuẩn hóa preset transform: `thumb` (w=200,q_auto:eco), `card` (w=600,c_fill,q_auto), `detail` (w=1200,q_auto,f_auto).
- Dùng HTTPS mặc định; bật HTTP/2/3 CDN.
- Giảm biến thể URL để kiểm soát cache chi phí.

## 4) Bảo mật & phân quyền
- Signature chỉ cấp cho user đã auth (Admin/Staff).
- Unsigned preset chỉ dùng môi trường nội bộ/demo; khóa chặt `allowed_formats`, `max_file_size`, `folder`.
- Với media nhạy cảm, dùng `type=authenticated` + signed URL khi phát.

## 5) Chi phí & quota
- Giới hạn kích thước upload; buộc nén client (canvas/HTML5) trước khi gửi nếu cần.
- Theo dõi số biến thể transform; dọn ảnh mồ côi (không còn tham chiếu DB).
- Log & cảnh báo khi số byte/tháng hoặc số transform vượt ngưỡng.

## 6) Observability
- Ghi `requestId`/`assetId` vào log để correlation.
- Dashboard: tỉ lệ lỗi upload, thời gian upload, phân bố kích thước, top biến thể.
- Alert khi webhook fail, khi invalidation tăng đột biến.

## 7) Migration/rollback
- Khi thay đổi preset hoặc folder, hỗ trợ chạy script re-upload/repoint metadata.
- Lưu version metadata; rollback bằng publicId/version cũ.

## 8) Integration với frontend
- Dùng custom loader (Next/React) hoặc component Cloudinary để tạo URL transform nhất quán.
- Prefetch ảnh quan trọng (hero) và lazy load phần còn lại; LQIP cho danh sách.


