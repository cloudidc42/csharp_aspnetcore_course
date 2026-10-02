# Part 044: Dependency Injection (DI)

## เนื้อหาใน Part นี้
- IoC container และ Dependency Injection concept
- Service lifetimes: Singleton, Scoped, Transient
- Constructor injection
- IServiceCollection.Add*
- IServiceProvider
- Resolve services
- โปรแกรมตัวอย่าง: Weather service with DI

---

## 1. Dependency Injection คืออะไร?

Dependency Injection (DI) คือ design pattern ที่ช่วยลด coupling ระหว่าง class โดยการ "inject" dependencies จากภายนอกแทนที่จะให้ class สร้าง dependencies เอง

### ปัญหาก่อนใช้ DI

```csharp
// ❌ แบบ tight coupling - class สร้าง dependency เอง
public class UserService
{
    private readonly EmailService _emailService;
    private readonly UserRepository _userRepository;

    public UserService()
    {
        // สร้าง dependencies เอง - ทดสอบยาก, แก้ไขยาก!
        _emailService = new EmailService("smtp.gmail.com", 587);
        _userRepository = new UserRepository("connection-string");
    }

    public async Task RegisterUserAsync(string email, string password)
    {
        await _userRepository.CreateAsync(new User { Email = email });
        await _emailService.SendWelcomeEmailAsync(email);
    }
}

// ใช้งาน
var service = new UserService(); // ต้องรู้ว่า dependencies ต้องการอะไร
```

**ปัญหา:**
- ทดสอบยาก (ไม่สามารถ mock EmailService ได้)
- เปลี่ยน implementation ยาก
- Tight coupling ระหว่าง classes
- ละเมิด Dependency Inversion Principle

### แก้ปัญหาด้วย DI

```csharp
// ✅ แบบ Dependency Injection - inject ผ่าน interface

// กำหนด interface
public interface IEmailService
{
    Task SendWelcomeEmailAsync(string email);
}

public interface IUserRepository
{
    Task<User> CreateAsync(User user);
}

// UserService รับ dependencies ผ่าน constructor
public class UserService
{
    private readonly IEmailService _emailService;
    private readonly IUserRepository _userRepository;

    public UserService(IEmailService emailService, IUserRepository userRepository)
    {
        _emailService = emailService;
        _userRepository = userRepository;
    }

    public async Task RegisterUserAsync(string email, string password)
    {
        await _userRepository.CreateAsync(new User { Email = email });
        await _emailService.SendWelcomeEmailAsync(email);
    }
}

// ลงทะเบียนใน DI container
builder.Services.AddScoped<IEmailService, GmailEmailService>();
builder.Services.AddScoped<IUserRepository, SqlUserRepository>();
builder.Services.AddScoped<UserService>();

// DI container สร้าง UserService พร้อม inject dependencies อัตโนมัติ
```

---

## 2. IoC Container ใน ASP.NET Core

ASP.NET Core มี built-in IoC container ที่ใช้ผ่าน `IServiceCollection`:

```
IServiceCollection (registration)
        │
        ▼
IServiceProvider (runtime resolution)
        │
        ├─ Create instances
        ├─ Manage lifetimes  
        └─ Inject dependencies
```

### การทำงานของ DI Container

```csharp
// ขั้นตอนที่ 1: Registration (ลงทะเบียน services)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();

var app = builder.Build();

// ขั้นตอนที่ 2: Resolution (ขอใช้ service)
// DI container จะสร้าง instance พร้อม inject dependencies อัตโนมัติ
app.MapGet("/users", (IUserService userService) => 
{
    // userService ถูก inject มาให้อัตโนมัติ
    return userService.GetAllUsers();
});
```

---

## 3. Service Lifetimes

### 3.1 Transient

สร้าง instance ใหม่ **ทุกครั้ง** ที่ request

```csharp
// ลงทะเบียน
builder.Services.AddTransient<IMyService, MyService>();

// หรือ
builder.Services.AddTransient<MyService>();

// ใช้งาน
public class MyController
{
    public MyController(IMyService service1, IMyService service2)
    {
        // service1 และ service2 เป็น instance คนละตัว!
        Console.WriteLine(service1 == service2); // false
    }
}
```

