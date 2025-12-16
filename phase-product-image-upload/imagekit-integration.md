## ImageKit Integration Notes

### Auth upload flow

- Backend:
  - Giữ `privateKey`, `publicKey`, `urlEndpoint` của ImageKit trong config/env.
  - Endpoint `/api/admin/imagekit/auth`:
    - Sinh `token` random.
    - Tính `expire` (timestamp).
    - Tạo `signature = HMAC-SHA1(token + expire, privateKey)`.
    - Trả về `{ token, expire, signature, publicKey }`.

- Frontend:
  - Nhận auth params từ BE.
  - Gọi SDK JS ImageKit hoặc trực tiếp `POST https://upload.imagekit.io/v1/files/upload` với:
    - `file` (binary / base64),
    - `fileName`,
    - `signature`, `token`, `expire`, `publicKey`.
  - Đọc `fileId` từ response và lưu vào `imageFileId` của product.

### URL hiển thị ảnh

- FE build URL từ `urlEndpoint` + `imageFileId` theo pattern ImageKit, hoặc dùng helper SDK/URL builder nếu cần transform/resize.


