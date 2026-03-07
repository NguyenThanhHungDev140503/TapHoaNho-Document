# Phần 6: Các biện pháp bảo mật tối ưu

## 6.1 Bảo vệ chống lại các lỗ hổng phổ biến

### 6.1.1 CSRF Protection

#### CSRF là gì?

CSRF (Cross-Site Request Forgery) là một lỗ hổng bảo mật cho phép attacker thực hiện các action không được phép thay mặt của user đã xác thực.

#### Tại sao cần bảo vệ CSRF?

**Vấn đề trong OAuth 2.0:**
- Attacker có thể lừa user gửi request đến Authorization Server
- Attacker có thể lấy authorization code hoặc token
- Attacker có thể cấp quyền truy cập cho ứng dụng của họ

#### Cách bảo vệ CSRF

##### 1. Sử dụng State Parameter

**Mô tả:** Client tạo một giá trị ngẫu nhiên (state) và gửi cùng với request ủy quyền. Authorization Server trả về state trong response. Client verify rằng state khớp với request ban đầu.

**Ví dụ:**
```csharp
// Client tạo state
var state = Guid.NewGuid().ToString();

// Gửi request ủy quyền với state
var properties = new AuthenticationProperties
{
    RedirectUri = Url.Action(nameof(Callback))
};

properties.Items["state"] = state;

return Challenge(properties, "oidc");

// Verify state trong callback
public async Task<IActionResult> Callback()
{
    var state = Request.Query["state"];
    var expectedState = HttpContext.Session.GetString("state");
    
    if (state != expectedState)
    {
        return BadRequest("Invalid state parameter");
    }
    
    // Tiếp tục xử lý
    var result = await HttpContext.AuthenticateAsync("oidc");
    // ...
}
```

##### 2. Sử dụng SameSite Cookie

**Mô tả:** Thiết lập SameSite attribute cho cookies để ngăn chặn CSRF attacks.

**Ví dụ:**
```csharp
services.AddAuthentication("Cookies")
    .AddCookie("Cookies", options =>
    {
        options.Cookie.SameSite = SameSiteMode.Strict;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    });
```

##### 3. Validate Referer Header

**Mô tả:** Kiểm tra Referer header để đảm bảo request đến từ cùng origin.

**Ví dụ:**
```csharp
public class CsrfProtectionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<CsrfProtectionMiddleware> _logger;
    
    public CsrfProtectionMiddleware(RequestDelegate next, ILogger<CsrfProtectionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var referer = context.Request.Headers["Referer"];
        
        if (string.IsNullOrEmpty(referer))
        {
            _logger.LogWarning("Missing Referer header");
            context.Response.StatusCode = 403;
            return;
        }
        
        var refererUri = new Uri(referer);
        
        if (!IsValidReferer(refererUri))
        {
            _logger.LogWarning("Invalid Referer: {Referer}", referer);
            context.Response.StatusCode = 403;
            return;
        }
        
        await _next(context);
    }
    
    private bool IsValidReferer(Uri refererUri)
    {
        // Kiểm tra xem referer có phải từ trusted origin không
        var trustedOrigins = new List<string>
        {
            "https://localhost:5001",
            "https://app.example.com"
        };
        
        var refererOrigin = $"{refererUri.Scheme}://{refererUri.Host}";
        return trustedOrigins.Contains(refererOrigin);
    }
}
```

### 6.1.2 XSS Protection

#### XSS là gì?

XSS (Cross-Site Scripting) là một lỗ hổng bảo mật cho phép attacker chèn script độc hại vào các trang web được xem bởi người dùng khác.

#### Tại sao cần bảo vệ XSS?

**Vấn đề trong OAuth 2.0:**
- Attacker có thể chèn script để lấy access token
- Attacker có thể chèn script để redirect user đến trang độc hại
- Attacker có thể chèn script để thực hiện các action không được phép

#### Cách bảo vệ XSS

##### 1. Sử dụng HttpOnly Cookies

**Mô tả:** Thiết lập HttpOnly attribute cho cookies để ngăn chặn JavaScript truy cập cookies.

**Ví dụ:**
```csharp
services.AddAuthentication("Cookies")
    .AddCookie("Cookies", options =>
    {
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    });
```

##### 2. Validate và Sanitize Input

**Mô tả:** Validate và sanitize tất cả input để ngăn chặn XSS attacks.

**Ví dụ:**
```csharp
public class InputSanitizer
{
    public static string Sanitize(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        // Loại bỏ các ký tự nguy hiểm
        var dangerousChars = new[] { '<', '>', '"', '\'', '&', '(', ')', ';', ':', '/', '\\', '\n', '\r' };
        foreach (var c in dangerousChars)
        {
            input = input.Replace(c.ToString(), string.Empty);
        }
        
        return input.Trim();
    }
}
```

##### 3. Sử dụng Content Security Policy

**Mô tả:** Thiết lập CSP để giới hạn các nguồn có thể tải tài nguyên.

**Ví dụ:**
```csharp
public void Configure(IApplicationBuilder app)
{
    app.Use(async (context, next) =>
    {
        context.Response.Headers.Add(
            "Content-Security-Policy",
            "default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data: https:picsum.photos; font-src 'self';"
        );
        await next();
    });
    
    app.UseStaticFiles();
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

##### 4. Encode Output

**Mô tả:** Encode tất cả output để ngăn chặn XSS attacks.

**Ví dụ:**
```csharp
public class OutputEncoder
{
    public static string Encode(string input)
    {
        if (string.IsNullOrEmpty(input))
        {
            return input;
        }
        
        return System.Web.HttpUtility.HtmlEncode(input);
    }
}

