# Phần 2: Kiến trúc hệ thống

## 2.1 Các thành phần chính (Resource Owner, Client, Authorization Server, Resource Server)

### 2.1.1 Tổng quan về kiến trúc OAuth 2.0

Kiến trúc OAuth 2.0 bao gồm bốn thành phần chính tương tác với nhau để cung cấp cơ chế ủy quyền an toàn và hiệu quả. Mỗi thành phần có vai trò và trách nhiệm riêng biệt, nhưng cùng phối hợp để đạt được mục tiêu chung: cho phép ứng dụng truy cập tài nguyên được bảo vệ mà không cần chia sẻ thông tin đăng nhập.

### 2.1.2 Resource Owner (Chủ sở hữu tài nguyên)

#### Định nghĩa

Resource Owner là entity có khả năng cấp quyền truy cập vào tài nguyên được bảo vệ. Trong hầu hết các trường hợp, Resource Owner là người dùng cuối (end-user), nhưng cũng có thể là một ứng dụng khác trong trường hợp service-to-service authentication.

#### Trách nhiệm

1. **Xác thực danh tính**: Đăng nhập vào hệ thống với thông tin đăng nhập
2. **Cấp quyền truy cập**: Chấp nhận hoặc từ chối yêu cầu truy cập từ Client
3. **Quản lý quyền**: Thu hồi hoặc sửa đổi quyền truy cập bất cứ lúc nào

#### Thông tin được lưu trữ

- Thông tin profile: username, email, name, avatar
- Thông tin đăng nhập: password hash, 2FA secrets, device fingerprints
- Thông tin quyền: roles, permissions, groups
- Thông tin session: current sessions, devices, locations

#### Ví dụ thực tế

**Người dùng Facebook:**
- Tài nguyên: Profile, Friends, Photos, Posts
- Quyền: Có thể cho phép ứng dụng truy cập Profile nhưng không Photos

**Người dùng Google:**
- Tài nguyên: Gmail, Calendar, Drive, Contacts
- Quyền: Có thể cho phép ứng dụng truy cập Calendar nhưng không Gmail

### 2.1.3 Client (Ứng dụng yêu cầu truy cập)

#### Định nghĩa

Client là ứng dụng yêu cầu truy cập vào tài nguyên được bảo vệ thay mặt cho Resource Owner. Client có thể là web application, mobile application, desktop application, hoặc server-side service.

#### Phân loại Client

##### 1. Confidential Client

**Đặc điểm:**
- Có thể bảo mật Client Secret
- Chạy trên server-side
- Có thể thực hiện các yêu cầu bảo mật cao

**Ví dụ:**
- Web applications (ASP.NET Core MVC, Spring Boot)
- Server-side services
- Background jobs

**Yêu cầu bảo mật:**
- Client Secret phải được lưu trữ an toàn (environment variables, Azure Key Vault, AWS Secrets Manager)
- HTTPS bắt buộc
- Valid redirect URIs

##### 2. Public Client

**Đặc điểm:**
- Không thể bảo mật Client Secret
- Chạy trên client-side
- Cần các biện pháp bảo mật bổ sung

**Ví dụ:**
- Mobile applications (iOS, Android)
- Single-page applications (React, Angular, Vue)
- Desktop applications

**Yêu cầu bảo mật:**
- Sử dụng PKCE (Proof Key for Code Exchange)
- Không lưu Client Secret
- Strict redirect URI validation
- Short-lived tokens

#### Thành phần cấu hình

```csharp
public class Client
{
    public string ClientId { get; set; }
    public string ClientName { get; set; }
    public List<string> RedirectUris { get; set; }
    public List<string> PostLogoutRedirectUris { get; set; }
    public List<string> AllowedGrantTypes { get; set; }
    public List<string> AllowedScopes { get; set; }
    public bool RequireClientSecret { get; set; }
    public bool RequirePkce { get; set; }
    public int AccessTokenLifetime { get; set; }
    public int RefreshTokenLifetime { get; set; }
    public bool AllowOfflineAccess { get; set; }
}
```

#### Ví dụ cấu hình

