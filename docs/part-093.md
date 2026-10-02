# Part 093: Real-world Project: E-commerce API

## เนื้อหาใน Part นี้
- Full project structure
- Product catalog
- Shopping cart
- Order processing
- Payment integration
- Complete API สมบูรณ์

---

## 1. Project Overview

สร้าง E-commerce REST API ที่ production-ready ด้วย .NET 9

### Feature Requirements
- Product catalog พร้อม categories, search, filtering
- Shopping cart ที่ persist
- Order management lifecycle
- Payment integration (Stripe-style)
- User authentication/authorization
- Admin panel features

### Tech Stack
```
- ASP.NET Core 9 Minimal API
- PostgreSQL + EF Core 9
- Redis (caching + cart storage)
- JWT Authentication
- MediatR (CQRS)
- FluentValidation
- Serilog
- Docker
```

---

## 2. Project Structure

```
EcommerceAPI/
├── src/
│   ├── EcommerceAPI.API/
│   │   ├── Endpoints/
│   │   │   ├── ProductEndpoints.cs
│   │   │   ├── CartEndpoints.cs
│   │   │   ├── OrderEndpoints.cs
│   │   │   └── AuthEndpoints.cs
│   │   ├── Middleware/
│   │   │   └── ExceptionHandlingMiddleware.cs
│   │   └── Program.cs
│   ├── EcommerceAPI.Application/
│   │   ├── Features/
│   │   │   ├── Products/
│   │   │   ├── Cart/
│   │   │   ├── Orders/
│   │   │   └── Auth/
│   │   └── Common/
│   ├── EcommerceAPI.Domain/
│   │   ├── Entities/
│   │   ├── Events/
│   │   └── Exceptions/
│   └── EcommerceAPI.Infrastructure/
│       ├── Data/
│       ├── Repositories/
│       ├── Services/
│       └── Caching/
├── tests/
│   ├── EcommerceAPI.UnitTests/
│   └── EcommerceAPI.IntegrationTests/
├── Dockerfile
└── docker-compose.yml
```

---

## 3. Domain Models

