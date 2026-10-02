# Part 053: EF Core: Querying

## เนื้อหาใน Part นี้
- LINQ กับ EF Core
- Include และ Eager Loading
- Lazy Loading
- Explicit Loading
- Raw SQL ด้วย FromSqlRaw
- การ filter, sort, paginate
- โปรแกรมตัวอย่าง: Product queries

---

## LINQ กับ EF Core

EF Core แปลง LINQ expressions เป็น SQL queries อัตโนมัติ เราสามารถเขียน query ได้สองรูปแบบ:

### Query Syntax

```csharp
// คล้าย SQL
var products = from p in context.Products
               where p.Price > 1000 && p.IsActive
               orderby p.Name
               select p;

var result = await products.ToListAsync();
```

### Method Syntax (แนะนำ)

```csharp
// Fluent style - นิยมใช้มากกว่า
var products = await context.Products
    .Where(p => p.Price > 1000 && p.IsActive)
    .OrderBy(p => p.Name)
    .ToListAsync();
```

### IQueryable vs IEnumerable

```csharp
// IQueryable - query ยังอยู่ฝั่ง database
IQueryable<Product> query = context.Products.Where(p => p.IsActive);

// เพิ่ม conditions ได้ (ยังไม่ hit database)
if (minPrice.HasValue)
    query = query.Where(p => p.Price >= minPrice.Value);

if (maxPrice.HasValue)
    query = query.Where(p => p.Price <= maxPrice.Value);

// Execute query (hit database ตอนนี้)
var result = await query.ToListAsync();

// IEnumerable - ดึงข้อมูลมาฝั่ง C# แล้ว filter ที่นี่
// ไม่แนะนำ - ดึงข้อมูลทั้งหมดมาก่อน
IEnumerable<Product> allProducts = await context.Products.ToListAsync();
var filtered = allProducts.Where(p => p.Price > 1000); // filter ใน memory
```

---

## Projection (Select)

Projection คือการเลือก fields ที่ต้องการแทนการดึงทั้ง entity:

```csharp
// ดึงทั้ง entity (ไม่แนะนำถ้าไม่ต้องการทุก field)
var allProducts = await context.Products.ToListAsync();

// Projection เป็น anonymous type
var productSummaries = await context.Products
    .Where(p => p.IsActive)
    .Select(p => new 
    { 
        p.Id, 
        p.Name, 
        p.Price,
        CategoryCount = p.ProductCategories.Count
    })
    .ToListAsync();

// Projection เป็น DTO
public record ProductDto(int Id, string Name, decimal Price, string CategoryName);

var dtos = await context.Products
    .Where(p => p.IsActive)
    .Select(p => new ProductDto(
        p.Id,
        p.Name,
        p.Price,
        p.ProductCategories.First().Category.Name
    ))
    .ToListAsync();
```

---

## Include - Eager Loading

Eager Loading คือการโหลด related data พร้อมกันใน query เดียว

```csharp
using Microsoft.EntityFrameworkCore;

// ดึง orders พร้อม customer และ items
var orders = await context.Orders
    .Include(o => o.Customer)           // Load related Customer
    .Include(o => o.Items)              // Load related Items
        .ThenInclude(oi => oi.Product)  // Then load Product ของแต่ละ Item
    .Include(o => o.Payment)            // Load related Payment
    .ToListAsync();
```

### Include ที่ซับซ้อน

```csharp
// ดึง blog พร้อม posts และ tags ของ posts และ comments ที่ approved
var blogs = await context.Blogs
    .Include(b => b.Posts.Where(p => p.IsPublished))  // Filtered Include (EF Core 5+)
        .ThenInclude(p => p.Tags)
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments.Where(c => c.IsApproved))
    .Where(b => b.IsActive)
    .ToListAsync();
```

### Include กับ OrderBy

```csharp
// Sort related collection
var customers = await context.Customers
    .Include(c => c.Orders
        .OrderByDescending(o => o.OrderDate)
        .Take(5))  // เอาแค่ 5 orders ล่าสุด
    .ToListAsync();
```

---

## Lazy Loading

Lazy Loading โหลด related data อัตโนมัติเมื่อเข้าถึง navigation property ครั้งแรก

