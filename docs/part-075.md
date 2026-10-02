# Part 75: Mocking ด้วย Moq

## เนื้อหาใน Part นี้
- Mocking คืออะไร
- Setup methods
- Verify calls
- Mock interfaces
- Mock properties
- InMemory Database สำหรับ testing
- โปรแกรมตัวอย่าง: Testing service with mocked repository

---

## Mocking คืออะไร?

**Mocking** คือการสร้าง "ตัวแทน" ปลอมของ dependencies เพื่อให้ unit test ทำงานได้โดยไม่ต้องใช้ของจริง

### ทำไมต้อง Mock?

```csharp
// ❌ ปัญหา: Service นี้ต้องการ database และ email จริงๆ
public class OrderService
{
    private readonly SqlConnection _db;
    private readonly SmtpClient _smtp;
    
    public async Task<bool> PlaceOrderAsync(OrderRequest request)
    {
        // ต้องมี real database
        var cmd = new SqlCommand("INSERT INTO Orders...", _db);
        await cmd.ExecuteNonQueryAsync();
        
        // ต้องมี real email server
        await _smtp.SendMailAsync(...);
        
        return true;
    }
}

// ❌ Unit test ที่ต้องการ real dependencies
public class OrderServiceTests
{
    [Fact]
    public async Task PlaceOrder_ValidRequest_ReturnsTrue()
    {
        var db = new SqlConnection("Server=real-server..."); // ต้องมี DB จริง!
        var smtp = new SmtpClient("mail.server.com");        // ต้องมี mail server จริง!
        
        var service = new OrderService(db, smtp);
        var result = await service.PlaceOrderAsync(new OrderRequest());
        
        Assert.True(result); // Test นี้ล้มเหลวถ้าไม่มี DB/Mail
    }
}
```

### แก้ปัญหาด้วย Mocking

```csharp
// ✅ ดีกว่า: ใช้ interfaces และ mock
public interface IOrderRepository
{
    Task SaveOrderAsync(Order order);
}

public interface IEmailService
{
    Task SendOrderConfirmationAsync(string email, Order order);
}

public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailService _emailService;
    
    public OrderService(IOrderRepository repository, IEmailService emailService)
    {
        _repository = repository;
        _emailService = emailService;
    }
    
    public async Task<bool> PlaceOrderAsync(OrderRequest request)
    {
        var order = Order.Create(request);
        await _repository.SaveOrderAsync(order);
        await _emailService.SendOrderConfirmationAsync(request.Email, order);
        return true;
    }
}
```

### ประเภทของ Test Doubles

```
Test Doubles:
├── Stub    - Return ค่าที่กำหนดเอาไว้
├── Mock    - ตรวจสอบว่าถูกเรียกหรือไม่ (มี Verify)
├── Fake    - Implementation จริงแต่ simple (เช่น InMemory DB)
├── Spy     - บันทึกการเรียก method
└── Dummy   - ส่งผ่านเพื่อ satisfy parameters แต่ไม่ได้ใช้
```

---

## ติดตั้ง Moq

```bash
dotnet add package Moq
dotnet add package Moq.AutoMock  # Optional: auto-create mocks
```

### การใช้งานพื้นฐาน

```csharp
using Moq;

public class BasicMoqTests
{
    [Fact]
    public void Mock_BasicSetup()
    {
        // สร้าง Mock
        var mockRepo = new Mock<IOrderRepository>();
        
        // Setup: กำหนดว่า method จะ return อะไร
        mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                .ReturnsAsync(new Order { Id = Guid.NewGuid() });
        
        // ใช้งาน Mock (เหมือน interface จริง)
        var repo = mockRepo.Object;
        
        // Verify: ตรวจสอบว่า method ถูกเรียก
        mockRepo.Verify(r => r.GetByIdAsync(It.IsAny<Guid>()), Times.Once);
    }
}
```

---

