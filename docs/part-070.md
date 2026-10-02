# Part 070: Minimal API ขั้นสูง (ASP.NET Core 7+)

## เนื้อหาใน Part นี้
- Route Groups - จัดกลุ่ม endpoints
- Endpoint Filters
- TypedResults สำหรับ type-safe responses
- OpenAPI Integration ใน Minimal API
- Validation ใน Minimal API
- โปรแกรมตัวอย่าง: Complete Minimal API

---

## 1. Route Groups

**Route Groups** ช่วยจัดระเบียบ endpoints ที่มี prefix และ middleware เหมือนกัน

### MapGroup พื้นฐาน

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// สร้าง group ที่มี common prefix
var apiGroup = app.MapGroup("/api/v1");

var productsGroup = apiGroup.MapGroup("/products")
    .WithTags("Products")           // OpenAPI tags
    .RequireAuthorization()         // ต้อง authenticate
    .WithOpenApi();

var categoriesGroup = apiGroup.MapGroup("/categories")
    .WithTags("Categories")
    .WithOpenApi();

// เพิ่ม endpoints ใน group
productsGroup.MapGet("/", GetAllProducts);
productsGroup.MapGet("/{id:int}", GetProductById);
productsGroup.MapPost("/", CreateProduct);
productsGroup.MapPut("/{id:int}", UpdateProduct);
productsGroup.MapDelete("/{id:int}", DeleteProduct);

categoriesGroup.MapGet("/", GetAllCategories);
categoriesGroup.MapGet("/{id:int}", GetCategoryById);

app.Run();
```

### Route Groups แบบ Organized

```csharp
// Endpoints/ProductEndpoints.cs
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;

namespace MinimalApiDemo.Endpoints;

public static class ProductEndpoints
{
    public static RouteGroupBuilder MapProductEndpoints(this RouteGroupBuilder group)
    {
        group.MapGet("/", GetAll)
             .WithName("GetAllProducts")
             .WithSummary("ดึงสินค้าทั้งหมด")
             .WithDescription("ดึงรายการสินค้าทั้งหมดพร้อม pagination");

        group.MapGet("/{id:int}", GetById)
             .WithName("GetProductById")
             .WithSummary("ดึงสินค้าตาม ID");

        group.MapPost("/", Create)
             .WithName("CreateProduct")
             .WithSummary("สร้างสินค้าใหม่")
             .RequireAuthorization();

        group.MapPut("/{id:int}", Update)
             .WithName("UpdateProduct")
             .WithSummary("อัปเดทสินค้า")
             .RequireAuthorization();

        group.MapDelete("/{id:int}", Delete)
             .WithName("DeleteProduct")
             .WithSummary("ลบสินค้า")
             .RequireAuthorization("AdminPolicy");

        return group;
    }

    private static async Task<Results<Ok<PagedResult<ProductDto>>, BadRequest<string>>> GetAll(
        [FromServices] IProductService service,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        [FromQuery] string? search = null)
    {
        if (page < 1 || pageSize < 1)
            return TypedResults.BadRequest("page และ pageSize ต้องมากกว่า 0");

        var result = await service.GetAllAsync(page, pageSize, search);
        return TypedResults.Ok(result);
    }

    private static async Task<Results<Ok<ProductDto>, NotFound>> GetById(
        int id,
        [FromServices] IProductService service)
    {
        var product = await service.GetByIdAsync(id);

        return product is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(product);
    }

    private static async Task<Results<Created<ProductDto>, ValidationProblem>> Create(
        [FromBody] CreateProductRequest request,
        [FromServices] IProductService service,
        [FromServices] IValidator<CreateProductRequest> validator,
        HttpContext httpContext)
    {
        var validationResult = await validator.ValidateAsync(request);

        if (!validationResult.IsValid)
        {
            return TypedResults.ValidationProblem(
                validationResult.ToDictionary());
        }

        var product = await service.CreateAsync(request);

        return TypedResults.Created(
            $"/api/v1/products/{product.Id}",
            product);
    }