### ติดตั้ง

```bash
dotnet add package Microsoft.EntityFrameworkCore.Proxies --version 9.0.0
```

### Configuration

```csharp
optionsBuilder
    .UseSqlite("Data Source=app.db")
    .UseLazyLoadingProxies();  // เปิดใช้ Lazy Loading
```

### Entity ต้องใช้ virtual

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    // ต้องเป็น virtual สำหรับ Lazy Loading
    public virtual List<Post> Posts { get; set; } = [];
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    
    public int BlogId { get; set; }
    
    // ต้องเป็น virtual
    public virtual Blog Blog { get; set; } = null!;
}
```

### การใช้งาน

```csharp
// โหลดแค่ Blog ก่อน
var blog = await context.Blogs.FirstAsync(b => b.Id == 1);
Console.WriteLine(blog.Name);  // Query: SELECT * FROM Blogs WHERE Id = 1

// เมื่อ access Posts -> EF Core จะ query อัตโนมัติ
foreach (var post in blog.Posts)  // Query: SELECT * FROM Posts WHERE BlogId = 1
{
    Console.WriteLine(post.Title);
}
```

**คำเตือน**: Lazy Loading อาจทำให้เกิด N+1 Problem (ดูในหัวข้อ Performance)

---

## Explicit Loading

โหลด related data เมื่อต้องการ โดยเรียกใช้ด้วยตนเอง:

```csharp
// โหลด Blog ก่อน
var blog = await context.Blogs.FirstAsync(b => b.Id == 1);

// โหลด Posts ในภายหลัง (explicit)
await context.Entry(blog)
    .Collection(b => b.Posts)
    .LoadAsync();

// โหลด Author ของ Post
var post = await context.Posts.FirstAsync(p => p.Id == 1);
await context.Entry(post)
    .Reference(p => p.Blog)
    .LoadAsync();

// โหลดพร้อม Query
await context.Entry(blog)
    .Collection(b => b.Posts)
    .Query()
    .Where(p => p.IsPublished)
    .LoadAsync();

// ตรวจสอบว่า loaded แล้วหรือยัง
var isLoaded = context.Entry(blog)
    .Collection(b => b.Posts)
    .IsLoaded;
```

---

## Raw SQL

### FromSqlRaw

```csharp
// ใช้ Raw SQL กับ DbSet
var products = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > {0}", 1000)
    .ToListAsync();

// Raw SQL + LINQ (สามารถต่อด้วย LINQ ได้)
var activeExpensiveProducts = await context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE IsActive = 1")
    .Where(p => p.Price > 5000)
    .OrderBy(p => p.Name)
    .ToListAsync();
```

### FromSqlInterpolated (แนะนำ - ปลอดภัยกว่า)

```csharp
decimal minPrice = 1000m;
string searchTerm = "iPhone";