```csharp
// Domain/Entities/Product.cs
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public string Description { get; private set; } = string.Empty;
    public decimal Price { get; private set; }
    public decimal? SalePrice { get; private set; }
    public string SKU { get; private set; } = string.Empty;
    public string Category { get; private set; } = string.Empty;
    public string Brand { get; private set; } = string.Empty;
    public int StockQuantity { get; private set; }
    public List<string> ImageUrls { get; private set; } = new();
    public List<string> Tags { get; private set; } = new();
    public bool IsActive { get; private set; }
    public decimal Rating { get; private set; }
    public int ReviewCount { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private Product() { }

    public static Product Create(
        string name, string description, decimal price, 
        string sku, string category, string brand, int stock)
    {
        return new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Description = description,
            Price = price,
            SKU = sku,
            Category = category,
            Brand = brand,
            StockQuantity = stock,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void UpdatePrice(decimal newPrice, decimal? salePrice = null)
    {
        if (newPrice <= 0) throw new DomainException("Price must be positive");
        Price = newPrice;
        SalePrice = salePrice;
        UpdatedAt = DateTime.UtcNow;
    }

    public void AdjustStock(int delta)
    {
        var newStock = StockQuantity + delta;
        if (newStock < 0) throw new DomainException("Stock cannot be negative");
        StockQuantity = newStock;
        UpdatedAt = DateTime.UtcNow;
    }

    public decimal GetEffectivePrice() => SalePrice ?? Price;
    public bool IsInStock() => StockQuantity > 0;
}

// Domain/Entities/Order.cs
public class Order
{
    public Guid Id { get; private set; }
    public string OrderNumber { get; private set; } = string.Empty;
    public Guid CustomerId { get; private set; }
    public List<OrderItem> Items { get; private set; } = new();
    public OrderStatus Status { get; private set; }
    public Address ShippingAddress { get; private set; } = null!;
    public decimal SubTotal { get; private set; }
    public decimal ShippingFee { get; private set; }
    public decimal Discount { get; private set; }
    public decimal Total { get; private set; }
    public string? CouponCode { get; private set; }
    public PaymentInfo? Payment { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private Order() { }

    public static Order Create(Guid customerId, List<OrderItem> items, Address shippingAddress)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            OrderNumber = GenerateOrderNumber(),
            CustomerId = customerId,
            Items = items,
            Status = OrderStatus.PendingPayment,
            ShippingAddress = shippingAddress,
            CreatedAt = DateTime.UtcNow
        };
        
        order.RecalculateTotals();
        return order;
    }

    public void ApplyCoupon(string couponCode, decimal discountAmount)
    {
        CouponCode = couponCode;
        Discount = discountAmount;
        RecalculateTotals();
    }

    public void ConfirmPayment(string paymentIntentId, string paymentMethod)
    {
        if (Status != OrderStatus.PendingPayment)
            throw new DomainException($"Cannot confirm payment for {Status} order");
        
        Status = OrderStatus.Processing;
        Payment = new PaymentInfo(paymentIntentId, paymentMethod, Total, DateTime.UtcNow);
        UpdatedAt = DateTime.UtcNow;
    }

    public void Ship(string trackingNumber)
    {
        if (Status != OrderStatus.Processing)
            throw new DomainException($"Cannot ship {Status} order");
        
        Status = OrderStatus.Shipped;
        UpdatedAt = DateTime.UtcNow;
    }

    public void Deliver() 
    {
        Status = OrderStatus.Delivered;
        UpdatedAt = DateTime.UtcNow;
    }

    public void Cancel(string reason)
    {
        if (Status == OrderStatus.Delivered)
            throw new DomainException("Cannot cancel delivered order");
        if (Status == OrderStatus.Cancelled)
            throw new DomainException("Order already cancelled");
        
        Status = OrderStatus.Cancelled;
        UpdatedAt = DateTime.UtcNow;
    }

    private void RecalculateTotals()
    {
        SubTotal = Items.Sum(i => i.UnitPrice * i.Quantity);
        ShippingFee = SubTotal >= 1000 ? 0 : 50;  // Free shipping over 1000 THB
        Total = SubTotal + ShippingFee - Discount;
    }

    private static string GenerateOrderNumber() => 
        $"ORD{DateTime.UtcNow:yyyyMMdd}{Random.Shared.Next(10000, 99999)}";
}

public enum OrderStatus 
{ 
    PendingPayment, 
    Processing, 
    Shipped, 
    Delivered, 
    Cancelled, 
    Refunded 
}

public record OrderItem(
    Guid ProductId, 
    string ProductName, 
    decimal UnitPrice, 
    int Quantity,
    string? SKU = null);

public record Address(
    string Street, string City, string State, 
    string ZipCode, string Country, string? Phone = null);

public record PaymentInfo(
    string PaymentIntentId, 
    string PaymentMethod, 
    decimal Amount, 
    DateTime PaidAt);
```

---

## 4. Application Layer (CQRS)

