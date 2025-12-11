# Advanced-Patterns

## 1) Signature service (backend .NET)
- Tạo endpoint `POST /media/signature` nhận `{ folder, overwrite?, eager? }`, trả `{ timestamp, signature, apiKey, cloudName, folder }`.
- Ký bằng `Api.SignParameters` (CloudinaryDotNet). Không log `api_secret`.
- Thêm rate limiting + auth (Admin/Staff).

## 2) Upload widget + preset
- Dùng upload widget cho trang Staff nội bộ; buộc `upload_preset` đã khóa: `folder=products`, `allowed_formats=jpg,png,webp`, giới hạn `max_file_size`, `unique_filename=true`, `overwrite=false`.
- Ưu tiên preset unsigned để giảm round-trip, nhưng chỉ cho nội bộ; với public upload → dùng signed.

## 3) Delivery tối ưu
- URL pattern: `.../image/upload/f_auto,q_auto,c_fill,w_{w},h_{h}/<publicId>`.
- Cho danh sách (thumbnail) dùng `q_auto:eco` hoặc `q_auto:low`, `w` nhỏ.
- Lazy load + LQIP: tạo biến thể `e_blur:2000,q_1,w_20` làm placeholder.

## 4) Quản lý version & overwrite
- Tắt overwrite mặc định; nếu cần cập nhật ảnh, tạo publicId mới (hoặc dùng `overwrite=true` + `invalidate=true` kèm webhook để đồng bộ cache).
- Lưu `assetId` và `version` vào DB để truy vết.

## 5) Bulk & large file
- Với file lớn: dùng `chunk_size` (SDK hỗ trợ) và `async=true` (video lớn).
- Hàng đợi: đưa upload vào background job (Hangfire/Quartz) với callback webhook cập nhật trạng thái.

## 6) Bảo mật
- Không cho client truyền tùy ý transform; chỉ cho phép preset transform đã kiểm soát.
- Ký URL nhạy cảm bằng URL signing khi cần private/authenticated delivery.
- Chặn MIME trái phép ở cả client lẫn server; dùng `resource_type` phù hợp (image/raw/video/auto).

## 7) Logging/Observability
- Log `publicId`, `bytes`, `width/height`, `requestId` vào ELK/OTel.
- Cảnh báo theo ngưỡng lỗi upload và dung lượng tăng bất thường.

## 8) Testing nâng cao
- Unit: hàm ký số đúng đầu vào (timestamp, folder).
- Integration: upload thật vào môi trường sandbox với preset khóa, assert trả về `secure_url`, `width/height`.
- E2E: frontend → backend ký số → Cloudinary; kiểm tra DB lưu `publicId`.


