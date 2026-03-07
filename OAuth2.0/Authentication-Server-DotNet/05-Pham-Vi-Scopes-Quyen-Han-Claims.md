# Phần 5: Phạm vi (Scopes) và Quyền hạn (Claims)

## 5.1 Định nghĩa Scopes

### 5.1.1 Scope là gì?

Scope là một chuỗi định nghĩa quyền truy cập cụ thể mà client yêu cầu. Scope được sử dụng để giới hạn quyền truy cập của access token, đảm bảo rằng client chỉ có thể truy cập các tài nguyên được phép.

### 5.1.2 Tại sao cần Scopes?

#### 1. Principle of Least Privilege

**Vấn đề:** Nếu client có quyền truy cập tất cả mọi tài nguyên, rủi ro bảo mật cao.

**Giải pháp:** Sử dụng scopes để cấp phát quyền truy cập tối thiểu.

**Ví dụ:**
- Mobile app chỉ cần đọc profile, không cần sửa đổi
- Web app chỉ cần đọc sản phẩm, không cần xóa
- Background service chỉ cần đọc báo cáo, không cần quản lý user

#### 2. Fine-grained Access Control

**Vấn đề:** Cần kiểm soát chi tiết quyền truy cập cho từng API endpoint.

**Giải pháp:** Sử dụng scopes để định nghĩa quyền truy cập cho từng nhóm API.

**Ví dụ:**
- `api1.read`: Đọc dữ liệu từ API 1
- `api1.write`: Ghi dữ liệu vào API 1
- `api1.delete`: Xóa dữ liệu từ API 1
- `api1.admin`: Quản trị API 1

#### 3. User Consent

**Vấn đề:** User cần biết và chấp nhận quyền truy cập mà client yêu cầu.

**Giải pháp:** Hiển thị scopes rõ ràng trong trang consent.

**Ví dụ:**
- "Ứng dụng này muốn truy cập tên và email của bạn"
- "Ứng dụng này muốn đọc và ghi dữ liệu sản phẩm của bạn"

### 5.1.3 Các loại Scopes

#### 1. OpenID Connect Scopes

**openid**: Bắt buộc cho OpenID Connect, cho phép client nhận ID token.

**profile**: Cho phép client truy cập thông tin profile cơ bản.

**email**: Cho phép client truy cập địa chỉ email.

**address**: Cho phép client truy cập địa chỉ.

**phone**: Cho phép client truy cập số điện thoại.

**Ví dụ:**
```csharp
services.AddIdentityServer()
    .AddInMemoryIdentityResources(new List<IdentityResource>
    {
        new IdentityResources.OpenId(),
        new IdentityResources.Profile(),
        new IdentityResources.Email(),
        new IdentityResources.Address(),
        new IdentityResources.Phone()
    });
```

#### 2. API Scopes

**Định nghĩa:** Scopes cho phép truy cập các API resources.

**Ví dụ:**
- `api1.read`: Đọc dữ liệu từ API 1
- `api1.write`: Ghi dữ liệu vào API 1
- `api1.admin`: Quản trị API 1
- `api2.read`: Đọc dữ liệu từ API 2

**Triển khai:**
```csharp
services.AddIdentityServer()
    .AddInMemoryApiResources(new List<ApiResource>
    {
        new ApiResource("api1")
        {
            DisplayName = "API 1",
            Description = "API for accessing application resources",
            Scopes = { 
                new Scope("api1.read", "Read access to API 1"),
                new Scope("api1.write", "Write access to API 1"),
                new Scope("api1.admin", "Admin access to API 1")
            },
            UserClaims = { "name", "email", "role" }
        },
        new ApiResource("api2")
        {
            DisplayName = "API 2",
            Description = "API for accessing application resources",
            Scopes = { 
                new Scope("api2.read", "Read access to API 2"),
                new Scope("api2.write", "Write access to API 2")
            },
            UserClaims = { "name", "email", "role" }
        }
    });
```

#### 3. Custom Scopes

**Định nghĩa:** Scopes tùy chỉnh theo yêu cầu của ứng dụng.

**Ví dụ:**
- `products.read`: Đọc sản phẩm
- `products.write`: Ghi sản phẩm
- `orders.read`: Đọc đơn hàng
- `orders.write`: Ghi đơn hàng
- `reports.read`: Đọc báo cáo
- `reports.export`: Xuất báo cáo

