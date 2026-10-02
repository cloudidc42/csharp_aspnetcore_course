# Part 088: Security Best Practices

## เนื้อหาใน Part นี้
- OWASP Top 10 ใน .NET
- SQL Injection prevention
- XSS prevention
- CSRF protection
- Secure headers
- Secret management
- Dependency scanning
- โปรแกรมตัวอย่าง: Security audit checklist

---

## 1. OWASP Top 10

OWASP (Open Web Application Security Project) เผยแพร่รายการช่องโหว่ที่พบบ่อยที่สุด

```
OWASP Top 10 (2021):
A01: Broken Access Control
A02: Cryptographic Failures
A03: Injection (SQL, NoSQL, OS, LDAP)
A04: Insecure Design
A05: Security Misconfiguration
A06: Vulnerable and Outdated Components
A07: Identification and Authentication Failures
A08: Software and Data Integrity Failures
A09: Security Logging and Monitoring Failures
A10: Server-Side Request Forgery (SSRF)
```

---

## 2. SQL Injection Prevention

### ตัวอย่าง SQL Injection

```csharp
// ไม่ดี - SQL Injection vulnerable!
public async Task<User?> LoginUnsafe(string username, string password)
{
    // username = "admin' OR '1'='1" จะดึง user ทุกคน!
    var sql = $"SELECT * FROM Users WHERE Username = '{username}' AND Password = '{password}'";
    return await dbContext.Users.FromSqlRaw(sql).FirstOrDefaultAsync();
}

// ดี - Parameterized queries
public async Task<User?> LoginSafe(string username, string password)
{
    // ใช้ EF Core - auto parameterized
    return await dbContext.Users
        .FirstOrDefaultAsync(u => u.Username == username && u.PasswordHash == HashPassword(password));
}

// ดี - Parameterized raw SQL
public async Task<User?> LoginSafeRaw(string username, string password)
{
    return await dbContext.Users
        .FromSqlInterpolated($"SELECT * FROM Users WHERE Username = {username} AND PasswordHash = {HashPassword(password)}")
        .FirstOrDefaultAsync();
}

// ดี - Stored procedure
public async Task<User?> LoginWithStoredProc(string username, string password)
{
    return await dbContext.Users
        .FromSqlRaw("EXEC sp_ValidateUser @Username, @PasswordHash",
            new SqlParameter("@Username", username),
            new SqlParameter("@PasswordHash", HashPassword(password)))
        .FirstOrDefaultAsync();
}
```

### Input Validation

```csharp
// Validation ที่ model level
public record RegisterUserRequest
{
    [Required]
    [StringLength(50, MinimumLength = 3)]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Username can only contain letters, numbers, and underscores")]
    public string Username { get; init; } = string.Empty;
    
    [Required]
    [EmailAddress]
    [StringLength(100)]
    public string Email { get; init; } = string.Empty;
    
    [Required]
    [StringLength(100, MinimumLength = 8)]
    [PasswordComplexity]  // Custom attribute
    public string Password { get; init; } = string.Empty;
}

// Custom validation attribute
[AttributeUsage(AttributeTargets.Property)]
public class PasswordComplexityAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext context)
    {
        if (value is not string password)
            return new ValidationResult("Password is required");
        
        var errors = new List<string>();
        
        if (!password.Any(char.IsUpper))
            errors.Add("at least one uppercase letter");
        if (!password.Any(char.IsLower))
            errors.Add("at least one lowercase letter");
        if (!password.Any(char.IsDigit))
            errors.Add("at least one digit");
        if (!password.Any(c => "!@#$%^&*".Contains(c)))
            errors.Add("at least one special character");
        
        if (errors.Any())
            return new ValidationResult($"Password must contain {string.Join(", ", errors)}");
        
        return ValidationResult.Success;
    }
}
```

---

## 3. XSS Prevention

### XSS ใน Razor Pages

```csharp
// ไม่ดี - XSS vulnerable
// ใน Controller:
ViewBag.UserComment = Request.Query["comment"]; // อาจมี <script>

// ใน Razor:
@Html.Raw(ViewBag.UserComment) // VULNERABLE! render as HTML

// ดี - Encoded by default
@ViewBag.UserComment // Razor auto-encodes

// ดี - HtmlEncoder explicit
@Html.Encode(userInput)
```

