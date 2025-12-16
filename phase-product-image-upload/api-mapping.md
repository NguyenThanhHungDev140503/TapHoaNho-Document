## API Mapping - Product Image Upload

### Backend APIs

- `POST /api/admin/imagekit/auth`
  - Mục đích: Trả về auth params cho upload ImageKit (`signature`, `token`, `expire`, `publicKey`).
- `POST /api/admin/products`
  - Request body: `CreateProductRequest` có thêm `imageFileId`.
  - Response: `ProductResponseDto` có `imageFileId`.
- `PUT /api/admin/products/{id}`
  - Request body: `UpdateProductRequest` có thể cập nhật `imageFileId`.
- `GET /api/admin/products/{id}`
  - Response: `ProductResponseDto` bao gồm `imageFileId` để FE build URL ảnh.

### Frontend Mapping

- `productApiService.createProduct` / `updateProduct`
  - Gửi thêm `imageFileId` trong payload.
- `productApiService.getProductDetails`
  - Nhận về `imageFileId` và dùng để hiển thị ảnh trong `ProductDetailModal`.