```csharp
new Client
{
    ClientId = "web-client",
    ClientName = "Web Application",
    RedirectUris = { "https://localhost:5001/signin-oidc" },
    PostLogoutRedirectUris = { "https://localhost:5001/signout-callback-oidc" },
    AllowedGrantTypes = GrantTypes.CodeAndClientCredentials,
    RequireClientSecret = true,
    ClientSecrets = { new Secret("secret".Sha256()) },
    AllowedScopes = { "openid", "profile", "api1" },
    AccessTokenLifetime = 3600,
    RefreshTokenUsage = TokenUsage.ReUse,
    RefreshTokenExpiration = TimeSpan.FromDays(30),
    AllowOfflineAccess = true
}
```

### 2.1.4 Authorization Server (Máy chủ ủy quyền)

#### Định nghĩa

Authorization Server là server phát hành access token sau khi xác thực Resource Owner và nhận được sự ủy quyền. Đây là thành phần trung tâm của hệ thống OAuth 2.0.

#### Chức năng chính

1. **Xác thực Resource Owner**
   - Login endpoint
   - Logout endpoint
   - MFA support
   - External identity providers (Google, Facebook, Microsoft)

2. **Xác thực Client**
   - Client authentication (Client Secret, JWT assertion, mTLS)
   - Client validation
   - Redirect URI validation

3. **Phát hành Token**
   - Access token generation
   - Refresh token generation
   - ID token generation (OpenID Connect)
   - Token signing and encryption

4. **Quản lý Token**
   - Token storage
   - Token revocation
   - Token introspection
   - Token rotation

5. **Quản lý Authorization**
   - Consent management
   - Scope validation
   - Claims mapping
   - Policy enforcement

#### Các endpoint chính

##### 1. Authorization Endpoint

```
GET /connect/authorize
```

**Chức năng:** Khởi tạo luồng ủy quyền, hiển thị trang login/consent

**Parameters:**
- `client_id`: ID của client
- `redirect_uri`: URI để redirect sau khi ủy quyền
- `response_type`: Loại response (code, token, id_token)
- `scope`: Các scopes được yêu cầu
- `state`: Giá trị ngẫu nhiên để prevent CSRF
- `code_challenge`: PKCE challenge
- `code_challenge_method`: PKCE method (S256, plain)

**Response:**
- Authorization code (nếu response_type=code)
- Access token (nếu response_type=token)
- ID token (nếu response_type=id_token)

##### 2. Token Endpoint

```
POST /connect/token
```

**Chức năng:** Đổi authorization code hoặc client credentials lấy access token

**Parameters:**
- `grant_type`: Loại grant (authorization_code, client_credentials, refresh_token)
- `code`: Authorization code (cho authorization_code grant)
- `redirect_uri`: Redirect URI (phải khớp với request ủy quyền)
- `client_id`: ID của client
- `client_secret`: Secret của client (cho confidential clients)
- `code_verifier`: PKCE verifier
- `refresh_token`: Refresh token (cho refresh_token grant)

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_xxxxxxxxxxxxxx",
  "scope": "openid profile api1"
}
```

##### 3. UserInfo Endpoint

```
GET /connect/userinfo
```

**Chức năng:** Trả về thông tin profile của user (OpenID Connect)

**Headers:**
```
Authorization: Bearer <access_token>
```

**Response:**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "picture": "https://example.com/avatar.jpg"
}
```

##### 4. Revocation Endpoint

```
POST /connect/revocation
```

**Chức năng:** Thu hồi access token hoặc refresh token

**Parameters:**
- `token`: Token cần thu hồi
- `token_type_hint`: Loại token (access_token, refresh_token)

##### 5. Introspection Endpoint

```
POST /connect/introspect
```

**Chức năng:** Kiểm tra trạng thái của token

**Parameters:**
- `token`: Token cần kiểm tra
- `token_type_hint`: Loại token

**Response:**
```json
{
  "active": true,
  "sub": "1234567890",
  "client_id": "web-client",
  "scope": "openid profile api1",
  "exp": 1516239022,
  "iat": 1516235422
}
```

#### Cấu trúc nội bộ

```mermaid
flowchart TD
    A[Authorization Server] --> B[Authentication Module]
    A --> C[Authorization Module]
    A --> D[Token Service]
    A --> E[Client Store]
    A --> F[User Store]
    A --> G[Token Store]
    A --> H[Signing Keys]
    
    B --> B1[Login Handler]
    B --> B2[Logout Handler]
    B --> B3[MFA Handler]
    B --> B4[External Providers]
    
    C --> C1[Consent Handler]
    C --> C2[Scope Validator]
    C --> C3[Claims Mapper]
    C --> C4[Policy Engine]
    
    D --> D1[Token Generator]
    D --> D2[Token Signer]
    D --> D3[Token Encryptor]
    D --> D4[Token Validator]
```

