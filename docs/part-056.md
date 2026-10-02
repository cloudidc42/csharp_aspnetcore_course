# Part 056: EF Core: Performance

## เนื้อหาใน Part นี้
- AsNoTracking
- Projection (Select specific fields)
- Batch operations
- Connection pooling
- Query caching
- N+1 Problem and solution
- โปรแกรมตัวอย่าง: Optimized product catalog

---

## ทำไม Performance ถึงสำคัญ

Database operations มักเป็น bottleneck หลักในแอปพลิเคชัน ปัญหาที่พบบ่อย:
- ดึงข้อมูลมากเกินความจำเป็น (Over-fetching)
- N+1 Query Problem
- ไม่ใช้ index ที่เหมาะสม
- Change Tracking overhead ที่ไม่จำเป็น

---

## AsNoTracking

โดยปกติ EF Core **track** ทุก entity ที่ query มา เพื่อ detect การเปลี่ยนแปลงเมื่อ SaveChanges()

```csharp
// ปกติ - มี tracking overhead
var products = await context.Products.ToListAsync();
// EF Core ติดตามทุก product ใน ChangeTracker

// AsNoTracking - ไม่ track = เร็วกว่า + ใช้ memory น้อยกว่า
var products = await context.Products
    .AsNoTracking()
    .ToListAsync();
```

### เมื่อไหรควรใช้ AsNoTracking

```csharp
// ✅ ใช้ AsNoTracking เมื่อ:
// - Read-only operations (ไม่ต้อง update)
// - List pages, reports
// - API responses

var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .ToListAsync();

// ❌ ไม่ใช้ AsNoTracking เมื่อ:
// - ต้อง update entity ที่ query มา
// - ใช้ navigation properties หลัง query

var product = await context.Products.FindAsync(id);  // tracked
product.Price = newPrice;
await context.SaveChangesAsync();  // EF Core รู้ว่า Modified
```

### AsNoTrackingWithIdentityResolution (EF Core 5+)

```csharp
// AsNoTracking อาจสร้าง entity ซ้ำถ้า join กับ table เดียวกัน
// AsNoTrackingWithIdentityResolution แก้ปัญหานี้
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToListAsync();
```

### Global No Tracking

```csharp
// ตั้ง NoTracking เป็น default ทั้ง DbContext
public class ReadOnlyDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlite("Data Source=app.db")
               .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
    }
}
```

---

## Projection

Projection คือการ SELECT เฉพาะ fields ที่ต้องการ แทนการดึงทั้ง entity:

```csharp
// ❌ ดึงทุก field (ไม่ดี)
var products = await context.Products
    .Where(p => p.IsActive)
    .ToListAsync();

// ✅ ดึงแค่ที่ต้องการ (ดี)
var productNames = await context.Products
    .Where(p => p.IsActive)
    .Select(p => new { p.Id, p.Name, p.Price })
    .ToListAsync();
```

### Projection กับ Related Data

```csharp
// แทนที่จะ Include ทั้งหมด
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ThenInclude(i => i.Product)
    .ToListAsync();  // ดึงข้อมูลมาก

// ใช้ Projection แทน
var orderSummaries = await context.Orders
    .Select(o => new OrderSummaryDto(
        o.Id,
        o.OrderNumber,
        o.Customer.FullName,       // แค่ FullName
        o.Items.Count,             // แค่ Count
        o.Items.Sum(i => i.Total)  // แค่ Sum
    ))
    .ToListAsync();
```

### DTO Classes

```csharp
// DTOs สำหรับ projection
public record ProductListDto(
    int Id,
    string Name,
    decimal Price,
    string CategoryName,
    int StockQuantity);

public record OrderSummaryDto(
    int Id,
    string OrderNumber,
    string CustomerName,
    int ItemCount,
    decimal Total);

// Query
var products = await context.Products
    .Where(p => p.IsActive)
    .Select(p => new ProductListDto(
        p.Id,
        p.Name,
        p.Price,
        p.Category.Name,
        p.StockQuantity))
    .OrderBy(p => p.Name)
    .ToListAsync();
```

---

## N+1 Problem

N+1 เกิดเมื่อเราดึงข้อมูล 1 query แล้ว loop ดึงข้อมูล related อีก N queries

