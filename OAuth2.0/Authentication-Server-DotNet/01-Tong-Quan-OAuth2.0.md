# Phần 1: Tổng quan về OAuth 2.0 và Authentication Server

## 1.1 Giới thiệu và vai trò của OAuth 2.0

### 1.1.1 OAuth 2.0 là gì?

OAuth 2.0 (Open Authorization 2.0) là một tiêu chuẩn ủy quyền (authorization framework) cho phép ứng dụng của bên thứ ba có được quyền truy cập giới hạn vào tài khoản người dùng trên một dịch vụ HTTP, mà không cần chia sẻ thông tin đăng nhập của người dùng.

**Đặc điểm chính:**

- **Ủ quyền thay vì Xác thực**: OAuth 2.0 tập trung vào việc cấp quyền truy cập, không phải xác thực danh tính người dùng
- **Delegated Access**: Cho phép người dùng ủy quyền quyền truy cập tài nguyên của họ cho ứng dụng khác
- **Token-based**: Sử dụng token (access token) thay vì thông tin đăng nhập để truy cập tài nguyên
- **Standardized**: Được định nghĩa trong RFC 6749 và được chấp nhận rộng rãi

### 1.1.2 Vai trò của OAuth 2.0

OAuth 2.0 đóng vai trò quan trọng trong kiến trúc ứng dụng hiện đại:

**1. Giải quyết vấn đề chia sẻ thông tin đăng nhập**

- Trước OAuth 2.0: Người dùng phải chia sẻ username/password cho ứng dụng của bên thứ ba
- Với OAuth 2.0: Người dùng chỉ cần cấp quyền truy cập giới hạn

**2. Cho phép Single Sign-On (SSO)**

- Người dùng đăng nhập một lần và truy cập nhiều ứng dụng
- Giảm tải cho người dùng và tăng trải nghiệm người dùng

**3. Cung cấp kiểm soát chi tiết**

- Người dùng có thể thu hồi quyền truy cập bất cứ lúc nào
- Ứng dụng chỉ có quyền truy cập các tài nguyên được cấp phép

**4. Hỗ trợ nhiều loại ứng dụng**

- Web applications
- Mobile applications
- Desktop applications
- Server-to-server communication

### 1.1.3 Lịch sử phát triển

- **OAuth 1.0** (2007): Phiên bản đầu tiên, phức tạp và khó triển khai
- **OAuth 2.0** (2012): Đơn giản hóa, tập trung vào client-side, được chấp nhận rộng rãi
- **OAuth 2.1** (Đang phát triển): Cải thiện bảo mật, yêu cầu PKCE, loại bỏ implicit flow

### 1.1.4 Các use case phổ biến

**1. Social Login**

- Đăng nhập bằng Google, Facebook, GitHub, v.v.
- Ứng dụng không cần quản lý thông tin đăng nhập

**2. API Access Control**

- Ứng dụng mobile truy cập API của backend
- Bên thứ ba tích hợp với API của bạn

**3. Microservices Communication**

- Service-to-service authentication
- Machine-to-machine communication

**4. Enterprise Integration**

- SSO cho các ứng dụng doanh nghiệp
- Federation với các identity providers khác

## 1.2 Các khái niệm cốt lõi (Grant Types, Roles, Tokens)

### 1.2.1 Các vai trò (Roles) trong OAuth 2.0

OAuth 2.0 định nghĩa bốn vai trò chính:

#### 1. Resource Owner

**Định nghĩa:** Entity có khả năng cấp quyền truy cập vào tài nguyên được bảo vệ

**Đặc điểm:**
- Thường là người dùng cuối (end-user)
- Có quyền sở hữu hoặc kiểm soát tài nguyên
- Quyết định cấp quyền cho client

**Ví dụ:**
- Người dùng Facebook sở hữu profile và dữ liệu của họ
- Người dùng Google sở hữu email và calendar của họ

#### 2. Client

**Định nghĩa:** Ứng dụng yêu cầu truy cập vào tài nguyên được bảo vệ

**Đặc điểm:**
- Có thể là web app, mobile app, desktop app, hoặc server-side service
- Được đăng ký với Authorization Server
- Có Client ID và (tùy chọn) Client Secret

**Phân loại Client:**

**Confidential Client:**
- Có thể bảo mật Client Secret
- Server-side web applications
- Có thể thực hiện các yêu cầu bảo mật cao

**Public Client:**
- Không thể bảo mật Client Secret
- Mobile apps, Single-page applications (SPAs)
- Cần các biện pháp bảo mật bổ sung (PKCE)

#### 3. Authorization Server

**Định nghĩa:** Server phát hành access token sau khi xác thực Resource Owner và nhận được sự ủy quyền