```csharp
// Application/Features/Products/Queries/GetProductsQuery.cs
public record GetProductsQuery : IRequest<PagedResult<ProductDto>>
{
    public string? Search { get; init; }
    public string? Category { get; init; }
    public string? Brand { get; init; }
    public decimal? MinPrice { get; init; }
    public decimal? MaxPrice { get; init; }
    public bool? InStock { get; init; }
    public string? SortBy { get; init; }
    public bool Descending { get; init; }
    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = 20;
}

public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, PagedResult<ProductDto>>
{
    private readonly IProductRepository _repo;
    private readonly ICacheService _cache;

    public GetProductsQueryHandler(IProductRepository repo, ICacheService cache)
    {
        _repo = repo;
        _cache = cache;
    }

    public async Task<PagedResult<ProductDto>> Handle(
        GetProductsQuery query, CancellationToken ct)
    {
        var cacheKey = $"products:{query.GetHashCode()}";
        
        // Try cache only for simple queries
        if (query.Search == null)
        {
            var cached = await _cache.GetAsync<PagedResult<ProductDto>>(cacheKey);
            if (cached != null) return cached;
        }
        
        var result = await _repo.GetPagedAsync(new ProductFilter
        {
            Search = query.Search,
            Category = query.Category,
            Brand = query.Brand,
            MinPrice = query.MinPrice,
            MaxPrice = query.MaxPrice,
            InStockOnly = query.InStock ?? false,
            SortBy = query.SortBy ?? "name",
            Descending = query.Descending,
            Page = query.Page,
            PageSize = Math.Min(query.PageSize, 100)
        }, ct);
        
        if (query.Search == null)
            await _cache.SetAsync(cacheKey, result, TimeSpan.FromMinutes(5));
        
        return result;
    }
}

// Application/Features/Cart/Commands/AddToCartCommand.cs
public record AddToCartCommand(Guid CustomerId, Guid ProductId, int Quantity) : IRequest<CartDto>;

public class AddToCartCommandHandler : IRequestHandler<AddToCartCommand, CartDto>
{
    private readonly ICartRepository _cartRepo;
    private readonly IProductRepository _productRepo;

    public AddToCartCommandHandler(ICartRepository cartRepo, IProductRepository productRepo)
    {
        _cartRepo = cartRepo;
        _productRepo = productRepo;
    }

    public async Task<CartDto> Handle(AddToCartCommand command, CancellationToken ct)
    {
        // Validate product
        var product = await _productRepo.GetByIdAsync(command.ProductId, ct)
            ?? throw new NotFoundException($"Product {command.ProductId} not found");
        
        if (!product.IsActive)
            throw new DomainException("Product is not available");
        
        if (!product.IsInStock())
            throw new DomainException("Product is out of stock");
        
        if (command.Quantity > product.StockQuantity)
            throw new DomainException($"Only {product.StockQuantity} items available");
        
        // Get or create cart
        var cart = await _cartRepo.GetOrCreateAsync(command.CustomerId, ct);
        
        cart.AddItem(
            command.ProductId,
            product.Name,
            product.GetEffectivePrice(),
            command.Quantity,
            product.ImageUrls.FirstOrDefault());
        
        await _cartRepo.SaveAsync(cart, ct);
        
        return CartDto.From(cart);
    }
}

// Cart
public class ShoppingCart
{
    public Guid CustomerId { get; set; }
    public List<CartItem> Items { get; set; } = new();
    public DateTime UpdatedAt { get; set; }

    public void AddItem(Guid productId, string name, decimal price, int quantity, string? imageUrl)
    {
        var existing = Items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
        {
            existing.Quantity += quantity;
        }
        else
        {
            Items.Add(new CartItem(productId, name, price, quantity, imageUrl));
        }
        UpdatedAt = DateTime.UtcNow;
    }

    public void RemoveItem(Guid productId)
    {
        Items.RemoveAll(i => i.ProductId == productId);
        UpdatedAt = DateTime.UtcNow;
    }

    public void UpdateQuantity(Guid productId, int quantity)
    {
        var item = Items.FirstOrDefault(i => i.ProductId == productId);
        if (item != null)
        {
            if (quantity <= 0)
                Items.Remove(item);
            else
                item.Quantity = quantity;
            UpdatedAt = DateTime.UtcNow;
        }
    }

    public decimal GetTotal() => Items.Sum(i => i.Price * i.Quantity);
    public bool IsEmpty() => !Items.Any();
}

public class CartItem
{
    public Guid ProductId { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Quantity { get; set; }
    public string? ImageUrl { get; set; }
    
    public CartItem(Guid productId, string name, decimal price, int quantity, string? imageUrl)
    {
        ProductId = productId;
        Name = name;
        Price = price;
        Quantity = quantity;
        ImageUrl = imageUrl;
    }
}
```