```csharp
// ❌ N+1 Problem
var blogs = await context.Blogs.ToListAsync();  // Query 1

foreach (var blog in blogs)
{
    // Query 2, 3, 4, ... สำหรับแต่ละ blog!
    var posts = await context.Posts
        .Where(p => p.BlogId == blog.Id)
        .ToListAsync();
    
    Console.WriteLine($"{blog.Name}: {posts.Count} posts");
}
// ถ้ามี 100 blogs = 101 queries!
```

### วิธีแก้ N+1

#### 1. Eager Loading (Include)

```csharp
// ✅ แก้ด้วย Include - 1 หรือ 2 queries
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ToListAsync();

foreach (var blog in blogs)
{
    // ไม่มี additional query!
    Console.WriteLine($"{blog.Name}: {blog.Posts.Count} posts");
}
```

#### 2. Projection กับ Sub-query

```csharp
// ✅ Projection - 1 query
var blogStats = await context.Blogs
    .Select(b => new
    {
        b.Name,
        PostCount = b.Posts.Count,
        PublishedCount = b.Posts.Count(p => p.IsPublished),
        LatestPost = b.Posts
            .OrderByDescending(p => p.CreatedAt)
            .Select(p => p.Title)
            .FirstOrDefault()
    })
    .ToListAsync();
```

#### 3. Load Related Data แยกกัน (Batch)

```csharp
// ✅ โหลดแยกแต่ batch (2 queries แทน N+1)
var blogs = await context.Blogs.ToListAsync();
var blogIds = blogs.Select(b => b.Id).ToList();

// โหลด posts ทั้งหมดพร้อมกัน
var posts = await context.Posts
    .Where(p => blogIds.Contains(p.BlogId))
    .ToListAsync();

// group ใน memory
var postsByBlog = posts.GroupBy(p => p.BlogId)
    .ToDictionary(g => g.Key, g => g.ToList());

foreach (var blog in blogs)
{
    var blogPosts = postsByBlog.GetValueOrDefault(blog.Id, []);
    Console.WriteLine($"{blog.Name}: {blogPosts.Count} posts");
}
```

---

## Split Query (EF Core 5+)

เมื่อ Include ทำให้ query ช้าจากการ JOIN หลาย tables:

```csharp
// Single Query - JOIN หลาย tables = อาจช้าและข้อมูลซ้ำ
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ThenInclude(p => p.Tags)
    .Include(b => b.Posts)
    .ThenInclude(p => p.Comments)
    .ToListAsync();

// Split Query - หลาย queries แต่ไม่มี data duplication
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .ThenInclude(p => p.Tags)
    .Include(b => b.Posts)
    .ThenInclude(p => p.Comments)
    .AsSplitQuery()  // แยกเป็นหลาย queries
    .ToListAsync();

// ตั้ง Split Query เป็น default
optionsBuilder.UseSqlite("...", options =>
    options.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

---

## Batch Operations

### Batch Insert

```csharp
// ❌ Loop Insert ช้า
foreach (var product in products)
{
    context.Products.Add(product);
    await context.SaveChangesAsync();  // SaveChanges ทุก item = N queries
}

// ✅ AddRange แล้ว SaveChanges ครั้งเดียว
context.Products.AddRange(products);
await context.SaveChangesAsync();  // 1 batch INSERT

// ✅ สำหรับข้อมูลจำนวนมาก (ทำ batch ย่อย)
const int batchSize = 1000;
for (int i = 0; i < allProducts.Count; i += batchSize)
{
    var batch = allProducts.Skip(i).Take(batchSize).ToList();
    context.Products.AddRange(batch);
    await context.SaveChangesAsync();
    context.ChangeTracker.Clear();  // Clear tracking หลัง save
}
```

### ExecuteUpdateAsync / ExecuteDeleteAsync (EF Core 7+)

```csharp
// ✅ Bulk Update - ไม่ต้อง load entities
await context.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(setter => setter
        .SetProperty(p => p.CategoryId, newCategoryId)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));

// ✅ Bulk Delete - ไม่ต้อง load entities
await context.AuditLogs
    .Where(l => l.CreatedAt < DateTime.UtcNow.AddMonths(-6))
    .ExecuteDeleteAsync();
