# Research Summary: OAuth 2.0 & OpenID Connect

## 1. Mục Tiêu Research
- **Mục đích**: Xây dựng hướng dẫn chuẩn hóa (Guideline) về việc implement xác thực (Authentication) và phân quyền (Authorization) sử dụng OAuth 2.0 và OpenID Connect (OIDC) cho hệ thống sử dụng .NET Core (Backend) và React (Frontend).
- **Phạm vi**:
  - **Backend**: ASP.NET Core Web API (sử dụng JWT Bearer Authentication).
  - **Frontend**: React Single Page Application (SPA) (sử dụng Authorization Code Flow với PKCE).
  - **Level**: Từ cơ bản (Junior) đến nâng cao (Principal/Architect - BFF Pattern).

## 2. Nguồn Thông Tin Đã Sử Dụng
1. **Ory Documentation - OAuth2 Authorization Code Flow**: Giải thích chi tiết về luồng PKCE và lý do tại sao nó cần thiết cho SPA.
2. **Duende IdentityServer / OpenIddict Docs**: Tham khảo về các patterns cho Auth Server (.NET).
3. **Microsoft Security Documentation**: Các best practices cho `Microsoft.AspNetCore.Authentication.JwtBearer`.
4. **React Libraries**: Nghiên cứu `oidc-client-ts` và `react-oidc-context`.
5. **Security Blogs (Okta, Auth0)**: Các bài viết về BFF (Backend for Frontend) pattern và tại sao nên tránh lưu Token ở LocalStorage cho các ứng dụng yêu cầu bảo mật cao.

## 3. Phát Hiện Chính (Key Findings)
- **Standard Protocol**: **Authorization Code Flow with PKCE** (Proof Key for Code Exchange) hiện là chuẩn bắt buộc cho SPA. Implicit Flow đã lỗi thời và không an toàn.
- **Library Selection**:
  - **Frontend**: `react-oidc-context` (wrapper của `oidc-client-ts`) là lựa chọn phổ biến, lightweight và chuẩn xác nhất cho React.
  - **Backend**: `Microsoft.AspNetCore.Authentication.JwtBearer` là standard cho Resource Server (API).
- **Token Storage Dilemma**:
  - **Junior/Middle Approach**: Lưu Access Token trong `sessionStorage` hoặc `localStorage` (dễ implement nhưng rủi ro XSS).
  - **Senior/Principal Approach**: Sử dụng **BFF (Backend for Frontend)** pattern. Token được giữ ở server-side (proxy/backend), browser chỉ giữ `httpOnly` cookie. Điều này loại bỏ hoàn toàn khả năng bị đánh cắp token qua XSS.

## 4. Kiến Trúc/Cách Dùng Đề Xuất
- **Default Architecture (SPA + API)**:
  - React App dùng `react-oidc-context` để login -> nhận Token -> Lưu In-Memory/Storage -> Gửi kèm Header `Authorization: Bearer` trong API request.
  - .NET API dùng `JwtBearer` để validate token.
- **Enterprise Architecture (High Security)**:
  - Sử dụng mô hình **BFF** (ví dụ dùng YARP Reverse Proxy trong .NET). React App chỉ gọi đến BFF endpoint (dùng Cookie), BFF sẽ attach Token vào request gọi xuống Downstream API.

## 5. Use Cases Đã Xác Định
- **UC1: Public Client Login**: React app login với Google/Auth0/IdentityServer (Chi tiết trong `OAuth2.0.md`).
- **UC2: Protect API Endpoint**: .NET API yêu cầu token hợp lệ để truy cập data (Chi tiết trong `OAuth2.0.md`).
- **UC3: Token Refresh**: Tự động renew token khi hết hạn mà không logout user (Chi tiết trong `Advanced-Patterns.md`).
- **UC4: Custom Permission/Policy**: Phân quyền chi tiết dựa trên Claims trong Token (Chi tiết trong `Advanced-Patterns.md`).
- **UC5: Secure Token Storage (BFF)**: Chuyển đổi từ Token-based sang Cookie-based session cho Frontend (Chi tiết trong `Principal-Level-Patterns.md`).
