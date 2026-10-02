# Part 76: Integration Testing

## เนื้อหาใน Part นี้
- Integration test vs Unit test
- WebApplicationFactory
- TestServer
- HttpClient testing
- Database testing
- โปรแกรมตัวอย่าง: API integration tests

---

## Integration Test vs Unit Test

| หัวข้อ | Unit Test | Integration Test |
|--------|-----------|------------------|
| ขอบเขต | Component เดียว | หลาย components รวมกัน |
| ความเร็ว | เร็วมาก (ms) | ช้ากว่า (s) |
| Dependencies | Mock ทั้งหมด | ใช้ของจริงบางส่วน |
| ความซับซ้อน | ต่ำ | สูงกว่า |
| จุดประสงค์ | Test logic | Test การทำงานร่วมกัน |
| เหมาะสำหรับ | Business logic | API endpoints, DB queries |

### เมื่อไหรควรใช้อะไร?

```
Unit Tests:
✅ Domain logic (calculations, validations)
✅ Service logic ที่มี complex branching
✅ Algorithm ต่างๆ

Integration Tests:
✅ API endpoints (request → response)
✅ Database queries (EF Core mappings)
✅ Authentication/Authorization flows
✅ Middleware pipeline
```

---

## WebApplicationFactory

`WebApplicationFactory<TEntryPoint>` คือ class ที่ช่วยสร้าง in-process test server สำหรับ ASP.NET Core

### ติดตั้ง

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

### การใช้งานพื้นฐาน

```csharp
// Tests/ApiTests.cs
using Microsoft.AspNetCore.Mvc.Testing;

public class ProductsApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public ProductsApiTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetProducts_ReturnsSuccessAndProducts()
    {
        // Act
        var response = await _client.GetAsync("/api/products");
        
        // Assert
        response.EnsureSuccessStatusCode();
        Assert.Equal("application/json; charset=utf-8",
            response.Content.Headers.ContentType?.ToString());
            
        var content = await response.Content.ReadAsStringAsync();
        Assert.NotEmpty(content);
    }
}
```

---

## Custom WebApplicationFactory

การ override WebApplicationFactory เพื่อ configure services สำหรับ test

```csharp
// Tests/CustomWebApplicationFactory.cs
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram>
    where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // ลบ real database service
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null)
                services.Remove(descriptor);
            
            // เพิ่ม InMemory database
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("TestDatabase"));
            
            // Replace real services ด้วย test doubles
            var emailDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(IEmailService));
            if (emailDescriptor != null)
                services.Remove(emailDescriptor);
            services.AddSingleton<IEmailService, FakeEmailService>();
        });
        
        builder.UseEnvironment("Testing");
    }
}

// Fake implementations สำหรับ test
public class FakeEmailService : IEmailService
{
    public List<EmailRecord> SentEmails { get; } = new();
    
    public Task SendAsync(string to, string subject, string body, CancellationToken ct = default)
    {
        SentEmails.Add(new EmailRecord(to, subject, body, DateTime.UtcNow));
        return Task.CompletedTask;
    }
}

public record EmailRecord(string To, string Subject, string Body, DateTime SentAt);
```

---

## TestServer

TestServer ช่วยทดสอบ HTTP requests โดยไม่ต้องใช้ network จริง

