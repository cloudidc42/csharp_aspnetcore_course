# Part 081: Microservices Architecture

## เนื้อหาใน Part นี้
- Monolith vs Microservices
- Service boundaries และ Domain-Driven Design
- Communication patterns (sync/async)
- API Gateway
- Service Discovery
- โปรแกรมตัวอย่าง: Microservices design

---

## 1. Monolith vs Microservices

### Monolithic Architecture

ในสถาปัตยกรรม Monolith ทุก component ของแอปพลิเคชันรวมอยู่ใน codebase เดียวและ deploy เป็นหน่วยเดียว

```
┌─────────────────────────────────────┐
│           Monolith App              │
│  ┌──────────┐  ┌──────────────────┐ │
│  │  Orders  │  │    Products      │ │
│  └──────────┘  └──────────────────┘ │
│  ┌──────────┐  ┌──────────────────┐ │
│  │  Users   │  │    Payments      │ │
│  └──────────┘  └──────────────────┘ │
│           Single Database           │
└─────────────────────────────────────┘
```

**ข้อดีของ Monolith:**
- Simple ในการพัฒนาเริ่มต้น
- Easy deployment
- No network latency ระหว่าง components
- Easier debugging
- ACID transactions ง่าย

**ข้อเสียของ Monolith:**
- Scale ทั้งระบบแม้ต้องการ scale เฉพาะบางส่วน
- Technology lock-in
- Team coupling สูง
- Deployment risk สูง
- Slow CI/CD

### Microservices Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────┐
│  Order  │   │Product  │   │  User   │
│ Service │   │ Service │   │ Service │
│   DB    │   │   DB    │   │   DB    │
└────┬────┘   └────┬────┘   └────┬────┘
     │              │              │
     └──────────────┴──────────────┘
                    │
             Message Bus / API Gateway
```

**ข้อดีของ Microservices:**
- Independent deployment
- Technology diversity
- Fault isolation
- Easier scaling
- Team autonomy

**ข้อเสียของ Microservices:**
- Distributed system complexity
- Network latency
- Data consistency challenges
- Testing complexity
- Operational overhead

---

## 2. Service Boundaries และ Domain-Driven Design

### Bounded Context

แต่ละ Microservice ควรสอดคล้องกับ Bounded Context ใน Domain-Driven Design

```csharp
// OrderService - ดูแลเรื่อง Order domain
namespace OrderService.Domain
{
    public class Order
    {
        public Guid Id { get; private set; }
        public Guid CustomerId { get; private set; }
        public List<OrderItem> Items { get; private set; }
        public OrderStatus Status { get; private set; }
        public Money TotalAmount { get; private set; }
        public DateTime CreatedAt { get; private set; }

        private Order() { }

        public static Order Create(Guid customerId)
        {
            return new Order
            {
                Id = Guid.NewGuid(),
                CustomerId = customerId,
                Items = new List<OrderItem>(),
                Status = OrderStatus.Pending,
                CreatedAt = DateTime.UtcNow
            };
        }

        public void AddItem(Guid productId, string productName, decimal price, int quantity)
        {
            var item = new OrderItem(productId, productName, price, quantity);
            Items.Add(item);
            RecalculateTotal();
        }

        public void Submit()
        {
            if (!Items.Any())
                throw new DomainException("Cannot submit empty order");

            Status = OrderStatus.Submitted;
        }

        public void Confirm()
        {
            if (Status != OrderStatus.Submitted)
                throw new DomainException("Only submitted orders can be confirmed");

            Status = OrderStatus.Confirmed;
        }

        private void RecalculateTotal()
        {
            TotalAmount = new Money(Items.Sum(i => i.Price * i.Quantity));
        }
    }

    public enum OrderStatus
    {
        Pending,
        Submitted,
        Confirmed,
        Processing,
        Shipped,
        Delivered,
        Cancelled
    }

    public record OrderItem(Guid ProductId, string ProductName, decimal Price, int Quantity);
    public record Money(decimal Amount);
}
```

### Service Boundaries ที่ดี

```csharp
// แต่ละ service มี data ของตัวเอง - ไม่แชร์ database
// OrderService
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    // ไม่มี Products หรือ Users tables ที่นี่
}