    private static async Task<Results<Ok<ProductDto>, NotFound, ValidationProblem>> Update(
        int id,
        [FromBody] UpdateProductRequest request,
        [FromServices] IProductService service,
        [FromServices] IValidator<UpdateProductRequest> validator)
    {
        var validationResult = await validator.ValidateAsync(request);

        if (!validationResult.IsValid)
            return TypedResults.ValidationProblem(validationResult.ToDictionary());

        var product = await service.UpdateAsync(id, request);

        return product is null
            ? TypedResults.NotFound()
            : TypedResults.Ok(product);
    }

    private static async Task<Results<NoContent, NotFound>> Delete(
        int id,
        [FromServices] IProductService service)
    {
        var deleted = await service.DeleteAsync(id);

        return deleted
            ? TypedResults.NoContent()
            : TypedResults.NotFound();
    }
}

// ใช้ใน Program.cs
app.MapGroup("/api/v1/products")
   .WithTags("Products")
   .MapProductEndpoints();
```

---

## 2. Endpoint Filters

**Endpoint Filters** ใช้เพิ่ม behavior ก่อน/หลัง endpoint handler (คล้าย Action Filters ใน MVC)

### IEndpointFilter

```csharp
// Filters/LoggingFilter.cs
namespace MinimalApiDemo.Filters;

public class LoggingFilter : IEndpointFilter
{
    private readonly ILogger<LoggingFilter> _logger;

    public LoggingFilter(ILogger<LoggingFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var request = context.HttpContext.Request;
        _logger.LogInformation(
            "Request: {Method} {Path}",
            request.Method,
            request.Path);

        var result = await next(context);

        _logger.LogInformation(
            "Response: {Method} {Path} completed",
            request.Method,
            request.Path);

        return result;
    }
}

// Validation Filter
public class ValidationFilter<T> : IEndpointFilter
{
    private readonly IValidator<T> _validator;

    public ValidationFilter(IValidator<T> validator)
    {
        _validator = validator;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        // หา argument ที่เป็น type T
        var argument = context.Arguments.OfType<T>().FirstOrDefault();

        if (argument is null)
            return await next(context);

        var validationResult = await _validator.ValidateAsync(argument);

        if (!validationResult.IsValid)
        {
            return TypedResults.ValidationProblem(validationResult.ToDictionary());
        }

        return await next(context);
    }
}

// Short-circuit filter
public class ApiKeyFilter : IEndpointFilter
{
    private const string ApiKeyHeader = "X-Api-Key";
    private readonly string _validApiKey;

    public ApiKeyFilter(IConfiguration configuration)
    {
        _validApiKey = configuration["ApiKey"]
            ?? throw new InvalidOperationException("ApiKey not configured");
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        if (!context.HttpContext.Request.Headers.TryGetValue(ApiKeyHeader, out var apiKey)
            || apiKey != _validApiKey)
        {
            return TypedResults.Unauthorized();
        }

        return await next(context);
    }
}
```

### ใช้ Filters

```csharp
// เพิ่ม filter ใน endpoint
app.MapGet("/api/products", GetProducts)
   .AddEndpointFilter<LoggingFilter>()
   .AddEndpointFilter<ApiKeyFilter>();

// เพิ่ม filter ใน group
var group = app.MapGroup("/api/products")
    .AddEndpointFilter<LoggingFilter>()
    .AddEndpointFilter(async (context, next) =>
    {
        // Inline filter
        Console.WriteLine("Before request");
        var result = await next(context);
        Console.WriteLine("After request");
        return result;
    });

// เพิ่ม validation filter
app.MapPost("/api/products", CreateProduct)
   .AddEndpointFilter<ValidationFilter<CreateProductRequest>>();
```

### Endpoint Filter Factory

```csharp
// ใช้ factory pattern สำหรับ compile-time optimization
app.MapPost("/api/products", CreateProduct)
   .AddEndpointFilterFactory((filterFactoryContext, next) =>
   {
       var parameterType = filterFactoryContext.MethodInfo
           .GetParameters()
           .FirstOrDefault(p => p.GetCustomAttributes<FromBodyAttribute>().Any())
           ?.ParameterType;

       if (parameterType is null) return next;

       return async (invocationContext) =>
       {
           // validate logic
           return await next(invocationContext);
       };
   });
