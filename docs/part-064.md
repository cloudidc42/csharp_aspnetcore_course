# Part 064: Logging ใน ASP.NET Core

## เนื้อหาใน Part นี้
- ILogger<T> - Built-in Logging
- Log Levels และการใช้งาน
- Structured Logging ด้วย Serilog
- Log Sinks: Console, File, Seq
- Correlation ID สำหรับ request tracing
- โปรแกรมตัวอย่าง: Application Logging System

---

## 1. ILogger<T> - Built-in Logging

ASP.NET Core มี logging framework ในตัวที่ใช้ผ่าน `ILogger<T>`

### การใช้งานเบื้องต้น

```csharp
// Services/OrderService.cs
namespace LoggingDemo.Services;

public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Log levels ต่างๆ
        _logger.LogTrace("Entering CreateOrderAsync with {Request}", request);
        _logger.LogDebug("Processing order for customer {CustomerId}", request.CustomerId);
        _logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);

        try
        {
            var order = new Order
            {
                CustomerId = request.CustomerId,
                Items = request.Items,
                Total = request.Items.Sum(i => i.Price * i.Quantity),
                CreatedAt = DateTime.UtcNow
            };

            _logger.LogInformation(
                "Order created successfully. OrderId: {OrderId}, Total: {Total:C}",
                order.Id,
                order.Total);

            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to create order for customer {CustomerId}",
                request.CustomerId);
            throw;
        }
    }
}
```

### Log Message Templates (Structured Logging)

```csharp
// ดี - ใช้ named placeholders
_logger.LogInformation("User {UserId} logged in from {IpAddress}", userId, ipAddress);

// ไม่ดี - string interpolation (ไม่ใช่ structured logging)
_logger.LogInformation($"User {userId} logged in from {ipAddress}");

// ดี - เก็บ complex objects
_logger.LogInformation("Order processed: {@Order}", order);  // @ = destructure object

// สร้าง log scope
using (_logger.BeginScope("Processing order {OrderId}", orderId))
{
    _logger.LogInformation("Step 1: Validate");
    _logger.LogInformation("Step 2: Process payment");
    _logger.LogInformation("Step 3: Ship");
}
```

---

## 2. Log Levels

```csharp
// Log Levels เรียงตาม severity (น้อยไปมาก)
_logger.LogTrace("ข้อมูลละเอียดมาก สำหรับ debugging เท่านั้น");      // Level 0
_logger.LogDebug("ข้อมูล debug เพื่อ development");                     // Level 1
_logger.LogInformation("ข้อมูล general เกี่ยวกับ flow ของ app");        // Level 2
_logger.LogWarning("บางอย่างผิดปกติแต่ยังทำงานได้");                   // Level 3
_logger.LogError(exception, "เกิด error แต่ app ยังทำงานได้");          // Level 4
_logger.LogCritical("เกิด failure ที่ร้ายแรง app อาจต้อง shutdown");   // Level 5

// ใช้ EventId สำหรับกรอง log
var orderId = new EventId(1001, "OrderCreated");
_logger.Log(LogLevel.Information, orderId, "Order {Id} created", order.Id);
```

### การตั้งค่า Log Levels ใน appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information",
      "LoggingDemo": "Debug",
      "LoggingDemo.Services.OrderService": "Trace"
    }
  }
}
```

### Log Level ตาม Environment

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information"
    }
  }
}

// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "LoggingDemo": "Information"
    }
  }
}
```

---

## 3. Structured Logging ด้วย Serilog

**Serilog** เป็น logging library ที่ทรงพลัง รองรับ structured logging และ sinks หลายแบบ

### การติดตั้ง

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
dotnet add package Serilog.Enrichers.Process
dotnet add package Serilog.Enrichers.CorrelationId
```

### การตั้งค่า Serilog

```csharp
// Program.cs
using Serilog;
using Serilog.Events;

// ตั้งค่า Serilog ก่อนสร้าง builder
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithThreadId()
    .Enrich.WithProcessId()
    .Enrich.WithMachineName()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/app-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30,
        fileSizeLimitBytes: 100_000_000,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);

    // ใช้ Serilog แทน built-in logging
    builder.Host.UseSerilog();

    // ... services
    var app = builder.Build();

    // Log HTTP requests
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("RequestScheme", httpContext.Request.Scheme);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers.UserAgent.ToString());
        };
    });

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### Serilog ผ่าน Configuration File

