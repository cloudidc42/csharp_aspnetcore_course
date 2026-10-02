# Part 043: Middleware Pipeline

## เนื้อหาใน Part นี้
- Middleware concept
- Built-in middleware: UseStaticFiles, UseRouting, UseAuthentication
- การสร้าง Custom Middleware
- Middleware order สำคัญมาก
- Request/Response pipeline
- โปรแกรมตัวอย่าง: Request logging middleware

---

## 1. Middleware คืออะไร?

Middleware คือ software component ที่ประมวลผล HTTP requests และ responses ใน pipeline ของ ASP.NET Core

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                  Middleware Pipeline                  │
│                                                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐ │
│  │  Logging │ → │   Auth   │ → │    Application   │ │
│  │Middleware│ ← │Middleware│ ← │    (Endpoint)    │ │
│  └──────────┘   └──────────┘   └──────────────────┘ │
│       ↑               ↑                  ↑           │
│    Request          Request           Request        │
│    ↓ Response       ↓ Response        ↓ Response    │
└─────────────────────────────────────────────────────┘
    │
    ▼
HTTP Response
```

แต่ละ Middleware สามารถ:
1. **ทำงานก่อน** request ส่งไปยัง middleware ถัดไป
2. **เรียก middleware ถัดไป** หรือ **short-circuit pipeline**
3. **ทำงานหลัง** response กลับมาจาก middleware ถัดไป

---

## 2. Middleware ใน Code

### 2.1 รูปแบบ Middleware

```csharp
// รูปแบบที่ 1: app.Use (anonymous middleware)
app.Use(async (HttpContext context, RequestDelegate next) =>
{
    // ทำงานก่อนส่งต่อ (before)
    Console.WriteLine($"Before: {context.Request.Path}");
    
    await next(context);  // ส่งต่อไป middleware ถัดไป
    
    // ทำงานหลังได้รับ response (after)
    Console.WriteLine($"After: {context.Response.StatusCode}");
});

// รูปแบบที่ 2: app.Run (terminal middleware - ไม่เรียก next)
app.Run(async (HttpContext context) =>
{
    await context.Response.WriteAsync("Hello from terminal middleware!");
    // ไม่มี next() - pipeline หยุดที่นี่
});

// รูปแบบที่ 3: app.UseMiddleware<T> (class-based)
app.UseMiddleware<RequestLoggingMiddleware>();
```

### 2.2 การ Short-circuit Pipeline

```csharp
app.Use(async (context, next) =>
{
    // ตรวจสอบ API key
    if (!context.Request.Headers.TryGetValue("X-Api-Key", out var apiKey) 
        || apiKey != "secret-key")
    {
        // Short-circuit: ส่ง response เลย ไม่ต้องไป middleware ถัดไป
        context.Response.StatusCode = 401;
        await context.Response.WriteAsJsonAsync(new { Error = "Unauthorized" });
        return; // ไม่เรียก next()!
    }
    
    await next(context);
});
```

---

## 3. Built-in Middleware

### 3.1 UseStaticFiles

```csharp
// Serve static files จาก wwwroot/
app.UseStaticFiles();

// Custom static file options
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "StaticFiles")),
    RequestPath = "/files",    // URL path ที่ serve files
    OnPrepareResponse = ctx =>
    {
        // เพิ่ม cache headers
        ctx.Context.Response.Headers.CacheControl = "public,max-age=600";
    }
});

// UseDirectoryBrowser - แสดง directory listing
app.UseDirectoryBrowser(new DirectoryBrowserOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "wwwroot")),
    RequestPath = "/browse"
});

// UseDefaultFiles - serve index.html by default
app.UseDefaultFiles();
app.UseStaticFiles();
```

### 3.2 UseRouting

```csharp
// เปิดใช้ routing middleware
app.UseRouting();

// หลังจาก UseRouting(), สามารถใช้ routing ใน middleware อื่นได้
app.Use(async (context, next) =>
{
    // ดึง route data
    var endpoint = context.GetEndpoint();
    var routeData = context.GetRouteData();
    
    Console.WriteLine($"Endpoint: {endpoint?.DisplayName}");
    
    await next(context);
});

// MapControllers ต้องอยู่หลัง UseRouting()
app.MapControllers();
```

### 3.3 UseAuthentication และ UseAuthorization

```csharp
// ต้องเรียงลำดับนี้:
app.UseAuthentication();  // ตรวจสอบว่าเป็นใคร (who are you?)
app.UseAuthorization();   // ตรวจสอบว่ามีสิทธิ์หรือไม่ (what can you do?)