---

## 5. Checkout และ Payment

```csharp
// Application/Features/Orders/Commands/CheckoutCommand.cs
public record CheckoutCommand(
    Guid CustomerId,
    Address ShippingAddress,
    string PaymentMethodId,
    string? CouponCode = null) : IRequest<OrderDto>;

public class CheckoutCommandHandler : IRequestHandler<CheckoutCommand, OrderDto>
{
    private readonly ICartRepository _cartRepo;
    private readonly IProductRepository _productRepo;
    private readonly IOrderRepository _orderRepo;
    private readonly IPaymentService _paymentService;
    private readonly ICouponService _couponService;
    private readonly ILogger<CheckoutCommandHandler> _logger;

    public CheckoutCommandHandler(
        ICartRepository cartRepo,
        IProductRepository productRepo,
        IOrderRepository orderRepo,
        IPaymentService paymentService,
        ICouponService couponService,
        ILogger<CheckoutCommandHandler> logger)
    {
        _cartRepo = cartRepo;
        _productRepo = productRepo;
        _orderRepo = orderRepo;
        _paymentService = paymentService;
        _couponService = couponService;
        _logger = logger;
    }

    public async Task<OrderDto> Handle(CheckoutCommand command, CancellationToken ct)
    {
        // Get cart
        var cart = await _cartRepo.GetOrCreateAsync(command.CustomerId, ct);
        
        if (cart.IsEmpty())
            throw new DomainException("Cart is empty");
        
        // Validate and get current prices
        var orderItems = new List<OrderItem>();
        foreach (var cartItem in cart.Items)
        {
            var product = await _productRepo.GetByIdAsync(cartItem.ProductId, ct)
                ?? throw new NotFoundException($"Product {cartItem.ProductId} not found");
            
            if (!product.IsActive)
                throw new DomainException($"Product '{product.Name}' is no longer available");
            
            if (product.StockQuantity < cartItem.Quantity)
                throw new DomainException($"Insufficient stock for '{product.Name}'. Available: {product.StockQuantity}");
            
            orderItems.Add(new OrderItem(
                product.Id,
                product.Name,
                product.GetEffectivePrice(),
                cartItem.Quantity,
                product.SKU));
        }
        
        // Create order
        var order = Order.Create(command.CustomerId, orderItems, command.ShippingAddress);
        
        // Apply coupon
        if (!string.IsNullOrEmpty(command.CouponCode))
        {
            var discount = await _couponService.ValidateAndGetDiscountAsync(
                command.CouponCode, order.SubTotal, ct);
            
            if (discount > 0)
                order.ApplyCoupon(command.CouponCode, discount);
        }
        
        // Process payment
        var paymentResult = await _paymentService.ProcessPaymentAsync(new ProcessPaymentRequest
        {
            Amount = order.Total,
            Currency = "THB",
            PaymentMethodId = command.PaymentMethodId,
            OrderId = order.Id.ToString(),
            CustomerId = command.CustomerId.ToString()
        }, ct);
        
        if (!paymentResult.Success)
        {
            _logger.LogWarning("Payment failed for order {OrderId}: {Error}", 
                order.Id, paymentResult.ErrorMessage);
            throw new PaymentFailedException(paymentResult.ErrorMessage ?? "Payment failed");
        }
        
        // Confirm payment and reserve stock
        order.ConfirmPayment(paymentResult.PaymentIntentId!, paymentResult.PaymentMethod!);
        
        foreach (var item in orderItems)
        {
            await _productRepo.AdjustStockAsync(item.ProductId, -item.Quantity, ct);
        }
        
        await _orderRepo.SaveAsync(order, ct);
        
        // Clear cart
        await _cartRepo.DeleteAsync(command.CustomerId, ct);
        
        _logger.LogInformation(
            "Order {OrderNumber} created for customer {CustomerId}, total: {Total:C}",
            order.OrderNumber, command.CustomerId, order.Total);
        
        return OrderDto.From(order);
    }
}

// Payment Service
public interface IPaymentService
{
    Task<PaymentResult> ProcessPaymentAsync(ProcessPaymentRequest request, CancellationToken ct);
    Task<PaymentResult> RefundAsync(string paymentIntentId, decimal amount, CancellationToken ct);
}

public class StripePaymentService : IPaymentService
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _config;
    private readonly ILogger<StripePaymentService> _logger;

    public StripePaymentService(
        HttpClient httpClient, 
        IConfiguration config,
        ILogger<StripePaymentService> logger)
    {
        _httpClient = httpClient;
        _config = config;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(
        ProcessPaymentRequest request, CancellationToken ct)
    {
        try
        {
            // Simulate Stripe API call
            var response = await _httpClient.PostAsJsonAsync("/v1/payment_intents", new
            {
                amount = (int)(request.Amount * 100),  // Stripe uses smallest currency unit
                currency = request.Currency.ToLower(),
                payment_method = request.PaymentMethodId,
                confirm = true,
                metadata = new { order_id = request.OrderId, customer_id = request.CustomerId }
            }, ct);
            
            if (!response.IsSuccessStatusCode)
            {
                var error = await response.Content.ReadFromJsonAsync<StripeError>(cancellationToken: ct);
                return PaymentResult.Failure(error?.Message ?? "Payment declined");
            }
            
            var intent = await response.Content.ReadFromJsonAsync<StripePaymentIntent>(cancellationToken: ct);
            
            return PaymentResult.Success(
                intent?.Id ?? Guid.NewGuid().ToString(),
                intent?.PaymentMethod ?? request.PaymentMethodId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment processing failed for order {OrderId}", request.OrderId);
            return PaymentResult.Failure("Payment processing error");
        }
    }

    public async Task<PaymentResult> RefundAsync(
        string paymentIntentId, decimal amount, CancellationToken ct)
    {
        var response = await _httpClient.PostAsJsonAsync("/v1/refunds", new
        {
            payment_intent = paymentIntentId,
            amount = (int)(amount * 100)
        }, ct);
        
        return response.IsSuccessStatusCode
            ? PaymentResult.Success(paymentIntentId, "refund")
            : PaymentResult.Failure("Refund failed");
    }
}

public class PaymentResult
{
    public bool Success { get; private set; }
    public string? PaymentIntentId { get; private set; }
    public string? PaymentMethod { get; private set; }
    public string? ErrorMessage { get; private set; }
    
    private PaymentResult() { }
    
    public static PaymentResult Success(string paymentIntentId, string paymentMethod) => new()
    {
        Success = true,
        PaymentIntentId = paymentIntentId,
        PaymentMethod = paymentMethod
    };
    
    public static PaymentResult Failure(string errorMessage) => new()
    {
        Success = false,
        ErrorMessage = errorMessage
    };
}
```