```csharp
// Program.cs - แบบ configuration
builder.Host.UseSerilog((context, services, configuration) =>
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext());
```

```json
// appsettings.json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.EntityFrameworkCore": "Information"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "theme": "Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme::Code, Serilog.Sinks.Console",
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"]
  }
}
```

---

## 4. Log Sinks

### Console Sink

```csharp
.WriteTo.Console(
    restrictedToMinimumLevel: LogEventLevel.Information,
    theme: Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme.Code,
    outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}{NewLine}  {Message:lj}{NewLine}{Exception}")
```

### File Sink

```csharp
.WriteTo.File(
    path: "logs/app-.log",
    rollingInterval: RollingInterval.Day,
    retainedFileCountLimit: 30,
    fileSizeLimitBytes: 100_000_000,  // 100MB
    rollOnFileSizeLimit: true,
    shared: false,
    flushToDiskInterval: TimeSpan.FromSeconds(1))

// JSON format
.WriteTo.File(
    new JsonFormatter(),
    path: "logs/app-.json",
    rollingInterval: RollingInterval.Day)
```

### Seq Sink (Centralized Logging)

```bash
# รัน Seq ด้วย Docker
docker run --name seq -e ACCEPT_EULA=Y -p 5341:80 datalust/seq:latest
```

```csharp
.WriteTo.Seq(
    serverUrl: "http://localhost:5341",
    apiKey: "your-api-key",
    restrictedToMinimumLevel: LogEventLevel.Debug,
    batchPostingLimit: 50,
    period: TimeSpan.FromSeconds(2))
```

---

## 5. Correlation ID

**Correlation ID** ใช้สำหรับติดตาม request ที่ผ่านหลาย services

### Middleware สำหรับ Correlation ID

```csharp
// Middleware/CorrelationIdMiddleware.cs
namespace LoggingDemo.Middleware;

public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-Id";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // ดึง Correlation ID จาก header หรือสร้างใหม่
        var correlationId = context.Request.Headers[CorrelationIdHeader].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        // เพิ่มกลับใน response header
        context.Response.Headers.TryAdd(CorrelationIdHeader, correlationId);

        // เพิ่มใน HttpContext
        context.Items["CorrelationId"] = correlationId;

        // เพิ่มใน Serilog LogContext
        using (Serilog.Context.LogContext.PushProperty("CorrelationId", correlationId))
        {
            await _next(context);
        }
    }
}

// Extension method
public static class CorrelationIdMiddlewareExtensions
{
    public static IApplicationBuilder UseCorrelationId(this IApplicationBuilder app)
    {
        return app.UseMiddleware<CorrelationIdMiddleware>();
    }
}
```

### ใช้งานใน Program.cs

```csharp
// Program.cs
app.UseCorrelationId(); // ต้องอยู่ก่อน UseSerilogRequestLogging
app.UseSerilogRequestLogging();
```

### HttpClient Factory พร้อม Correlation ID

```csharp
// Services/ApiClient.cs
using System.Net.Http;

namespace LoggingDemo.Services;

public class ApiClient
{
    private readonly HttpClient _httpClient;
    private readonly IHttpContextAccessor _httpContextAccessor;
    private const string CorrelationIdHeader = "X-Correlation-Id";

    public ApiClient(HttpClient httpClient, IHttpContextAccessor httpContextAccessor)
    {
        _httpClient = httpClient;
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task<T?> GetAsync<T>(string url)
    {
        // ส่ง correlation ID ไปยัง downstream service
        var correlationId = _httpContextAccessor.HttpContext?.Items["CorrelationId"]?.ToString()
            ?? Guid.NewGuid().ToString();

        _httpClient.DefaultRequestHeaders.TryAddWithoutValidation(
            CorrelationIdHeader, correlationId);

        var response = await _httpClient.GetAsync(url);
        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<T>();
    }
}
```

---

## 6. Log Enrichers

