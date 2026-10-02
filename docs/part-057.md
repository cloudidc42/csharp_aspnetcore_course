# Part 057: Repository Pattern กับ EF Core

## เนื้อหาใน Part นี้
- Repository Pattern คืออะไร
- Repository Interface
- Generic Repository
- Unit of Work Pattern
- Dependency Injection กับ Repositories
- Testing Repositories
- โปรแกรมตัวอย่าง: Clean Repository Layer

---

## Repository Pattern คืออะไร

**Repository Pattern** คือ design pattern ที่แยก logic การเข้าถึงข้อมูล (data access) ออกจาก business logic

```
Business Layer  →  Repository Interface  →  Repository Implementation  →  Database
    (Service)         (IProductRepository)    (ProductRepository : EF Core)
```

### ข้อดีของ Repository Pattern

1. **Abstraction**: แยก business logic ออกจาก data access
2. **Testability**: Mock repositories ง่ายในการเขียน unit tests
3. **Maintainability**: เปลี่ยน ORM หรือ database ได้โดยไม่แก้ business logic
4. **Single Responsibility**: แต่ละ class มีหน้าที่เดียว

### ข้อเสีย

1. เพิ่ม complexity
2. อาจเป็น "leaky abstraction" ถ้า design ไม่ดี
3. EF Core DbContext เองก็เป็น Unit of Work อยู่แล้ว

---

## Models ที่ใช้ใน Part นี้

```csharp
// Models/Product.cs
namespace RepositoryApp.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    public List<Product> Products { get; set; } = [];
}

public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int CustomerId { get; set; }
    public List<OrderItem> Items { get; set; } = [];
}

public class OrderItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }
```

---

## Repository Interface

```csharp
// Repositories/Interfaces/IRepository.cs
namespace RepositoryApp.Repositories;

public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(int id);
    Task<bool> ExistsAsync(int id);
}
```

```csharp
// Repositories/Interfaces/IProductRepository.cs
using RepositoryApp.Models;

namespace RepositoryApp.Repositories;

public interface IProductRepository : IRepository<Product>
{
    Task<Product?> GetBySkuAsync(string sku);
    Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId);
    Task<IEnumerable<Product>> GetActiveProductsAsync();
    Task<IEnumerable<Product>> SearchAsync(string keyword);
    Task<PagedResult<Product>> GetPagedAsync(int page, int pageSize, string? sortBy = null);
    Task<bool> IsSkuUniqueAsync(string sku, int? excludeId = null);
}

public class PagedResult<T>
{
    public IEnumerable<T> Items { get; set; } = [];
    public int TotalCount { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
}
```

```csharp
// Repositories/Interfaces/IOrderRepository.cs
using RepositoryApp.Models;

namespace RepositoryApp.Repositories;

public interface IOrderRepository : IRepository<Order>
{
    Task<Order?> GetByOrderNumberAsync(string orderNumber);
    Task<IEnumerable<Order>> GetByCustomerAsync(int customerId);
    Task<IEnumerable<Order>> GetByStatusAsync(OrderStatus status);
    Task<IEnumerable<Order>> GetOrdersWithItemsAsync(int customerId);
    Task<decimal> GetTotalRevenueAsync(DateTime from, DateTime to);
}
```

---

## Generic Repository Implementation

```csharp
// Repositories/GenericRepository.cs
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Data;

namespace RepositoryApp.Repositories;

public class GenericRepository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext _context;
    protected readonly DbSet<T> _dbSet;
    
    public GenericRepository(AppDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }
    
    public virtual async Task<T?> GetByIdAsync(int id)
    {
        return await _dbSet.FindAsync(id);
    }
    
    public virtual async Task<IEnumerable<T>> GetAllAsync()
    {
        return await _dbSet.AsNoTracking().ToListAsync();
    }
    
    public virtual async Task<T> AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
        return entity;
    }
    
    public virtual Task UpdateAsync(T entity)
    {
        _dbSet.Update(entity);
        return Task.CompletedTask;
    }
    
    public virtual async Task DeleteAsync(int id)
    {
        var entity = await _dbSet.FindAsync(id);
        if (entity != null)
            _dbSet.Remove(entity);
    }
    
    public virtual async Task<bool> ExistsAsync(int id)
    {
        var entity = await _dbSet.FindAsync(id);
        return entity != null;
    }
    
    // Helper: Query with conditions
    protected IQueryable<T> Query(bool asNoTracking = true)
    {
        return asNoTracking 
            ? _dbSet.AsNoTracking() 
            : _dbSet.AsQueryable();
    }
}
```

