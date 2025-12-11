# RESEARCH_SUMMARY

## 1. Mục tiêu
- Tích hợp Cloudinary cho lưu trữ ảnh sản phẩm/khuyến mãi trong RetailStoreManagement (.NET 9 backend, React frontend).
- Đảm bảo upload ký số an toàn, tối ưu delivery (CDN, f_auto/q_auto), và lưu metadata vào DB theo ERD mở rộng.

## 2. Nguồn thông tin (đã truy cập 2025-12-09)
- Cloudinary Upload guide — https://cloudinary.com/documentation/upload_images
- Cloudinary .NET SDK — https://cloudinary.com/documentation/dotnet_integration
- Cloudinary Node SDK — https://cloudinary.com/documentation/node_integration
- Image transformations — https://cloudinary.com/documentation/image_transformations
- .NET upload article (upload, SignParameters) — https://cloudinary.com/documentation/dotnet_image_and_video_upload
- Upload widget & unsigned preset — https://cloudinary.com/documentation/upload_widget
- Best practices (caching/transform) — https://dev.to/melvinprince/how-to-optimize-image-caching-in-nextjs-for-blazing-fast-loading-times-3k8l
- API upload reference — https://cloudinary.com/documentation/image_upload_api_reference

## 3. Phát hiện chính
- Signed upload là mặc định an toàn; unsigned phải dùng preset bị khóa (size/type/folder, no overwrite).
- Backend SDK tạo signature tự động; REST cần tự ký.
- Delivery tối ưu: `f_auto,q_auto` + crop/resize CDN giảm chi phí băng thông.
- Webhook hỗ trợ đồng bộ metadata/ trạng thái; cần lưu `publicId` để transform/xóa.
- Quản lý biến thể URL quan trọng để tránh bùng nổ cache/chi phí.

## 4. Kiến trúc/cách dùng đề xuất
- MediaService (.NET) cấp signature/preset, nhận webhook, lưu metadata vào bảng Product (hoặc ProductImages).
- Flow: frontend xin signature → upload trực tiếp Cloudinary → webhook cập nhật DB → UI đọc `secureUrl` với transform chuẩn.
- Preset: `folder=products`, `unique_filename=true`, `overwrite=false`, hạn chế MIME/size.
- Transform chuẩn: thumb/card/detail với `f_auto,q_auto`.

## 5. Use cases đã xác định
- Upload ảnh sản phẩm (Staff/Admin) qua signed upload.
- Upload widget nội bộ với preset khóa.
- Hiển thị list/detail sản phẩm với transform tối ưu.
- Xóa/rollback ảnh bằng `publicId`/`version`.