**เหมาะสำหรับ:**
- Services ที่ lightweight
- Services ที่ไม่มี state
- หรือเมื่อต้องการ new instance ทุกครั้ง

```csharp
// ตัวอย่าง: Email sender, validator, formatter
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.AddTransient<IPasswordHasher, BcryptPasswordHasher>();
```

### 3.2 Scoped

สร้าง instance ใหม่ **ต่อ request** (หนึ่ง instance ต่อ HTTP request)

```csharp
// ลงทะเบียน
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// ใช้งาน
public class OrderController
{
    private readonly IOrderService _orderService;
    private readonly IUserService _userService;
    private readonly AppDbContext _context;

    public OrderController(
        IOrderService orderService,
        IUserService userService, 
        AppDbContext context)
    {
        // ทั้งหมดเป็น instance เดียวกันใน request เดียวกัน
        _orderService = orderService;
        _userService = userService;
        _context = context;
    }
}
```

**เหมาะสำหรับ:**
- Database context (DbContext)
- Repository classes
- Unit of Work pattern
- Services ที่ต้องการ consistency ใน request เดียวกัน

### 3.3 Singleton

สร้าง instance เดียว **ตลอด lifetime** ของ application

```csharp
// ลงทะเบียน
builder.Services.AddSingleton<ICacheService, MemoryCacheService>();
builder.Services.AddSingleton<IConfiguration>(_ => config);

// ลงทะเบียนด้วย instance ที่สร้างเอง
var cache = new MemoryCacheService();
builder.Services.AddSingleton<ICacheService>(cache);

// ลงทะเบียนด้วย factory function
builder.Services.AddSingleton<ICacheService>(provider =>
{
    var logger = provider.GetRequiredService<ILogger<MemoryCacheService>>();
    return new MemoryCacheService(logger, capacity: 1000);
});
```

**เหมาะสำหรับ:**
- In-memory cache
- Configuration
- Logging
- Services ที่ expensive ในการสร้าง
- Services ที่แชร์ state ระหว่าง requests

**⚠️ ข้อควรระวัง Singleton:**
```csharp
// ❌ อย่าใช้ Scoped service ใน Singleton!
public class MySingletonService
{
    private readonly IScopedService _scopedService; // WRONG!
    
    public MySingletonService(IScopedService scopedService) // จะเกิด error หรือ captive dependency
    {
        _scopedService = scopedService;
    }
}

// ✅ ถ้าต้องการ Scoped service ใน Singleton ให้ใช้ IServiceScopeFactory
public class MySingletonService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public MySingletonService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
        await scopedService.ProcessAsync();
    }
}
```

### 3.4 เปรียบเทียบ Lifetimes

| Lifetime | สร้างใหม่เมื่อ | จำนวน instance | เหมาะกับ |
|----------|--------------|----------------|---------|
| Transient | ทุก request | หลาย instances | Lightweight, stateless |
| Scoped | ต่อ HTTP request | 1 ต่อ request | DB context, repositories |
| Singleton | ครั้งแรก | 1 ตลอด app | Cache, config, logging |

```csharp
// ทดสอบ lifetimes
public class LifetimeDemo
{
    public Guid Id { get; } = Guid.NewGuid();
}

builder.Services.AddTransient<TransientService>();
builder.Services.AddScoped<ScopedService>();
builder.Services.AddSingleton<SingletonService>();

app.MapGet("/lifetimes", (
    TransientService t1, TransientService t2,
    ScopedService s1, ScopedService s2,
    SingletonService sg1, SingletonService sg2) =>
{
    return new
    {
        Transient = new { 
            Same = t1.Id == t2.Id,  // false - คนละ instance
            Id1 = t1.Id, 
            Id2 = t2.Id 
        },
        Scoped = new { 
            Same = s1.Id == s2.Id,   // true - instance เดียวกันใน request นี้
            Id1 = s1.Id 
        },
        Singleton = new { 
            Same = sg1.Id == sg2.Id, // true - instance เดียวกันตลอด
            Id = sg1.Id 
        }
    };
});
```

---

## 4. Constructor Injection