### 2.1.5 Resource Server (Máy chủ tài nguyên)

#### Định nghĩa

Resource Server là server lưu trữ tài nguyên được bảo vệ và chấp nhận access token. Resource Server chịu trách nhiệm xác thực token và kiểm tra quyền truy cập trước khi cung cấp tài nguyên.

#### Chức năng chính

1. **Xác thực Access Token**
   - Token validation (signature, expiration, issuer)
   - Token introspection (nếu sử dụng opaque tokens)
   - Token revocation checking

2. **Kiểm tra Quyền Truy Cập**
   - Scope validation
   - Claims-based authorization
   - Policy enforcement

3. **Cung cấp Tài Nguyên**
   - API endpoints
   - Data access
   - Business logic

#### Cấu hình ASP.NET Core

```csharp
services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
{
    options.Authority = "https://auth.example.com";
    options.Audience = "api1";
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new X509SecurityKey(new X509Certificate2("certificate.cer"))
    };
});

services.AddAuthorization(options =>
{
    options.AddPolicy("ReadScope", policy =>
        policy.RequireClaim("scope", "api1.read"));
    
    options.AddPolicy("WriteScope", policy =>
        policy.RequireClaim("scope", "api1.write"));
});
```

#### Ví dụ API Controller

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
    
    [HttpGet("{id}")]
    [Authorize(Policy = "ReadScope")]
    public async Task<IActionResult> GetProduct(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        if (product == null)
            return NotFound();
        return Ok(product);
    }
}
```

## 2.2 Luồng tương tác giữa các thành phần

### 2.2.1 Luồng tổng quan

```mermaid
sequenceDiagram
    participant RO as Resource Owner
    participant C as Client
    participant AS as Authorization Server
    participant RS as Resource Server
    
    Note over RO,RS: Authorization Code Flow
    
    RO->>C: 1. Yêu cầu truy cập tài nguyên
    C->>AS: 2. Gửi request ủy quyền
    AS->>RO: 3. Hiển thị trang login
    RO->>AS: 4. Nhập thông tin đăng nhập
    AS->>RO: 5. Hiển thị trang consent
    RO->>AS: 6. Chấp nhận quyền truy cập
    AS->>C: 7. Redirect với authorization code
    C->>AS: 8. Đổi code lấy access token
    AS->>C: 9. Trả về access token
    C->>RS: 10. Gửi request với access token
    RS->>AS: 11. Xác thực access token (tùy chọn)
    RS->>C: 12. Trả về tài nguyên
```

### 2.2.2 Chi tiết từng bước

#### Bước 1: Resource Owner yêu cầu truy cập

**Kịch bản:** Resource Owner muốn sử dụng Client để truy cập tài nguyên của họ

**Ví dụ:**
- Người dùng muốn sử dụng ứng dụng photo editor để truy cập ảnh trên Google Photos
- Người dùng muốn sử dụng ứng dụng fitness để truy cập dữ liệu hoạt động trên Apple Health

#### Bước 2: Client gửi request ủy quyền

**Request:**
```
GET /connect/authorize?
    client_id=web-client&
    redirect_uri=https://localhost:5001/signin-oidc&
    response_type=code&
    scope=openid profile api1&
    state=xyz123&
    code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8KWEeRv21o-
    code_challenge_method=S256
```

**Điểm quan trọng:**
- `client_id`: Xác định client
- `redirect_uri`: Đảm bảo redirect đến URI được phép
- `response_type=code`: Sử dụng authorization code flow
- `scope`: Xác định quyền truy cập được yêu cầu
- `state`: Prevent CSRF attacks
- `code_challenge`: PKCE (cho public clients)

#### Bước 3-4: Resource Owner đăng nhập

**Kịch bản:** Authorization Server hiển thị trang login, Resource Owner nhập thông tin đăng nhập

**Điểm quan trọng:**
- Sử dụng HTTPS
- Validate input
- Rate limiting để prevent brute force attacks
- Hỗ trợ MFA

#### Bước 5-6: Resource Owner chấp nhận quyền truy cập

**Kịch bản:** Authorization Server hiển thị trang consent, Resource Owner chấp nhận hoặc từ chối

**Điểm quan trọng:**
- Hiển thị rõ ràng các scopes được yêu cầu
- Cho phép Resource Owner chọn scopes cụ thể
- Lưu consent để không cần hỏi lại lần sau

#### Bước 7: Authorization Server redirect với authorization code

**Response:**
```
HTTP/1.1 302 Found
Location: https://localhost:5001/signin-oidc?
    code=Grz7wW7g7G7g7g7g7g7g7g7g7g7g7g7g7g&
    state=xyz123
