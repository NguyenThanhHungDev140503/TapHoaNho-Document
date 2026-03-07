# Phần 4: Quản lý Token

## 4.1 Access Token

### 4.1.1 Định dạng JWT

#### JWT là gì?

JWT (JSON Web Token) là một chuẩn mở (RFC 7519) định nghĩa một cách compact và self-contained để truyền thông tin an toàn giữa các parties dưới dạng JSON object. JWT được sử dụng rộng rãi trong OAuth 2.0 và OpenID Connect để đại diện cho access token và ID token.

#### Cấu trúc JWT

JWT bao gồm ba phần, phân cách bằng dấu chấm (.):

```
header.payload.signature
```

##### 1. Header

Header chứa thông tin về cách token được ký và mã hóa.

**Ví dụ:**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-123"
}
```

**Các trường phổ biến:**
- `alg`: Thuật toán ký (RS256, RS512, ES256, HS256)
- `typ`: Loại token (thường là "JWT")
- `kid`: Key ID (để xác định key nào được sử dụng để ký)

##### 2. Payload

Payload chứa các claims (thông tin về user và token).

**Ví dụ:**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "role": "admin",
  "scope": "openid profile api1.read api1.write",
  "client_id": "web-client",
  "iss": "https://auth.example.com",
  "aud": "api1",
  "exp": 1516239022,
  "iat": 1516235422,
  "auth_time": 1516235422,
  "jti": "unique-token-id",
  "nbf": 1516235422
}
```

**Các claims chuẩn:**

**Registered Claims:**
- `iss` (Issuer): Người phát hành token
- `sub` (Subject): Subject của token (thường là user ID)
- `aud` (Audience): Audience được phép sử dụng token
- `exp` (Expiration Time): Thời gian hết hạn
- `nbf` (Not Before): Token không hợp lệ trước thời gian này
- `iat` (Issued At): Thời gian phát hành
- `jti` (JWT ID): ID duy nhất của token

**Public Claims:**
- `name`: Tên đầy đủ
- `email`: Địa chỉ email
- `picture`: URL ảnh đại diện
- `given_name`: Tên
- `family_name`: Họ
- `locale`: Ngôn ngữ
- `preferred_username`: Username ưa thích

**Private Claims:**
- Tùy chỉnh theo ứng dụng
- Ví dụ: `role`, `permissions`, `department`

##### 3. Signature

Signature được sử dụng để verify rằng token không bị thay đổi trong quá trình truyền.

**Ví dụ:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

**Đối với RS256:**
```
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  privateKey
)
```

#### Các loại JWT

##### 1. Signed JWT (JWS)

**Đặc điểm:**
- Chỉ ký, không mã hóa
- Bất kỳ ai cũng có thể đọc payload
- Sử dụng để verify tính toàn vẹn

**Khi nào sử dụng:**
- Access token
- ID token
- Khi payload không chứa thông tin nhạy cảm

##### 2. Encrypted JWT (JWE)

**Đặc điểm:**
- Mã hóa payload
- Chỉ người có key mới có thể đọc
- Bảo mật thông tin nhạy cảm

**Khi nào sử dụng:**
- Khi payload chứa thông tin nhạy cảm
- Khi cần bảo mật cao hơn

#### Thuật toán ký

##### 1. HS256 (HMAC SHA-256)

**Đặc điểm:**
- Sử dụng shared secret
- Đơn giản và nhanh
- Cần bảo mật secret

**Khi nào sử dụng:**
- Khi có thể bảo mật shared secret
- Khi cần performance cao

**Ví dụ:**
```csharp
var tokenDescriptor = new SecurityTokenDescriptor
{
    Subject = "1234567890",
    Expires = DateTime.UtcNow.AddHours(1),
    SigningCredentials = new SigningCredentials(
        new SymmetricSecurityKey(Encoding.UTF8.GetBytes("your-secret-key-that-is-at-least-32-bytes-long")),
        SecurityAlgorithms.HmacSha256Signature)
};

var tokenHandler = new JwtSecurityTokenHandler();
var token = tokenHandler.CreateToken(tokenDescriptor);
var tokenString = tokenHandler.WriteToken(token);
```

##### 2. RS256 (RSA SHA-256)

**Đặc điểm:**
- Sử dụng asymmetric key pair (private/public)
- Private key để ký, public key để verify
- An toàn hơn cho distributed systems

**Khi nào sử dụng:**
- Khi không thể chia sẻ secret
- Khi cần verify token ở nhiều nơi
- Khi cần key rotation

**Ví dụ:**
```csharp
var rsa = RSA.Create(2048);
var privateKey = new RsaSecurityKey(rsa);
var publicKey = new RsaSecurityKey(rsa.ExportParameters(false));

var tokenDescriptor = new SecurityTokenDescriptor
{
    Subject = "1234567890",
    Expires = DateTime.UtcNow.AddHours(1),
    SigningCredentials = new SigningCredentials(privateKey, SecurityAlgorithms.RsaSha256Signature)
};

var tokenHandler = new JwtSecurityTokenHandler();
var token = tokenHandler.CreateToken(tokenDescriptor);
var tokenString = tokenHandler.WriteToken(token);
```

##### 3. ES256 (ECDSA P-256)

**Đặc điểm:**
- Sử dụng elliptic curve cryptography
- Key ngắn hơn RSA
- Performance tốt hơn RSA

