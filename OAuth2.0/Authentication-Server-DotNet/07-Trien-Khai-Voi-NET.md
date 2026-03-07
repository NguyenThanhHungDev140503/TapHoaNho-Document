# Phần 7: Triển khai với .NET

## 7.1 Tổng quan về ASP.NET Core Identity

### 7.1.1 ASP.NET Core Identity là gì?

ASP.NET Core Identity là một framework quản lý danh tính được tích hợp sẵn trong ASP.NET Core, cung cấp:

- User management (quản lý người dùng)
- Role management (quản lý vai trò)
- Password hashing (mã hóa mật khẩu)
- Token-based authentication (xác thực dựa trên token)
- External login providers (nhà cung cấp đăng nhập bên ngoài)

### 7.1.2 Tích hợp ASP.NET Core Identity với OAuth 2.0

#### Tại sao cần tích hợp?

**Lợi ích:**
- Sử dụng các tính năng có sẵn của ASP.NET Core Identity
- Giảm thời gian phát triển
- Tăng tính bảo mật
- Dễ dàng mở rộng

#### Cách tích hợp

##### 1. Cài đặt ASP.NET Core Identity

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
```

##### 2. Tạo User và Role Entities

```csharp
// ApplicationUser.cs
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
    
    public ICollection<IdentityUserClaim<string>> Claims { get; set; }
    public ICollection<IdentityUserLogin<string>> Logins { get; set; }
    public ICollection<IdentityUserToken<string>> Tokens { get; set; }
    public ICollection<IdentityUserRole<string>> Roles { get; set; }
}

// ApplicationRole.cs
public class ApplicationRole : IdentityRole
{
    public string Description { get; set; }
}
```

##### 3. Tạo DbContext

```csharp
// ApplicationDbContext.cs
public class ApplicationDbContext : IdentityDbContext<ApplicationUser, ApplicationRole, string>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Cấu hình các entities
        builder.Entity<ApplicationUser>(entity =>
        {
            entity.ToTable("Users");
            entity.Property(u => u.FirstName).HasMaxLength(100);
            entity.Property(u => u.LastName).HasMaxLength(100);
        });
        
        builder.Entity<ApplicationRole>(entity =>
        {
            entity.ToTable("Roles");
            entity.Property(r => r.Description).HasMaxLength(200);
        });
    }
}
```

##### 4. Cấu hình Dependency Injection

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // Cấu hình ASP.NET Core Identity
    services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
    {
        // Password options
        options.Password.RequireDigit = true;
        options.Password.RequireLowercase = true;
        options.Password.RequireUppercase = true;
        options.Password.RequireNonAlphanumeric = true;
        options.Password.RequiredLength = 8;
        
        // Lockout options
        options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
        options.Lockout.MaxFailedAccessAttempts = 5;
        options.Lockout.AllowedForNewUsers = true;
        
        // User options
        options.User.RequireUniqueEmail = true;
    })
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();
    
    // Cấu hình authentication
    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = IdentityConstants.ApplicationScheme;
        options.DefaultChallengeScheme = IdentityConstants.ApplicationScheme;
    })
    .AddCookie(IdentityConstants.ApplicationScheme, options =>
    {
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Strict;
        options.ExpireTimeSpan = TimeSpan.FromDays(7);
        options.SlidingExpiration = true;
    });
    
    // Cấu hình DbContext
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
}
```

##### 5. Tạo Database

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

## 7.2 Duende IdentityServer

### 7.2.1 Tổng quan về Duende IdentityServer

Duende IdentityServer là một framework cho phép xây dựng Authorization Server và OpenID Connect Provider trên nền tảng ASP.NET Core. Nó hỗ trợ đầy đủ các chuẩn OAuth 2.0 và OpenID Connect.

#### Tại sao chọn Duende IdentityServer?

**Lợi ích:**
- Hỗ trợ đầy đủ OAuth 2.0 và OpenID Connect
- Tích hợp dễ dàng với ASP.NET Core Identity
- Mã nguồn mở (với license thương mại)
- Được sử dụng rộng rãi trong cộng đồng .NET
- Tài liệu chi tiết và cộng đồng hỗ trợ tốt

#### Cài đặt Duende IdentityServer

```bash
dotnet add package Duende.IdentityServer.EntityFramework.Storage
dotnet add package Duende.IdentityServer.AspNetIdentity
```

### 7.2.2 Cấu hình cơ bản

##### 1. Cấu hình Duende IdentityServer