// Sử dụng trong Razor views
@OutputEncoder.Encode(Model.UserName)
```

### 6.1.3 Redirect URI Validation

#### Tại sao cần validate Redirect URI?

**Vấn đề:**
- Attacker có thể thay đổi redirect URI để redirect user đến trang độc hại
- Attacker có thể lấy authorization code bằng cách redirect đến server của họ

#### Cách validate Redirect URI

##### 1. Exact Match Validation

**Mô tả:** Redirect URI phải khớp chính xác với một trong các URIs được phép.

**Ví dụ:**
```csharp
public class CustomRedirectUriValidator : IRedirectUriValidator
{
    private readonly ILogger<CustomRedirectUriValidator> _logger;
    
    public CustomRedirectUriValidator(ILogger<CustomRedirectUriValidator> logger)
    {
        _logger = logger;
    }
    
    public async Task ValidateAsync(RedirectUriValidationContext context)
    {
        var redirectUri = context.Request.RedirectUri;
        
        if (string.IsNullOrEmpty(redirectUri))
        {
            _logger.LogWarning("Redirect URI is missing");
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI is required"
                });
            return;
        }
        
        // Exact match validation
        var allowedUris = context.Client.RedirectUris;
        if (!allowedUris.Contains(redirectUri))
        {
            _logger.LogWarning("Redirect URI {RedirectUri} is not allowed for client {ClientId}", 
                redirectUri, context.Client.ClientId);
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI is not allowed"
                });
            return;
        }
        
        await Task.CompletedTask;
    }
}
```

##### 2. Prefix Match Validation

**Mô tả:** Redirect URI phải bắt đầu với một trong các prefixes được phép.

**Ví dụ:**
```csharp
public class CustomRedirectUriValidator : IRedirectUriValidator
{
    public async Task ValidateAsync(RedirectUriValidationContext context)
    {
        var redirectUri = context.Request.RedirectUri;
        
        if (string.IsNullOrEmpty(redirectUri))
        {
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI is required"
                });
            return;
        }
        
        // Prefix match validation
        var allowedPrefixes = new List<string>
        {
            "https://localhost:5001/",
            "https://app.example.com/"
        };
        
        var isValid = allowedPrefixes.Any(prefix => redirectUri.StartsWith(prefix));
        
        if (!isValid)
        {
            _logger.LogWarning("Redirect URI {RedirectUri} does not match any allowed prefix", redirectUri);
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI is not allowed"
                });
            return;
        }
        
        await Task.CompletedTask;
    }
}
```

##### 3. Custom Validation Logic

**Ví dụ:**
```csharp
public class CustomRedirectUriValidator : IRedirectUriValidator
{
    public async Task ValidateAsync(RedirectUriValidationContext context)
    {
        var redirectUri = context.Request.RedirectUri;
        
        if (string.IsNullOrEmpty(redirectUri))
        {
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI is required"
                });
            return;
        }
        
        // Custom validation logic
        var uri = new Uri(redirectUri);
        
        // Kiểm tra scheme
        if (uri.Scheme != "https")
        {
            _logger.LogWarning("Redirect URI {RedirectUri} must use HTTPS", redirectUri);
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI must use HTTPS"
                });
            return;
        }
        
        // Kiểm tra host
        var allowedHosts = new List<string>
        {
            "localhost",
            "app.example.com"
        };
        
        if (!allowedHosts.Contains(uri.Host))
        {
            _logger.LogWarning("Redirect URI host {Host} is not allowed", uri.Host);
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI host is not allowed"
                });
            return;
        }
        
        // Kiểm tra path
        var allowedPaths = new List<string>
        {
            "/signin-oidc",
            "/signout-callback-oidc"
        };
        
        if (!allowedPaths.Contains(uri.AbsolutePath))
        {
            _logger.LogWarning("Redirect URI path {Path} is not allowed", uri.AbsolutePath);
            context.Result = new ValidationResult(
                new AuthorizationError
                {
                    Error = OAuth2Constants.AuthorizeErrors.InvalidRequest,
                    ErrorDescription = "Redirect URI path is not allowed"
                });
            return;
        }
        
        await Task.CompletedTask;
    }
}
```

### 6.1.4 State Parameter

#### Tại sao cần State Parameter?

**Vấn đề:**
- Ngăn chặn CSRF attacks
- Đảm bảo tính toàn vẹn của request ủy quyền
- Giúp client verify rằng response đến từ request ban đầu

#### Cách sử dụng State Parameter

##### 1. Tạo State Parameter

**Ví dụ:**
```csharp
public class AuthorizationController : Controller
{
    [HttpGet]
    public IActionResult Login()
    {
        // Tạo state ngẫu nhiên
        var state = Guid.NewGuid().ToString();
        
        // Lưu state vào session
        HttpContext.Session.SetString("state", state);
        
        var properties = new AuthenticationProperties
        {
            RedirectUri = Url.Action(nameof(Callback))
        };
        
        properties.Items["state"] = state;
        
        return Challenge(properties, "oidc");
    }
}
```

##### 2. Validate State Parameter

**Ví dụ:**
```csharp
[HttpGet]
public async Task<IActionResult> Callback()
{
    var state = Request.Query["state"];
    var expectedState = HttpContext.Session.GetString("state");
    
    if (string.IsNullOrEmpty(state) || string.IsNullOrEmpty(expectedState))
    {
        return BadRequest("State parameter is missing");
    }
    
    if (state != expectedState)
    {
        _logger.LogWarning("State parameter mismatch. Expected: {Expected}, Received: {Received}", 
            expectedState, state);
        return BadRequest("Invalid state parameter");
    }
    
    // Xóa state khỏi session
    HttpContext.Session.Remove("state");
    
    // Tiếp tục xử lý
    var result = await HttpContext.AuthenticateAsync("oidc");
    // ...
}
```

##### 3. Sử dụng Encrypted State

**Ví dụ:**
```csharp
public class StateService : IStateService
{
    private readonly IDataProtectionProvider _dataProtectionProvider;
    
    public StateService(IDataProtectionProvider dataProtectionProvider)
    {
        _dataProtectionProvider = dataProtectionProvider;
    }
    