---

## Specific Repository Implementations

```csharp
// Repositories/ProductRepository.cs
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Data;
using RepositoryApp.Models;

namespace RepositoryApp.Repositories;

public class ProductRepository : GenericRepository<Product>, IProductRepository
{
    public ProductRepository(AppDbContext context) : base(context) { }
    
    public override async Task<Product?> GetByIdAsync(int id)
    {
        return await _dbSet
            .AsNoTracking()
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id);
    }
    
    public override async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _dbSet
            .AsNoTracking()
            .Include(p => p.Category)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<Product?> GetBySkuAsync(string sku)
    {
        return await _dbSet
            .AsNoTracking()
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Sku == sku);
    }
    
    public async Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.CategoryId == categoryId)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<IEnumerable<Product>> GetActiveProductsAsync()
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.IsActive)
            .Include(p => p.Category)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<IEnumerable<Product>> SearchAsync(string keyword)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.Name.Contains(keyword) || p.Sku.Contains(keyword))
            .Include(p => p.Category)
            .OrderBy(p => p.Name)
            .ToListAsync();
    }
    
    public async Task<PagedResult<Product>> GetPagedAsync(
        int page, 
        int pageSize, 
        string? sortBy = null)
    {
        var query = _dbSet
            .AsNoTracking()
            .Include(p => p.Category)
            .Where(p => p.IsActive);
        
        query = sortBy switch
        {
            "price_asc" => query.OrderBy(p => p.Price),
            "price_desc" => query.OrderByDescending(p => p.Price),
            "name_desc" => query.OrderByDescending(p => p.Name),
            _ => query.OrderBy(p => p.Name)
        };
        
        var total = await query.CountAsync();
        var items = await query
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
        
        return new PagedResult<Product>
        {
            Items = items,
            TotalCount = total,
            Page = page,
            PageSize = pageSize
        };
    }
    
    public async Task<bool> IsSkuUniqueAsync(string sku, int? excludeId = null)
    {
        var query = _dbSet.Where(p => p.Sku == sku);
        if (excludeId.HasValue)
            query = query.Where(p => p.Id != excludeId);
        
        return !await query.AnyAsync();
    }
}
```

```csharp
// Repositories/OrderRepository.cs
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Data;
using RepositoryApp.Models;

namespace RepositoryApp.Repositories;

public class OrderRepository : GenericRepository<Order>, IOrderRepository
{
    public OrderRepository(AppDbContext context) : base(context) { }
    
    public override async Task<Order?> GetByIdAsync(int id)
    {
        return await _dbSet
            .AsNoTracking()
            .Include(o => o.Items)
                .ThenInclude(i => i.Product)
            .FirstOrDefaultAsync(o => o.Id == id);
    }
    
    public async Task<Order?> GetByOrderNumberAsync(string orderNumber)
    {
        return await _dbSet
            .AsNoTracking()
            .Include(o => o.Items)
                .ThenInclude(i => i.Product)
            .FirstOrDefaultAsync(o => o.OrderNumber == orderNumber);
    }
    
    public async Task<IEnumerable<Order>> GetByCustomerAsync(int customerId)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync();
    }
    
    public async Task<IEnumerable<Order>> GetByStatusAsync(OrderStatus status)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(o => o.Status == status)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync();
    }
    
    public async Task<IEnumerable<Order>> GetOrdersWithItemsAsync(int customerId)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .Include(o => o.Items)
                .ThenInclude(i => i.Product)
                    .ThenInclude(p => p.Category)
            .OrderByDescending(o => o.CreatedAt)
            .ToListAsync();
    }
    
    public async Task<decimal> GetTotalRevenueAsync(DateTime from, DateTime to)
    {
        return await _dbSet
            .Where(o => o.Status != OrderStatus.Cancelled &&
                        o.CreatedAt >= from &&
                        o.CreatedAt <= to)
            .SumAsync(o => o.Total);
    }
}
```