**Khi nào sử dụng:**
- Khi cần performance cao
- Khi muốn key ngắn hơn

### 4.1.2 Claims chuẩn và tùy chỉnh

#### Claims chuẩn (Standard Claims)

##### 1. Issuer (iss)

**Mô tả:** Người phát hành token

**Ví dụ:**
```json
"iss": "https://auth.example.com"
```

**Validation:**
```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidIssuers = new List<string> { "https://auth.example.com" }
};
```

##### 2. Subject (sub)

**Mô tả:** Subject của token (thường là user ID)

**Ví dụ:**
```json
"sub": "1234567890"
```

**Điểm quan trọng:**
- Phải là duy nhất và ổn định
- Không nên thay đổi trong vòng đời của user
- Có thể sử dụng UUID hoặc database ID

##### 3. Audience (aud)

**Mô tả:** Audience được phép sử dụng token

**Ví dụ:**
```json
"aud": "api1"
```

**Validation:**
```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateAudience = true,
    ValidAudiences = new List<string> { "api1" }
};
```

##### 4. Expiration Time (exp)

**Mô tả:** Thời gian hết hạn của token (Unix timestamp)

**Ví dụ:**
```json
"exp": 1516239022
```

**Điểm quan trọng:**
- Phải là số (không phải string)
- Được tính bằng giây từ epoch
- Token hết hạn sẽ bị từ chối

##### 5. Issued At (iat)

**Mô tả:** Thời gian phát hành token (Unix timestamp)

**Ví dụ:**
```json
"iat": 1516235422
```

##### 6. Not Before (nbf)

**Mô tả:** Token không hợp lệ trước thời gian này (Unix timestamp)

**Ví dụ:**
```json
"nbf": 1516235422
```

##### 7. JWT ID (jti)

**Mô tả:** ID duy nhất của token

**Ví dụ:**
```json
"jti": "unique-token-id-123456"
```

**Điểm quan trọng:**
- Phải là duy nhất
- Có thể được sử dụng để thu hồi token
- Hữu ích cho tracking và debugging

#### Claims tùy chỉnh (Custom Claims)

##### 1. Role Claim

**Mô tả:** Vai trò của user

**Ví dụ:**
```json
"role": "admin"
```

**Triển khai:**
```csharp
var claims = new List<Claim>
{
    new Claim(JwtRegisteredClaimNames.Sub, userId),
    new Claim("role", "admin"),
    new Claim("permission", "read:users"),
    new Claim("permission", "write:users")
};

var tokenDescriptor = new SecurityTokenDescriptor
{
    Subject = new ClaimsIdentity(claims),
    Expires = DateTime.UtcNow.AddHours(1),
    SigningCredentials = signingCredentials
};
```

##### 2. Permission Claim

**Mô tả:** Quyền hạn cụ thể của user

**Ví dụ:**
```json
"permissions": ["read:users", "write:users", "delete:users"]
```

**Triển khai:**
```csharp
var permissions = new List<string> { "read:users", "write:users", "delete:users" };
var claims = new List<Claim>
{
    new Claim(JwtRegisteredClaimNames.Sub, userId),
    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
};

foreach (var permission in permissions)
{
    claims.Add(new Claim("permission", permission));
}
```

##### 3. Department Claim

**Mô tả:** Phòng ban của user

**Ví dụ:**
```json
"department": "engineering"
```

##### 4. Tenant Claim (cho multi-tenant)

**Mô tả:** Tenant ID của user

**Ví dụ:**
```json
"tenant_id": "tenant-123"
```

**Validation:**
```csharp
public class TenantRequirement : IAuthorizationRequirement
{
    public string TenantId { get; }

    public TenantRequirement(string tenantId)
    {
        TenantId = tenantId;
    }
}

public class TenantHandler : AuthorizationHandler<TenantRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        TenantRequirement requirement)
    {
        var tenantClaim = context.User.FindFirst("tenant_id");
        
        if (tenantClaim == null || tenantClaim.Value != requirement.TenantId)
        {
            context.Fail();
            return Task.CompletedTask;
        }
        
        context.Succeed(requirement);
        return Task.CompletedTask;
    }
}
```

#### Claims Mapping

##### 1. Mapping từ User Profile

```csharp
public class ClaimsService : IClaimsService
{
    public IEnumerable<Claim> GetClaimsFromUser(ApplicationUser user)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim("role", user.Role),
            new Claim("tenant_id", user.TenantId)
        };
        
        // Add permissions
        foreach (var permission in user.Permissions)
        {
            claims.Add(new Claim("permission", permission.Name));
        }
        
        return claims;
    }
}
```

##### 2. Custom Claims Provider (Duende IdentityServer)

```csharp
public class CustomProfileService : IProfileService
{
    private readonly UserManager<ApplicationUser> _userManager;
    
    public CustomProfileService(UserManager<ApplicationUser> userManager)
    {
        _userManager = userManager;
    }
    
    public async Task GetProfileDataAsync(ProfileDataRequest context)
    {
        var user = await _userManager.GetUserAsync(context.Subject.GetSubjectId());
        
        if (user == null)
        {
            throw new InvalidOperationException("User not found");
        }
        
        context.IssuedClaims.Add(new Claim("role", user.Role));
        context.IssuedClaims.Add(new Claim("tenant_id", user.TenantId));
        
        foreach (var permission in user.Permissions)
        {
            context.IssuedClaims.Add(new Claim("permission", permission.Name));
        }
        
        await Task.CompletedTask;
    }
}
```