```csharp
// Tests/TestServerExample.cs
public class OrdersControllerTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    
    public OrdersControllerTests(CustomWebApplicationFactory<Program> factory)
    {
        _factory = factory;
        _client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false,  // อย่า follow redirect
            HandleCookies = true,       // จัดการ cookies
            BaseAddress = new Uri("http://localhost")
        });
    }
    
    [Fact]
    public async Task CreateOrder_ValidRequest_Returns201Created()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = "customer-001",
            Items = new List<OrderItemRequest>
            {
                new() { ProductId = "p001", Quantity = 2 }
            }
        };
        
        var content = new StringContent(
            JsonSerializer.Serialize(request),
            Encoding.UTF8,
            "application/json");
        
        // Act
        var response = await _client.PostAsync("/api/orders", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
        
        var body = await response.Content.ReadFromJsonAsync<CreateOrderResult>();
        Assert.NotNull(body);
        Assert.NotEqual(Guid.Empty, body.OrderId);
    }
    
    [Fact]
    public async Task GetOrder_NonExistentId_Returns404NotFound()
    {
        // Act
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");
        
        // Assert
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
    
    [Fact]
    public async Task CreateOrder_EmptyItems_Returns400BadRequest()
    {
        // Arrange
        var request = new { CustomerId = "cust-001", Items = new List<object>() };
        var content = JsonContent.Create(request);
        
        // Act
        var response = await _client.PostAsync("/api/orders", content);
        
        // Assert
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
        
        var errors = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
        Assert.NotNull(errors);
        Assert.Contains("Items", errors.Errors.Keys);
    }
}
```

---

## HttpClient Testing

```csharp
// Tests/HttpClientExtensions.cs
public static class HttpClientExtensions
{
    public static async Task<T?> GetFromJsonOrNullAsync<T>(
        this HttpClient client, string requestUri)
    {
        var response = await client.GetAsync(requestUri);
        if (!response.IsSuccessStatusCode) return default;
        return await response.Content.ReadFromJsonAsync<T>();
    }
    
    public static async Task<HttpResponseMessage> PostJsonAsync<T>(
        this HttpClient client, string requestUri, T data)
    {
        return await client.PostAsync(requestUri, JsonContent.Create(data));
    }
}

// Tests สำหรับ Authentication
public class AuthenticationTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public AuthenticationTests(CustomWebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task ProtectedEndpoint_WithoutToken_Returns401Unauthorized()
    {
        var response = await _client.GetAsync("/api/admin/users");
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
    
    [Fact]
    public async Task ProtectedEndpoint_WithValidToken_ReturnsSuccess()
    {
        // ได้ token ก่อน
        var loginResponse = await _client.PostJsonAsync("/api/auth/login",
            new { Email = "admin@test.com", Password = "Admin@123" });
        
        loginResponse.EnsureSuccessStatusCode();
        var tokenResult = await loginResponse.Content
            .ReadFromJsonAsync<LoginResult>();
        
        // ใช้ token
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", tokenResult!.AccessToken);
        
        var response = await _client.GetAsync("/api/admin/users");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
    
    [Fact]
    public async Task ProtectedEndpoint_WithInsufficientRole_Returns403Forbidden()
    {
        // Login ด้วย user ธรรมดา
        var loginResponse = await _client.PostJsonAsync("/api/auth/login",
            new { Email = "user@test.com", Password = "User@123" });
        
        var token = await loginResponse.Content.ReadFromJsonAsync<LoginResult>();
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token!.AccessToken);
        
        // เข้า endpoint ที่ต้องการ Admin role
        var response = await _client.GetAsync("/api/admin/users");
        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }
}
```

---

## Database Testing