**Chức năng:**
- Xác thực Resource Owner (login)
- Xác thực Client
- Cấp phát access token
- Quản lý refresh token
- Xác thực và ủy quyền request

**Ví dụ:**
- Duende IdentityServer
- Auth0
- Okta
- Microsoft Azure AD

#### 4. Resource Server

**Định nghĩa:** Server lưu trữ tài nguyên được bảo vệ và chấp nhận access token

**Chức năng:**
- Xác thực access token
- Kiểm tra quyền truy cập (scopes/claims)
- Cung cấp tài nguyên được yêu cầu

**Ví dụ:**
- API backend
- Microservice
- RESTful service

### 1.2.2 Các loại ủy quyền (Grant Types)

OAuth 2.0 định nghĩa bốn loại ủy quyền chính:

#### 1. Authorization Code Grant

**Mô tả:** Client nhận authorization code từ Authorization Server và đổi lấy access token

**Đặc điểm:**
- An toàn nhất cho confidential clients
- Authorization code không chứa token
- Hỗ trợ refresh token
- Yêu cầu client authentication

**Use case:**
- Web applications (server-side)
- Mobile applications (với PKCE)
- SPAs (với PKCE)

#### 2. Implicit Grant

**Mô tả:** Access token được trả về trực tiếp trong URL redirect

**Đặc điểm:**
- Đơn giản hơn authorization code flow
- Access token hiển thị trong URL (không an toàn)
- Không hỗ trợ refresh token
- **Được khuyến nghị không sử dụng** (được thay thế bởi PKCE)

**Use case:**
- SPAs (trước đây)
- Mobile apps (trước đây)

#### 3. Resource Owner Password Credentials Grant

**Mô tả:** Resource Owner cung cấp username/password trực tiếp cho client

**Đặc điểm:**
- Client nhận được thông tin đăng nhập
- Chỉ nên sử dụng cho trusted clients
- Không nên sử dụng cho third-party apps
- **Được khuyến nghị không sử dụng** trừ khi cần thiết

**Use case:**
- Legacy applications
- First-party mobile apps
- Khi không thể sử dụng các grant types khác

#### 4. Client Credentials Grant

**Mô tả:** Client sử dụng thông tin đăng nhập của chính mình (không phải resource owner) để lấy access token

**Đặc điểm:**
- Không có sự tham gia của resource owner
- Token đại diện cho client, không phải user
- Không hỗ trợ refresh token
- Hỗ trợ machine-to-machine communication

**Use case:**
- Service-to-service communication
- Batch jobs
- Background processes

### 1.2.3 Các loại Token

#### 1. Access Token

**Mô tả:** Token được sử dụng để truy cập tài nguyên được bảo vệ

**Đặc điểm:**
- Được gửi trong Authorization header: `Authorization: Bearer <access_token>`
- Có thời gian sống ngắn (thường 5-60 phút)
- Có thể là JWT hoặc opaque token
- Chứa scopes và claims

**Ví dụ:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

#### 2. Refresh Token

**Mô tả:** Token được sử dụng để lấy access token mới khi access token hết hạn

**Đặc điểm:**
- Có thời gian sống dài hơn access token
- Chỉ được sử dụng với Authorization Server
- Có thể được thu hồi
- Nên được lưu trữ an toàn

**Ví dụ:**
```http
POST /oauth/token
grant_type=refresh_token
refresh_token=rt_xxxxxxxxxxxxxx
```

#### 3. ID Token (OpenID Connect)

**Mô tả:** Token chứa thông tin về người dùng (authentication)

**Đặc điểm:**
- Chỉ có trong OpenID Connect
- Luôn là JWT
- Chứa thông tin profile của user
- Được ký bởi Authorization Server