### 4.1.3 Vòng đời và expiration

#### Vòng đời Access Token

##### 1. Access Token Lifetime

**Khuyến nghị:**
- **Short-lived**: 5-60 phút
- **Mặc định**: 30 phút
- **Tối đa**: 1 giờ

**Tại sao ngắn?**
- Giảm thiệt hại nếu token bị lộ
- Giảm nguy cơ token replay attacks
- Cho phép revoke token dễ dàng hơn

##### 2. Cấu hình Access Token Lifetime

**Duende IdentityServer:**
```csharp
new Client
{
    ClientId = "web-client",
    AccessTokenLifetime = 3600, // 60 phút
}
```

**ASP.NET Core JWT:**
```csharp
services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ClockSkew = TimeSpan.FromMinutes(5) // Cho phép lệch thời gian 5 phút
    };
});
```

#### Token Expiration Handling

##### 1. Client-side Handling

**Kiểm tra expiration:**
```javascript
function isTokenExpired(token) {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const exp = payload.exp;
    const now = Math.floor(Date.now() / 1000);
    return now >= exp;
}

// Sử dụng
if (isTokenExpired(accessToken)) {
    // Refresh token
    await refreshAccessToken();
}
```

**Auto-refresh trước khi hết hạn:**
```javascript
function shouldRefreshToken(token, bufferSeconds = 300) {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const exp = payload.exp;
    const now = Math.floor(Date.now() / 1000);
    return (exp - now) <= bufferSeconds;
}

// Sử dụng
if (shouldRefreshToken(accessToken, 300)) {
    // Refresh token 5 phút trước khi hết hạn
    await refreshAccessToken();
}
```

##### 2. Server-side Handling

**ASP.NET Core:**
```csharp
[ApiController]
[Authorize]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetProducts()
    {
        var accessToken = await HttpContext.GetTokenAsync("access_token");
        
        if (string.IsNullOrEmpty(accessToken))
        {
            return Unauthorized(new { error = "Access token is required" });
        }
        
        // Kiểm tra expiration
        var tokenHandler = new JwtSecurityTokenHandler();
        var jsonToken = tokenHandler.ReadJwtToken(accessToken);
        
        if (jsonToken.ValidTo < DateTime.UtcNow)
        {
            return Unauthorized(new { error = "Access token has expired" });
        }
        
        // Tiếp tục xử lý
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }
}
```

#### Token Revocation

##### 1. Refresh Token Revocation

**Thu hồi refresh token:**
```csharp
public class TokenRevocationService : ITokenRevocationService
{
    private readonly IPersistedGrantStore _persistedGrantStore;
    
    public async Task RevokeRefreshTokenAsync(string refreshToken)
    {
        var grant = await _persistedGrantStore.GetAsync(refreshToken);
        
        if (grant != null)
        {
            await _persistedGrantStore.RemoveAsync(refreshToken);
        }
    }
}
```

##### 2. Access Token Revocation (Reference Tokens)

**Sử dụng reference tokens thay vì JWT:**
```csharp
services.AddIdentityServer()
    .AddInMemoryClients(Clients.Get())
    .AddInMemoryApiResources(Resources.GetApiResources())
    .AddTestUsers(TestUsers.Users)
    .AddDeveloperSigningCredential()
    // Sử dụng reference tokens
    .AddReferenceTokenStore<InMemoryReferenceTokenStore>();
```

**Thu hồi reference token:**
```csharp
public class TokenRevocationService : ITokenRevocationService
{
    private readonly IReferenceTokenStore _referenceTokenStore;
    
    public async Task RevokeAccessTokenAsync(string accessToken)
    {
        await _referenceTokenStore.RemoveReferenceAsync(accessToken);
    }
}
```

##### 3. Token Introspection

**Kiểm tra trạng thái token:**
```csharp
[HttpPost("introspect")]
public async Task<IActionResult> IntrospectToken([FromBody] TokenIntrospectionRequest request)
{
    var token = request.Token;
    var tokenHandler = new JwtSecurityTokenHandler();
    
    try
    {
        var jsonToken = tokenHandler.ReadJwtToken(token);
        
        // Kiểm tra xem token có trong danh sách revoke không
        var isRevoked = await _tokenRevocationService.IsTokenRevokedAsync(token);
        
        return Ok(new TokenIntrospectionResponse
        {
            Active = !isRevoked && jsonToken.ValidTo > DateTime.UtcNow,
            Sub = jsonToken.Subject,
            ClientId = jsonToken.Claims.FirstOrDefault(c => c.Type == "client_id")?.Value,
            Scope = jsonToken.Claims.FirstOrDefault(c => c.Type == "scope")?.Value,
            Exp = jsonToken.ValidTo?.ToUnixTimeSeconds()
        });
    }
    catch (Exception)
    {
        return Ok(new TokenIntrospectionResponse
        {
            Active = false
        });
    }
}
```

## 4.2 Refresh Token

### 4.2.1 Mục đích sử dụng

#### Tại sao cần Refresh Token?

##### 1. Giảm friction cho người dùng

**Vấn đề:** Nếu access token hết hạn, người dùng phải đăng nhập lại.

**Giải pháp:** Sử dụng refresh token để lấy access token mới mà không cần đăng nhập lại.

##### 2. Access token ngắn-lived