```

---

## 3. TypedResults

**TypedResults** ใน .NET 7+ ช่วยให้ return types เป็น strongly typed

### TypedResults ต่างๆ

```csharp
// ตัวอย่าง TypedResults ที่ใช้บ่อย
TypedResults.Ok(data)                       // HTTP 200 with body
TypedResults.Created(url, data)             // HTTP 201
TypedResults.CreatedAtRoute(routeName, routeValues, data)  // HTTP 201
TypedResults.NoContent()                    // HTTP 204
TypedResults.NotFound()                     // HTTP 404
TypedResults.NotFound(detail)              // HTTP 404 with detail
TypedResults.BadRequest(message)            // HTTP 400
TypedResults.ValidationProblem(errors)     // HTTP 400 with validation errors
TypedResults.Unauthorized()                 // HTTP 401
TypedResults.Forbid()                       // HTTP 403
TypedResults.Conflict()                     // HTTP 409
TypedResults.UnprocessableEntity()          // HTTP 422
TypedResults.Problem(detail, title, statusCode) // Problem Details
TypedResults.Json(data, options)            // Custom JSON
TypedResults.Text(text, contentType)        // Text response
TypedResults.File(bytes, contentType)       // File response
TypedResults.Redirect(url)                  // HTTP 302
TypedResults.RedirectToRoute(routeName)     // Redirect to named route
TypedResults.Stream(stream, contentType)    // Stream response
TypedResults.Bytes(bytes, contentType)      // Raw bytes
```

### Union Return Types

```csharp
// ระบุ return types ที่เป็นไปได้ทั้งหมด
app.MapGet("/products/{id}", async Task<Results<
    Ok<ProductDto>,
    NotFound,
    BadRequest<string>>> (
    int id,
    IProductService service) =>
{
    if (id <= 0)
        return TypedResults.BadRequest("ID ต้องมากกว่า 0");

    var product = await service.GetByIdAsync(id);

    if (product is null)
        return TypedResults.NotFound();

    return TypedResults.Ok(product);
});

// ซับซ้อนกว่า - หลาย outcomes
static async Task<Results<
    Created<OrderDto>,
    NotFound,
    Conflict<string>,
    ValidationProblem>> CreateOrder(
    CreateOrderRequest request,
    IOrderService orderService,
    IProductService productService)
{
    // ตรวจสอบ product มีอยู่
    var product = await productService.GetByIdAsync(request.ProductId);
    if (product is null)
        return TypedResults.NotFound();

    // ตรวจสอบ duplicate
    var existing = await orderService.GetExistingOrderAsync(request.CustomerId, request.ProductId);
    if (existing is not null)
        return TypedResults.Conflict("มีคำสั่งซื้อสินค้านี้แล้ว");

    // Validate
    if (request.Quantity <= 0)
    {
        return TypedResults.ValidationProblem(new Dictionary<string, string[]>
        {
            ["quantity"] = new[] { "จำนวนต้องมากกว่า 0" }
        });
    }

    var order = await orderService.CreateAsync(request);
    return TypedResults.Created($"/api/orders/{order.Id}", order);
}
```

---

## 4. OpenAPI Integration ใน Minimal API

### การตั้งค่า OpenAPI

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Minimal API",
        Version = "v1"
    });
});

// หรือใช้ Microsoft.AspNetCore.OpenApi (.NET 9)
builder.Services.AddOpenApi();
```

### Endpoint Metadata

```csharp
app.MapGet("/api/products", GetProducts)
   .WithName("GetProducts")           // Operation ID
   .WithSummary("ดึงสินค้าทั้งหมด")
   .WithDescription("ดึงรายการสินค้าพร้อม pagination, search, และ filter")
   .WithTags("Products")
   .WithOpenApi(operation =>
   {
       operation.Parameters[0].Description = "หน้าที่ต้องการ (เริ่มจาก 1)";
       operation.Parameters[1].Description = "จำนวนรายการต่อหน้า (สูงสุด 100)";
       return operation;
   })
   .Produces<PagedResult<ProductDto>>(200)
   .Produces<ProblemDetails>(400)
   .RequireAuthorization()
   .AllowAnonymous()  // ยกเลิก RequireAuthorization
   .CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(5)));
```