---

## Unit of Work Pattern

Unit of Work รวม repositories และ SaveChanges() ไว้ด้วยกัน:

```csharp
// Repositories/Interfaces/IUnitOfWork.cs
namespace RepositoryApp.Repositories;

public interface IUnitOfWork : IDisposable, IAsyncDisposable
{
    IProductRepository Products { get; }
    IOrderRepository Orders { get; }
    ICategoryRepository Categories { get; }
    
    Task<int> SaveChangesAsync();
    Task BeginTransactionAsync();
    Task CommitTransactionAsync();
    Task RollbackTransactionAsync();
}
```

```csharp
// Repositories/UnitOfWork.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Storage;
using RepositoryApp.Data;

namespace RepositoryApp.Repositories;

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private IDbContextTransaction? _transaction;
    
    private IProductRepository? _products;
    private IOrderRepository? _orders;
    private ICategoryRepository? _categories;
    
    public UnitOfWork(AppDbContext context)
    {
        _context = context;
    }
    
    // Lazy initialization ของ repositories
    public IProductRepository Products => 
        _products ??= new ProductRepository(_context);
    
    public IOrderRepository Orders => 
        _orders ??= new OrderRepository(_context);
    
    public ICategoryRepository Categories => 
        _categories ??= new CategoryRepository(_context);
    
    public async Task<int> SaveChangesAsync()
    {
        return await _context.SaveChangesAsync();
    }
    
    public async Task BeginTransactionAsync()
    {
        _transaction = await _context.Database.BeginTransactionAsync();
    }
    
    public async Task CommitTransactionAsync()
    {
        if (_transaction == null)
            throw new InvalidOperationException("No transaction in progress");
        
        await _transaction.CommitAsync();
        await _transaction.DisposeAsync();
        _transaction = null;
    }
    
    public async Task RollbackTransactionAsync()
    {
        if (_transaction == null) return;
        
        await _transaction.RollbackAsync();
        await _transaction.DisposeAsync();
        _transaction = null;
    }
    
    public void Dispose()
    {
        _transaction?.Dispose();
        _context.Dispose();
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_transaction != null)
            await _transaction.DisposeAsync();
        await _context.DisposeAsync();
    }
}
```

---

## Category Repository

```csharp
// Repositories/Interfaces/ICategoryRepository.cs
using RepositoryApp.Models;

namespace RepositoryApp.Repositories;

public interface ICategoryRepository : IRepository<Category>
{
    Task<Category?> GetBySlugAsync(string slug);
    Task<IEnumerable<Category>> GetWithProductCountAsync();
    Task<bool> HasProductsAsync(int categoryId);
}
```

```csharp
// Repositories/CategoryRepository.cs
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Data;
using RepositoryApp.Models;

namespace RepositoryApp.Repositories;

public class CategoryRepository : GenericRepository<Category>, ICategoryRepository
{
    public CategoryRepository(AppDbContext context) : base(context) { }
    
    public async Task<Category?> GetBySlugAsync(string slug)
    {
        return await _dbSet
            .AsNoTracking()
            .FirstOrDefaultAsync(c => c.Slug == slug);
    }
    
    public async Task<IEnumerable<Category>> GetWithProductCountAsync()
    {
        return await _dbSet
            .AsNoTracking()
            .Select(c => new Category
            {
                Id = c.Id,
                Name = c.Name,
                Slug = c.Slug,
                Products = c.Products.Where(p => p.IsActive).ToList()
            })
            .OrderBy(c => c.Name)
            .ToListAsync();
    }
    
    public async Task<bool> HasProductsAsync(int categoryId)
    {
        return await _context.Set<Product>()
            .AnyAsync(p => p.CategoryId == categoryId);
    }
}
```

---

## Dependency Injection

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Models;

namespace RepositoryApp.Data;