    public string CreateState(string userId, string clientId)
    {
        var stateData = new StateData
        {
            UserId = userId,
            ClientId = clientId,
            Timestamp = DateTime.UtcNow,
            Nonce = Guid.NewGuid().ToString()
        };
        
        var protector = _dataProtectionProvider.CreateProtector("StatePurpose");
        var protectedData = protector.Protect(Encoding.UTF8.GetBytes(JsonSerializer.Serialize(stateData)));
        
        return Base64UrlEncode(protectedData);
    }
    
    public bool ValidateState(string state, string userId, string clientId)
    {
        try
        {
            var protector = _dataProtectionProvider.CreateProtector("StatePurpose");
            var unprotectedData = protector.Unprotect(Base64UrlDecode(state));
            var stateData = JsonSerializer.Deserialize<StateData>(Encoding.UTF8.GetString(unprotectedData));
            
            // Validate state data
            if (stateData.UserId != userId || stateData.ClientId != clientId)
            {
                return false;
            }
            
            // Kiểm tra expiration (5 phút)
            if ((DateTime.UtcNow - stateData.Timestamp).TotalMinutes > 5)
            {
                return false;
            }
            
            return true;
        }
        catch
        {
            return false;
        }
    }
    
    private string Base64UrlEncode(byte[] input)
    {
        return Convert.ToBase64String(input)
            .Replace('+', '-')
            .Replace('/', '_')
            .TrimEnd('=');
    }
    
    private byte[] Base64UrlDecode(string input)
    {
        var s = input;
        s = s.Replace('-', '+');
        s = s.Replace('_', '/');
        
        switch (s.Length % 4)
        {
            case 2: s += "=="; break;
            case 3: s += "="; break;
        }
        
        return Convert.FromBase64String(s);
    }
}

public class StateData
{
    public string UserId { get; set; }
    public string ClientId { get; set; }
    public DateTime Timestamp { get; set; }
    public string Nonce { get; set; }
}
```

## 6.2 Quản lý bí mật (Secrets Management)

### 6.2.1 Client Secrets

#### Tại sao cần bảo vệ Client Secrets?

**Vấn đề:**
- Client secret được sử dụng để authenticate client
- Nếu bị lộ, attacker có thể đóng vai trò client
- Attacker có thể lấy access token với quyền của client

#### Cách bảo vệ Client Secrets

##### 1. Sử dụng Environment Variables

**Ví dụ:**
```bash
# .env
CLIENT_SECRET=your-secret-key-here
```

```csharp
// appsettings.json
{
  "OAuth": {
    "ClientId": "web-client",
    "ClientSecret": "${CLIENT_SECRET}"
  }
}

// appsettings.Development.json
{
  "OAuth": {
    "ClientId": "web-client",
    "ClientSecret": "dev-secret-key-here"
  }
}
```

##### 2. Sử dụng Secret Management Service

**Ví dụ (Azure Key Vault):**
```csharp
public class SecretService : ISecretService
{
    private readonly SecretClient _secretClient;
    
    public SecretService(SecretClient secretClient)
    {
        _secretClient = secretClient;
    }
    
    public async Task<string> GetClientSecretAsync(string clientId)
    {
        var secretName = $"client-secret-{clientId}";
        var secret = await _secretClient.GetSecretAsync(secretName);
        
        return secret.Value;
    }
}

// Sử dụng
var clientSecret = await _secretService.GetClientSecretAsync("web-client");
```

**Ví dụ (AWS Secrets Manager):**
```csharp
public class SecretService : ISecretService
{
    private readonly IAmazonSecretsManager _secretsManager;
    
    public SecretService(IAmazonSecretsManager secretsManager)
    {
        _secretsManager = secretsManager;
    }
    
    public async Task<string> GetClientSecretAsync(string clientId)
    {
        var secretName = $"client-secret-{clientId}";
        var request = new GetSecretValueRequest
        {
            SecretId = secretName
        };
        
        var response = await _secretsManager.GetSecretValueAsync(request);
        return response.SecretString;
    }
}
```

##### 3. Sử dụng Hashed Secrets

**Ví dụ:**
```csharp
public class SecretHasher
{
    public static string HashSecret(string secret)
    {
        using (var sha256 = SHA256.Create())
        {
            var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(secret));
            return Convert.ToBase64String(hash);
        }
    }
}

// Cấu hình client với hashed secret
new Client
{
    ClientId = "web-client",
    ClientSecrets = { new Secret(SecretHasher.HashSecret("your-secret-key-here").Sha256()) }
}
```

##### 4. Rotate Secrets Thường xuyên

**Ví dụ:**
```csharp
public class SecretRotationService : ISecretRotationService
{
    private readonly ISecretService _secretService;
    private readonly IClientRepository _clientRepository;
    private readonly ILogger<SecretRotationService> _logger;
    
    public SecretRotationService(
        ISecretService secretService,
        IClientRepository clientRepository,
        ILogger<SecretRotationService> logger)
    {
        _secretService = secretService;
        _clientRepository = clientRepository;
        _logger = logger;
    }
    
    public async Task RotateClientSecretAsync(string clientId)
    {
        // Tạo secret mới
        var newSecret = GenerateSecureSecret();
        
        // Lưu secret mới
        await _secretService.SetClientSecretAsync(clientId, newSecret);
        
        // Cập nhật client với secret mới
        var client = await _clientRepository.GetByIdAsync(clientId);
        client.ClientSecrets.Clear();
        client.ClientSecrets.Add(new Secret(newSecret.Sha256()));
        await _clientRepository.UpdateAsync(client);
        
        _logger.LogInformation("Rotated secret for client {ClientId}", clientId);
    }
    