**Triển khai:**
```csharp
services.AddIdentityServer()
    .AddInMemoryApiResources(new List<ApiResource>
    {
        new ApiResource("products-api")
        {
            DisplayName = "Products API",
            Description = "API for managing products",
            Scopes = { 
                new Scope("products.read", "Read products"),
                new Scope("products.write", "Create and update products"),
                new Scope("products.delete", "Delete products")
            }
        },
        new ApiResource("orders-api")
        {
            DisplayName = "Orders API",
            Description = "API for managing orders",
            Scopes = { 
                new Scope("orders.read", "Read orders"),
                new Scope("orders.write", "Create and update orders"),
                new Scope("orders.delete", "Delete orders")
            }
        },
        new ApiResource("reports-api")
        {
            DisplayName = "Reports API",
            Description = "API for generating reports",
            Scopes = { 
                new Scope("reports.read", "Read reports"),
                new Scope("reports.export", "Export reports")
            }
        }
    });
```

### 5.1.4 Định nghĩa Scopes

#### 1. Naming Conventions

**Best practices:**
- Sử dụng format: `resource.action`
- Sử dụng lowercase
- Sử dụng dấu chấm (.) để phân tách resource và action
- Sử dụng động từ: read, write, delete, admin, manage

**Ví dụ tốt:**
- `api1.read`
- `products.write`
- `orders.delete`
- `reports.export`

**Ví dụ kém:**
- `API1_READ` (không nhất quán)
- `read-api1` (không rõ ràng)
- `api1` (không cụ thể)

#### 2. Scope Description

**Tại sao cần description?**
- Giúp user hiểu quyền truy cập
- Hiển thị trong trang consent
- Tài liệu cho nhà phát triển

**Ví dụ:**
```csharp
new Scope("products.read", "Read product information including name, price, and description")
new Scope("products.write", "Create and update product information")
new Scope("orders.read", "Read order information including status and items")
new Scope("orders.write", "Create and update order information including status and items")
```

#### 3. Scope Hierarchy

**Tại sao cần hierarchy?**
- Giảm số lượng scopes
- Dễ dàng quản lý
- Rõ ràng hơn cho user

**Ví dụ:**
```
api1.read
  ├── products.read
  ├── orders.read
  └── reports.read

api1.write
  ├── products.write
  ├── orders.write
  └── reports.export
```

**Triển khai:**
```csharp
services.AddIdentityServer()
    .AddInMemoryApiResources(new List<ApiResource>
    {
        new ApiResource("api1")
        {
            DisplayName = "API 1",
            Description = "API for accessing application resources",
            Scopes = { 
                new Scope("api1.read", "Read access to API 1", 
                    new List<string> { "products.read", "orders.read", "reports.read" }),
                new Scope("api1.write", "Write access to API 1",
                    new List<string> { "products.write", "orders.write", "reports.export" })
            },
            UserClaims = { "name", "email", "role" }
        }
    });
```

## 5.2 Cấp phát và kiểm tra Scopes

### 5.2.1 Cấp phát Scopes

#### 1. Client Configuration

**Định nghĩa scopes cho client:**
```csharp
new Client
{
    ClientId = "web-client",
    ClientName = "Web Application",
    AllowedScopes = { 
        "openid", 
        "profile", 
        "api1.read", 
        "api1.write" 
    }
}
```

#### 2. User Consent

**Hiển thị scopes trong trang consent:**
```html
<form method="post" action="/connect/authorize/consent">
    <h2>Web Application muốn truy cập:</h2>
    
    <h3>Thông tin cá nhân của bạn</h3>
    <p>Web Application muốn truy cập tên và email của bạn.</p>
    <input type="checkbox" name="granted_scopes" value="openid" checked disabled />
    <input type="checkbox" name="granted_scopes" value="profile" checked disabled />
    
    <h3>API của bạn</h3>
    <p>Web Application muốn đọc và ghi dữ liệu trên API của bạn.</p>
    <input type="checkbox" name="granted_scopes" value="api1.read" checked />
    <input type="checkbox" name="granted_scopes" value="api1.write" checked />
    
    <label>
        <input type="checkbox" name="remember_consent" value="true" />
        Nhớ quyết định của tôi
    </label>
    
    <button type="submit" name="consent" value="grant">Chấp nhận</button>
    <button type="submit" name="consent" value="deny">Từ chối</button>
</form>
```