```csharp
// Constructor injection - วิธีหลักในการ inject dependencies
public class ProductService
{
    private readonly IProductRepository _repository;
    private readonly ICacheService _cache;
    private readonly ILogger<ProductService> _logger;
    private readonly IMapper _mapper;

    // Constructor รับ dependencies ทั้งหมดที่ต้องการ
    public ProductService(
        IProductRepository repository,
        ICacheService cache,
        ILogger<ProductService> logger,
        IMapper mapper)
    {
        _repository = repository ?? throw new ArgumentNullException(nameof(repository));
        _cache = cache ?? throw new ArgumentNullException(nameof(cache));
        _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        _mapper = mapper ?? throw new ArgumentNullException(nameof(mapper));
    }

    public async Task<List<ProductDto>> GetAllAsync()
    {
        var cacheKey = "all-products";
        
        if (_cache.TryGetValue(cacheKey, out List<ProductDto>? cached))
        {
            _logger.LogDebug("Cache hit for {Key}", cacheKey);
            return cached!;
        }

        _logger.LogInformation("Loading products from database");
        var products = await _repository.GetAllAsync();
        var dtos = _mapper.Map<List<ProductDto>>(products);
        
        _cache.Set(cacheKey, dtos, TimeSpan.FromMinutes(5));
        return dtos;
    }
}
```

### Optional Dependencies

```csharp
// ใช้ ? หรือ default value สำหรับ optional dependencies
public class NotificationService
{
    private readonly IEmailService _emailService;
    private readonly ISmsService? _smsService;  // Optional

    public NotificationService(
        IEmailService emailService,
        ISmsService? smsService = null)  // Optional
    {
        _emailService = emailService;
        _smsService = smsService;
    }

    public async Task SendAsync(string message, string? phone = null)
    {
        await _emailService.SendAsync(message);
        
        // ใช้เฉพาะเมื่อมี SMS service
        if (_smsService != null && phone != null)
        {
            await _smsService.SendAsync(phone, message);
        }
    }
}

// ลงทะเบียน (SMS service optional)
builder.Services.AddScoped<IEmailService, EmailService>();
// builder.Services.AddScoped<ISmsService, TwilioSmsService>(); // optional
```

---

## 5. IServiceCollection Methods

### 5.1 Add Methods

```csharp
// เพิ่ม service ด้วย interface และ implementation
builder.Services.AddSingleton<IMyService, MyService>();
builder.Services.AddScoped<IMyService, MyService>();
builder.Services.AddTransient<IMyService, MyService>();

// เพิ่ม service โดยไม่มี interface
builder.Services.AddSingleton<MyService>();

// เพิ่มด้วย factory function
builder.Services.AddScoped<IMyService>(provider =>
{
    var config = provider.GetRequiredService<IConfiguration>();
    return new MyService(config["ApiKey"]!);
});

// เพิ่มด้วย instance โดยตรง
var myService = new MyService("fixed-value");
builder.Services.AddSingleton<IMyService>(myService);
```

### 5.2 TryAdd Methods (ไม่ override ถ้ามีอยู่แล้ว)

```csharp
// TryAdd - เพิ่มเฉพาะกรณีที่ยังไม่มี
builder.Services.TryAddScoped<IMyService, MyService>();
builder.Services.TryAddSingleton<IMyService, MyService>();

// มีประโยชน์สำหรับ library ที่ต้องการให้ user override ได้
// user code:
builder.Services.AddScoped<IMyService, CustomService>();

// library code (จะไม่ override เพราะ user เพิ่มไปแล้ว):
builder.Services.TryAddScoped<IMyService, DefaultService>();
```

### 5.3 Multiple Registrations

```csharp
// เพิ่ม service หลายตัวที่ implement interface เดียวกัน
builder.Services.AddTransient<INotificationHandler, EmailNotificationHandler>();
builder.Services.AddTransient<INotificationHandler, SmsNotificationHandler>();
builder.Services.AddTransient<INotificationHandler, PushNotificationHandler>();

// รับ IEnumerable<INotificationHandler> ใน class
public class NotificationService
{
    private readonly IEnumerable<INotificationHandler> _handlers;

    public NotificationService(IEnumerable<INotificationHandler> handlers)
    {
        _handlers = handlers;
    }

    public async Task NotifyAsync(string message)
    {
        // ส่งทุก channels
        foreach (var handler in _handlers)
        {
            await handler.HandleAsync(message);
        }
    }
}
```

