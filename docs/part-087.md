# Part 087: Performance Optimization

## เนื้อหาใน Part นี้
- Profiling .NET apps
- Memory optimization
- CPU optimization
- Database query optimization
- Caching strategies
- Async/parallel optimization
- Benchmark.NET
- โปรแกรมตัวอย่าง: Optimize slow API

---

## 1. Profiling .NET Apps

ก่อน optimize ต้อง measure ก่อนเสมอ - "Don't optimize what you haven't measured"

### เครื่องมือ Profiling

```bash
# dotnet-trace - CPU/memory profiling
dotnet tool install -g dotnet-trace
dotnet-trace collect --process-id <pid> --duration 00:00:10
dotnet-trace analyze myapp.nettrace

# dotnet-counters - real-time metrics
dotnet tool install -g dotnet-counters
dotnet-counters monitor --process-id <pid> System.Runtime

# dotnet-dump - memory dump analysis
dotnet tool install -g dotnet-dump
dotnet-dump collect --process-id <pid>
dotnet-dump analyze core_20231001_120000.dmp
```

### Profiling ด้วย Visual Studio Diagnostic Tools

```csharp
// เพิ่ม EventSource สำหรับ custom metrics
using System.Diagnostics.Tracing;

[EventSource(Name = "MyApp.OrderService")]
public sealed class OrderServiceEventSource : EventSource
{
    public static readonly OrderServiceEventSource Log = new();
    
    [Event(1, Level = EventLevel.Informational)]
    public void OrderCreated(string orderId, decimal amount)
    {
        WriteEvent(1, orderId, amount);
    }
    
    [Event(2, Level = EventLevel.Error)]
    public void OrderFailed(string orderId, string error)
    {
        WriteEvent(2, orderId, error);
    }
}
```

### Memory Profiling

```csharp
// ดู memory allocation patterns
using var memoryUsage = new MemoryTracker();

// ดู GC collections
public class GCMonitor
{
    public static void PrintGCStats()
    {
        Console.WriteLine($"Gen0: {GC.CollectionCount(0)}");
        Console.WriteLine($"Gen1: {GC.CollectionCount(1)}");
        Console.WriteLine($"Gen2: {GC.CollectionCount(2)}");
        Console.WriteLine($"Total Memory: {GC.GetTotalMemory(false):N0} bytes");
        Console.WriteLine($"Heap Info: {GC.GetGCMemoryInfo().HeapSizeBytes:N0} bytes");
    }
}
```

---

## 2. Memory Optimization

### Span<T> และ Memory<T>

```csharp
// ไม่ดี - allocate ใหม่ทุกครั้ง
public string ProcessName(string fullName)
{
    var parts = fullName.Split(' ');  // allocation
    return string.Join(", ", parts.Reverse());  // more allocations
}

// ดี - ใช้ Span เพื่อลด allocation
public bool TryParseFirstName(ReadOnlySpan<char> fullName, out ReadOnlySpan<char> firstName)
{
    var spaceIndex = fullName.IndexOf(' ');
    if (spaceIndex == -1)
    {
        firstName = fullName;
        return true;
    }
    firstName = fullName[..spaceIndex];
    return true;
}

// การใช้งาน
var name = "John Doe";
if (TryParseFirstName(name.AsSpan(), out var firstName))
{
    Console.WriteLine(firstName.ToString());  // "John" - ไม่มี string allocation ใหม่
}
```

### ArrayPool<T>

```csharp
using System.Buffers;

// ไม่ดี - allocate array ใหม่ทุกครั้ง
public byte[] ConvertToBytes(string data)
{
    return Encoding.UTF8.GetBytes(data);  // new byte[] ทุกครั้ง
}

// ดี - reuse buffer จาก pool
public void ProcessData(string data, Stream output)
{
    var pool = ArrayPool<byte>.Shared;
    var bufferSize = Encoding.UTF8.GetMaxByteCount(data.Length);
    var buffer = pool.Rent(bufferSize);
    
    try
    {
        var bytesWritten = Encoding.UTF8.GetBytes(data, buffer);
        output.Write(buffer, 0, bytesWritten);
    }
    finally
    {
        pool.Return(buffer);  // คืน buffer กลับ pool
    }
}
```