    private string GenerateSecureSecret()
    {
        var randomBytes = new byte[32];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(randomBytes);
        }
        return Convert.ToBase64String(randomBytes);
    }
}
```

### 6.2.2 Signing Keys

#### Tại sao cần bảo vệ Signing Keys?

**Vấn đề:**
- Signing keys được sử dụng để ký JWT tokens
- Nếu bị lộ, attacker có thể tạo token giả mạo
- Attacker có thể giả mạo là user hoặc client

#### Cách bảo vệ Signing Keys

##### 1. Sử dụng Asymmetric Keys

**Ví dụ:**
```csharp
// Tạo RSA key pair
var rsa = RSA.Create(2048);
var privateKey = new RsaSecurityKey(rsa);
var publicKey = new RsaSecurityKey(rsa.ExportParameters(false));

// Lưu private key an toàn
var privateKeyPem = "private_key.pem";
File.WriteAllBytes(privateKeyPem, privateKey.ExportPkcs8PrivateKey());

// Sử dụng private key để ký
var tokenDescriptor = new SecurityTokenDescriptor
{
    Subject = "1234567890",
    Expires = DateTime.UtcNow.AddHours(1),
    SigningCredentials = new SigningCredentials(privateKey, SecurityAlgorithms.RsaSha256Signature)
};
```

##### 2. Sử dụng Key Store

**Ví dụ (Azure Key Vault):**
```csharp
public class KeyVaultService : IKeyVaultService
{
    private readonly SecretClient _secretClient;
    
    public KeyVaultService(SecretClient secretClient)
    {
        _secretClient = secretClient;
    }
    
    public async Task<RsaSecurityKey> GetSigningKeyAsync()
    {
        var secret = await _secretClient.GetSecretAsync("signing-key");
        
        var privateKey = new RsaSecurityKey(
            Convert.FromBase64String(secret.Value),
            true,
            true);
        
        return privateKey;
    }
}
```

**Ví dụ (AWS KMS):**
```csharp
public class KmsService : IKmsService
{
    private readonly IAmazonKeyManagementService _kmsClient;
    
    public KmsService(IAmazonKeyManagementService kmsClient)
    {
        _kmsClient = kmsClient;
    }
    
    public async Task<RsaSecurityKey> GetSigningKeyAsync(string keyId)
    {
        var request = new GetPublicKeyRequest
        {
            KeyId = keyId
        };
        
        var response = await _kmsClient.GetPublicKeyAsync(request);
        
        var publicKey = new RsaSecurityKey(
            Convert.FromBase64String(response.PublicKey),
            false,
            true);
        
        return publicKey;
    }
}
```

##### 3. Sử dụng Hardware Security Module (HSM)

**Ví dụ:**
```csharp
public class HsmService : IHsmService
{
    private readonly IHardwareSecurityModule _hsm;
    
    public HsmService(IHardwareSecurityModule hsm)
    {
        _hsm = hsm;
    }
    
    public async Task<byte[]> SignDataAsync(byte[] data)
    {
        var result = await _hsm.SignAsync("signing-key", data);
        return result;
    }
}
```

### 6.2.3 Key Rotation

#### Tại sao cần Rotate Keys?

**Vấn đề:**
- Giảm thiệt hại nếu key bị lộ
- Tuân thủ các yêu cầu compliance (GDPR, PCI DSS)
- Giảm thời gian key được sử dụng

#### Cách Rotate Keys

##### 1. Manual Key Rotation

**Ví dụ:**
```csharp
public class KeyRotationService : IKeyRotationService
{
    public async Task RotateSigningKeysAsync()
    {
        // Tạo key pair mới
        var newRsa = RSA.Create(2048);
        var newPrivateKey = new RsaSecurityKey(newRsa);
        var newPublicKey = new RsaSecurityKey(newRsa.ExportParameters(false));
        
        // Lưu keys mới
        var newPrivateKeyPem = $"private_key_{DateTime.UtcNow:yyyyMMdd}.pem";
        File.WriteAllBytes(newPrivateKeyPem, newPrivateKey.ExportPkcs8PrivateKey());
        
        var newPublicKeyPem = $"public_key_{DateTime.UtcNow:yyyyMMdd}.pem";
        File.WriteAllBytes(newPublicKeyPem, newPublicKey.ExportSubjectPublicKeyInfo());
        
        // Cập nhật cấu hình
        UpdateSigningKeys(newPrivateKey, newPublicKey);
        
        _logger.LogInformation("Rotated signing keys");
    }
    
    private void UpdateSigningKeys(RsaSecurityKey privateKey, RsaSecurityKey publicKey)
    {
        // Cập nhật cấu hình IdentityServer
        // ...
    }
}
```

##### 2. Automatic Key Rotation

**Ví dụ:**
```csharp
public class KeyRotationScheduler
{
    private readonly IKeyRotationService _keyRotationService;
    private readonly ILogger<KeyRotationScheduler> _logger;
    
    public KeyRotationScheduler(IKeyRotationService keyRotationService, ILogger<KeyRotationScheduler> logger)
    {
        _keyRotationService = keyRotationService;
        _logger = logger;
    }
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(TimeSpan.FromDays(90), cancellationToken);
                
                await _keyRotationService.RotateSigningKeysAsync();
                
                _logger.LogInformation("Signing keys rotated successfully");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error rotating signing keys");
            }
        }
    }
}
```

##### 3. Multiple Active Keys

**Ví dụ:**
```csharp
public class MultiKeySigningCredentials : SigningCredentials
{
    private readonly List<SigningCredentials> _credentials;
    
    public MultiKeySigningCredentials(params IEnumerable<SigningCredentials> credentials)
    {
        _credentials = credentials.ToList();
    }
    
    public string Sign(string algorithm, params IEnumerable<string> parts)
    {
        // Sử dụng key đầu tiên
        return _credentials[0].Sign(algorithm, parts);
    }
    
    public void AddKey(SigningCredentials credentials)
    {
        _credentials.Add(credentials);
    }
    
