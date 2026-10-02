# Part 042: Program.cs และ Startup (Minimal API)

## เนื้อหาใน Part นี้
- Minimal API pattern ใน .NET 6+
- `builder.Services` - Dependency Injection container
- `app.Use*` และ `app.Map*`
- Middleware order
- Environment (Development, Production)
- Logging
- โปรแกรมตัวอย่าง: Hello World API

---

## 1. วิวัฒนาการของ Program.cs

### .NET 5 และก่อนหน้า (แบบเก่า)

ก่อน .NET 6 โปรเจค ASP.NET Core ต้องมีทั้ง `Program.cs` และ `Startup.cs`:

```csharp
// Program.cs (แบบเก่า .NET 5)
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

// Startup.cs (แบบเก่า)
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    // เรียกเมื่อ runtime - ใช้เพิ่ม services
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddEndpointsApiExplorer();
        services.AddSwaggerGen();
    }

    // เรียกเมื่อ runtime - ใช้ configure HTTP pipeline
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseSwagger();
            app.UseSwaggerUI();
        }

        app.UseHttpsRedirection();
        app.UseRouting();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

### .NET 6+ (Minimal API - แบบใหม่)

```csharp
// Program.cs (แบบใหม่ .NET 6+)
// ไม่ต้องการ class Program และ method Main อีกต่อไป!
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure HTTP pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**ข้อดีของ Minimal API:**
- โค้ดน้อยลงมาก
- ไม่ต้องมี `Startup.cs` แยก
- ใช้ Top-level statements
- อ่านง่าย เข้าใจง่าย

---

## 2. WebApplication.CreateBuilder

`WebApplication.CreateBuilder(args)` สร้าง `WebApplicationBuilder` ซึ่งใช้:
- ตั้งค่า services (DI container)
- ตั้งค่า configuration
- ตั้งค่า logging
- ตั้งค่า hosting

```csharp
var builder = WebApplication.CreateBuilder(args);

// builder มี properties เหล่านี้:
// builder.Services      - IServiceCollection สำหรับเพิ่ม services
// builder.Configuration - IConfiguration สำหรับ configuration
// builder.Environment   - IWebHostEnvironment
// builder.Logging       - ILoggingBuilder
// builder.Host          - IHostBuilder
// builder.WebHost       - IWebHostBuilder
```

### ตัวอย่างการใช้ builder properties

```csharp
var builder = WebApplication.CreateBuilder(args);

// ตรวจสอบ environment
Console.WriteLine($"Environment: {builder.Environment.EnvironmentName}");
Console.WriteLine($"App Name: {builder.Environment.ApplicationName}");
Console.WriteLine($"Content Root: {builder.Environment.ContentRootPath}");

// อ่าน configuration
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var port = builder.Configuration.GetValue<int>("Port", defaultValue: 5000);

// ตั้งค่า logging
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddDebug();

// ตั้งค่า Kestrel web server
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenLocalhost(5000);
    options.ListenLocalhost(5001, listenOptions =>
    {
        listenOptions.UseHttps();
    });
});
```

---

## 3. builder.Services - Dependency Injection Container

`builder.Services` เป็น `IServiceCollection` ใช้ลงทะเบียน services ที่จะใช้ใน application:

### 3.1 Services ที่มักใช้บ่อย

```csharp
var builder = WebApplication.CreateBuilder(args);

// MVC / Web API
builder.Services.AddControllers();
builder.Services.AddControllersWithViews();
builder.Services.AddRazorPages();

// Minimal API
builder.Services.AddEndpointsApiExplorer();

// Swagger
builder.Services.AddSwaggerGen();

// CORS
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });

    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("https://myapp.com", "https://localhost:3000")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

// Authentication & Authorization
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* config */ });
builder.Services.AddAuthorization();

// Caching
builder.Services.AddMemoryCache();
builder.Services.AddDistributedMemoryCache();
builder.Services.AddResponseCaching();

// HttpClient
builder.Services.AddHttpClient();
builder.Services.AddHttpClient("WeatherApi", client =>
{
    client.BaseAddress = new Uri("https://api.weather.com");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

// Custom Services
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<ICacheService, CacheService>();
builder.Services.AddTransient<IEmailService, EmailService>();
```

### 3.2 Options Pattern