### Content Security Policy (CSP)

```csharp
// Program.cs - Add CSP headers
app.Use(async (context, next) =>
{
    context.Response.Headers.Add(
        "Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{nonce}'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data: https:; " +
        "font-src 'self' https://fonts.gstatic.com; " +
        "connect-src 'self' https://api.myapp.com; " +
        "frame-ancestors 'none';");
    
    await next();
});

// หรือใช้ Middleware ที่ generate nonce
public class CspMiddleware
{
    private readonly RequestDelegate _next;
    
    public CspMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var nonce = Convert.ToBase64String(RandomNumberGenerator.GetBytes(16));
        context.Items["csp-nonce"] = nonce;
        
        context.Response.OnStarting(() =>
        {
            context.Response.Headers["Content-Security-Policy"] =
                $"default-src 'self'; " +
                $"script-src 'self' 'nonce-{nonce}'; " +
                $"style-src 'self' 'nonce-{nonce}';";
            return Task.CompletedTask;
        });
        
        await _next(context);
    }
}
```

### Output Encoding

```csharp
using System.Web;
using System.Text.Encodings.Web;

public class SafeContentBuilder
{
    private readonly HtmlEncoder _htmlEncoder;
    private readonly JavaScriptEncoder _jsEncoder;
    private readonly UrlEncoder _urlEncoder;
    
    public SafeContentBuilder(
        HtmlEncoder htmlEncoder,
        JavaScriptEncoder jsEncoder,
        UrlEncoder urlEncoder)
    {
        _htmlEncoder = htmlEncoder;
        _jsEncoder = jsEncoder;
        _urlEncoder = urlEncoder;
    }
    
    // Encode สำหรับ HTML context
    public string ForHtml(string input) => _htmlEncoder.Encode(input);
    
    // Encode สำหรับ JavaScript context
    public string ForJavaScript(string input) => _jsEncoder.Encode(input);
    
    // Encode สำหรับ URL context
    public string ForUrl(string input) => _urlEncoder.Encode(input);
}
```

---

## 4. CSRF Protection

```csharp
// Program.cs - CSRF protection
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";  // สำหรับ SPA
    options.Cookie.Name = "XSRF-TOKEN";
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

// ใน Controller
[ValidateAntiForgeryToken]  // ตรวจสอบ CSRF token อัตโนมัติ
[HttpPost]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    // ...
}

// สำหรับ API (stateless) - ใช้ SameSite cookies แทน
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer();  // JWT ใน Authorization header ไม่ต้องกังวล CSRF

// สำหรับ SPA - Double Submit Cookie Pattern
app.MapGet("/api/antiforgery/token", (IAntiforgery antiforgery, HttpContext context) =>
{
    var tokens = antiforgery.GetAndStoreTokens(context);
    return Results.Ok(new { token = tokens.RequestToken });
});
```

---

## 5. Secure Headers

```csharp
// Middleware สำหรับ security headers
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;
    
    public SecurityHeadersMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        var headers = context.Response.Headers;
        
        // ป้องกัน clickjacking
        headers["X-Frame-Options"] = "DENY";
        
        // ป้องกัน MIME type sniffing
        headers["X-Content-Type-Options"] = "nosniff";
        
        // ป้องกัน XSS (browser-level)
        headers["X-XSS-Protection"] = "1; mode=block";
        
        // Force HTTPS
        headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload";
        
        // ลด information disclosure
        headers.Remove("Server");
        headers.Remove("X-Powered-By");
        headers["X-Permitted-Cross-Domain-Policies"] = "none";
        
        // Referrer Policy
        headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
        
        // Permissions Policy
        headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
        
        await _next(context);
    }
}

// หรือใช้ NWebsec library
app.UseNWebSecurityHeaders(options =>
{
    options.AddDefaultSecurePolicy()
        .AddStrictTransportSecurity(maxAgeInSeconds: 31536000, includeSubdomains: true);
});
```

---

## 6. Authentication Security

### JWT Best Practices

