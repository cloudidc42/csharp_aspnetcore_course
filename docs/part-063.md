# Part 063: Caching ใน ASP.NET Core

## เนื้อหาใน Part นี้
- In-memory cache ด้วย IMemoryCache
- Distributed cache ด้วย IDistributedCache
- Redis cache
- Response caching
- Cache invalidation strategies
- โปรแกรมตัวอย่าง: Product Catalog Caching

---

## 1. In-Memory Cache (IMemoryCache)

**IMemoryCache** เก็บ cache ใน memory ของ application เหมาะสำหรับ single-server deployment

### การตั้งค่า

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม memory cache
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // จำกัด size (ถ้าใช้ SetSize)
    options.CompactionPercentage = 0.25; // ลบ 25% เมื่อเต็ม
    options.ExpirationScanFrequency = TimeSpan.FromMinutes(5); // scan ทุก 5 นาที
});
```

### การใช้งาน IMemoryCache

```csharp
// Services/ProductCacheService.cs
using Microsoft.Extensions.Caching.Memory;

namespace CachingDemo.Services;

public class ProductCacheService
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<ProductCacheService> _logger;

    // Cache key constants
    private static class CacheKeys
    {
        public const string AllProducts = "all_products";
        public static string Product(int id) => $"product_{id}";
        public static string CategoryProducts(int categoryId) => $"category_products_{categoryId}";
    }

    public ProductCacheService(IMemoryCache cache, ILogger<ProductCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    // GetOrCreate pattern - วิธีที่แนะนำ
    public async Task<List<Product>> GetAllProductsAsync()
    {
        return await _cache.GetOrCreateAsync(
            CacheKeys.AllProducts,
            async entry =>
            {
                entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
                entry.SlidingExpiration = TimeSpan.FromMinutes(10);
                entry.Priority = CacheItemPriority.Normal;

                _logger.LogInformation("Cache miss - loading products from database");
                return await LoadProductsFromDbAsync();
            }) ?? new List<Product>();
    }

    // Get แล้วค่อย Set แยก
    public async Task<Product?> GetProductAsync(int id)
    {
        if (_cache.TryGetValue(CacheKeys.Product(id), out Product? product))
        {
            _logger.LogDebug("Cache hit for product {Id}", id);
            return product;
        }

        _logger.LogDebug("Cache miss for product {Id}", id);
        product = await LoadProductFromDbAsync(id);

        if (product != null)
        {
            var options = new MemoryCacheEntryOptions
            {
                AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(1),
                SlidingExpiration = TimeSpan.FromMinutes(15),
                Priority = CacheItemPriority.High,
                Size = 1 // สำหรับ SizeLimit
            };

            // Register callback เมื่อ cache entry ถูกลบ
            options.RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                _logger.LogDebug("Cache evicted: {Key}, Reason: {Reason}", key, reason);
            });

            _cache.Set(CacheKeys.Product(id), product, options);
        }

        return product;
    }

    // Invalidate cache
    public void InvalidateProduct(int id)
    {
        _cache.Remove(CacheKeys.Product(id));
        _cache.Remove(CacheKeys.AllProducts);
        _logger.LogInformation("Cache invalidated for product {Id}", id);
    }

    public void InvalidateAll()
    {
        _cache.Remove(CacheKeys.AllProducts);
        // ต้องลบแต่ละ key แยก หรือใช้ CancellationChangeToken
    }

    // จำลอง database loading
    private async Task<List<Product>> LoadProductsFromDbAsync()
    {
        await Task.Delay(100); // จำลอง latency
        return Enumerable.Range(1, 10).Select(i => new Product
        {
            Id = i,
            Name = $"Product {i}",
            Price = i * 99.99m
        }).ToList();
    }

    private async Task<Product?> LoadProductFromDbAsync(int id)
    {
        await Task.Delay(50);
        return id <= 10 ? new Product { Id = id, Name = $"Product {id}", Price = id * 99.99m } : null;
    }
}
```

### Cache Dependencies และ Expiration Tokens

```csharp
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Primitives;

// สร้าง CancellationChangeToken สำหรับ group invalidation
public class CacheInvalidationService
{
    private CancellationTokenSource _productsCts = new();

    public IChangeToken GetProductsChangeToken()
    {
        return new CancellationChangeToken(_productsCts.Token);
    }