```csharp
// กำหนด Options class
public class DatabaseOptions
{
    public const string SectionName = "Database";
    
    public string ConnectionString { get; set; } = string.Empty;
    public int MaxPoolSize { get; set; } = 100;
    public int CommandTimeout { get; set; } = 30;
}

// ใน Program.cs
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));

// appsettings.json
// {
//   "Database": {
//     "ConnectionString": "...",
//     "MaxPoolSize": 50,
//     "CommandTimeout": 60
//   }
// }

// ใช้งานใน service
public class UserService
{
    private readonly DatabaseOptions _options;

    public UserService(IOptions<DatabaseOptions> options)
    {
        _options = options.Value;
    }
}
```

---

## 4. WebApplication - Middleware และ Routing

หลัง build ได้ `WebApplication` ใช้สำหรับ:
- เพิ่ม middleware
- กำหนด routes
- Start application

```csharp
var app = builder.Build();

// app มี properties เหล่านี้:
// app.Environment    - IWebHostEnvironment
// app.Configuration  - IConfiguration
// app.Logger        - ILogger
// app.Services      - IServiceProvider
// app.Urls          - collection of URLs
```

### 4.1 app.Use* - เพิ่ม Middleware

```csharp
// UseRouting - เปิดใช้ routing
app.UseRouting();

// UseAuthentication - ตรวจสอบ authentication
app.UseAuthentication();

// UseAuthorization - ตรวจสอบ authorization
app.UseAuthorization();

// UseHttpsRedirection - redirect HTTP -> HTTPS
app.UseHttpsRedirection();

// UseStaticFiles - serve static files จาก wwwroot
app.UseStaticFiles();

// UseSwagger / UseSwaggerUI
app.UseSwagger();
app.UseSwaggerUI();

// UseCors - เปิดใช้ CORS
app.UseCors("AllowFrontend");

// UseResponseCaching - caching responses
app.UseResponseCaching();

// UseExceptionHandler - handle exceptions
app.UseExceptionHandler("/error");

// UseDeveloperExceptionPage - แสดง error details (เฉพาะ dev)
app.UseDeveloperExceptionPage();

// Custom middleware
app.Use(async (context, next) =>
{
    // ทำงานก่อน request
    Console.WriteLine($"Request: {context.Request.Method} {context.Request.Path}");
    
    await next(context); // เรียก middleware ถัดไป
    
    // ทำงานหลัง response
    Console.WriteLine($"Response: {context.Response.StatusCode}");
});

// UseMiddleware<T> - เพิ่ม middleware class
app.UseMiddleware<RequestLoggingMiddleware>();
```

### 4.2 app.Map* - กำหนด Endpoints

```csharp
// MapGet - HTTP GET
app.MapGet("/", () => "Hello World!");

// MapPost - HTTP POST
app.MapPost("/items", (Item item) => Results.Created($"/items/{item.Id}", item));

// MapPut - HTTP PUT
app.MapPut("/items/{id}", (int id, Item item) => Results.Ok(item));

// MapDelete - HTTP DELETE
app.MapDelete("/items/{id}", (int id) => Results.NoContent());

// MapPatch - HTTP PATCH
app.MapPatch("/items/{id}", (int id, JsonPatchDocument<Item> patch) => Results.Ok());

// MapControllers - เพิ่ม controller endpoints
app.MapControllers();

// MapRazorPages - เพิ่ม Razor Pages endpoints
app.MapRazorPages();

// MapFallback - fallback route
app.MapFallback(() => Results.NotFound("Page not found"));

// MapGroup - จัดกลุ่ม endpoints
var api = app.MapGroup("/api/v1").RequireAuthorization();
api.MapGet("/users", () => { /* ... */ });
api.MapGet("/products", () => { /* ... */ });
```

---

## 5. Middleware Order (ลำดับสำคัญมาก!)

ลำดับของ middleware มีผลต่อการทำงาน ต้องเรียงให้ถูกต้อง:

```csharp
var app = builder.Build();

// ลำดับที่แนะนำสำหรับ Web API:
app.UseExceptionHandler("/error");     // 1. Exception handling (ควรอยู่ต้นสุด)
app.UseHsts();                          // 2. HSTS header
app.UseHttpsRedirection();              // 3. HTTPS redirect
app.UseStaticFiles();                   // 4. Static files
app.UseRouting();                       // 5. Routing
app.UseCors();                          // 6. CORS (หลัง routing)
app.UseAuthentication();               // 7. Authentication
app.UseAuthorization();                // 8. Authorization
app.UseResponseCaching();              // 9. Response caching
// app.MapControllers();               // 10. Endpoints (ท้ายสุด)
```

### ผลกระทบของ Order ที่ผิด