## Setup Methods

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id);
    Task<List<Product>> GetAllAsync();
    Task<List<Product>> SearchAsync(string keyword);
    Task<bool> ExistsAsync(string sku);
    Task SaveAsync(Product product);
    Task DeleteAsync(Guid id);
}

public class SetupExamplesTests
{
    private readonly Mock<IProductRepository> _mockRepo;
    
    public SetupExamplesTests()
    {
        _mockRepo = new Mock<IProductRepository>();
    }
    
    [Fact]
    public async Task Setup_ReturnsAsync_ReturnsSpecificValue()
    {
        // กำหนดให้ return specific object
        var product = new Product { Id = Guid.NewGuid(), Name = "iPhone" };
        _mockRepo.Setup(r => r.GetByIdAsync(product.Id))
                 .ReturnsAsync(product);
        
        var result = await _mockRepo.Object.GetByIdAsync(product.Id);
        
        Assert.Equal(product.Id, result!.Id);
    }
    
    [Fact]
    public async Task Setup_ReturnsAsyncWithNull_ReturnsNull()
    {
        // กำหนดให้ return null
        _mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                 .ReturnsAsync((Product?)null);
        
        var result = await _mockRepo.Object.GetByIdAsync(Guid.NewGuid());
        
        Assert.Null(result);
    }
    
    [Fact]
    public async Task Setup_ThrowsException_ThrowsWhenCalled()
    {
        // กำหนดให้ throw exception
        _mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                 .ThrowsAsync(new DatabaseException("Connection failed"));
        
        await Assert.ThrowsAsync<DatabaseException>(
            () => _mockRepo.Object.GetByIdAsync(Guid.NewGuid()));
    }
    
    [Fact]
    public async Task Setup_WithCallback_ExecutesCallbackAndReturns()
    {
        Guid capturedId = Guid.Empty;
        
        // กำหนด callback ที่ทำงานเมื่อ method ถูกเรียก
        _mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                 .Callback<Guid>(id => capturedId = id)
                 .ReturnsAsync(new Product());
        
        var testId = Guid.NewGuid();
        await _mockRepo.Object.GetByIdAsync(testId);
        
        Assert.Equal(testId, capturedId);
    }
    
    [Fact]
    public async Task Setup_SequentialReturns_ReturnsDifferentValues()
    {
        // Return ค่าต่างกันในแต่ละครั้ง
        _mockRepo.SetupSequence(r => r.GetAllAsync())
                 .ReturnsAsync(new List<Product> { new Product() })     // ครั้งที่ 1
                 .ReturnsAsync(new List<Product>())                      // ครั้งที่ 2
                 .ThrowsAsync(new Exception("Error on 3rd call"));       // ครั้งที่ 3
        
        var first = await _mockRepo.Object.GetAllAsync();
        var second = await _mockRepo.Object.GetAllAsync();
        await Assert.ThrowsAsync<Exception>(() => _mockRepo.Object.GetAllAsync());
        
        Assert.Single(first);
        Assert.Empty(second);
    }
}
```

---

## It (Argument Matchers)

```csharp
public class ItMatcherTests
{
    private readonly Mock<IProductRepository> _mockRepo = new();
    
