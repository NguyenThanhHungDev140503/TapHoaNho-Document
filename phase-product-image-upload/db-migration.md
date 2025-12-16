## DB Migration - Products.image_file_id

### Hiện trạng

- Bảng `Products` hiện có các cột:
  - `Id`, `category_id`, `supplier_id`, `product_name`, `barcode`, `price`, `unit`, `CreatedAt`, `UpdatedAt`, `DeletedAt`.

### Thay đổi đề xuất

- Thêm cột mới:
  - `image_file_id` (varchar / text, nullable)
  - Dùng để lưu `fileId` do ImageKit trả về sau khi upload ảnh sản phẩm.

### Bước thực hiện

1. Tạo migration mới trong dự án `RetailStoreManagement` (EF Core) để thêm cột `image_file_id` vào bảng `Products`.
2. Apply migration lên database Neon (project `TapHoaNho`).
3. Cập nhật entity + DTO + AutoMapper để đọc/ghi `ImageFileId`.