```

**Điểm quan trọng:**
- Authorization code chỉ có hiệu lực một lần
- Code hết hạn nhanh (thường 5-10 phút)
- State phải khớp với request ban đầu

#### Bước 8-9: Client đổi code lấy access token

**Request:**
```
POST /connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=Grz7wW7g7G7g7g7g7g7g7g7g7g7g7g7g7g&
redirect_uri=https://localhost:5001/signin-oidc&
client_id=web-client&
client_secret=secret&
code_verifier=dBjftJeZ4Cv-mft92mrhf2sWFp27JbNytTYMv-
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_xxxxxxxxxxxxxx",
  "scope": "openid profile api1",
  "id_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Điểm quan trọng:**
- Client Secret chỉ được gửi qua HTTPS
- Redirect URI phải khớp với request ban đầu
- Code verifier phải khớp với code challenge

#### Bước 10-12: Client truy cập tài nguyên

**Request:**
```
GET /api/products
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "Product 1",
    "price": 100.00
  },
  {
    "id": 2,
    "name": "Product 2",
    "price": 200.00
  }
]
```

**Điểm quan trọng:**
- Access token được gửi trong Authorization header
- Resource Server xác thực token trước khi xử lý request
- Scope được kiểm tra để đảm bảo quyền truy cập

## 2.3 Diagram kiến trúc tổng quan

### 2.3.1 Diagram kiến trúc OAuth 2.0

```mermaid
flowchart TD
    subgraph "Resource Owner Layer"
        RO[Resource Owner]
    end
    
    subgraph "Client Layer"
        C1[Web App]
        C2[Mobile App]
        C3[Desktop App]
        C4[Service]
    end
    
    subgraph "Authorization Server Layer"
        AS[Authorization Server]
        AS1[Authentication Module]
        AS2[Authorization Module]
        AS3[Token Service]
        AS4[Client Store]
        AS5[User Store]
        AS6[Token Store]
        AS7[Signing Keys]
        
        AS --> AS1
        AS --> AS2
        AS --> AS3
        AS --> AS4
        AS --> AS5
        AS --> AS6
        AS --> AS7
    end
    
    subgraph "Resource Server Layer"
        RS1[API 1]
        RS2[API 2]
        RS3[API 3]
    end
    
    RO --> C1
    RO --> C2
    RO --> C3
    
    C1 --> AS
    C2 --> AS
    C3 --> AS
    C4 --> AS
    
    AS --> RS1
    AS --> RS2
    AS --> RS3
    
    C1 --> RS1
    C2 --> RS1
    C3 --> RS2
    C4 --> RS3
```

### 2.3.2 Diagram kiến trúc chi tiết

```mermaid
flowchart TD
    subgraph "Resource Owner"
        RO[Resource Owner]
    end
    
    subgraph "Client Applications"
        subgraph "Confidential Clients"
            WC[Web Application]
            SS[Server Service]
        end
        
        subgraph "Public Clients"
            MA[Mobile Application]
            SPA[Single-Page App]
        end
    end
    
    subgraph "Authorization Server"
        subgraph "Authentication"
            Login[Login Endpoint]
            Logout[Logout Endpoint]
            MFA[MFA Handler]
            Ext[External Providers]
        end
        
        subgraph "Authorization"
            Consent[Consent Handler]
            Scope[Scope Validator]
            Claims[Claims Mapper]
            Policy[Policy Engine]
        end
        
        subgraph "Token Management"
            Gen[Token Generator]
            Sign[Token Signer]
            Encr[Token Encryptor]
            Val[Token Validator]
        end
        
        subgraph "Storage"
            CS[Client Store]
            US[User Store]
            TS[Token Store]
            SK[Signing Keys]
        end
    end
    
    subgraph "Resource Servers"
        API1[Product API]
        API2[Order API]
        API3[User API]
    end
    
    RO --> WC
    RO --> MA
    RO --> SPA
    
    WC --> Login
    MA --> Login
    SPA --> Login
    
    Login --> Consent
    Consent --> Gen
    Gen --> Sign
    Sign --> TS
    
    WC --> API1
    MA --> API1
    SPA --> API2
    SS --> API3
    
    API1 --> Val
    API2 --> Val
    API3 --> Val
    
    Val --> TS
```