```csharp
// Startup.cs
public void ConfigureServices(IServiceCollection services)
{
    // Cấu hình ASP.NET Core Identity
    services.AddIdentity<ApplicationUser, ApplicationRole>()
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultTokenProviders();
    
    // Cấu hình Duende IdentityServer
    var builder = services.AddIdentityServer(options =>
    {
        options.Events.RaiseErrorEvents = true;
        options.Events.RaiseInformationEvents = true;
        options.Events.RaiseFailureEvents = true;
        options.Events.RaiseSuccessEvents = true;
        
        // Cấu hình issuer
        options.IssuerUri = "https://localhost:5001";
    })
    .AddAspNetIdentity<ApplicationUser>()
    .AddConfigurationStore(options =>
    {
        options.ConfigureDbContext = builder =>
            builder.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                sql => sql.MigrationsAssembly(typeof(Startup).Assembly.GetName().Name));
    })
    .AddOperationalStore(options =>
    {
        options.ConfigureDbContext = builder =>
            builder.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                sql => sql.MigrationsAssembly(typeof(Startup).Assembly.GetName().Name));
        
        options.EnableTokenCleanup = true;
        options.TokenCleanupInterval = 3600; // 1 giờ
    });
    
    // Cấu hình signing credentials
    builder.AddSigningCredential(new X509Certificate2("certificate.pfx", "certificate-password"));
}
```

##### 2. Cấu hình Middleware

```csharp
// Startup.cs
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
        app.UseDatabaseErrorPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
        app.UseHsts();
    }
    
    app.UseHttpsRedirection();
    app.UseStaticFiles();
    
    app.UseRouting();
    
    app.UseIdentityServer();
    app.UseAuthorization();
    
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapRazorPages();
        endpoints.MapControllers();
    });
}
```

##### 3. Tạo Migrations

```bash
dotnet ef migrations add AddIdentityServerConfiguration -c ConfigurationDbContext
dotnet ef migrations add AddIdentityServerPersistedGrants -c PersistedGrantDbContext
dotnet ef database update -c ConfigurationDbContext
dotnet ef database update -c PersistedGrantDbContext
```

### 7.2.3 Client Configuration

#### Các loại Client

##### 1. Confidential Client

**Mô tả:** Client có thể giữ bí mật an toàn (server-side applications)

**Ví dụ:**
```csharp
public static class Clients
{
    public static IEnumerable<Client> Get()
    {
        return new List<Client>
        {
            new Client
            {
                ClientId = "web-client",
                ClientName = "Web Client Application",
                ClientSecrets = { new Secret("secret".Sha256()) },
                AllowedGrantTypes = GrantTypes.Code,
                RequirePkce = true,
                RequireClientSecret = true,
                RedirectUris = { "https://localhost:5001/signin-oidc" },
                PostLogoutRedirectUris = { "https://localhost:5001/signout-callback-oidc" },
                AllowedScopes = { "openid", "profile", "email", "api1" },
                AllowOfflineAccess = true,
                AccessTokenLifetime = 3600, // 1 giờ
                IdentityTokenLifetime = 300, // 5 phút
                RefreshTokenUsage = TokenUsage.OneTimeOnly,
                RefreshTokenExpiration = TokenExpiration.Sliding,
                SlidingRefreshTokenLifetime = 2592000, // 30 ngày
                AbsoluteRefreshTokenLifetime = 7776000, // 90 ngày
                AlwaysSendClientClaims = true,
                ClientClaimsPrefix = "client_"
            }
        };
    }
}
```

##### 2. Public Client

**Mô tả:** Client không thể giữ bí mật an toàn (native apps, SPAs)

**Ví dụ:**
```csharp
new Client
{
    ClientId = "spa-client",
    ClientName = "SPA Client Application",
    AllowedGrantTypes = GrantTypes.Code,
    RequirePkce = true,
    RequireClientSecret = false,
    RedirectUris = { "https://localhost:5002/callback.html" },
    PostLogoutRedirectUris = { "https://localhost:5002/index.html" },
    AllowedCorsOrigins = { "https://localhost:5002" },
    AllowedScopes = { "openid", "profile", "email", "api1" },
    AllowOfflineAccess = true,
    AccessTokenLifetime = 3600,
    AlwaysIncludeUserClaimsInIdToken = true
}
```

##### 3. Machine-to-Machine Client

**Mô tả:** Client không có user (backend services, microservices)

**Ví dụ:**
```csharp
new Client
{
    ClientId = "backend-client",
    ClientName = "Backend Client",
    AllowedGrantTypes = GrantTypes.ClientCredentials,
    ClientSecrets = { new Secret("backend-secret".Sha256()) },
    AllowedScopes = { "api1", "api2" },
    AccessTokenLifetime = 3600,
    Claims = new List<ClientClaim>
    {
        new ClientClaim("client_type", "backend"),
        new ClientClaim("department", "engineering")
    }
}
```

