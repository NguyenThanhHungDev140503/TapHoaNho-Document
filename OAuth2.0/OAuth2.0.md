# OAuth 2.0 & OpenID Connect: Hướng Dẫn Toàn Diện

## Mục Lục
- [Giới Thiệu Tổng Quan](#giới-thiệu-tổng-quan)
- [Junior Level - Cơ Bản](#junior-level---cơ-bản)
- [Middle Level - Trung Cấp](#middle-level---trung-cấp)
- [Senior Level - Nâng Cao](#senior-level---nâng-cao)

---

## Giới Thiệu Tổng Quan

**OAuth 2.0** là giao thức ủy quyền (authorization) tiêu chuẩn, cho phép ứng dụng của bạn truy cập tài nguyên thay mặt cho người dùng. **OpenID Connect (OIDC)** là một lớp định danh (authentication) nằm trên OAuth 2.0.

- **Identity Provider (IdP)**: Nơi chứa user database và cấp phát token (ví dụ: Auth0, Duende IdentityServer, Google).
- **Client**: Ứng dụng React của bạn.
- **Resource Server**: .NET API của bạn.

### Các Khái Niệm Cốt Lõi
1. **Access Token**: "Chìa khóa" để truy cập API (thường là JWT). Có thời hạn ngắn.
2. **Refresh Token**: Dùng để xin Access Token mới khi cái cũ hết hạn mà không cần user login lại.
3. **PKCE (Proof Key for Code Exchange)**: Cơ chế bảo mật bắt buộc cho SPA để chống lại việc đánh cắp Authorization Code.

---

## Junior Level - Cơ Bản

### 1. Cài Đặt (Frontend - React)

Sử dụng thư viện `react-oidc-context` (wrapper chuẩn của `oidc-client-ts`).

```bash
npm install react-oidc-context oidc-client-ts
```

Setup `AuthProvider` ở `App.tsx`:

```tsx
import { AuthProvider, AuthProviderProps } from "react-oidc-context";

const oidcConfig: AuthProviderProps = {
  authority: "https://your-auth-server.com", // URL của IdP
  client_id: "your-client-id",
  redirect_uri: window.location.origin, // Nơi user quay về sau khi login
  scope: "openid profile api.read", // Các quyền muốn xin
  onSigninCallback: (_user) => {
      // Xóa query params sau khi login thành công để URL đẹp hơn
      window.history.replaceState({}, document.title, window.location.pathname);
  }
};

function App() {
  return (
    <AuthProvider {...oidcConfig}>
      <YourApp />
    </AuthProvider>
  );
}
```

### 2. Login, Logout & User Info (Frontend)

Sử dụng hook `useAuth()`:

```tsx
import { useAuth } from "react-oidc-context";

function UserProfile() {
  const auth = useAuth();

  if (auth.isLoading) return <div>Loading...</div>;
  if (auth.error) return <div>Auth Error: {auth.error.message}</div>;

  if (auth.isAuthenticated) {
    return (
      <div>
        <pre>Hello: {auth.user?.profile.name}</pre>
        <pre>Access Token: {auth.user?.access_token}</pre>
        <button onClick={() => auth.removeUser()}>Log out</button>
      </div>
    );
  }

  return <button onClick={() => auth.signinRedirect()}>Log in</button>;
}
```

### 3. Cài Đặt (Backend - .NET Core)

Cài đặt package JWT Bearer:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

Cấu hình trong `Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

// 1. Add Authentication Services
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://your-auth-server.com"; // Phải khớp với config ở React
        options.Audience = "api"; // Tên Resource mà Client xin quyền truy cập
        
        // Development only: Bỏ qua HTTPS check nếu chạy local
        options.RequireHttpsMetadata = false; 
    });

var app = builder.Build();

// 2. Use Middleware (Thứ tự quan trọng!)
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.Run();
```

### 4. Bảo Vệ API Endpoint

Thêm attribute `[Authorize]` vào Controller hoặc Action cần bảo vệ.

```csharp
[ApiController]
[Route("[controller]")]
[Authorize] // 🔒 Yêu cầu phải có Token hợp lệ mới vào được
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        // ...
    }
}
```

---

## Middle Level - Trung Cấp

### 1. Token Management & API Requests (Frontend)

Không nên thủ công lấy token rồi add vào header ở mọi chỗ. Hãy dùng **Interceptor** (nếu dùng Axios) hoặc custom fetch wrapper.

**Ví dụ với Axios:**

```ts
import axios from 'axios';
import { User } from 'oidc-client-ts';

// Helper load user từ storage (sessionStorage/localStorage)
function getUser() {
  const oidcStorage = sessionStorage.getItem(`oidc.user:https://your-auth-server.com:your-client-id`);
  if (!oidcStorage) return null;
  return User.fromStorageString(oidcStorage);
}

const apiClient = axios.create({ baseURL: 'https://api.yourservice.com' });

apiClient.interceptors.request.use(async (config) => {
  const user = getUser();
  // Nếu có access token hợp lệ, attach vào header
  if (user?.access_token) {
    config.headers.Authorization = `Bearer ${user.access_token}`;
  }
  return config;
});
```

### 2. Silent Refresh (Tự động gia hạn Token)

`oidc-client-ts` hỗ trợ tự động refresh token trước khi nó hết hạn.
Trong `oidcConfig` ở Junior Level, thêm:

```ts
const oidcConfig = {
  // ...
  automaticSilentRenew: true, // Auto call refresh endpoint
};
```

**Lưu ý**: Refresh Token cần được Auth Server hỗ trợ và Client phải xin scope `offline_access`.

### 3. Phân quyền dựa trên Claims (.NET Backend)

Nếu bạn muốn endpoint chỉ dành cho Admin, đừng chỉ check `[Authorize]`. Hãy check **Roles** hoặc **Policies**.

```csharp
// Program.cs - Định nghĩa Policy
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("MustBeAdmin", policy => 
        policy.RequireClaim("role", "admin")); // Check claim "role" có value "admin"
});

// Controller
[Authorize(Policy = "MustBeAdmin")] // ✅
[HttpPost]
public IActionResult DeleteUser(int id) { ... }
```

---

## Senior Level - Nâng Cao

### 1. Xử lý Token Expiration & Error 401

Khi Access Token hết hạn ngay giữa chừng request (hoặc Silent Renew thất bại), API trả về 401. Bạn cần cơ chế retry.

- **Interceptor Response**:
  1. Catch lỗi 401.
  2. Gọi hàm renew token thủ công (`userManager.signinSilent()`).
  3. Lấy token mới, update header của failed request.
  4. Retry request đó.

### 2. Custom Authorization Handlers

Với các logic phức tạp (ví dụ: User phải > 18 tuổi VÀ đã verify email), `RequireClaim` là không đủ.

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(c => c.Type == ClaimTypes.DateOfBirth);
        // Logic tính tuổi...
        if (age >= requirement.MinimumAge) context.Succeed(requirement);
        return Task.CompletedTask;
    }
}
```

### 3. Monitoring & Security Headers

Đảm bảo API luôn trả về các header bảo mật:
- `Strict-Transport-Security` (HSTS).
- `Content-Security-Policy` (CSP).

---

## Common Pitfalls (Cần tránh)

| Sai Lầm (❌) | Cách Làm Đúng (✅) |
|--------------|--------------------|
| Lưu Access Token vào `localStorage` lâu dài cho app tài chính/banking. | Sử dụng mô hình BFF (Backend for Frontend) với HttpOnly Cookie (Xem file `Principal-Level-Patterns.md`). |
| Implement luồng "Implicit Flow" (response_type=token). | Luôn dùng "Authorization Code Flow with PKCE" (response_type=code). |
| Validate Token thủ công (parse string). | Dùng middleware chuẩn `AddJwtBearer`. |