**Vấn đề:** Access token nên ngắn-lived để bảo mật, nhưng gây bất tiện cho người dùng.

**Giải pháp:** Sử dụng refresh token dài-lived để lấy access token mới.

##### 3. Thu hồi quyền truy cập

**Vấn đề:** Nếu access token dài-lived, khó thu hồi quyền truy cập khi token còn hiệu lực.

**Giải pháp:** Access token ngắn-lived, refresh token có thể được thu hồi.

#### Khi nào sử dụng Refresh Token?

- Khi cần long-lived sessions
- Khi muốn giảm friction cho người dùng
- Khi sử dụng authorization code flow hoặc resource owner password credentials grant
- Khi cần thu hồi quyền truy cập dễ dàng

### 4.2.2 Quản lý và rotation

#### Refresh Token Lifetime

**Khuyến nghị:**
- **Mặc định**: 7-30 ngày
- **Tối đa**: 90 ngày
- **Tối thiểu**: 1 năm

**Cấu hình:**
```csharp
new Client
{
    ClientId = "web-client",
    RefreshTokenUsage = TokenUsage.ReUse, // hoặc TokenUsage.OneTimeOnly
    RefreshTokenExpiration = TimeSpan.FromDays(30),
    AllowOfflineAccess = true
}
```

#### Refresh Token Rotation

##### 1. One-time Refresh Tokens

**Mô tả:** Refresh token chỉ được sử dụng một lần. Sau khi sử dụng, một refresh token mới được phát hành.

**Ưu điểm:**
- An toàn hơn
- Giảm nguy cơ token theft
- Dễ dàng thu hồi token

**Nhược điểm:**
- Cần lưu trữ nhiều refresh tokens
- Phức tạp hơn

**Triển khai:**
```csharp
new Client
{
    ClientId = "web-client",
    RefreshTokenUsage = TokenUsage.OneTimeOnly,
    RefreshTokenExpiration = TimeSpan.FromDays(30)
}
```

##### 2. Re-use Refresh Tokens

**Mô tả:** Refresh token có thể được sử dụng nhiều lần.

**Ưu điểm:**
- Đơn giản hơn
- Ít storage

**Nhược điểm:**
- Ít an toàn hơn
- Khó thu hồi token

**Triển khai:**
```csharp
new Client
{
    ClientId = "web-client",
    RefreshTokenUsage = TokenUsage.ReUse,
    RefreshTokenExpiration = TimeSpan.FromDays(30)
}
```

#### Refresh Token Security

##### 1. Storage

**Server-side storage:**
- Lưu trữ trong database
- Lưu trữ trong Redis (cho performance)
- Lưu trữ trong distributed cache

**Ví dụ:**
```csharp
public class RefreshToken
{
    public string Token { get; set; }
    public string ClientId { get; set; }
    public string UserId { get; set; }
    public DateTime CreationTime { get; set; }
    public DateTime ExpirationTime { get; set; }
    public bool IsRevoked { get; set; }
    public string DeviceId { get; set; }
    public string IpAddress { get; set; }
}
```

##### 2. Validation

**Kiểm tra khi sử dụng:**
```csharp
public async Task<RefreshToken> ValidateRefreshTokenAsync(string refreshToken, string clientId)
{
    var token = await _refreshTokenStore.GetAsync(refreshToken);
    
    if (token == null)
    {
        throw new InvalidOperationException("Refresh token not found");
    }
    
    if (token.ClientId != clientId)
    {
        throw new InvalidOperationException("Refresh token does not belong to this client");
    }
    
    if (token.IsRevoked)
    {
        throw new InvalidOperationException("Refresh token has been revoked");
    }
    
    if (token.ExpirationTime < DateTime.UtcNow)
    {
        throw new InvalidOperationException("Refresh token has expired");
    }
    
    return token;
}
```

##### 3. Device Binding

**Gắn refresh token với device:**
```csharp
public async Task<string> CreateRefreshTokenAsync(string userId, string clientId, string deviceId)
{
    var refreshToken = new RefreshToken
    {
        Token = GenerateSecureToken(),
        ClientId = clientId,
        UserId = userId,
        CreationTime = DateTime.UtcNow,
        ExpirationTime = DateTime.UtcNow.AddDays(30),
        IsRevoked = false,
        DeviceId = deviceId,
        IpAddress = GetClientIpAddress()
    };
    
    await _refreshTokenStore.AddAsync(refreshToken);
    return refreshToken.Token;
}
```

#### Refresh Token Revocation

##### 1. Thu hồi tất cả refresh tokens của user

```csharp
public async Task RevokeAllUserRefreshTokensAsync(string userId)
{
    var tokens = await _refreshTokenStore.GetByUserIdAsync(userId);
    
    foreach (var token in tokens)
    {
        token.IsRevoked = true;
        await _refreshTokenStore.UpdateAsync(token);
    }
}
```

##### 2. Thu hồi refresh token của một device

```csharp
public async Task RevokeDeviceRefreshTokensAsync(string userId, string deviceId)
{
    var tokens = await _refreshTokenStore.GetByUserIdAndDeviceIdAsync(userId, deviceId);
    
    foreach (var token in tokens)
    {
        token.IsRevoked = true;
        await _refreshTokenStore.UpdateAsync(token);
    }
}
```

##### 3. Thu hồi refresh token khi password thay đổi