```csharp
// Tests/DatabaseIntegrationTests.cs
public class ProductRepositoryIntegrationTests : IAsyncLifetime
{
    private readonly CustomWebApplicationFactory<Program> _factory;
    private AppDbContext _context = null!;
    
    public ProductRepositoryIntegrationTests()
    {
        _factory = new CustomWebApplicationFactory<Program>();
    }
    
    public async Task InitializeAsync()
    {
        // Setup: สร้าง context และ seed data
        var scope = _factory.Services.CreateScope();
        _context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await _context.Database.EnsureCreatedAsync();
        await SeedTestDataAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _context.Database.EnsureDeletedAsync();
        _context.Dispose();
        _factory.Dispose();
    }
    
    private async Task SeedTestDataAsync()
    {
        _context.Products.AddRange(
            new Product { Id = Guid.Parse("11111111-0000-0000-0000-000000000000"),
                         Name = "Product A", Price = 100, StockQuantity = 10, IsActive = true },
            new Product { Id = Guid.Parse("22222222-0000-0000-0000-000000000000"),
                         Name = "Product B", Price = 200, StockQuantity = 5, IsActive = true },
            new Product { Id = Guid.Parse("33333333-0000-0000-0000-000000000000"),
                         Name = "Inactive Product", Price = 50, StockQuantity = 0, IsActive = false }
        );
        await _context.SaveChangesAsync();
    }
    
    [Fact]
    public async Task GetAllActive_ReturnsOnlyActiveProducts()
    {
        // Arrange
        var client = _factory.CreateClient();
        
        // Act
        var response = await client.GetAsync("/api/products?activeOnly=true");
        response.EnsureSuccessStatusCode();
        
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        
        // Assert - ต้องได้เฉพาะ active products
        Assert.NotNull(products);
        Assert.Equal(2, products.Count);
        Assert.All(products, p => Assert.True(p.IsActive));
    }
    
    [Fact]
    public async Task CreateAndRetrieveProduct_RoundTrip_DataPersisted()
    {
        // Arrange
        var client = _factory.CreateClient();
        var newProduct = new { Name = "Test Product", Price = 999.99m, Stock = 100 };
        
        // Act - Create
        var createResponse = await client.PostAsync("/api/products",
            JsonContent.Create(newProduct));
        createResponse.EnsureSuccessStatusCode();
        
        var created = await createResponse.Content.ReadFromJsonAsync<ProductDto>();
        Assert.NotNull(created);
        
        // Act - Retrieve
        var getResponse = await client.GetAsync($"/api/products/{created.Id}");
        getResponse.EnsureSuccessStatusCode();
        
        var retrieved = await getResponse.Content.ReadFromJsonAsync<ProductDto>();
        
        // Assert
        Assert.NotNull(retrieved);
        Assert.Equal("Test Product", retrieved.Name);
        Assert.Equal(999.99m, retrieved.Price);
    }
}
```

---

## โปรแกรมตัวอย่าง: API Integration Tests