## 2.4 Mô hình triển khai (Single-tenant, Multi-tenant)

### 2.4.1 Single-tenant Architecture

#### Định nghĩa

Single-tenant architecture là mô hình mà mỗi khách hàng có một instance riêng biệt của Authentication Server.

#### Đặc điểm

**Ưu điểm:**
1. **Isolation hoàn toàn:** Dữ liệu và cấu hình của mỗi khách hàng được tách biệt
2. **Tùy chỉnh cao:** Mỗi khách hàng có thể có cấu hình riêng
3. **Security cao:** Giảm nguy cơ data leakage giữa các khách hàng
4. **Performance:** Không có resource contention giữa các khách hàng

**Nhược điểm:**
1. **Chi phí cao:** Cần nhiều resources (server, database, storage)
2. **Quản lý phức tạp:** Cần quản lý nhiều instances
3. **Khó scale:** Scale theo nhu cầu của từng khách hàng
4. **Maintenance:** Cần cập nhật và patch nhiều instances

#### Kiến trúc

```mermaid
flowchart TD
    subgraph "Customer A"
        AS_A[Auth Server A]
        DB_A[Database A]
        RS_A[Resource Server A]
    end
    
    subgraph "Customer B"
        AS_B[Auth Server B]
        DB_B[Database B]
        RS_B[Resource Server B]
    end
    
    subgraph "Customer C"
        AS_C[Auth Server C]
        DB_C[Database C]
        RS_C[Resource Server C]
    end
    
    UserA[User A] --> AS_A
    UserB[User B] --> AS_B
    UserC[User C] --> AS_C
    
    AS_A --> DB_A
    AS_B --> DB_B
    AS_C --> DB_C
    
    AS_A --> RS_A
    AS_B --> RS_B
    AS_C --> RS_C
```

#### Khi nào sử dụng

- Enterprise customers yêu cầu isolation hoàn toàn
- Các ngành có yêu cầu compliance cao (banking, healthcare, government)
- Khách hàng yêu cầu tùy chỉnh sâu
- Khi security là ưu tiên hàng đầu

### 2.4.2 Multi-tenant Architecture

#### Định nghĩa

Multi-tenant architecture là mô hình mà một instance Authentication Server phục vụ nhiều khách hàng.

#### Đặc điểm

**Ưu điểm:**
1. **Chi phí thấp:** Chia sẻ resources giữa nhiều khách hàng
2. **Quản lý đơn giản:** Chỉ cần quản lý một instance
3. **Dễ scale:** Scale toàn bộ hệ thống thay vì từng khách hàng
4. **Rapid deployment:** Deploy mới một lần cho tất cả khách hàng

**Nhược điểm:**
1. **Isolation thấp:** Dữ liệu và cấu hình có thể bị ảnh hưởng lẫn nhau
2. **Tùy chỉnh hạn chế:** Khó tùy chỉnh cho từng khách hàng
3. **Security rủi ro:** Có thể có data leakage nếu không được triển khai đúng
4. **Performance contention:** Nhiều khách hàng chia sẻ resources

#### Kiến trúc

```mermaid
flowchart TD
    subgraph "Shared Infrastructure"
        AS[Auth Server]
        DB[(Database)]
        RS[Resource Server]
    end
    
    subgraph "Tenant A"
        UserA[User A]
        DataA[Tenant Data A]
    end
    
    subgraph "Tenant B"
        UserB[User B]
        DataB[Tenant Data B]
    end
    
    subgraph "Tenant C"
        UserC[User C]
        DataC[Tenant Data C]
    end
    
    UserA --> AS
    UserB --> AS
    UserC --> AS
    
    AS --> DB
    
    DB --> DataA
    DB --> DataB
    DB --> DataC
    
    AS --> RS
```

#### Chiến lược Multi-tenancy

##### 1. Shared Database, Shared Schema