public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    
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
            e.HasIndex(p => p.Sku).IsUnique();
            
            e.HasOne(p => p.Category)
             .WithMany(c => c.Products)
             .HasForeignKey(p => p.CategoryId);
        });
        
        modelBuilder.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.Property(o => o.OrderNumber).IsRequired().HasMaxLength(50);
            e.HasIndex(o => o.OrderNumber).IsUnique();
            e.Property(o => o.Total).HasColumnType("decimal(18,2)");
        });
        
        modelBuilder.Entity<OrderItem>(e =>
        {
            e.HasKey(i => i.Id);
            e.Property(i => i.UnitPrice).HasColumnType("decimal(18,2)");
            
            e.HasOne(i => i.Product)
             .WithMany()
             .HasForeignKey(i => i.ProductId);
        });
    }
}
```

```csharp
// Program.cs (ASP.NET Core)
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Data;
using RepositoryApp.Repositories;

var builder = WebApplication.CreateBuilder(args);

// Register DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(
        builder.Configuration.GetConnectionString("Default") 
        ?? "Data Source=app.db"));

// Register Repositories
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<ICategoryRepository, CategoryRepository>();

// Register Unit of Work
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();

// Register Services
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IOrderService, OrderService>();
```

---

## Service Layer กับ Repository

```csharp
// Services/IProductService.cs
using RepositoryApp.Models;
using RepositoryApp.Repositories;

namespace RepositoryApp.Services;

public interface IProductService
{
    Task<Product?> GetProductAsync(int id);
    Task<PagedResult<Product>> GetProductsAsync(int page, int pageSize);
    Task<Product> CreateProductAsync(CreateProductRequest request);
    Task UpdateProductAsync(int id, UpdateProductRequest request);
    Task DeleteProductAsync(int id);
}

public record CreateProductRequest(
    string Name,
    string Sku,
    decimal Price,
    int StockQuantity,
    int CategoryId);

public record UpdateProductRequest(
    string Name,
    decimal Price,
    int StockQuantity,
    bool IsActive);
```

```csharp
// Services/ProductService.cs
using RepositoryApp.Models;
using RepositoryApp.Repositories;

namespace RepositoryApp.Services;

public class ProductService : IProductService
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly ILogger<ProductService> _logger;
    
    public ProductService(IUnitOfWork unitOfWork, ILogger<ProductService> logger)
    {
        _unitOfWork = unitOfWork;
        _logger = logger;
    }
    
    public async Task<Product?> GetProductAsync(int id)
    {
        return await _unitOfWork.Products.GetByIdAsync(id);
    }
    
    public async Task<PagedResult<Product>> GetProductsAsync(int page, int pageSize)
    {
        return await _unitOfWork.Products.GetPagedAsync(page, pageSize);
    }
    
    public async Task<Product> CreateProductAsync(CreateProductRequest request)
    {
        // Business validation
        if (!await _unitOfWork.Products.IsSkuUniqueAsync(request.Sku))
            throw new InvalidOperationException($"SKU '{request.Sku}' already exists");
        
        if (!await _unitOfWork.Categories.ExistsAsync(request.CategoryId))
            throw new KeyNotFoundException($"Category {request.CategoryId} not found");
        
        var product = new Product
        {
            Name = request.Name,
            Sku = request.Sku,
            Price = request.Price,
            StockQuantity = request.StockQuantity,
            CategoryId = request.CategoryId
        };
        
        await _unitOfWork.Products.AddAsync(product);
        await _unitOfWork.SaveChangesAsync();
        
        _logger.LogInformation("Created product {Id}: {Name}", product.Id, product.Name);
        
        return product;
    }
    
    public async Task UpdateProductAsync(int id, UpdateProductRequest request)
    {
        var product = await _unitOfWork.Products.GetByIdAsync(id)
            ?? throw new KeyNotFoundException($"Product {id} not found");
        
        product.Name = request.Name;
        product.Price = request.Price;
        product.StockQuantity = request.StockQuantity;
        product.IsActive = request.IsActive;
        product.UpdatedAt = DateTime.UtcNow;
        
        await _unitOfWork.Products.UpdateAsync(product);
        await _unitOfWork.SaveChangesAsync();
        
        _logger.LogInformation("Updated product {Id}", id);
    }
    
    public async Task DeleteProductAsync(int id)
    {
        var product = await _unitOfWork.Products.GetByIdAsync(id)
            ?? throw new KeyNotFoundException($"Product {id} not found");
        
        // Business rule: ไม่ลบถ้ามี orders
        var hasOrders = await _unitOfWork.Orders.ExistsAsync(id);
        
        if (hasOrders)
        {
            // Soft delete
            product.IsActive = false;
            await _unitOfWork.Products.UpdateAsync(product);
        }
        else
        {
            await _unitOfWork.Products.DeleteAsync(id);
        }
        
        await _unitOfWork.SaveChangesAsync();
    }
}
```

```csharp
// Services/OrderService.cs
using RepositoryApp.Models;
using RepositoryApp.Repositories;

