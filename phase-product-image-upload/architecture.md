## Kiến trúc tổng quan Product Image Upload

### Luồng tổng thể

- Frontend:
  - Modal tạo sản phẩm có nút **Upload Image** để chọn file ảnh từ máy.
  - FE gọi API backend để lấy auth params (signature, token, expire, publicKey) cho ImageKit.
  - FE upload trực tiếp file lên ImageKit, nhận về `fileId` (image_file_id).
  - FE lưu `imageFileId` vào form tạo sản phẩm và hiển thị preview ảnh.
- Backend:
  - Cung cấp endpoint auth upload ImageKit (không lộ private key).
  - Lưu `image_file_id` trong bảng `Products` thông qua entity + DTO.
- Database:
  - Bảng `Products` thêm cột `image_file_id` (nullable) để map ảnh với product.