    public void RemoveKey(SigningCredentials credentials)
    {
        _credentials.Remove(credentials);
    }
}
```

## 6.3 Mã hóa và bảo mật dữ liệu

### 6.3.1 HTTPS/TLS

#### Tại sao cần HTTPS/TLS?

**Vấn đề:**
- Ngăn chặn man-in-the-middle attacks
- Mã hóa dữ liệu trong quá trình truyền
- Đảm bảo tính toàn vẹn của dữ liệu

#### Cấu hình HTTPS/TLS

##### 1. Sử dụng HTTPS trong Production

**Ví dụ:**
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseHsts();
    }
    
    app.UseHttpsRedirection();
    app.UseStaticFiles();
    app.UseRouting();
    
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

##### 2. Cấu hình TLS 1.2+

**Ví dụ:**
```csharp
public static IWebHostBuilder CreateWebHostBuilder(string[] args)
{
    return WebHost.CreateDefaultBuilder(args)
        .UseKestrel(options =>
        {
            options.Listen(IPAddress.Any, 5000, listenOptions =>
            {
                listenOptions.UseHttps("certificate.pfx", "certificate-password");
                listenOptions.Protocols = HttpProtocols.Tls12 | HttpProtocols.Tls13;
            });
        });
}
```

##### 3. Sử dụng Valid SSL Certificates

**Ví dụ:**
```csharp
public static IWebHostBuilder CreateWebHostBuilder(string[] args)
{
    return WebHost.CreateDefaultBuilder(args)
        .UseKestrel(options =>
        {
            options.ConfigureHttpsDefaults(httpsOptions =>
            {
                httpsOptions.ServerCertificate = new X509Certificate2("certificate.pfx", "certificate-password");
                httpsOptions.ClientCertificateMode = ClientCertificateMode.RequireCertificate;
                httpsOptions.CheckCertificateRevocation = true;
                httpsOptions.ClientCertificateRevocationMode = X509RevocationMode.Online;
            });
        });
}
```

### 6.3.2 Hashing Passwords

#### Tại sao cần Hash Passwords?

**Vấn đề:**
- Không bao giờ lưu trữ password dạng plain text
- Ngăn chặn attacker đọc password nếu database bị lộ
- Ngăn chặn attacker biết password của user

#### Cách Hash Passwords

##### 1. Sử dụng Strong Hashing Algorithm

**Ví dụ:**
```csharp
public class PasswordHasher : IPasswordHasher
{
    public string HashPassword(string password)
    {
        // Sử dụng ASP.NET Core Identity's default password hasher
        var passwordHasher = new PasswordHasher<ApplicationUser>();
        return passwordHasher.HashPassword(password);
    }
}
```

##### 2. Sử dụng Salt

**Ví dụ:**
```csharp
public class PasswordHasher : IPasswordHasher
{
    public string HashPassword(string password)
    {
        // Tạo salt ngẫu nhiên
        var salt = GenerateSalt();
        
        // Hash password với salt
        var hash = HashPasswordWithSalt(password, salt);
        
        // Lưu salt cùng với hash
        return $"{salt}.{hash}";
    }
    
    private string GenerateSalt()
    {
        var randomBytes = new byte[16];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(randomBytes);
        }
        return Convert.ToBase64String(randomBytes);
    }
    
    private string HashPasswordWithSalt(string password, string salt)
    {
        using (var sha256 = SHA256.Create())
        {
            var saltedPassword = password + salt;
            var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(saltedPassword));
            return Convert.ToBase64String(hash);
        }
    }
}
```

##### 3. Sử dụng Iterative Hashing

**Ví dụ:**
```csharp
public class PasswordHasher : IPasswordHasher
{
    private const int Iterations = 10000;
    private const int HashSize = 256;
    
    public string HashPassword(string password)
    {
        var salt = GenerateSalt();
        var hash = password;
        
        for (int i = 0; i < Iterations; i++)
        {
            hash = HashWithIterations(hash, salt, 1);
        }
        
        return $"{salt}.{hash}";
    }
    
    private string HashWithIterations(string input, string salt, int iterations)
    {
        using (var pbkdf2 = new Rfc2898DeriveBytes())
        {
            var hash = pbkdf2.DeriveKey(
                salt,
                iterations,
                HashSize,
                HashAlgorithmName.SHA256);
            var bytes = pbkdf2.GetBytes(input);
            return Convert.ToBase64String(bytes);
        }
    }
    
    private string GenerateSalt()
    {
        var randomBytes = new byte[16];
        using (var rng = RandomNumberGenerator.Create())
        {
            rng.GetBytes(randomBytes);
        }
        return Convert.ToBase64String(randomBytes);
    }
}
```

### 6.3.3 Encryption at Rest

#### Tại sao cần Encryption at Rest?

**Vấn đề:**
- Ngăn chặn attacker đọc dữ liệu nhạy cảm nếu database bị lộ
- Tuân thủ các yêu cầu compliance (GDPR, PCI DSS)
- Bảo vệ thông tin nhạy cảm của user

#### Cách Encrypt Data at Rest

##### 1. Sử dụng Transparent Data Encryption (TDE)

**Ví dụ (SQL Server):**
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Cấu hình TDE cho các entities nhạy cảm
        modelBuilder.Entity<User>()
            .Property(u => u.Email)
            .IsEncrypted();
        
        modelBuilder.Entity<User>()
            .Property(u => u.PhoneNumber)
            .IsEncrypted();
        
        modelBuilder.Entity<User>()
            .Property(u => u.Ssn)
            .IsEncrypted();
    }
}
```

##### 2. Sử dụng Application-Level Encryption

**Ví dụ:**
```csharp
public class EncryptionService : IEncryptionService
{
    private readonly IDataProtector _dataProtector;
    
    public EncryptionService(IDataProtector dataProtector)
    {
        _dataProtector = dataProtector;
    }
    
    public string Encrypt(string plainText)
    {
        var plainBytes = Encoding.UTF8.GetBytes(plainText);
        var protectedBytes = _dataProtector.Protect(plainBytes);
        return Convert.ToBase64String(protectedBytes);
    }
    
    public string Decrypt(string cipherText)
    {
        var protectedBytes = Convert.FromBase64String(cipherText);
        var plainBytes = _dataProtector.Unprotect(protectedBytes);
        return Encoding.UTF8.GetString(plainBytes);
    }
}

// Sử dụng
var encryptedEmail = _encryptionService.Encrypt(user.Email);
var decryptedEmail = _encryptionService.Decrypt(encryptedEmail);
```