    [Fact]
    public void It_IsAny_MatchesAnyValue()
    {
        _mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>()))
                 .ReturnsAsync(new Product());
        
        // จะ match ไม่ว่า Guid อะไร
    }
    
    [Fact]
    public void It_Is_MatchesWithCondition()
    {
        // Match เฉพาะ Guid ที่ไม่ว่าง
        _mockRepo.Setup(r => r.GetByIdAsync(It.Is<Guid>(id => id != Guid.Empty)))
                 .ReturnsAsync(new Product());
    }
    
    [Fact]
    public async Task It_IsIn_MatchesListOfValues()
    {
        var allowedIds = new[] { Guid.Parse("11111111-0000-0000-0000-000000000000"),
                                  Guid.Parse("22222222-0000-0000-0000-000000000000") };
        
        _mockRepo.Setup(r => r.GetByIdAsync(It.IsIn(allowedIds)))
                 .ReturnsAsync(new Product { Name = "Found" });
        
        _mockRepo.Setup(r => r.GetByIdAsync(
            It.IsNotIn(allowedIds)))
                 .ReturnsAsync((Product?)null);
    }
    
    [Fact]
    public async Task It_IsRegex_MatchesPattern()
    {
        _mockRepo.Setup(r => r.SearchAsync(It.IsRegex(@"^[A-Z].*")))
                 .ReturnsAsync(new List<Product> { new() });
        
        // จะ match กับ keyword ที่ขึ้นต้นด้วยตัวพิมพ์ใหญ่
        var result = await _mockRepo.Object.SearchAsync("Apple");
        Assert.Single(result);
        
        // จะไม่ match กับ keyword อื่น
        var empty = await _mockRepo.Object.SearchAsync("apple");
        Assert.Empty(empty);
    }
}
```

---

## Verify Calls

```csharp
public class VerifyExamplesTests
{
    private readonly Mock<IOrderRepository> _mockRepo = new();
    private readonly Mock<IEmailService> _mockEmail = new();
    private readonly OrderService _service;
    
    public VerifyExamplesTests()
    {
        _service = new OrderService(_mockRepo.Object, _mockEmail.Object);
    }
    
    [Fact]
    public async Task PlaceOrder_ValidRequest_SavesAndSendsEmail()
    {
        // Arrange
        var request = new OrderRequest 
        { 
            CustomerId = "cust-001",
            Email = "customer@example.com",
            Items = new List<OrderItem> { new() { ProductId = "p1", Quantity = 1 } }
        };
        
        // Act
        await _service.PlaceOrderAsync(request);
        
        // Assert - ตรวจสอบว่า method ถูกเรียก
        _mockRepo.Verify(r => r.SaveOrderAsync(It.IsAny<Order>()), Times.Once);
        _mockEmail.Verify(
            e => e.SendOrderConfirmationAsync(
                "customer@example.com", It.IsAny<Order>()),
            Times.Once);
    }
    
    [Fact]
    public async Task PlaceOrder_RepositoryFails_DoesNotSendEmail()
    {
        // Arrange
        _mockRepo.Setup(r => r.SaveOrderAsync(It.IsAny<Order>()))
                 .ThrowsAsync(new Exception("DB Error"));
        
        // Act
        await Assert.ThrowsAsync<Exception>(
            () => _service.PlaceOrderAsync(new OrderRequest()));
        
        // Assert - email ต้องไม่ถูกส่ง
        _mockEmail.Verify(
            e => e.SendOrderConfirmationAsync(It.IsAny<string>(), It.IsAny<Order>()),
            Times.Never);
    }
    
    [Fact]
    public async Task ProcessOrders_MultipleOrders_SavesEachOne()
    {
        // Arrange
        var orders = Enumerable.Range(1, 5)
            .Select(i => new OrderRequest { CustomerId = $"cust-{i}" })
            .ToList();
        
        // Act
        foreach (var order in orders)
            await _service.PlaceOrderAsync(order);
        
        // Assert - SaveOrderAsync ถูกเรียก 5 ครั้ง
        _mockRepo.Verify(
            r => r.SaveOrderAsync(It.IsAny<Order>()),
            Times.Exactly(5));
    }
    