```

### BulkExtensions (Third-party library สำหรับ large-scale)

```bash
# สำหรับ SQL Server / PostgreSQL
dotnet add package EFCore.BulkExtensions
```

```csharp
// BulkInsert - เร็วมากสำหรับข้อมูลจำนวนมาก
await context.BulkInsertAsync(products);
await context.BulkUpdateAsync(products);
await context.BulkDeleteAsync(products);
await context.BulkInsertOrUpdateAsync(products);  // Upsert
```

---

## Connection Pooling

EF Core ใช้ Connection Pooling อัตโนมัติผ่าน ADO.NET

### DbContext Pooling (ASP.NET Core)

```csharp
// Program.cs - ใช้ AddDbContextPool แทน AddDbContext
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlite(
        builder.Configuration.GetConnectionString("Default")),
    poolSize: 128);  // จำนวน contexts ใน pool

// สำหรับ factory pattern
builder.Services.AddPooledDbContextFactory<AppDbContext>(options =>
    options.UseSqlite("..."), poolSize: 128);
```

### Connection String Options

```csharp
// SQL Server
"Server=.;Database=MyDb;Integrated Security=true;" +
"Min Pool Size=5;Max Pool Size=100;" +
"Connection Timeout=30;Command Timeout=60"

// PostgreSQL
"Host=localhost;Database=mydb;Username=user;Password=pass;" +
"Minimum Pool Size=5;Maximum Pool Size=100;Connection Idle Lifetime=300"
```

---

## Compiled Queries

สำหรับ queries ที่ใช้บ่อย สามารถ pre-compile เพื่อลด overhead:

```csharp
// Compiled query - compile ครั้งเดียว ใช้ได้เรื่อยๆ
private static readonly Func<AppDbContext, int, Task<Product?>> 
    GetProductByIdQuery = EF.CompileAsyncQuery(
        (AppDbContext ctx, int id) =>
            ctx.Products
               .AsNoTracking()
               .FirstOrDefault(p => p.Id == id));

// ใช้งาน
var product = await GetProductByIdQuery(context, productId);
```

```csharp
// Compiled query กับ multiple parameters
private static readonly Func<AppDbContext, bool, int, int, IAsyncEnumerable<Product>>
    GetProductsPagedQuery = EF.CompileAsyncQuery(
        (AppDbContext ctx, bool activeOnly, int skip, int take) =>
            ctx.Products
               .AsNoTracking()
               .Where(p => !activeOnly || p.IsActive)
               .OrderBy(p => p.Name)
               .Skip(skip)
               .Take(take));

// ใช้งาน
await foreach (var product in GetProductsPagedQuery(context, true, 0, 10))
{
    Console.WriteLine(product.Name);
}
```

---

## Query Caching กับ Second Level Cache

EF Core มี plan caching อัตโนมัติ แต่ไม่มี result caching

### ใช้ IMemoryCache

```csharp
// Services/CachedProductService.cs
using Microsoft.Extensions.Caching.Memory;
using Microsoft.EntityFrameworkCore;

public class CachedProductService
{
    private readonly AppDbContext _context;
    private readonly IMemoryCache _cache;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);
    
    public CachedProductService(AppDbContext context, IMemoryCache cache)
    {
        _context = context;
        _cache = cache;
    }
    
    public async Task<List<ProductDto>> GetActiveProductsAsync()
    {
        const string cacheKey = "active_products";
        
        if (_cache.TryGetValue(cacheKey, out List<ProductDto>? cached) && cached != null)
        {
            return cached;
        }
        
        var products = await _context.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
            .ToListAsync();
        
        _cache.Set(cacheKey, products, CacheDuration);
        
        return products;
    }
    
    public void InvalidateProductCache()
    {
        _cache.Remove("active_products");
    }
}
```

### Cache Patterns

```csharp
// Sliding expiration - reset เมื่อถูก access
_cache.Set(cacheKey, data, new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(5)
});

// Absolute expiration
_cache.Set(cacheKey, data, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
});

// Size-based eviction
services.AddMemoryCache(options =>
{
    options.SizeLimit = 1000;  // max 1000 entries
});