```csharp
// ===== API (ตัวอย่าง) =====
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddDbContext<TestAppDbContext>(opt =>
    opt.UseInMemoryDatabase("IntegrationTestDb"));
builder.Services.AddScoped<IProductRepository, EfProductRepository>();

var app = builder.Build();
app.MapControllers();
app.Run();

public partial class Program { } // ต้องมีสำหรับ WebApplicationFactory

// Models
public class Product
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public bool IsActive { get; set; } = true;
}

public record CreateProductDto(string Name, decimal Price, int Stock);
public record ProductResponseDto(Guid Id, string Name, decimal Price, int Stock, bool IsActive);

// Controller
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly TestAppDbContext _context;
    
    public ProductsController(TestAppDbContext context)
    {
        _context = context;
    }
    
    [HttpGet]
    public async Task<ActionResult<List<ProductResponseDto>>> GetAll([FromQuery] bool activeOnly = false)
    {
        var query = _context.Products.AsQueryable();
        if (activeOnly) query = query.Where(p => p.IsActive);
        
        var products = await query
            .Select(p => new ProductResponseDto(p.Id, p.Name, p.Price, p.Stock, p.IsActive))
            .ToListAsync();
            
        return Ok(products);
    }
    
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<ProductResponseDto>> GetById(Guid id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product is null) return NotFound(new { Message = $"ไม่พบสินค้า ID: {id}" });
        
        return Ok(new ProductResponseDto(
            product.Id, product.Name, product.Price, product.Stock, product.IsActive));
    }
    
    [HttpPost]
    public async Task<ActionResult<ProductResponseDto>> Create([FromBody] CreateProductDto dto)
    {
        if (string.IsNullOrWhiteSpace(dto.Name))
            return BadRequest(new { Message = "ชื่อสินค้าต้องไม่ว่าง" });
        if (dto.Price <= 0)
            return BadRequest(new { Message = "ราคาต้องมากกว่า 0" });
        if (dto.Stock < 0)
            return BadRequest(new { Message = "จำนวนสินค้าต้องไม่ติดลบ" });
        
        var product = new Product
        {
            Name = dto.Name,
            Price = dto.Price,
            Stock = dto.Stock
        };
        
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        
        var response = new ProductResponseDto(
            product.Id, product.Name, product.Price, product.Stock, product.IsActive);
            
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, response);
    }
    
    [HttpPut("{id:guid}")]
    public async Task<IActionResult> Update(Guid id, [FromBody] CreateProductDto dto)
    {
        var product = await _context.Products.FindAsync(id);
        if (product is null) return NotFound();
        
        product.Name = dto.Name;
        product.Price = dto.Price;
        product.Stock = dto.Stock;
        
        await _context.SaveChangesAsync();
        return NoContent();
    }
    
    [HttpDelete("{id:guid}")]
    public async Task<IActionResult> Delete(Guid id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product is null) return NotFound();
        
        product.IsActive = false; // Soft delete
        await _context.SaveChangesAsync();
        return NoContent();
    }
}

// DbContext
public class TestAppDbContext : DbContext
{
    public TestAppDbContext(DbContextOptions<TestAppDbContext> options) : base(options) { }
    public DbSet<Product> Products => Set<Product>();
}

// ===== Integration Tests =====
public class ProductsApiIntegrationTests : IClassFixture<WebApplicationFactory<Program>>, IDisposable
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly HttpClient _client;
    private static readonly Guid ExistingProductId = Guid.NewGuid();
    private static bool _seeded = false;
    
    public ProductsApiIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // ใช้ InMemory database แยกสำหรับ test แต่ละ class
                var descriptor = services.SingleOrDefault(
                    d => d.ServiceType == typeof(DbContextOptions<TestAppDbContext>));
                if (descriptor != null) services.Remove(descriptor);
                
                services.AddDbContext<TestAppDbContext>(opt =>
                    opt.UseInMemoryDatabase($"TestDb-{Guid.NewGuid()}"));
            });
        });
        
        _client = _factory.CreateClient();
        SeedDataAsync().GetAwaiter().GetResult();
    }
    
    private async Task SeedDataAsync()
    {
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<TestAppDbContext>();
        
        db.Products.Add(new Product
        {
            Id = ExistingProductId,
            Name = "Seeded Product",
            Price = 100m,
            Stock = 20,
            IsActive = true
        });
        db.Products.Add(new Product
        {
            Name = "Another Product",
            Price = 200m,
            Stock = 5,
            IsActive = true
        });
        db.Products.Add(new Product
        {
            Name = "Inactive Product",
            Price = 50m,
            Stock = 0,
            IsActive = false
        });
        
        await db.SaveChangesAsync();
    }
    
    // GET /api/products Tests
    [Fact]
    public async Task GetAll_NoFilter_ReturnsAllProducts()
    {
        var response = await _client.GetAsync("/api/products");
        response.EnsureSuccessStatusCode();
        
        var products = await response.Content
            .ReadFromJsonAsync<List<ProductResponseDto>>();
            
        Assert.NotNull(products);
        Assert.Equal(3, products.Count);
    }
    
    [Fact]
    public async Task GetAll_ActiveOnly_ReturnsOnlyActiveProducts()
    {
        var response = await _client.GetAsync("/api/products?activeOnly=true");
        response.EnsureSuccessStatusCode();
        
        var products = await response.Content
            .ReadFromJsonAsync<List<ProductResponseDto>>();
            
        Assert.NotNull(products);
        Assert.Equal(2, products.Count);
        Assert.All(products, p => Assert.True(p.IsActive));
    }
    
    // GET /api/products/{id} Tests
    [Fact]
    public async Task GetById_ExistingProduct_ReturnsProduct()
    {
        var response = await _client.GetAsync($"/api/products/{ExistingProductId}");
        
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
        
        var product = await response.Content.ReadFromJsonAsync<ProductResponseDto>();
        Assert.NotNull(product);
        Assert.Equal(ExistingProductId, product.Id);
        Assert.Equal("Seeded Product", product.Name);
    }
    
    [Fact]
    public async Task GetById_NonExistentProduct_Returns404()
    {
        var response = await _client.GetAsync($"/api/products/{Guid.NewGuid()}");
        
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
    
    // POST /api/products Tests
    [Fact]
    public async Task Create_ValidProduct_Returns201WithLocation()
    {
        var newProduct = new CreateProductDto("New iPhone", 45000m, 50);
        
        var response = await _client.PostAsync("/api/products", JsonContent.Create(newProduct));
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
        
        var created = await response.Content.ReadFromJsonAsync<ProductResponseDto>();
        Assert.NotNull(created);
        Assert.NotEqual(Guid.Empty, created.Id);
        Assert.Equal("New iPhone", created.Name);
        Assert.Equal(45000m, created.Price);
    }
    
    [Theory]
    [InlineData("", 100, 10, "ชื่อสินค้า")]
    [InlineData("Valid Name", -1, 10, "ราคา")]
    [InlineData("Valid Name", 100, -1, "จำนวน")]
    public async Task Create_InvalidData_Returns400BadRequest(
        string name, decimal price, int stock, string errorField)
    {
        var newProduct = new CreateProductDto(name, price, stock);
        var response = await _client.PostAsync("/api/products", JsonContent.Create(newProduct));
        
        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
    
    [Fact]
    public async Task Create_ThenGet_ProductPersisted()
    {
        // Create
        var createDto = new CreateProductDto("MacBook Pro", 89000m, 5);
        var createResponse = await _client.PostAsync(
            "/api/products", JsonContent.Create(createDto));
        createResponse.EnsureSuccessStatusCode();
        
        var created = await createResponse.Content.ReadFromJsonAsync<ProductResponseDto>();
        
        // Get
        var getResponse = await _client.GetAsync($"/api/products/{created!.Id}");
        getResponse.EnsureSuccessStatusCode();
        
        var retrieved = await getResponse.Content.ReadFromJsonAsync<ProductResponseDto>();
        
        Assert.Equal(created.Name, retrieved!.Name);
        Assert.Equal(created.Price, retrieved.Price);
    }
    
    // DELETE /api/products/{id} Tests
    [Fact]
    public async Task Delete_ExistingProduct_Returns204()
    {
        // สร้างสินค้าก่อนลบ
        var createDto = new CreateProductDto("To Be Deleted", 100m, 1);
        var createResponse = await _client.PostAsync(
            "/api/products", JsonContent.Create(createDto));
        var product = await createResponse.Content.ReadFromJsonAsync<ProductResponseDto>();
        
        // Delete
        var deleteResponse = await _client.DeleteAsync($"/api/products/{product!.Id}");
        Assert.Equal(HttpStatusCode.NoContent, deleteResponse.StatusCode);
        
        // ยืนยันว่า inactive แล้ว (Soft delete)
        var getResponse = await _client.GetAsync($"/api/products/{product.Id}");
        var retrieved = await getResponse.Content.ReadFromJsonAsync<ProductResponseDto>();
        Assert.False(retrieved!.IsActive);
    }
    
    [Fact]
    public async Task Delete_NonExistentProduct_Returns404()
    {
        var response = await _client.DeleteAsync($"/api/products/{Guid.NewGuid()}");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
    
    public void Dispose()
    {
        _client.Dispose();
        _factory.Dispose();
    }
}

// ===== Response Types =====
public record ProductResponseDto(
    Guid Id, string Name, decimal Price, int Stock, bool IsActive);
    
public record CreateProductDto(string Name, decimal Price, int Stock);

// ===== Main Program สำหรับรัน Demo =====
Console.WriteLine("=== Integration Testing Demo ===");
Console.WriteLine("รัน tests ด้วยคำสั่ง: dotnet test");
Console.WriteLine("\nTests ที่มี:");
Console.WriteLine("  ✓ GetAll_NoFilter_ReturnsAllProducts");
Console.WriteLine("  ✓ GetAll_ActiveOnly_ReturnsOnlyActiveProducts");
Console.WriteLine("  ✓ GetById_ExistingProduct_ReturnsProduct");
Console.WriteLine("  ✓ GetById_NonExistentProduct_Returns404");
Console.WriteLine("  ✓ Create_ValidProduct_Returns201WithLocation");
Console.WriteLine("  ✓ Create_InvalidData_Returns400BadRequest");
Console.WriteLine("  ✓ Create_ThenGet_ProductPersisted");
Console.WriteLine("  ✓ Delete_ExistingProduct_Returns204");
Console.WriteLine("  ✓ Delete_NonExistentProduct_Returns404");
```

