# Advanced Patterns: OAuth 2.0 Implementation

Tài liệu này dành cho Senior Developers, tập trung vào việc tối ưu hóa, xử lý các kịch bản phức tạp và tích hợp sâu (deep integration).

## 1. Custom Auth Hook Architecture

Thay vì dùng trực tiếp `useAuth()` sơ khai từ library ở mọi nơi, hãy tạo một custom hook `useMyAuth()` để normalize data và expose helper methods phù hợp với domain của dự án.

```tsx
// hooks/useMyAuth.ts
import { useAuth } from "react-oidc-context";

export const useMyAuth = () => {
    const auth = useAuth();
    
    // Derived state
    const isAdmin = auth.user?.profile?.role === 'admin';
    const userId = auth.user?.profile?.sub;
    
    // Helper check permission (giả sử permission lưu trong claim 'permissions')
    const hasPermission = (perm: string) => {
        const perms = (auth.user?.profile?.permissions as string[]) || [];
        return perms.includes(perm);
    };

    return {
        ...auth,
        isAdmin,
        userId,
        hasPermission,
        token: auth.user?.access_token
    };
};
```

**Lợi ích:**
- **Abstraction**: UI Components không cần biết cấu trúc sâu của User Profile.
- **Maintainability**: Logic check quyền (`isAdmin`) nằm ở một chỗ duy nhất.

## 2. Global Error Handling & Token Rotation

Khi Access Token hết hạn, API trả về 401. User không nên bị logout ngay lập tức. Cần cơ chế Silent Refresh và Retry Request.

### Axios Interceptor Strategy

```ts
// api/axiosClient.ts
import axios from "axios";
import { User, UserManager } from "oidc-client-ts";

const userManager = new UserManager({/*...config...*/});

apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true; // Đánh dấu đã retry 1 lần để tránh loop vô hạn

            try {
                // 1. Thử renew token
                const user = await userManager.signinSilent();
                
                // 2. Update token mới vào header
                axios.defaults.headers.common["Authorization"] = "Bearer " + user.access_token;
                originalRequest.headers["Authorization"] = "Bearer " + user.access_token;

                // 3. Gọi lại request ban đầu
                return apiClient(originalRequest);
            } catch (err) {
                // Renew thất bại -> Token thật sự hết hạn hoặc Session chết -> Logout
                await userManager.signoutRedirect(); 
                return Promise.reject(err);
            }
        }
        return Promise.reject(error);
    }
);
```

## 3. Dynamic Policy Authorization (.NET Core)

Khi hệ thống lớn, việc hardcode Policy trong `Program.cs` (`RequireClaim("role", "admin")`) là không đủ, vì quyền hạn có thể thay đổi động trong Database.

**Giải pháp**: Implement `IAuthorizationPolicyProvider`.

```csharp
public class PermissionPolicyProvider : IAuthorizationPolicyProvider
{
    public Task<AuthorizationPolicy> GetPolicyAsync(string policyName)
    {
        // Tự động tạo Policy dựa trên tên (VD: "Permission:User.Create")
        if (policyName.StartsWith("Permission:", StringComparison.OrdinalIgnoreCase))
        {
            var permission = policyName.Substring("Permission:".Length);
            var policy = new AuthorizationPolicyBuilder()
                .RequireAuthenticatedUser()
                .AddRequirements(new PermissionRequirement(permission))
                .Build();
            return Task.FromResult(policy);
        }
        
        // Fallback về default provider
        return Task.FromResult<AuthorizationPolicy>(null);
    }
    // ... implement other methods
}
```

Sử dụng:
```csharp
[Authorize(Policy = "Permission:Product.Edit")]
public IActionResult EditProduct() { ... }
```

## 4. Performance Optimization

### Caching User Permissions
Việc giải mã JWT và check permissions tuy nhanh nhưng nếu application gọi `hasPermission()` liên tục trong các vòng lặp render lớn, có thể gây chậm.
- **React**: Memoize kết quả của hook `useMyAuth` hoặc dùng `useMemo` cho danh sách permissions.
- **Backend**: Caching Policies PolicyProvider nếu việc query database lấy permission quá chậm.

### Payload Size Optimization
Access Token không nên quá lớn. Đừng nhét toàn bộ thông tin user vào token.
- **Bad**: Access Token chứa full address, phone, bio.
- **Good**: Access Token chỉ chứa `sub` (userId), `role`, `scope`.
- Dùng **UserInfo Endpoint** (`/connect/userinfo`) để lấy thông tin chi tiết khi cần hiển thị Profile.
