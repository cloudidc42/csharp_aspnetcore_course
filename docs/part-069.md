# Part 069: CORS และ Security Headers

## เนื้อหาใน Part นี้
- CORS Policy - การตั้งค่าและการใช้งาน
- AllowSpecificOrigins
- Security Headers (HSTS, X-Frame-Options, CSP)
- Rate Limiting ใน ASP.NET Core 7+
- Input Sanitization
- โปรแกรมตัวอย่าง: Secure API Configuration

---

## 1. CORS Policy

**CORS (Cross-Origin Resource Sharing)** เป็น mechanism ที่ browser ใช้ตรวจสอบว่า web app สามารถ request ไปยัง domain อื่นได้หรือไม่

### CORS Policy พื้นฐาน

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    // Policy สำหรับ development
    options.AddPolicy("DevelopmentPolicy", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });

    // Policy สำหรับ production
    options.AddPolicy("ProductionPolicy", policy =>
    {
        policy.WithOrigins(
                "https://myapp.com",
                "https://www.myapp.com",
                "https://admin.myapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
              .WithHeaders("Content-Type", "Authorization", "X-Correlation-Id")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });

    // Policy สำหรับ public API
    options.AddPolicy("PublicApiPolicy", policy =>
    {
        policy.AllowAnyOrigin()
              .WithMethods("GET")
              .WithHeaders("Content-Type")
              .SetPreflightMaxAge(TimeSpan.FromHours(1));
    });
});

var app = builder.Build();

// ใช้ CORS policy ตาม environment
if (app.Environment.IsDevelopment())
    app.UseCors("DevelopmentPolicy");
else
    app.UseCors("ProductionPolicy");
```

---

## 2. AllowSpecificOrigins

### Dynamic Origins จาก Configuration

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("ConfiguredOrigins", policy =>
    {
        var allowedOrigins = builder.Configuration
            .GetSection("Cors:AllowedOrigins")
            .Get<string[]>() ?? Array.Empty<string>();

        if (allowedOrigins.Any())
        {
            policy.WithOrigins(allowedOrigins)
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials();
        }
        else
        {
            // Fallback ถ้าไม่มีค่าใน config
            policy.WithOrigins("https://localhost:3000")
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        }
    });
});
```

```json
// appsettings.Production.json
{
  "Cors": {
    "AllowedOrigins": [
      "https://app.example.com",
      "https://www.example.com"
    ]
  }
}
```

### CORS per Controller/Action

```csharp
// Controllers/PublicController.cs
using Microsoft.AspNetCore.Cors;
using Microsoft.AspNetCore.Mvc;

namespace SecureApi.Controllers;

[ApiController]
[Route("api/[controller]")]
[EnableCors("PublicApiPolicy")]  // ใช้ specific policy
public class PublicController : ControllerBase
{
    [HttpGet("data")]
    public IActionResult GetPublicData()
    {
        return Ok(new { data = "public data" });
    }
}

[ApiController]
[Route("api/[controller]")]
[EnableCors("ProductionPolicy")]
public class PrivateController : ControllerBase
{
    [HttpGet("private")]
    [DisableCors]  // ปิด CORS สำหรับ endpoint นี้
    public IActionResult GetPrivate()
    {
        return Ok();
    }
}
```

---

## 3. Security Headers

### เพิ่ม Security Headers ด้วย Middleware