```csharp
// ตั้งค่า JWT อย่างปลอดภัย
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!)),
            ClockSkew = TimeSpan.Zero  // ไม่ allow clock drift
        };
        
        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = async ctx =>
            {
                // ตรวจสอบ token ว่ายัง valid (ไม่ถูก revoke)
                var tokenValidator = ctx.HttpContext.RequestServices
                    .GetRequiredService<ITokenValidator>();
                
                var jti = ctx.Principal?.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
                if (jti != null && await tokenValidator.IsRevokedAsync(jti))
                {
                    ctx.Fail("Token has been revoked");
                }
            }
        };
    });

// Token generation ที่ปลอดภัย
public class JwtTokenService
{
    private readonly IConfiguration _config;
    private readonly ITokenRepository _tokenRepo;
    
    public async Task<TokenResponse> GenerateTokenAsync(User user)
    {
        var jti = Guid.NewGuid().ToString();
        var now = DateTime.UtcNow;
        var expiry = now.AddMinutes(15);  // Short expiry!
        
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, jti),
            new(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString()),
            new(ClaimTypes.Role, user.Role),
        };
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:SecretKey"]!));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            notBefore: now,
            expires: expiry,
            signingCredentials: credentials);
        
        var accessToken = new JwtSecurityTokenHandler().WriteToken(token);
        
        // Generate refresh token
        var refreshToken = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
        await _tokenRepo.StoreRefreshTokenAsync(user.Id, refreshToken, 
            now.AddDays(7));  // Longer refresh token
        
        return new TokenResponse(accessToken, refreshToken, expiry);
    }
}
```

### Password Security

```csharp
// ใช้ ASP.NET Core Identity password hasher (BCrypt/PBKDF2)
public class PasswordService
{
    private readonly IPasswordHasher<User> _hasher;
    
    public PasswordService(IPasswordHasher<User> hasher)
    {
        _hasher = hasher;
    }
    
    public string HashPassword(User user, string password)
    {
        return _hasher.HashPassword(user, password);
    }
    
    public bool VerifyPassword(User user, string hashedPassword, string providedPassword)
    {
        var result = _hasher.VerifyHashedPassword(user, hashedPassword, providedPassword);
        return result != PasswordVerificationResult.Failed;
    }
}

// ตรวจสอบ common passwords
public class CommonPasswordValidator : IPasswordValidator<User>
{
    private static readonly HashSet<string> CommonPasswords = new(StringComparer.OrdinalIgnoreCase)
    {
        "password", "123456", "password1", "qwerty", "abc123",
        "monkey", "1234567", "letmein", "trustno1", "dragon"
    };
    
    public Task<IdentityResult> ValidateAsync(UserManager<User> manager, User user, string? password)
    {
        if (password != null && CommonPasswords.Contains(password))
        {
            return Task.FromResult(IdentityResult.Failed(new IdentityError
            {
                Code = "CommonPassword",
                Description = "Password is too common"
            }));
        }
        return Task.FromResult(IdentityResult.Success);
    }
}
```

---

## 7. Secret Management

### User Secrets (Development)

```bash
# สร้าง user secrets
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Host=localhost;..."
dotnet user-secrets set "Jwt:SecretKey" "my-dev-secret"

# ดู secrets
dotnet user-secrets list
```

### Environment Variables

```csharp
// appsettings.json - อย่าเก็บ secrets ที่นี่!
{
  "ConnectionStrings": {
    "DefaultConnection": "" // ว่างเปล่า - override ด้วย env var
  }
}

// Program.cs - อ่านจาก env var
var connectionString = 
    builder.Configuration.GetConnectionString("DefaultConnection")  // env var: ConnectionStrings__DefaultConnection
    ?? throw new InvalidOperationException("Connection string not configured");
```

### Secret Scanning

```csharp
// ตรวจสอบว่า secrets ไม่รั่วใน logs
public class SensitiveDataFilterLogger : ILogger
{
    private readonly ILogger _inner;
    private static readonly string[] SensitiveKeywords = 
    {
        "password", "secret", "key", "token", "connectionstring"
    };
    
    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state,
        Exception? exception, Func<TState, Exception?, string> formatter)
    {
        var message = formatter(state, exception);
        
        // ตรวจสอบว่า message ไม่มี sensitive data
        if (ContainsSensitiveData(message))
        {
            message = "[REDACTED - Sensitive data detected]";
        }
        
        _inner.Log(logLevel, eventId, message, exception, (s, e) => s);
    }
    
    private static bool ContainsSensitiveData(string message)
    {
        return SensitiveKeywords.Any(keyword => 
            message.Contains(keyword, StringComparison.OrdinalIgnoreCase));
    }
    
    public bool IsEnabled(LogLevel logLevel) => _inner.IsEnabled(logLevel);
    public IDisposable? BeginScope<TState>(TState state) where TState : notnull => _inner.BeginScope(state);
}
```