### Endpoint Metadata Attributes

```csharp
// Endpoint Filters พร้อม OpenAPI description
app.MapPost("/api/products", CreateProduct)
   .WithName("CreateProduct")
   .WithOpenApi(operation =>
   {
       operation.RequestBody!.Description = "ข้อมูลสินค้าที่ต้องการสร้าง";
       operation.RequestBody.Required = true;
       return operation;
   })
   .Accepts<CreateProductRequest>("application/json")
   .Produces<ProductDto>(201)
   .Produces<ValidationProblemDetails>(400)
   .Produces(401)
   .WithTags("Products");
```

---

## 5. Validation ใน Minimal API

### FluentValidation

```bash
dotnet add package FluentValidation
dotnet add package FluentValidation.DependencyInjectionExtensions
```

```csharp
// Validators/ProductValidators.cs
using FluentValidation;

namespace MinimalApiDemo.Validators;

public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("ชื่อสินค้าจำเป็น")
            .Length(3, 200).WithMessage("ชื่อต้องมี 3-200 ตัวอักษร");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("ราคาต้องมากกว่า 0")
            .LessThanOrEqualTo(9_999_999).WithMessage("ราคาสูงเกินไป");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("สต็อกต้องไม่ติดลบ");

        RuleFor(x => x.CategoryId)
            .GreaterThan(0).WithMessage("ต้องระบุหมวดหมู่");
    }
}

// ลงทะเบียน validators
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Custom validation extension
public static class ValidationExtensions
{
    public static RouteHandlerBuilder WithValidation<T>(
        this RouteHandlerBuilder builder) where T : class
    {
        return builder.AddEndpointFilter<ValidationFilter<T>>();
    }
}
```

### Minimal Validation ใน Handler

```csharp
// ใช้ Data Annotations
app.MapPost("/api/products", async (
    [AsParameters] CreateProductRequest request,
    IProductService service,
    HttpContext context) =>
{
    // Validate ด้วย .NET built-in
    var validationContext = new ValidationContext(request);
    var validationResults = new List<ValidationResult>();

    if (!Validator.TryValidateObject(request, validationContext, validationResults, true))
    {
        var errors = validationResults
            .GroupBy(r => r.MemberNames.FirstOrDefault() ?? "")
            .ToDictionary(
                g => g.Key,
                g => g.Select(r => r.ErrorMessage ?? "").ToArray());

        return TypedResults.ValidationProblem(errors);
    }

    var product = await service.CreateAsync(request);
    return TypedResults.Created($"/api/products/{product.Id}", product);
});
```

---

## โปรแกรมตัวอย่าง: Complete Minimal API

ระบบ e-commerce API แบบ Minimal API ที่สมบูรณ์

### โครงสร้าง

```
MinimalApiEcommerce/
├── Endpoints/
│   ├── ProductEndpoints.cs
│   ├── CategoryEndpoints.cs
│   ├── OrderEndpoints.cs
│   └── AuthEndpoints.cs
├── Filters/
│   ├── ValidationFilter.cs
│   └── AuditFilter.cs
├── Models/
│   ├── Product.cs
│   ├── Category.cs
│   └── Order.cs
├── DTOs/
│   ├── ProductDtos.cs
│   └── OrderDtos.cs
├── Services/
│   ├── IProductService.cs
│   └── ProductService.cs
├── Validators/
│   └── ProductValidators.cs
└── Program.cs
```

### DTOs

```csharp
// DTOs/ProductDtos.cs
using System.ComponentModel.DataAnnotations;

namespace MinimalApiEcommerce.DTOs;

public record ProductDto(
    int Id,
    string Name,
    string Description,
    decimal Price,
    int Stock,
    string CategoryName,
    bool IsActive,
    DateTime CreatedAt
);

public record CreateProductRequest(
    [Required, StringLength(200, MinimumLength = 3)] string Name,
    string? Description,
    [Range(0.01, 9_999_999)] decimal Price,
    [Range(0, 1_000_000)] int Stock,
    [Range(1, int.MaxValue)] int CategoryId
);

public record UpdateProductRequest(
    string? Name,
    string? Description,
    decimal? Price,
    int? Stock
);

public record PagedResult<T>(
    List<T> Items,
    int TotalCount,
    int Page,
    int PageSize,
    bool HasNextPage,
    bool HasPreviousPage
);

public record ProductSearchQuery(
    int Page = 1,
    int PageSize = 20,
    string? Search = null,
    int? CategoryId = null,
    decimal? MinPrice = null,
    decimal? MaxPrice = null,
    bool? InStock = null
);
```