    public void InvalidateProducts()
    {
        var cts = Interlocked.Exchange(ref _productsCts, new CancellationTokenSource());
        cts.Cancel();
        cts.Dispose();
    }
}

// ใช้งาน
public async Task<List<Product>> GetProductsWithTokenAsync()
{
    return await _cache.GetOrCreateAsync(
        "products_with_token",
        async entry =>
        {
            // ผูก token กับ cache entry
            entry.AddExpirationToken(_invalidationService.GetProductsChangeToken());
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);

            return await LoadProductsFromDbAsync();
        }) ?? [];
}
```

---

## 2. Distributed Cache (IDistributedCache)

**IDistributedCache** เหมาะสำหรับ multi-server (load balanced) environment

### SQL Server Distributed Cache

```bash
dotnet add package Microsoft.Extensions.Caching.SqlServer
```

```csharp
// Program.cs
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("DefaultConnection");
    options.SchemaName = "dbo";
    options.TableName = "CacheTable";
});

// สร้างตาราง
// dotnet sql-cache create "Connection String" dbo CacheTable
```

### Redis Distributed Cache

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "CachingDemo:";
});
```

### การใช้งาน IDistributedCache

```csharp
// Services/DistributedCacheService.cs
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

namespace CachingDemo.Services;

public class DistributedCacheService
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<DistributedCacheService> _logger;

    public DistributedCacheService(
        IDistributedCache cache,
        ILogger<DistributedCacheService> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    // บันทึก object ลง cache (serialize เป็น JSON)
    public async Task SetAsync<T>(string key, T value, TimeSpan? absoluteExpiration = null)
    {
        var json = JsonSerializer.Serialize(value);
        var bytes = System.Text.Encoding.UTF8.GetBytes(json);

        var options = new DistributedCacheEntryOptions();

        if (absoluteExpiration.HasValue)
            options.AbsoluteExpirationRelativeToNow = absoluteExpiration;
        else
            options.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);

        await _cache.SetAsync(key, bytes, options);
        _logger.LogDebug("Cache set: {Key}", key);
    }

    // ดึง object จาก cache
    public async Task<T?> GetAsync<T>(string key)
    {
        var bytes = await _cache.GetAsync(key);

        if (bytes == null)
        {
            _logger.LogDebug("Cache miss: {Key}", key);
            return default;
        }

        _logger.LogDebug("Cache hit: {Key}", key);
        var json = System.Text.Encoding.UTF8.GetString(bytes);
        return JsonSerializer.Deserialize<T>(json);
    }

    // GetOrCreate pattern
    public async Task<T?> GetOrCreateAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan? expiration = null)
    {
        var cached = await GetAsync<T>(key);
        if (cached != null) return cached;

        var value = await factory();

        if (value != null)
            await SetAsync(key, value, expiration);

        return value;
    }

    // ลบ cache
    public async Task RemoveAsync(string key)
    {
        await _cache.RemoveAsync(key);
        _logger.LogDebug("Cache removed: {Key}", key);
    }

    // Refresh sliding expiration
    public async Task RefreshAsync(string key)
    {
        await _cache.RefreshAsync(key);
    }
}
```

---

## 3. Redis Cache - ขั้นสูง

### StackExchange.Redis โดยตรง

```bash
dotnet add package StackExchange.Redis
```