### Object Pooling

```csharp
using Microsoft.Extensions.ObjectPool;

// สร้าง pooled object
public class ExpensiveObject : IResettable
{
    private readonly StringBuilder _builder = new();
    
    public string BuildReport(IEnumerable<string> items)
    {
        _builder.Clear();
        foreach (var item in items)
        {
            _builder.AppendLine(item);
        }
        return _builder.ToString();
    }
    
    public bool TryReset()
    {
        _builder.Clear();
        return true;
    }
}

// Registration
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton<ObjectPool<ExpensiveObject>>(serviceProvider =>
{
    var provider = serviceProvider.GetRequiredService<ObjectPoolProvider>();
    return provider.Create<ExpensiveObject>();
});

// ใช้งาน
public class ReportService
{
    private readonly ObjectPool<ExpensiveObject> _pool;
    
    public ReportService(ObjectPool<ExpensiveObject> pool)
    {
        _pool = pool;
    }
    
    public string GenerateReport(IEnumerable<string> items)
    {
        var obj = _pool.Get();
        try
        {
            return obj.BuildReport(items);
        }
        finally
        {
            _pool.Return(obj);
        }
    }
}
```

### String Optimization

```csharp
// ไม่ดี - string concatenation ใน loop
public string BuildCsv(List<string[]> rows)
{
    var result = "";
    foreach (var row in rows)
    {
        result += string.Join(",", row) + "\n";  // O(n²) allocations
    }
    return result;
}

// ดี - StringBuilder
public string BuildCsvWithStringBuilder(List<string[]> rows)
{
    var sb = new StringBuilder(rows.Count * 50);  // pre-size hint
    foreach (var row in rows)
    {
        for (int i = 0; i < row.Length; i++)
        {
            if (i > 0) sb.Append(',');
            sb.Append(row[i]);
        }
        sb.AppendLine();
    }
    return sb.ToString();
}

// ดีกว่า - ใช้ interpolated string handler (C# 10+)
public static string Format(ref DefaultInterpolatedStringHandler handler)
{
    return handler.ToStringAndClear();
}

// String interning สำหรับ strings ที่ใช้บ่อย
var status = string.Intern("active");  // reuse existing string instance
```

### Struct vs Class

```csharp
// Class - allocated บน heap, GC pressure สูง
public class Point
{
    public double X { get; set; }
    public double Y { get; set; }
}

// Struct - allocated บน stack (ถ้าไม่ boxed), ไม่มี GC pressure
public readonly struct Point3D
{
    public readonly double X;
    public readonly double Y;
    public readonly double Z;
    
    public Point3D(double x, double y, double z) => (X, Y, Z) = (x, y, z);
    
    public double DistanceTo(Point3D other)
    {
        var dx = X - other.X;
        var dy = Y - other.Y;
        var dz = Z - other.Z;
        return Math.Sqrt(dx * dx + dy * dy + dz * dz);
    }
}

// Record struct (C# 10+) - immutable struct ที่สะดวก
public readonly record struct Money(decimal Amount, string Currency);

// ใช้งาน
Span<Point3D> points = stackalloc Point3D[100];  // stack allocation!
for (int i = 0; i < points.Length; i++)
{
    points[i] = new Point3D(i, i * 2, i * 3);
}
```

---

## 3. CPU Optimization

### LINQ Optimization

```csharp
// ไม่ดี - LINQ ทำงานหลายรอบ
var expensiveProducts = products
    .Where(p => p.Price > 100)
    .OrderBy(p => p.Price)
    .ToList();

var cheapProducts = products
    .Where(p => p.Price <= 100)
    .OrderBy(p => p.Price)
    .ToList();

// ดี - group ทีเดียว
var groupedProducts = products
    .GroupBy(p => p.Price > 100 ? "expensive" : "cheap")
    .ToDictionary(g => g.Key, g => g.OrderBy(p => p.Price).ToList());

// ดีมาก - ใช้ for loop แทน LINQ สำหรับ performance critical path
var expensiveList = new List<Product>(products.Count / 2);
var cheapList = new List<Product>(products.Count / 2);

for (int i = 0; i < products.Count; i++)
{
    var product = products[i];
    if (product.Price > 100)
        expensiveList.Add(product);
    else
        cheapList.Add(product);
}

// ใช้ CollectionsMarshal สำหรับ access array behind List
var span = CollectionsMarshal.AsSpan(expensiveList);
for (int i = 0; i < span.Length; i++)
{
    ref var product = ref span[i];
    // direct reference - no copy
}
```