// ProductService
public class ProductDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }
    // ไม่มี Orders ที่นี่
}

// UserService
public class UserDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<UserProfile> Profiles { get; set; }
    // ไม่มี Orders หรือ Products ที่นี่
}
```

---

## 3. Communication Patterns

### Synchronous Communication (REST/gRPC)

ใช้เมื่อ client ต้องการผลลัพธ์ทันที

```csharp
// OrderService เรียก ProductService แบบ synchronous
public class ProductServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ProductServiceClient> _logger;

    public ProductServiceClient(HttpClient httpClient, ILogger<ProductServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<ProductDto?> GetProductAsync(Guid productId, CancellationToken ct = default)
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/products/{productId}", ct);
            
            if (response.StatusCode == System.Net.HttpStatusCode.NotFound)
                return null;

            response.EnsureSuccessStatusCode();
            return await response.Content.ReadFromJsonAsync<ProductDto>(cancellationToken: ct);
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Failed to get product {ProductId}", productId);
            throw new ServiceUnavailableException("Product service is unavailable", ex);
        }
    }
}

// Registration ใน DI
builder.Services.AddHttpClient<ProductServiceClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:ProductService:Url"]!);
    client.Timeout = TimeSpan.FromSeconds(5);
})
.AddStandardResilienceHandler(); // Polly retry/circuit breaker
```

### Asynchronous Communication (Message Queue)

ใช้เมื่อต้องการ loose coupling และ fault tolerance

```csharp
// Event ที่ส่งระหว่าง services
public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderItemDto> Items { get; init; } = new();
    public decimal TotalAmount { get; init; }
    public DateTime CreatedAt { get; init; }
}

// Publisher - OrderService ส่ง event
public class OrderService
{
    private readonly IEventBus _eventBus;
    private readonly IOrderRepository _repository;

    public OrderService(IEventBus eventBus, IOrderRepository repository)
    {
        _eventBus = eventBus;
        _repository = repository;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderCommand command)
    {
        var order = Order.Create(command.CustomerId);
        
        foreach (var item in command.Items)
            order.AddItem(item.ProductId, item.ProductName, item.Price, item.Quantity);
        
        order.Submit();
        await _repository.SaveAsync(order);

        // Publish event สำหรับ services อื่น
        await _eventBus.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Items = order.Items.Select(i => new OrderItemDto(i.ProductId, i.Price, i.Quantity)).ToList(),
            TotalAmount = order.TotalAmount.Amount,
            CreatedAt = order.CreatedAt
        });

        return order;
    }
}

// Subscriber - InventoryService รับ event
public class OrderCreatedEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IInventoryRepository _inventoryRepo;
    private readonly ILogger<OrderCreatedEventHandler> _logger;

    public OrderCreatedEventHandler(
        IInventoryRepository inventoryRepo,
        ILogger<OrderCreatedEventHandler> logger)
    {
        _inventoryRepo = inventoryRepo;
        _logger = logger;
    }

    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        _logger.LogInformation("Processing OrderCreated event for order {OrderId}", @event.OrderId);

        foreach (var item in @event.Items)
        {
            await _inventoryRepo.ReserveStockAsync(item.ProductId, item.Quantity);
        }
    }
}
```

### Event-Driven Architecture

```csharp
// Interface สำหรับ Event Bus
public interface IEventBus
{
    Task PublishAsync<T>(T @event) where T : class;
    Task SubscribeAsync<T, THandler>() where T : class where THandler : IEventHandler<T>;
}

// Implementation ด้วย MassTransit (RabbitMQ)
public class MassTransitEventBus : IEventBus
{
    private readonly IBus _bus;

    public MassTransitEventBus(IBus bus)
    {
        _bus = bus;
    }

    public async Task PublishAsync<T>(T @event) where T : class
    {
        await _bus.Publish(@event);
    }