---

## 6. API Endpoints

```csharp
// Endpoints/ProductEndpoints.cs
public static class ProductEndpoints
{
    public static void MapProductEndpoints(this IEndpointRouteBuilder app)
    {
        var products = app.MapGroup("/api/products").WithTags("Products");
        
        products.MapGet("/", GetProducts);
        products.MapGet("/{id:guid}", GetProduct);
        products.MapGet("/search", SearchProducts);
        products.MapGet("/categories", GetCategories);
        
        var adminProducts = products.MapGroup("/admin").RequireAuthorization("Admin");
        adminProducts.MapPost("/", CreateProduct);
        adminProducts.MapPut("/{id:guid}", UpdateProduct);
        adminProducts.MapDelete("/{id:guid}", DeleteProduct);
        adminProducts.MapPatch("/{id:guid}/stock", UpdateStock);
    }
    
    private static async Task<IResult> GetProducts(
        [AsParameters] GetProductsQuery query,
        ISender sender)
    {
        var result = await sender.Send(query);
        return Results.Ok(result);
    }
    
    private static async Task<IResult> GetProduct(Guid id, ISender sender)
    {
        var query = new GetProductByIdQuery(id);
        var product = await sender.Send(query);
        return product == null ? Results.NotFound() : Results.Ok(product);
    }
    
    private static async Task<IResult> CreateProduct(
        CreateProductRequest request,
        ISender sender)
    {
        var command = new CreateProductCommand(
            request.Name, request.Description, request.Price,
            request.SKU, request.Category, request.Brand, request.InitialStock);
        
        var product = await sender.Send(command);
        return Results.Created($"/api/products/{product.Id}", product);
    }
    
    private static async Task<IResult> UpdateStock(
        Guid id,
        UpdateStockRequest request,
        ISender sender)
    {
        await sender.Send(new UpdateStockCommand(id, request.Delta));
        return Results.Ok();
    }
}

// Endpoints/CartEndpoints.cs
public static class CartEndpoints
{
    public static void MapCartEndpoints(this IEndpointRouteBuilder app)
    {
        var cart = app.MapGroup("/api/cart")
            .RequireAuthorization()
            .WithTags("Cart");
        
        cart.MapGet("/", GetCart);
        cart.MapPost("/items", AddToCart);
        cart.MapPut("/items/{productId:guid}", UpdateCartItem);
        cart.MapDelete("/items/{productId:guid}", RemoveFromCart);
        cart.MapDelete("/", ClearCart);
        cart.MapPost("/checkout", Checkout);
    }
    
    private static async Task<IResult> GetCart(HttpContext ctx, ISender sender)
    {
        var customerId = ctx.GetUserId();
        var cart = await sender.Send(new GetCartQuery(customerId));
        return Results.Ok(cart);
    }
    
    private static async Task<IResult> AddToCart(
        AddToCartRequest request,
        HttpContext ctx,
        ISender sender)
    {
        var customerId = ctx.GetUserId();
        var cart = await sender.Send(new AddToCartCommand(customerId, request.ProductId, request.Quantity));
        return Results.Ok(cart);
    }
    
    private static async Task<IResult> Checkout(
        CheckoutRequest request,
        HttpContext ctx,
        ISender sender)
    {
        var customerId = ctx.GetUserId();
        var order = await sender.Send(new CheckoutCommand(
            customerId,
            new Address(request.Street, request.City, request.State, request.ZipCode, request.Country),
            request.PaymentMethodId,
            request.CouponCode));
        
        return Results.Created($"/api/orders/{order.Id}", order);
    }
}

// Endpoints/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var orders = app.MapGroup("/api/orders")
            .RequireAuthorization()
            .WithTags("Orders");
        
        orders.MapGet("/", GetMyOrders);
        orders.MapGet("/{id:guid}", GetOrder);
        orders.MapPost("/{id:guid}/cancel", CancelOrder);
        
        var adminOrders = orders.MapGroup("/admin").RequireAuthorization("Admin");
        adminOrders.MapGet("/", GetAllOrders);
        adminOrders.MapPatch("/{id:guid}/status", UpdateOrderStatus);
        adminOrders.MapPost("/{id:guid}/ship", ShipOrder);
    }
    
    private static async Task<IResult> GetMyOrders(
        HttpContext ctx,
        ISender sender,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var customerId = ctx.GetUserId();
        var orders = await sender.Send(new GetCustomerOrdersQuery(customerId, page, pageSize));
        return Results.Ok(orders);
    }
    
    private static async Task<IResult> GetOrder(
        Guid id,
        HttpContext ctx,
        ISender sender)
    {
        var customerId = ctx.GetUserId();
        var order = await sender.Send(new GetOrderQuery(id, customerId));
        return order == null ? Results.NotFound() : Results.Ok(order);
    }
    
    private static async Task<IResult> CancelOrder(
        Guid id,
        CancelOrderRequest request,
        HttpContext ctx,
        ISender sender)
    {
        var customerId = ctx.GetUserId();
        await sender.Send(new CancelOrderCommand(id, customerId, request.Reason));
        return Results.Ok();
    }
}
```