```csharp
// Custom Log Enricher
using Serilog.Core;
using Serilog.Events;

public class RequestEnricher : ILogEventEnricher
{
    private readonly IHttpContextAccessor _contextAccessor;

    public RequestEnricher(IHttpContextAccessor contextAccessor)
    {
        _contextAccessor = contextAccessor;
    }

    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        var context = _contextAccessor.HttpContext;
        if (context == null) return;

        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("UserId",
                context.User.FindFirst("sub")?.Value ?? "anonymous"));

        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("RequestPath", context.Request.Path));

        logEvent.AddPropertyIfAbsent(
            propertyFactory.CreateProperty("RemoteIP",
                context.Connection.RemoteIpAddress?.ToString() ?? "unknown"));
    }
}
```

---

## โปรแกรมตัวอย่าง: Application Logging System

ระบบ logging ที่สมบูรณ์สำหรับ production application

### โครงสร้าง

```
AppLogging/
├── Middleware/
│   ├── CorrelationIdMiddleware.cs
│   └── RequestLoggingMiddleware.cs
├── Services/
│   ├── IAuditLogger.cs
│   └── AuditLogger.cs
├── Models/
│   └── AuditLog.cs
├── Filters/
│   └── LoggingActionFilter.cs
└── Program.cs
```

### Custom Request Logging Middleware

```csharp
// Middleware/RequestLoggingMiddleware.cs
using System.Diagnostics;

namespace AppLogging.Middleware;

public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var requestId = context.TraceIdentifier;

        _logger.LogInformation(
            "Request START {Method} {Path} {QueryString} - RequestId: {RequestId}",
            context.Request.Method,
            context.Request.Path,
            context.Request.QueryString,
            requestId);

        try
        {
            await _next(context);

            stopwatch.Stop();

            var level = context.Response.StatusCode >= 400
                ? LogLevel.Warning
                : LogLevel.Information;

            _logger.Log(level,
                "Request END {Method} {Path} - Status: {StatusCode}, Duration: {Elapsed}ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex,
                "Request FAILED {Method} {Path} - Duration: {Elapsed}ms",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}
```

### Audit Logger

```csharp
// Models/AuditLog.cs
namespace AppLogging.Models;

public class AuditLog
{
    public int Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public string UserName { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;
    public string EntityType { get; set; } = string.Empty;
    public string? EntityId { get; set; }
    public string? OldValue { get; set; }
    public string? NewValue { get; set; }
    public string IpAddress { get; set; } = string.Empty;
    public string? CorrelationId { get; set; }
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public bool Success { get; set; }
    public string? ErrorMessage { get; set; }
}

// Services/IAuditLogger.cs
namespace AppLogging.Services;

public interface IAuditLogger
{
    Task LogAsync(AuditLogEntry entry);
    Task<List<AuditLog>> GetLogsAsync(AuditLogFilter filter);
}

public class AuditLogEntry
{
    public string Action { get; set; } = string.Empty;
    public string EntityType { get; set; } = string.Empty;
    public string? EntityId { get; set; }
    public object? OldValue { get; set; }
    public object? NewValue { get; set; }
    public bool Success { get; set; } = true;
    public string? ErrorMessage { get; set; }
}

public class AuditLogFilter
{
    public string? UserId { get; set; }
    public string? EntityType { get; set; }
    public string? Action { get; set; }
    public DateTime? From { get; set; }
    public DateTime? To { get; set; }
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 20;
}

// Services/AuditLogger.cs
using System.Text.Json;
using Microsoft.EntityFrameworkCore;
using AppLogging.Data;
using AppLogging.Models;

namespace AppLogging.Services;

public class AuditLogger : IAuditLogger
{
    private readonly AppDbContext _db;
    private readonly IHttpContextAccessor _contextAccessor;
    private readonly ILogger<AuditLogger> _logger;

    public AuditLogger(
        AppDbContext db,
        IHttpContextAccessor contextAccessor,
        ILogger<AuditLogger> logger)
    {
        _db = db;
        _contextAccessor = contextAccessor;
        _logger = logger;
    }

    public async Task LogAsync(AuditLogEntry entry)
    {
        var context = _contextAccessor.HttpContext;

        var auditLog = new AuditLog
        {
            UserId = context?.User?.FindFirst("sub")?.Value ?? "system",
            UserName = context?.User?.Identity?.Name ?? "system",
            Action = entry.Action,
            EntityType = entry.EntityType,
            EntityId = entry.EntityId?.ToString(),
            OldValue = entry.OldValue != null
                ? JsonSerializer.Serialize(entry.OldValue) : null,
            NewValue = entry.NewValue != null
                ? JsonSerializer.Serialize(entry.NewValue) : null,
            IpAddress = context?.Connection?.RemoteIpAddress?.ToString() ?? "unknown",
            CorrelationId = context?.Items["CorrelationId"]?.ToString(),
            Success = entry.Success,
            ErrorMessage = entry.ErrorMessage,
            Timestamp = DateTime.UtcNow
        };

        _db.AuditLogs.Add(auditLog);

        try
        {
            await _db.SaveChangesAsync();
        }
        catch (Exception ex)
        {
            // ถ้าบันทึก audit log ไม่สำเร็จ ให้ log แทน (ไม่ throw exception)
            _logger.LogError(ex, "Failed to save audit log for {Action} on {EntityType}",
                entry.Action, entry.EntityType);
        }
    }

    public async Task<List<AuditLog>> GetLogsAsync(AuditLogFilter filter)
    {
        var query = _db.AuditLogs.AsQueryable();

        if (!string.IsNullOrEmpty(filter.UserId))
            query = query.Where(l => l.UserId == filter.UserId);

        if (!string.IsNullOrEmpty(filter.EntityType))
            query = query.Where(l => l.EntityType == filter.EntityType);

        if (!string.IsNullOrEmpty(filter.Action))
            query = query.Where(l => l.Action == filter.Action);

        if (filter.From.HasValue)
            query = query.Where(l => l.Timestamp >= filter.From.Value);

        if (filter.To.HasValue)
            query = query.Where(l => l.Timestamp <= filter.To.Value);

        return await query
            .OrderByDescending(l => l.Timestamp)
            .Skip((filter.Page - 1) * filter.PageSize)
            .Take(filter.PageSize)
            .ToListAsync();
    }
}
```