// ตัวอย่าง JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
    options.AddPolicy("PremiumUser", policy => 
        policy.RequireClaim("subscription", "premium"));
});
```

### 3.4 UseExceptionHandler

```csharp
// Production: handle exceptions gracefully
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";
        
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        
        var errorResponse = new
        {
            StatusCode = 500,
            Message = "เกิดข้อผิดพลาดภายใน",
            Detail = app.Environment.IsDevelopment() ? exception?.Message : null
        };
        
        await context.Response.WriteAsJsonAsync(errorResponse);
    });
});

// Development: show detailed error page
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
```

### 3.5 UseHttpsRedirection และ UseHsts

```csharp
// redirect HTTP -> HTTPS
app.UseHttpsRedirection();

// เพิ่ม Strict-Transport-Security header
app.UseHsts();
// builder.Services.AddHsts(options =>
// {
//     options.Preload = true;
//     options.IncludeSubDomains = true;
//     options.MaxAge = TimeSpan.FromDays(365);
// });
```

### 3.6 UseCors

```csharp
// กำหนด CORS policy
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
        policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());

    options.AddPolicy("AllowSpecific", policy =>
        policy.WithOrigins("https://myapp.com", "https://admin.myapp.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Content-Type", "Authorization")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10)));
});

// ใช้ CORS
app.UseCors("AllowSpecific");

// CORS เฉพาะ endpoint
app.MapGet("/public", () => "Public endpoint")
   .RequireCors("AllowAll");

app.MapGet("/private", () => "Private endpoint")
   .RequireCors("AllowSpecific")
   .RequireAuthorization();
```

### 3.7 UseResponseCompression

```csharp
// เพิ่ม response compression
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/octet-stream", "image/svg+xml" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = CompressionLevel.Fastest;
});

app.UseResponseCompression();
```

---

## 4. Custom Middleware

### 4.1 Inline Middleware (Lambda)

```csharp
// Simple request timer
app.Use(async (context, next) =>
{
    var stopwatch = System.Diagnostics.Stopwatch.StartNew();
    
    await next(context);
    
    stopwatch.Stop();
    context.Response.Headers.Append("X-Response-Time", 
        $"{stopwatch.ElapsedMilliseconds}ms");
});
```

### 4.2 Class-based Middleware

รูปแบบมาตรฐานสำหรับ middleware class:

```csharp
// RequestLoggingMiddleware.cs
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    // Constructor injection - รับ RequestDelegate และ services
    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // Invoke หรือ InvokeAsync - ต้อง public
    public async Task InvokeAsync(HttpContext context)
    {
        // ทำงานก่อน request
        var requestTime = DateTime.UtcNow;
        var traceId = Activity.Current?.TraceId.ToString() ?? Guid.NewGuid().ToString();
        
        _logger.LogInformation(
            "REQUEST [{TraceId}] {Method} {Path} started at {Time}",
            traceId, context.Request.Method, context.Request.Path, requestTime);

        try
        {
            await _next(context);  // ส่งต่อ
        }
        finally
        {
            // ทำงานหลัง response (ทำงานแม้มี exception)
            var duration = (DateTime.UtcNow - requestTime).TotalMilliseconds;
            
            _logger.LogInformation(
                "RESPONSE [{TraceId}] {Method} {Path} {StatusCode} completed in {Duration}ms",
                traceId, context.Request.Method, context.Request.Path,
                context.Response.StatusCode, duration);
        }
    }
}

// Extension method สำหรับ registration
public static class RequestLoggingMiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<RequestLoggingMiddleware>();
    }
}

// ใช้งาน
app.UseRequestLogging();
```

### 4.3 Middleware ที่ใช้ Scoped Services

Middleware constructor รับเฉพาะ Singleton services ได้เท่านั้น  
สำหรับ Scoped services ต้องรับใน InvokeAsync:

```csharp
public class AuditMiddleware
{
    private readonly RequestDelegate _next;
    // Singleton services รับใน constructor
    private readonly ILogger<AuditMiddleware> _logger;