### Product Service

```csharp
// Services/ProductService.cs
using Microsoft.EntityFrameworkCore;
using MinimalApiEcommerce.Data;
using MinimalApiEcommerce.DTOs;
using MinimalApiEcommerce.Models;

namespace MinimalApiEcommerce.Services;

public interface IProductService
{
    Task<PagedResult<ProductDto>> GetAllAsync(ProductSearchQuery query);
    Task<ProductDto?> GetByIdAsync(int id);
    Task<ProductDto> CreateAsync(CreateProductRequest request);
    Task<ProductDto?> UpdateAsync(int id, UpdateProductRequest request);
    Task<bool> DeleteAsync(int id);
}

public class ProductService : IProductService
{
    private readonly AppDbContext _db;

    public ProductService(AppDbContext db)
    {
        _db = db;
    }

    public async Task<PagedResult<ProductDto>> GetAllAsync(ProductSearchQuery query)
    {
        var dbQuery = _db.Products
            .Include(p => p.Category)
            .Where(p => p.IsActive)
            .AsQueryable();

        if (!string.IsNullOrEmpty(query.Search))
            dbQuery = dbQuery.Where(p =>
                p.Name.Contains(query.Search) ||
                p.Description.Contains(query.Search));

        if (query.CategoryId.HasValue)
            dbQuery = dbQuery.Where(p => p.CategoryId == query.CategoryId);

        if (query.MinPrice.HasValue)
            dbQuery = dbQuery.Where(p => p.Price >= query.MinPrice);

        if (query.MaxPrice.HasValue)
            dbQuery = dbQuery.Where(p => p.Price <= query.MaxPrice);

        if (query.InStock == true)
            dbQuery = dbQuery.Where(p => p.Stock > 0);

        var totalCount = await dbQuery.CountAsync();

        var products = await dbQuery
            .OrderBy(p => p.Name)
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .Select(p => new ProductDto(
                p.Id,
                p.Name,
                p.Description,
                p.Price,
                p.Stock,
                p.Category!.Name,
                p.IsActive,
                p.CreatedAt))
            .ToListAsync();

        return new PagedResult<ProductDto>(
            products,
            totalCount,
            query.Page,
            query.PageSize,
            query.Page * query.PageSize < totalCount,
            query.Page > 1);
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        return await _db.Products
            .Include(p => p.Category)
            .Where(p => p.Id == id && p.IsActive)
            .Select(p => new ProductDto(
                p.Id, p.Name, p.Description, p.Price,
                p.Stock, p.Category!.Name, p.IsActive, p.CreatedAt))
            .FirstOrDefaultAsync();
    }

    public async Task<ProductDto> CreateAsync(CreateProductRequest request)
    {
        var product = new Product
        {
            Name = request.Name,
            Description = request.Description ?? string.Empty,
            Price = request.Price,
            Stock = request.Stock,
            CategoryId = request.CategoryId,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };

        _db.Products.Add(product);
        await _db.SaveChangesAsync();

        return (await GetByIdAsync(product.Id))!;
    }

    public async Task<ProductDto?> UpdateAsync(int id, UpdateProductRequest request)
    {
        var product = await _db.Products.FindAsync(id);
        if (product is null || !product.IsActive) return null;

        if (request.Name is not null) product.Name = request.Name;
        if (request.Description is not null) product.Description = request.Description;
        if (request.Price.HasValue) product.Price = request.Price.Value;
        if (request.Stock.HasValue) product.Stock = request.Stock.Value;

        await _db.SaveChangesAsync();

        return await GetByIdAsync(id);
    }

    public async Task<bool> DeleteAsync(int id)
    {
        var product = await _db.Products.FindAsync(id);
        if (product is null) return false;

        product.IsActive = false;
        await _db.SaveChangesAsync();
        return true;
    }
}
```