```csharp
public async Task OnPasswordChangedAsync(string userId)
{
    // Thu hồi tất cả refresh tokens
    await RevokeAllUserRefreshTokensAsync(userId);
    
    // Gửi notification cho tất cả devices
    await _notificationService.SendPasswordChangedNotificationAsync(userId);
}
```

### 4.2.3 Ví dụ code refresh

#### Client-side (ASP.NET Core)

##### 1. Refresh Access Token

```csharp
public class TokenService : ITokenService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IConfiguration _configuration;
    
    public TokenService(IHttpClientFactory httpClientFactory, IConfiguration configuration)
    {
        _httpClientFactory = httpClientFactory;
        _configuration = configuration;
    }
    
    public async Task<string> GetAccessTokenAsync()
    {
        var accessToken = _tokenCache.GetAccessToken();
        
        if (!string.IsNullOrEmpty(accessToken) && !IsTokenExpired(accessToken))
        {
            return accessToken;
        }
        
        // Refresh token
        var refreshToken = _tokenCache.GetRefreshToken();
        
        if (string.IsNullOrEmpty(refreshToken))
        {
            throw new InvalidOperationException("Refresh token is required");
        }
        
        var clientId = _configuration["OAuth:ClientId"];
        var clientSecret = _configuration["OAuth:ClientSecret"];
        var tokenEndpoint = _configuration["OAuth:TokenEndpoint"];
        
        var client = _httpClientFactory.CreateClient();
        var parameters = new Dictionary<string, string>
        {
            { "grant_type", "refresh_token" },
            { "refresh_token", refreshToken },
            { "client_id", clientId },
            { "client_secret", clientSecret }
        };
        
        var content = new FormUrlEncodedContent(parameters);
        var response = await client.PostAsync(tokenEndpoint, content);
        
        if (!response.IsSuccessStatusCode)
        {
            // Refresh token hết hạn hoặc bị thu hồi
            _tokenCache.ClearTokens();
            throw new UnauthorizedException("Session expired. Please login again.");
        }
        
        var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>();
        
        // Lưu tokens mới
        _tokenCache.SetAccessToken(tokenResponse.AccessToken);
        _tokenCache.SetRefreshToken(tokenResponse.RefreshToken);
        
        return tokenResponse.AccessToken;
    }
    
    private bool IsTokenExpired(string token)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var jsonToken = tokenHandler.ReadJwtToken(token);
        return jsonToken.ValidTo < DateTime.UtcNow;
    }
}

public class TokenResponse
{
    [JsonPropertyName("access_token")]
    public string AccessToken { get; set; }
    
    [JsonPropertyName("refresh_token")]
    public string RefreshToken { get; set; }
    
    [JsonPropertyName("token_type")]
    public string TokenType { get; set; }
    
    [JsonPropertyName("expires_in")]
    public int ExpiresIn { get; set; }
    
    [JsonPropertyName("scope")]
    public string Scope { get; set; }
}
```

##### 2. HTTP Client với Auto-refresh

```csharp
public class AuthenticatedHttpClient : DelegatingHandler
{
    private readonly ITokenService _tokenService;
    
    public AuthenticatedHttpClient(ITokenService tokenService)
    {
        _tokenService = tokenService;
    }
    
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        var accessToken = await _tokenService.GetAccessTokenAsync();
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
        
        var response = await base.SendAsync(request, cancellationToken);
        
        if (response.StatusCode == HttpStatusCode.Unauthorized)
        {
            // Try refresh token and retry
            accessToken = await _tokenService.GetAccessTokenAsync(true); // Force refresh
            request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
            response = await base.SendAsync(request, cancellationToken);
        }
        
        return response;
    }
}

// Sử dụng
services.AddHttpClient("api", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
})
.AddHttpMessageHandler<AuthenticatedHttpClient>();
```

#### Authorization Server-side (Duende IdentityServer)

##### 1. Custom Refresh Token Service