#### Cấu hình Client trong Database

##### 1. Seed Clients

```csharp
public static class SeedData
{
    public static async Task EnsureSeedData(IServiceProvider serviceProvider)
    {
        using (var scope = serviceProvider.CreateScope())
        {
            var context = scope.ServiceProvider.GetRequiredService<ConfigurationDbContext>();
            
            // Seed clients
            if (!context.Clients.Any())
            {
                foreach (var client in Clients.Get())
                {
                    context.Clients.Add(client.ToEntity());
                }
                
                await context.SaveChangesAsync();
            }
            
            // Seed identity resources
            if (!context.IdentityResources.Any())
            {
                foreach (var resource in IdentityResources.Get())
                {
                    context.IdentityResources.Add(resource.ToEntity());
                }
                
                await context.SaveChangesAsync();
            }
            
            // Seed API resources
            if (!context.ApiResources.Any())
            {
                foreach (var resource in ApiResources.Get())
                {
                    context.ApiResources.Add(resource.ToEntity());
                }
                
                await context.SaveChangesAsync();
            }
        }
    }
}
```

##### 2. Gọi SeedData trong Program.cs

```csharp
public static async Task Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();
    
    // Seed data
    await SeedData.EnsureSeedData(host.Services);
    
    await host.RunAsync();
}
```

### 7.2.4 API Resources và Identity Resources

#### Identity Resources

**Mô tả:** Identity Resources đại diện cho thông tin về user (claims) được trả về trong ID Token.

**Ví dụ:**
```csharp
public static class IdentityResources
{
    public static IEnumerable<IdentityResource> Get()
    {
        return new List<IdentityResource>
        {
            new IdentityResources.OpenId(),
            new IdentityResources.Profile(),
            new IdentityResources.Email(),
            
            new IdentityResource
            {
                Name = "roles",
                UserClaims = { JwtClaimTypes.Role, "role" }
            },
            
            new IdentityResource
            {
                Name = "custom",
                UserClaims = 
                {
                    "given_name",
                    "family_name",
                    "website",
                    "address"
                }
            }
        };
    }
}
```

#### API Resources

**Mô tả:** API Resources đại diện cho các API được bảo vệ mà client có thể truy cập.

**Ví dụ:**
```csharp
public static class ApiResources
{
    public static IEnumerable<ApiResource> Get()
    {
        return new List<ApiResource>
        {
            new ApiResource("api1", "My API #1")
            {
                Scopes = { "api1.read", "api1.write" },
                UserClaims = { JwtClaimTypes.Name, JwtClaimTypes.Email }
            },
            
            new ApiResource("api2", "My API #2")
            {
                Scopes = { "api2.read", "api2.write", "api2.admin" },
                UserClaims = { JwtClaimTypes.Role }
            }
        };
    }
}
```

#### API Scopes

**Mô tả:** API Scopes đại diện cho các quyền truy cập cụ thể trong API.

**Ví dụ:**
```csharp
public static class ApiScopes
{
    public static IEnumerable<ApiScope> Get()
    {
        return new List<ApiScope>
        {
            new ApiScope("api1.read", "Read access to API 1"),
            new ApiScope("api1.write", "Write access to API 1"),
            
            new ApiScope("api2.read", "Read access to API 2"),
            new ApiScope("api2.write", "Write access to API 2"),
            new ApiScope("api2.admin", "Admin access to API 2")
        };
    }
}
```

## 7.3 Ví dụ code triển khai

### 7.3.1 Startup Configuration

#### Program.cs

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        var host = CreateHostBuilder(args).Build();
        
        // Seed data
        await SeedData.EnsureSeedData(host.Services);
        
        await host.RunAsync();
    }
    
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