---

## Exercises

1. **Exercise 1**: เพิ่ม test สำหรับ Pagination:
   - GET /api/products?page=1&pageSize=5 ต้องคืน 5 สินค้า
   - GET /api/products?page=2&pageSize=5 ต้องคืนสินค้าจาก index 5

2. **Exercise 2**: เขียน Integration Test สำหรับ Authentication flow:
   - Register → Login → Access protected resource

3. **Exercise 3**: ทดสอบ Error handling:
   - Unhandled exceptions ต้อง return 500 พร้อม ProblemDetails
   - Validation errors ต้อง return 400 พร้อม error messages

4. **Exercise 4**: ทดสอบ Rate Limiting:
   - เรียก API เกิน limit ต้อง return 429 Too Many Requests

5. **Exercise 5**: ใช้ `TestContainers` สำหรับรัน real PostgreSQL ใน Docker ระหว่าง test

---

## Advanced Integration Testing Techniques

### Test Ordering

บางครั้ง tests ต้องทำงานตามลำดับ xUnit รองรับด้วย `TestCaseOrderer`

```csharp
// กำหนดลำดับ test ด้วย attribute
[Collection("OrderedTests")]
public class OrderedIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    private static Guid _createdProductId;
    
    public OrderedIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact, TestPriority(1)]
    public async Task Step1_CreateProduct_Returns201()
    {
        var response = await _client.PostAsync("/api/products",
            JsonContent.Create(new { Name = "Test Product", Price = 100m, Stock = 10 }));
        response.EnsureSuccessStatusCode();
        
        var product = await response.Content.ReadFromJsonAsync<ProductResponseDto>();
        _createdProductId = product!.Id;
        
        Assert.NotEqual(Guid.Empty, _createdProductId);
    }
    
    [Fact, TestPriority(2)]
    public async Task Step2_GetCreatedProduct_ReturnsProduct()
    {
        Assert.NotEqual(Guid.Empty, _createdProductId);
        
        var response = await _client.GetAsync($"/api/products/{_createdProductId}");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
    
    [Fact, TestPriority(3)]
    public async Task Step3_DeleteProduct_Returns204()
    {
        var response = await _client.DeleteAsync($"/api/products/{_createdProductId}");
        Assert.Equal(HttpStatusCode.NoContent, response.StatusCode);
    }
}

[AttributeUsage(AttributeTargets.Method)]
public class TestPriorityAttribute : Attribute
{
    public TestPriorityAttribute(int priority) => Priority = priority;
    public int Priority { get; }
}
```