```csharp
// Services/RedisService.cs
using StackExchange.Redis;
using System.Text.Json;

namespace CachingDemo.Services;

public class RedisService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IDatabase _db;

    public RedisService(IConnectionMultiplexer redis)
    {
        _redis = redis;
        _db = redis.GetDatabase();
    }

    // String operations
    public async Task<bool> SetStringAsync(string key, string value, TimeSpan? expiry = null)
    {
        return await _db.StringSetAsync(key, value, expiry);
    }

    public async Task<string?> GetStringAsync(string key)
    {
        return await _db.StringGetAsync(key);
    }

    // Object operations (JSON)
    public async Task<bool> SetObjectAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var json = JsonSerializer.Serialize(value);
        return await _db.StringSetAsync(key, json, expiry);
    }

    public async Task<T?> GetObjectAsync<T>(string key)
    {
        var value = await _db.StringGetAsync(key);
        if (!value.HasValue) return default;
        return JsonSerializer.Deserialize<T>(value!);
    }

    // Increment/Decrement (สำหรับ counter)
    public async Task<long> IncrementAsync(string key, long amount = 1)
    {
        return await _db.StringIncrementAsync(key, amount);
    }

    // List operations
    public async Task PushToListAsync(string key, string value)
    {
        await _db.ListRightPushAsync(key, value);
    }

    public async Task<List<string>> GetListAsync(string key)
    {
        var values = await _db.ListRangeAsync(key);
        return values.Select(v => v.ToString()).ToList();
    }

    // Hash operations (เหมือน Dictionary)
    public async Task SetHashFieldAsync(string key, string field, string value)
    {
        await _db.HashSetAsync(key, field, value);
    }

    public async Task<string?> GetHashFieldAsync(string key, string field)
    {
        return await _db.HashGetAsync(key, field);
    }

    public async Task<Dictionary<string, string>> GetAllHashAsync(string key)
    {
        var entries = await _db.HashGetAllAsync(key);
        return entries.ToDictionary(e => e.Name.ToString(), e => e.Value.ToString());
    }

    // Set operations
    public async Task AddToSetAsync(string key, string value)
    {
        await _db.SetAddAsync(key, value);
    }

    public async Task<bool> IsInSetAsync(string key, string value)
    {
        return await _db.SetContainsAsync(key, value);
    }

    // Pub/Sub
    public async Task PublishAsync(string channel, string message)
    {
        var pub = _redis.GetSubscriber();
        await pub.PublishAsync(channel, message);
    }

    public async Task SubscribeAsync(string channel, Action<string, string> handler)
    {
        var sub = _redis.GetSubscriber();
        await sub.SubscribeAsync(channel, (chan, msg) =>
            handler(chan!, msg!));
    }

    // Transaction
    public async Task<bool> ExecuteTransactionAsync(Action<ITransaction> transactionAction)
    {
        var transaction = _db.CreateTransaction();
        transactionAction(transaction);
        return await transaction.ExecuteAsync();
    }

    // Check if key exists
    public async Task<bool> ExistsAsync(string key)
    {
        return await _db.KeyExistsAsync(key);
    }

    // Delete
    public async Task<bool> DeleteAsync(string key)
    {
        return await _db.KeyDeleteAsync(key);
    }

    // Set TTL
    public async Task<bool> ExpireAsync(string key, TimeSpan expiry)
    {
        return await _db.KeyExpireAsync(key, expiry);
    }
}

// ลงทะเบียนใน Program.cs
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));
builder.Services.AddSingleton<RedisService>();
```

---

## 4. Response Caching

**Response Caching** cache HTTP responses เพื่อลด load บน server

```csharp
// Program.cs
builder.Services.AddResponseCaching(options =>
{
    options.MaximumBodySize = 1024; // KB
    options.UseCaseSensitivePaths = false;
});

var app = builder.Build();

app.UseResponseCaching();
```

### ใช้ใน Controller

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;

namespace CachingDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // Cache response 60 วินาที (public cache)
    [HttpGet]
    [ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any, VaryByQueryKeys = new[] { "*" })]
    public async Task<ActionResult<List<Product>>> GetAll()
    {
        // ...
        return Ok(new List<Product>());
    }

    // Cache 30 วินาที โดย vary ตาม id
    [HttpGet("{id}")]
    [ResponseCache(Duration = 30, VaryByHeader = "Accept-Language")]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        // ...
        return Ok(new Product());
    }

    // ไม่ cache (สำหรับ data ที่เปลี่ยนบ่อย)
    [HttpGet("realtime")]
    [ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
    public ActionResult<string> GetRealtime()
    {
        return Ok(DateTime.Now.ToString());
    }
}
```

### Output Caching (ASP.NET Core 7+)

```csharp
// Program.cs
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder =>
        builder.Expire(TimeSpan.FromSeconds(10)));

    options.AddPolicy("Products", builder =>
        builder.Expire(TimeSpan.FromMinutes(5))
               .Tag("products"));

    options.AddPolicy("UserSpecific", builder =>
        builder.VaryByValue(httpContext =>
            new KeyValuePair<string, string>("userId",
                httpContext.User.FindFirst("sub")?.Value ?? "anonymous"))
        .Expire(TimeSpan.FromMinutes(1)));
});

app.UseOutputCache();

// ใน Controller
[HttpGet]
[OutputCache(PolicyName = "Products")]
public async Task<IActionResult> GetProducts() { ... }

