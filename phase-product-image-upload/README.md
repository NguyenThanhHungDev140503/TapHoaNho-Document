## Phase: Product Image Upload với ImageKit

- Mục tiêu: Thêm chức năng upload ảnh sản phẩm qua ImageKit, lưu `image_file_id` trong database, và hiển thị ảnh ở form tạo sản phẩm + màn chi tiết sản phẩm.
- Phạm vi:
  - Frontend: Thêm nút upload ảnh, preview ảnh trong form create product và modal chi tiết.
  - Backend: Thêm trường `image_file_id` cho product, endpoint auth upload ImageKit, cập nhật DTO/Service.
  - Database: Migration bảng `Products` để map được `image_file_id`.