### Parallel Processing

```csharp
// Parallel.For
var results = new ConcurrentBag<ProcessedItem>();

Parallel.For(0, items.Count, new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount
}, i =>
{
    var result = ProcessItem(items[i]);
    results.Add(result);
});

// PLINQ
var processedItems = items
    .AsParallel()
    .WithDegreeOfParallelism(4)
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .Select(ProcessItem)
    .ToList();

// Parallel.ForEachAsync (async)
await Parallel.ForEachAsync(
    items,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    async (item, ct) =>
    {
        await ProcessItemAsync(item, ct);
    });
```

### Vectorization (SIMD)

```csharp
using System.Numerics;
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

// Auto-vectorizable loop
public static float SumArray(float[] values)
{
    float sum = 0;
    for (int i = 0; i < values.Length; i++)
        sum += values[i];
    return sum;
}

// Explicit SIMD สำหรับ maximum performance
public static float SumArraySIMD(float[] values)
{
    var sum = Vector<float>.Zero;
    int i = 0;
    
    // Process 8 floats at a time (256-bit SIMD)
    int vectorSize = Vector<float>.Count;
    for (; i <= values.Length - vectorSize; i += vectorSize)
    {
        var vec = new Vector<float>(values, i);
        sum += vec;
    }
    
    float result = Vector.Sum(sum);
    
    // Handle remaining elements
    for (; i < values.Length; i++)
        result += values[i];
    
    return result;
}
```

---

## 4. Database Query Optimization

### N+1 Problem

```csharp
// ไม่ดี - N+1 queries
public async Task<List<OrderDto>> GetOrdersWithProductsAsync()
{
    var orders = await dbContext.Orders.ToListAsync();  // 1 query
    
    var dtos = new List<OrderDto>();
    foreach (var order in orders)
    {
        // N queries!
        var product = await dbContext.Products.FindAsync(order.ProductId);
        dtos.Add(new OrderDto(order.Id, product?.Name ?? "Unknown"));
    }
    
    return dtos;
}

// ดี - Eager loading
public async Task<List<OrderDto>> GetOrdersWithProductsAsync_Better()
{
    return await dbContext.Orders
        .Include(o => o.Product)  // 1 query with JOIN
        .Select(o => new OrderDto(o.Id, o.Product.Name))
        .ToListAsync();
}

// ดีกว่า - Projection ตรงๆ
public async Task<List<OrderDto>> GetOrdersWithProductsAsync_Best()
{
    return await dbContext.Orders
        .Select(o => new OrderDto(
            o.Id,
            o.Product.Name))  // EF แปลงเป็น SELECT ที่ดีที่สุด
        .ToListAsync();
}
```

### Query Optimization

```csharp
// Compiled Queries - avoid re-parsing EF queries
private static readonly Func<AppDbContext, Guid, Task<Order?>> GetOrderByIdQuery =
    EF.CompileAsyncQuery((AppDbContext db, Guid id) =>
        db.Orders.FirstOrDefault(o => o.Id == id));

public async Task<Order?> GetOrderAsync(Guid id)
{
    return await GetOrderByIdQuery(dbContext, id);
}

// AsNoTracking - สำหรับ read-only queries
public async Task<List<ProductDto>> GetProductsAsync()
{
    return await dbContext.Products
        .AsNoTracking()  // ไม่ track changes = เร็วกว่า
        .Where(p => p.IsActive)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .ToListAsync();
}

// Split queries - สำหรับ collection navigation
public async Task<Order?> GetOrderWithDetailsAsync(Guid id)
{
    return await dbContext.Orders
        .AsSplitQuery()  // แยก query สำหรับ collections
        .Include(o => o.Items)
        .Include(o => o.StatusHistory)
        .FirstOrDefaultAsync(o => o.Id == id);
}

// Pagination ที่ถูกต้อง
public async Task<PagedResult<Product>> GetProductsPagedAsync(int page, int pageSize)
{
    var query = dbContext.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Name);
    
    var total = await query.CountAsync();
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductDto(p.Id, p.Name, p.Price))
        .ToListAsync();
    
    return new PagedResult<Product>(items, total, page, pageSize);
}
```