    public Task SubscribeAsync<T, THandler>() where T : class where THandler : IEventHandler<T>
    {
        // Handled by MassTransit consumer registration
        return Task.CompletedTask;
    }
}
```

---

## 4. API Gateway

API Gateway ทำหน้าที่เป็น single entry point สำหรับ client

```csharp
// YARP (Yet Another Reverse Proxy) - API Gateway
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// Authentication/Authorization
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });

builder.Services.AddAuthorization();

// Rate Limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", config =>
    {
        config.Window = TimeSpan.FromMinutes(1);
        config.PermitLimit = 100;
        config.QueueLimit = 10;
    });
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapReverseProxy(proxyPipeline =>
{
    // Custom middleware ใน proxy pipeline
    proxyPipeline.Use(async (context, next) =>
    {
        // Add correlation ID
        if (!context.Request.Headers.ContainsKey("X-Correlation-Id"))
        {
            context.Request.Headers["X-Correlation-Id"] = Guid.NewGuid().ToString();
        }
        await next();
    });
});

app.Run();
```

```json
// appsettings.json - YARP Configuration
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          { "PathRemovePrefix": "/api/orders" },
          { "RequestHeader": "X-Service-Name", "Set": "OrderService" }
        ]
      },
      "products-route": {
        "ClusterId": "products-cluster",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        }
      },
      "users-route": {
        "ClusterId": "users-cluster",
        "Match": {
          "Path": "/api/users/{**catch-all}"
        },
        "AuthorizationPolicy": "RequireAuthenticatedUser"
      }
    },
    "Clusters": {
      "orders-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "orders-1": { "Address": "http://order-service:8080" },
          "orders-2": { "Address": "http://order-service-2:8080" }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Path": "/health"
          }
        }
      },
      "products-cluster": {
        "Destinations": {
          "products-1": { "Address": "http://product-service:8080" }
        }
      },
      "users-cluster": {
        "Destinations": {
          "users-1": { "Address": "http://user-service:8080" }
        }
      }
    }
  }
}
```

---

## 5. Service Discovery

### ด้วย Consul

```csharp
// Register service กับ Consul
public class ConsulServiceRegistration : IHostedService
{
    private readonly IConsulClient _consul;
    private readonly IConfiguration _config;
    private string? _registrationId;

    public ConsulServiceRegistration(IConsulClient consul, IConfiguration config)
    {
        _consul = consul;
        _config = config;
    }

    public async Task StartAsync(CancellationToken ct)
    {
        var serviceId = $"{_config["ServiceName"]}-{Environment.MachineName}";
        _registrationId = serviceId;

        var registration = new AgentServiceRegistration
        {
            ID = serviceId,
            Name = _config["ServiceName"],
            Address = _config["ServiceHost"],
            Port = int.Parse(_config["ServicePort"]!),
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{_config["ServiceHost"]}:{_config["ServicePort"]}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(5),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            },
            Tags = new[] { "api", "v1" }
        };

        await _consul.Agent.ServiceRegister(registration, ct);
    }

    public async Task StopAsync(CancellationToken ct)
    {
        if (_registrationId != null)
            await _consul.Agent.ServiceDeregister(_registrationId, ct);
    }
}

// Discover service
public class ConsulServiceDiscovery : IServiceDiscovery
{
    private readonly IConsulClient _consul;

    public ConsulServiceDiscovery(IConsulClient consul)
    {
        _consul = consul;
    }