// Invalidate ด้วย tag
app.MapPost("/invalidate-products", async (IOutputCacheStore cache, CancellationToken ct) =>
{
    await cache.EvictByTagAsync("products", ct);
    return Results.Ok("Products cache invalidated");
});
```

---

## 5. Cache Invalidation Strategies

### Strategy 1: Time-based Expiration

```csharp
// Absolute expiration - หมดอายุตามเวลาที่กำหนด
entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);

// Sliding expiration - หมดอายุถ้าไม่มีการเข้าถึง
entry.SlidingExpiration = TimeSpan.FromMinutes(10);

// ใช้ทั้งสองอย่างร่วมกัน (take whichever expires first)
entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1);
entry.SlidingExpiration = TimeSpan.FromMinutes(15);
```

### Strategy 2: Event-based Invalidation

```csharp
// Services/CacheInvalidationService.cs
namespace CachingDemo.Services;

public interface ICacheInvalidator
{
    Task InvalidateProductAsync(int productId);
    Task InvalidateCategoryAsync(int categoryId);
    Task InvalidateAllProductsAsync();
}

public class CacheInvalidationService : ICacheInvalidator
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    private readonly ILogger<CacheInvalidationService> _logger;

    public CacheInvalidationService(
        IMemoryCache memoryCache,
        IDistributedCache distributedCache,
        ILogger<CacheInvalidationService> logger)
    {
        _memoryCache = memoryCache;
        _distributedCache = distributedCache;
        _logger = logger;
    }

    public async Task InvalidateProductAsync(int productId)
    {
        var keys = new[]
        {
            $"product_{productId}",
            "all_products",
            $"product_details_{productId}"
        };

        foreach (var key in keys)
        {
            _memoryCache.Remove(key);
            await _distributedCache.RemoveAsync(key);
        }

        _logger.LogInformation("Invalidated cache for product {Id}", productId);
    }

    public async Task InvalidateCategoryAsync(int categoryId)
    {
        _memoryCache.Remove($"category_{categoryId}");
        _memoryCache.Remove($"category_products_{categoryId}");
        await _distributedCache.RemoveAsync($"category_{categoryId}");
        await _distributedCache.RemoveAsync($"category_products_{categoryId}");
    }

    public async Task InvalidateAllProductsAsync()
    {
        _memoryCache.Remove("all_products");
        await _distributedCache.RemoveAsync("all_products");
        _logger.LogInformation("Invalidated all products cache");
    }
}
```

### Strategy 3: Write-Through Cache

```csharp
// Services/WriteThoughProductService.cs
namespace CachingDemo.Services;

public class WriteThoughProductService
{
    private readonly IProductRepository _repository;
    private readonly IMemoryCache _cache;

    public WriteThoughProductService(IProductRepository repository, IMemoryCache cache)
    {
        _repository = repository;
        _cache = cache;
    }

    // อัปเดททั้ง database และ cache พร้อมกัน
    public async Task<Product> UpdateProductAsync(Product product)
    {
        var updated = await _repository.UpdateAsync(product);

        // อัปเดท cache ทันที
        _cache.Set($"product_{product.Id}", updated,
            TimeSpan.FromMinutes(30));

        // ลบ cache ที่เกี่ยวข้อง
        _cache.Remove("all_products");

        return updated;
    }
}
```

---

## โปรแกรมตัวอย่าง: Product Catalog Caching

ระบบ product catalog ที่ใช้ multi-layer caching เพื่อประสิทธิภาพสูงสุด

### โครงสร้าง

```
ProductCatalog/
├── Models/
│   ├── Product.cs
│   └── Category.cs
├── Data/
│   └── AppDbContext.cs
├── Services/
│   ├── ProductService.cs
│   └── CacheService.cs
├── Controllers/
│   └── ProductsController.cs
└── Program.cs
```

### Models

```csharp
// Models/Product.cs
namespace ProductCatalog.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public int CategoryId { get; set; }
    public Category? Category { get; set; }
    public DateTime UpdatedAt { get; set; }
    public bool IsActive { get; set; } = true;
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public List<Product> Products { get; set; } = new();
}
```

### CacheService - Multi-layer Cache

```csharp
// Services/CacheService.cs
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

namespace ProductCatalog.Services;

public class MultiLayerCacheService
{
    private readonly IMemoryCache _l1Cache;  // Layer 1: In-memory (ไว, ขนาดน้อย)
    private readonly IDistributedCache _l2Cache;  // Layer 2: Redis (ช้ากว่า, ขนาดใหญ่)
    private readonly ILogger<MultiLayerCacheService> _logger;