#### Startup.cs

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }
    
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }
    
    public void ConfigureServices(IServiceCollection services)
    {
        // Cấu hình DbContext
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
        
        // Cấu hình ASP.NET Core Identity
        services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
        {
            options.Password.RequireDigit = true;
            options.Password.RequireLowercase = true;
            options.Password.RequireUppercase = true;
            options.Password.RequireNonAlphanumeric = true;
            options.Password.RequiredLength = 8;
            
            options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
            options.Lockout.MaxFailedAccessAttempts = 5;
            options.Lockout.AllowedForNewUsers = true;
            
            options.User.RequireUniqueEmail = true;
        })
        .AddEntityFrameworkStores<ApplicationDbContext>()
        .AddDefaultTokenProviders();
        
        // Cấu hình Duende IdentityServer
        var builder = services.AddIdentityServer(options =>
        {
            options.Events.RaiseErrorEvents = true;
            options.Events.RaiseInformationEvents = true;
            options.Events.RaiseFailureEvents = true;
            options.Events.RaiseSuccessEvents = true;
            
            options.IssuerUri = Configuration["IdentityServer:IssuerUri"];
        })
        .AddAspNetIdentity<ApplicationUser>()
        .AddConfigurationStore(options =>
        {
            options.ConfigureDbContext = builder =>
                builder.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                    sql => sql.MigrationsAssembly(typeof(Startup).Assembly.GetName().Name));
        })
        .AddOperationalStore(options =>
        {
            options.ConfigureDbContext = builder =>
                builder.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"),
                    sql => sql.MigrationsAssembly(typeof(Startup).Assembly.GetName().Name));
            
            options.EnableTokenCleanup = true;
            options.TokenCleanupInterval = 3600;
        });
        
        // Cấu hình signing credentials
        var certificatePath = Configuration["IdentityServer:CertificatePath"];
        var certificatePassword = Configuration["IdentityServer:CertificatePassword"];
        builder.AddSigningCredential(new X509Certificate2(certificatePath, certificatePassword));
        
        // Cấu hình authentication
        services.AddAuthentication(options =>
        {
            options.DefaultAuthenticateScheme = IdentityConstants.ApplicationScheme;
            options.DefaultChallengeScheme = IdentityConstants.ApplicationScheme;
        })
        .AddCookie(IdentityConstants.ApplicationScheme, options =>
        {
            options.Cookie.HttpOnly = true;
            options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
            options.Cookie.SameSite = SameSiteMode.Strict;
            options.ExpireTimeSpan = TimeSpan.FromDays(7);
            options.SlidingExpiration = true;
        });
        
        // Cấu hình MVC
        services.AddControllersWithViews();
        services.AddRazorPages();
        
        // Cấu hình CORS
        services.AddCors(options =>
        {
            options.AddPolicy("default", policy =>
            {
                policy.WithOrigins(Configuration["Cors:AllowedOrigins"].Split(","))
                    .AllowAnyHeader()
                    .AllowAnyMethod();
            });
        });
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
            app.UseDatabaseErrorPage();
        }
        else
        {
            app.UseExceptionHandler("/Error");
            app.UseHsts();
        }
        
        app.UseHttpsRedirection();
        app.UseStaticFiles();
        app.UseRouting();
        
        app.UseCors("default");
        
        app.UseIdentityServer();
        app.UseAuthorization();
        
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapRazorPages();
            endpoints.MapControllers();
        });
    }
}
```

#### appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=IdentityServer;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "IdentityServer": {
    "IssuerUri": "https://localhost:5001",
    "CertificatePath": "certificate.pfx",
    "CertificatePassword": "your-certificate-password"
  },
  "Cors": {
    "AllowedOrigins": "https://localhost:5001,https://localhost:5002"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

### 7.3.2 User Management

#### Register User

```csharp
// Controllers/AccountController.cs
[ApiController]
[Route("api/[controller]")]
public class AccountController : ControllerBase
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly ILogger<AccountController> _logger;
    
    public AccountController(
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        ILogger<AccountController> logger)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _logger = logger;
    }
    
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterViewModel model)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState);
        }
        
        var user = new ApplicationUser
        {
            UserName = model.Email,
            Email = model.Email,
            FirstName = model.FirstName,
            LastName = model.LastName,
            CreatedAt = DateTime.UtcNow
        };
        
        var result = await _userManager.CreateAsync(user, model.Password);
        
        if (!result.Succeeded)
        {
            foreach (var error in result.Errors)
            {
                ModelState.AddModelError(string.Empty, error.Description);
            }
            return BadRequest(ModelState);
        }
        
        _logger.LogInformation("User created a new account with password.");
        
        return Ok(new { Message = "User registered successfully" });
    }
}