_cache.Set(cacheKey, data, new MemoryCacheEntryOptions
{
    Size = 1
});
```

---

## Indexes

Index ที่เหมาะสมช่วย query ได้มาก:

```csharp
modelBuilder.Entity<Product>(e =>
{
    // Single column index
    e.HasIndex(p => p.Name);
    
    // Unique index
    e.HasIndex(p => p.Sku).IsUnique();
    
    // Composite index
    e.HasIndex(p => new { p.CategoryId, p.IsActive });
    
    // Filtered index (SQL Server)
    // e.HasIndex(p => p.Email)
    //  .HasFilter("[Email] IS NOT NULL");
    
    // Named index
    e.HasIndex(p => p.CreatedAt)
     .HasDatabaseName("IX_Products_CreatedAt");
    
    // Descending index (EF Core 7+)
    e.HasIndex(p => p.CreatedAt)
     .IsDescending();
});
```

---

## โปรแกรมตัวอย่าง: Optimized Product Catalog

```csharp
// Models
namespace ProductCatalog.Models;

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    public List<Product> Products { get; set; } = [];
}

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;
    public double AverageRating { get; set; }
    public int ReviewCount { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    
    public List<ProductView> Views { get; set; } = [];
}

public class ProductView
{
    public int Id { get; set; }
    public DateTime ViewedAt { get; set; } = DateTime.UtcNow;
    public string? UserId { get; set; }
    
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
}
```

```csharp
// Data/CatalogDbContext.cs
using Microsoft.EntityFrameworkCore;
using ProductCatalog.Models;