---

## 7. Program.cs สมบูรณ์

```csharp
// Program.cs
using MediatR;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Logging
builder.Host.UseSerilog((ctx, config) =>
    config.ReadFrom.Configuration(ctx.Configuration)
          .Enrich.FromLogContext());

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Redis
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = builder.Configuration["Redis:ConnectionString"]);

// MediatR
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<GetProductsQuery>());

// FluentValidation
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductCommandValidator>();

// Repositories
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<ICartRepository, RedisCartRepository>();

// Services
builder.Services.AddScoped<ICacheService, RedisCacheService>();
builder.Services.AddScoped<ICouponService, CouponService>();
builder.Services.AddHttpClient<IPaymentService, StripePaymentService>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Stripe:BaseUrl"]!);
    client.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue(
        "Bearer", builder.Configuration["Stripe:SecretKey"]);
});

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Admin", policy => policy.RequireRole("Admin"));
});

// API
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo { Title = "E-commerce API", Version = "v1" });
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer"
    });
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        [new OpenApiSecurityScheme { Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }}] 
        = Array.Empty<string>()
    });
});

// Health checks
builder.Services.AddHealthChecks()
    .AddNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!);

// Exception handling
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();

// Migrate DB
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}

app.UseExceptionHandler();
app.UseSwagger();
app.UseSwaggerUI();
app.UseSerilogRequestLogging();
app.UseAuthentication();
app.UseAuthorization();

// Map endpoints
app.MapProductEndpoints();
app.MapCartEndpoints();
app.MapOrderEndpoints();
app.MapAuthEndpoints();
app.MapHealthChecks("/health");

app.Run();

// Global exception handler
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    
    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }
    
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        var (statusCode, title) = exception switch
        {
            NotFoundException => (StatusCodes.Status404NotFound, "Not Found"),
            DomainException => (StatusCodes.Status400BadRequest, "Bad Request"),
            PaymentFailedException => (StatusCodes.Status402PaymentRequired, "Payment Required"),
            ValidationException e => (StatusCodes.Status422UnprocessableEntity, "Validation Error"),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden, "Forbidden"),
            _ => (StatusCodes.Status500InternalServerError, "Internal Server Error")
        };
        
        if (statusCode == 500)
            _logger.LogError(exception, "Unhandled exception");
        
        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = statusCode,
            Title = title,
            Detail = exception.Message
        }, ct);
        
        return true;
    }
}
```