---

## 8. Dependency Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday

jobs:
  vulnerability-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '9.0.x'
    
    # .NET vulnerability scan
    - name: Run dotnet audit
      run: dotnet list package --vulnerable --include-transitive
    
    # OWASP Dependency Check
    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'MyApp'
        path: '.'
        format: 'HTML,JSON'
        failBuildOnCVSS: 7  # Fail ถ้า CVSS score >= 7
    
    # Trivy scan
    - name: Trivy filesystem scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        severity: 'HIGH,CRITICAL'
```

### NuGet Audit

```bash
# ตรวจสอบ vulnerable packages
dotnet list package --vulnerable

# Output:
# The following sources were used:
#    https://api.nuget.org/v3/index.json
# 
# Project `MyApp.API` has the following vulnerable packages
#    [net9.0]:
#    Top-level Package          Requested   Resolved   Severity   Advisory URL
#    > Newtonsoft.Json          12.0.3      12.0.3     High       https://...
```

```xml
<!-- เพิ่ม NuGetAudit ใน .csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <NuGetAudit>enable</NuGetAudit>
    <NuGetAuditLevel>high</NuGetAuditLevel>
    <NuGetAuditMode>all</NuGetAuditMode>
  </PropertyGroup>
</Project>
```

---

## 9. Rate Limiting

```csharp
// Program.cs
builder.Services.AddRateLimiter(options =>
{
    // Global rate limit
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        var ipAddress = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: ipAddress,
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 0
            });
    });
    
    // Per-endpoint rate limits
    options.AddFixedWindowLimiter("login", config =>
    {
        config.PermitLimit = 5;  // 5 attempts
        config.Window = TimeSpan.FromMinutes(15);  // per 15 minutes
        config.QueueLimit = 0;
    });
    
    options.AddFixedWindowLimiter("api", config =>
    {
        config.PermitLimit = 1000;
        config.Window = TimeSpan.FromHours(1);
    });
    
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = 
                retryAfter.TotalSeconds.ToString();
        }
        
        await context.HttpContext.Response.WriteAsync("Too many requests. Please try again later.");
    };
});

app.UseRateLimiter();

// ใช้กับ endpoint
app.MapPost("/api/auth/login", async (LoginRequest request, IAuthService auth) =>
{
    var result = await auth.LoginAsync(request);
    return result.Success ? Results.Ok(result) : Results.Unauthorized();
})
.RequireRateLimiting("login");
```

---

## 10. โปรแกรมตัวอย่าง: Security Audit Checklist

### SecurityAuditService

```csharp
// Services/SecurityAuditService.cs
public class SecurityAuditResult
{
    public bool Passed { get; set; }
    public List<SecurityIssue> Issues { get; set; } = new();
    public List<SecurityWarning> Warnings { get; set; } = new();
    
    public string Summary => Passed 
        ? $"Security audit passed with {Warnings.Count} warnings"
        : $"Security audit FAILED: {Issues.Count} critical issues, {Warnings.Count} warnings";
}

public record SecurityIssue(string Category, string Description, string Severity);
public record SecurityWarning(string Category, string Description, string Recommendation);

public class SecurityAuditService
{
    private readonly IConfiguration _config;
    private readonly IWebHostEnvironment _env;
    
    public SecurityAuditService(IConfiguration config, IWebHostEnvironment env)
    {
        _config = config;
        _env = env;
    }
    
    public SecurityAuditResult RunAudit()
    {
        var result = new SecurityAuditResult();
        
        CheckHttpsConfiguration(result);
        CheckJwtConfiguration(result);
        CheckDatabaseConnection(result);
        CheckCorsConfiguration(result);
        CheckSecurityHeaders(result);
        CheckEnvironmentConfiguration(result);
        
        result.Passed = !result.Issues.Any();
        return result;
    }
    
