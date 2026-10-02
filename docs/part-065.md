# Part 065: Health Checks

## เนื้อหาใน Part นี้
- AddHealthChecks - การตั้งค่าพื้นฐาน
- Custom Health Checks
- Database Health Check
- External Service Health Check
- Health Check UI
- โปรแกรมตัวอย่าง: Production-ready Health Checks

---

## 1. AddHealthChecks - การตั้งค่าพื้นฐาน

**Health Checks** เป็น mechanism ที่ช่วยให้ระบบ monitoring (เช่น Kubernetes, Load Balancer) ตรวจสอบสถานะของ application ได้

### การตั้งค่าเบื้องต้น

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม Health Checks
builder.Services.AddHealthChecks();

var app = builder.Build();

// Map health check endpoint
app.MapHealthChecks("/health");

// หรือระบุ options
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var response = new
        {
            status = report.Status.ToString(),
            duration = report.TotalDuration.TotalMilliseconds + "ms",
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds + "ms",
                error = e.Value.Exception?.Message
            })
        };
        await context.Response.WriteAsJsonAsync(response);
    }
});

app.Run();
```

### Health Status

```csharp
// HealthStatus enum
HealthStatus.Healthy    // ทำงานปกติ - HTTP 200
HealthStatus.Degraded   // ทำงานได้แต่ประสิทธิภาพลดลง - HTTP 200
HealthStatus.Unhealthy  // ไม่สามารถทำงานได้ - HTTP 503
```

---

## 2. Built-in Health Checks

### Entity Framework Core

```bash
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore
```

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(
        name: "database",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "sql", "critical" });
```

### ตรวจสอบ Disk Space และ Memory

```bash
dotnet add package AspNetCore.HealthChecks.System
```

```csharp
builder.Services.AddHealthChecks()
    .AddDiskStorageHealthCheck(options =>
        options.AddDrive("C:\\", minimumFreeMegabytes: 1024),  // ต้องมีพื้นที่ > 1GB
        name: "disk-storage",
        tags: new[] { "system" })
    .AddProcessAllocatedMemoryHealthCheck(
        maximumMegabytesAllocated: 512,
        name: "memory",
        tags: new[] { "system" });
```

### Redis Health Check

```bash
dotnet add package AspNetCore.HealthChecks.Redis
```

```csharp
builder.Services.AddHealthChecks()
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        tags: new[] { "cache" });
```

### SQL Server Health Check

```bash
dotnet add package AspNetCore.HealthChecks.SqlServer
```

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        healthQuery: "SELECT 1",
        name: "sql-server",
        tags: new[] { "db", "critical" });
```

---

## 3. Custom Health Checks

### Interface IHealthCheck

```csharp
public interface IHealthCheck
{
    Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default);
}
```

### ตัวอย่าง Custom Health Check

```csharp
// HealthChecks/DatabaseHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.EntityFrameworkCore;

namespace HealthCheckDemo.HealthChecks;

public class DatabaseHealthCheck : IHealthCheck
{
    private readonly AppDbContext _db;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(AppDbContext db, ILogger<DatabaseHealthCheck> logger)
    {
        _db = db;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // ทดสอบ connection และ query
            var canConnect = await _db.Database.CanConnectAsync(cancellationToken);

            if (!canConnect)
            {
                return HealthCheckResult.Unhealthy(
                    description: "ไม่สามารถเชื่อมต่อ database ได้");
            }

            // ทดสอบ query จริง
            var count = await _db.Products.CountAsync(cancellationToken);

            var data = new Dictionary<string, object>
            {
                ["ProductCount"] = count,
                ["DatabaseServer"] = _db.Database.GetDbConnection().DataSource
            };

            return HealthCheckResult.Healthy(
                description: $"Database เชื่อมต่อได้ ({count} products)",
                data: data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");

            return HealthCheckResult.Unhealthy(
                description: "Database health check failed",
                exception: ex);
        }
    }
}
```

### Degraded Status

```csharp
// HealthChecks/PerformanceHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;
using System.Diagnostics;

namespace HealthCheckDemo.HealthChecks;

public class PerformanceHealthCheck : IHealthCheck
{
    private readonly AppDbContext _db;