#### 3. Lưu trữ User Consent

**Triển khai:**
```csharp
public class UserConsent
{
    public string UserId { get; set; }
    public string ClientId { get; set; }
    public List<string> GrantedScopes { get; set; }
    public DateTime ConsentDate { get; set; }
    public bool Remembered { get; set; }
}

public class UserConsentRepository : IUserConsentRepository
{
    public async Task SaveConsentAsync(UserConsent consent)
    {
        _context.UserConsents.Add(consent);
        await _context.SaveChangesAsync();
    }
    
    public async Task<UserConsent> GetConsentAsync(string userId, string clientId)
    {
        return await _context.UserConsents
            .FirstOrDefaultAsync(c => c.UserId == userId && c.ClientId == clientId);
    }
}
```

### 5.2.2 Kiểm tra Scopes

#### 1. Client-side Request

**Yêu cầu scopes khi lấy token:**
```csharp
var properties = new AuthenticationProperties
{
    RedirectUri = Url.Action(nameof(Callback))
};

properties.Items["scope"] = "openid profile api1.read api1.write";

return Challenge(properties, "oidc");
```

#### 2. Authorization Server Validation

**Validate scopes khi phát hành token:**
```csharp
public class CustomTokenRequestValidator : ICustomTokenRequestValidator
{
    public async Task ValidateAsync(CustomTokenRequestValidationContext context)
    {
        // Validate scopes
        if (context.ValidatedRequest.RequestedScopes.Any())
        {
            var allowedScopes = context.ValidatedRequest.Client.AllowedScopes;
            var requestedScopes = context.ValidatedRequest.RequestedScopes;
            
            foreach (var scope in requestedScopes)
            {
                if (!allowedScopes.Contains(scope))
                {
                    _logger.LogWarning("Client {ClientId} requested scope {Scope} that is not allowed", 
                        context.ValidatedRequest.Client.ClientId, scope);
                    context.Result = new ValidationResult(
                        new TokenRequestValidationError
                        {
                            Error = OAuth2Constants.TokenErrors.InvalidScope,
                            ErrorDescription = $"Scope '{scope}' is not allowed for this client"
                        });
                    return;
                }
            }
        }
        
        await Task.CompletedTask;
    }
}
```

#### 3. Resource Server Validation

**Validate scopes khi truy cập API:**
```csharp
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "ReadScope")]
    public async Task<IActionResult> GetProducts()
    {
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
    
    [HttpPost]
    [Authorize(Policy = "WriteScope")]
    public async Task<IActionResult> CreateProduct([FromBody] CreateProductDto dto)
    {
        var product = await _productService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
    
    [HttpDelete("{id}")]
    [Authorize(Policy = "DeleteScope")]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        await _productService.DeleteAsync(id);
        return NoContent();
    }
}
```

**Định nghĩa policy:**
```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("ReadScope", policy =>
        policy.RequireClaim("scope", "products.read"));
    
    options.AddPolicy("WriteScope", policy =>
        policy.RequireClaim("scope", "products.write"));
    
    options.AddPolicy("DeleteScope", policy =>
        policy.RequireClaim("scope", "products.delete"));
});
```

#### 4. Custom Scope Validation

**Validate scopes tùy chỉnh:**
```csharp
public class ScopeRequirement : IAuthorizationRequirement
{
    public string Scope { get; }

    public ScopeRequirement(string scope)
    {
        Scope = scope;
    }
}

public class ScopeHandler : AuthorizationHandler<ScopeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ScopeRequirement requirement)
    {
        var scopeClaim = context.User.FindFirst("scope");
        
        if (scopeClaim == null || scopeClaim.Value != requirement.Scope)
        {
            context.Fail();
            return Task.CompletedTask;
        }
        
        context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

// Sử dụng
[HttpGet]
[Authorize(Policy = "ReadScope")]
public async Task<IActionResult> GetProducts()
{
    // ...
}

// Định nghĩa policy
services.AddAuthorization(options =>
{
    options.AddPolicy("ReadScope", policy =>
        policy.Requirements.Add(new ScopeRequirement("products.read")));
});
```