namespace RepositoryApp.Services;

public class OrderService
{
    private readonly IUnitOfWork _unitOfWork;
    
    public OrderService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }
    
    public async Task<Order> PlaceOrderAsync(
        int customerId,
        List<(int ProductId, int Quantity)> items)
    {
        await _unitOfWork.BeginTransactionAsync();
        
        try
        {
            var order = new Order
            {
                OrderNumber = $"ORD-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid():N[..8]}",
                CustomerId = customerId,
                Status = OrderStatus.Pending
            };
            
            decimal total = 0;
            foreach (var (productId, quantity) in items)
            {
                var product = await _unitOfWork.Products.GetByIdAsync(productId)
                    ?? throw new KeyNotFoundException($"Product {productId} not found");
                
                if (product.StockQuantity < quantity)
                    throw new InvalidOperationException(
                        $"Insufficient stock for {product.Name}");
                
                // Reduce stock
                product.StockQuantity -= quantity;
                await _unitOfWork.Products.UpdateAsync(product);
                
                var lineTotal = product.Price * quantity;
                total += lineTotal;
                
                order.Items.Add(new OrderItem
                {
                    ProductId = productId,
                    Quantity = quantity,
                    UnitPrice = product.Price
                });
            }
            
            order.Total = total;
            await _unitOfWork.Orders.AddAsync(order);
            await _unitOfWork.SaveChangesAsync();
            
            await _unitOfWork.CommitTransactionAsync();
            
            return order;
        }
        catch
        {
            await _unitOfWork.RollbackTransactionAsync();
            throw;
        }
    }
}
```

---

## Testing Repositories

### Unit Test ด้วย In-Memory Database

```csharp
// Tests/ProductRepositoryTests.cs
using Microsoft.EntityFrameworkCore;
using RepositoryApp.Data;
using RepositoryApp.Models;
using RepositoryApp.Repositories;

namespace RepositoryApp.Tests;

public class ProductRepositoryTests : IDisposable
{
    private readonly AppDbContext _context;
    private readonly ProductRepository _repository;
    
    public ProductRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        
        _context = new AppDbContext(options);
        _repository = new ProductRepository(_context);
        
