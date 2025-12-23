## Implementation Plan - Product Image Upload

1. Backend
   - Thêm field `ImageFileId` vào `ProductEntity` và các DTO liên quan.
   - Cập nhật AutoMapper profile cho Product.
   - Tạo migration thêm cột `image_file_id` vào bảng `Products` và apply lên Neon.
   - Tạo endpoint auth upload ImageKit (trả về signature, token, expire, publicKey).
2. Frontend
   - Cập nhật `ProductEntity` / `ProductDetailsDto` type để có `imageFileId`.
   - Cập nhật API service để gửi/nhận `imageFileId`.
   - Thêm nút upload ảnh trong `ProductForm`:
     - Gọi BE lấy auth params.
     - Upload file lên ImageKit.
     - Lưu `imageFileId` vào form.
     - Hiển thị preview ảnh.
   - Thêm preview ảnh trong `ProductDetailModal` dựa trên `imageFileId`.
3. Kiểm thử
   - Tạo mới sản phẩm có/không có ảnh.
   - Xem chi tiết sản phẩm và kiểm tra ảnh hiển thị đúng.
   - Kiểm tra các case lỗi upload (mạng, file không hợp lệ, auth fail).