```csharp
// ❌ ผิด: Authorization ก่อน Authentication
app.UseAuthorization();    // จะไม่ทำงาน!
app.UseAuthentication();   

// ✅ ถูก: Authentication ก่อน Authorization
app.UseAuthentication();
app.UseAuthorization();

// ❌ ผิด: CORS หลัง Authentication
app.UseAuthentication();
app.UseCors();     // จะทำงาน แต่ preflight requests จะถูก reject

// ✅ ถูก: CORS ก่อน Authentication
app.UseCors();
app.UseAuthentication();
```

---

## 6. Environment (Development, Production, Staging)

### 6.1 ตรวจสอบ Environment

```csharp
var app = builder.Build();

// วิธีที่ 1: ผ่าน app.Environment
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}
else if (app.Environment.IsProduction())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}
else if (app.Environment.IsStaging())
{
    app.UseSwagger(); // อาจต้องการ swagger ใน staging ด้วย
}

// วิธีที่ 2: ใช้ custom environment name
if (app.Environment.IsEnvironment("QA"))
{
    // ...
}

// ใน builder stage
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDatabaseDeveloperPageExceptionFilter();
}
```

### 6.2 ตั้งค่า Environment

```bash
# ตั้งค่าผ่าน environment variable
export ASPNETCORE_ENVIRONMENT=Production
dotnet run

# ตั้งค่าผ่าน command line
dotnet run --environment Production

# Windows
set ASPNETCORE_ENVIRONMENT=Production
dotnet run
```

### 6.3 Environment-specific Configuration

```json
// appsettings.json (base config)
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}

// appsettings.Development.json (override สำหรับ dev)
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information"
    }
  }
}

// appsettings.Production.json (override สำหรับ production)
{
  "Logging": {
    "LogLevel": {
      "Default": "Error"
    }
  }
}
```

---

## 7. Logging

ASP.NET Core มี built-in logging system ที่ใช้งานง่าย:

### 7.1 การตั้งค่า Logging

```csharp
var builder = WebApplication.CreateBuilder(args);

// ตั้งค่า logging
builder.Logging
    .ClearProviders()                    // ลบ providers ทั้งหมด
    .AddConsole()                        // เพิ่ม console logging
    .AddDebug()                          // เพิ่ม debug logging
    .AddEventLog()                       // Windows Event Log
    .SetMinimumLevel(LogLevel.Debug);   // ตั้งค่า minimum level

// หรือใช้ configuration
builder.Logging.AddConfiguration(builder.Configuration.GetSection("Logging"));
```

### 7.2 Log Levels

```
Trace    (0) - ข้อมูลละเอียดมาก (development only)
Debug    (1) - ข้อมูล debug
Information (2) - ข้อมูลทั่วไป
Warning  (3) - คำเตือน
Error    (4) - ข้อผิดพลาด
Critical (5) - ข้อผิดพลาดร้ายแรง
None     (6) - ปิด logging
```

### 7.3 การใช้ ILogger

```csharp
// ใช้ผ่าน DI
public class WeatherService
{
    private readonly ILogger<WeatherService> _logger;

    public WeatherService(ILogger<WeatherService> logger)
    {
        _logger = logger;
    }

    public async Task<Weather> GetWeatherAsync(string city)
    {
        _logger.LogInformation("กำลังดึงข้อมูลอากาศสำหรับ {City}", city);
        
        try
        {
            // ... get weather data
            var weather = new Weather { City = city, Temperature = 30 };
            _logger.LogDebug("ดึงข้อมูลสำเร็จ: {Weather}", weather);
            return weather;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "เกิดข้อผิดพลาดในการดึงข้อมูลอากาศสำหรับ {City}", city);
            throw;
        }
    }
}

// ใน Minimal API
app.MapGet("/weather/{city}", (string city, ILogger<Program> logger) =>
{
    logger.LogInformation("Request สำหรับ weather ของ {City}", city);
    return new { City = city, Temperature = 30, Condition = "Sunny" };
});
```

### 7.4 Structured Logging

```csharp
// Structured logging - ใช้ {} placeholders แทน string interpolation
// ❌ ไม่แนะนำ
_logger.LogInformation($"User {userId} logged in at {DateTime.Now}");

// ✅ แนะนำ - Structured logging
_logger.LogInformation("User {UserId} logged in at {LoginTime}", userId, DateTime.Now);

// ช่วยให้ log aggregation tools (เช่น Elasticsearch, Seq) ค้นหาได้ง่าย
```

---

## 8. โปรแกรมตัวอย่าง: Hello World API

มาสร้าง Hello World API ที่สมบูรณ์กัน:

```csharp
// Program.cs
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// ============================================
// SERVICES CONFIGURATION
// ============================================

// Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Title = "Hello World API",
        Version = "v1",
        Description = "API ตัวอย่างสำหรับ Part 042"
    });
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader();
    });
});

// Logging
builder.Logging.AddConsole();

// สร้าง greeting service
builder.Services.AddSingleton<IGreetingService, GreetingService>();

var app = builder.Build();

// ============================================
// MIDDLEWARE PIPELINE
// ============================================

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    app.Logger.LogInformation("🚀 Running in Development mode");
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
    app.Logger.LogInformation("🚀 Running in Production mode");
}

app.UseHttpsRedirection();
app.UseCors();

// Custom middleware - log ทุก request
app.Use(async (context, next) =>
{
    var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
    var startTime = DateTime.UtcNow;
    
    logger.LogInformation("➡️ {Method} {Path} started", 
        context.Request.Method, context.Request.Path);
    
    await next(context);
    
    var elapsed = (DateTime.UtcNow - startTime).TotalMilliseconds;
    logger.LogInformation("⬅️ {Method} {Path} completed {StatusCode} in {Elapsed}ms",
        context.Request.Method, context.Request.Path, 
        context.Response.StatusCode, elapsed);
});

// ============================================
// ENDPOINTS
// ============================================

// Endpoint กลุ่ม hello
var helloGroup = app.MapGroup("/hello").WithTags("Hello");

helloGroup.MapGet("/", () => new { Message = "Hello, World!", Time = DateTime.UtcNow })
    .WithSummary("ทักทายทั่วไป");

helloGroup.MapGet("/{name}", (string name, IGreetingService greetingService) =>
{
    var greeting = greetingService.Greet(name);
    return Results.Ok(new { Message = greeting, Name = name });
})
.WithSummary("ทักทายตามชื่อ");

helloGroup.MapPost("/", ([FromBody] GreetRequest request, IGreetingService greetingService) =>
{
    if (string.IsNullOrWhiteSpace(request.Name))
    {
        return Results.BadRequest(new { Error = "กรุณาระบุชื่อ" });
    }

    var greeting = greetingService.Greet(request.Name, request.Language);
    return Results.Ok(new { Message = greeting });
})
.WithSummary("ทักทายพร้อมระบุภาษา");

// Endpoint แสดง server info
app.MapGet("/info", (IWebHostEnvironment env) => new
{
    Environment = env.EnvironmentName,
    ApplicationName = env.ApplicationName,
    ServerTime = DateTime.UtcNow,
    Runtime = System.Runtime.InteropServices.RuntimeInformation.FrameworkDescription
})
.WithTags("System")
.WithSummary("ข้อมูล server");

// Error handling endpoint
app.MapGet("/error", () => Results.Problem("เกิดข้อผิดพลาดที่ไม่คาดคิด"))
    .ExcludeFromDescription();

// Root redirect ไป swagger
app.MapGet("/", () => Results.Redirect("/swagger"));

app.Logger.LogInformation("✅ Application configured successfully");
app.Run();

// ============================================
// MODELS & INTERFACES
// ============================================

public record GreetRequest(string Name, string Language = "th");

public interface IGreetingService
{
    string Greet(string name, string language = "th");
}

public class GreetingService : IGreetingService
{
    private readonly ILogger<GreetingService> _logger;

    private readonly Dictionary<string, string> _greetings = new()
    {
        ["th"] = "สวัสดี",
        ["en"] = "Hello",
        ["ja"] = "こんにちは",
        ["zh"] = "你好",
        ["ko"] = "안녕하세요",
        ["fr"] = "Bonjour",
        ["de"] = "Hallo",
        ["es"] = "Hola"
    };

    public GreetingService(ILogger<GreetingService> logger)
    {
        _logger = logger;
    }

    public string Greet(string name, string language = "th")
    {
        _logger.LogDebug("สร้าง greeting สำหรับ {Name} ในภาษา {Language}", name, language);
        
        var greeting = _greetings.TryGetValue(language.ToLower(), out var g) 
            ? g 
            : _greetings["en"];
            
        return $"{greeting}, {name}!";
    }
}
```

### ทดสอบ API

```bash
# รัน
dotnet run

# ทักทายทั่วไป
curl https://localhost:7001/hello

# ทักทายตามชื่อ
curl https://localhost:7001/hello/John

# ทักทายพร้อมระบุภาษา
curl -X POST https://localhost:7001/hello \
  -H "Content-Type: application/json" \
  -d '{"name": "สมชาย", "language": "th"}'

# ข้อมูล server
curl https://localhost:7001/info
```

### ผลลัพธ์ที่คาดหวัง