```csharp
// Middleware/SecurityHeadersMiddleware.cs
namespace SecureApi.Middleware;

public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // X-Content-Type-Options - ป้องกัน MIME sniffing
        context.Response.Headers.Append("X-Content-Type-Options", "nosniff");

        // X-Frame-Options - ป้องกัน clickjacking
        context.Response.Headers.Append("X-Frame-Options", "DENY");

        // X-XSS-Protection - เปิด browser XSS filter (legacy)
        context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");

        // Referrer-Policy - ควบคุม referrer information
        context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

        // Permissions-Policy - ปิด browser features ที่ไม่จำเป็น
        context.Response.Headers.Append("Permissions-Policy",
            "accelerometer=(), camera=(), geolocation=(), gyroscope=(), " +
            "magnetometer=(), microphone=(), payment=(), usb=()");

        // Content-Security-Policy
        context.Response.Headers.Append("Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' https://cdnjs.cloudflare.com; " +
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; " +
            "font-src 'self' https://fonts.gstatic.com; " +
            "img-src 'self' data: https:; " +
            "connect-src 'self'; " +
            "frame-ancestors 'none';");

        // ลบ Server header (ซ่อนข้อมูล server)
        context.Response.Headers.Remove("Server");
        context.Response.Headers.Remove("X-Powered-By");

        await _next(context);
    }
}

// Extension method
public static class SecurityHeadersMiddlewareExtensions
{
    public static IApplicationBuilder UseSecurityHeaders(this IApplicationBuilder app)
    {
        return app.UseMiddleware<SecurityHeadersMiddleware>();
    }
}
```

### HSTS (HTTP Strict Transport Security)

```csharp
// Program.cs
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
    options.ExcludedHosts.Add("localhost");
    options.ExcludedHosts.Add("127.0.0.1");
});

// HTTPS Redirection
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}

app.UseSecurityHeaders(); // Custom security headers
```

### ใช้ NWebSec Library

```bash
dotnet add package NWebSec.AspNetCore.Middleware
```

```csharp
// Program.cs
app.UseXContentTypeOptions();
app.UseReferrerPolicy(opts => opts.StrictOriginWhenCrossOrigin());
app.UseXfo(opts => opts.Deny());
app.UseCsp(opts => opts
    .DefaultSources(s => s.Self())
    .ScriptSources(s => s.Self().UnsafeInline().CustomSources("https://cdnjs.cloudflare.com"))
    .StyleSources(s => s.Self().UnsafeInline())
    .ImageSources(s => s.Self().CustomSources("data:"))
    .FontSources(s => s.Self())
    .ConnectSources(s => s.Self())
    .FrameAncestors(s => s.None())
);
```

---

## 4. Rate Limiting (ASP.NET Core 7+)

**Rate Limiting** ป้องกัน API จากการถูก abuse หรือ DDoS attacks

### การตั้งค่า Rate Limiting

```csharp
// Program.cs
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Global fallback policy
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        httpContext =>
        {
            var clientIp = httpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";
            return RateLimitPartition.GetFixedWindowLimiter(
                partitionKey: clientIp,
                factory: partition => new FixedWindowRateLimiterOptions
                {
                    AutoReplenishment = true,
                    PermitLimit = 1000,
                    Window = TimeSpan.FromHour(1)
                });
        });

    // Fixed Window - limit per IP per window
    options.AddFixedWindowLimiter("FixedWindow", opts =>
    {
        opts.PermitLimit = 100;
        opts.Window = TimeSpan.FromMinutes(1);
        opts.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opts.QueueLimit = 5;
    });

    // Sliding Window - smoother than fixed window
    options.AddSlidingWindowLimiter("SlidingWindow", opts =>
    {
        opts.PermitLimit = 100;
        opts.Window = TimeSpan.FromMinutes(1);
        opts.SegmentsPerWindow = 6; // 10-second segments
        opts.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opts.QueueLimit = 2;
    });

    // Token Bucket - burst traffic allowed
    options.AddTokenBucketLimiter("TokenBucket", opts =>
    {
        opts.TokenLimit = 100;
        opts.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opts.QueueLimit = 3;
        opts.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        opts.TokensPerPeriod = 20;
        opts.AutoReplenishment = true;
    });

    // Concurrency Limiter - limit concurrent requests
    options.AddConcurrencyLimiter("Concurrency", opts =>
    {
        opts.PermitLimit = 10;
        opts.QueueProcessingOrder = QueueProcessingOrder.NewestFirst;
        opts.QueueLimit = 5;
    });

    // Per-User rate limiting
    options.AddPolicy("PerUser", httpContext =>
    {
        var userId = httpContext.User.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: userId,
            factory: partition => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = userId == "anonymous" ? 10 : 100,
                Window = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6
            });
    });
});
```

