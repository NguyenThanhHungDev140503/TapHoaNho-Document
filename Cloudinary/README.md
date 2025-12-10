# Cloudinary Documentation

Tổng quan sử dụng Cloudinary để lưu trữ, biến đổi và phân phối ảnh/video cho hệ thống RetailStoreManagement (React frontend + .NET 9 backend).

## Cấu Trúc Tài Liệu
- `Cloudinary.md` — Hướng dẫn chính (Junior/Middle/Senior).
- `Advanced-Patterns.md` — Patterns nâng cao (ký số, widget, tối ưu delivery).
- `Principal-Level-Patterns.md` — Kiến trúc enterprise, scale, giám sát.
- `RESEARCH_SUMMARY.md` — Nguồn research, phát hiện chính, khuyến nghị.

## Cách Sử Dụng Tài Liệu
- Junior: đọc mục Junior trong `Cloudinary.md`, thử upload ký số cơ bản.
- Middle: tiếp tục phần Middle (transform/delivery/caching) trong `Cloudinary.md`.
- Senior: đọc phần Senior + toàn bộ `Advanced-Patterns.md`.
- Principal/Staff: xem `Principal-Level-Patterns.md` (kiến trúc, chi phí, observability).

## Key Concepts
- Signed upload: chữ ký tạo ở backend bằng `api_key`, `api_secret`, `timestamp`.
- Upload preset: cấu hình sẵn (folder, allowed formats/size, overwrite) dùng cho unsigned uploads.
- Transformation URL: `/image/upload/<transformations>/<publicId>` với `f_auto,q_auto,c_fill,w,h`.

## External Resources
- Cloudinary Upload guide: https://cloudinary.com/documentation/upload_images
- Cloudinary .NET SDK: https://cloudinary.com/documentation/dotnet_integration
- Cloudinary Node SDK: https://cloudinary.com/documentation/node_integration
- Image transformations: https://cloudinary.com/documentation/image_transformations

## Quick Start
```bash
# Backend (.NET)
dotnet add package CloudinaryDotNet

# Frontend (Node)
npm install cloudinary
```

```csharp
// .NET configure (appsettings)
"Cloudinary": { "CloudName": "...", "ApiKey": "...", "ApiSecret": "..." }
```

```typescript
// Node SDK init
import { v2 as cloudinary } from 'cloudinary'
cloudinary.config({ cloud_name: '...', api_key: '...', api_secret: '...' })
```