```csharp
public class CustomRefreshTokenService : IRefreshTokenService
{
    private readonly IRefreshTokenStore _refreshTokenStore;
    private readonly ILogger<CustomRefreshTokenService> _logger;
    
    public CustomRefreshTokenService(
        IRefreshTokenStore refreshTokenStore,
        ILogger<CustomRefreshTokenService> logger)
    {
        _refreshTokenStore = refreshTokenStore;
        _logger = logger;
    }
    
    public async Task<string> CreateRefreshTokenAsync(RefreshTokenCreationContext context)
    {
        var refreshToken = new RefreshToken
        {
            Token = GenerateSecureToken(),
            ClientId = context.Client.ClientId,
            Subject = context.Subject.GetSubjectId(),
            CreationTime = DateTime.UtcNow,
            ExpirationTime = DateTime.UtcNow.AddDays(30),
            IsRevoked = false,
            DeviceId = context.DeviceId,
            IpAddress = context.IpAddress
        };
        
        await _refreshTokenStore.AddAsync(refreshToken);
        
        _logger.LogInformation("Created refresh token for user {UserId}, client {ClientId}", 
            refreshToken.Subject, refreshToken.ClientId);
        
        return refreshToken.Token;
    }
    
    public async Task ValidateRefreshTokenAsync(RefreshTokenValidationContext context)
    {
        var token = await _refreshTokenStore.GetAsync(context.RefreshToken);
        
        if (token == null)
        {
            _logger.LogWarning("Refresh token not found for client {ClientId}", 
                context.Client.ClientId);
            context.Result = new TokenValidationResult(new TokenError(OidcConstants.TokenErrors.InvalidGrant));
            return;
        }
        
        if (token.ClientId != context.Client.ClientId)
        {
            _logger.LogWarning("Refresh token does not belong to client {ClientId}", 
                context.Client.ClientId);
            context.Result = new TokenValidationResult(new TokenError(OidcConstants.TokenErrors.InvalidGrant));
            return;
        }
        
        if (token.Subject != context.Subject.GetSubjectId())
        {
            _logger.LogWarning("Refresh token does not belong to user {UserId}", 
                context.Subject.GetSubjectId());
            context.Result = new TokenValidationResult(new TokenError(OidcConstants.TokenErrors.InvalidGrant));
            return;
        }
        
        if (token.IsRevoked)
        {
            _logger.LogWarning("Refresh token has been revoked for user {UserId}", 
                token.Subject);
            context.Result = new TokenValidationResult(new TokenError(OidcConstants.TokenErrors.InvalidGrant));
            return;
        }
        
        if (token.ExpirationTime < DateTime.UtcNow)
        {
            _logger.LogWarning("Refresh token has expired for user {UserId}", 
                token.Subject);
            context.Result = new TokenValidationResult(new TokenError(OidcConstants.TokenErrors.InvalidGrant));
            return;
        }
        
        // Kiểm tra device binding
        if (!string.IsNullOrEmpty(token.DeviceId) && 
            token.DeviceId != context.DeviceId)
        {
            _logger.LogWarning("Refresh token device mismatch for user {UserId}", 
                token.Subject);
            context.Result = new TokenValidationResult(new TokenError(OidcConstants.TokenErrors.InvalidGrant));
            return;
        }
        
        context.Result = new TokenValidationResult(token.Subject);
    }
    
    public async Task UpdateRefreshTokenAsync(RefreshTokenUpdateContext context)
    {
        var oldToken = await _refreshTokenStore.GetAsync(context.RefreshToken);
        
        if (oldToken != null)
        {
            await _refreshTokenStore.RemoveAsync(context.RefreshToken);
        }
        
        var newToken = await CreateRefreshTokenAsync(context);
        context.Result = new TokenUpdateResult(newToken);
    }
    
    private string GenerateSecureToken()
    {
        var randomBytes = new byte[32];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(randomBytes);
        }
        return Base64UrlEncode(randomBytes);
    }
    
    private string Base64UrlEncode(byte[] input)
    {
        return Convert.ToBase64String(input)
            .Replace('+', '-')
            .Replace('/', '_')
            .TrimEnd('=');
    }
}
```

##### 2. Đăng ký Custom Refresh Token Service

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddTransient<IRefreshTokenService, CustomRefreshTokenService>();
    
    services.AddIdentityServer()
        .AddInMemoryClients(Clients.Get())
        .AddInMemoryApiResources(Resources.GetApiResources())
        .AddTestUsers(TestUsers.Users)
        .AddDeveloperSigningCredential();
}
```

## 4.3 Token Storage

### 4.3.1 Client-side storage

#### Browser Storage Options

##### 1. LocalStorage

**Đặc điểm:**
- Lưu trữ vĩnh viễn (không hết hạn khi đóng browser)
- Dễ truy cập từ JavaScript
- Không an toàn cho tokens

**Khi nào sử dụng:**
- Không nên sử dụng cho access tokens
- Có thể sử dụng cho refresh tokens (với các biện pháp bảo mật bổ sung)

**Ví dụ:**
```javascript
// Lưu access token
localStorage.setItem('access_token', accessToken);

// Lấy access token
const accessToken = localStorage.getItem('access_token');

// Xóa access token
localStorage.removeItem('access_token');
```

##### 2. SessionStorage

**Đặc điểm:**
- Lưu trữ trong session (hết hạn khi đóng tab)
- Dễ truy cập từ JavaScript
- Ít an toàn hơn localStorage

**Khi nào sử dụng:**
- Có thể sử dụng cho access tokens
- Không nên sử dụng cho refresh tokens

**Ví dụ:**
```javascript
// Lưu access token
sessionStorage.setItem('access_token', accessToken);

// Lấy access token
const accessToken = sessionStorage.getItem('access_token');

// Xóa access token
sessionStorage.removeItem('access_token');
```

##### 3. Cookies

**Đặc điểm:**
- Có thể được gửi tự động với mỗi request
- Có thể được bảo vệ bởi HttpOnly và Secure flags
- An toàn hơn localStorage và sessionStorage

**Khi nào sử dụng:**
- Khuyến nghị sử dụng cho access tokens
- Có thể sử dụng cho refresh tokens (với các biện pháp bảo mật bổ sung)

**Ví dụ:**
```javascript
// Set cookie
document.cookie = `access_token=${accessToken}; path=/; Secure; HttpOnly; SameSite=Strict`;

// Get cookie (không thể truy cập HttpOnly cookies từ JavaScript)
// Phải gửi từ server
```

#### Mobile App Storage

##### 1. Keychain/Keystore (iOS)

**Đặc điểm:**
- An toàn nhất
- Lưu trữ encrypted
- Hệ điều hành quản lý

**Ví dụ (Swift):**
```swift
let token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

// Lưu vào keychain
let query: [String: Any] = [
    kSecClass as String,
    kSecAttrAccount as String,
    "access_token",
    kSecValueData as String,
    token.data(using: .utf8)!
]
let status = SecItemAdd(query as CFDictionary, nil)

