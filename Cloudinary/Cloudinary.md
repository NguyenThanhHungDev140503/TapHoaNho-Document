# Cloudinary.md

## Mục lục
- Giới thiệu tổng quan
- Junior: Cài đặt & upload cơ bản
- Middle: Biến đổi, delivery, caching
- Senior: Bảo mật, webhook, lỗi thường gặp
- Tài liệu tham khảo nhanh

## Giới thiệu tổng quan
Cloudinary là dịch vụ lưu trữ/phân phối media với CDN, hỗ trợ upload ký số, unsigned preset, transform động (auto format/quality), và SDK đa ngôn ngữ (.NET, Node). Với RetailStoreManagement, Cloudinary giữ file gốc, DB lưu metadata (publicId, secureUrl, format, width/height, size).

## Junior — Cơ bản
### Cài đặt & cấu hình
- .NET: `dotnet add package CloudinaryDotNet`
- Node: `npm install cloudinary`
- Cấu hình (server only):
```csharp
var account = new Account(cloudName, apiKey, apiSecret);
var cloudinary = new Cloudinary(account) { Api = { Secure = true } };
```

### Signed upload (khuyến nghị)
```csharp
// Tạo signature (server)
var timestamp = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
var parameters = new SortedDictionary<string, object> {
  { "timestamp", timestamp },
  { "folder", "products" }
};
var signature = cloudinary.Api.SignParameters(parameters);
// Trả về cho client: { timestamp, signature, apiKey, cloudName, folder }
```
Client gửi file đến `/v1_1/<cloudName>/image/upload` kèm `file`, `timestamp`, `signature`, `api_key`, `folder`.

### Unsigned upload + upload preset
- Tạo upload preset (console) với giới hạn mime/size, folder cố định, không overwrite.
- Client dùng `upload_preset` thay cho signature khi chấp nhận rủi ro thấp (demo/internal).

### Hello World (.NET server upload)
```csharp
var uploadParams = new ImageUploadParams {
  File = new FileDescription(localPath),
  Folder = "products",
  UseFilename = true,
  UniqueFilename = true
};
var result = await cloudinary.UploadAsync(uploadParams);
// Lưu result.PublicId, result.SecureUrl, result.Format, result.Width, result.Height
```

### Common pitfalls (Junior)
- ❌ Expose `api_secret` cho frontend. ✅ Chỉ ký số tại backend.
- ❌ Không giới hạn loại file/size. ✅ Dùng upload preset hoặc validate server-side.
- ❌ Không lưu `publicId`. ✅ Cần để xóa/transform.

## Middle — Transform/Delivery/Caching
### Transformation URL
`https://res.cloudinary.com/<cloudName>/image/upload/c_fill,w_800,h_800,f_auto,q_auto/<publicId>.jpg`

### Best practices
- Dùng `f_auto,q_auto` để tự chọn định dạng/chất lượng.
- Resize/crop trên CDN thay vì tại client/server.
- Bật `secure=true` để luôn dùng HTTPS.
- Với React/Next: custom loader Cloudinary hoặc `<CldImage />` (frontend-frameworks).

### Caching
- Cloudinary CDN tự cache; hạn chế tạo quá nhiều biến thể URL.
- Dùng folder + naming nhất quán để quản trị.
- Khi cần xóa phiên bản cũ, cân nhắc invalidate hoặc versioning qua `publicId` mới.

## Senior — Bảo mật & vận hành
### Ký số bắt buộc
- Luôn ký số tại server cho upload quan trọng; unsigned chỉ kèm preset khóa chặt (size/type/folder, disable overwrite).

### Webhook (notification_url)
- Đăng ký webhook để nhận sự kiện upload thành công/thất bại; ghi log và cập nhật DB (status, size, mime).

### Error handling
- Thêm retry có backoff cho upload.
- Giới hạn concurrency khi xử lý nhiều file lớn.
- Log requestId/assetId từ response để truy vết.

## Tài liệu tham khảo nhanh
- Upload: https://cloudinary.com/documentation/upload_images
- .NET SDK: https://cloudinary.com/documentation/dotnet_integration
- Node SDK: https://cloudinary.com/documentation/node_integration
- Image transformations: https://cloudinary.com/documentation/image_transformations