### Testing Middleware

```csharp
public class MiddlewareIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public MiddlewareIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task RequestLoggingMiddleware_AddsCorrelationIdHeader()
    {
        var response = await _client.GetAsync("/api/products");
        
        // ตรวจสอบว่า middleware เพิ่ม header
        Assert.True(response.Headers.Contains("X-Correlation-Id"));
    }
    
    [Fact]
    public async Task ExceptionHandlingMiddleware_UnhandledException_ReturnsProblemDetails()
    {
        // เรียก endpoint ที่จะ throw exception
        var response = await _client.GetAsync("/api/test/error");
        
        Assert.Equal(HttpStatusCode.InternalServerError, response.StatusCode);
        
        var problem = await response.Content.ReadFromJsonAsync<ProblemDetails>();
        Assert.NotNull(problem);
        Assert.Equal(500, problem.Status);
    }
    
    [Fact]
    public async Task CorsMiddleware_WithAllowedOrigin_AddsHeaders()
    {
        var request = new HttpRequestMessage(HttpMethod.Get, "/api/products");
        request.Headers.Add("Origin", "https://allowed-origin.com");
        
        var response = await _client.SendAsync(request);
        
        Assert.True(response.Headers.Contains("Access-Control-Allow-Origin"));
    }
}
```

### Testing with Real Database (TestContainers)