    [Fact]
    public void Verify_Times_Options()
    {
        // Times options:
        // Times.Once           - เรียกพอดี 1 ครั้ง
        // Times.Never          - ไม่ถูกเรียกเลย
        // Times.Exactly(n)     - เรียกพอดี n ครั้ง
        // Times.AtLeast(n)     - เรียกอย่างน้อย n ครั้ง
        // Times.AtLeastOnce()  - เรียกอย่างน้อย 1 ครั้ง
        // Times.AtMost(n)      - เรียกไม่เกิน n ครั้ง
        // Times.AtMostOnce()   - เรียกไม่เกิน 1 ครั้ง
        // Times.Between(n, m, Range.Inclusive) - เรียก n-m ครั้ง
    }
}
```

---

## Mock Interfaces และ Properties

```csharp
public interface ICurrentUser
{
    Guid Id { get; }
    string Name { get; }
    string Email { get; }
    bool IsAuthenticated { get; }
    IReadOnlyList<string> Roles { get; }
    bool IsInRole(string role);
}

public class MockPropertiesTests
{
    [Fact]
    public void Mock_Properties_CanBeSetup()
    {
        var mockUser = new Mock<ICurrentUser>();
        
        // Setup properties
        mockUser.Setup(u => u.Id).Returns(Guid.NewGuid());
        mockUser.Setup(u => u.Name).Returns("สมชาย ใจดี");
        mockUser.Setup(u => u.Email).Returns("somchai@example.com");
        mockUser.Setup(u => u.IsAuthenticated).Returns(true);
        mockUser.Setup(u => u.Roles).Returns(new List<string> { "Admin", "Manager" });
        
        // Setup method
        mockUser.Setup(u => u.IsInRole("Admin")).Returns(true);
        mockUser.Setup(u => u.IsInRole("User")).Returns(false);
        
        // Use
        var user = mockUser.Object;
        
        Assert.Equal("สมชาย ใจดี", user.Name);
        Assert.True(user.IsInRole("Admin"));
        Assert.False(user.IsInRole("User"));
        Assert.Contains("Manager", user.Roles);
    }
    
    [Fact]
    public void Mock_SetupAllProperties_AutomaticallySetupProperties()
    {
        var mockUser = new Mock<ICurrentUser>();
        mockUser.SetupAllProperties(); // Setup ทุก property อัตโนมัติ
        
        // ตั้งค่าผ่าน .Object
        mockUser.Object.Name = "Test User"; // ต้องเป็น settable property
        
        Assert.Equal("Test User", mockUser.Object.Name);
    }
}

// Mock Callbacks สำหรับ complex scenarios
public class ComplexMockTests
{
    [Fact]
    public async Task Mock_ComplexScenario_WorksCorrectly()
    {
        var mockRepo = new Mock<IProductRepository>();
        var callCount = 0;
        var savedProducts = new List<Product>();
        
        mockRepo.Setup(r => r.SaveAsync(It.IsAny<Product>()))
                .Callback<Product>(p => 
                {
                    callCount++;
                    savedProducts.Add(p);
                })
                .Returns(Task.CompletedTask);
        
        // Act
        var products = new[]
        {
            new Product { Name = "Product A" },
            new Product { Name = "Product B" }
        };
        
        foreach (var p in products)
            await mockRepo.Object.SaveAsync(p);
        
        // Assert
        Assert.Equal(2, callCount);
        Assert.Equal(2, savedProducts.Count);
        Assert.Contains(savedProducts, p => p.Name == "Product A");
        Assert.Contains(savedProducts, p => p.Name == "Product B");
    }
}
```

---

## InMemory Database สำหรับ Testing

```csharp
// ใช้ EF Core InMemory สำหรับ Integration-like Unit Tests
using Microsoft.EntityFrameworkCore;

public class InMemoryDbTests : IDisposable
{
    private readonly AppDbContext _context;
    private readonly ProductRepository _repository;
    
    public InMemoryDbTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
            