### Raw SQL สำหรับ Complex Queries

```csharp
// ใช้ Raw SQL สำหรับ performance-critical queries
public async Task<List<SalesReportDto>> GetSalesReportAsync(DateRange range)
{
    var sql = @"
        SELECT 
            p.Id AS ProductId,
            p.Name AS ProductName,
            SUM(oi.Quantity) AS TotalQuantity,
            SUM(oi.Quantity * oi.UnitPrice) AS TotalRevenue
        FROM OrderItems oi
        INNER JOIN Products p ON p.Id = oi.ProductId
        INNER JOIN Orders o ON o.Id = oi.OrderId
        WHERE o.CreatedAt BETWEEN @startDate AND @endDate
            AND o.Status = 'Completed'
        GROUP BY p.Id, p.Name
        ORDER BY TotalRevenue DESC";
    
    return await dbContext.Database
        .SqlQueryRaw<SalesReportDto>(sql,
            new SqlParameter("@startDate", range.Start),
            new SqlParameter("@endDate", range.End))
        .ToListAsync();
}
```

---

## 5. Caching Strategies

### In-Memory Cache

```csharp
// IMemoryCache
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repository;

    public async Task<Product?> GetProductAsync(Guid id)
    {
        var cacheKey = $"product:{id}";
        
        if (_cache.TryGetValue(cacheKey, out Product? cached))
            return cached;

        var product = await _repository.GetByIdAsync(id);
        
        if (product != null)
        {
            var cacheOptions = new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration = TimeSpan.FromMinutes(2),
                Priority = CacheItemPriority.Normal,
                Size = 1  // ต้อง set ถ้าใช้ SizeLimit
            };
            cacheOptions.RegisterPostEvictionCallback((key, value, reason, state) =>
            {
                Console.WriteLine($"Cache evicted: {key}, reason: {reason}");
            });
            
            _cache.Set(cacheKey, product, cacheOptions);
        }
        
        return product;
    }
}

// Cache Aside Pattern ที่ clean ขึ้น
public static class MemoryCacheExtensions
{
    public static async Task<T?> GetOrCreateAsync<T>(
        this IMemoryCache cache,
        string key,
        Func<Task<T?>> factory,
        TimeSpan expiry)
    {
        if (cache.TryGetValue(key, out T? value))
            return value;
        
        value = await factory();
        if (value != null)
            cache.Set(key, value, expiry);
        
        return value;
    }
}
```

### Distributed Cache (Redis)

```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class RedisCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly JsonSerializerOptions _jsonOptions;

    public RedisCacheService(IDistributedCache cache)
    {
        _cache = cache;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        };
    }

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var data = await _cache.GetStringAsync(key, ct);
        if (data == null) return default;
        return JsonSerializer.Deserialize<T>(data, _jsonOptions);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null, CancellationToken ct = default)
    {
        var options = new DistributedCacheEntryOptions();
        if (expiry.HasValue)
            options.AbsoluteExpirationRelativeToNow = expiry;
        
        var data = JsonSerializer.Serialize(value, _jsonOptions);
        await _cache.SetStringAsync(key, data, options, ct);
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
    {
        await _cache.RemoveAsync(key, ct);
    }
}
```

### Output Cache (ASP.NET Core)