### ใช้ Rate Limiting ใน Controller

```csharp
// Controllers/ApiController.cs
using Microsoft.AspNetCore.RateLimiting;

[ApiController]
[Route("api/[controller]")]
[EnableRateLimiting("FixedWindow")]  // ใช้กับทั้ง controller
public class ProductsController : ControllerBase
{
    // ใช้ policy อื่นสำหรับ endpoint นี้
    [HttpGet]
    [EnableRateLimiting("SlidingWindow")]
    public IActionResult GetAll() => Ok();

    // ปิด rate limiting สำหรับ endpoint นี้
    [HttpGet("public")]
    [DisableRateLimiting]
    public IActionResult GetPublic() => Ok();

    // Upload มี rate limit เข้มกว่า
    [HttpPost("upload")]
    [EnableRateLimiting("TokenBucket")]
    public IActionResult Upload() => Ok();
}
```

### Custom Rate Limiting Response

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = 429;
        context.HttpContext.Response.ContentType = "application/json";

        // เพิ่ม Retry-After header
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();
        }

        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            type = "https://tools.ietf.org/html/rfc6585#section-4",
            title = "Too Many Requests",
            status = 429,
            detail = "คุณส่ง request มากเกินไป กรุณาลองใหม่ภายหลัง",
            retryAfter = context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var ra)
                ? (int)ra.TotalSeconds
                : 60
        }, token);
    };
});
```

---

## 5. Input Sanitization

### Anti-XSS

```bash
dotnet add package HtmlSanitizer
```

```csharp
// Services/InputSanitizationService.cs
using Ganss.Xss;
using System.Text.RegularExpressions;

namespace SecureApi.Services;

public interface IInputSanitizationService
{
    string SanitizeHtml(string input);
    string SanitizePlainText(string input);
    bool IsSafeUrl(string url);
    string SanitizeFileName(string fileName);
    string RemoveSqlInjectionRisk(string input);
}

public class InputSanitizationService : IInputSanitizationService
{
    private readonly HtmlSanitizer _sanitizer;

    public InputSanitizationService()
    {
        _sanitizer = new HtmlSanitizer();

        // อนุญาต tags ที่ safe
        _sanitizer.AllowedTags.Clear();
        foreach (var tag in new[] { "p", "br", "b", "i", "u", "strong", "em",
            "ul", "ol", "li", "h1", "h2", "h3", "h4", "h5", "h6",
            "table", "tr", "td", "th", "thead", "tbody", "a", "img" })
        {
            _sanitizer.AllowedTags.Add(tag);
        }

        // อนุญาต attributes ที่ safe
        _sanitizer.AllowedAttributes.Clear();
        _sanitizer.AllowedAttributes.Add("href");
        _sanitizer.AllowedAttributes.Add("src");
        _sanitizer.AllowedAttributes.Add("alt");
        _sanitizer.AllowedAttributes.Add("class");
        _sanitizer.AllowedAttributes.Add("style");

        // อนุญาต URL schemes
        _sanitizer.AllowedSchemes.Clear();
        _sanitizer.AllowedSchemes.Add("http");
        _sanitizer.AllowedSchemes.Add("https");
        _sanitizer.AllowedSchemes.Add("mailto");
    }

    public string SanitizeHtml(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return string.Empty;

        return _sanitizer.Sanitize(input);
    }

    public string SanitizePlainText(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return string.Empty;

        // ลบ HTML tags ทั้งหมด
        var noHtml = Regex.Replace(input, "<[^>]+>", string.Empty);

        // Normalize whitespace
        var normalized = Regex.Replace(noHtml, @"\s+", " ").Trim();

        return normalized;
    }

    public bool IsSafeUrl(string url)
    {
        if (string.IsNullOrWhiteSpace(url))
            return false;

        if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
            return false;

        return uri.Scheme == "https" || uri.Scheme == "http";
    }