        _context = new AppDbContext(options);
        _repository = new ProductRepository(_context);
    }
    
    public void Dispose()
    {
        _context.Database.EnsureDeleted();
        _context.Dispose();
    }
    
    [Fact]
    public async Task GetById_ExistingProduct_ReturnsProduct()
    {
        // Arrange - เพิ่มข้อมูลใน InMemory DB
        var product = new Product { Id = Guid.NewGuid(), Name = "Test Product", Price = 100 };
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        
        // Act
        var result = await _repository.GetByIdAsync(product.Id);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal("Test Product", result.Name);
    }
    
    [Fact]
    public async Task GetAll_WithProducts_ReturnsAllProducts()
    {
        // Arrange
        _context.Products.AddRange(
            new Product { Id = Guid.NewGuid(), Name = "A" },
            new Product { Id = Guid.NewGuid(), Name = "B" },
            new Product { Id = Guid.NewGuid(), Name = "C" }
        );
        await _context.SaveChangesAsync();
        
        // Act
        var results = await _repository.GetAllAsync();
        
        // Assert
        Assert.Equal(3, results.Count());
    }
    
    [Fact]
    public async Task Save_NewProduct_PersistsToDatabase()
    {
        // Arrange
        var product = new Product { Id = Guid.NewGuid(), Name = "New Product", Price = 500 };
        
        // Act
        await _repository.SaveAsync(product);
        
        // Assert
        var saved = await _context.Products.FindAsync(product.Id);
        Assert.NotNull(saved);
        Assert.Equal(500, saved.Price);
    }
}
```

---

## โปรแกรมตัวอย่าง: Testing Service with Mocked Repository

```csharp
// ===== Interfaces =====
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<Product>> GetAllActiveAsync(CancellationToken ct = default);
    Task<bool> ExistsAsync(string sku, CancellationToken ct = default);
    Task SaveAsync(Product product, CancellationToken ct = default);
    Task UpdateAsync(Product product, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}

public interface INotificationService
{
    Task SendLowStockAlertAsync(Product product, CancellationToken ct = default);
    Task SendProductCreatedNotificationAsync(Product product, CancellationToken ct = default);
}

public interface ICacheService
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
    Task RemoveAsync(string key);
}

// ===== Model =====
public class Product
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

public record CreateProductRequest(string Sku, string Name, decimal Price, int InitialStock);
public record UpdateStockRequest(Guid ProductId, int Quantity, string Reason);

// ===== Service =====
public class ProductService
{
    private readonly IProductRepository _repository;
    private readonly INotificationService _notificationService;
    private readonly ICacheService _cacheService;
    private readonly ILogger<ProductService> _logger;
    private const int LowStockThreshold = 5;
    
    public ProductService(
        IProductRepository repository,
        INotificationService notificationService,
        ICacheService cacheService,
        ILogger<ProductService> logger)
    {
        _repository = repository;
        _notificationService = notificationService;
        _cacheService = cacheService;
        _logger = logger;
    }
    
    public async Task<Product> CreateProductAsync(
        CreateProductRequest request, CancellationToken ct = default)
    {
        // Validate
        if (string.IsNullOrWhiteSpace(request.Sku))
            throw new ArgumentException("SKU ต้องไม่ว่าง");
        if (request.Price <= 0)
            throw new ArgumentException("ราคาต้องมากกว่า 0");
            
        // Check duplicate SKU
        if (await _repository.ExistsAsync(request.Sku, ct))
            throw new DuplicateSkuException($"SKU {request.Sku} มีอยู่แล้ว");
            
        var product = new Product
        {
            Sku = request.Sku,
            Name = request.Name,
            Price = request.Price,
            StockQuantity = request.InitialStock
        };
        
        await _repository.SaveAsync(product, ct);
        await _notificationService.SendProductCreatedNotificationAsync(product, ct);
        
        _logger.LogInformation("สร้างสินค้า {Sku} สำเร็จ", request.Sku);
        return product;
    }
    
    public async Task<Product?> GetProductAsync(
        Guid id, CancellationToken ct = default)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cacheService.GetAsync<Product>(cacheKey);
        if (cached != null) return cached;
        
        var product = await _repository.GetByIdAsync(id, ct);
        if (product != null)
            await _cacheService.SetAsync(cacheKey, product, TimeSpan.FromMinutes(5));
            