// Lấy từ keychain
let query: [String: Any] = [
    kSecClass as String,
    kSecAttrAccount as String,
    "access_token",
    kSecReturnData as String,
    true
]
var result: AnyObject?
let status = SecItemCopyMatching(query as CFDictionary, &result)

if let data = result as? Data {
    let token = String(data: String, encoding: .utf8)
}
```

##### 2. Encrypted SharedPreferences (Android)

**Đặc điểm:**
- An toàn hơn SharedPreferences thường
- Lưu trữ encrypted
- App quản lý encryption

**Ví dụ (Kotlin):**
```kotlin
val token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

// Lưu vào encrypted SharedPreferences
val sharedPreferences = getSharedPreferences("auth_prefs", Context.MODE_PRIVATE)
val editor = sharedPreferences.edit()
editor.putString("access_token", encrypt(token))
editor.apply()

// Lấy từ encrypted SharedPreferences
val encryptedToken = sharedPreferences.getString("access_token", null)
val token = decrypt(encryptedToken)
```

### 4.3.2 Server-side storage

#### Database Storage

##### 1. SQL Database

**Ví dụ schema:**
```sql
CREATE TABLE refresh_tokens (
    id INT PRIMARY KEY AUTO_INCREMENT,
    token VARCHAR(255) UNIQUE NOT NULL,
    client_id VARCHAR(255) NOT NULL,
    user_id VARCHAR(255) NOT NULL,
    creation_time DATETIME NOT NULL,
    expiration_time DATETIME NOT NULL,
    is_revoked BOOLEAN DEFAULT FALSE,
    device_id VARCHAR(255),
    ip_address VARCHAR(45),
    INDEX idx_token (token),
    INDEX idx_user_id (user_id),
    INDEX idx_client_id (client_id)
);
```

**Ví dụ implementation:**
```csharp
public class RefreshTokenRepository : IRefreshTokenRepository
{
    private readonly ApplicationDbContext _context;
    
    public RefreshTokenRepository(ApplicationDbContext context)
    {
        _context = context;
    }
    
    public async Task AddAsync(RefreshToken token)
    {
        _context.RefreshTokens.Add(token);
        await _context.SaveChangesAsync();
    }
    
    public async Task<RefreshToken> GetAsync(string token)
    {
        return await _context.RefreshTokens
            .FirstOrDefaultAsync(t => t.Token == token);
    }
    
    public async Task<List<RefreshToken>> GetByUserIdAsync(string userId)
    {
        return await _context.RefreshTokens
            .Where(t => t.UserId == userId && !t.IsRevoked)
            .ToListAsync();
    }
    
    public async Task RemoveAsync(string token)
    {
        var refreshToken = await GetAsync(token);
        if (refreshToken != null)
        {
            _context.RefreshTokens.Remove(refreshToken);
            await _context.SaveChangesAsync();
        }
    }
    
    public async Task RevokeAllForUserAsync(string userId)
    {
        var tokens = await GetByUserIdAsync(userId);
        foreach (var token in tokens)
        {
            token.IsRevoked = true;
        }
        await _context.SaveChangesAsync();
    }
}
```

##### 2. NoSQL Database

**Ví dụ (MongoDB):**
```csharp
public class RefreshToken
{
    [BsonId]
    public string Id { get; set; }
    
    [BsonElement("token")]
    public string Token { get; set; }
    
    [BsonElement("clientId")]
    public string ClientId { get; set; }
    
    [BsonElement("userId")]
    public string UserId { get; set; }
    
    [BsonElement("creationTime")]
    public DateTime CreationTime { get; set; }
    
    [BsonElement("expirationTime")]
    public DateTime ExpirationTime { get; set; }
    
    [BsonElement("isRevoked")]
    public bool IsRevoked { get; set; }
    
    [BsonElement("deviceId")]
    public string DeviceId { get; set; }
    
    [BsonElement("ipAddress")]
    public string IpAddress { get; set; }
}

public class RefreshTokenRepository : IRefreshTokenRepository
{
    private readonly IMongoCollection<RefreshToken> _collection;
    
    public RefreshTokenRepository(IMongoDatabase database)
    {
        _collection = database.GetCollection<RefreshToken>("refresh_tokens");
    }
    
    public async Task AddAsync(RefreshToken token)
    {
        await _collection.InsertOneAsync(token);
    }
    
    public async Task<RefreshToken> GetAsync(string token)
    {
        return await _collection.Find(t => t.Token == token).FirstOrDefaultAsync();
    }
    
    public async Task<List<RefreshToken>> GetByUserIdAsync(string userId)
    {
        return await _collection.Find(t => t.UserId == userId && !t.IsRevoked).ToListAsync();
    }
    
    public async Task RemoveAsync(string token)
    {
        await _collection.DeleteOneAsync(t => t.Token == token);
    }
    
    public async Task RevokeAllForUserAsync(string userId)
    {
        var filter = Builders<RefreshToken>.Filter.Eq(t => t.UserId, userId);
        var update = Builders<RefreshToken>.Update.Set(t => t.IsRevoked, true);
        await _collection.UpdateManyAsync(filter, update);
    }
}
```

#### Cache Storage

##### 1. Redis

**Ưu điểm:**
- Performance cao
- Hỗ trợ expiration tự động
- Distributed cache

**Ví dụ:**
```csharp
public class RefreshTokenCache : IRefreshTokenCache
{
    private readonly IDatabase _database;
    private readonly ILogger<RefreshTokenCache> _logger;
    