---

## Exercises / Project Tasks

### Exercise 1: Add Product Reviews
เพิ่ม review system:
- ReviewEntity, AddReviewCommand
- GetProductReviews query
- Calculate average rating

### Exercise 2: Coupon System
สร้าง coupon system:
- CouponEntity (code, type, value, expiry)
- Apply coupon at checkout
- Validate constraints (min order, max uses)

### Exercise 3: Order Notifications
เพิ่ม email notifications:
- OrderCreated → send confirmation email
- OrderShipped → send tracking email
- Use background service

### Exercise 4: Admin Dashboard APIs
สร้าง admin endpoints:
- Sales summary (today, this week, this month)
- Top selling products
- Low stock alerts

---

## สรุป

- **Clean Architecture** แยก concerns ชัดเจน: Domain, Application, Infrastructure, API
- **CQRS ด้วย MediatR** แยก read/write concerns
- **Domain-Driven Design** ให้ business logic อยู่ใน Domain layer
- **Repository Pattern** abstract data access
- **Validation** ที่ command level ด้วย FluentValidation
- **Exception Handling** global handler แปลง exceptions เป็น HTTP responses
- **Payment Integration** แบบ resilient พร้อม error handling

---

## Part ถัดไป

**Part 094: Real-world Project: Social Platform** - สร้าง social network API

---

*Part 093/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