### 5.4 Decorated Services

```csharp
// Decorator pattern ด้วย DI
// Original service
builder.Services.AddScoped<IUserService, UserService>();

// Decorate ด้วย caching
builder.Services.Decorate<IUserService, CachedUserService>();

// ถ้าไม่มี library สำหรับ Decorate ทำได้ด้วยตนเอง:
builder.Services.AddScoped<UserService>();
builder.Services.AddScoped<IUserService>(provider =>
{
    var inner = provider.GetRequiredService<UserService>();
    var cache = provider.GetRequiredService<IMemoryCache>();
    return new CachedUserService(inner, cache);
});
```

---

## 6. IServiceProvider

### 6.1 GetService vs GetRequiredService

```csharp
// GetService - return null ถ้าไม่พบ
var service = provider.GetService<IMyService>();
if (service != null) { /* use it */ }

// GetRequiredService - throw exception ถ้าไม่พบ
var service = provider.GetRequiredService<IMyService>(); // throw InvalidOperationException ถ้าไม่พบ

// GetServices - ดึงทุก registrations ของ type
var services = provider.GetServices<INotificationHandler>();
```

### 6.2 สร้าง Scope

```csharp
// สร้าง scope เองเมื่อต้องการใช้ scoped services นอก HTTP context
// (เช่น ใน background service)
public class DataProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public DataProcessingService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // สร้าง scope สำหรับ processing แต่ละรอบ
            using (var scope = _scopeFactory.CreateScope())
            {
                var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                var repository = scope.ServiceProvider.GetRequiredService<IDataRepository>();
                
                await ProcessDataAsync(dbContext, repository, stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

### 6.3 Service Locator Pattern (ไม่แนะนำ)

```csharp
// ❌ Service Locator - anti-pattern โดยทั่วไป
public class MyService
{
    private readonly IServiceProvider _provider;

    public MyService(IServiceProvider provider)
    {
        _provider = provider;
    }

    public void DoWork()
    {
        // ไม่ดี: ซ่อน dependencies, ทดสอบยาก
        var repo = _provider.GetRequiredService<IRepository>();
        // ...
    }
}

// ✅ ใช้ constructor injection แทน
public class MyService
{
    private readonly IRepository _repo;