##### 3. Sử dụng Column-Level Encryption

**Ví dụ:**
```csharp
public class User
{
    public int Id { get; set; }
    
    [Column(Type = "nvarchar(max)", Name = "Email")]
    public string Email { get; set; }
    
    [NotMapped]
    public string EmailEncrypted { get; set; }
    
    public void SetEmail(string email, IEncryptionService encryptionService)
    {
        Email = email;
        EmailEncrypted = encryptionService.Encrypt(email);
    }
    
    public string GetEmail(IEncryptionService encryptionService)
    {
        if (!string.IsNullOrEmpty(EmailEncrypted))
        {
            return encryptionService.Decrypt(EmailEncrypted);
        }
        return Email;
    }
}
```

## 6.4 Rate Limiting và Throttling

#### Tại sao cần Rate Limiting?

**Vấn đề:**
- Ngăn chặn brute force attacks
- Ngăn chặn DDoS attacks
- Bảo vệ resource availability
- Ngăn chặn abuse

#### Cách thực hiện Rate Limiting

##### 1. Sử dụng ASP.NET Core Rate Limiting Middleware

**Ví dụ:**
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddRateLimiter(options =>
    {
        options.GlobalLimiter = PartitionedRateLimiterBuilder.Create
            .PerClientAllowanceLimiter(options)
            .Build();
    });
}

public void Configure(IApplicationBuilder app)
{
    app.UseRateLimiter();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}

// Cấu hình rate limiting
services.AddRateLimiter(options =>
{
    options.AddPolicy("TokenEndpoint", policy =>
        policy.PermitLimit = 100, // 100 requests mỗi phút
        policy.Window = TimeSpan.FromMinutes(1),
        policy.SlidingWindow = true);
});
```

##### 2. Sử dụng IP-Based Rate Limiting

**Ví dụ:**
```csharp
public class IpRateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<IpRateLimitMiddleware> _logger;
    private readonly IRateLimitStore _rateLimitStore;
    
    public IpRateLimitMiddleware(
        RequestDelegate next,
        ILogger<IpRateLimitMiddleware> logger,
        IRateLimitStore rateLimitStore)
    {
        _next = next;
        _logger = logger;
        _rateLimitStore = rateLimitStore;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var ipAddress = context.Connection.RemoteIpAddress;
        var endpoint = context.Request.Path;
        
        // Kiểm tra rate limit
        if (await _rateLimitStore.IsRateLimitedAsync(ipAddress, endpoint))
        {
            _logger.LogWarning("Rate limit exceeded for IP {IpAddress} on endpoint {Endpoint}", 
                ipAddress, endpoint);
            context.Response.StatusCode = 429;
            return;
        }
        
        await _next(context);
    }
}
```

##### 3. Sử dụng User-Based Rate Limiting

**Ví dụ:**
```csharp
public class UserRateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<UserRateLimitMiddleware> _logger;
    private readonly IRateLimitStore _rateLimitStore;
    
    public UserRateLimitMiddleware(
        RequestDelegate next,
        ILogger<UserRateLimitMiddleware> logger,
        IRateLimitStore rateLimitStore)
    {
        _next = next;
        _logger = logger;
        _rateLimitStore = rateLimitStore;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var userId = context.User?.FindFirst(JwtRegisteredClaimNames.Sub)?.Value;
        var endpoint = context.Request.Path;
        
        if (string.IsNullOrEmpty(userId))
        {
            await _next(context);
            return;
        }
        
        // Kiểm tra rate limit cho user
        if (await _rateLimitStore.IsUserRateLimitedAsync(userId, endpoint))
        {
            _logger.LogWarning("Rate limit exceeded for user {UserId} on endpoint {Endpoint}", 
                userId, endpoint);
            context.Response.StatusCode = 429;
            return;
        }
        
        await _next(context);
    }
}
```

## 6.5 Auditing và Logging

### 6.5.1 Event Logging

#### Tại sao cần Event Logging?

**Vấn đề:**
- Theo dõi các hoạt động bảo mật
- Phát hiện các hoạt động đáng ngờ
- Giúp debug và điều tra
- Tuân thủ các yêu cầu compliance

#### Các sự kiện cần log

##### 1. Authentication Events

**Ví dụ:**
```csharp
public class AuditEvent
{
    public string UserId { get; set; }
    public string ClientId { get; set; }
    public string EventType { get; set; }
    public string Description { get; set; }
    public DateTime Timestamp { get; set; }
    public string IpAddress { get; set; }
    public string UserAgent { get; set; }
}

public class AuditService : IAuditService
{
    private readonly IAuditEventRepository _auditEventRepository;
    
    public AuditService(IAuditEventRepository auditEventRepository)
    {
        _auditEventRepository = auditEventRepository;
    }
    
    public async Task LogLoginAsync(string userId, string clientId, string ipAddress, string userAgent)
    {
        var auditEvent = new AuditEvent
        {
            UserId = userId,
            ClientId = clientId,
            EventType = "Login",
            Description = "User logged in",
            Timestamp = DateTime.UtcNow,
            IpAddress = ipAddress,
            UserAgent = userAgent
        };
        
        await _auditEventRepository.AddAsync(auditEvent);
    }
    
    public async Task LogLogoutAsync(string userId, string clientId)
    {
        var auditEvent = new AuditEvent
        {
            UserId = userId,
            ClientId = clientId,
            EventType = "Logout",
            Description = "User logged out",
            Timestamp = DateTime.UtcNow
        };
        
        await _auditEventRepository.AddAsync(auditEvent);
    }
    