        return product;
    }
    
    public async Task<bool> UpdateStockAsync(
        UpdateStockRequest request, CancellationToken ct = default)
    {
        var product = await _repository.GetByIdAsync(request.ProductId, ct);
        if (product is null)
        {
            _logger.LogWarning("ไม่พบสินค้า {ProductId}", request.ProductId);
            return false;
        }
        
        var newStock = product.StockQuantity + request.Quantity;
        if (newStock < 0)
            throw new InvalidOperationException(
                $"สินค้าคงคลังไม่เพียงพอ มีเพียง {product.StockQuantity}");
                
        product.StockQuantity = newStock;
        await _repository.UpdateAsync(product, ct);
        
        // ล้าง cache
        await _cacheService.RemoveAsync($"product:{product.Id}");
        
        // แจ้งเตือนถ้า stock ต่ำ
        if (product.StockQuantity <= LowStockThreshold)
            await _notificationService.SendLowStockAlertAsync(product, ct);
            
        return true;
    }
}

// ===== Tests =====
public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _mockRepo;
    private readonly Mock<INotificationService> _mockNotification;
    private readonly Mock<ICacheService> _mockCache;
    private readonly Mock<ILogger<ProductService>> _mockLogger;
    private readonly ProductService _service;
    
    public ProductServiceTests()
    {
        _mockRepo = new Mock<IProductRepository>();
        _mockNotification = new Mock<INotificationService>();
        _mockCache = new Mock<ICacheService>();
        _mockLogger = new Mock<ILogger<ProductService>>();
        
        _service = new ProductService(
            _mockRepo.Object,
            _mockNotification.Object,
            _mockCache.Object,
            _mockLogger.Object);
    }
    
    // CreateProduct Tests
    [Fact]
    public async Task CreateProduct_ValidRequest_ReturnsCreatedProduct()
    {
        // Arrange
        var request = new CreateProductRequest("SKU001", "iPhone 15", 45000m, 10);
        _mockRepo.Setup(r => r.ExistsAsync("SKU001", It.IsAny<CancellationToken>()))
                 .ReturnsAsync(false);
        _mockRepo.Setup(r => r.SaveAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()))
                 .Returns(Task.CompletedTask);
        
        // Act
        var result = await _service.CreateProductAsync(request);
        
        // Assert
        Assert.NotNull(result);
        Assert.Equal("SKU001", result.Sku);
        Assert.Equal("iPhone 15", result.Name);
        Assert.Equal(45000m, result.Price);
        Assert.Equal(10, result.StockQuantity);
        
        _mockRepo.Verify(r => r.SaveAsync(
            It.Is<Product>(p => p.Sku == "SKU001"),
            It.IsAny<CancellationToken>()), Times.Once);
        _mockNotification.Verify(
            n => n.SendProductCreatedNotificationAsync(
                It.IsAny<Product>(), It.IsAny<CancellationToken>()),
            Times.Once);
    }
    
    [Fact]
    public async Task CreateProduct_EmptySku_ThrowsArgumentException()
    {
        // Arrange
        var request = new CreateProductRequest("", "iPhone", 45000m, 10);
        
        // Act & Assert
        var ex = await Assert.ThrowsAsync<ArgumentException>(
            () => _service.CreateProductAsync(request));
        Assert.Equal("SKU ต้องไม่ว่าง", ex.Message);
        
        // Repository ต้องไม่ถูกเรียก
        _mockRepo.Verify(r => r.SaveAsync(
            It.IsAny<Product>(), It.IsAny<CancellationToken>()), Times.Never);
    }
    
    [Fact]
    public async Task CreateProduct_DuplicateSku_ThrowsDuplicateSkuException()
    {
        // Arrange
        var request = new CreateProductRequest("EXISTING-SKU", "Product", 1000m, 5);
        _mockRepo.Setup(r => r.ExistsAsync("EXISTING-SKU", It.IsAny<CancellationToken>()))
                 .ReturnsAsync(true);
        
        // Act & Assert
        await Assert.ThrowsAsync<DuplicateSkuException>(
            () => _service.CreateProductAsync(request));
        
        _mockRepo.Verify(r => r.SaveAsync(
            It.IsAny<Product>(), It.IsAny<CancellationToken>()), Times.Never);
    }
    
    // GetProduct Tests
    [Fact]
    public async Task GetProduct_CachedProduct_ReturnsFromCache()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var cachedProduct = new Product { Id = productId, Name = "Cached" };
        
        _mockCache.Setup(c => c.GetAsync<Product>($"product:{productId}"))
                  .ReturnsAsync(cachedProduct);
        
        // Act
        var result = await _service.GetProductAsync(productId);
        
        // Assert - ต้อง return จาก cache ไม่ไป hit DB
        Assert.Equal("Cached", result?.Name);
        _mockRepo.Verify(r => r.GetByIdAsync(
            It.IsAny<Guid>(), It.IsAny<CancellationToken>()), Times.Never);
    }
    
    [Fact]
    public async Task GetProduct_NotCached_FetchesFromRepositoryAndCaches()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var dbProduct = new Product { Id = productId, Name = "From DB" };
        
        _mockCache.Setup(c => c.GetAsync<Product>(It.IsAny<string>()))
                  .ReturnsAsync((Product?)null);
        _mockRepo.Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
                 .ReturnsAsync(dbProduct);
        
        // Act
        var result = await _service.GetProductAsync(productId);
        
        // Assert
        Assert.Equal("From DB", result?.Name);
        
        // ต้อง cache ผลลัพธ์
        _mockCache.Verify(c => c.SetAsync(
            $"product:{productId}",
            It.IsAny<Product>(),
            It.Is<TimeSpan?>(t => t == TimeSpan.FromMinutes(5))),
            Times.Once);
    }
    
    // UpdateStock Tests
    [Fact]
    public async Task UpdateStock_ValidDecrease_UpdatesStockAndInvalidatesCache()
    {
        // Arrange
        var product = new Product { Id = Guid.NewGuid(), StockQuantity = 10 };
        _mockRepo.Setup(r => r.GetByIdAsync(product.Id, It.IsAny<CancellationToken>()))
                 .ReturnsAsync(product);
        
        // Act
        var result = await _service.UpdateStockAsync(
            new UpdateStockRequest(product.Id, -3, "Sale"));
        
        // Assert
        Assert.True(result);
        Assert.Equal(7, product.StockQuantity);
        
        _mockRepo.Verify(r => r.UpdateAsync(product, It.IsAny<CancellationToken>()), Times.Once);
        _mockCache.Verify(c => c.RemoveAsync($"product:{product.Id}"), Times.Once);
        _mockNotification.Verify(
            n => n.SendLowStockAlertAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()),
            Times.Never); // Stock ยังไม่ต่ำ
    }
    
    [Fact]
    public async Task UpdateStock_StockDropsBelowThreshold_SendsLowStockAlert()
    {
        // Arrange
        var product = new Product { Id = Guid.NewGuid(), StockQuantity = 6 };
        _mockRepo.Setup(r => r.GetByIdAsync(product.Id, It.IsAny<CancellationToken>()))
                 .ReturnsAsync(product);
        
        // Act - ลด stock ให้เหลือ 3 (ต่ำกว่า threshold = 5)
        await _service.UpdateStockAsync(
            new UpdateStockRequest(product.Id, -3, "Sale"));
        
        // Assert - ต้องส่ง alert
        _mockNotification.Verify(
            n => n.SendLowStockAlertAsync(
                It.Is<Product>(p => p.StockQuantity == 3),
                It.IsAny<CancellationToken>()),
            Times.Once);
    }
    
    [Fact]
    public async Task UpdateStock_ExceedsCurrentStock_ThrowsInvalidOperationException()
    {
        // Arrange
        var product = new Product { Id = Guid.NewGuid(), StockQuantity = 5 };
        _mockRepo.Setup(r => r.GetByIdAsync(product.Id, It.IsAny<CancellationToken>()))
                 .ReturnsAsync(product);
        
        // Act & Assert - ลด stock มากกว่าที่มี
        await Assert.ThrowsAsync<InvalidOperationException>(
            () => _service.UpdateStockAsync(
                new UpdateStockRequest(product.Id, -10, "Sale")));
        
        _mockRepo.Verify(r => r.UpdateAsync(
            It.IsAny<Product>(), It.IsAny<CancellationToken>()), Times.Never);
    }
    
    [Fact]
    public async Task UpdateStock_ProductNotFound_ReturnsFalse()
    {
        // Arrange
        var nonExistentId = Guid.NewGuid();
        _mockRepo.Setup(r => r.GetByIdAsync(nonExistentId, It.IsAny<CancellationToken>()))
                 .ReturnsAsync((Product?)null);
        
        // Act
        var result = await _service.UpdateStockAsync(
            new UpdateStockRequest(nonExistentId, -1, "Sale"));
        
        // Assert
        Assert.False(result);
    }
}