    public PerformanceHealthCheck(AppDbContext db)
    {
        _db = db;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await _db.Database.ExecuteSqlRawAsync("SELECT 1", cancellationToken);
            stopwatch.Stop();

            var responseTime = stopwatch.ElapsedMilliseconds;
            var data = new Dictionary<string, object>
            {
                ["ResponseTimeMs"] = responseTime
            };

            if (responseTime > 1000)
            {
                return HealthCheckResult.Unhealthy(
                    $"Database response too slow: {responseTime}ms (limit: 1000ms)",
                    data: data);
            }

            if (responseTime > 500)
            {
                return HealthCheckResult.Degraded(
                    $"Database response slow: {responseTime}ms (warning: 500ms)",
                    data: data);
            }

            return HealthCheckResult.Healthy(
                $"Database response normal: {responseTime}ms",
                data: data);
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                "Database performance check failed",
                exception: ex);
        }
    }
}
```

---

## 4. External Service Health Check

```csharp
// HealthChecks/ExternalApiHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace HealthCheckDemo.HealthChecks;

public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;
    private readonly string _apiUrl;
    private readonly ILogger<ExternalApiHealthCheck> _logger;

    public ExternalApiHealthCheck(
        IHttpClientFactory httpClientFactory,
        IConfiguration configuration,
        ILogger<ExternalApiHealthCheck> logger)
    {
        _httpClient = httpClientFactory.CreateClient("HealthCheck");
        _apiUrl = configuration["ExternalApi:HealthEndpoint"]
            ?? throw new InvalidOperationException("ExternalApi:HealthEndpoint not configured");
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            cts.CancelAfter(TimeSpan.FromSeconds(5)); // timeout 5 วินาที

            var response = await _httpClient.GetAsync(_apiUrl, cts.Token);

            var data = new Dictionary<string, object>
            {
                ["StatusCode"] = (int)response.StatusCode,
                ["Url"] = _apiUrl
            };

            if (response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Healthy(
                    $"External API is healthy. Status: {response.StatusCode}",
                    data: data);
            }

            if ((int)response.StatusCode >= 500)
            {
                return HealthCheckResult.Unhealthy(
                    $"External API server error. Status: {response.StatusCode}",
                    data: data);
            }

            return HealthCheckResult.Degraded(
                $"External API returned non-success status: {response.StatusCode}",
                data: data);
        }
        catch (OperationCanceledException)
        {
            return HealthCheckResult.Unhealthy(
                "External API health check timed out",
                data: new Dictionary<string, object> { ["Url"] = _apiUrl });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "External API health check failed for {Url}", _apiUrl);
            return HealthCheckResult.Unhealthy(
                $"External API is unreachable: {ex.Message}",
                exception: ex,
                data: new Dictionary<string, object> { ["Url"] = _apiUrl });
        }
    }
}
```

### Health Check สำหรับ Email Service

```csharp
// HealthChecks/EmailServiceHealthCheck.cs
using MailKit.Net.Smtp;
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace HealthCheckDemo.HealthChecks;

public class EmailServiceHealthCheck : IHealthCheck
{
    private readonly IConfiguration _configuration;

    public EmailServiceHealthCheck(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var client = new SmtpClient();

            await client.ConnectAsync(
                _configuration["Email:SmtpHost"],
                int.Parse(_configuration["Email:SmtpPort"] ?? "587"),
                false,
                cancellationToken);

            var data = new Dictionary<string, object>
            {
                ["Host"] = _configuration["Email:SmtpHost"] ?? "unknown",
                ["IsConnected"] = client.IsConnected
            };

            await client.DisconnectAsync(true, cancellationToken);

            return HealthCheckResult.Healthy("Email server is reachable", data: data);
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                $"Email server is unreachable: {ex.Message}",
                exception: ex);
        }
    }
}
```

---

## 5. Health Check Tags และ Filtering

```csharp
// Program.cs - ตั้งค่า health checks พร้อม tags
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>(
        name: "database",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "critical" })
    .AddCheck<ExternalApiHealthCheck>(
        name: "payment-api",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "external", "payment" })
    .AddCheck<EmailServiceHealthCheck>(
        name: "email",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "external", "email" })
    .AddCheck<PerformanceHealthCheck>(
        name: "db-performance",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "performance" });

// Map endpoints แยกตาม tag

// Liveness probe - ตรวจว่า app ยังทำงานอยู่
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // ไม่ต้องตรวจอะไร แค่ return healthy ถ้า app ยัง run
});

// Readiness probe - ตรวจว่าพร้อมรับ traffic
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("critical"),
    ResultStatusCodes =
    {
        [HealthStatus.Healthy] = StatusCodes.Status200OK,
        [HealthStatus.Degraded] = StatusCodes.Status200OK,
        [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
    }
});