    public MyService(IRepository repo)  // dependencies ชัดเจน
    {
        _repo = repo;
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Weather Service with DI

```csharp
// Program.cs
using System.Collections.Concurrent;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ลงทะเบียน services
builder.Services.AddSingleton<IWeatherCache, InMemoryWeatherCache>();
builder.Services.AddHttpClient<IWeatherApiClient, OpenWeatherApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.openweathermap.org/");
    client.Timeout = TimeSpan.FromSeconds(10);
});
builder.Services.AddScoped<IWeatherService, WeatherService>();

// ลงทะเบียน Options
builder.Services.Configure<WeatherOptions>(
    builder.Configuration.GetSection("Weather"));

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Endpoints
var weatherGroup = app.MapGroup("/api/weather").WithTags("Weather");

weatherGroup.MapGet("/{city}", async (string city, IWeatherService weatherService) =>
{
    var weather = await weatherService.GetWeatherAsync(city);
    return weather is null
        ? Results.NotFound(new { Message = $"ไม่พบข้อมูลอากาศสำหรับ {city}" })
        : Results.Ok(weather);
})
.WithSummary("ดึงข้อมูลอากาศตามเมือง");

weatherGroup.MapGet("/", async (IWeatherService weatherService) =>
{
    var cities = new[] { "Bangkok", "Tokyo", "London", "New York", "Sydney" };
    var tasks = cities.Select(city => weatherService.GetWeatherAsync(city));
    var results = await Task.WhenAll(tasks);
    return Results.Ok(results.Where(r => r != null));
})
.WithSummary("ดึงข้อมูลอากาศหลายเมือง");

weatherGroup.MapDelete("/cache", (IWeatherCache cache) =>
{
    cache.Clear();
    return Results.Ok(new { Message = "ล้าง cache สำเร็จ" });
})
.WithSummary("ล้าง cache");

app.Run();

// ============================================
// MODELS
// ============================================

public record WeatherData(
    string City,
    double TemperatureCelsius,
    double TemperatureFahrenheit,
    string Condition,
    int Humidity,
    double WindSpeed,
    DateTime RecordedAt
);

public class WeatherOptions
{
    public const string SectionName = "Weather";
    public string ApiKey { get; set; } = string.Empty;
    public int CacheDurationMinutes { get; set; } = 15;
    public bool UseRealApi { get; set; } = false;
}

// ============================================
// INTERFACES
// ============================================

public interface IWeatherService
{
    Task<WeatherData?> GetWeatherAsync(string city);
}

public interface IWeatherApiClient
{
    Task<WeatherData?> FetchWeatherAsync(string city);
}

public interface IWeatherCache
{
    bool TryGet(string city, out WeatherData? data);
    void Set(string city, WeatherData data, TimeSpan duration);
    void Clear();
}

// ============================================
// IMPLEMENTATIONS
// ============================================

// WeatherService - Scoped
public class WeatherService : IWeatherService
{
    private readonly IWeatherApiClient _apiClient;
    private readonly IWeatherCache _cache;
    private readonly ILogger<WeatherService> _logger;
    private readonly WeatherOptions _options;

    public WeatherService(
        IWeatherApiClient apiClient,
        IWeatherCache cache,
        ILogger<WeatherService> logger,
        IOptions<WeatherOptions> options)
    {
        _apiClient = apiClient;
        _cache = cache;
        _logger = logger;
        _options = options.Value;
    }

    public async Task<WeatherData?> GetWeatherAsync(string city)
    {
        // Check cache first
        if (_cache.TryGet(city, out var cached))
        {
            _logger.LogDebug("Cache hit: {City}", city);
            return cached;
        }

        _logger.LogInformation("Fetching weather for {City}", city);

        var weather = await _apiClient.FetchWeatherAsync(city);
        
        if (weather != null)
        {
            _cache.Set(city, weather, TimeSpan.FromMinutes(_options.CacheDurationMinutes));
            _logger.LogInformation("Weather fetched and cached for {City}", city);
        }

        return weather;
    }
}

// OpenWeatherApiClient - ใช้ HttpClient
public class OpenWeatherApiClient : IWeatherApiClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<OpenWeatherApiClient> _logger;
    private readonly WeatherOptions _options;

    // Mock data สำหรับ demo (ไม่ต้องใช้ API key จริง)
    private static readonly Dictionary<string, WeatherData> MockData = new()
    {
        ["Bangkok"] = new("Bangkok", 34.5, 94.1, "Sunny", 75, 12.5, DateTime.UtcNow),
        ["Tokyo"] = new("Tokyo", 18.3, 64.9, "Cloudy", 60, 8.2, DateTime.UtcNow),
        ["London"] = new("London", 12.1, 53.8, "Rainy", 85, 15.3, DateTime.UtcNow),
        ["New York"] = new("New York", 22.4, 72.3, "Partly Cloudy", 55, 20.1, DateTime.UtcNow),
        ["Sydney"] = new("Sydney", 25.6, 78.1, "Clear", 50, 10.8, DateTime.UtcNow)
    };

    public OpenWeatherApiClient(
        HttpClient httpClient,
        ILogger<OpenWeatherApiClient> logger,
        IOptions<WeatherOptions> options)
    {
        _httpClient = httpClient;
        _logger = logger;
        _options = options.Value;
    }

    public async Task<WeatherData?> FetchWeatherAsync(string city)
    {
        // ใช้ mock data ใน demo
        if (!_options.UseRealApi)
        {
            await Task.Delay(100); // simulate API call
            
            if (MockData.TryGetValue(city, out var mockData))
                return mockData with { RecordedAt = DateTime.UtcNow };
            
            return null;
        }

        // ใช้ OpenWeather API จริง
        try
        {
            var response = await _httpClient.GetFromJsonAsync<OpenWeatherResponse>(
                $"data/2.5/weather?q={city}&appid={_options.ApiKey}&units=metric");

            if (response == null) return null;

            var celsius = response.Main.Temp;
            return new WeatherData(
                City: city,
                TemperatureCelsius: celsius,
                TemperatureFahrenheit: (celsius * 9 / 5) + 32,
                Condition: response.Weather.FirstOrDefault()?.Description ?? "Unknown",
                Humidity: response.Main.Humidity,
                WindSpeed: response.Wind.Speed,
                RecordedAt: DateTime.UtcNow
            );
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to fetch weather for {City}", city);
            return null;
        }
    }
}

// Mock response model สำหรับ OpenWeather API
public class OpenWeatherResponse
{
    public MainData Main { get; set; } = new();
    public List<WeatherDescription> Weather { get; set; } = [];
    public WindData Wind { get; set; } = new();

    public class MainData
    {
        public double Temp { get; set; }
        public int Humidity { get; set; }
    }

    public class WeatherDescription
    {
        public string Description { get; set; } = string.Empty;
    }

    public class WindData
    {
        public double Speed { get; set; }
    }
}

// InMemoryWeatherCache - Singleton
public class InMemoryWeatherCache : IWeatherCache
{
    private readonly ConcurrentDictionary<string, CacheEntry> _cache = new();
    private readonly ILogger<InMemoryWeatherCache> _logger;

    public InMemoryWeatherCache(ILogger<InMemoryWeatherCache> logger)
    {
        _logger = logger;
    }

    public bool TryGet(string city, out WeatherData? data)
    {
        if (_cache.TryGetValue(city.ToLower(), out var entry))
        {
            if (entry.ExpiresAt > DateTime.UtcNow)
            {
                data = entry.Data;
                return true;
            }
            
            // Expired - remove
            _cache.TryRemove(city.ToLower(), out _);
        }

        data = null;
        return false;
    }

    public void Set(string city, WeatherData data, TimeSpan duration)
    {
        _cache[city.ToLower()] = new CacheEntry(data, DateTime.UtcNow.Add(duration));
        _logger.LogDebug("Cached weather for {City} until {Expiry}", city, DateTime.UtcNow.Add(duration));
    }

    public void Clear()
    {
        _cache.Clear();
        _logger.LogInformation("Weather cache cleared");
    }

    private record CacheEntry(WeatherData Data, DateTime ExpiresAt);
}
```

### appsettings.json

```json
{
  "Weather": {
    "ApiKey": "your-openweather-api-key",
    "CacheDurationMinutes": 15,
    "UseRealApi": false
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### ทดสอบ API

```bash
dotnet run

# ดึงอากาศกรุงเทพ
curl https://localhost:7001/api/weather/Bangkok

# ดึงอากาศหลายเมือง
curl https://localhost:7001/api/weather/

# ล้าง cache
curl -X DELETE https://localhost:7001/api/weather/cache
```

### ผลลัพธ์

```json
{
  "city": "Bangkok",
  "temperatureCelsius": 34.5,
  "temperatureFahrenheit": 94.1,
  "condition": "Sunny",
  "humidity": 75,
  "windSpeed": 12.5,
  "recordedAt": "2024-01-15T10:30:00Z"
}
```

---

## 8. Advanced DI Patterns

### 8.1 Keyed Services (.NET 8+)

```csharp
// ลงทะเบียน services ด้วย key
builder.Services.AddKeyedSingleton<IPaymentService, StripePaymentService>("stripe");
builder.Services.AddKeyedSingleton<IPaymentService, PayPalPaymentService>("paypal");
builder.Services.AddKeyedSingleton<IPaymentService, OmisePaymentService>("omise");

// ใช้งานด้วย [FromKeyedServices]
app.MapPost("/payment/{provider}", async (
    string provider,
    [FromKeyedServices("stripe")] IPaymentService stripeService,
    PaymentRequest request) =>
{
    // ใช้ stripe service
    return await stripeService.ProcessAsync(request);
});

// ใช้ใน controller/service
public class PaymentController
{
    public PaymentController(
        [FromKeyedServices("stripe")] IPaymentService stripeService,
        [FromKeyedServices("paypal")] IPaymentService paypalService)
    {
        // ...
    }
}

// หรือดึงด้วย key
app.MapPost("/payment", async (
    string provider,
    IServiceProvider services,
    PaymentRequest request) =>
{
    var paymentService = services.GetRequiredKeyedService<IPaymentService>(provider);
    return await paymentService.ProcessAsync(request);
});
```

### 8.2 Generic Services

```csharp
// ลงทะเบียน generic service
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// ใช้งาน
public class ProductService
{
    private readonly IRepository<Product> _productRepo;
    private readonly IRepository<Category> _categoryRepo;

    public ProductService(
        IRepository<Product> productRepo,
        IRepository<Category> categoryRepo)
    {
        _productRepo = productRepo;
        _categoryRepo = categoryRepo;
    }
}

// Interface และ Implementation
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
}

public class Repository<T> : IRepository<T> where T : class
{
    private readonly AppDbContext _context;
    