### Complete Program.cs

```csharp
// Program.cs
using System.Security.Claims;
using FluentValidation;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using MinimalApiEcommerce.Data;
using MinimalApiEcommerce.DTOs;
using MinimalApiEcommerce.Endpoints;
using MinimalApiEcommerce.Filters;
using MinimalApiEcommerce.Services;
using MinimalApiEcommerce.Validators;

var builder = WebApplication.CreateBuilder(args);

// ============ Services ============
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=ecommerce.db"));

builder.Services.AddScoped<IProductService, ProductService>();

// Validators
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                System.Text.Encoding.UTF8.GetBytes(
                    builder.Configuration["Jwt:Secret"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminPolicy", policy =>
        policy.RequireClaim(ClaimTypes.Role, "Admin"));
});

// Output Caching
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("Products", policy =>
        policy.Expire(TimeSpan.FromMinutes(5))
              .Tag("products")
              .VaryByQuery("page", "pageSize", "search", "categoryId"));
});

// Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddSlidingWindowLimiter("api", opts =>
    {
        opts.PermitLimit = 100;
        opts.Window = TimeSpan.FromMinutes(1);
        opts.SegmentsPerWindow = 6;
    });
});

var app = builder.Build();

// ============ Middleware ============
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseOutputCache();
app.UseRateLimiter();

// ============ Endpoints ============

// Root group
var api = app.MapGroup("/api/v1")
    .WithOpenApi()
    .AddEndpointFilter<AuditFilter>()
    .RequireRateLimiting("api");

// Products
api.MapGroup("/products")
   .WithTags("Products")
   .MapProductEndpoints();

// Categories
api.MapGroup("/categories")
   .WithTags("Categories")
   .MapCategoryEndpoints();

// Orders
api.MapGroup("/orders")
   .WithTags("Orders")
   .RequireAuthorization()
   .MapOrderEndpoints();

// Auth endpoints (no rate limiting on auth group for login)
app.MapGroup("/api/v1/auth")
   .WithTags("Authentication")
   .MapAuthEndpoints();

// Health check
app.MapGet("/health", () => TypedResults.Ok(new
{
    Status = "Healthy",
    Timestamp = DateTime.UtcNow
})).WithTags("Health");

// Initialize database
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
await db.Database.MigrateAsync();

app.Run();
```

### Auth Endpoints

```csharp
// Endpoints/AuthEndpoints.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.IdentityModel.Tokens;

namespace MinimalApiEcommerce.Endpoints;

public static class AuthEndpoints
{
    public static RouteGroupBuilder MapAuthEndpoints(this RouteGroupBuilder group)
    {
        group.MapPost("/login", Login)
             .WithName("Login")
             .WithSummary("เข้าสู่ระบบ")
             .AllowAnonymous();

        group.MapPost("/register", Register)
             .WithName("Register")
             .WithSummary("สมัครสมาชิก")
             .AllowAnonymous();

        group.MapPost("/refresh", RefreshToken)
             .WithName("RefreshToken")
             .WithSummary("ต่ออายุ token")
             .RequireAuthorization();

        return group;
    }

    private static async Task<Results<Ok<LoginResponse>, UnauthorizedHttpResult>> Login(
        LoginRequest request,
        IConfiguration configuration)
    {
        // จำลอง user lookup (ควรใช้ real user service)
        if (request.Email != "admin@example.com" || request.Password != "password123")
            return TypedResults.Unauthorized();

        var token = GenerateToken(
            userId: "1",
            email: request.Email,
            role: "Admin",
            configuration: configuration);

        return TypedResults.Ok(new LoginResponse(
            Token: token,
            ExpiresIn: 3600,
            Email: request.Email
        ));
    }

    private static async Task<Results<Created<string>, ValidationProblem>> Register(
        RegisterRequest request)
    {
        // Validate
        if (request.Password != request.ConfirmPassword)
        {
            return TypedResults.ValidationProblem(new Dictionary<string, string[]>
            {
                ["confirmPassword"] = new[] { "รหัสผ่านไม่ตรงกัน" }
            });
        }

        // TODO: สร้าง user จริงๆ
        return TypedResults.Created("/api/v1/users/1", "User created");
    }

    private static Ok<LoginResponse> RefreshToken(
        ClaimsPrincipal user,
        IConfiguration configuration)
    {
        var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? "0";
        var email = user.FindFirst(ClaimTypes.Email)?.Value ?? "";
        var role = user.FindFirst(ClaimTypes.Role)?.Value ?? "User";

        var token = GenerateToken(userId, email, role, configuration);

        return TypedResults.Ok(new LoginResponse(token, 3600, email));
    }

    private static string GenerateToken(
        string userId,
        string email,
        string role,
        IConfiguration configuration)
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId),
            new Claim(ClaimTypes.Email, email),
            new Claim(ClaimTypes.Role, role),
            new Claim("sub", userId)
        };

        var key = new SymmetricSecurityKey(
            System.Text.Encoding.UTF8.GetBytes(configuration["Jwt:Secret"]!));

        var token = new JwtSecurityToken(
            issuer: configuration["Jwt:Issuer"],
            audience: configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}

public record LoginRequest(string Email, string Password);
public record RegisterRequest(string Email, string Password, string ConfirmPassword, string Name);
public record LoginResponse(string Token, int ExpiresIn, string Email);
```