### Logging Action Filter

```csharp
// Filters/LoggingActionFilter.cs
using Microsoft.AspNetCore.Mvc.Filters;

namespace AppLogging.Filters;

public class LoggingActionFilter : IAsyncActionFilter
{
    private readonly ILogger<LoggingActionFilter> _logger;

    public LoggingActionFilter(ILogger<LoggingActionFilter> logger)
    {
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var actionName = context.ActionDescriptor.DisplayName;

        _logger.LogDebug("Executing action: {ActionName}", actionName);

        // Log parameters (ระวัง sensitive data)
        foreach (var (key, value) in context.ActionArguments)
        {
            _logger.LogDebug("Parameter {Key}: {@Value}", key, value);
        }

        var result = await next();

        if (result.Exception != null)
        {
            _logger.LogError(result.Exception,
                "Action {ActionName} threw an exception", actionName);
        }
        else
        {
            _logger.LogDebug("Action {ActionName} completed successfully", actionName);
        }
    }
}
```

### Exception Handler Middleware

```csharp
// Middleware/GlobalExceptionHandler.cs
using System.Net;
using System.Text.Json;

namespace AppLogging.Middleware;

public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var correlationId = httpContext.Items["CorrelationId"]?.ToString() ?? "unknown";

        _logger.LogError(exception,
            "Unhandled exception. CorrelationId: {CorrelationId}, Path: {Path}",
            correlationId,
            httpContext.Request.Path);

        var response = new
        {
            Status = (int)HttpStatusCode.InternalServerError,
            Title = "An error occurred",
            CorrelationId = correlationId
        };

        httpContext.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
        httpContext.Response.ContentType = "application/json";

        await httpContext.Response.WriteAsync(
            JsonSerializer.Serialize(response),
            cancellationToken);

        return true;
    }
}
```

### Program.cs สมบูรณ์