    public Repository(AppDbContext context)
    {
        _context = context;
    }
    
    // ... implementations
}
```

### 8.3 Validation ด้วย DI

```csharp
// Validation service
public interface IValidator<T>
{
    ValidationResult Validate(T model);
}

public class UserRegistrationValidator : IValidator<UserRegistrationDto>
{
    public ValidationResult Validate(UserRegistrationDto model)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(model.Email))
            errors.Add("Email is required");
        else if (!model.Email.Contains('@'))
            errors.Add("Email format is invalid");
            
        if (model.Password.Length < 8)
            errors.Add("Password must be at least 8 characters");
            
        return new ValidationResult(errors);
    }
}

builder.Services.AddScoped<IValidator<UserRegistrationDto>, UserRegistrationValidator>();
```

---

## Exercises

### Exercise 1: สร้าง Product Service ด้วย DI
สร้าง service สำหรับจัดการ products โดยใช้ DI pattern:

```csharp
// ต้องการ:
// 1. IProductService interface
// 2. ProductService implementation (Scoped)
// 3. IProductRepository interface  
// 4. InMemoryProductRepository implementation (Singleton)
// 5. IProductValidator interface
// 6. ProductValidator implementation (Transient)
// 7. ลงทะเบียนใน DI container และสร้าง endpoints