    public async Task<string?> GetServiceAddressAsync(string serviceName)
    {
        var services = await _consul.Health.Service(serviceName, tag: null, passingOnly: true);
        
        if (!services.Response.Any())
            return null;

        // Simple random load balancing
        var service = services.Response[Random.Shared.Next(services.Response.Length)];
        return $"http://{service.Service.Address}:{service.Service.Port}";
    }
}
```

### ด้วย Kubernetes DNS

```yaml
# K8s Service จะสร้าง DNS ให้อัตโนมัติ
# order-service.default.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: default
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
```

```csharp
// ใน K8s environment ใช้ service name โดยตรง
builder.Services.AddHttpClient<ProductServiceClient>(client =>
{
    // K8s DNS resolves 'product-service' ให้อัตโนมัติ
    client.BaseAddress = new Uri("http://product-service");
});
```

---

## 6. โปรแกรมตัวอย่าง: Microservices Design

### โครงสร้างโปรเจกต์

```
ecommerce/
├── src/
│   ├── ApiGateway/
│   │   ├── ApiGateway.csproj
│   │   └── Program.cs
│   ├── OrderService/
│   │   ├── OrderService.API/
│   │   ├── OrderService.Application/
│   │   ├── OrderService.Domain/
│   │   └── OrderService.Infrastructure/
│   ├── ProductService/
│   │   ├── ProductService.API/
│   │   ├── ProductService.Application/
│   │   ├── ProductService.Domain/
│   │   └── ProductService.Infrastructure/
│   └── UserService/
│       ├── UserService.API/
│       └── UserService.Infrastructure/
├── shared/
│   └── SharedContracts/      # Shared events/DTOs
└── docker-compose.yml
```

### SharedContracts - Events ที่แชร์ระหว่าง services

```csharp
// SharedContracts/Events/OrderEvents.cs
namespace SharedContracts.Events;

public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public List<OrderItemContract> Items { get; init; } = new();
    public decimal TotalAmount { get; init; }
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record OrderCancelledEvent
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public string Reason { get; init; } = string.Empty;
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

public record OrderItemContract
{
    public Guid ProductId { get; init; }
    public string ProductName { get; init; } = string.Empty;
    public decimal UnitPrice { get; init; }
    public int Quantity { get; init; }
}
```

### OrderService Domain

```csharp
// OrderService.Domain/Entities/Order.cs
namespace OrderService.Domain.Entities;

public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = new();
    private readonly List<IDomainEvent> _domainEvents = new();

    public Guid CustomerId { get; private set; }
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }
    public Address? ShippingAddress { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private Order() { Total = Money.Zero; }

    public static Order Create(Guid customerId, Address shippingAddress)
    {
        ArgumentNullException.ThrowIfNull(shippingAddress);
        
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Draft,
            ShippingAddress = shippingAddress,
            CreatedAt = DateTime.UtcNow
        };

        return order;
    }

    public Result AddItem(Guid productId, string productName, Money price, int quantity)
    {
        if (Status != OrderStatus.Draft)
            return Result.Failure("Cannot add items to non-draft order");

        if (quantity <= 0)
            return Result.Failure("Quantity must be positive");

        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
        {
            existingItem.UpdateQuantity(existingItem.Quantity + quantity);
        }
        else
        {
            _items.Add(new OrderItem(Guid.NewGuid(), Id, productId, productName, price, quantity));
        }

        RecalculateTotal();
        UpdatedAt = DateTime.UtcNow;
        return Result.Success();
    }

    public Result Submit()
    {
        if (Status != OrderStatus.Draft)
            return Result.Failure("Only draft orders can be submitted");

        if (!_items.Any())
            return Result.Failure("Cannot submit empty order");

        Status = OrderStatus.Submitted;
        UpdatedAt = DateTime.UtcNow;

        AddDomainEvent(new OrderSubmittedDomainEvent(Id, CustomerId, _items.ToList(), Total));
        return Result.Success();
    }

    public Result Cancel(string reason)
    {
        if (Status == OrderStatus.Delivered || Status == OrderStatus.Cancelled)
            return Result.Failure($"Cannot cancel order in {Status} status");

        Status = OrderStatus.Cancelled;
        UpdatedAt = DateTime.UtcNow;
        AddDomainEvent(new OrderCancelledDomainEvent(Id, CustomerId, reason));
        return Result.Success();
    }

    private void RecalculateTotal()
    {
        Total = new Money(_items.Sum(i => i.Price.Amount * i.Quantity));
    }

    private void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }

    public IReadOnlyList<IDomainEvent> GetDomainEvents() => _domainEvents.AsReadOnly();
    public void ClearDomainEvents() => _domainEvents.Clear();
}

public record Money(decimal Amount)
{
    public static Money Zero => new(0);
    public static Money operator +(Money a, Money b) => new(a.Amount + b.Amount);
    public static Money operator *(Money a, int b) => new(a.Amount * b);
}