    public AuditMiddleware(RequestDelegate next, ILogger<AuditMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    // Scoped services รับใน InvokeAsync
    public async Task InvokeAsync(HttpContext context, IAuditService auditService)
    {
        await _next(context);
        
        // ใช้ scoped service หลัง request
        if (context.User.Identity?.IsAuthenticated == true)
        {
            await auditService.LogAsync(new AuditEntry
            {
                UserId = context.User.Identity.Name ?? "unknown",
                Action = $"{context.Request.Method} {context.Request.Path}",
                StatusCode = context.Response.StatusCode,
                Timestamp = DateTime.UtcNow
            });
        }
    }
}
```

### 4.4 Middleware Factory Pattern (IMiddleware)

```csharp
// ใช้ IMiddleware interface - Scoped lifetime ด้วย DI
public class ApiKeyMiddleware : IMiddleware
{
    private readonly IApiKeyService _apiKeyService;
    private readonly ILogger<ApiKeyMiddleware> _logger;

    public ApiKeyMiddleware(IApiKeyService apiKeyService, ILogger<ApiKeyMiddleware> logger)
    {
        _apiKeyService = apiKeyService;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        if (!context.Request.Headers.TryGetValue("X-Api-Key", out var apiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { Error = "API key required" });
            return;
        }

        if (!await _apiKeyService.ValidateAsync(apiKey!))
        {
            _logger.LogWarning("Invalid API key attempt from {IP}", 
                context.Connection.RemoteIpAddress);
            context.Response.StatusCode = 403;
            await context.Response.WriteAsJsonAsync(new { Error = "Invalid API key" });
            return;
        }

        await next(context);
    }
}

// ต้อง register ใน DI container
builder.Services.AddTransient<ApiKeyMiddleware>();
// และใช้งาน
app.UseMiddleware<ApiKeyMiddleware>();
```

---

## 5. Middleware Order - ลำดับที่ถูกต้อง

```csharp
// ลำดับที่แนะนำสำหรับ ASP.NET Core Web API
var app = builder.Build();

// 1. Exception handling - ต้องอยู่ต้นสุดเสมอ
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// 2. HTTPS Redirection
app.UseHttpsRedirection();

// 3. Static files (ก่อน routing เพื่อประสิทธิภาพ)
app.UseStaticFiles();

// 4. Custom middleware ที่ต้องทำงานก่อน routing
app.UseRequestLogging();

// 5. Routing
app.UseRouting();

// 6. CORS (หลัง UseRouting แต่ก่อน UseAuthentication)
app.UseCors();

// 7. Rate limiting
app.UseRateLimiter();

// 8. Authentication
app.UseAuthentication();

// 9. Authorization
app.UseAuthorization();

// 10. Response Caching
app.UseResponseCaching();

// 11. Endpoints (ท้ายสุด)
app.MapControllers();
app.MapRazorPages();
```

---

## 6. Request/Response Pipeline อย่างละเอียด

### 6.1 HttpContext

`HttpContext` มีข้อมูล request และ response ทั้งหมด:

```csharp
app.Use(async (context, next) =>
{
    // ข้อมูล Request
    var method = context.Request.Method;          // GET, POST, etc.
    var path = context.Request.Path;              // /api/users
    var queryString = context.Request.QueryString; // ?page=1&size=10
    var headers = context.Request.Headers;         // HTTP headers
    var body = context.Request.Body;              // Request body stream
    var contentType = context.Request.ContentType; // application/json
    var host = context.Request.Host;              // localhost:5000
    var scheme = context.Request.Scheme;          // http or https
    var isHttps = context.Request.IsHttps;
    var cookies = context.Request.Cookies;
    var form = context.Request.Form;              // Form data
    var ip = context.Connection.RemoteIpAddress;

    // Query parameters
    var page = context.Request.Query["page"].ToString();
    var name = context.Request.Query.TryGetValue("name", out var n) ? n.ToString() : null;

    // Request body (อ่านแบบ JSON)
    if (context.Request.HasJsonContentType())
    {
        var data = await context.Request.ReadFromJsonAsync<MyModel>();
    }

    // User information (หลัง Authentication middleware)
    var isAuthenticated = context.User.Identity?.IsAuthenticated ?? false;
    var username = context.User.Identity?.Name;
    var claims = context.User.Claims.ToList();

    await next(context);

    // ข้อมูล Response
    var statusCode = context.Response.StatusCode;
    var responseHeaders = context.Response.Headers;
    
    // เพิ่ม response headers
    context.Response.Headers.Append("X-Custom-Header", "my-value");
    context.Response.Headers.CacheControl = "no-cache";
});
```

### 6.2 Middleware ที่อ่าน Request Body

```csharp
// Request body อ่านได้ครั้งเดียวโดย default
// ต้อง enable buffering ถ้าต้องการอ่านหลายครั้ง
app.Use(async (context, next) =>
{
    // Enable buffering ก่อนอ่าน body
    context.Request.EnableBuffering();
    
    // อ่าน body
    var body = await new StreamReader(context.Request.Body).ReadToEndAsync();
    
    // Reset position ให้ middleware ถัดไปอ่านได้
    context.Request.Body.Position = 0;
    
    Console.WriteLine($"Request body: {body}");
    
    await next(context);
});
```

### 6.3 Middleware ที่อ่าน/แก้ Response Body

```csharp
// แก้ไข response body ต้อง wrap stream
app.Use(async (context, next) =>
{
    var originalBody = context.Response.Body;
    
    try
    {
        using var memStream = new MemoryStream();
        context.Response.Body = memStream;
        
        await next(context);
        
        // อ่าน response ที่สร้างแล้ว
        memStream.Seek(0, SeekOrigin.Begin);
        var responseBody = await new StreamReader(memStream).ReadToEndAsync();
        
        Console.WriteLine($"Response body: {responseBody}");
        
        // Write response กลับไป
        memStream.Seek(0, SeekOrigin.Begin);
        await memStream.CopyToAsync(originalBody);
    }
    finally
    {
        context.Response.Body = originalBody;
    }
});
```

---

## 7. โปรแกรมตัวอย่าง: Request Logging Middleware

มาสร้าง request logging middleware ที่สมบูรณ์กัน:

```csharp
// Program.cs
using System.Diagnostics;
using System.Text;
using Microsoft.IO;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register RecyclableMemoryStreamManager สำหรับประสิทธิภาพ
builder.Services.AddSingleton(new RecyclableMemoryStreamManager());

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// ใช้ Request Logging Middleware
app.UseMiddleware<AdvancedRequestLoggingMiddleware>();

// Sample endpoints สำหรับทดสอบ
app.MapGet("/api/test", () => new { Status = "OK", Time = DateTime.UtcNow });

app.MapPost("/api/echo", async (HttpContext context) =>
{
    var body = await new StreamReader(context.Request.Body).ReadToEndAsync();
    return Results.Ok(new { EchoedBody = body, Time = DateTime.UtcNow });
});

app.MapGet("/api/error", () =>
{
    throw new InvalidOperationException("Test error for logging!");
});

app.Run();

// ============================================
// ADVANCED REQUEST LOGGING MIDDLEWARE
// ============================================

public class AdvancedRequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<AdvancedRequestLoggingMiddleware> _logger;
    private readonly RecyclableMemoryStreamManager _streamManager;