    private void CheckHttpsConfiguration(SecurityAuditResult result)
    {
        var httpsEnabled = _config.GetValue<bool>("ASPNETCORE_HTTPS_PORTS") != 0;
        var httpsRedirect = _config.GetValue<bool>("UseHttpsRedirection", true);
        
        if (!httpsRedirect && _env.IsProduction())
        {
            result.Issues.Add(new SecurityIssue(
                "HTTPS",
                "HTTPS redirect is not enabled in production",
                "Critical"));
        }
    }
    
    private void CheckJwtConfiguration(SecurityAuditResult result)
    {
        var jwtKey = _config["Jwt:SecretKey"];
        
        if (string.IsNullOrEmpty(jwtKey))
        {
            result.Issues.Add(new SecurityIssue(
                "JWT",
                "JWT secret key is not configured",
                "Critical"));
        }
        else if (jwtKey.Length < 32)
        {
            result.Issues.Add(new SecurityIssue(
                "JWT",
                "JWT secret key is too short (minimum 32 characters)",
                "High"));
        }
        else if (jwtKey == "your-secret-key" || jwtKey == "development-key")
        {
            result.Issues.Add(new SecurityIssue(
                "JWT",
                "JWT secret key appears to be a default/insecure value",
                "Critical"));
        }
        
        var expiry = _config.GetValue<int>("Jwt:ExpiryMinutes", 0);
        if (expiry > 60 && _env.IsProduction())
        {
            result.Warnings.Add(new SecurityWarning(
                "JWT",
                $"JWT expiry is {expiry} minutes - consider shorter expiry",
                "Use shorter access token lifetime (15-30 min) with refresh tokens"));
        }
    }
    
    private void CheckDatabaseConnection(SecurityAuditResult result)
    {
        var connectionString = _config.GetConnectionString("DefaultConnection");
        
        if (string.IsNullOrEmpty(connectionString))
        {
            result.Issues.Add(new SecurityIssue(
                "Database",
                "Database connection string is not configured",
                "Critical"));
            return;
        }
        
        // ตรวจ plain text password ใน connection string
        if (connectionString.Contains("Password=") || connectionString.Contains("pwd="))
        {
            if (_env.IsProduction())
            {
                result.Warnings.Add(new SecurityWarning(
                    "Database",
                    "Plain text password in connection string",
                    "Use Managed Identity or Azure Key Vault for secrets"));
            }
        }
    }
    
    private void CheckCorsConfiguration(SecurityAuditResult result)
    {
        var allowedOrigins = _config.GetSection("Cors:AllowedOrigins").Get<string[]>();
        
        if (allowedOrigins != null && allowedOrigins.Contains("*") && _env.IsProduction())
        {
            result.Issues.Add(new SecurityIssue(
                "CORS",
                "CORS allows all origins (*) in production",
                "High"));
        }
    }
    
    private void CheckSecurityHeaders(SecurityAuditResult result)
    {
        // Check if security headers middleware is configured
        // This would typically be checked via middleware registration
        result.Warnings.Add(new SecurityWarning(
            "Headers",
            "Verify security headers are configured",
            "Add X-Frame-Options, X-Content-Type-Options, HSTS, CSP headers"));
    }
    
    private void CheckEnvironmentConfiguration(SecurityAuditResult result)
    {
        if (!_env.IsProduction())
        {
            result.Warnings.Add(new SecurityWarning(
                "Environment",
                $"Running in {_env.EnvironmentName} mode",
                "Ensure production environment is properly configured before deployment"));
        }
        
        if (_env.IsProduction() && _config.GetValue<bool>("ShowDetailedErrors"))
        {
            result.Issues.Add(new SecurityIssue(
                "Error Handling",
                "Detailed errors are shown in production",
                "High"));
        }
    }
}
```

### Security Middleware Stack

```csharp
// Program.cs - Complete security setup
var builder = WebApplication.CreateBuilder(args);

// ─── Security Services ─────────────────────────────────────────

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!)),
            ClockSkew = TimeSpan.Zero
        };
    });