    public MultiLayerCacheService(
        IMemoryCache l1Cache,
        IDistributedCache l2Cache,
        ILogger<MultiLayerCacheService> logger)
    {
        _l1Cache = l1Cache;
        _l2Cache = l2Cache;
        _logger = logger;
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        // ตรวจสอบ L1 cache ก่อน
        if (_l1Cache.TryGetValue(key, out T? l1Value))
        {
            _logger.LogDebug("L1 Cache HIT: {Key}", key);
            return l1Value;
        }

        // ตรวจสอบ L2 cache
        var l2Bytes = await _l2Cache.GetAsync(key);
        if (l2Bytes != null)
        {
            _logger.LogDebug("L2 Cache HIT: {Key}", key);
            var l2Value = JsonSerializer.Deserialize<T>(l2Bytes);

            // เพิ่มกลับ L1 cache
            _l1Cache.Set(key, l2Value, TimeSpan.FromMinutes(5));

            return l2Value;
        }

        _logger.LogDebug("Cache MISS: {Key}", key);
        return default;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan l1Expiry, TimeSpan l2Expiry)
    {
        // บันทึกทั้ง L1 และ L2
        _l1Cache.Set(key, value, l1Expiry);

        var bytes = JsonSerializer.SerializeToUtf8Bytes(value);
        await _l2Cache.SetAsync(key, bytes, new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = l2Expiry
        });
    }

    public async Task RemoveAsync(string key)
    {
        _l1Cache.Remove(key);
        await _l2Cache.RemoveAsync(key);
    }

    public async Task<T?> GetOrCreateAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan l1Expiry,
        TimeSpan l2Expiry)
    {
        var cached = await GetAsync<T>(key);
        if (cached != null) return cached;

        var value = await factory();
        if (value != null)
            await SetAsync(key, value, l1Expiry, l2Expiry);

        return value;
    }
}
```

### ProductService

```csharp
// Services/ProductService.cs
using Microsoft.EntityFrameworkCore;
using ProductCatalog.Data;
using ProductCatalog.Models;

namespace ProductCatalog.Services;

public class ProductService
{
    private readonly AppDbContext _db;
    private readonly MultiLayerCacheService _cache;
    private readonly ILogger<ProductService> _logger;

    private static readonly TimeSpan L1Expiry = TimeSpan.FromMinutes(5);
    private static readonly TimeSpan L2Expiry = TimeSpan.FromMinutes(30);

    public ProductService(
        AppDbContext db,
        MultiLayerCacheService cache,
        ILogger<ProductService> logger)
    {
        _db = db;
        _cache = cache;
        _logger = logger;
    }

    public async Task<List<Product>> GetAllProductsAsync()
    {
        return await _cache.GetOrCreateAsync(
            "products:all",
            async () =>
            {
                _logger.LogInformation("Loading all products from database");
                return await _db.Products
                    .Include(p => p.Category)
                    .Where(p => p.IsActive)
                    .OrderBy(p => p.Name)
                    .ToListAsync();
            },
            L1Expiry,
            L2Expiry) ?? [];
    }