        // Seed test data
        SeedTestData();
    }
    
    private void SeedTestData()
    {
        var category = new Category { Id = 1, Name = "Electronics", Slug = "electronics" };
        _context.Categories.Add(category);
        
        var products = new[]
        {
            new Product 
            { 
                Id = 1, Name = "iPhone 16", Sku = "IP16", Price = 45900m, 
                StockQuantity = 10, CategoryId = 1, IsActive = true 
            },
            new Product 
            { 
                Id = 2, Name = "MacBook Pro", Sku = "MBP14", Price = 74900m, 
                StockQuantity = 5, CategoryId = 1, IsActive = true 
            },
            new Product 
            { 
                Id = 3, Name = "Old Phone", Sku = "OLD001", Price = 1000m, 
                StockQuantity = 0, CategoryId = 1, IsActive = false 
            }
        };
        _context.Products.AddRange(products);
        _context.SaveChanges();
    }
    
    [Fact]
    public async Task GetByIdAsync_ExistingId_ReturnsProduct()
    {
        var product = await _repository.GetByIdAsync(1);
        
        Assert.NotNull(product);
        Assert.Equal("iPhone 16", product.Name);
        Assert.Equal("IP16", product.Sku);
    }
    
    [Fact]
    public async Task GetByIdAsync_NonExistingId_ReturnsNull()
    {
        var product = await _repository.GetByIdAsync(999);
        
        Assert.Null(product);
    }
    
    [Fact]
    public async Task GetBySkuAsync_ExistingSku_ReturnsProduct()
    {
        var product = await _repository.GetBySkuAsync("MBP14");
        
        Assert.NotNull(product);
        Assert.Equal("MacBook Pro", product.Name);
    }
    
    [Fact]
    public async Task GetActiveProductsAsync_ReturnsOnlyActiveProducts()
    {
        var products = await _repository.GetActiveProductsAsync();
        
        Assert.Equal(2, products.Count());
        Assert.All(products, p => Assert.True(p.IsActive));
    }
    
    [Fact]
    public async Task SearchAsync_WithKeyword_ReturnsMatchingProducts()
    {
        var results = await _repository.SearchAsync("book");
        
        Assert.Single(results);
        Assert.Equal("MacBook Pro", results.First().Name);
    }
    
    [Fact]
    public async Task AddAsync_NewProduct_IsPersistedAfterSave()
    {
        var newProduct = new Product
        {
            Name = "AirPods Pro",
            Sku = "APP3",
            Price = 8900m,
            StockQuantity = 20,
            CategoryId = 1
        };
        
        await _repository.AddAsync(newProduct);
        await _context.SaveChangesAsync();
        
        var saved = await _repository.GetBySkuAsync("APP3");
        Assert.NotNull(saved);
        Assert.Equal("AirPods Pro", saved.Name);
    }
    
    [Fact]
    public async Task UpdateAsync_ExistingProduct_ChangesAreSaved()
    {
        var product = await _repository.GetByIdAsync(1);
        Assert.NotNull(product);
        
        product.Price = 49900m;
        await _repository.UpdateAsync(product);
        await _context.SaveChangesAsync();
        
        var updated = await _repository.GetByIdAsync(1);
        Assert.Equal(49900m, updated?.Price);
    }
    
    [Fact]
    public async Task DeleteAsync_ExistingProduct_IsRemovedAfterSave()
    {
        await _repository.DeleteAsync(1);
        await _context.SaveChangesAsync();
        
        var deleted = await _repository.GetByIdAsync(1);
        Assert.Null(deleted);
    }
    
    [Fact]
    public async Task IsSkuUniqueAsync_ExistingSku_ReturnsFalse()
    {
        var isUnique = await _repository.IsSkuUniqueAsync("IP16");
        Assert.False(isUnique);
    }
    
    [Fact]
    public async Task IsSkuUniqueAsync_NewSku_ReturnsTrue()
    {
        var isUnique = await _repository.IsSkuUniqueAsync("NEWSKU999");
        Assert.True(isUnique);
    }
    
    [Fact]
    public async Task GetPagedAsync_ReturnsCorrectPage()
    {
        var result = await _repository.GetPagedAsync(page: 1, pageSize: 2);
        
        Assert.Equal(2, result.Items.Count());
        Assert.Equal(2, result.TotalCount);  // 2 active products
        Assert.Equal(1, result.TotalPages);
    }
    
    public void Dispose()
    {
        _context.Dispose();
    }
}
```

### Test ด้วย Moq (Mock Repositories)

```csharp
// Tests/ProductServiceTests.cs
using Moq;
using RepositoryApp.Models;
using RepositoryApp.Repositories;
using RepositoryApp.Services;
using Microsoft.Extensions.Logging;

namespace RepositoryApp.Tests;

public class ProductServiceTests
{
    private readonly Mock<IUnitOfWork> _mockUoW;
    private readonly Mock<IProductRepository> _mockProductRepo;
    private readonly Mock<ICategoryRepository> _mockCategoryRepo;
    private readonly Mock<ILogger<ProductService>> _mockLogger;
    private readonly ProductService _service;
    