### Audit Filter

```csharp
// Filters/AuditFilter.cs
namespace MinimalApiEcommerce.Filters;

public class AuditFilter : IEndpointFilter
{
    private readonly ILogger<AuditFilter> _logger;

    public AuditFilter(ILogger<AuditFilter> logger)
    {
        _logger = logger;
    }

    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var httpContext = context.HttpContext;
        var userId = httpContext.User.FindFirst("sub")?.Value ?? "anonymous";
        var requestId = httpContext.TraceIdentifier;

        using var scope = _logger.BeginScope(new Dictionary<string, object>
        {
            ["UserId"] = userId,
            ["RequestId"] = requestId
        });

        var result = await next(context);

        // Log mutations
        if (httpContext.Request.Method != "GET")
        {
            _logger.LogInformation(
                "API Audit: {Method} {Path} by {UserId}",
                httpContext.Request.Method,
                httpContext.Request.Path,
                userId);
        }

        return result;
    }
}
```

---

## Exercises

### Exercise 1: Versioning ใน Minimal API
เพิ่ม API versioning ใน Minimal API โดยใช้ Route Groups

### Exercise 2: HATEOAS
เพิ่ม HATEOAS links ใน responses ของ Minimal API

### Exercise 3: Batch Operations
สร้าง endpoint สำหรับ batch create/update/delete operations

### Exercise 4: GraphQL-like Query
สร้าง flexible query endpoint ที่รองรับ field selection เหมือน GraphQL

### Exercise 5: WebSocket Endpoint
เพิ่ม WebSocket support ใน Minimal API สำหรับ real-time updates

---

## สรุป

- **Route Groups** ช่วยจัดระเบียบ endpoints และลด code duplication
- **Endpoint Filters** เพิ่ม cross-cutting concerns แบบ reusable
- **TypedResults** ให้ type safety และช่วย Swagger/OpenAPI generate documentation ถูกต้อง
- **FluentValidation** หรือ Data Annotations ใช้ validate ได้ใน Minimal API
- Minimal API เหมาะกับ microservices และ simple APIs ที่ต้องการ performance สูง

---

## Phase 4 สรุป

Part 061-070 ครอบคลุม:
- SignalR สำหรับ real-time communication
- Background Services
- Caching (Memory, Distributed, Redis)
- Logging ด้วย Serilog
- Health Checks
- API Documentation ด้วย Swagger
- File Upload/Download
- Email Service
- Security (CORS, Headers, Rate Limiting)
- Minimal API ขั้นสูง

---

## Part ถัดไป

**Part 071: Testing ใน ASP.NET Core** - เรียนรู้การ Unit Test, Integration Test, และ End-to-End Test

---

*Part 070/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