// Authorization with policies
builder.Services.AddAuthorization(options =>
{
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
    
    options.AddPolicy("SameUserOrAdmin", policy =>
        policy.RequireAssertion(context =>
        {
            var userId = context.User.FindFirst("sub")?.Value;
            var resourceUserId = (context.Resource as string);
            return context.User.IsInRole("Admin") || userId == resourceUserId;
        }));
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedOrigins", policy =>
    {
        var allowedOrigins = builder.Configuration
            .GetSection("Cors:AllowedOrigins")
            .Get<string[]>() ?? Array.Empty<string>();
        
        policy.WithOrigins(allowedOrigins)
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials()
              .WithExposedHeaders("X-Pagination");
    });
});

// Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddSlidingWindowLimiter("default", config =>
    {
        config.PermitLimit = 100;
        config.Window = TimeSpan.FromMinutes(1);
        config.SegmentsPerWindow = 6;
    });
});

// Data Protection
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(builder.Configuration["Azure:BlobStorage:ConnectionString"])
    .ProtectKeysWithAzureKeyVault(builder.Configuration["Azure:KeyVault:KeyId"],
        new DefaultAzureCredential());

var app = builder.Build();

// ─── Security Middleware (order matters!) ─────────────────────

if (app.Environment.IsProduction())
{
    app.UseHsts();
}

app.UseHttpsRedirection();

// Security headers
app.UseMiddleware<SecurityHeadersMiddleware>();

app.UseCors("AllowedOrigins");

app.UseRateLimiter();

app.UseAuthentication();
app.UseAuthorization();

// Audit logging
app.Use(async (context, next) =>
{
    var start = Stopwatch.GetTimestamp();
    await next();
    var elapsed = Stopwatch.GetElapsedTime(start);
    
    if (context.Response.StatusCode is 401 or 403)
    {
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogWarning(
            "Access denied: {Method} {Path} {StatusCode} {UserId} {IP}",
            context.Request.Method,
            context.Request.Path,
            context.Response.StatusCode,
            context.User.FindFirst("sub")?.Value ?? "anonymous",
            context.Connection.RemoteIpAddress);
    }
});

// Security audit endpoint (admin only)
app.MapGet("/api/admin/security-audit", (SecurityAuditService auditService) =>
{
    var result = auditService.RunAudit();
    return result.Passed 
        ? Results.Ok(result) 
        : Results.BadRequest(result);
})
.RequireAuthorization("AdminOnly");
```

---

## Exercises / Project Tasks

### Exercise 1: SQL Injection Testing
เขียน tests ที่ตรวจสอบ SQL injection:
```
- Input: "'; DROP TABLE Users; --"
- Input: "' OR '1'='1"
- ตรวจสอบว่า parameterized query ป้องกันได้
```

### Exercise 2: Security Headers
เพิ่ม security headers และตรวจสอบด้วย:
- https://securityheaders.com
- HSTS, CSP, X-Frame-Options

### Exercise 3: Rate Limiting
Implement rate limiting ที่:
- Login: 5 attempts per 15 minutes per IP
- API: 1000 requests per hour per user
- Registration: 3 accounts per hour per IP

### Exercise 4: Security Audit
รัน security checklist:
- dotnet list package --vulnerable
- OWASP dependency check
- Code review checklist

---

## สรุป

- **SQL Injection**: ใช้ parameterized queries เสมอ, ไม่ concatenate SQL string
- **XSS**: Razor auto-encodes output, เพิ่ม CSP headers
- **CSRF**: ใช้ AntiForgery tokens หรือ SameSite cookies
- **Secure Headers**: X-Frame-Options, HSTS, X-Content-Type-Options
- **JWT**: Short expiry, strong secret, validate ทุก claim
- **Passwords**: ใช้ BCrypt/PBKDF2, ตรวจสอบ complexity
- **Secrets**: อย่า hardcode, ใช้ env vars, Key Vault
- **Dependencies**: Scan regularly ด้วย dotnet audit, OWASP
- **Rate Limiting**: ป้องกัน brute force และ DDoS

---

## Part ถัดไป

**Part 089: gRPC ใน .NET** - เรียนรู้การสร้าง high-performance services ด้วย gRPC

---

*Part 088/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