    public async Task LogTokenIssuedAsync(string userId, string clientId, string tokenType)
    {
        var auditEvent = new AuditEvent
        {
            UserId = userId,
            ClientId = clientId,
            EventType = "TokenIssued",
            Description = $"Token of type {tokenType} issued",
            Timestamp = DateTime.UtcNow
        };
        
        await _auditEventRepository.AddAsync(auditEvent);
    }
    
    public async Task LogTokenRefreshedAsync(string userId, string clientId)
    {
        var auditEvent = new AuditEvent
        {
            UserId = userId,
            ClientId = clientId,
            EventType = "TokenRefreshed",
            Description = "Access token refreshed",
            Timestamp = DateTime.UtcNow
        };
        
        await _auditEventRepository.AddAsync(auditEvent);
    }
}
```

##### 2. Authorization Events

**Ví dụ:**
```csharp
public async Task LogAuthorizationGrantedAsync(string userId, string clientId, List<string> scopes)
{
    var auditEvent = new AuditEvent
    {
        UserId = userId,
        ClientId = clientId,
        EventType = "AuthorizationGranted",
        Description = $"Authorization granted with scopes: {string.Join(", ", scopes)}",
        Timestamp = DateTime.UtcNow
    };
    
    await _auditEventRepository.AddAsync(auditEvent);
}

public async Task LogAuthorizationDeniedAsync(string userId, string clientId, string reason)
{
    var auditEvent = new AuditEvent
    {
        UserId = userId,
        ClientId = clientId,
        EventType = "AuthorizationDenied",
        Description = $"Authorization denied: {reason}",
        Timestamp = DateTime.UtcNow
    };
    
    await _auditEventRepository.AddAsync(auditEvent);
}
```

##### 3. Security Events

**Ví dụ:**
```csharp
public async Task LogInvalidTokenAttemptAsync(string token, string ipAddress, string userAgent)
{
    var auditEvent = new AuditEvent
    {
        UserId = null, // Unknown user
        ClientId = null, // Unknown client
        EventType = "InvalidTokenAttempt",
        Description = "Invalid access token attempt",
        Timestamp = DateTime.UtcNow,
        IpAddress = ipAddress,
        UserAgent = userAgent
    };
    
    await _auditEventRepository.AddAsync(auditEvent);
}

public async Task LogRateLimitExceededAsync(string ipAddress, string endpoint)
{
    var auditEvent = new AuditEvent
    {
        UserId = null,
        ClientId = null,
        EventType = "RateLimitExceeded",
        Description = $"Rate limit exceeded for endpoint {endpoint}",
        Timestamp = DateTime.UtcNow,
        IpAddress = ipAddress
    };
    
    await _auditEventRepository.AddAsync(auditEvent);
}
```

### 6.5.2 Security Monitoring

#### Tại sao cần Security Monitoring?

**Vấn đề:**
- Phát hiện các hoạt động đáng ngờ trong thời gian thực
- Giúp phản ứng nhanh với các threats
- Cung cấp thông tin cho điều tra và debug
- Giúp cải thiện bảo mật theo thời gian

#### Các chỉ số cần theo dõi

##### 1. Authentication Metrics

**Ví dụ:**
```csharp
public class AuthenticationMetrics
{
    public int SuccessfulLogins { get; set; }
    public int FailedLogins { get; set; }
    public int ActiveSessions { get; set; }
    public int TokensIssued { get; set; }
    public int TokensRefreshed { get; set; }
}

public class MetricsService : IMetricsService
{
    private readonly IMetricsRepository _metricsRepository;
    
    public MetricsService(IMetricsRepository metricsRepository)
    {
        _metricsRepository = metricsRepository;
    }
    
    public async Task IncrementSuccessfulLoginsAsync()
    {
        await _metricsRepository.IncrementMetricAsync("SuccessfulLogins");
    }
    
    public async Task IncrementFailedLoginsAsync()
    {
        await _metricsRepository.IncrementMetricAsync("FailedLogins");
    }
    
    public async Task RecordTokenIssuedAsync()
    {
        await _metricsRepository.IncrementMetricAsync("TokensIssued");
    }
    
    public async Task<AuthenticationMetrics> GetMetricsAsync()
    {
        return await _metricsRepository.GetAuthenticationMetricsAsync();
    }
}
```

##### 2. Security Metrics

**Ví dụ:**
```csharp
public class SecurityMetrics
{
    public int InvalidTokenAttempts { get; set; }
    public int RateLimitExceeded { get; set; }
    public int CsrfAttempts { get; set; }
    public int SuspiciousActivities { get; set; }
}

public class SecurityMetricsService : ISecurityMetricsService
{
    private readonly ISecurityMetricsRepository _securityMetricsRepository;
    
    public SecurityMetricsService(ISecurityMetricsRepository securityMetricsRepository)
    {
        _securityMetricsRepository = securityMetricsRepository;
    }
    
    public async Task RecordInvalidTokenAttemptAsync()
    {
        await _securityMetricsRepository.IncrementMetricAsync("InvalidTokenAttempts");
    }
    
    public async Task RecordRateLimitExceededAsync()
    {
        await _securityMetricsRepository.IncrementMetricAsync("RateLimitExceeded");
    }
    
    public async Task RecordCsrfAttemptAsync()
    {
        await _securityMetricsRepository.IncrementMetricAsync("CsrfAttempts");
    }
    
    public async Task<SecurityMetrics> GetMetricsAsync()
    {
        return await _securityMetricsRepository.GetSecurityMetricsAsync();
    }
}
```

##### 3. Alerting

**Ví dụ:**
```csharp
public class AlertService : IAlertService
{
    private readonly IEmailService _emailService;
    private readonly ISmsService _smsService;
    private readonly ILogger<AlertService> _logger;
    
    public AlertService(
        IEmailService emailService,
        ISmsService smsService,
        ILogger<AlertService> logger)
    {
        _emailService = emailService;
        _smsService = smsService;
        _logger = logger;
    }
    