// Models/RegisterViewModel.cs
public class RegisterViewModel
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Required]
    [StringLength(100, MinimumLength = 1)]
    public string FirstName { get; set; }
    
    [Required]
    [StringLength(100, MinimumLength = 1)]
    public string LastName { get; set; }
    
    [Required]
    [StringLength(100, MinimumLength = 6)]
    [DataType(DataType.Password)]
    public string Password { get; set; }
    
    [DataType(DataType.Password)]
    [Compare("Password", ErrorMessage = "The password and confirmation password do not match.")]
    public string ConfirmPassword { get; set; }
}
```

#### Login User

```csharp
[HttpPost("login")]
public async Task<IActionResult> Login([FromBody] LoginViewModel model)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    var user = await _userManager.FindByEmailAsync(model.Email);
    
    if (user == null)
    {
        return Unauthorized(new { Message = "Invalid email or password" });
    }
    
    var result = await _signInManager.PasswordSignInAsync(
        user,
        model.Password,
        model.RememberMe,
        lockoutOnFailure: true);
    
    if (result.Succeeded)
    {
        _logger.LogInformation("User logged in.");
        
        // Update last login time
        user.LastLoginAt = DateTime.UtcNow;
        await _userManager.UpdateAsync(user);
        
        return Ok(new { Message = "Login successful" });
    }
    
    if (result.RequiresTwoFactor)
    {
        return Ok(new { RequiresTwoFactor = true });
    }
    
    if (result.IsLockedOut)
    {
        _logger.LogWarning("User account locked out.");
        return Unauthorized(new { Message = "Account is locked out" });
    }
    
    return Unauthorized(new { Message = "Invalid email or password" });
}

// Models/LoginViewModel.cs
public class LoginViewModel
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Required]
    [DataType(DataType.Password)]
    public string Password { get; set; }
    
    [Display(Name = "Remember me?")]
    public bool RememberMe { get; set; }
}
```

#### Manage User Roles

```csharp
[HttpPost("{userId}/roles")]
[Authorize(Roles = "Admin")]
public async Task<IActionResult> AddUserToRole(string userId, [FromBody] AddRoleViewModel model)
{
    var user = await _userManager.FindByIdAsync(userId);
    
    if (user == null)
    {
        return NotFound(new { Message = "User not found" });
    }
    
    var result = await _userManager.AddToRoleAsync(user, model.RoleName);
    
    if (!result.Succeeded)
    {
        return BadRequest(result.Errors);
    }
    
    return Ok(new { Message = $"User added to role {model.RoleName}" });
}

[HttpDelete("{userId}/roles/{roleName}")]
[Authorize(Roles = "Admin")]
public async Task<IActionResult> RemoveUserFromRole(string userId, string roleName)
{
    var user = await _userManager.FindByIdAsync(userId);
    
    if (user == null)
    {
        return NotFound(new { Message = "User not found" });
    }
    
    var result = await _userManager.RemoveFromRoleAsync(user, roleName);
    
    if (!result.Succeeded)
    {
        return BadRequest(result.Errors);
    }
    
    return Ok(new { Message = $"User removed from role {roleName}" });
}

// Models/AddRoleViewModel.cs
public class AddRoleViewModel
{
    [Required]
    public string RoleName { get; set; }
}
```

### 7.3.3 Token Generation

#### Custom Token Service

```csharp
// Services/CustomTokenService.cs
public class CustomTokenService : ICustomTokenService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly IConfiguration _configuration;
    private readonly ILogger<CustomTokenService> _logger;
    
    public CustomTokenService(
        UserManager<ApplicationUser> userManager,
        IConfiguration configuration,
        ILogger<CustomTokenService> logger)
    {
        _userManager = userManager;
        _configuration = configuration;
        _logger = logger;
    }
    
    public async Task<string> GenerateAccessTokenAsync(ApplicationUser user, List<string> scopes)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(), ClaimValueTypes.Integer64),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim("given_name", user.FirstName),
            new Claim("family_name", user.LastName)
        };
        
        // Thêm user roles
        var roles = await _userManager.GetRolesAsync(user);
        claims.AddRange(roles.Select(role => new Claim(JwtClaimTypes.Role, role)));
        
        // Thêm scopes
        claims.AddRange(scopes.Select(scope => new Claim("scope", scope)));
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public async Task<string> GenerateRefreshTokenAsync(ApplicationUser user)
    {
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(), ClaimValueTypes.Integer64),
            new Claim("type", "refresh")
        };
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddDays(30),
            signingCredentials: credentials);
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

#### Token Endpoint