    public string SanitizeFileName(string fileName)
    {
        // ลบ characters ที่อันตราย
        var invalidChars = Path.GetInvalidFileNameChars()
            .Concat(new[] { '\\', '/', ':', '*', '?', '"', '<', '>', '|', '\0' })
            .ToArray();

        var sanitized = string.Join("_", fileName.Split(invalidChars));

        // ป้องกัน path traversal
        sanitized = sanitized.Replace("..", "_");

        // จำกัดความยาว
        if (sanitized.Length > 255)
            sanitized = sanitized.Substring(0, 255);

        return sanitized;
    }

    public string RemoveSqlInjectionRisk(string input)
    {
        // ไม่ควรใช้วิธีนี้เป็น primary defense (ใช้ parameterized queries แทน)
        // แต่เป็น additional layer
        var dangerous = new[] { "'", "--", ";", "/*", "*/", "xp_", "EXEC", "EXECUTE", "DROP", "DELETE" };

        foreach (var pattern in dangerous)
        {
            input = input.Replace(pattern, string.Empty, StringComparison.OrdinalIgnoreCase);
        }

        return input;
    }
}
```

### Model Validation

```csharp
// Validators/ProductValidator.cs
using FluentValidation;

namespace SecureApi.Validators;

public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("ชื่อสินค้าจำเป็น")
            .Length(3, 200).WithMessage("ชื่อต้องมี 3-200 ตัวอักษร")
            .Matches(@"^[a-zA-Zก-๙0-9\s\-_\.]+$")
            .WithMessage("ชื่อสินค้าต้องมีแค่ตัวอักษร ตัวเลข และ - _ .");

        RuleFor(x => x.Description)
            .MaximumLength(2000).WithMessage("รายละเอียดยาวเกินไป")
            .When(x => x.Description != null);

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("ราคาต้องมากกว่า 0")
            .LessThanOrEqualTo(9_999_999).WithMessage("ราคาสูงเกินไป");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("จำนวนสต็อกต้องไม่ติดลบ")
            .LessThanOrEqualTo(1_000_000).WithMessage("จำนวนสูงเกินไป");

        RuleFor(x => x.WebsiteUrl)
            .Must(url => Uri.TryCreate(url, UriKind.Absolute, out var uri)
                && (uri.Scheme == "https" || uri.Scheme == "http"))
            .WithMessage("URL ต้องเป็น http หรือ https")
            .When(x => !string.IsNullOrEmpty(x.WebsiteUrl));
    }
}
```

---

## โปรแกรมตัวอย่าง: Secure API Configuration

การตั้งค่า security ที่สมบูรณ์สำหรับ production API

### Security Settings

```csharp
// Settings/SecuritySettings.cs
namespace SecureApi.Settings;

public class SecuritySettings
{
    public CorsSettings Cors { get; set; } = new();
    public RateLimitSettings RateLimit { get; set; } = new();
    public SecurityHeaderSettings Headers { get; set; } = new();
}

public class CorsSettings
{
    public string[] AllowedOrigins { get; set; } = Array.Empty<string>();
    public string[] AllowedMethods { get; set; } = new[] { "GET", "POST", "PUT", "DELETE" };
    public string[] AllowedHeaders { get; set; } = new[] { "Content-Type", "Authorization" };
    public bool AllowCredentials { get; set; } = true;
    public int PreflightMaxAgeMinutes { get; set; } = 10;
}

public class RateLimitSettings
{
    public int RequestsPerMinute { get; set; } = 100;
    public int BurstSize { get; set; } = 200;
    public int AuthenticatedRequestsPerMinute { get; set; } = 1000;
    public string[] WhitelistedIps { get; set; } = Array.Empty<string>();
}