    // Headers ที่ไม่ควร log (sensitive)
    private static readonly HashSet<string> SensitiveHeaders = new(StringComparer.OrdinalIgnoreCase)
    {
        "Authorization", "Cookie", "X-Api-Key", "X-Auth-Token"
    };

    public AdvancedRequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<AdvancedRequestLoggingMiddleware> logger,
        RecyclableMemoryStreamManager streamManager)
    {
        _next = next;
        _logger = logger;
        _streamManager = streamManager;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // สร้าง unique ID สำหรับแต่ละ request
        var requestId = context.TraceIdentifier;
        var stopwatch = Stopwatch.StartNew();

        // Log request
        await LogRequestAsync(context, requestId);

        // Capture response body
        var originalResponseBody = context.Response.Body;
        await using var responseBody = _streamManager.GetStream();
        context.Response.Body = responseBody;

        Exception? exception = null;
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            exception = ex;
            throw;
        }
        finally
        {
            stopwatch.Stop();

            // Copy response back
            responseBody.Seek(0, SeekOrigin.Begin);
            await responseBody.CopyToAsync(originalResponseBody);
            context.Response.Body = originalResponseBody;

            // Log response
            await LogResponseAsync(context, requestId, stopwatch.Elapsed, exception);
        }
    }

    private async Task LogRequestAsync(HttpContext context, string requestId)
    {
        context.Request.EnableBuffering();

        var requestLog = new StringBuilder();
        requestLog.AppendLine($"┌── REQUEST [{requestId}] ──────────────────────");
        requestLog.AppendLine($"│ {context.Request.Method} {context.Request.Scheme}://{context.Request.Host}{context.Request.Path}{context.Request.QueryString}");
        requestLog.AppendLine($"│ Time: {DateTime.UtcNow:yyyy-MM-dd HH:mm:ss.fff} UTC");
        requestLog.AppendLine($"│ IP: {context.Connection.RemoteIpAddress}");

        // Log headers (ยกเว้น sensitive)
        foreach (var header in context.Request.Headers)
        {
            var value = SensitiveHeaders.Contains(header.Key)
                ? "[REDACTED]"
                : header.Value.ToString();
            requestLog.AppendLine($"│ Header: {header.Key}: {value}");
        }

        // Log request body (ถ้ามี)
        if (context.Request.ContentLength > 0 && context.Request.ContentLength < 10240) // max 10KB
        {
            var body = await ReadBodyAsync(context.Request.Body);
            context.Request.Body.Position = 0;

            if (!string.IsNullOrWhiteSpace(body))
            {
                requestLog.AppendLine($"│ Body ({context.Request.ContentLength} bytes): {TruncateBody(body)}");
            }
        }

        requestLog.AppendLine($"└────────────────────────────────────────────");

        _logger.LogInformation(requestLog.ToString());
    }

    private async Task LogResponseAsync(
        HttpContext context, 
        string requestId, 
        TimeSpan duration, 
        Exception? exception)
    {
        var responseBody = context.Response.Body;
        responseBody.Seek(0, SeekOrigin.Begin);
        var body = await ReadBodyAsync(responseBody);

        var statusCode = context.Response.StatusCode;
        var logLevel = statusCode switch
        {
            >= 500 => LogLevel.Error,
            >= 400 => LogLevel.Warning,
            _ => LogLevel.Information
        };

        var responseLog = new StringBuilder();
        responseLog.AppendLine($"┌── RESPONSE [{requestId}] ─────────────────────");
        responseLog.AppendLine($"│ Status: {statusCode} ({GetStatusDescription(statusCode)})");
        responseLog.AppendLine($"│ Duration: {duration.TotalMilliseconds:F2}ms");
        responseLog.AppendLine($"│ Content-Type: {context.Response.ContentType}");

        if (!string.IsNullOrWhiteSpace(body) && body.Length < 1000)
        {
            responseLog.AppendLine($"│ Body: {TruncateBody(body)}");
        }

        if (exception != null)
        {
            responseLog.AppendLine($"│ Exception: {exception.GetType().Name}: {exception.Message}");
        }

        responseLog.AppendLine($"└────────────────────────────────────────────");

        _logger.Log(logLevel, responseLog.ToString());
    }

    private static async Task<string> ReadBodyAsync(Stream body)
    {
        try
        {
            using var reader = new StreamReader(body, Encoding.UTF8, leaveOpen: true);
            return await reader.ReadToEndAsync();
        }
        catch
        {
            return string.Empty;
        }
    }

    private static string TruncateBody(string body, int maxLength = 500)
    {
        if (body.Length <= maxLength)
            return body;
        return body[..maxLength] + "... [truncated]";
    }

    private static string GetStatusDescription(int statusCode) => statusCode switch
    {
        200 => "OK",
        201 => "Created",
        204 => "No Content",
        301 => "Moved Permanently",
        302 => "Found",
        400 => "Bad Request",
        401 => "Unauthorized",
        403 => "Forbidden",
        404 => "Not Found",
        422 => "Unprocessable Entity",
        500 => "Internal Server Error",
        503 => "Service Unavailable",
        _ => "Unknown"
    };
}
```

### ผลลัพธ์ Logging

```
┌── REQUEST [0HN3KJQM4A7AB:00000001] ──────────────────────
│ GET https://localhost:7001/api/test
│ Time: 2024-01-15 10:30:00.123 UTC
│ IP: ::1
│ Header: Accept: */*
│ Header: User-Agent: curl/7.68.0
│ Header: Authorization: [REDACTED]
└────────────────────────────────────────────

┌── RESPONSE [0HN3KJQM4A7AB:00000001] ─────────────────────
│ Status: 200 (OK)
│ Duration: 45.23ms
│ Content-Type: application/json; charset=utf-8
│ Body: {"status":"OK","time":"2024-01-15T10:30:00Z"}
└────────────────────────────────────────────
```

---

## 8. Middleware ขั้นสูง

### 8.1 Conditional Middleware

```csharp
// MapWhen - เพิ่ม middleware เฉพาะเมื่อ condition เป็น true
app.MapWhen(
    context => context.Request.Path.StartsWithSegments("/api"),
    apiApp =>
    {
        apiApp.UseMiddleware<ApiKeyMiddleware>();
        apiApp.UseMiddleware<RateLimitingMiddleware>();
    });

// UseWhen - เพิ่ม middleware แบบ conditional แต่ยังคง pipeline ต่อ
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/admin"),
    adminApp =>
    {
        adminApp.UseAuthentication();
        adminApp.UseAuthorization();
    });
```

### 8.2 Branch Middleware Pipeline

```csharp
// Map - แยก pipeline สำหรับ path prefix
app.Map("/api", apiApp =>
{
    apiApp.UseMiddleware<ApiLoggingMiddleware>();
    apiApp.Run(async context =>
    {
        await context.Response.WriteAsync("API branch");
    });
});

app.Map("/admin", adminApp =>
{
    adminApp.UseAuthentication();
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin branch");
    });
});
```

### 8.3 Terminal Middleware

```csharp
// Run - terminal middleware ไม่เรียก next
app.Run(async context =>
{
    await context.Response.WriteAsync("This is the final middleware");
    // Pipeline stops here - ไม่มี middleware ถัดไปทำงาน
});
```

---

## Exercises

### Exercise 1: IP Blacklist Middleware
สร้าง middleware ที่ block IP addresses ที่ระบุไว้ใน blacklist:

```csharp
// ต้องการ:
// 1. อ่าน blacklisted IPs จาก configuration (appsettings.json)
// 2. เช็ค IP ของแต่ละ request
// 3. ถ้า IP อยู่ใน blacklist ให้ return 403 Forbidden
// 4. Log การ block พร้อม IP และ timestamp

public class IpBlacklistMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<IpBlacklistMiddleware> _logger;
    private readonly HashSet<string> _blacklistedIps;

    public IpBlacklistMiddleware(
        RequestDelegate next, 
        ILogger<IpBlacklistMiddleware> logger,
        IConfiguration configuration)
    {
        _next = next;
        _logger = logger;
        _blacklistedIps = configuration
            .GetSection("Security:BlacklistedIPs")
            .Get<string[]>()
            ?.ToHashSet() ?? [];
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // TODO: implement
        await _next(context);
    }
}
```

### Exercise 2: Response Time Header Middleware
สร้าง middleware ที่เพิ่ม `X-Response-Time` header:

```csharp
// ต้องการ:
// 1. บันทึก start time
// 2. หลัง response ให้เพิ่ม header X-Response-Time: {ms}ms
// 3. เพิ่ม header X-Request-Id: {guid}
// 4. Log slow requests (> 1000ms) ด้วย LogLevel.Warning

public class ResponseTimingMiddleware
{
    // TODO: implement
}
```

### Exercise 3: Request Validator Middleware
สร้าง middleware ที่ validate Content-Type สำหรับ POST/PUT requests:

```csharp
// ต้องการ:
// 1. สำหรับ POST/PUT/PATCH ตรวจสอบว่ามี Content-Type
// 2. ถ้า Content-Type ไม่ใช่ application/json ให้ return 415 Unsupported Media Type
// 3. ตรวจสอบ Content-Length ไม่เกิน 1MB
// 4. ยกเว้น endpoints ที่มี attribute [SkipContentTypeValidation]
```

---

## สรุป

✅ Middleware คือ component ที่ประมวลผล request/response ใน pipeline  
✅ `app.Use` เพิ่ม middleware ที่เรียก next, `app.Run` เพิ่ม terminal middleware  
✅ `app.UseMiddleware<T>()` ใช้ class-based middleware  
✅ Built-in middleware: UseStaticFiles, UseRouting, UseAuthentication, UseAuthorization  
✅ ลำดับ middleware มีผลมาก: Exception → HTTPS → Static → Routing → CORS → Auth → Endpoints  
✅ Scoped services รับใน InvokeAsync ไม่ใช่ constructor  
✅ ใช้ `IMiddleware` interface เพื่อให้ middleware มี scoped lifetime  
✅ `MapWhen` และ `UseWhen` ใช้เพิ่ม conditional middleware  

---

## Part ถัดไป

ใน **Part 044** เราจะเรียนรู้เรื่อง **Dependency Injection (DI)** อย่างละเอียด:
- IoC container
- Service lifetimes: Singleton, Scoped, Transient
- Constructor injection
- การ resolve services

---

*Part 043/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