```csharp
[HttpPost("token")]
public async Task<IActionResult> Token([FromBody] TokenRequestViewModel model)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    var user = await _userManager.FindByEmailAsync(model.Email);
    
    if (user == null)
    {
        return Unauthorized(new { Message = "Invalid email or password" });
    }
    
    var result = await _signInManager.CheckPasswordSignInAsync(user, model.Password, false);
    
    if (!result.Succeeded)
    {
        return Unauthorized(new { Message = "Invalid email or password" });
    }
    
    // Generate tokens
    var accessToken = await _customTokenService.GenerateAccessTokenAsync(user, model.Scopes);
    var refreshToken = await _customTokenService.GenerateRefreshTokenAsync(user);
    
    // Save refresh token
    await _tokenService.SaveRefreshTokenAsync(user.Id, refreshToken);
    
    return Ok(new TokenResponse
    {
        AccessToken = accessToken,
        RefreshToken = refreshToken,
        ExpiresIn = 3600, // 1 giờ
        TokenType = "Bearer"
    });
}

[HttpPost("refresh")]
public async Task<IActionResult> Refresh([FromBody] RefreshTokenViewModel model)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    // Validate refresh token
    var tokenData = await _tokenService.GetRefreshTokenAsync(model.RefreshToken);
    
    if (tokenData == null)
    {
        return Unauthorized(new { Message = "Invalid refresh token" });
    }
    
    var user = await _userManager.FindByIdAsync(tokenData.UserId);
    
    if (user == null)
    {
        return Unauthorized(new { Message = "User not found" });
    }
    
    // Generate new tokens
    var accessToken = await _customTokenService.GenerateAccessTokenAsync(user, tokenData.Scopes);
    var newRefreshToken = await _customTokenService.GenerateRefreshTokenAsync(user);
    
    // Revoke old refresh token
    await _tokenService.RevokeRefreshTokenAsync(model.RefreshToken);
    
    // Save new refresh token
    await _tokenService.SaveRefreshTokenAsync(user.Id, newRefreshToken, tokenData.Scopes);
    
    return Ok(new TokenResponse
    {
        AccessToken = accessToken,
        RefreshToken = newRefreshToken,
        ExpiresIn = 3600,
        TokenType = "Bearer"
    });
}

// Models/TokenRequestViewModel.cs
public class TokenRequestViewModel
{
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Required]
    public string Password { get; set; }
    
    public List<string> Scopes { get; set; }
}

// Models/RefreshTokenViewModel.cs
public class RefreshTokenViewModel
{
    [Required]
    public string RefreshToken { get; set; }
}

// Models/TokenResponse.cs
public class TokenResponse
{
    public string AccessToken { get; set; }
    public string RefreshToken { get; set; }
    public int ExpiresIn { get; set; }
    public string TokenType { get; set; }
}
```

## 7.4 Testing và Debugging

### 7.4.1 Unit Testing

#### Test User Registration

```csharp
// Tests/AccountControllerTests.cs
public class AccountControllerTests
{
    private readonly Mock<UserManager<ApplicationUser>> _mockUserManager;
    private readonly Mock<SignInManager<ApplicationUser>> _mockSignInManager;
    private readonly Mock<ILogger<AccountController>> _mockLogger;
    private readonly AccountController _controller;
    
    public AccountControllerTests()
    {
        var store = new Mock<IUserStore<ApplicationUser>>();
        _mockUserManager = new Mock<UserManager<ApplicationUser>>(
            store.Object, null, null, null, null, null, null, null, null);
        
        var contextAccessor = new Mock<IHttpContextAccessor>();
        var userPrincipalFactory = new Mock<IUserClaimsPrincipalFactory<ApplicationUser>>();
        _mockSignInManager = new Mock<SignInManager<ApplicationUser>>(
            _mockUserManager.Object, contextAccessor.Object, userPrincipalFactory.Object, null, null, null, null);
        
        _mockLogger = new Mock<ILogger<AccountController>>();
        
        _controller = new AccountController(
            _mockUserManager.Object,
            _mockSignInManager.Object,
            _mockLogger.Object);
    }
    
    [Fact]
    public async Task Register_WithValidData_ReturnsOk()
    {
        // Arrange
        var model = new RegisterViewModel
        {
            Email = "test@example.com",
            FirstName = "John",
            LastName = "Doe",
            Password = "Password123!",
            ConfirmPassword = "Password123!"
        };
        
        _mockUserManager
            .Setup(x => x.CreateAsync(It.IsAny<ApplicationUser>(), It.IsAny<string>()))
            .ReturnsAsync(IdentityResult.Success);
        
        // Act
        var result = await _controller.Register(model);
        
        // Assert
        var okResult = Assert.IsType<OkObjectResult>(result);
        Assert.NotNull(okResult.Value);
    }
    
    [Fact]
    public async Task Register_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var model = new RegisterViewModel
        {
            Email = "invalid-email",
            FirstName = "John",
            LastName = "Doe",
            Password = "Password123!",
            ConfirmPassword = "Password123!"
        };
        
        _controller.ModelState.AddModelError("Email", "Invalid email format");
        
        // Act
        var result = await _controller.Register(model);
        
        // Assert
        Assert.IsType<BadRequestObjectResult>(result);
    }
}
```