public class SecurityHeaderSettings
{
    public bool EnableHsts { get; set; } = true;
    public int HstsMaxAgeDays { get; set; } = 365;
    public bool HstsIncludeSubdomains { get; set; } = true;
    public string ContentSecurityPolicy { get; set; } =
        "default-src 'self'; script-src 'self'; style-src 'self';";
}
```

### Program.cs สมบูรณ์

```csharp
// Program.cs
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;
using SecureApi.Middleware;
using SecureApi.Services;
using SecureApi.Settings;
using FluentValidation;
using FluentValidation.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

// Security settings
var securitySettings = builder.Configuration
    .GetSection("Security")
    .Get<SecuritySettings>() ?? new SecuritySettings();

// ============ CORS ============
builder.Services.AddCors(options =>
{
    options.AddPolicy("DefaultPolicy", policy =>
    {
        if (builder.Environment.IsDevelopment())
        {
            policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader();
        }
        else
        {
            var origins = securitySettings.Cors.AllowedOrigins;
            if (origins.Any())
                policy.WithOrigins(origins);

            policy.WithMethods(securitySettings.Cors.AllowedMethods)
                  .WithHeaders(securitySettings.Cors.AllowedHeaders);

            if (securitySettings.Cors.AllowCredentials && origins.Any())
                policy.AllowCredentials();

            policy.SetPreflightMaxAge(
                TimeSpan.FromMinutes(securitySettings.Cors.PreflightMaxAgeMinutes));
        }
    });
});

// ============ Rate Limiting ============
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.ContentType = "application/json";
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString();
        }

        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            status = 429,
            title = "Too Many Requests",
            detail = "คุณส่ง request มากเกินไป กรุณารอสักครู่"
        }, token);
    };

    // Default per-IP policy
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        httpContext =>
        {
            var clientIp = httpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";

            // Whitelist IPs
            if (securitySettings.RateLimit.WhitelistedIps.Contains(clientIp))
            {
                return RateLimitPartition.GetNoLimiter(clientIp);
            }

            // Authenticated users get higher limits
            var userId = httpContext.User.FindFirst("sub")?.Value;
            var permitLimit = userId != null
                ? securitySettings.RateLimit.AuthenticatedRequestsPerMinute
                : securitySettings.RateLimit.RequestsPerMinute;

            var partitionKey = userId ?? clientIp;

            return RateLimitPartition.GetSlidingWindowLimiter(
                partitionKey: partitionKey,
                factory: _ => new SlidingWindowRateLimiterOptions
                {
                    PermitLimit = permitLimit,
                    Window = TimeSpan.FromMinutes(1),
                    SegmentsPerWindow = 6,
                    QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                    QueueLimit = 5
                });
        });

    // Strict policy สำหรับ auth endpoints
    options.AddSlidingWindowLimiter("AuthPolicy", opts =>
    {
        opts.PermitLimit = 5;
        opts.Window = TimeSpan.FromMinutes(15);
        opts.SegmentsPerWindow = 5;
        opts.QueueLimit = 0;
    });
});

// ============ HTTPS ============
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(securitySettings.Headers.HstsMaxAgeDays);
    options.IncludeSubDomains = securitySettings.Headers.HstsIncludeSubdomains;
    options.Preload = true;
});

// ============ Validation ============
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// ============ Services ============
builder.Services.AddSingleton<IInputSanitizationService, InputSanitizationService>();
builder.Services.AddControllers();

var app = builder.Build();

// ============ Middleware Pipeline ============

// Security headers (ก่อนทุกอย่าง)
app.UseSecurityHeaders();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}

app.UseCors("DefaultPolicy");
app.UseRateLimiter();

// Authentication & Authorization
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### Secure Controller