// ใช้ string interpolation อย่างปลอดภัย (ป้องกัน SQL injection)
var products = await context.Products
    .FromSqlInterpolated($@"
        SELECT * FROM Products 
        WHERE Price > {minPrice} 
        AND Name LIKE {'%' + searchTerm + '%'}")
    .ToListAsync();
```

### ExecuteSqlRaw (สำหรับ INSERT, UPDATE, DELETE)

```csharp
// Execute non-query SQL
int rowsAffected = await context.Database
    .ExecuteSqlRawAsync(
        "UPDATE Products SET IsActive = 0 WHERE Price < {0}", 
        100m);

Console.WriteLine($"Updated {rowsAffected} rows");

// ใช้ parameters ที่ชัดเจน
await context.Database.ExecuteSqlRawAsync(
    "UPDATE Products SET StockQuantity = StockQuantity - {0} WHERE Id = {1}",
    5, productId);
```

### SqlQuery (EF Core 8+) - สำหรับ arbitrary types

```csharp
// Query ที่ไม่ return entity type
var summaries = await context.Database
    .SqlQuery<ProductSummary>($@"
        SELECT 
            p.Id,
            p.Name,
            COUNT(oi.Id) as OrderCount,
            SUM(oi.Quantity) as TotalSold
        FROM Products p
        LEFT JOIN OrderItems oi ON oi.ProductId = p.Id
        GROUP BY p.Id, p.Name")
    .ToListAsync();

// DTO
public record ProductSummary(int Id, string Name, int OrderCount, int TotalSold);
```

---

## Filtering

```csharp
// WHERE
var activeProducts = await context.Products
    .Where(p => p.IsActive)
    .ToListAsync();

// Multiple conditions
var filteredProducts = await context.Products
    .Where(p => p.IsActive && p.Price >= 100m && p.Price <= 1000m)
    .ToListAsync();

// Contains (IN clause)
var ids = new[] { 1, 2, 3, 4, 5 };
var specificProducts = await context.Products
    .Where(p => ids.Contains(p.Id))
    .ToListAsync();

// String operations
var searchResults = await context.Products
    .Where(p => p.Name.Contains("iPhone"))  // SQL LIKE '%iPhone%'
    .ToListAsync();

var startsWithApple = await context.Products
    .Where(p => p.Name.StartsWith("Apple"))  // SQL LIKE 'Apple%'
    .ToListAsync();

// Null checks
var productsWithDescription = await context.Products
    .Where(p => p.Description != null)
    .ToListAsync();

// Date range
var recentOrders = await context.Orders
    .Where(o => o.OrderDate >= DateTime.UtcNow.AddDays(-30))
    .ToListAsync();

// Or condition
var discountedOrFeatured = await context.Products
    .Where(p => p.HasDiscount || p.IsFeatured)
    .ToListAsync();

// Nested condition (related entity)
var blogsWithPublishedPosts = await context.Blogs
    .Where(b => b.Posts.Any(p => p.IsPublished))
    .ToListAsync();
```

---

## Sorting

```csharp
// Order ascending
var byNameAsc = await context.Products
    .OrderBy(p => p.Name)
    .ToListAsync();

// Order descending
var byPriceDesc = await context.Products
    .OrderByDescending(p => p.Price)
    .ToListAsync();

// Multiple sort
var multiSort = await context.Products
    .OrderBy(p => p.Category)
    .ThenBy(p => p.Price)
    .ThenByDescending(p => p.Name)
    .ToListAsync();

// Dynamic sorting
string sortBy = "Price";
bool ascending = true;

IQueryable<Product> query = context.Products;
query = (sortBy, ascending) switch
{
    ("Name", true)  => query.OrderBy(p => p.Name),
    ("Name", false) => query.OrderByDescending(p => p.Name),
    ("Price", true) => query.OrderBy(p => p.Price),
    ("Price", false) => query.OrderByDescending(p => p.Price),
    _ => query.OrderBy(p => p.Id)
};

var sorted = await query.ToListAsync();
```

---

## Pagination

```csharp
// Skip/Take Pagination
int page = 1;        // หน้าที่ต้องการ (เริ่มจาก 1)
int pageSize = 10;   // จำนวนต่อหน้า

var pagedProducts = await context.Products
    .OrderBy(p => p.Id)  // ต้อง order ก่อน pagination
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// ดึง total count ด้วย
var totalCount = await context.Products.CountAsync();
var totalPages = (int)Math.Ceiling(totalCount / (double)pageSize);

Console.WriteLine($"Page {page}/{totalPages} - Showing {pagedProducts.Count} of {totalCount} products");
```

### Pagination Helper

```csharp
// PagedResult class
public class PagedResult<T>
{
    public List<T> Items { get; set; } = [];
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => Page > 1;
    public bool HasNextPage => Page < TotalPages;
}

// Extension method
public static class QueryableExtensions
{
    public static async Task<PagedResult<T>> ToPagedResultAsync<T>(
        this IQueryable<T> query,
        int page,
        int pageSize)
    {
        var totalCount = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
            
        return new PagedResult<T>
        {
            Items = items,
            TotalCount = totalCount,
            Page = page,
            PageSize = pageSize
        };
    }
}

// ใช้งาน
var result = await context.Products
    .Where(p => p.IsActive)
    .OrderBy(p => p.Name)
    .ToPagedResultAsync(page: 1, pageSize: 10);
    
Console.WriteLine($"Page {result.Page}/{result.TotalPages}");
Console.WriteLine($"Total: {result.TotalCount}");
foreach (var product in result.Items)
{
    Console.WriteLine($"  - {product.Name}");
}
```

### Cursor-based Pagination (เหมาะกับ large datasets)

```csharp
// ใช้ keyset pagination แทน offset pagination
// เร็วกว่า Skip/Take สำหรับข้อมูลจำนวนมาก
public async Task<List<Product>> GetProductsAfterAsync(
    int lastId, 
    int pageSize = 10)
{
    return await context.Products
        .Where(p => p.Id > lastId)  // ใช้ cursor แทน Skip
        .OrderBy(p => p.Id)
        .Take(pageSize)
        .ToListAsync();
}
```

---

## Aggregate Functions

```csharp
// Count
int totalProducts = await context.Products.CountAsync();
int activeProducts = await context.Products.CountAsync(p => p.IsActive);

// Sum
decimal? totalRevenue = await context.OrderItems
    .SumAsync(oi => oi.Quantity * oi.UnitPrice);

// Average
decimal? avgPrice = await context.Products
    .Where(p => p.IsActive)
    .AverageAsync(p => p.Price);

// Min/Max
decimal? minPrice = await context.Products.MinAsync(p => p.Price);
decimal? maxPrice = await context.Products.MaxAsync(p => p.Price);

// GroupBy
var salesByProduct = await context.OrderItems
    .GroupBy(oi => oi.ProductId)
    .Select(g => new
    {
        ProductId = g.Key,
        TotalOrders = g.Count(),
        TotalQuantity = g.Sum(oi => oi.Quantity),
        TotalRevenue = g.Sum(oi => oi.Quantity * oi.UnitPrice)
    })
    .OrderByDescending(x => x.TotalRevenue)
    .ToListAsync();
```

---

## Any, All, First, Single

```csharp
// Any - มีข้อมูลที่ตรงเงื่อนไขหรือไม่
bool hasExpensiveProducts = await context.Products
    .AnyAsync(p => p.Price > 100000m);

// All - ทุกรายการตรงเงื่อนไขหรือไม่
bool allProductsActive = await context.Products
    .AllAsync(p => p.IsActive);

// First - อันแรก (throw ถ้าไม่มี)
var firstProduct = await context.Products
    .OrderBy(p => p.Name)
    .FirstAsync();

// FirstOrDefault - อันแรก หรือ null ถ้าไม่มี
var maybeProduct = await context.Products
    .Where(p => p.Price > 1000000m)
    .FirstOrDefaultAsync();

// Single - ต้องมีแค่ 1 (throw ถ้าไม่มี หรือมีมากกว่า 1)
var uniqueProduct = await context.Products
    .SingleAsync(p => p.Sku == "APL-IP16PRO");

// SingleOrDefault
var maybeUniqueProduct = await context.Products
    .SingleOrDefaultAsync(p => p.Sku == "NOT-EXIST");

// Find - ค้นหาด้วย PK (เร็วที่สุด ดูใน tracking cache ก่อน)
var productById = await context.Products.FindAsync(1);
```

---

## โปรแกรมตัวอย่าง: Product Queries

```csharp
// Models/Product.cs
namespace ProductApp.Models;

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
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
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
    
    public List<Review> Reviews { get; set; } = [];
}

public class Review
{
    public int Id { get; set; }
    public int Rating { get; set; }  // 1-5
    public string Comment { get; set; } = string.Empty;
    public string ReviewerName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
}
```

```csharp
// Data/ProductDbContext.cs
using Microsoft.EntityFrameworkCore;
using ProductApp.Models;

namespace ProductApp.Data;

public class ProductDbContext : DbContext
{
    public DbSet<Category> Categories { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<Review> Reviews { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlite("Data Source=products.db");
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Category>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.Name).IsRequired().HasMaxLength(100);
        });
        
        modelBuilder.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Name).IsRequired().HasMaxLength(200);
            e.Property(p => p.Sku).IsRequired().HasMaxLength(50);
            e.Property(p => p.Price).HasColumnType("decimal(18,2)");
            e.HasIndex(p => p.Sku).IsUnique();
            
            e.HasOne(p => p.Category)
             .WithMany(c => c.Products)
             .HasForeignKey(p => p.CategoryId);
        });
        
        modelBuilder.Entity<Review>(e =>
        {
            e.HasKey(r => r.Id);
            
            e.HasOne(r => r.Product)
             .WithMany(p => p.Reviews)
             .HasForeignKey(r => r.ProductId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

```csharp
// Services/ProductQueryService.cs
using Microsoft.EntityFrameworkCore;
using ProductApp.Data;
using ProductApp.Models;

namespace ProductApp.Services;

public class ProductQueryService
{
    private readonly ProductDbContext _context;
    
    public ProductQueryService(ProductDbContext context)
    {
        _context = context;
    }
    
    // Basic search
    public async Task<List<Product>> SearchProductsAsync(
        string? keyword = null,
        int? categoryId = null,
        decimal? minPrice = null,
        decimal? maxPrice = null,
        bool activeOnly = true)
    {
        var query = _context.Products
            .Include(p => p.Category)
            .AsQueryable();
        
        if (activeOnly)
            query = query.Where(p => p.IsActive);
        
        if (!string.IsNullOrWhiteSpace(keyword))
            query = query.Where(p => 
                p.Name.Contains(keyword) || 
                p.Sku.Contains(keyword));
        
        if (categoryId.HasValue)
            query = query.Where(p => p.CategoryId == categoryId);
        
        if (minPrice.HasValue)
            query = query.Where(p => p.Price >= minPrice);
        
        if (maxPrice.HasValue)
            query = query.Where(p => p.Price <= maxPrice);
        
        return await query
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    // Paged results
    public async Task<PagedResult<ProductDto>> GetPagedProductsAsync(
        int page = 1,
        int pageSize = 10,
        string sortBy = "Name",
        bool ascending = true)
    {
        var query = _context.Products
            .Where(p => p.IsActive)
            .Select(p => new ProductDto(
                p.Id,
                p.Name,
                p.Price,
                p.Category.Name,
                p.Reviews.Any() 
                    ? Math.Round(p.Reviews.Average(r => r.Rating), 1)
                    : 0,
                p.Reviews.Count,
                p.StockQuantity
            ));
        
        // Dynamic sort
        query = (sortBy.ToLower(), ascending) switch
        {
            ("name", true) => query.OrderBy(p => p.Name),
            ("name", false) => query.OrderByDescending(p => p.Name),
            ("price", true) => query.OrderBy(p => p.Price),
            ("price", false) => query.OrderByDescending(p => p.Price),
            _ => query.OrderBy(p => p.Name)
        };
        
        var total = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
        
        return new PagedResult<ProductDto>
        {
            Items = items,
            TotalCount = total,
            Page = page,
            PageSize = pageSize
        };
    }
    
    // Top rated products
    public async Task<List<ProductRatingDto>> GetTopRatedProductsAsync(int count = 10)
    {
        return await _context.Products
            .Where(p => p.IsActive && p.Reviews.Any())
            .Select(p => new ProductRatingDto(
                p.Id,
                p.Name,
                p.Price,
                Math.Round(p.Reviews.Average(r => r.Rating), 2),
                p.Reviews.Count
            ))
            .OrderByDescending(p => p.AverageRating)
            .ThenByDescending(p => p.ReviewCount)
            .Take(count)
            .ToListAsync();
    }
    
    // Low stock products
    public async Task<List<Product>> GetLowStockProductsAsync(int threshold = 10)
    {
        return await _context.Products
            .Include(p => p.Category)
            .Where(p => p.IsActive && p.StockQuantity <= threshold)
            .OrderBy(p => p.StockQuantity)
            .ToListAsync();
    }
    
    // Category statistics
    public async Task<List<CategoryStatsDto>> GetCategoryStatsAsync()
    {
        return await _context.Categories
            .Select(c => new CategoryStatsDto(
                c.Id,
                c.Name,
                c.Products.Count(p => p.IsActive),
                c.Products.Where(p => p.IsActive).Any() 
                    ? c.Products.Where(p => p.IsActive).Average(p => p.Price)
                    : 0,
                c.Products.Where(p => p.IsActive).Any() 
                    ? c.Products.Where(p => p.IsActive).Min(p => p.Price)
                    : 0,
                c.Products.Where(p => p.IsActive).Any() 
                    ? c.Products.Where(p => p.IsActive).Max(p => p.Price)
                    : 0
            ))
            .OrderByDescending(c => c.ProductCount)
            .ToListAsync();
    }
    
    // Full text search with details
    public async Task<List<ProductDetailDto>> GetProductDetailsAsync(int productId)
    {
        return await _context.Products
            .Where(p => p.Id == productId)
            .Select(p => new ProductDetailDto(
                p.Id,
                p.Name,
                p.Sku,
                p.Price,
                p.StockQuantity,
                p.Category.Name,
                p.Reviews
                    .OrderByDescending(r => r.CreatedAt)
                    .Select(r => new ReviewDto(
                        r.Rating,
                        r.Comment,
                        r.ReviewerName,
                        r.CreatedAt
                    ))
                    .ToList()
            ))
            .ToListAsync();
    }
}

// DTOs
public record ProductDto(
    int Id,
    string Name,
    decimal Price,
    string CategoryName,
    double AverageRating,
    int ReviewCount,
    int StockQuantity);

public record ProductRatingDto(
    int Id,
    string Name,
    decimal Price,
    double AverageRating,
    int ReviewCount);

public record CategoryStatsDto(
    int Id,
    string Name,
    int ProductCount,
    double AvgPrice,
    decimal MinPrice,
    decimal MaxPrice);

public record ProductDetailDto(
    int Id,
    string Name,
    string Sku,
    decimal Price,
    int StockQuantity,
    string CategoryName,
    List<ReviewDto> Reviews);

public record ReviewDto(
    int Rating,
    string Comment,
    string ReviewerName,
    DateTime CreatedAt);
```

```csharp
// Program.cs
using ProductApp.Data;
using ProductApp.Models;
using ProductApp.Services;
using Microsoft.EntityFrameworkCore;

Console.WriteLine("=== Product Query System ===\n");

await using var context = new ProductDbContext();
await context.Database.EnsureDeletedAsync();
await context.Database.EnsureCreatedAsync();

// Seed data
await SeedDataAsync(context);

var service = new ProductQueryService(context);

// 1. Basic search
Console.WriteLine("1. Search 'phone' products:");
var phones = await service.SearchProductsAsync(keyword: "phone");
foreach (var p in phones)
    Console.WriteLine($"   {p.Name} ({p.Category.Name}) - ฿{p.Price:N0}");

// 2. Filter by price range
Console.WriteLine("\n2. Products ฿1,000 - ฿50,000:");
var midRange = await service.SearchProductsAsync(minPrice: 1000, maxPrice: 50000);
foreach (var p in midRange)
    Console.WriteLine($"   {p.Name} - ฿{p.Price:N0}");

// 3. Paged results
Console.WriteLine("\n3. Paged Products (Page 1, Size 3):");
var paged = await service.GetPagedProductsAsync(page: 1, pageSize: 3, sortBy: "Price", ascending: false);
Console.WriteLine($"   Total: {paged.TotalCount}, Pages: {paged.TotalPages}");
foreach (var p in paged.Items)
    Console.WriteLine($"   {p.Name} - ฿{p.Price:N0} ★{p.AverageRating}");

// 4. Top rated
Console.WriteLine("\n4. Top Rated Products:");
var topRated = await service.GetTopRatedProductsAsync(count: 3);
foreach (var p in topRated)
    Console.WriteLine($"   {p.Name} - ★{p.AverageRating} ({p.ReviewCount} reviews)");

// 5. Low stock
Console.WriteLine("\n5. Low Stock Products (< 5):");
var lowStock = await service.GetLowStockProductsAsync(threshold: 5);
foreach (var p in lowStock)
    Console.WriteLine($"   {p.Name} - {p.StockQuantity} items left");

// 6. Category stats
Console.WriteLine("\n6. Category Statistics:");
var stats = await service.GetCategoryStatsAsync();
foreach (var s in stats)
    Console.WriteLine($"   {s.Name}: {s.ProductCount} products, avg ฿{s.AvgPrice:N0}");

Console.WriteLine("\n=== Done! ===");

// Seed data function
static async Task SeedDataAsync(ProductDbContext ctx)
{
    var categories = new[]
    {
        new Category { Name = "Smartphones" },
        new Category { Name = "Laptops" },
        new Category { Name = "Accessories" }
    };
    ctx.Categories.AddRange(categories);
    await ctx.SaveChangesAsync();
    
    var products = new[]
    {
        new Product { Name = "iPhone 16 Pro", Sku = "IP16PRO", Price = 45900m, StockQuantity = 50, CategoryId = categories[0].Id },
        new Product { Name = "Samsung S24", Sku = "SS24", Price = 35900m, StockQuantity = 3, CategoryId = categories[0].Id },
        new Product { Name = "Google Pixel 9", Sku = "GP9", Price = 28900m, StockQuantity = 20, CategoryId = categories[0].Id },
        new Product { Name = "MacBook Pro 14", Sku = "MBP14", Price = 74900m, StockQuantity = 15, CategoryId = categories[1].Id },
        new Product { Name = "Dell XPS 15", Sku = "DXPS15", Price = 55000m, StockQuantity = 4, CategoryId = categories[1].Id },
        new Product { Name = "AirPods Pro", Sku = "APP3", Price = 8900m, StockQuantity = 2, CategoryId = categories[2].Id },
        new Product { Name = "USB-C Hub", Sku = "USBCH", Price = 1290m, StockQuantity = 100, CategoryId = categories[2].Id }
    };
    ctx.Products.AddRange(products);
    await ctx.SaveChangesAsync();
    
    var reviews = new[]
    {
        new Review { ProductId = products[0].Id, Rating = 5, Comment = "ดีมาก!", ReviewerName = "สมชาย" },
        new Review { ProductId = products[0].Id, Rating = 4, Comment = "ราคาแพงไปหน่อย", ReviewerName = "สมหญิง" },
        new Review { ProductId = products[1].Id, Rating = 4, Comment = "กล้องดี", ReviewerName = "มนตรี" },
        new Review { ProductId = products[3].Id, Rating = 5, Comment = "เร็วมากๆ", ReviewerName = "วิไล" },
        new Review { ProductId = products[3].Id, Rating = 5, Comment = "แบตดีเยี่ยม", ReviewerName = "ประสิทธิ์" }
    };
    ctx.Reviews.AddRange(reviews);
    await ctx.SaveChangesAsync();
}
```

---

## Exercises

### แบบฝึกหัดที่ 1: Advanced Filtering
เขียน method `GetProductsByFiltersAsync` ที่รองรับ:
- Filter ด้วย multiple categories (ส่ง list of categoryIds)
- Filter ด้วย minimum average rating
- Include/exclude out-of-stock products
- Sort ด้วย multiple fields

### แบบฝึกหัดที่ 2: Search Autocomplete
เขียน `GetProductSuggestionsAsync(string prefix)` ที่:
- ค้นหา products ที่ชื่อขึ้นต้นด้วย prefix
- Return แค่ Id, Name, Price
- Limit 10 รายการ
- เร็วที่สุดเท่าที่จะเป็นไปได้

### แบบฝึกหัดที่ 3: Related Products
เขียน `GetRelatedProductsAsync(int productId, int count = 5)` ที่:
- หา products ที่อยู่ category เดียวกัน
- ไม่รวม product ตัวเอง
- Sort ด้วย average rating
- Include category และ average rating

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **LINQ with EF Core** ทั้ง Query Syntax และ Method Syntax
2. **IQueryable vs IEnumerable** และความแตกต่างด้าน performance
3. **Eager Loading** ด้วย Include/ThenInclude
4. **Lazy Loading** สะดวกแต่ต้องระวัง N+1
5. **Explicit Loading** โหลดเมื่อต้องการ
6. **Raw SQL** ด้วย FromSqlRaw และ ExecuteSqlRaw
7. **Filtering, Sorting, Pagination** แบบ dynamic

---

## Part ถัดไป

ใน **Part 054** เราจะเรียนรู้เกี่ยวกับ **EF Core: CRUD Operations** อย่างละเอียด:
- Add, AddRange
- Update, UpdateRange
- Remove, RemoveRange
- SaveChanges และ SaveChangesAsync
- Transactions
- Optimistic Concurrency

---

*Part 053/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