#### Test Token Generation

```csharp
// Tests/CustomTokenServiceTests.cs
public class CustomTokenServiceTests
{
    private readonly Mock<UserManager<ApplicationUser>> _mockUserManager;
    private readonly Mock<IConfiguration> _mockConfiguration;
    private readonly Mock<ILogger<CustomTokenService>> _mockLogger;
    private readonly CustomTokenService _service;
    
    public CustomTokenServiceTests()
    {
        var store = new Mock<IUserStore<ApplicationUser>>();
        _mockUserManager = new Mock<UserManager<ApplicationUser>>(
            store.Object, null, null, null, null, null, null, null, null);
        
        _mockConfiguration = new Mock<IConfiguration>();
        _mockConfiguration.Setup(x => x["Jwt:Key"]).Returns("your-secret-key-should-be-long-enough");
        _mockConfiguration.Setup(x => x["Jwt:Issuer"]).Returns("https://localhost:5001");
        _mockConfiguration.Setup(x => x["Jwt:Audience"]).Returns("api1");
        
        _mockLogger = new Mock<ILogger<CustomTokenService>>();
        
        _service = new CustomTokenService(
            _mockUserManager.Object,
            _mockConfiguration.Object,
            _mockLogger.Object);
    }
    
    [Fact]
    public async Task GenerateAccessTokenAsync_ReturnsValidToken()
    {
        // Arrange
        var user = new ApplicationUser
        {
            Id = "1",
            Email = "test@example.com",
            FirstName = "John",
            LastName = "Doe"
        };
        
        _mockUserManager
            .Setup(x => x.GetRolesAsync(user))
            .ReturnsAsync(new List<string> { "User" });
        
        // Act
        var token = await _service.GenerateAccessTokenAsync(user, new List<string> { "api1.read" });
        
        // Assert
        Assert.NotNull(token);
        
        var handler = new JwtSecurityTokenHandler();
        var jwt = handler.ReadJwtToken(token);
        
        Assert.Equal("1", jwt.Subject);
        Assert.Equal("test@example.com", jwt.Claims.First(c => c.Type == JwtRegisteredClaimNames.Email).Value);
        Assert.Equal("John", jwt.Claims.First(c => c.Type == "given_name").Value);
        Assert.Equal("Doe", jwt.Claims.First(c => c.Type == "family_name").Value);
        Assert.Equal("User", jwt.Claims.First(c => c.Type == JwtClaimTypes.Role).Value);
    }
}
```

### 7.4.2 Integration Testing

#### Test Authorization Flow

```csharp
// Tests/AuthorizationFlowTests.cs
public class AuthorizationFlowTests : IClassFixture<WebApplicationFactory<Startup>>
{
    private readonly WebApplicationFactory<Startup> _factory;
    
    public AuthorizationFlowTests(WebApplicationFactory<Startup> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task AuthorizationCodeFlow_ReturnsAuthorizationCode()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/connect/authorize?" +
            "client_id=web-client&" +
            "redirect_uri=https://localhost:5001/signin-oidc&" +
            "response_type=code&" +
            "scope=openid profile email api1&" +
            "state=test-state&" +
            "code_challenge=test-challenge&" +
            "code_challenge_method=S256");
        
        // Assert
        Assert.Equal(HttpStatusCode.Redirect, response.StatusCode);
        Assert.Contains("signin-oidc", response.Headers.Location.ToString());
    }
    
    [Fact]
    public async Task TokenEndpoint_ReturnsAccessToken()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        var tokenRequest = new Dictionary<string, string>
        {
            { "client_id", "web-client" },
            { "client_secret", "secret" },
            { "grant_type", "authorization_code" },
            { "code", "test-code" },
            { "redirect_uri", "https://localhost:5001/signin-oidc" }
        };
        
        // Act
        var content = new FormUrlEncodedContent(tokenRequest);
        var response = await client.PostAsync("/connect/token", content);
        
        // Assert
        response.EnsureSuccessStatusCode();
        var responseContent = await response.Content.ReadAsStringAsync();
        var tokenResponse = JsonSerializer.Deserialize<JsonElement>(responseContent);
        
        Assert.True(tokenResponse.TryGetProperty("access_token", out var accessToken));
        Assert.True(tokenResponse.TryGetProperty("token_type", out var tokenType));
        Assert.Equal("Bearer", tokenType.GetString());
    }
}
```

### 7.4.3 Debugging Tips

#### Enable IdentityServer Logging