```csharp
// Program.cs
using Serilog;
using Serilog.Events;
using AppLogging.Middleware;
using AppLogging.Services;
using AppLogging.Filters;
using Microsoft.EntityFrameworkCore;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore.Database.Command", LogEventLevel.Information)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] [{CorrelationId}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/app-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 7,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] [{CorrelationId}] {SourceContext}{NewLine}  {Message:lj}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Host.UseSerilog();

    builder.Services.AddDbContext<AppDbContext>(options =>
        options.UseSqlite("Data Source=app.db"));

    builder.Services.AddHttpContextAccessor();
    builder.Services.AddScoped<IAuditLogger, AuditLogger>();

    builder.Services.AddControllers(options =>
    {
        options.Filters.Add<LoggingActionFilter>();
    });

    builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
    builder.Services.AddProblemDetails();

    var app = builder.Build();

    app.UseCorrelationId();
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate =
            "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms - [{CorrelationId}]";
        options.GetLevel = (ctx, elapsed, ex) =>
            ex != null || ctx.Response.StatusCode > 499
                ? LogEventLevel.Error
                : elapsed > 1000
                    ? LogEventLevel.Warning
                    : LogEventLevel.Information;
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("CorrelationId",
                httpContext.Items["CorrelationId"]?.ToString() ?? "unknown");
            diagnosticContext.Set("UserId",
                httpContext.User?.FindFirst("sub")?.Value ?? "anonymous");
        };
    });

    app.UseExceptionHandler();
    app.UseHttpsRedirection();
    app.UseAuthorization();
    app.MapControllers();
    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application start-up failed");
}
finally
{
    Log.CloseAndFlush();
}
```

### Controller ตัวอย่าง

```csharp
// Controllers/OrdersController.cs
using Microsoft.AspNetCore.Mvc;
using AppLogging.Services;

namespace AppLogging.Controllers;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly ILogger<OrdersController> _logger;
    private readonly IAuditLogger _auditLogger;

    public OrdersController(ILogger<OrdersController> logger, IAuditLogger auditLogger)
    {
        _logger = logger;
        _auditLogger = auditLogger;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["OrderCustomerId"] = request.CustomerId,
            ["OrderAmount"] = request.Items.Sum(i => i.Price * i.Quantity)
        });

        _logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);

        try
        {
            var order = new Order { Id = Guid.NewGuid(), CustomerId = request.CustomerId };

            await _auditLogger.LogAsync(new AuditLogEntry
            {
                Action = "CREATE",
                EntityType = "Order",
                EntityId = order.Id.ToString(),
                NewValue = order,
                Success = true
            });

            _logger.LogInformation(
                "Order {OrderId} created successfully for customer {CustomerId}",
                order.Id,
                request.CustomerId);

            return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create order for customer {CustomerId}",
                request.CustomerId);

            await _auditLogger.LogAsync(new AuditLogEntry
            {
                Action = "CREATE",
                EntityType = "Order",
                Success = false,
                ErrorMessage = ex.Message
            });

            throw;
        }
    }

    [HttpGet("{id}")]
    public IActionResult GetOrder(Guid id)
    {
        _logger.LogDebug("Getting order {OrderId}", id);
        return Ok(new { Id = id });
    }
}
```

---

## Exercises

### Exercise 1: Performance Logging
สร้าง attribute `[LogPerformance]` ที่ log เวลาที่ใช้ในแต่ละ action method

### Exercise 2: Sensitive Data Masking
สร้าง Serilog Enricher ที่ mask ข้อมูล sensitive เช่น email, credit card number ใน log

### Exercise 3: Log Aggregation Dashboard
สร้าง endpoint ที่อ่านข้อมูลจาก Seq API และแสดง log summary

### Exercise 4: Structured Exception Logging
สร้าง custom exception middleware ที่ log structured exception info รวมถึง stack trace และ context

### Exercise 5: Log Sampling
ปรับแต่ง Serilog ให้ sample เพียง 10% ของ debug logs เพื่อลด volume ใน production

---

## สรุป

- **ILogger<T>** เป็น built-in logging ที่ใช้งานง่ายและ extensible
- ใช้ **named placeholders** ใน log messages เพื่อ structured logging
- **Serilog** เป็น library ที่แนะนำสำหรับ production logging
- **Correlation ID** ช่วย trace request ผ่านหลาย services
- เลือก **log level** ให้เหมาะสม อย่า log ข้อมูล sensitive
- ใช้ **log sinks** หลายอย่างร่วมกัน เช่น Console + File + Seq

---

## Part ถัดไป

**Part 065: Health Checks** - เรียนรู้การสร้าง health checks เพื่อ monitor application ใน production

---

*Part 064/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