```csharp
// Program.cs
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(10)));
    options.AddPolicy("Products", b => b
        .Tag("products")
        .Expire(TimeSpan.FromMinutes(5)));
});

app.UseOutputCache();

// Endpoint caching
app.MapGet("/api/products", async (IProductService service) =>
{
    var products = await service.GetAllAsync();
    return Results.Ok(products);
})
.CacheOutput("Products");

// Cache invalidation
app.MapPost("/api/products", async (
    CreateProductRequest request,
    IProductService service,
    IOutputCacheStore outputCacheStore) =>
{
    var product = await service.CreateAsync(request);
    await outputCacheStore.EvictByTagAsync("products", CancellationToken.None);
    return Results.Created($"/api/products/{product.Id}", product);
});
```

### Cache-Aside vs Write-Through

```csharp
// Cache-Aside Pattern (most common)
public async Task<Product?> GetProductAsync(Guid id)
{
    var cached = await _cache.GetAsync<Product>($"product:{id}");
    if (cached != null) return cached;
    
    var product = await _repo.GetByIdAsync(id);
    if (product != null)
        await _cache.SetAsync($"product:{id}", product, TimeSpan.FromMinutes(10));
    
    return product;
}

// Write-Through Pattern (consistency สูงกว่า)
public async Task<Product> UpdateProductAsync(Guid id, UpdateProductRequest request)
{
    var product = await _repo.UpdateAsync(id, request);
    
    // Update cache at the same time as DB
    await _cache.SetAsync($"product:{id}", product, TimeSpan.FromMinutes(10));
    
    return product;
}
```

---

## 6. Async/Parallel Optimization

### ไม่ Block Async

```csharp
// ไม่ดี - deadlock risk
public string GetData()
{
    return GetDataAsync().Result;  // DEADLOCK!
}

// ดี - async ตลอด
public async Task<string> GetDataAsync()
{
    return await _service.FetchAsync();
}

// ConfigureAwait(false) ใน library code
public async Task<string> LibraryMethodAsync()
{
    var data = await _httpClient.GetStringAsync(url)
        .ConfigureAwait(false);  // ไม่ต้องรอ sync context
    return Process(data);
}
```

### Parallel Async Operations

```csharp
// ไม่ดี - sequential แม้จะ async
public async Task<ProductDetails> GetProductDetailsAsync(Guid id)
{
    var product = await _productService.GetByIdAsync(id);
    var reviews = await _reviewService.GetByProductIdAsync(id);  // รอ product ก่อน
    var stock = await _inventoryService.GetStockAsync(id);  // รอ reviews ก่อน
    
    return new ProductDetails(product, reviews, stock);
}

// ดี - parallel
public async Task<ProductDetails> GetProductDetailsAsync_Parallel(Guid id)
{
    var productTask = _productService.GetByIdAsync(id);
    var reviewsTask = _reviewService.GetByProductIdAsync(id);
    var stockTask = _inventoryService.GetStockAsync(id);
    
    await Task.WhenAll(productTask, reviewsTask, stockTask);  // run in parallel
    
    return new ProductDetails(
        await productTask,
        await reviewsTask,
        await stockTask);
}

// ดีกว่า - ValueTask สำหรับ hot paths
public async ValueTask<Product?> GetCachedProductAsync(Guid id)
{
    if (_cache.TryGetValue(id, out Product? product))
        return product;  // sync path, no Task allocation
    
    return await FetchFromDbAsync(id);  // async path
}
```

### Channel สำหรับ Producer-Consumer

```csharp
using System.Threading.Channels;

// High-performance producer-consumer
public class DataProcessor
{
    private readonly Channel<DataItem> _channel;
    
    public DataProcessor()
    {
        _channel = Channel.CreateBounded<DataItem>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait
        });
    }
    
    public async Task ProduceAsync(IEnumerable<DataItem> items, CancellationToken ct)
    {
        var writer = _channel.Writer;
        foreach (var item in items)
        {
            await writer.WriteAsync(item, ct);
        }
        writer.Complete();
    }
    
    public async Task ConsumeAsync(CancellationToken ct)
    {
        var reader = _channel.Reader;
        await foreach (var item in reader.ReadAllAsync(ct))
        {
            await ProcessItemAsync(item);
        }
    }
    
    private Task ProcessItemAsync(DataItem item)
    {
        // Process item
        return Task.CompletedTask;
    }
}
```

---