    public ProductServiceTests()
    {
        _mockUoW = new Mock<IUnitOfWork>();
        _mockProductRepo = new Mock<IProductRepository>();
        _mockCategoryRepo = new Mock<ICategoryRepository>();
        _mockLogger = new Mock<ILogger<ProductService>>();
        
        _mockUoW.Setup(u => u.Products).Returns(_mockProductRepo.Object);
        _mockUoW.Setup(u => u.Categories).Returns(_mockCategoryRepo.Object);
        
        _service = new ProductService(_mockUoW.Object, _mockLogger.Object);
    }
    
    [Fact]
    public async Task CreateProductAsync_ValidRequest_CreatesProduct()
    {
        // Arrange
        var request = new CreateProductRequest(
            "Test Product", "TST001", 100m, 10, 1);
        
        _mockProductRepo
            .Setup(r => r.IsSkuUniqueAsync("TST001", null))
            .ReturnsAsync(true);
        
        _mockCategoryRepo
            .Setup(r => r.ExistsAsync(1))
            .ReturnsAsync(true);
        
        _mockProductRepo
            .Setup(r => r.AddAsync(It.IsAny<Product>()))
            .ReturnsAsync((Product p) => p);
        
        _mockUoW
            .Setup(u => u.SaveChangesAsync())
            .ReturnsAsync(1);
        
        // Act
        var result = await _service.CreateProductAsync(request);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal("Test Product", result.Name);
        Assert.Equal("TST001", result.Sku);
        
        _mockProductRepo.Verify(r => r.AddAsync(It.IsAny<Product>()), Times.Once);
        _mockUoW.Verify(u => u.SaveChangesAsync(), Times.Once);
    }
    
    [Fact]
    public async Task CreateProductAsync_DuplicateSku_ThrowsException()
    {
        // Arrange
        var request = new CreateProductRequest(
            "Test", "EXISTING", 100m, 10, 1);
        
        _mockProductRepo
            .Setup(r => r.IsSkuUniqueAsync("EXISTING", null))
            .ReturnsAsync(false);
        
        // Act & Assert
        await Assert.ThrowsAsync<InvalidOperationException>(
            () => _service.CreateProductAsync(request));
        
        _mockProductRepo.Verify(r => r.AddAsync(It.IsAny<Product>()), Times.Never);
    }
}
```

---

## โปรแกรมตัวอย่าง: Clean Repository Layer

```csharp
// Program.cs - Console app demo
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using RepositoryApp.Data;
using RepositoryApp.Models;
using RepositoryApp.Repositories;
using RepositoryApp.Services;

Console.WriteLine("=== Repository Pattern Demo ===\n");

// Setup DI
var services = new ServiceCollection();
services.AddLogging(b => b.AddConsole().SetMinimumLevel(LogLevel.Warning));
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=repository_demo.db"));

services.AddScoped<IProductRepository, ProductRepository>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<ICategoryRepository, CategoryRepository>();
services.AddScoped<IUnitOfWork, UnitOfWork>();
services.AddScoped<IProductService, ProductService>();
services.AddScoped<OrderService>();

var sp = services.BuildServiceProvider();

await using var scope = sp.CreateAsyncScope();

var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await context.Database.EnsureDeletedAsync();
await context.Database.EnsureCreatedAsync();

// Seed
var categoryRepo = scope.ServiceProvider.GetRequiredService<ICategoryRepository>();
var uow = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();

var electronics = new Category { Name = "Electronics", Slug = "electronics" };
var clothing = new Category { Name = "Clothing", Slug = "clothing" };

await uow.Categories.AddAsync(electronics);
await uow.Categories.AddAsync(clothing);
await uow.SaveChangesAsync();

// Use ProductService
var productService = scope.ServiceProvider.GetRequiredService<IProductService>();

Console.WriteLine("Creating products...");
var p1 = await productService.CreateProductAsync(new(
    "iPhone 16 Pro", "IP16PRO", 45900m, 50, electronics.Id));
var p2 = await productService.CreateProductAsync(new(
    "MacBook Pro 14", "MBP14", 74900m, 20, electronics.Id));
var p3 = await productService.CreateProductAsync(new(
    "T-Shirt Large", "TSH-L", 490m, 100, clothing.Id));