    public async Task SendAlertAsync(string alertType, string message)
    {
        var alert = new Alert
        {
            Type = alertType,
            Message = message,
            Timestamp = DateTime.UtcNow
        };
        
        // Lưu alert vào database
        await _alertRepository.AddAsync(alert);
        
        // Gửi email notification
        await _emailService.SendAlertEmailAsync(alert);
        
        // Gửi SMS notification cho các alerts quan trọng
        if (alertType == "Critical")
        {
            await _smsService.SendAlertSmsAsync(alert);
        }
        
        _logger.LogInformation("Alert sent: {Type} - {Message}", alertType, message);
    }
    
    public async Task CheckAndSendAlertsAsync()
    {
        var securityMetrics = await _securityMetricsService.GetMetricsAsync();
        
        // Kiểm tra invalid token attempts
        if (securityMetrics.InvalidTokenAttempts > 10)
        {
            await SendAlertAsync("Warning", 
                $"High number of invalid token attempts detected: {securityMetrics.InvalidTokenAttempts}");
        }
        
        // Kiểm tra rate limit exceeded
        if (securityMetrics.RateLimitExceeded > 100)
        {
            await SendAlertAsync("Critical", 
                $"Rate limit exceeded {securityMetrics.RateLimitExceeded} times");
        }
        
        // Kiểm tra CSRF attempts
        if (securityMetrics.CsrfAttempts > 5)
        {
            await SendAlertAsync("Warning", 
                $"Multiple CSRF attempts detected: {securityMetrics.CsrfAttempts}");
        }
    }
}
```

### 6.5.3 Compliance

#### Các yêu cầu compliance phổ biến

##### 1. GDPR (General Data Protection Regulation)

**Yêu cầu chính:**
- Right to be forgotten (quyền xóa dữ liệu)
- Data portability (xuất dữ liệu)
- Data minimization (chỉ lưu trữ dữ liệu cần thiết)
- Data protection by design and by default
- Data breach notification (thông báo vi phạm)

**Triển khai:**
```csharp
public class GdprService : IGdprService
{
    private readonly IUserRepository _userRepository;
    private readonly ITokenService _tokenService;
    private readonly IAuditService _auditService;
    
    public GdprService(
        IUserRepository userRepository,
        ITokenService tokenService,
        IAuditService auditService)
    {
        _userRepository = userRepository;
        _tokenService = tokenService;
        _auditService = auditService;
    }
    
    public async Task HandleDataDeletionRequestAsync(string userId)
    {
        // Thu hồi tất cả tokens của user
        await _tokenService.RevokeAllUserTokensAsync(userId);
        
        // Xóa user account
        await _userRepository.DeleteAsync(userId);
        
        // Log deletion
        await _auditService.LogDataDeletionAsync(userId);
    }
    
    public async Task ExportUserDataAsync(string userId)
    {
        // Lấy tất cả dữ liệu của user
        var userData = await _userRepository.GetAllUserDataAsync(userId);
        
        // Export ra file
        var exportData = new UserDataExport
        {
            UserId = userId,
            ExportDate = DateTime.UtcNow,
            Data = userData
        };
        
        await _exportRepository.AddAsync(exportData);
        
        // Log export
        await _auditService.LogDataExportAsync(userId);
    }
}
```

##### 2. PCI DSS (Payment Card Industry Data Security Standard)

**Yêu cầu chính:**
- Protect cardholder data
- Maintain secure systems
- Regular security testing
- Network security controls
- Information security policy

**Triển khai:**
```csharp
public class PciDssService : IPciDssService
{
    public async Task LogCardDataAccessAsync(string userId, string cardData, string action)
    {
        var auditEvent = new AuditEvent
        {
            UserId = userId,
            EventType = "CardDataAccess",
            Description = $"Card data accessed: {action}",
            Timestamp = DateTime.UtcNow,
            IsSensitive = true
        };
        
        await _auditEventRepository.AddAsync(auditEvent);
    }
    
    public async Task EncryptCardDataAsync(string cardData)
    {
        // Mã hóa dữ liệu thẻ theo chuẩn PCI DSS
        var encryptedData = _encryptionService.Encrypt(cardData);
        
        await _cardDataRepository.SaveAsync(encryptedData);
    }
}
```

##### 3. SOC 2 (Service Organization Control 2)

**Yêu cầu chính:**
- Access control
- Incident management
- Compliance management
- Risk assessment
- Security awareness training

**Triển khai:**
```csharp
public class Soc2Service : ISoc2Service
{
    public async Task LogAccessAsync(string userId, string resource, string action)
    {
        var auditEvent = new AuditEvent
        {
            UserId = userId,
            EventType = "ResourceAccess",
            Description = $"Access to resource: {resource}, Action: {action}",
            Timestamp = DateTime.UtcNow
        };
        
        await _auditEventRepository.AddAsync(auditEvent);
    }
    
    public async Task LogIncidentAsync(string incidentType, string description, string severity)
    {
        var incident = new Incident
        {
            Type = incidentType,
            Description = description,
            Severity = severity,
            Status = "Open",
            CreatedAt = DateTime.UtcNow
        };
        
        await _incidentRepository.AddAsync(incident);
        
        // Gửi alert cho security team
        await _alertService.SendAlertAsync(severity, description);
    }
}
```

## Tóm tắt

Phần này đã mô tả chi tiết các biện pháp bảo mật tối ưu:

1. **Bảo vệ chống lại các lỗ hổng phổ biến:** CSRF Protection, XSS Protection, Redirect URI validation, State parameter
2. **Quản lý bí mật:** Client secrets, Signing keys, Key rotation
3. **Mã hóa và bảo mật dữ liệu:** HTTPS/TLS, Hashing passwords, Encryption at rest
4. **Rate Limiting và Throttling:** Các chiến lược và triển khai
5. **Auditing và Logging:** Event logging, Security monitoring, Compliance

Phần tiếp theo sẽ đi sâu vào triển khai với .NET.