public class CatalogDbContext : DbContext
{
    public DbSet<Category> Categories { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<ProductView> ProductViews { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlite("Data Source=catalog.db");
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Category>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.Name).IsRequired().HasMaxLength(100);
            e.Property(c => c.Slug).IsRequired().HasMaxLength(100);
            e.HasIndex(c => c.Slug).IsUnique();
        });
        
        modelBuilder.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Name).IsRequired().HasMaxLength(200);
            e.Property(p => p.Sku).IsRequired().HasMaxLength(50);
            e.Property(p => p.Price).HasColumnType("decimal(18,2)");
            
            // Indexes สำคัญ
            e.HasIndex(p => p.Sku).IsUnique();
            e.HasIndex(p => new { p.CategoryId, p.IsActive });
            e.HasIndex(p => p.Price);
            e.HasIndex(p => p.AverageRating).IsDescending();
            e.HasIndex(p => p.CreatedAt).IsDescending();
            
            e.HasOne(p => p.Category)
             .WithMany(c => c.Products)
             .HasForeignKey(p => p.CategoryId);
        });
        
        modelBuilder.Entity<ProductView>(e =>
        {
            e.HasKey(v => v.Id);
            e.HasIndex(v => new { v.ProductId, v.ViewedAt });
            
            e.HasOne(v => v.Product)
             .WithMany(p => p.Views)
             .HasForeignKey(v => v.ProductId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

```csharp
// Services/OptimizedProductService.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Caching.Memory;
using ProductCatalog.Models;

public class OptimizedProductService
{
    private readonly CatalogDbContext _context;
    private readonly IMemoryCache _cache;
    
    // Compiled queries สำหรับ hot paths
    private static readonly Func<CatalogDbContext, int, Task<Product?>> 
        FindProductQuery = EF.CompileAsyncQuery(
            (CatalogDbContext ctx, int id) =>
                ctx.Products
                   .AsNoTracking()
                   .Include(p => p.Category)
                   .FirstOrDefault(p => p.Id == id && p.IsActive));
    
    private static readonly Func<CatalogDbContext, string, Task<Product?>> 
        FindProductBySkuQuery = EF.CompileAsyncQuery(
            (CatalogDbContext ctx, string sku) =>
                ctx.Products
                   .AsNoTracking()
                   .FirstOrDefault(p => p.Sku == sku));
    
    public OptimizedProductService(CatalogDbContext context, IMemoryCache cache)
    {
        _context = context;
        _cache = cache;
    }
    
    // Fast product lookup ด้วย compiled query
    public async Task<Product?> GetProductAsync(int id)
    {
        return await FindProductQuery(_context, id);
    }
    
    // Catalog page - optimized ด้วย projection + cache
    public async Task<ProductCatalogDto> GetCatalogPageAsync(
        int categoryId = 0,
        int page = 1,
        int pageSize = 20,
        string sortBy = "name")
    {
        var cacheKey = $"catalog_{categoryId}_{page}_{pageSize}_{sortBy}";
        
        if (_cache.TryGetValue(cacheKey, out ProductCatalogDto? cached) && cached != null)
            return cached;
        
        // Build query
        var query = _context.Products
            .AsNoTracking()
            .Where(p => p.IsActive);
        
        if (categoryId > 0)
            query = query.Where(p => p.CategoryId == categoryId);
        
        // Sort
        query = sortBy switch
        {
            "price_asc" => query.OrderBy(p => p.Price),
            "price_desc" => query.OrderByDescending(p => p.Price),
            "rating" => query.OrderByDescending(p => p.AverageRating),
            "newest" => query.OrderByDescending(p => p.CreatedAt),
            _ => query.OrderBy(p => p.Name)
        };
        
        // Projection - ดึงแค่ที่ต้องการ
        var totalCount = await query.CountAsync();
        
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .Select(p => new ProductCardDto(
                p.Id,
                p.Name,
                p.Sku,
                p.Price,
                p.StockQuantity > 0,
                p.AverageRating,
                p.ReviewCount,
                p.Category.Name))
            .ToListAsync();
        
        var result = new ProductCatalogDto(
            items,
            totalCount,
            page,
            pageSize,
            (int)Math.Ceiling(totalCount / (double)pageSize));
        
        // Cache 5 นาที
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(5));
        
        return result;
    }
    
    // Trending products - cached 10 นาที
    public async Task<List<ProductCardDto>> GetTrendingProductsAsync(int count = 10)
    {
        const string cacheKey = "trending_products";
        
        if (_cache.TryGetValue(cacheKey, out List<ProductCardDto>? cached) && cached != null)
            return cached;
        
        // คำนวณ trending จาก views ใน 7 วันที่ผ่านมา
        var since = DateTime.UtcNow.AddDays(-7);
        
        var trending = await _context.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .Select(p => new ProductCardDto(
                p.Id,
                p.Name,
                p.Sku,
                p.Price,
                p.StockQuantity > 0,
                p.AverageRating,
                p.ReviewCount,
                p.Category.Name))
            .Take(count)
            .ToListAsync();
        
        _cache.Set(cacheKey, trending, TimeSpan.FromMinutes(10));
        
        return trending;
    }
    
    // Search - optimized
    public async Task<List<ProductCardDto>> SearchProductsAsync(
        string query,
        int limit = 20)
    {
        if (string.IsNullOrWhiteSpace(query) || query.Length < 2)
            return [];
        
        return await _context.Products
            .AsNoTracking()
            .Where(p => p.IsActive && 
                   (p.Name.Contains(query) || p.Sku.Contains(query)))
            .OrderByDescending(p => p.Name.StartsWith(query))  // exact match first
            .ThenBy(p => p.Name)
            .Take(limit)
            .Select(p => new ProductCardDto(
                p.Id, p.Name, p.Sku, p.Price,
                p.StockQuantity > 0, p.AverageRating, p.ReviewCount,
                p.Category.Name))
            .ToListAsync();
    }
    
    // Batch update prices
    public async Task<int> UpdatePricesAsync(
        int categoryId,
        decimal multiplier)
    {
        // ExecuteUpdateAsync - ไม่ต้อง load entities
        return await _context.Products
            .Where(p => p.CategoryId == categoryId && p.IsActive)
            .ExecuteUpdateAsync(setter => setter
                .SetProperty(p => p.Price, p => p.Price * multiplier)
                .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));
    }
    
    // Record view (write-optimized)
    public async Task RecordViewAsync(int productId, string? userId)
    {
        // ไม่ต้อง check product exists - FK constraint จะ handle
        _context.ProductViews.Add(new ProductView
        {
            ProductId = productId,
            UserId = userId
        });
        
        // Fire and forget (ไม่ await เพื่อ performance)
        // ในงานจริงอาจใช้ background job queue แทน
        _ = _context.SaveChangesAsync();
    }
    
    // Stats - projection เฉพาะที่ต้องการ
    public async Task<CatalogStatsDto> GetCatalogStatsAsync()
    {
        const string cacheKey = "catalog_stats";
        
        if (_cache.TryGetValue(cacheKey, out CatalogStatsDto? cached) && cached != null)
            return cached;
        
        var stats = await _context.Products
            .AsNoTracking()
            .GroupBy(p => p.IsActive)
            .Select(g => new { IsActive = g.Key, Count = g.Count() })
            .ToListAsync();
        
        var categoryStats = await _context.Categories
            .AsNoTracking()
            .Select(c => new
            {
                c.Name,
                ProductCount = c.Products.Count(p => p.IsActive)
            })
            .OrderByDescending(c => c.ProductCount)
            .Take(5)
            .ToListAsync();
        
        var result = new CatalogStatsDto(
            TotalProducts: stats.Sum(s => s.Count),
            ActiveProducts: stats.FirstOrDefault(s => s.IsActive)?.Count ?? 0,
            TopCategories: categoryStats.Select(c => $"{c.Name}({c.ProductCount})").ToList()
        );
        
        _cache.Set(cacheKey, result, TimeSpan.FromMinutes(15));
        
        return result;
    }
}

// DTOs
public record ProductCardDto(
    int Id,
    string Name,
    string Sku,
    decimal Price,
    bool InStock,
    double AverageRating,
    int ReviewCount,
    string CategoryName);

public record ProductCatalogDto(
    List<ProductCardDto> Products,
    int TotalCount,
    int CurrentPage,
    int PageSize,
    int TotalPages);

public record CatalogStatsDto(
    int TotalProducts,
    int ActiveProducts,
    List<string> TopCategories);
```

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Caching.Memory;
using Microsoft.Extensions.DependencyInjection;
using ProductCatalog.Models;

Console.WriteLine("=== Optimized Product Catalog ===\n");

// Setup DI
var services = new ServiceCollection();
services.AddDbContext<CatalogDbContext>();
services.AddMemoryCache();
services.AddScoped<OptimizedProductService>();

var serviceProvider = services.BuildServiceProvider();

await using var scope = serviceProvider.CreateAsyncScope();
var context = scope.ServiceProvider.GetRequiredService<CatalogDbContext>();
var productService = scope.ServiceProvider.GetRequiredService<OptimizedProductService>();

// Setup database
await context.Database.EnsureDeletedAsync();
await context.Database.EnsureCreatedAsync();

// Seed data
await SeedDataAsync(context);

// Test optimized queries
Console.WriteLine("Testing optimized queries...\n");

// 1. Get catalog page
Console.WriteLine("1. Catalog Page (All, Sort by Name):");
var catalog = await productService.GetCatalogPageAsync(page: 1, pageSize: 5);
Console.WriteLine($"   Total: {catalog.TotalCount}, Pages: {catalog.TotalPages}");
foreach (var p in catalog.Products)
    Console.WriteLine($"   {p.Name} ฿{p.Price:N0} [{(p.InStock ? "In Stock" : "Out of Stock")}]");

// 2. Get catalog (cached)
Console.WriteLine("\n2. Same query (from cache):");
var sw = System.Diagnostics.Stopwatch.StartNew();
var cached = await productService.GetCatalogPageAsync(page: 1, pageSize: 5);
sw.Stop();
Console.WriteLine($"   Retrieved {cached.Products.Count} products in {sw.ElapsedMilliseconds}ms (from cache)");

// 3. Search
Console.WriteLine("\n3. Search 'pro':");
var results = await productService.SearchProductsAsync("pro");
foreach (var p in results)
    Console.WriteLine($"   {p.Name} ({p.Sku})");

// 4. Batch price update
Console.WriteLine("\n4. Update prices (+10%):");
var updated = await productService.UpdatePricesAsync(categoryId: 1, multiplier: 1.1m);
Console.WriteLine($"   Updated {updated} products");

// 5. Stats
Console.WriteLine("\n5. Catalog Statistics:");
var stats = await productService.GetCatalogStatsAsync();
Console.WriteLine($"   Total: {stats.TotalProducts}");
Console.WriteLine($"   Active: {stats.ActiveProducts}");
Console.WriteLine($"   Top Categories: {string.Join(", ", stats.TopCategories)}");

Console.WriteLine("\n=== Performance Tips Summary ===");
Console.WriteLine("✓ AsNoTracking for read-only queries");
Console.WriteLine("✓ Projection to select only needed fields");
Console.WriteLine("✓ Include to avoid N+1 queries");
Console.WriteLine("✓ Compiled queries for hot paths");
Console.WriteLine("✓ ExecuteUpdateAsync/ExecuteDeleteAsync for bulk ops");
Console.WriteLine("✓ IMemoryCache for repeated queries");
Console.WriteLine("✓ Proper indexes on filtered/sorted columns");

Console.WriteLine("\n=== Done! ===");

static async Task SeedDataAsync(CatalogDbContext ctx)
{
    var categories = new[]
    {
        new Category { Name = "Electronics", Slug = "electronics" },
        new Category { Name = "Clothing", Slug = "clothing" }
    };
    ctx.Categories.AddRange(categories);
    await ctx.SaveChangesAsync();
    
    var rng = new Random(42);
    var products = Enumerable.Range(1, 50).Select(i => new Product
    {
        Name = $"Product {i:D3} Pro",
        Sku = $"SKU-{i:D5}",
        Price = rng.Next(100, 50000),
        StockQuantity = rng.Next(0, 100),
        AverageRating = Math.Round(rng.NextDouble() * 4 + 1, 1),
        ReviewCount = rng.Next(0, 500),
        CategoryId = i % 2 == 0 ? categories[0].Id : categories[1].Id
    }).ToList();
    
    ctx.Products.AddRange(products);
    await ctx.SaveChangesAsync();
}
```

---

## Performance Checklist

```
✅ ใช้ AsNoTracking() สำหรับ read-only queries
✅ ใช้ Projection (Select) เพื่อดึงแค่ fields ที่ต้องการ
✅ ใช้ Include() แทน Lazy Loading เพื่อหลีกเลี่ยง N+1
✅ ใช้ AsSplitQuery() สำหรับ complex includes
✅ ใช้ ExecuteUpdateAsync/ExecuteDeleteAsync สำหรับ bulk operations
✅ ใช้ AddDbContextPool แทน AddDbContext ใน ASP.NET Core
✅ ตั้ง index ที่เหมาะสมบน WHERE, ORDER BY columns
✅ ใช้ Compiled Queries สำหรับ frequently-used queries
✅ Cache results ที่ไม่เปลี่ยนบ่อย
✅ ใช้ batch inserts แทน loop inserts
```

---

## Exercises

### แบบฝึกหัดที่ 1: Benchmark
เขียน benchmark เปรียบเทียบ:
1. `ToListAsync()` vs `AsNoTracking().ToListAsync()`
2. `Include()` vs N+1 Lazy Loading
3. Projection vs Full entity

### แบบฝึกหัดที่ 2: Fix N+1
มีโค้ดต่อไปนี้ที่มี N+1 problem ให้แก้ไข:
```csharp
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
{
    var customer = await context.Customers.FindAsync(order.CustomerId);
    var items = await context.OrderItems
        .Where(i => i.OrderId == order.Id)
        .ToListAsync();
    Console.WriteLine($"{customer?.FullName}: {items.Count} items");
}
```

### แบบฝึกหัดที่ 3: Cache Implementation
สร้าง `CacheService` ที่:
1. Cache top 10 products 5 นาที
2. Cache categories 1 ชั่วโมง
3. Invalidate cache เมื่อ update product

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **AsNoTracking**: ลด overhead สำหรับ read-only queries
2. **Projection**: SELECT เฉพาะ fields ที่ต้องการ
3. **N+1 Problem**: ปัญหาที่พบบ่อยและวิธีแก้ด้วย Include
4. **Split Query**: แก้ปัญหา large JOINs
5. **Batch Operations**: ExecuteUpdateAsync/ExecuteDeleteAsync
6. **Connection Pooling**: DbContextPool
7. **Compiled Queries**: Pre-compile สำหรับ hot paths
8. **Caching**: IMemoryCache สำหรับ query results
9. **Indexes**: เพิ่ม query performance อย่างมาก

---

## Part ถัดไป

ใน **Part 057** เราจะเรียนรู้เกี่ยวกับ **Repository Pattern กับ EF Core**:
- Repository interface
- Generic repository
- Unit of Work pattern
- Dependency injection กับ repositories
- Testing repositories

---

*Part 056/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