Console.WriteLine($"Created: {p1.Name}, {p2.Name}, {p3.Name}");

// Paged query
Console.WriteLine("\nPaged products:");
var paged = await productService.GetProductsAsync(page: 1, pageSize: 2);
Console.WriteLine($"Page 1/{paged.TotalPages} (Total: {paged.TotalCount})");
foreach (var p in paged.Items)
    Console.WriteLine($"  - {p.Name} ฿{p.Price:N0}");

// Search
Console.WriteLine("\nSearch 'pro':");
var searchResults = await uow.Products.SearchAsync("pro");
foreach (var p in searchResults)
    Console.WriteLine($"  - {p.Name} ({p.Sku})");

// Category products
Console.WriteLine($"\nElectronics products:");
var electronicProducts = await uow.Products.GetByCategoryAsync(electronics.Id);
foreach (var p in electronicProducts)
    Console.WriteLine($"  - {p.Name} ฿{p.Price:N0}");

// Order with transaction
Console.WriteLine("\nPlacing order...");
var orderService = scope.ServiceProvider.GetRequiredService<OrderService>();

var order = await orderService.PlaceOrderAsync(
    customerId: 1,
    items: [(p1.Id, 1), (p3.Id, 2)]);

Console.WriteLine($"Order placed: {order.OrderNumber}");
Console.WriteLine($"Total: ฿{order.Total:N0}");
Console.WriteLine($"Items: {order.Items.Count}");

// Check stock update
var updatedProduct = await uow.Products.GetByIdAsync(p1.Id);
Console.WriteLine($"\nStock after order - {updatedProduct?.Name}: {updatedProduct?.StockQuantity}");

// Update product
Console.WriteLine("\nUpdating product...");
await productService.UpdateProductAsync(p2.Id, new(
    "MacBook Pro 14 M4", 79900m, 15, true));
var updated = await productService.GetProductAsync(p2.Id);
Console.WriteLine($"Updated: {updated?.Name} ฿{updated?.Price:N0}");

// Duplicate SKU check
Console.WriteLine("\nTesting duplicate SKU...");
try
{
    await productService.CreateProductAsync(new(
        "Another iPhone", "IP16PRO", 1000m, 1, electronics.Id));
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Expected error: {ex.Message}");
}

Console.WriteLine("\n=== Done! ===");
```

---

## Exercises

### แบบฝึกหัดที่ 1: เพิ่ม Repository
สร้าง `ICustomerRepository` และ `CustomerRepository` ที่มี methods:
- `GetByEmailAsync(string email)`
- `GetTopCustomersAsync(int count)` - เรียงตาม total orders
- `GetCustomersWithPendingOrdersAsync()`
- `UpdatePurchaseTotalAsync(int customerId, decimal amount)`

### แบบฝึกหัดที่ 2: Generic Specifications
สร้าง Specification Pattern:
```csharp
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    Expression<Func<T, object>>? OrderBy { get; }
}

// ใช้งาน
var spec = new ActiveProductsInCategorySpec(categoryId: 1);
var products = await repository.FindAsync(spec);
```

### แบบฝึกหัดที่ 3: Integration Tests
เขียน integration tests ด้วย `TestContainers` หรือ SQLite in-memory:
1. Create product และ verify ใน database
2. Place order และ verify stock update
3. Test concurrency ด้วย 2 requests พร้อมกัน

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Repository Pattern**: แยก data access ออกจาก business logic
2. **Generic Repository**: Reusable base implementation
3. **Specific Repositories**: Custom methods สำหรับแต่ละ entity
4. **Unit of Work**: จัดการ repositories และ transactions รวมกัน
5. **Dependency Injection**: Register repositories กับ DI container
6. **Testing**: ทั้ง In-Memory DB และ Moq

---

## Part ถัดไป

ใน **Part 058** เราจะเริ่มเรียนเรื่อง **Authentication พื้นฐาน**:
- Authentication vs Authorization
- Cookie authentication
- JWT tokens เบื้องต้น
- Claims
- ASP.NET Core Identity

---

*Part 057/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