## 5.3 Claims và mapping

### 5.3.1 Claims là gì?

Claim là một cặp key-value chứa thông tin về subject (thường là user). Claims được lưu trữ trong token và được sử dụng để xác thực và ủy quyền.

### 5.3.2 Các loại Claims

#### 1. Standard Claims

**iss (Issuer):** Người phát hành token

**sub (Subject):** Subject của token (thường là user ID)

**aud (Audience):** Audience được phép sử dụng token

**exp (Expiration Time):** Thời gian hết hạn

**nbf (Not Before):** Token không hợp lệ trước thời gian này

**iat (Issued At):** Thời gian phát hành

**jti (JWT ID):** ID duy nhất của token

#### 2. Profile Claims

**name:** Tên đầy đủ

**email:** Địa chỉ email

**given_name:** Tên

**family_name:** Họ

**picture:** URL ảnh đại diện

**locale:** Ngôn ngữ

#### 3. Custom Claims

**role:** Vai trò của user

**permissions:** Quyền hạn cụ thể

**tenant_id:** Tenant ID (cho multi-tenant)

**department:** Phòng ban

**organization:** Tổ chức

### 5.3.3 Claims Mapping

#### 1. Mapping từ User Profile

**Triển khai:**
```csharp
public class ClaimsService : IClaimsService
{
    public IEnumerable<Claim> GetClaimsFromUser(ApplicationUser user)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };
        
        // Add name claims
        if (!string.IsNullOrEmpty(user.FirstName))
        {
            claims.Add(new Claim(JwtRegisteredClaimNames.GivenName, user.FirstName));
        }
        
        if (!string.IsNullOrEmpty(user.LastName))
        {
            claims.Add(new Claim(JwtRegisteredClaimNames.FamilyName, user.LastName));
        }
        
        // Add custom claims
        if (!string.IsNullOrEmpty(user.Role))
        {
            claims.Add(new Claim("role", user.Role));
        }
        
        if (!string.IsNullOrEmpty(user.TenantId))
        {
            claims.Add(new Claim("tenant_id", user.TenantId));
        }
        
        // Add permissions
        foreach (var permission in user.Permissions)
        {
            claims.Add(new Claim("permission", permission.Name));
        }
        
        return claims;
    }
}
```

#### 2. Custom Profile Service (Duende IdentityServer)

**Triển khai:**
```csharp
public class CustomProfileService : IProfileService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly IUserClaimsPrincipalFactory<ApplicationUser, string> _claimsFactory;
    
    public CustomProfileService(
        UserManager<ApplicationUser> userManager,
        IUserClaimsPrincipalFactory<ApplicationUser, string> claimsFactory)
    {
        _userManager = userManager;
        _claimsFactory = claimsFactory;
    }
    
    public async Task GetProfileDataAsync(ProfileDataRequest context)
    {
        var user = await _userManager.GetUserAsync(context.Subject.GetSubjectId());
        
        if (user == null)
        {
            throw new InvalidOperationException("User not found");
        }
        
        var principal = await _claimsFactory.CreateAsync(user);
        var claims = principal.Claims.ToList();
        
        // Add custom claims
        context.IssuedClaims.Add(new Claim("role", user.Role));
        context.IssuedClaims.Add(new Claim("tenant_id", user.TenantId));
        
        // Add permissions
        foreach (var permission in user.Permissions)
        {
            context.IssuedClaims.Add(new Claim("permission", permission.Name));
        }
        
        await Task.CompletedTask;
    }
}
```

#### 3. Claims Transformation

**Triển khai:**
```csharp
public class ClaimsTransformationService : IClaimsTransformationService
{
    public IEnumerable<Claim> TransformClaims(IEnumerable<Claim> claims)
    {
        var transformedClaims = new List<Claim>();
        
        // Transform role to permissions
        var roleClaim = claims.FirstOrDefault(c => c.Type == "role");
        if (roleClaim != null)
        {
            var permissions = GetPermissionsForRole(roleClaim.Value);
            foreach (var permission in permissions)
            {
                transformedClaims.Add(new Claim("permission", permission));
            }
        }
        
        // Add other claims
        foreach (var claim in claims)
        {
            if (claim.Type != "role")
            {
                transformedClaims.Add(claim);
            }
        }
        
        return transformedClaims;
    }
    
    private List<string> GetPermissionsForRole(string role)
    {
        // Map role to permissions
        return role switch
        {
            "admin" => new List<string> { "read:users", "write:users", "delete:users", "read:products", "write:products", "delete:products" },
            "user" => new List<string> { "read:products", "write:products" },
            "guest" => new List<string> { "read:products" },
            _ => new List<string>()
        };
    }
}
```