```json
// GET /hello
{
  "message": "Hello, World!",
  "time": "2024-01-15T10:30:00Z"
}

// GET /hello/สมชาย
{
  "message": "สวัสดี, สมชาย!",
  "name": "สมชาย"
}

// POST /hello
{
  "message": "สวัสดี, สมชาย!"
}

// GET /info
{
  "environment": "Development",
  "applicationName": "HelloWorldApi",
  "serverTime": "2024-01-15T10:30:00Z",
  "runtime": ".NET 9.0.0"
}
```

---

## 9. WebApplication Builder ขั้นสูง

### 9.1 Custom Configuration Sources

```csharp
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม custom configuration source
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddEnvironmentVariables(prefix: "MYAPP_")   // เฉพาะ env vars ที่ขึ้นต้นด้วย MYAPP_
    .AddCommandLine(args)
    .AddUserSecrets<Program>(optional: true);     // User secrets (development only)
```

### 9.2 Generic Host vs Web Host

```csharp
// WebApplication.CreateBuilder - สำหรับ web applications
var webBuilder = WebApplication.CreateBuilder(args);

// Host.CreateDefaultBuilder - สำหรับ background services
var hostBuilder = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddHostedService<MyBackgroundService>();
    });
```

### 9.3 การใช้ SlimBuilder สำหรับ AOT

```csharp
// สำหรับ Ahead-of-Time compilation (ลด startup time)
var builder = WebApplication.CreateSlimBuilder(args);

// ต้องใช้ source generation สำหรับ JSON serialization
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

var app = builder.Build();
app.MapGet("/", () => new Todo(1, "Test", false));
app.Run();

// Source generation context
[JsonSerializable(typeof(Todo))]
internal partial class AppJsonSerializerContext : JsonSerializerContext { }

record Todo(int Id, string Title, bool IsCompleted);
```

---

## Exercises

### Exercise 1: สร้าง System Info API
สร้าง API ที่แสดงข้อมูลระบบ:

```csharp
// สร้าง endpoints เหล่านี้:
// GET /system/info - แสดงข้อมูล OS, .NET version, machine name
// GET /system/memory - แสดง memory usage
// GET /system/time - แสดงเวลาปัจจุบันในหลาย timezone
// GET /system/environment - แสดง environment variables (เฉพาะ dev)

// ใช้ System.Runtime.InteropServices สำหรับ OS info
// ใช้ GC.GetTotalMemory() สำหรับ memory
// ใช้ TimeZoneInfo สำหรับ timezone
```

### Exercise 2: Request Counter Middleware
สร้าง middleware ที่นับจำนวน requests:

```csharp
// Middleware ต้องทำ:
// 1. นับ request ทุกครั้ง
// 2. เพิ่ม header X-Request-Count ใน response
// 3. แสดง endpoint GET /stats ที่แสดงจำนวน requests แยกตาม path
// 4. ใช้ ConcurrentDictionary สำหรับ thread-safe counting

public class RequestCounterMiddleware
{
    private static readonly ConcurrentDictionary<string, int> _counts = new();
    // ... implement
}
```

### Exercise 3: Environment-aware Configuration
สร้าง API ที่แสดงพฤติกรรมต่างกันตาม environment:

```csharp
// Development: แสดง detailed error + stack trace
// Staging: แสดง error message แต่ไม่ stack trace  
// Production: แสดงเพียง "Internal Server Error"
// ทุก environment: เพิ่ม X-Environment header ใน response
```

---

## สรุป

✅ .NET 6+ ใช้ Minimal API pattern - ไม่ต้องมี Startup.cs แยก  
✅ `WebApplication.CreateBuilder(args)` สร้าง builder สำหรับตั้งค่า application  
✅ `builder.Services` ใช้ลงทะเบียน services ใน DI container  
✅ `builder.Build()` สร้าง `WebApplication` instance  
✅ `app.Use*` เพิ่ม middleware, `app.Map*` กำหนด endpoints  
✅ ลำดับ middleware มีผลต่อการทำงาน - Authentication ก่อน Authorization  
✅ Environment ควบคุมพฤติกรรมของ app (Development, Staging, Production)  
✅ Structured logging ดีกว่า string interpolation สำหรับ log aggregation  
✅ `app.MapGroup()` ช่วยจัดกลุ่ม endpoints  

---

## Part ถัดไป

ใน **Part 043** เราจะเรียนรู้เรื่อง **Middleware Pipeline** อย่างละเอียด:
- Middleware concept และ how it works
- Built-in middleware ที่สำคัญ
- การสร้าง Custom Middleware
- Request/Response pipeline

---

*Part 042/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