    public RefreshTokenCache(IConnectionMultiplexer redis, ILogger<RefreshTokenCache> logger)
    {
        _database = redis.GetDatabase();
        _logger = logger;
    }
    
    public async Task SetAsync(string key, RefreshToken token, TimeSpan expiration)
    {
        var json = JsonSerializer.Serialize(token);
        await _database.StringSetAsync(key, json, expiration);
        _logger.LogInformation("Stored refresh token in cache with key {Key}", key);
    }
    
    public async Task<RefreshToken> GetAsync(string key)
    {
        var json = await _database.StringGetAsync(key);
        
        if (string.IsNullOrEmpty(json))
        {
            return null;
        }
        
        return JsonSerializer.Deserialize<RefreshToken>(json);
    }
    
    public async Task RemoveAsync(string key)
    {
        await _database.KeyDeleteAsync(key);
        _logger.LogInformation("Removed refresh token from cache with key {Key}", key);
    }
    
    public async Task RevokeAllForUserAsync(string userId)
    {
        var pattern = $"refresh_token:{userId}:*";
        var endpoints = _database.Multiplexer.GetServer(_database.Database).Endpoints;
        
        foreach (var endpoint in endpoints)
        {
            var server = _database.Multiplexer.GetServer(endpoint);
            var db = server.GetDatabase(_database.Database);
            var keys = db.Multiplexer.GetServer(endpoint).Keys(pattern, _database.Database).ToList();
            
            foreach (var key in keys)
            {
                await db.KeyDeleteAsync(key);
            }
        }
        
        _logger.LogInformation("Revoked all refresh tokens for user {UserId}", userId);
    }
}
```

### 4.3.3 Best Practices

#### Access Token Storage

##### 1. Sử dụng HttpOnly Cookies

**Ưu điểm:**
- Không thể truy cập từ JavaScript
- Giảm nguy cơ XSS attacks
- Được gửi tự động với mỗi request

**Ví dụ:**
```csharp
services.AddAuthentication("Cookies")
    .AddCookie("Cookies", options =>
    {
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Strict;
        options.ExpireTimeSpan = TimeSpan.FromMinutes(30);
    });
```

##### 2. Sử dụng Memory Storage (SPA)

**Ưu điểm:**
- Không lưu trữ persistent
- Hết hạn khi đóng tab
- An toàn hơn localStorage

**Ví dụ:**
```javascript
// Sử dụng in-memory storage
let accessToken = null;
let refreshToken = null;

function setAccessToken(token) {
    accessToken = token;
}

function getAccessToken() {
    return accessToken;
}

function clearAccessToken() {
    accessToken = null;
}
```

##### 3. Không lưu Access Token trong LocalStorage

**Lý do:**
- LocalStorage có thể truy cập từ JavaScript
- Dễ bị lộ qua XSS attacks
- Không an toàn cho tokens

#### Refresh Token Storage

##### 1. Lưu trữ trong HttpOnly Cookies

**Ưu điểm:**
- Không thể truy cập từ JavaScript
- Giảm nguy cơ XSS attacks
- Được gửi tự động với mỗi request

**Ví dụ:**
```csharp
services.AddAuthentication("Cookies")
    .AddCookie("Cookies", options =>
    {
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Strict;
        options.ExpireTimeSpan = TimeSpan.FromDays(30);
    });
```

##### 2. Lưu trữ trong Secure Storage (Mobile Apps)

**Ưu điểm:**
- Encrypted storage
- An toàn hơn
- Hệ điều hành quản lý

##### 3. Server-side Storage

**Ưu điểm:**
- An toàn nhất
- Dễ dàng thu hồi
- Có thể validate device binding

#### Token Rotation

##### 1. Rotate Access Tokens

**Chiến lược:**
- Sử dụng access token ngắn-lived
- Refresh access token trước khi hết hạn
- Luôn sử dụng access token mới nhất

**Ví dụ:**
```javascript
async function getAccessToken() {
    // Kiểm tra xem access token hết hạn trong 5 phút không
    if (shouldRefreshToken(accessToken, 300)) {
        accessToken = await refreshAccessToken();
    }
    
    return accessToken;
}
```

##### 2. Rotate Refresh Tokens

**Chiến lược:**
- Sử dụng one-time refresh tokens
- Phát hành refresh token mới mỗi khi sử dụng
- Thu hồi refresh token cũ

**Ví dụ:**
```csharp
public async Task<string> RefreshAccessTokenAsync(string refreshToken)
{
    var newRefreshToken = await _tokenService.RefreshAccessTokenAsync(refreshToken);
    
    // Lưu refresh token mới
    _tokenCache.SetRefreshToken(newRefreshToken.RefreshToken);
    
    return newRefreshToken.AccessToken;
}
```

## Tóm tắt

Phần này đã mô tả chi tiết quản lý token:

1. **Access Token:** Định dạng JWT, claims chuẩn và tùy chỉnh, vòng đời và expiration
2. **Refresh Token:** Mục đích sử dụng, quản lý và rotation, ví dụ code refresh
3. **Token Storage:** Client-side storage, server-side storage, best practices

Phần tiếp theo sẽ đi sâu vào phạm vi (scopes) và quyền hạn (claims).