public record Address(
    string Street,
    string City,
    string State,
    string ZipCode,
    string Country);
```

### OrderService API

```csharp
// OrderService.API/Endpoints/OrderEndpoints.cs
using Microsoft.AspNetCore.Mvc;

namespace OrderService.API.Endpoints;

public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .RequireAuthorization()
            .WithTags("Orders");

        group.MapPost("/", CreateOrder).WithName("CreateOrder");
        group.MapGet("/{id:guid}", GetOrder).WithName("GetOrder");
        group.MapGet("/customer/{customerId:guid}", GetCustomerOrders).WithName("GetCustomerOrders");
        group.MapPost("/{id:guid}/submit", SubmitOrder).WithName("SubmitOrder");
        group.MapPost("/{id:guid}/cancel", CancelOrder).WithName("CancelOrder");
    }

    private static async Task<IResult> CreateOrder(
        CreateOrderRequest request,
        IOrderService orderService,
        HttpContext context)
    {
        var customerId = context.GetUserId();
        var result = await orderService.CreateOrderAsync(customerId, request);
        
        return result.IsSuccess
            ? Results.CreatedAtRoute("GetOrder", new { id = result.Value!.Id }, result.Value)
            : Results.BadRequest(result.Errors);
    }

    private static async Task<IResult> GetOrder(
        Guid id,
        IOrderService orderService,
        HttpContext context)
    {
        var customerId = context.GetUserId();
        var order = await orderService.GetOrderAsync(id, customerId);
        return order is null ? Results.NotFound() : Results.Ok(order);
    }

    private static async Task<IResult> GetCustomerOrders(
        Guid customerId,
        [FromQuery] int page,
        [FromQuery] int pageSize,
        IOrderService orderService,
        HttpContext context)
    {
        // Verify customer can only see their own orders
        if (customerId != context.GetUserId())
            return Results.Forbid();

        var orders = await orderService.GetCustomerOrdersAsync(customerId, page, pageSize);
        return Results.Ok(orders);
    }

    private static async Task<IResult> SubmitOrder(
        Guid id,
        IOrderService orderService,
        HttpContext context)
    {
        var customerId = context.GetUserId();
        var result = await orderService.SubmitOrderAsync(id, customerId);
        
        return result.IsSuccess ? Results.Ok() : Results.BadRequest(result.Errors);
    }

    private static async Task<IResult> CancelOrder(
        Guid id,
        CancelOrderRequest request,
        IOrderService orderService,
        HttpContext context)
    {
        var customerId = context.GetUserId();
        var result = await orderService.CancelOrderAsync(id, customerId, request.Reason);
        
        return result.IsSuccess ? Results.Ok() : Results.BadRequest(result.Errors);
    }
}

public record CreateOrderRequest(
    List<OrderItemRequest> Items,
    AddressRequest ShippingAddress);

public record OrderItemRequest(
    Guid ProductId,
    string ProductName,
    decimal Price,
    int Quantity);

public record AddressRequest(
    string Street,
    string City,
    string State,
    string ZipCode,
    string Country);

public record CancelOrderRequest(string Reason);
```

### Program.cs สำหรับ OrderService

```csharp
// OrderService.API/Program.cs
using OrderService.API.Endpoints;
using OrderService.Infrastructure;
using OrderService.Application;
using MassTransit;

var builder = WebApplication.CreateBuilder(args);

// Application Services
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);

// MassTransit (RabbitMQ)
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<PaymentCompletedConsumer>();
    x.AddConsumer<InventoryReservedConsumer>();

    x.UsingRabbitMq((context, config) =>
    {
        config.Host(builder.Configuration["RabbitMQ:Host"], h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"]!);
            h.Password(builder.Configuration["RabbitMQ:Password"]!);
        });

        config.ConfigureEndpoints(context);
    });
});

// API
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();

// Health checks
builder.Services.AddHealthChecks()
    .AddNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRabbitMQ(builder.Configuration["RabbitMQ:Host"]!);

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthentication();
app.UseAuthorization();

app.MapOrderEndpoints();
app.MapHealthChecks("/health");