```csharp
// Controllers/SecureProductsController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;
using SecureApi.Services;

namespace SecureApi.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class SecureProductsController : ControllerBase
{
    private readonly IInputSanitizationService _sanitizer;
    private readonly ILogger<SecureProductsController> _logger;

    public SecureProductsController(
        IInputSanitizationService sanitizer,
        ILogger<SecureProductsController> logger)
    {
        _sanitizer = sanitizer;
        _logger = logger;
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetAll([FromQuery] string? search)
    {
        // Sanitize query parameters
        var safeSearch = search != null ? _sanitizer.SanitizePlainText(search) : null;

        _logger.LogInformation("Search products: {Search}", safeSearch);
        return Ok(new { Search = safeSearch });
    }

    [HttpPost]
    [EnableRateLimiting("AuthPolicy")]
    public IActionResult Create([FromBody] CreateProductRequest request)
    {
        // Sanitize input
        request.Name = _sanitizer.SanitizePlainText(request.Name);
        request.Description = _sanitizer.SanitizeHtml(request.Description ?? string.Empty);

        // Log audit
        var userId = User.FindFirst("sub")?.Value;
        _logger.LogInformation("User {UserId} creating product: {Name}", userId, request.Name);

        return CreatedAtAction(nameof(GetById), new { id = 1 }, request);
    }

    [HttpGet("{id:int}")]
    [AllowAnonymous]
    public IActionResult GetById(int id)
    {
        if (id <= 0) return BadRequest("ID ต้องมากกว่า 0");
        return Ok(new { Id = id });
    }

    // IDOR Prevention
    [HttpPut("{id:int}")]
    public IActionResult Update(int id, [FromBody] UpdateProductRequest request)
    {
        var userId = User.FindFirst("sub")?.Value;

        // ตรวจสอบว่า user เป็นเจ้าของ resource
        if (!IsOwnerOfProduct(id, userId))
        {
            _logger.LogWarning(
                "User {UserId} attempted to modify product {ProductId} without permission",
                userId, id);
            return Forbid();
        }

        return Ok();
    }

    private bool IsOwnerOfProduct(int productId, string? userId)
    {
        // TODO: ตรวจสอบกับ database จริงๆ
        return userId != null;
    }
}
```

### appsettings.json Security Config

```json
{
  "Security": {
    "Cors": {
      "AllowedOrigins": [
        "https://app.example.com",
        "https://www.example.com"
      ],
      "AllowedMethods": ["GET", "POST", "PUT", "DELETE", "PATCH"],
      "AllowedHeaders": ["Content-Type", "Authorization", "X-Correlation-Id"],
      "AllowCredentials": true,
      "PreflightMaxAgeMinutes": 10
    },
    "RateLimit": {
      "RequestsPerMinute": 100,
      "BurstSize": 200,
      "AuthenticatedRequestsPerMinute": 1000,
      "WhitelistedIps": ["127.0.0.1", "::1"]
    },
    "Headers": {
      "EnableHsts": true,
      "HstsMaxAgeDays": 365,
      "HstsIncludeSubdomains": true,
      "ContentSecurityPolicy": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';"
    }
  }
}
```

---

## Exercises

### Exercise 1: CSRF Protection
เพิ่ม CSRF protection สำหรับ form-based endpoints

### Exercise 2: IP Blacklist
สร้าง middleware ที่ block requests จาก IP addresses ที่ระบุใน blocklist

### Exercise 3: Request Size Limit
สร้าง attribute สำหรับ limit request body size per endpoint

### Exercise 4: API Key Rate Limiting
ปรับ rate limiting ให้แยก limit ตาม API key ที่ใช้

### Exercise 5: Security Audit Log
สร้าง middleware ที่ log security events เช่น failed authentication, rate limit exceeded

---

## สรุป

- **CORS** ควรตั้งค่า specific origins ใน production ห้ามใช้ AllowAnyOrigin กับ credentials
- **Security headers** เพิ่ม layer of protection จาก XSS, clickjacking, MIME sniffing
- **HSTS** บังคับ HTTPS สำหรับ browser ที่เคยเข้าชม
- **Rate limiting** ป้องกัน brute force และ DDoS
- **Input sanitization** เป็น defense-in-depth ร่วมกับ parameterized queries และ validation

---

## Part ถัดไป

**Part 070: Minimal API ขั้นสูง** - เรียนรู้ Route Groups, Filters, TypedResults ใน ASP.NET Core 7+

---

*Part 069/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