### 5.3.4 Claims trong Token

#### 1. Access Token Claims

**Ví dụ payload:**
```json
{
  "sub": "1234567890",
  "client_id": "web-client",
  "scope": "api1.read api1.write",
  "role": "admin",
  "tenant_id": "tenant-123",
  "permissions": ["read:users", "write:users", "read:products", "write:products"],
  "iss": "https://auth.example.com",
  "aud": "api1",
  "exp": 1516239022,
  "iat": 1516235422,
  "jti": "unique-token-id"
}
```

#### 2. ID Token Claims

**Ví dụ payload:**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "given_name": "John",
  "family_name": "Doe",
  "picture": "https://example.com/avatar.jpg",
  "iss": "https://auth.example.com",
  "aud": "web-client",
  "exp": 1516239022,
  "iat": 1516235422,
  "auth_time": 1516235422,
  "nonce": "random-nonce-value"
}
```

## 5.4 Ví dụ code Scopes và Claims

### 5.4.1 Định nghĩa Scopes trong Duende IdentityServer

```csharp
public static class Resources
{
    public static IEnumerable<IdentityResource> GetIdentityResources()
    {
        return new List<IdentityResource>
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile(),
            new IdentityResources.Email(),
            new IdentityResources.Phone(),
            new IdentityResources.Address()
        };
    }
    
    public static IEnumerable<ApiResource> GetApiResources()
    {
        return new List<ApiResource>
        {
            new ApiResource("products-api")
            {
                DisplayName = "Products API",
                Description = "API for managing products",
                Scopes = { 
                    new Scope("products.read", "Read products"),
                    new Scope("products.write", "Create and update products"),
                    new Scope("products.delete", "Delete products")
                },
                UserClaims = { "name", "email", "role" }
            },
            new ApiResource("orders-api")
            {
                DisplayName = "Orders API",
                Description = "API for managing orders",
                Scopes = { 
                    new Scope("orders.read", "Read orders"),
                    new Scope("orders.write", "Create and update orders"),
                    new Scope("orders.delete", "Delete orders")
                },
                UserClaims = { "name", "email", "role" }
            },
            new ApiResource("reports-api")
            {
                DisplayName = "Reports API",
                Description = "API for generating reports",
                Scopes = { 
                    new Scope("reports.read", "Read reports"),
                    new Scope("reports.export", "Export reports")
                },
                UserClaims = { "name", "email", "role" }
            }
        };
    }
}
```

### 5.4.2 Định nghĩa Claims trong Duende IdentityServer

```csharp
public class CustomProfileService : IProfileService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ApplicationDbContext _context;
    
    public CustomProfileService(
        UserManager<ApplicationUser> userManager,
        ApplicationDbContext context)
    {
        _userManager = userManager;
        _context = context;
    }
    
    public async Task GetProfileDataAsync(ProfileDataRequest context)
    {
        var user = await _userManager.GetUserAsync(context.Subject.GetSubjectId());
        
        if (user == null)
        {
            throw new InvalidOperationException("User not found");
        }
        
        // Add profile claims
        context.IssuedClaims.Add(new Claim(JwtRegisteredClaimNames.Name, $"{user.FirstName} {user.LastName}"));
        context.IssuedClaims.Add(new Claim(JwtRegisteredClaimNames.Email, user.Email));
        
        // Add custom claims
        context.IssuedClaims.Add(new Claim("role", user.Role));
        context.IssuedClaims.Add(new Claim("tenant_id", user.TenantId));
        
        // Add permissions
        var permissions = await _context.UserPermissions
            .Where(up => up.UserId == user.Id)
            .Select(up => up.Permission.Name)
            .ToListAsync();
        
        foreach (var permission in permissions)
        {
            context.IssuedClaims.Add(new Claim("permission", permission));
        }
        
        await Task.CompletedTask;
    }
}
```

### 5.4.3 Kiểm tra Scopes trong Resource Server

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;
    
    public ProductsController(IAuthorizationService authorizationService)
    {
        _authorizationService = authorizationService;
    }
    
    [HttpGet]
    [Authorize]
    public async Task<IActionResult> GetProducts()
    {
        // Kiểm tra scope
        if (!await _authorizationService.HasScopeAsync(User, "products.read"))
        {
            return Forbid();
        }
        
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
    
    [HttpPost]
    [Authorize]
    public async Task<IActionResult> CreateProduct([FromBody] CreateProductDto dto)
    {
        // Kiểm tra scope
        if (!await _authorizationService.HasScopeAsync(User, "products.write"))
        {
            return Forbid();
        }
        
        var product = await _productService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
    
    [HttpDelete("{id}")]
    [Authorize]
    public async Task<IActionResult> DeleteProduct(int id)
    {
        // Kiểm tra scope
        if (!await _authorizationService.HasScopeAsync(User, "products.delete"))
        {
            return Forbid();
        }
        
        await _productService.DeleteAsync(id);
        return NoContent();
    }
}

public class AuthorizationService : IAuthorizationService
{
    public async Task<bool> HasScopeAsync(ClaimsPrincipal user, string requiredScope)
    {
        var scopeClaim = user.FindFirst("scope");
        
        if (scopeClaim == null)
        {
            return false;
        }
        
        var scopes = scopeClaim.Value.Split(' ');
        return scopes.Contains(requiredScope);
    }
}
```

