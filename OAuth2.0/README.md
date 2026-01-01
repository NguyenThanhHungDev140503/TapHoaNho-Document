# OAuth 2.0 & OpenID Connect Guidelines

## Tổng Quan
Bộ tài liệu này cung cấp hướng dẫn toàn diện về việc triển khai OAuth 2.0 và OpenID Connect (OIDC) trong hệ sinh thái .NET Core (Backend) và React (Frontend). Chúng tôi tập trung vào các chuẩn bảo mật hiện đại nhất như **Authorization Code Flow with PKCE**.

## Cấu Trúc Tài Liệu

| File | Level | Nội Dung |
|------|-------|----------|
| **[OAuth2.0.md](./OAuth2.0.md)** | Junior/Middle | Hướng dẫn cốt lõi: Cài đặt Auth cơ bản, cấu hình .NET API `JwtBearer`, tích hợp React với `oidc-client-ts`. |
| **[Advanced-Patterns.md](./Advanced-Patterns.md)** | Senior | Xử lý Token Refresh, Axios Interceptors, Custom Authorization Handlers, Performance. |
| **[Principal-Level-Patterns.md](./Principal-Level-Patterns.md)** | Principal | Kiến trúc bảo mật cấp cao: BFF (Backend for Frontend), Centralized Identity, System Design. |
| **[RESEARCH_SUMMARY.md](./RESEARCH_SUMMARY.md)** | All | Tóm tắt quá trình nghiên cứu, nguồn tham khảo và các quyết định kiến trúc. |

## Cách Sử Dụng Tài Liệu

- **Junior Developer**:
  - Đọc kỹ **Mục 3 (Junior Level)** trong `OAuth2.0.md`.
  - Thực hành setup một React App login được với Identity Provider demo (ví dụ Google hoặc Duende Demo).
  - Hiểu cách gọi API với Bearer Token.

- **Middle Developer**:
  - Đọc hết `OAuth2.0.md` và phần đầu của `Advanced-Patterns.md`.
  - Nắm vững cơ chế **Token Rotation** và xử lý lỗi 401/403.
  - Implement được Dynamic Permissions/Policies trên Backend.

- **Senior Developer**:
  - Review toàn bộ `Advanced-Patterns.md`.
  - Tối ưu hóa việc gọi API (batching, caching permissions).
  - Review code và config security cho team.

- **Principal/Architect**:
  - Tập trung vào `Principal-Level-Patterns.md`.
  - Ra quyết định về việc sử dụng mô hình SPA thuần túy hay **BFF (Backend for Frontend)** để tối ưu bảo mật.
  - Thiết kế chiến lược Centralized Identity cho toàn bộ hệ thống Microservices.

## Key Concepts
- **PKCE (Proof Key for Code Exchange)**: Cơ chế bảo mật bắt buộc cho Mobile/Web Apps để ngăn chặn tấn công chèn mã độc vào luồng Auth Code.
- **JWT (JSON Web Token)**: Định dạng token phổ biến chứa thông tin user (claims) và chữ ký số.
- **BFF (Backend for Frontend)**: Pattern dùng server-side proxy để giữ Access Token, Frontend chỉ dùng Cookie.

## Quick Start (Minimal Setup)

### 1. Backend (.NET Core)
```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.Authority = "https://demo.duendesoftware.com"; // Identity Provider
        options.Audience = "api";
    });
```

### 2. Frontend (React)
```bash
npm install react-oidc-context oidc-client-ts
```
```tsx
// App.tsx
import { AuthProvider } from "react-oidc-context";

const oidcConfig = {
  authority: "https://demo.duendesoftware.com",
  client_id: "interactive.public",
  redirect_uri: window.location.origin,
  // ...
};

return <AuthProvider {...oidcConfig}><App /></AuthProvider>;
```