**Ví dụ JWT payload:**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "email": "john.doe@example.com",
  "iss": "https://auth.example.com",
  "aud": "client-id",
  "exp": 1516239022,
  "iat": 1516239022
}
```

## 1.3 OAuth 2.0 vs OpenID Connect

### 1.3.1 Sự khác biệt cơ bản

**OAuth 2.0:**
- Tập trung vào **Authorization** (ủy quyền)
- Trả về Access Token
- Không cung cấp thông tin về người dùng
- Câu hỏi: "Ứng dụng này có quyền truy cập tài nguyên này không?"

**OpenID Connect (OIDC):**
- Mở rộng OAuth 2.0 để hỗ trợ **Authentication** (xác thực)
- Trả về ID Token (JWT) bên cạnh Access Token
- Cung cấp thông tin profile của người dùng
- Câu hỏi: "Người dùng này là ai?"

### 1.3.2 OpenID Connect thêm gì vào OAuth 2.0?

#### 1. ID Token

- JWT chứa thông tin về người dùng
- Chứa claims chuẩn: `sub`, `name`, `email`, `picture`, v.v.
- Được ký bởi Authorization Server

#### 2. UserInfo Endpoint

- Endpoint để lấy thông tin profile của người dùng
- Được bảo vệ bởi Access Token
- Trả về JSON với thông tin user

#### 3. Các claims chuẩn

**Standard Claims:**
- `iss`: Issuer (người phát hành token)
- `sub`: Subject (ID duy nhất của user)
- `aud`: Audience (client được phép sử dụng token)
- `exp`: Expiration time
- `iat`: Issued at time
- `auth_time`: Thời gian xác thực
- `nonce`: Giá trị ngẫu nhiên để prevent replay attacks

#### 4. Discovery Document

- Endpoint cung cấp metadata về Authorization Server
- URL: `/.well-known/openid-configuration`
- Chứa thông tin về endpoints, scopes, claims, v.v.

### 1.3.3 Khi nào sử dụng OAuth 2.0 hay OpenID Connect?

**Sử dụng OAuth 2.0 khi:**
- Chỉ cần ủy quyền truy cập tài nguyên
- Không cần thông tin về người dùng
- Service-to-service communication
- API access control

**Sử dụng OpenID Connect khi:**
- Cần xác thực người dùng
- Cần thông tin profile của user
- Social login
- SSO cho ứng dụng web/mobile

**Thực tế:**
- Hầu hết các triển khai hiện đại đều sử dụng OpenID Connect
- OIDC cung cấp đầy đủ tính năng của OAuth 2.0
- Các framework như Duende IdentityServer hỗ trợ cả hai

## 1.4 Authentication Server và mục đích sử dụng

### 1.4.1 Authentication Server là gì?

**Định nghĩa:** Authentication Server (hay Identity Server) là một thành phần trung tâm chịu trách nhiệm quản lý danh tính và quyền truy cập trong hệ thống phân tán.

**Chức năng chính:**
1. **User Authentication:** Xác thực người dùng (login, logout)
2. **Token Issuance:** Phát hành access token, refresh token, ID token
3. **Client Management:** Quản lý các ứng dụng được phép sử dụng hệ thống
4. **Authorization:** Kiểm soát quyền truy cập dựa trên scopes và claims
5. **Federation:** Hỗ trợ đăng nhập qua các identity providers khác
6. **Single Sign-On (SSO):** Cho phép đăng nhập một lần, truy cập nhiều ứng dụng

### 1.4.2 Tại sao cần Authentication Server?

#### 1. Centralized Identity Management

**Vấn đề:** Mỗi ứng dụng quản lý user riêng
- Trùng lặp thông tin user
- Khó đồng bộ dữ liệu
- Khó quản lý quyền truy cập

**Giải pháp:** Authentication Server trung tâm
- User lưu trữ tại một nơi
- Được chia sẻ giữa nhiều ứng dụng
- Dễ dàng quản lý và đồng bộ

#### 2. Security Benefits

**Bảo mật tập trung:**
- Logic bảo mật được triển khai tại một nơi
- Dễ dàng cập nhật và patch
- Giảm bề mặt tấn công

**Token-based Security:**
- Không chia sẻ thông tin đăng nhập
- Token có thể được thu hồi
- Fine-grained access control

#### 3. Developer Productivity

**Giảm tải cho nhà phát triển:**
- Không cần triển khai authentication từ đầu
- Tích hợp nhanh với các social logins
- Sử dụng các thư viện chuẩn

**Tích hợp dễ dàng:**
- Các ứng dụng mới có thể tích hợp nhanh
- Sử dụng các protocol chuẩn (OAuth 2.0, OIDC)
- Hỗ trợ nhiều loại ứng dụng

#### 4. User Experience

**Single Sign-On (SSO):**
- Đăng nhập một lần, truy cập nhiều ứng dụng
- Giảm friction cho người dùng
- Tăng trải nghiệm người dùng

**Consent Management:**
- Người dùng kiểm soát quyền truy cập
- Có thể thu hồi quyền bất cứ lúc nào
- Minh bạch về quyền truy cập

### 1.4.3 Các thành phần của Authentication Server

#### 1. User Store

**Chức năng:** Lưu trữ thông tin người dùng

**Thành phần:**
- User profile (username, email, name, v.v.)
- Credentials (password hash, 2FA secrets)
- User attributes (roles, permissions, groups)
- Device information

**Công nghệ:**
- ASP.NET Core Identity
- Entity Framework / Dapper
- SQL Server / PostgreSQL / MySQL

#### 2. Client Store

**Chức năng:** Quản lý các ứng dụng được phép sử dụng hệ thống

**Thành phần:**
- Client ID và Client Secret
- Redirect URIs
- Allowed grant types
- Allowed scopes
- Client metadata

**Ví dụ:**
```csharp
public class Client
{
    public string ClientId { get; set; }
    public string ClientSecret { get; set; }
    public List<string> RedirectUris { get; set; }
    public List<string> AllowedGrantTypes { get; set; }
    public List<string> AllowedScopes { get; set; }
}
```

#### 3. Token Store

**Chức năng:** Quản lý tokens đã phát hành

**Thành phần:**
- Refresh tokens
- Reference tokens (nếu sử dụng opaque tokens)
- Token metadata (expiration, revocation)
- Device codes (cho device flow)

**Lưu trữ:**
- Database (SQL, NoSQL)
- Redis (cho performance)
- Distributed cache

#### 4. Signing Keys

**Chức năng:** Ký và xác thực JWT tokens

**Thành phần:**
- RSA signing keys
- Key rotation
- Key storage (HSM, Azure Key Vault, AWS KMS)

**Ví dụ:**
```csharp
services.AddIdentityServer()
    .AddSigningCredential(new X509Certificate2("certificate.pfx", "password"));