app.Run();
```

---

## 7. Resilience Patterns

### Circuit Breaker และ Retry

```csharp
// การใช้ Microsoft.Extensions.Http.Resilience (Polly)
builder.Services.AddHttpClient<IProductServiceClient, ProductServiceClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:Product"]!);
})
.AddResilienceHandler("product-service", pipeline =>
{
    // Retry policy
    pipeline.AddRetry(new HttpRetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(300),
        BackoffType = DelayBackoffType.Exponential,
        ShouldHandle = args => args.Outcome switch
        {
            { Result: { IsSuccessStatusCode: false } } => PredicateResult.True(),
            { Exception: HttpRequestException } => PredicateResult.True(),
            _ => PredicateResult.False()
        }
    });

    // Circuit breaker
    pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
    {
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 5,
        FailureRatio = 0.5,
        BreakDuration = TimeSpan.FromSeconds(30),
        OnOpened = args =>
        {
            Console.WriteLine("Circuit breaker opened!");
            return ValueTask.CompletedTask;
        }
    });

    // Timeout
    pipeline.AddTimeout(TimeSpan.FromSeconds(5));
});
```

### Fallback Pattern

```csharp
public class ProductServiceClientWithFallback : IProductServiceClient
{
    private readonly ProductServiceClient _inner;
    private readonly IProductCacheRepository _cache;
    private readonly ILogger<ProductServiceClientWithFallback> _logger;

    public ProductServiceClientWithFallback(
        ProductServiceClient inner,
        IProductCacheRepository cache,
        ILogger<ProductServiceClientWithFallback> logger)
    {
        _inner = inner;
        _cache = cache;
        _logger = logger;
    }

    public async Task<ProductDto?> GetProductAsync(Guid productId)
    {
        try
        {
            var product = await _inner.GetProductAsync(productId);
            // Update cache on successful call
            if (product != null)
                await _cache.SetAsync(productId, product);
            return product;
        }
        catch (Exception ex) when (ex is HttpRequestException or BrokenCircuitException)
        {
            _logger.LogWarning("Product service unavailable, falling back to cache");
            return await _cache.GetAsync(productId);
        }
    }
}
```

---

## Exercises / Project Tasks

### Exercise 1: Service Decomposition
แบ่ง Monolith application เป็น Microservices:
```
Monolith: BlogApp (Posts, Comments, Users, Tags, Files)
ภาระกิจ: แบ่งเป็น services ที่เหมาะสม กำหนด boundaries และ events
```

### Exercise 2: API Gateway
สร้าง API Gateway ด้วย YARP:
- Route requests ไป services ต่างๆ
- Authentication/Authorization
- Rate limiting
- Request logging

### Exercise 3: Event-Driven Communication
ออกแบบ event flow สำหรับ E-commerce:
```
User places order → 
  OrderService creates order →
    InventoryService reserves stock →
      PaymentService processes payment →
        NotificationService sends email
```

### Exercise 4: Resilience
เพิ่ม resilience ให้ service-to-service communication:
- Retry with exponential backoff
- Circuit breaker
- Timeout
- Fallback to cache

---

## สรุป

- **Monolith** เหมาะกับโปรเจกต์เล็ก/เริ่มต้น แต่มีปัญหาด้าน scale
- **Microservices** เหมาะกับระบบขนาดใหญ่ที่ต้องการ independent deployment
- **Service Boundaries** ควรสอดคล้องกับ Business Domain (Bounded Context)
- **Synchronous** communication ใช้เมื่อต้องการผลทันที แต่เพิ่ม coupling
- **Asynchronous** communication ผ่าน message queue ให้ loose coupling และ resilience ดีกว่า
- **API Gateway** เป็น single entry point จัดการ cross-cutting concerns
- **Service Discovery** ช่วยให้ services หากันเจอใน dynamic environment
- **Resilience patterns** (retry, circuit breaker) จำเป็นมากใน distributed systems

---

## Part ถัดไป

**Part 082: Message Queue กับ RabbitMQ** - เรียนรู้การใช้ RabbitMQ และ MassTransit สำหรับ asynchronous communication ระหว่าง microservices

---

*Part 081/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