### 5.4.4 Kiểm tra Claims trong Resource Server

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [Authorize(Policy = "AdminOnly")]
    public async Task<IActionResult> GetUsers()
    {
        var users = await _userService.GetAllAsync();
        return Ok(users);
    }
    
    [HttpGet("{id}")]
    [Authorize]
    public async Task<IActionResult> GetUser(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        
        // Kiểm tra xem user có quyền truy cập thông tin này không
        var currentUserId = User.FindFirst(JwtRegisteredClaimNames.Sub)?.Value;
        var currentUserRole = User.FindFirst("role")?.Value;
        
        if (currentUserId == id.ToString())
        {
            // User có thể xem thông tin của chính mình
            return Ok(user);
        }
        
        if (currentUserRole == "admin")
        {
            // Admin có thể xem thông tin của tất cả users
            return Ok(user);
        }
        
        // User thường không có quyền xem thông tin của user khác
        return Forbid();
    }
}

// Định nghĩa policy
services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("admin"));
});
```

### 5.4.5 Custom Claims Authorization

```csharp
public class PermissionRequirement : IAuthorizationRequirement
{
    public string Permission { get; }

    public PermissionRequirement(string permission)
    {
        Permission = permission;
    }
}

public class PermissionHandler : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        var permissionClaims = context.User.FindAll("permission");
        var userPermissions = permissionClaims.Select(c => c.Value).ToList();
        
        if (!userPermissions.Contains(requirement.Permission))
        {
            context.Fail();
            return Task.CompletedTask;
        }
        
        context.Succeed(requirement);
        return Task.CompletedTask;
    }
}

// Sử dụng
[HttpPost]
[Authorize(Policy = "DeleteUserPermission")]
public async Task<IActionResult> DeleteUser(int id)
{
    await _userService.DeleteAsync(id);
    return NoContent();
}

// Định nghĩa policy
services.AddAuthorization(options =>
{
    options.AddPolicy("DeleteUserPermission", policy =>
        policy.Requirements.Add(new PermissionRequirement("delete:users")));
});
```

## Tóm tắt

Phần này đã mô tả chi tiết về phạm vi (scopes) và quyền hạn (claims):

1. **Scopes:** Định nghĩa, các loại scopes, cấp phát và kiểm tra scopes
2. **Claims:** Các loại claims, claims mapping, claims trong token
3. **Ví dụ code:** Định nghĩa scopes và claims trong Duende IdentityServer, kiểm tra scopes và claims trong Resource Server

Phần tiếp theo sẽ đi sâu vào các biện pháp bảo mật tối ưu.