## 7. Benchmark.NET

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(BenchmarkDotNet.Jobs.RuntimeMoniker.Net90)]
public class StringBenchmarks
{
    private const int ItemCount = 1000;
    private List<string> _items = null!;

    [GlobalSetup]
    public void Setup()
    {
        _items = Enumerable.Range(0, ItemCount)
            .Select(i => $"item-{i}")
            .ToList();
    }

    [Benchmark(Baseline = true)]
    public string StringConcatenation()
    {
        var result = "";
        foreach (var item in _items)
            result += item + ",";
        return result;
    }

    [Benchmark]
    public string StringJoin()
    {
        return string.Join(",", _items);
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var item in _items)
        {
            sb.Append(item);
            sb.Append(',');
        }
        return sb.ToString();
    }

    [Benchmark]
    public string StringBuilderWithCapacity()
    {
        var sb = new System.Text.StringBuilder(ItemCount * 8);
        foreach (var item in _items)
        {
            sb.Append(item);
            sb.Append(',');
        }
        return sb.ToString();
    }
}

// Benchmark ด้วย parameters
[MemoryDiagnoser]
public class CollectionBenchmarks
{
    [Params(10, 100, 1000, 10000)]
    public int N { get; set; }

    [Benchmark]
    public int ListSum()
    {
        var list = Enumerable.Range(0, N).ToList();
        return list.Sum();
    }

    [Benchmark]
    public int ArraySum()
    {
        var array = Enumerable.Range(0, N).ToArray();
        int sum = 0;
        for (int i = 0; i < array.Length; i++)
            sum += array[i];
        return sum;
    }

    [Benchmark]
    public int SpanSum()
    {
        var array = Enumerable.Range(0, N).ToArray();
        var span = array.AsSpan();
        int sum = 0;
        foreach (var item in span)
            sum += item;
        return sum;
    }
}

// Program.cs
BenchmarkRunner.Run<StringBenchmarks>();
BenchmarkRunner.Run<CollectionBenchmarks>();
```

---

## 8. โปรแกรมตัวอย่าง: Optimize Slow API

### ก่อน Optimize - Slow API

```csharp
// Version 1 - ช้า
[HttpGet("/api/products/slow")]
public async Task<IActionResult> GetProductsSlowly()
{
    // N+1 query
    var products = await dbContext.Products.ToListAsync();
    var result = new List<ProductWithReviewsDto>();
    
    foreach (var product in products)
    {
        // แต่ละ product = 1 query
        var reviews = await dbContext.Reviews
            .Where(r => r.ProductId == product.Id)
            .ToListAsync();
        
        // String concatenation ใน loop
        var reviewText = "";
        foreach (var review in reviews)
            reviewText += review.Text + " ";
        
        result.Add(new ProductWithReviewsDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            ReviewCount = reviews.Count,
            AverageRating = reviews.Any() ? reviews.Average(r => r.Rating) : 0,
            ReviewSummary = reviewText
        });
    }
    
    return Ok(result);
}
```

### หลัง Optimize - Fast API

```csharp
// Version 2 - เร็ว
[HttpGet("/api/products/fast")]
[OutputCache(Duration = 60, VaryByQueryKeys = new[] { "page", "pageSize" })]
public async Task<IActionResult> GetProductsFast(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20)
{
    // Single query with projection
    var products = await dbContext.Products
        .AsNoTracking()
        .Where(p => p.IsActive)
        .OrderBy(p => p.Name)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(p => new ProductWithReviewsDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            ReviewCount = p.Reviews.Count(),
            AverageRating = p.Reviews.Any() ? p.Reviews.Average(r => r.Rating) : 0,
            // Projection แทน string concatenation
            ReviewSummary = p.Reviews
                .OrderByDescending(r => r.CreatedAt)
                .Select(r => r.Text)
                .FirstOrDefault() ?? ""
        })
        .ToListAsync();
    
    return Ok(products);
}