**Đặc điểm:**
- Tất cả tenants chia sẻ cùng database và schema
- Thêm tenant_id vào mỗi bảng
- Sử dụng row-level security

**Ưu điểm:**
- Đơn giản nhất
- Chi phí thấp nhất

**Nhược điểm:**
- Rủi ro data leakage cao
- Khó tùy chỉnh cho từng tenant
- Performance contention

**Ví dụ schema:**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    tenant_id INT NOT NULL,
    username VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    INDEX idx_tenant (tenant_id)
);

CREATE TABLE clients (
    id INT PRIMARY KEY,
    tenant_id INT NOT NULL,
    client_id VARCHAR(255) NOT NULL,
    client_name VARCHAR(255) NOT NULL,
    INDEX idx_tenant (tenant_id)
);
```

##### 2. Shared Database, Separate Schema

**Đặc điểm:**
- Tất cả tenants chia sẻ cùng database
- Mỗi tenant có schema riêng
- Tách biệt dữ liệu tốt hơn

**Ưu điểm:**
- Tách biệt dữ liệu tốt hơn
- Có thể tùy chỉnh schema cho từng tenant

**Nhược điểm:**
- Quản lý nhiều schemas
- Chi phí cao hơn shared schema

**Ví dụ schema:**
```sql
-- Tenant A schema
CREATE SCHEMA tenant_a;
CREATE TABLE tenant_a.users (
    id INT PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL
);

-- Tenant B schema
CREATE SCHEMA tenant_b;
CREATE TABLE tenant_b.users (
    id INT PRIMARY KEY,
    username VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL
);
```

##### 3. Separate Database

**Đặc điểm:**
- Mỗi tenant có database riêng
- Tách biệt dữ liệu hoàn toàn
- Có thể scale từng database

**Ưu điểm:**
- Tách biệt dữ liệu hoàn toàn
- Có thể tùy chỉnh schema cho từng tenant
- Có thể scale từng database

**Nhược điểm:**
- Quản lý nhiều databases
- Chi phí cao nhất

**Ví dụ:**
```csharp
// Determine tenant from request
var tenantId = GetTenantIdFromRequest();

// Get connection string for tenant
var connectionString = _configuration.GetConnectionString($"Tenant_{tenantId}");

// Create DbContext with tenant-specific connection string
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));
```

#### Khi nào sử dụng

- SaaS applications
- Startups và SMEs
- Khi chi phí là ưu tiên
- Khi cần rapid deployment

### 2.4.3 Hybrid Architecture

#### Định nghĩa

Hybrid architecture kết hợp single-tenant và multi-tenant để tận dụng ưu điểm của cả hai.

#### Đặc điểm

**Ưu điểm:**
- Linh hoạt trong việc phân loại khách hàng
- Enterprise customers có thể có instance riêng
- SMEs có thể chia sẻ resources
- Tối ưu hóa chi phí

**Nhược điểm:**
- Quản lý phức tạp
- Cần chiến lược phân loại khách hàng rõ ràng

#### Kiến trúc

```mermaid
flowchart TD
    subgraph "Enterprise Customers"
        subgraph "Customer A"
            AS_A[Auth Server A]
            DB_A[Database A]
        end
        
        subgraph "Customer B"
            AS_B[Auth Server B]
            DB_B[Database B]
        end
    end
    
    subgraph "SME Customers"
        subgraph "Shared Infrastructure"
            AS[Auth Server]
            DB[(Database)]
        end
        
        subgraph "Tenant A"
            UserA[User A]
            DataA[Tenant Data A]
        end
        
        subgraph "Tenant B"
            UserB[User B]
            DataB[Tenant Data B]
        end
    end
    
    UserA --> AS
    UserB --> AS
    AS --> DB
    DB --> DataA
    DB --> DataB
```

## Tóm tắt

Phần này đã mô tả chi tiết kiến trúc hệ thống OAuth 2.0:

1. **Các thành phần chính:** Resource Owner, Client, Authorization Server, Resource Server
2. **Luồng tương tác:** Chi tiết từng bước trong authorization code flow
3. **Diagram kiến trúc:** Tổng quan và chi tiết về cách các thành phần tương tác
4. **Mô hình triển khai:** Single-tenant, multi-tenant, và hybrid architecture

Phần tiếp theo sẽ đi sâu vào các luồng ủy quyền phổ biến và cách triển khai chúng.