```csharp
// ติดตั้ง: dotnet add package Testcontainers.PostgreSql
// ให้ใช้ real PostgreSQL ใน Docker สำหรับ Integration Tests

public class PostgreSqlIntegrationTests : IAsyncLifetime
{
    private PostgreSqlContainer _postgres = null!;
    private WebApplicationFactory<Program> _factory = null!;
    private HttpClient _client = null!;
    
    public async Task InitializeAsync()
    {
        // สร้าง PostgreSQL container
        _postgres = new PostgreSqlBuilder()
            .WithDatabase("testdb")
            .WithUsername("testuser")
            .WithPassword("testpass")
            .Build();
        
        await _postgres.StartAsync();
        
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // ใช้ connection string จาก container
                    var descriptor = services.SingleOrDefault(
                        d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
                    if (descriptor != null) services.Remove(descriptor);
                    
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseNpgsql(_postgres.GetConnectionString()));
                });
            });
        
        _client = _factory.CreateClient();
        
        // Run migrations
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }
    
    public async Task DisposeAsync()
    {
        _client?.Dispose();
        _factory?.Dispose();
        await _postgres.StopAsync();
        await _postgres.DisposeAsync();
    }
    
    [Fact]
    public async Task Create_WithRealPostgres_PersistsData()
    {
        var createDto = new { Name = "Real DB Product", Price = 999m, Stock = 5 };
        
        var createResponse = await _client.PostAsync(
            "/api/products", JsonContent.Create(createDto));
        createResponse.EnsureSuccessStatusCode();
        
        var created = await createResponse.Content.ReadFromJsonAsync<ProductResponseDto>();
        
        // Query directly from database
        using var scope = _factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var product = await db.Products.FindAsync(created!.Id);
        
        Assert.NotNull(product);
        Assert.Equal("Real DB Product", product.Name);
    }
}
```

### Snapshot Testing

```csharp
// การทดสอบที่ compare response กับ "snapshot" ที่บันทึกไว้
public class SnapshotTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public SnapshotTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetProducts_ResponseMatchesSnapshot()
    {
        var response = await _client.GetAsync("/api/products");
        var content = await response.Content.ReadAsStringAsync();
        
        // Save snapshot ครั้งแรก หรือ compare กับที่มีอยู่
        var snapshotPath = "Snapshots/get_products.json";
        
        if (!File.Exists(snapshotPath))
        {
            Directory.CreateDirectory("Snapshots");
            await File.WriteAllTextAsync(snapshotPath, content);
            return; // ครั้งแรก save snapshot
        }
        
        var expected = await File.ReadAllTextAsync(snapshotPath);
        
        // Normalize JSON สำหรับ comparison
        var expectedJson = JsonDocument.Parse(expected);
        var actualJson = JsonDocument.Parse(content);
        
        Assert.Equal(
            JsonSerializer.Serialize(expectedJson, new JsonSerializerOptions { WriteIndented = true }),
            JsonSerializer.Serialize(actualJson, new JsonSerializerOptions { WriteIndented = true }));
    }
}
```

---

## สรุป

Integration Testing ด้วย WebApplicationFactory ช่วยให้เรา:
- **ทดสอบ HTTP endpoints** แบบ end-to-end
- **ใช้ InMemory Database** แทน production database
- **Test authentication/authorization** flows
- **ยืนยัน response format** และ status codes
- **IAsyncLifetime** สำหรับ async setup/teardown
- **TestContainers** สำหรับ test กับ database จริงใน Docker
- **Middleware testing** ตรวจสอบ headers และ error handling

---

## Part ถัดไป

ใน Part 77 เราจะเรียนรู้เรื่อง **Design Patterns: Creational** เช่น Singleton, Factory Method, Builder

---

*Part 76/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