```csharp
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information",
      "Microsoft.Hosting.Lifetime": "Information",
      "Duende.IdentityServer": "Debug"
    }
  }
}
```

#### Use IdentityServer Admin UI

```bash
dotnet add package Duende.IdentityServer.Admin
```

```csharp
// Startup.cs
services.AddIdentityServerAdminUI();
```

#### Use Developer Tools

- Chrome DevTools: Kiểm tra network requests, cookies, local storage
- Fiddler/Charles: Debug OAuth 2.0 flow
- Postman: Test các endpoints
- JWT.io: Decode và verify JWT tokens

## 7.5 Deployment Considerations

### 7.5.1 Environment Configuration

#### Development Environment

```json
// appsettings.Development.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=IdentityServerDev;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "IdentityServer": {
    "IssuerUri": "https://localhost:5001",
    "CertificatePath": "certificate-dev.pfx",
    "CertificatePassword": "dev-certificate-password"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information",
      "Duende.IdentityServer": "Debug"
    }
  }
}
```

#### Production Environment

```json
// appsettings.Production.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=production-server;Database=IdentityServerProd;User Id=identity-user;Password=production-password;MultipleActiveResultSets=true"
  },
  "IdentityServer": {
    "IssuerUri": "https://auth.example.com",
    "CertificatePath": "/certs/certificate.pfx",
    "CertificatePassword": "${CERTIFICATE_PASSWORD}"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Warning",
      "Duende.IdentityServer": "Warning"
    }
  }
}
```

### 7.5.2 SSL/TLS Configuration

#### Use Valid SSL Certificate

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseKestrel(options =>
            {
                options.Listen(IPAddress.Any, 5000);
                
                options.Listen(IPAddress.Any, 5001, listenOptions =>
                {
                    var certificatePath = Environment.GetEnvironmentVariable("CERTIFICATE_PATH");
                    var certificatePassword = Environment.GetEnvironmentVariable("CERTIFICATE_PASSWORD");
                    
                    listenOptions.UseHttps(certificatePath, certificatePassword);
                    listenOptions.Protocols = HttpProtocols.Tls12 | HttpProtocols.Tls13;
                });
            })
            .UseStartup<Startup>();
        });
```

### 7.5.3 Database Deployment

#### Use Database Migrations

```bash
# Tạo migration
dotnet ef migrations add AddNewFeature -c ConfigurationDbContext

# Apply migration
dotnet ef database update -c ConfigurationDbContext
```

#### Use Database Seeding

```csharp
// Program.cs
public static async Task Main(string[] args)
{
    var host = CreateHostBuilder(args).Build();
    
    var environment = host.Services.GetRequiredService<IWebHostEnvironment>();
    
    if (environment.IsDevelopment() || environment.IsStaging())
    {
        // Seed data trong development và staging
        await SeedData.EnsureSeedData(host.Services);
    }
    
    await host.RunAsync();
}
```

### 7.5.4 Monitoring and Logging

#### Use Application Insights

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

```csharp
// Startup.cs
services.AddApplicationInsightsTelemetry(Configuration);
```

#### Use Serilog

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
```

```csharp
// Program.cs
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseSerilog((context, configuration) =>
        {
            configuration
                .ReadFrom.Configuration(context.Configuration)
                .Enrich.FromLogContext()
                .WriteTo.Console()
                .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day);
        })
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

### 7.5.5 Health Checks

```bash
dotnet add package Microsoft.AspNetCore.Diagnostics.HealthChecks
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore
```

```csharp
// Startup.cs
services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>()
    .AddIdentityServer("identity-server");

// Startup.cs
public void Configure(IApplicationBuilder app)
{
    app.UseHealthChecks("/health");
    
    app.UseHealthChecks("/health/ready", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("ready")
    });
    
    app.UseHealthChecks("/health/live", new HealthCheckOptions
    {
        Predicate = _ => false
    });
}
```

## Tóm tắt

Phần này đã mô tả chi tiết cách triển khai Authentication Server với .NET:

1. **Tổng quan về ASP.NET Core Identity:** Tích hợp ASP.NET Core Identity với OAuth 2.0
2. **Duende IdentityServer:** Cấu hình cơ bản, Client configuration, API Resources và Identity Resources
3. **Ví dụ code triển khai:** Startup configuration, User management, Token generation
4. **Testing và Debugging:** Unit testing, Integration testing, Debugging tips
5. **Deployment considerations:** Environment configuration, SSL/TLS configuration, Database deployment, Monitoring and logging, Health checks

Phần tiếp theo sẽ đi sâu vào Best Practices và Checklist.