    public async Task<Product?> GetProductByIdAsync(int id)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async () =>
            {
                return await _db.Products
                    .Include(p => p.Category)
                    .FirstOrDefaultAsync(p => p.Id == id && p.IsActive);
            },
            TimeSpan.FromMinutes(10),
            TimeSpan.FromHours(1));
    }

    public async Task<List<Product>> GetProductsByCategoryAsync(int categoryId)
    {
        return await _cache.GetOrCreateAsync(
            $"products:category:{categoryId}",
            async () =>
            {
                return await _db.Products
                    .Include(p => p.Category)
                    .Where(p => p.CategoryId == categoryId && p.IsActive)
                    .OrderBy(p => p.Name)
                    .ToListAsync();
            },
            L1Expiry,
            L2Expiry) ?? [];
    }

    public async Task<Product> CreateProductAsync(Product product)
    {
        product.UpdatedAt = DateTime.UtcNow;
        _db.Products.Add(product);
        await _db.SaveChangesAsync();

        // Invalidate related caches
        await InvalidateCachesAsync(product);

        return product;
    }

    public async Task<Product?> UpdateProductAsync(Product product)
    {
        var existing = await _db.Products.FindAsync(product.Id);
        if (existing == null) return null;

        existing.Name = product.Name;
        existing.Description = product.Description;
        existing.Price = product.Price;
        existing.Stock = product.Stock;
        existing.CategoryId = product.CategoryId;
        existing.UpdatedAt = DateTime.UtcNow;

        await _db.SaveChangesAsync();

        // Invalidate caches
        await InvalidateCachesAsync(existing);

        return existing;
    }

    public async Task<bool> DeleteProductAsync(int id)
    {
        var product = await _db.Products.FindAsync(id);
        if (product == null) return false;

        product.IsActive = false;
        await _db.SaveChangesAsync();

        await InvalidateCachesAsync(product);

        return true;
    }

    private async Task InvalidateCachesAsync(Product product)
    {
        await _cache.RemoveAsync($"product:{product.Id}");
        await _cache.RemoveAsync("products:all");
        await _cache.RemoveAsync($"products:category:{product.CategoryId}");

        _logger.LogInformation(
            "Cache invalidated for product {Id} and related keys",
            product.Id);
    }
}
```

### Controller

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;
using ProductCatalog.Models;
using ProductCatalog.Services;

namespace ProductCatalog.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ProductService _productService;

    public ProductsController(ProductService productService)
    {
        _productService = productService;
    }

    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetAll()
    {
        var products = await _productService.GetAllProductsAsync();
        return Ok(products);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        var product = await _productService.GetProductByIdAsync(id);
        if (product == null) return NotFound();
        return Ok(product);
    }

    [HttpGet("category/{categoryId}")]
    public async Task<ActionResult<List<Product>>> GetByCategory(int categoryId)
    {
        var products = await _productService.GetProductsByCategoryAsync(categoryId);
        return Ok(products);
    }

    [HttpPost]
    public async Task<ActionResult<Product>> Create(Product product)
    {
        var created = await _productService.CreateProductAsync(product);
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    [HttpPut("{id}")]
    public async Task<ActionResult<Product>> Update(int id, Product product)
    {
        if (id != product.Id) return BadRequest();
        var updated = await _productService.UpdateProductAsync(product);
        if (updated == null) return NotFound();
        return Ok(updated);
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _productService.DeleteProductAsync(id);
        if (!deleted) return NotFound();
        return NoContent();
    }
}
```

### Program.cs

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;
using ProductCatalog.Data;
using ProductCatalog.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=products.db"));

// Layer 1: In-memory cache
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 500;
});

// Layer 2: Redis distributed cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis")
        ?? "localhost:6379";
    options.InstanceName = "ProductCatalog:";
});

builder.Services.AddSingleton<MultiLayerCacheService>();
builder.Services.AddScoped<ProductService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();

app.Run();
```

---

## Exercises

### Exercise 1: Cache Statistics
สร้าง endpoint `/api/cache/stats` ที่แสดง cache hit rate, miss rate, และจำนวน keys ปัจจุบัน

### Exercise 2: Distributed Cache Serialization
ปรับปรุง DistributedCacheService ให้ support custom serializer (เช่น MessagePack แทน JSON)

### Exercise 3: Cache Warming
สร้าง background service ที่ pre-load cache เมื่อ application เริ่มต้น

### Exercise 4: Circuit Breaker
เพิ่ม circuit breaker ใน MultiLayerCacheService เมื่อ Redis ไม่ available ให้ fallback ไปใช้แค่ L1

### Exercise 5: Cache Tags
สร้างระบบ tag-based cache invalidation เพื่อลบ cache หลายๆ key พร้อมกัน

---

## สรุป

- **IMemoryCache** เหมาะสำหรับ single-server, เร็ว, แต่ข้อมูลหายเมื่อ restart
- **IDistributedCache** เหมาะสำหรับ multi-server, ข้อมูลคงอยู่, แต่ช้ากว่า
- **Redis** เป็น in-memory database ที่ powerful, รองรับ data structures หลายแบบ
- **Response/Output Caching** ช่วยลด load โดย cache HTTP responses
- ใช้ **multi-layer caching** เพื่อสมดุลระหว่างความเร็วและ consistency
- ออกแบบ **cache invalidation** ให้ดีตั้งแต่ต้น

---

## Part ถัดไป

**Part 064: Logging ใน ASP.NET Core** - เรียนรู้การ logging ด้วย ILogger, Serilog, และ structured logging

---

*Part 063/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