// Full health check - ตรวจทุกอย่าง
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = WriteHealthCheckResponse
});
```

---

## 6. Health Check UI

```bash
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.Client
dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage
```

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags: new[] { "critical" })
    .AddCheck<ExternalApiHealthCheck>("external-api");

// Health Check UI
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(30);  // ตรวจทุก 30 วินาที
    options.MaximumHistoryEntriesPerEndpoint(50);
    options.AddHealthCheckEndpoint("API", "/health");  // endpoint ที่จะ monitor
})
.AddInMemoryStorage();

// ...

var app = builder.Build();

// Health Check endpoint พร้อม detailed response
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Health Check UI
app.MapHealthChecksUI(options =>
{
    options.UIPath = "/healthui";
    options.ApiPath = "/healthui-api";
});
```

---

## โปรแกรมตัวอย่าง: Production-ready Health Checks

ระบบ health checks ที่ครบครันสำหรับ production

### โครงสร้าง

```
ProductionHealthChecks/
├── HealthChecks/
│   ├── DatabaseHealthCheck.cs
│   ├── RedisHealthCheck.cs
│   ├── ExternalApiHealthCheck.cs
│   ├── DiskSpaceHealthCheck.cs
│   └── SystemResourcesHealthCheck.cs
├── Extensions/
│   └── HealthCheckExtensions.cs
└── Program.cs
```

### System Resources Health Check

```csharp
// HealthChecks/SystemResourcesHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace ProductionHealthChecks.HealthChecks;

public class SystemResourcesHealthCheck : IHealthCheck
{
    private const long WarningMemoryMb = 500;
    private const long CriticalMemoryMb = 900;
    private const double WarningCpuPercent = 80.0;
    private const double CriticalCpuPercent = 95.0;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var process = System.Diagnostics.Process.GetCurrentProcess();

        // Memory usage
        var memoryMb = process.WorkingSet64 / (1024 * 1024);

        // GC info
        var gcInfo = GC.GetGCMemoryInfo();
        var gcMemoryMb = gcInfo.HeapSizeBytes / (1024 * 1024);

        var data = new Dictionary<string, object>
        {
            ["ProcessMemoryMB"] = memoryMb,
            ["GCHeapMemoryMB"] = gcMemoryMb,
            ["Gen0Collections"] = GC.CollectionCount(0),
            ["Gen1Collections"] = GC.CollectionCount(1),
            ["Gen2Collections"] = GC.CollectionCount(2),
            ["ThreadCount"] = process.Threads.Count,
            ["HandleCount"] = process.HandleCount
        };

        if (memoryMb > CriticalMemoryMb)
        {
            return HealthCheckResult.Unhealthy(
                $"Memory usage critical: {memoryMb}MB (limit: {CriticalMemoryMb}MB)",
                data: data);
        }

        if (memoryMb > WarningMemoryMb)
        {
            return HealthCheckResult.Degraded(
                $"Memory usage high: {memoryMb}MB (warning: {WarningMemoryMb}MB)",
                data: data);
        }

        return HealthCheckResult.Healthy(
            $"System resources normal. Memory: {memoryMb}MB",
            data: data);
    }
}
```

### Disk Space Health Check

```csharp
// HealthChecks/DiskSpaceHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace ProductionHealthChecks.HealthChecks;

public class DiskSpaceHealthCheck : IHealthCheck
{
    private readonly string _drivePath;
    private const long WarningFreeSpaceMb = 1024;   // 1 GB
    private const long CriticalFreeSpaceMb = 512;   // 512 MB

    public DiskSpaceHealthCheck(IConfiguration configuration)
    {
        _drivePath = configuration["HealthChecks:DiskPath"]
            ?? (OperatingSystem.IsWindows() ? "C:\\" : "/");
    }

    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var driveInfo = new DriveInfo(_drivePath);
            var freeMb = driveInfo.AvailableFreeSpace / (1024 * 1024);
            var totalMb = driveInfo.TotalSize / (1024 * 1024);
            var usedPercent = (double)(totalMb - freeMb) / totalMb * 100;

            var data = new Dictionary<string, object>
            {
                ["Drive"] = _drivePath,
                ["FreeSpaceMB"] = freeMb,
                ["TotalSpaceMB"] = totalMb,
                ["UsedPercent"] = Math.Round(usedPercent, 2)
            };

            if (freeMb < CriticalFreeSpaceMb)
            {
                return Task.FromResult(HealthCheckResult.Unhealthy(
                    $"Disk space critical: {freeMb}MB free (minimum: {CriticalFreeSpaceMb}MB)",
                    data: data));
            }

            if (freeMb < WarningFreeSpaceMb)
            {
                return Task.FromResult(HealthCheckResult.Degraded(
                    $"Disk space low: {freeMb}MB free (warning: {WarningFreeSpaceMb}MB)",
                    data: data));
            }

            return Task.FromResult(HealthCheckResult.Healthy(
                $"Disk space OK: {freeMb}MB free ({Math.Round(100 - usedPercent, 1)}%)",
                data: data));
        }
        catch (Exception ex)
        {
            return Task.FromResult(HealthCheckResult.Unhealthy(
                $"Disk space check failed: {ex.Message}",
                exception: ex));
        }
    }
}
```