```

#### 5. Configuration Store

**Chức năng:** Lưu trữ cấu hình hệ thống

**Thành phần:**
- Identity resources (claims được phép trả về)
- API resources (APIs được bảo vệ)
- Scopes (quyền truy cập)
- Identity providers (external logins)

### 1.4.4 Các Authentication Server phổ biến

#### 1. Duende IdentityServer (.NET)

**Đặc điểm:**
- Framework cho ASP.NET Core
- Hỗ trợ đầy đủ OAuth 2.0 và OpenID Connect
- Mở rộng cao, có thể tùy chỉnh
- Cộng đồng lớn, tài liệu phong phú

**Lợi ích:**
- Tích hợp sâu với ASP.NET Core Identity
- Performance cao
- Active development
- Commercial support có sẵn

#### 2. Auth0

**Đặc điểm:**
- Cloud-based identity platform
- Hỗ trợ nhiều protocol (OAuth 2.0, OIDC, SAML)
- Rich features (MFA, social logins, user management)
- Easy integration

**Lợi ích:**
- Không cần quản lý infrastructure
- Bảo mật được quản lý bởi Auth0
- Rich dashboard và analytics
- Hỗ trợ enterprise

#### 3. Okta

**Đặc điểm:**
- Cloud-based identity platform
- Tập trung vào enterprise
- Hỗ trợ SSO, MFA, lifecycle management
- Strong security

**Lợi ích:**
- Enterprise-grade security
- Rich integration options
- Good documentation
- Active community

#### 4. Azure Active Directory / Microsoft Entra ID

**Đặc điểm:**
- Microsoft's cloud identity service
- Tích hợp sâu với Microsoft ecosystem
- Hỗ trợ enterprise features
- Strong security

**Lợi ích:**
- Tích hợp với Microsoft 365
- Enterprise-grade security
- Rich features
- Global availability

### 1.4.5 Khi nào nên xây dựng Authentication Server riêng?

**Nên xây dựng riêng khi:**
1. Cần tùy chỉnh sâu logic authentication/authorization
2. Yêu cầu compliance đặc biệt (GDPR, HIPAA, v.v.)
3. Cần tích hợp với các legacy systems
4. Yêu cầu performance đặc biệt
5. Cần kiểm soát hoàn toàn về security

**Nên sử dụng dịch vụ cloud khi:**
1. Không muốn quản lý infrastructure
2. Cần triển khai nhanh
3. Không có team security chuyên sâu
4. Cung cấp features sẵn có là đủ
5. Cần scalability tự động

## Tóm tắt

Phần này đã giới thiệu tổng quan về OAuth 2.0 và Authentication Server:

1. **OAuth 2.0** là một framework ủy quyền cho phép ứng dụng truy cập tài nguyên được bảo vệ mà không cần chia sẻ thông tin đăng nhập
2. **Các vai trò chính** bao gồm Resource Owner, Client, Authorization Server, và Resource Server
3. **Các grant types** phổ biến là Authorization Code, Implicit, Password Credentials, và Client Credentials
4. **OpenID Connect** mở rộng OAuth 2.0 để hỗ trợ authentication, cung cấp ID Token và UserInfo endpoint
5. **Authentication Server** là thành phần trung tâm quản lý danh tính và quyền truy cập, cung cấp nhiều lợi ích về bảo mật, developer productivity, và user experience

Phần tiếp theo sẽ đi sâu vào kiến trúc hệ thống và các thành phần chi tiết.