// ===== Main Program =====
Console.WriteLine("=== Product Service Mock Testing Demo ===\n");

// รัน test แบบ manual สำหรับ demo
var tests = new ProductServiceTests();

Console.WriteLine("1. ทดสอบสร้างสินค้าสำเร็จ...");
await tests.CreateProduct_ValidRequest_ReturnsCreatedProduct();
Console.WriteLine("   ✓ ผ่าน");

Console.WriteLine("2. ทดสอบ SKU ซ้ำ...");
await tests.CreateProduct_DuplicateSku_ThrowsDuplicateSkuException();
Console.WriteLine("   ✓ ผ่าน");

Console.WriteLine("3. ทดสอบดึงจาก cache...");
await tests.GetProduct_CachedProduct_ReturnsFromCache();
Console.WriteLine("   ✓ ผ่าน");

Console.WriteLine("4. ทดสอบส่ง Low Stock Alert...");
await tests.UpdateStock_StockDropsBelowThreshold_SendsLowStockAlert();
Console.WriteLine("   ✓ ผ่าน");

Console.WriteLine("\nทุก test ผ่านแล้ว!");
```

---

## Exercises

1. **Exercise 1**: Mock `ILogger` และ Verify ว่ามีการ log เมื่อเกิด error

2. **Exercise 2**: เขียน test สำหรับ `UserAuthService` ที่ใช้ `IPasswordHasher` และ `ITokenGenerator`

3. **Exercise 3**: ทดสอบ retry logic - service ลองใหม่ 3 ครั้งเมื่อเกิด `TransientException`

4. **Exercise 4**: ใช้ `MockBehavior.Strict` เพื่อตรวจสอบว่า setup ครบทุก method ที่เรียก

5. **Exercise 5**: สร้าง Custom matcher ด้วย `It.Is<T>` สำหรับ complex object matching

---

## สรุป

Moq เป็น library ที่ช่วยให้เราสร้าง Mock objects ได้ง่าย:
- **Setup**: กำหนด behavior ของ mock (return value, exception, callback)
- **Verify**: ตรวจสอบว่า method ถูกเรียกจำนวนครั้งที่ถูกต้อง
- **It matchers**: กำหนดเงื่อนไขสำหรับ parameters
- **Callback**: ดักจับ arguments ที่ส่งมา
- **InMemory DB**: ใช้สำหรับ test ที่ต้องการ database จริงๆ

---

## Part ถัดไป

ใน Part 76 เราจะเรียนรู้เรื่อง **Integration Testing** เพื่อทดสอบระบบทั้งหมดรวมกันด้วย WebApplicationFactory

---

*Part 75/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