### Health Check Extensions

```csharp
// Extensions/HealthCheckExtensions.cs
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using System.Text.Json;

namespace ProductionHealthChecks.Extensions;

public static class HealthCheckExtensions
{
    public static IServiceCollection AddProductionHealthChecks(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddHealthChecks()
            // Critical checks
            .AddCheck<DatabaseHealthCheck>(
                "database",
                failureStatus: HealthStatus.Unhealthy,
                tags: new[] { "critical", "db" })
            .AddCheck<RedisHealthCheck>(
                "redis",
                failureStatus: HealthStatus.Unhealthy,
                tags: new[] { "critical", "cache" })

            // System checks
            .AddCheck<DiskSpaceHealthCheck>(
                "disk-space",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "system" })
            .AddCheck<SystemResourcesHealthCheck>(
                "system-resources",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "system" })

            // External services
            .AddCheck<ExternalApiHealthCheck>(
                "payment-api",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "external" })
            .AddCheck<EmailServiceHealthCheck>(
                "email-service",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "external" });

        return services;
    }

    public static WebApplication UseProductionHealthChecks(this WebApplication app)
    {
        // Liveness probe (Kubernetes)
        app.MapHealthChecks("/health/live", new HealthCheckOptions
        {
            Predicate = _ => false,
            ResponseWriter = WriteSimpleResponse
        });

        // Readiness probe (Kubernetes) - ตรวจแค่ critical
        app.MapHealthChecks("/health/ready", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("critical"),
            ResponseWriter = WriteSimpleResponse,
            ResultStatusCodes =
            {
                [HealthStatus.Healthy] = StatusCodes.Status200OK,
                [HealthStatus.Degraded] = StatusCodes.Status200OK,
                [HealthStatus.Unhealthy] = StatusCodes.Status503ServiceUnavailable
            }
        });

        // Detailed health endpoint
        app.MapHealthChecks("/health", new HealthCheckOptions
        {
            ResponseWriter = WriteDetailedResponse
        });

        return app;
    }

    private static async Task WriteSimpleResponse(
        HttpContext context,
        HealthReport report)
    {
        context.Response.ContentType = "application/json";
        var response = new
        {
            status = report.Status.ToString(),
            duration = $"{report.TotalDuration.TotalMilliseconds:F2}ms"
        };
        await context.Response.WriteAsJsonAsync(response);
    }

    private static async Task WriteDetailedResponse(
        HttpContext context,
        HealthReport report)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = report.Status == HealthStatus.Healthy
            ? 200
            : report.Status == HealthStatus.Degraded ? 200 : 503;

        var response = new
        {
            status = report.Status.ToString(),
            totalDuration = $"{report.TotalDuration.TotalMilliseconds:F2}ms",
            timestamp = DateTime.UtcNow,
            checks = report.Entries.Select(entry => new
            {
                name = entry.Key,
                status = entry.Value.Status.ToString(),
                description = entry.Value.Description,
                duration = $"{entry.Value.Duration.TotalMilliseconds:F2}ms",
                data = entry.Value.Data,
                error = entry.Value.Exception?.Message,
                tags = entry.Value.Tags
            })
        };

        await context.Response.WriteAsJsonAsync(response,
            new JsonSerializerOptions { WriteIndented = true });
    }
}
```

### Program.cs สมบูรณ์

```csharp
// Program.cs
using ProductionHealthChecks.Extensions;
using ProductionHealthChecks.HealthChecks;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

builder.Services.AddHttpClient("HealthCheck", client =>
{
    client.Timeout = TimeSpan.FromSeconds(10);
});

// เพิ่ม Health Checks
builder.Services.AddProductionHealthChecks(builder.Configuration);

// Health Check UI
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(30);
    options.MaximumHistoryEntriesPerEndpoint(100);
    options.SetMinimumSecondsBetweenFailureNotifications(60);
    options.AddHealthCheckEndpoint("Application", "/health");
})
.AddInMemoryStorage();

builder.Services.AddControllers();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

// Setup health check endpoints
app.UseProductionHealthChecks();

// Health Check UI
app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";
    options.ApiPath = "/health-ui-api";
});

app.Run();
```

### Redis Health Check Implementation