public interface IProductService
{
    Task<List<Product>> GetAllAsync(ProductFilter? filter = null);
    Task<Product?> GetByIdAsync(int id);
    Task<Product> CreateAsync(CreateProductDto dto);
    Task<Product?> UpdateAsync(int id, UpdateProductDto dto);
    Task<bool> DeleteAsync(int id);
}
```

### Exercise 2: Cache Decorator
สร้าง decorator สำหรับ cache ที่ wraps existing service:

```csharp
// ต้องการ:
// 1. สร้าง CachedProductService ที่ wraps IProductService
// 2. Cache results ใน IMemoryCache
// 3. Invalidate cache เมื่อ create/update/delete
// 4. ลงทะเบียน decorator ใน DI container

public class CachedProductService : IProductService
{
    private readonly IProductService _inner;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedProductService> _logger;
    
    // TODO: implement
}
```

### Exercise 3: Multiple Notification Handlers
สร้าง notification system ที่ส่งผ่านหลาย channels:

```csharp
// ต้องการ:
// 1. INotificationHandler interface
// 2. EmailNotificationHandler implementation
// 3. LogNotificationHandler implementation
// 4. NotificationService ที่รับ IEnumerable<INotificationHandler>
// 5. ลงทะเบียน handlers ทั้งหมด
// 6. ทดสอบว่า notification ถูกส่งผ่านทุก handler
```

---

## สรุป

✅ Dependency Injection ลด coupling ระหว่าง classes  
✅ ASP.NET Core มี built-in DI container ผ่าน `IServiceCollection`  
✅ Transient - instance ใหม่ทุกครั้ง, Scoped - instance ต่อ request, Singleton - instance เดียวตลอด  
✅ Constructor injection คือวิธีหลักในการ inject dependencies  
✅ Singleton ไม่สามารถรับ Scoped service ใน constructor ได้ (ใช้ IServiceScopeFactory แทน)  
✅ `GetRequiredService<T>()` throw exception ถ้าไม่พบ, `GetService<T>()` return null  
✅ Keyed services (.NET 8+) ช่วย register services หลายตัวที่มี interface เดียวกัน  
✅ Generic services ด้วย `typeof(IRepository<>)` ช่วยลดการลงทะเบียนซ้ำ  

---

## Part ถัดไป

ใน **Part 045** เราจะเรียนรู้เรื่อง **Configuration และ appsettings** อย่างละเอียด:
- appsettings.json structure
- IConfiguration
- Environment-specific config
- User Secrets
- Environment Variables
- Options pattern

---

*Part 044/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