// Version 3 - เร็วที่สุด ด้วย Dapper
[HttpGet("/api/products/fastest")]
public async Task<IActionResult> GetProductsFastest(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20)
{
    // Check cache first
    var cacheKey = $"products:page:{page}:size:{pageSize}";
    var cached = await _cache.GetAsync<List<ProductWithReviewsDto>>(cacheKey);
    if (cached != null) return Ok(cached);
    
    // Raw SQL with Dapper
    const string sql = @"
        SELECT 
            p.Id,
            p.Name,
            p.Price,
            COUNT(r.Id) AS ReviewCount,
            COALESCE(AVG(CAST(r.Rating AS DECIMAL)), 0) AS AverageRating,
            (SELECT TOP 1 r2.Text FROM Reviews r2 
             WHERE r2.ProductId = p.Id 
             ORDER BY r2.CreatedAt DESC) AS ReviewSummary
        FROM Products p
        LEFT JOIN Reviews r ON r.ProductId = p.Id
        WHERE p.IsActive = 1
        GROUP BY p.Id, p.Name, p.Price
        ORDER BY p.Name
        OFFSET @Offset ROWS
        FETCH NEXT @PageSize ROWS ONLY";
    
    using var connection = new SqlConnection(_connectionString);
    var result = (await connection.QueryAsync<ProductWithReviewsDto>(sql, new
    {
        Offset = (page - 1) * pageSize,
        PageSize = pageSize
    })).AsList();
    
    await _cache.SetAsync(cacheKey, result, TimeSpan.FromMinutes(5));
    return Ok(result);
}
```

### Performance Comparison

```csharp
// Benchmark ทั้ง 3 versions
[MemoryDiagnoser]
public class ApiBenchmarks
{
    private WebApplicationFactory<Program> _factory = null!;
    private HttpClient _client = null!;

    [GlobalSetup]
    public void Setup()
    {
        _factory = new WebApplicationFactory<Program>();
        _client = _factory.CreateClient();
    }

    [Benchmark(Baseline = true)]
    public async Task<string> SlowEndpoint()
    {
        var response = await _client.GetAsync("/api/products/slow");
        return await response.Content.ReadAsStringAsync();
    }

    [Benchmark]
    public async Task<string> FastEndpoint()
    {
        var response = await _client.GetAsync("/api/products/fast");
        return await response.Content.ReadAsStringAsync();
    }

    [Benchmark]
    public async Task<string> FastestEndpoint()
    {
        var response = await _client.GetAsync("/api/products/fastest");
        return await response.Content.ReadAsStringAsync();
    }
    
    [GlobalCleanup]
    public void Cleanup()
    {
        _client.Dispose();
        _factory.Dispose();
    }
}
```

---

## Exercises / Project Tasks

### Exercise 1: Memory Optimization
Optimize code ที่มี memory issues:
- แทนที่ string concatenation ด้วย StringBuilder
- ใช้ ArrayPool แทน new array
- ใช้ Span<T> แทน array slicing

### Exercise 2: Database Optimization
Fix N+1 query:
- ใช้ Include() สำหรับ related data
- ใช้ AsNoTracking() สำหรับ read-only
- เพิ่ม database indexes ที่เหมาะสม

### Exercise 3: Caching
เพิ่ม caching layers:
- IMemoryCache สำหรับ hot data
- Redis สำหรับ distributed cache
- Output Cache สำหรับ API responses

### Exercise 4: Benchmark
วัด performance improvement:
- สร้าง BenchmarkDotNet benchmark
- เปรียบเทียบ before/after
- Report memory allocation

---

## สรุป

- **Profile ก่อน optimize** - ใช้ dotnet-trace, dotnet-counters เพื่อหา bottleneck
- **Memory**: ใช้ Span<T>, ArrayPool, struct แทน class สำหรับ value types
- **CPU**: หลีกเลี่ยง LINQ ใน hot path, ใช้ parallel processing เมื่อเหมาะสม
- **Database**: แก้ N+1, ใช้ AsNoTracking, compiled queries, proper indexes
- **Caching**: Cache ที่ appropriate level - memory, distributed, output
- **Async**: อย่า block async, ใช้ parallel async สำหรับ independent operations
- **Benchmark.NET** ช่วย measure และ compare performance อย่างแม่นยำ

---

## Part ถัดไป

**Part 088: Security Best Practices** - เรียนรู้การ secure .NET applications จาก OWASP Top 10

---

*Part 087/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