```csharp
// HealthChecks/RedisHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;
using StackExchange.Redis;

namespace ProductionHealthChecks.HealthChecks;

public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _redis;

    public RedisHealthCheck(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var db = _redis.GetDatabase();
            var testKey = $"healthcheck:{Guid.NewGuid()}";

            // ทดสอบ write
            await db.StringSetAsync(testKey, "ok", TimeSpan.FromSeconds(5));

            // ทดสอบ read
            var value = await db.StringGetAsync(testKey);

            // ลบ test key
            await db.KeyDeleteAsync(testKey);

            if (value != "ok")
            {
                return HealthCheckResult.Unhealthy("Redis read/write test failed");
            }

            // ดึง server info
            var server = _redis.GetServer(_redis.GetEndPoints().First());
            var info = await server.InfoAsync("server");

            var data = new Dictionary<string, object>
            {
                ["RedisVersion"] = info.FirstOrDefault(g => g.Key == "server")
                    ?.FirstOrDefault(e => e.Key == "redis_version")
                    .Value ?? "unknown",
                ["ConnectedClients"] = _redis.GetCounters().TotalOutstanding
            };

            return HealthCheckResult.Healthy("Redis is healthy", data: data);
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                $"Redis is unhealthy: {ex.Message}",
                exception: ex);
        }
    }
}
```

### Health Check Notification

```csharp
// Services/HealthCheckNotificationService.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;

namespace ProductionHealthChecks.Services;

// Service ที่คอยฟัง health check failures และส่ง alert
public class HealthCheckNotificationService : BackgroundService
{
    private readonly IHealthCheckService _healthCheckService;
    private readonly ILogger<HealthCheckNotificationService> _logger;
    private readonly Dictionary<string, HealthStatus> _previousStatuses = new();

    public HealthCheckNotificationService(
        IHealthCheckService healthCheckService,
        ILogger<HealthCheckNotificationService> logger)
    {
        _healthCheckService = healthCheckService;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var report = await _healthCheckService.CheckHealthAsync(stoppingToken);

            foreach (var entry in report.Entries)
            {
                var name = entry.Key;
                var currentStatus = entry.Value.Status;

                if (_previousStatuses.TryGetValue(name, out var previousStatus))
                {
                    // สถานะเปลี่ยน
                    if (currentStatus != previousStatus)
                    {
                        _logger.LogWarning(
                            "Health check {Name} status changed: {Previous} -> {Current}",
                            name, previousStatus, currentStatus);

                        if (currentStatus == HealthStatus.Unhealthy)
                        {
                            await SendAlertAsync(name, entry.Value, stoppingToken);
                        }
                    }
                }

                _previousStatuses[name] = currentStatus;
            }

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task SendAlertAsync(
        string checkName,
        HealthReportEntry entry,
        CancellationToken cancellationToken)
    {
        // ส่ง email, SMS, หรือ Slack notification
        _logger.LogCritical(
            "ALERT: Health check {CheckName} is UNHEALTHY. Description: {Description}",
            checkName,
            entry.Description);

        // TODO: ส่ง notification จริงๆ
        await Task.CompletedTask;
    }
}
```

---

## Exercises

### Exercise 1: Certificate Expiry Check
สร้าง health check ที่ตรวจสอบ SSL certificate expiry date และแจ้งเตือนล่วงหน้า 30 วัน

### Exercise 2: Message Queue Health Check
สร้าง health check สำหรับ RabbitMQ หรือ Azure Service Bus

### Exercise 3: Third-party Dependencies
สร้าง health check ที่ตรวจสอบ dependency หลายอย่างพร้อมกัน (parallel health checks)

### Exercise 4: Custom Health Check UI
สร้าง custom dashboard แสดงสถานะ health checks แบบ real-time ด้วย SignalR

### Exercise 5: Health Check History
บันทึก history ของ health check results ลง database และสร้าง endpoint แสดง trend

---

## สรุป

- **Health Checks** เป็น essential สำหรับ production deployment
- ใช้ **tags** เพื่อจัดกลุ่มและ filter checks ตามประเภท
- แยก **liveness** (ยัง run อยู่) และ **readiness** (พร้อมรับ traffic)
- **Health Check UI** ช่วย visualize สถานะของ system
- ควร set **timeout** ที่เหมาะสมใน health checks
- Log สถานะเมื่อมีการเปลี่ยนแปลง และส่ง alert เมื่อ unhealthy

---

## Part ถัดไป

**Part 066: API Documentation กับ Swagger** - เรียนรู้การสร้าง API documentation ด้วย Swashbuckle และ OpenAPI

---

*Part 065/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
