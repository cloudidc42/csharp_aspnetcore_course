# Part 71: Clean Architecture

## เนื้อหาใน Part นี้
- ทำไมต้องใช้ Clean Architecture
- Layers: Domain, Application, Infrastructure, Presentation
- Dependency Rule
- โครงสร้าง Solution
- Domain Entities
- โปรแกรมตัวอย่าง: E-commerce clean architecture structure

---

## ทำไมต้องใช้ Clean Architecture?

Clean Architecture คือรูปแบบการออกแบบซอฟต์แวร์ที่เน้นการแยก concerns ออกจากกัน เพื่อให้ระบบมีความยืดหยุ่น ทดสอบได้ง่าย และบำรุงรักษาได้ในระยะยาว

### ปัญหาของ Traditional Architecture

เมื่อพัฒนาแอปพลิเคชันโดยไม่มีโครงสร้างที่ดี มักเกิดปัญหาเหล่านี้:

```csharp
// โค้ดแบบ "สปาเก็ตตี้" ที่ไม่มีโครงสร้าง
public class OrderController : Controller
{
    private readonly SqlConnection _connection;
    
    public OrderController()
    {
        // Hard-coded connection string - ทดสอบยาก!
        _connection = new SqlConnection("Server=prod;Database=shop;...");
    }
    
    [HttpPost]
    public IActionResult CreateOrder(OrderRequest request)
    {
        // Business logic ปนกับ Data access - บำรุงรักษายาก!
        if (request.Items.Count == 0)
            return BadRequest("ไม่มีสินค้า");
            
        decimal total = 0;
        foreach (var item in request.Items)
        {
            // ส่ง email และ SMS โดยตรงใน Controller - ทดสอบยากมาก!
            total += item.Price * item.Quantity;
        }
        
        // SQL ปะปนอยู่ใน Controller
        var cmd = new SqlCommand(
            "INSERT INTO Orders VALUES (@total)", _connection);
        cmd.ExecuteNonQuery();
        
        // ส่ง email โดยตรง
        new SmtpClient("mail.server.com").Send(
            new MailMessage("shop@store.com", request.Email,
                "Order Confirmed", $"Total: {total}"));
        
        return Ok();
    }
}
```

ปัญหาที่เห็นได้ชัด:
1. **ยึดกับ SQL Server** - เปลี่ยนฐานข้อมูลไม่ได้
2. **ยึดกับ SMTP** - เปลี่ยนระบบส่ง email ไม่ได้
3. **ทดสอบไม่ได้** - ต้องมี database และ email server จริงๆ
4. **แก้ไขยาก** - เปลี่ยนส่วนหนึ่งกระทบทุกส่วน

### ประโยชน์ของ Clean Architecture

1. **Independent of Frameworks** - ไม่ผูกติดกับ library หรือ framework ใดๆ
2. **Testable** - Business rules ทดสอบได้โดยไม่ต้องมี UI, Database หรือ Web Server
3. **Independent of UI** - UI เปลี่ยนได้โดยไม่กระทบ business rules
4. **Independent of Database** - เปลี่ยนฐานข้อมูลได้โดยไม่กระทบ business rules
5. **Independent of External Agencies** - Business rules ไม่รู้จักโลกภายนอก

---

## Layers ใน Clean Architecture

Clean Architecture ประกอบด้วย 4 ชั้นหลัก เรียงจากในสุดออกมา:

```
┌─────────────────────────────────────────┐
│           Presentation Layer            │
│    (Controllers, Views, API Endpoints)  │
├─────────────────────────────────────────┤
│          Infrastructure Layer           │
│   (Database, Email, File System, etc.)  │
├─────────────────────────────────────────┤
│           Application Layer             │
│    (Use Cases, Commands, Queries)       │
├─────────────────────────────────────────┤
│             Domain Layer                │
│    (Entities, Value Objects, Rules)     │
└─────────────────────────────────────────┘
```

### 1. Domain Layer (ชั้นในสุด)

Domain Layer คือหัวใจของระบบ ประกอบด้วย:
- **Entities** - วัตถุหลักของธุรกิจ
- **Value Objects** - วัตถุที่กำหนดโดยค่า
- **Domain Events** - เหตุการณ์ที่เกิดขึ้นในโดเมน
- **Repository Interfaces** - อินเตอร์เฟซสำหรับ data access
- **Domain Services** - ตรรกะทางธุรกิจที่ไม่เข้ากับ entity ใดๆ

```csharp
// Domain/Entities/Order.cs
namespace ECommerce.Domain.Entities;

public class Order
{
    public Guid Id { get; private set; }
    public string CustomerId { get; private set; }
    public DateTime OrderDate { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    // Domain Events
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    protected Order() { } // สำหรับ EF Core
    
    public static Order Create(string customerId)
    {
        if (string.IsNullOrEmpty(customerId))
            throw new DomainException("CustomerId ต้องไม่ว่าง");
            
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            OrderDate = DateTime.UtcNow,
            Status = OrderStatus.Pending
        };
        
        order.AddDomainEvent(new OrderCreatedEvent(order.Id, customerId));
        return order;
    }
    
    public void AddItem(string productId, string productName, decimal price, int quantity)
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("ไม่สามารถเพิ่มสินค้าในคำสั่งซื้อที่ไม่ใช่สถานะ Pending");
            
        if (quantity <= 0)
            throw new DomainException("จำนวนสินค้าต้องมากกว่า 0");
            
        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem != null)
        {
            existingItem.UpdateQuantity(existingItem.Quantity + quantity);
        }
        else
        {
            _items.Add(OrderItem.Create(Id, productId, productName, price, quantity));
        }
    }
    
    public void Confirm()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("สามารถยืนยันได้เฉพาะคำสั่งซื้อสถานะ Pending");
            
        if (!_items.Any())
            throw new DomainException("ไม่สามารถยืนยันคำสั่งซื้อที่ไม่มีสินค้า");
            
        Status = OrderStatus.Confirmed;
        AddDomainEvent(new OrderConfirmedEvent(Id));
    }
    
    public decimal GetTotalAmount() => _items.Sum(i => i.GetSubtotal());
    
    private void AddDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);
        
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

### 2. Application Layer

Application Layer ประกอบด้วย:
- **Use Cases / Application Services** - logic ของแอปพลิเคชัน
- **DTOs** - วัตถุสำหรับส่งข้อมูลระหว่างชั้น
- **Interfaces** - อินเตอร์เฟซที่ Infrastructure จะ implement

```csharp
// Application/Interfaces/IOrderRepository.cs
namespace ECommerce.Application.Interfaces;

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IEnumerable<Order>> GetByCustomerIdAsync(string customerId, CancellationToken cancellationToken = default);
    Task AddAsync(Order order, CancellationToken cancellationToken = default);
    Task UpdateAsync(Order order, CancellationToken cancellationToken = default);
}

// Application/Interfaces/IEmailService.cs
public interface IEmailService
{
    Task SendOrderConfirmationAsync(string email, Order order, CancellationToken cancellationToken = default);
}
```

### 3. Infrastructure Layer

Infrastructure Layer implement interfaces จาก Application Layer:

```csharp
// Infrastructure/Persistence/OrderRepository.cs
namespace ECommerce.Infrastructure.Persistence;

public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    
    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }
    
    public async Task<IEnumerable<Order>> GetByCustomerIdAsync(
        string customerId, CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .Where(o => o.CustomerId == customerId)
            .ToListAsync(cancellationToken);
    }
    
    public async Task AddAsync(Order order, CancellationToken cancellationToken = default)
    {
        await _context.Orders.AddAsync(order, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);
    }
    
    public async Task UpdateAsync(Order order, CancellationToken cancellationToken = default)
    {
        _context.Orders.Update(order);
        await _context.SaveChangesAsync(cancellationToken);
    }
}
```

### 4. Presentation Layer

Presentation Layer คือส่วนที่ user โต้ตอบด้วย:

```csharp
// Presentation/Controllers/OrdersController.cs
namespace ECommerce.Presentation.Controllers;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder(
        [FromBody] CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetOrder), new { id = result.OrderId }, result);
    }
    
    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new GetOrderQuery(id), cancellationToken);
        return result is null ? NotFound() : Ok(result);
    }
}
```

---

## Dependency Rule

กฎสำคัญที่สุดของ Clean Architecture:

> **Source code dependencies ต้องชี้เข้าด้านใน เท่านั้น**

หมายความว่า:
- Domain layer ไม่รู้จัก Application, Infrastructure หรือ Presentation
- Application layer รู้จัก Domain แต่ไม่รู้จัก Infrastructure หรือ Presentation
- Infrastructure layer รู้จัก Application และ Domain
- Presentation layer รู้จัก Application (ผ่าน interfaces)

```
Presentation → Application → Domain
Infrastructure → Application → Domain
```

### การใช้ Dependency Inversion

```csharp
// ❌ ผิด: Application depend on Infrastructure
namespace ECommerce.Application.UseCases;
public class CreateOrderUseCase
{
    // ผิด! Application ไม่ควร depend on Infrastructure
    private readonly SqlOrderRepository _repository;
}

// ✅ ถูก: Application depend on Interface
namespace ECommerce.Application.UseCases;
public class CreateOrderUseCase
{
    // ถูก! Depend on Abstraction (Interface)
    private readonly IOrderRepository _repository;
    
    public CreateOrderUseCase(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

---

## โครงสร้าง Solution

```
ECommerce/
├── src/
│   ├── ECommerce.Domain/
│   │   ├── Entities/
│   │   │   ├── Order.cs
│   │   │   ├── OrderItem.cs
│   │   │   ├── Product.cs
│   │   │   └── Customer.cs
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   ├── Address.cs
│   │   │   └── Email.cs
│   │   ├── Events/
│   │   │   ├── OrderCreatedEvent.cs
│   │   │   └── OrderConfirmedEvent.cs
│   │   ├── Exceptions/
│   │   │   └── DomainException.cs
│   │   └── Enums/
│   │       └── OrderStatus.cs
│   │
│   ├── ECommerce.Application/
│   │   ├── Interfaces/
│   │   │   ├── IOrderRepository.cs
│   │   │   ├── IProductRepository.cs
│   │   │   └── IEmailService.cs
│   │   ├── Orders/
│   │   │   ├── Commands/
│   │   │   │   ├── CreateOrderCommand.cs
│   │   │   │   └── ConfirmOrderCommand.cs
│   │   │   └── Queries/
│   │   │       ├── GetOrderQuery.cs
│   │   │       └── GetOrdersByCustomerQuery.cs
│   │   └── Common/
│   │       ├── Mappings/
│   │       └── Behaviors/
│   │
│   ├── ECommerce.Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── AppDbContext.cs
│   │   │   ├── OrderRepository.cs
│   │   │   └── ProductRepository.cs
│   │   ├── Services/
│   │   │   └── EmailService.cs
│   │   └── DependencyInjection.cs
│   │
│   └── ECommerce.Presentation/
│       ├── Controllers/
│       │   ├── OrdersController.cs
│       │   └── ProductsController.cs
│       ├── Program.cs
│       └── appsettings.json
│
└── tests/
    ├── ECommerce.Domain.Tests/
    ├── ECommerce.Application.Tests/
    └── ECommerce.Integration.Tests/
```

---

## โปรแกรมตัวอย่าง: E-Commerce Clean Architecture

ต่อไปนี้คือตัวอย่างการ implement Clean Architecture แบบสมบูรณ์สำหรับระบบ E-commerce

### Domain Layer

```csharp
// Domain/Enums/OrderStatus.cs
namespace ECommerce.Domain.Enums;

public enum OrderStatus
{
    Pending = 1,
    Confirmed = 2,
    Processing = 3,
    Shipped = 4,
    Delivered = 5,
    Cancelled = 6
}

// Domain/Exceptions/DomainException.cs
namespace ECommerce.Domain.Exceptions;

public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
    public DomainException(string message, Exception innerException) 
        : base(message, innerException) { }
}

// Domain/Events/IDomainEvent.cs
namespace ECommerce.Domain.Events;

public interface IDomainEvent
{
    DateTime OccurredOn { get; }
}

// Domain/Events/OrderCreatedEvent.cs
namespace ECommerce.Domain.Events;

public record OrderCreatedEvent(Guid OrderId, string CustomerId) : IDomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

public record OrderConfirmedEvent(Guid OrderId) : IDomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

// Domain/ValueObjects/Money.cs
namespace ECommerce.Domain.ValueObjects;

public record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);
    
    public static Money operator +(Money left, Money right)
    {
        if (left.Currency != right.Currency)
            throw new DomainException($"ไม่สามารถบวกสกุลเงินต่างกันได้: {left.Currency} และ {right.Currency}");
        return new Money(left.Amount + right.Amount, left.Currency);
    }
    
    public static Money operator *(Money money, int quantity)
        => new(money.Amount * quantity, money.Currency);
        
    public override string ToString() => $"{Amount:N2} {Currency}";
}

// Domain/Entities/OrderItem.cs
namespace ECommerce.Domain.Entities;

public class OrderItem
{
    public Guid Id { get; private set; }
    public Guid OrderId { get; private set; }
    public string ProductId { get; private set; }
    public string ProductName { get; private set; }
    public decimal UnitPrice { get; private set; }
    public int Quantity { get; private set; }
    
    protected OrderItem() { }
    
    public static OrderItem Create(
        Guid orderId, string productId, string productName, 
        decimal unitPrice, int quantity)
    {
        if (unitPrice <= 0)
            throw new DomainException("ราคาสินค้าต้องมากกว่า 0");
            
        return new OrderItem
        {
            Id = Guid.NewGuid(),
            OrderId = orderId,
            ProductId = productId,
            ProductName = productName,
            UnitPrice = unitPrice,
            Quantity = quantity
        };
    }
    
    public void UpdateQuantity(int newQuantity)
    {
        if (newQuantity <= 0)
            throw new DomainException("จำนวนสินค้าต้องมากกว่า 0");
        Quantity = newQuantity;
    }
    
    public decimal GetSubtotal() => UnitPrice * Quantity;
}

// Domain/Entities/Product.cs
namespace ECommerce.Domain.Entities;

public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public string Description { get; private set; }
    public decimal Price { get; private set; }
    public int StockQuantity { get; private set; }
    public bool IsActive { get; private set; }
    
    protected Product() { }
    
    public static Product Create(
        string name, string description, decimal price, int stockQuantity)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new DomainException("ชื่อสินค้าต้องไม่ว่าง");
            
        if (price <= 0)
            throw new DomainException("ราคาสินค้าต้องมากกว่า 0");
            
        return new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Description = description,
            Price = price,
            StockQuantity = stockQuantity,
            IsActive = true
        };
    }
    
    public void ReduceStock(int quantity)
    {
        if (quantity > StockQuantity)
            throw new DomainException($"สินค้าคงคลังไม่เพียงพอ มีเพียง {StockQuantity} ชิ้น");
        StockQuantity -= quantity;
    }
    
    public bool IsInStock(int requestedQuantity) => StockQuantity >= requestedQuantity;
}
```

### Application Layer

```csharp
// Application/Interfaces/IOrderRepository.cs
namespace ECommerce.Application.Interfaces;

public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IEnumerable<Order>> GetByCustomerIdAsync(
        string customerId, CancellationToken cancellationToken = default);
    Task AddAsync(Order order, CancellationToken cancellationToken = default);
    Task UpdateAsync(Order order, CancellationToken cancellationToken = default);
}

public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IEnumerable<Product>> GetAllAsync(CancellationToken cancellationToken = default);
    Task UpdateAsync(Product product, CancellationToken cancellationToken = default);
}

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

// Application/Orders/Commands/CreateOrderCommand.cs
namespace ECommerce.Application.Orders.Commands;

public record CreateOrderCommand(
    string CustomerId,
    List<CreateOrderItemDto> Items) : IRequest<CreateOrderResult>;

public record CreateOrderItemDto(
    Guid ProductId,
    int Quantity);

public record CreateOrderResult(
    Guid OrderId,
    decimal TotalAmount,
    string Status);

// Application/Orders/Commands/CreateOrderCommandHandler.cs
public class CreateOrderCommandHandler 
    : IRequestHandler<CreateOrderCommand, CreateOrderResult>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IProductRepository _productRepository;
    private readonly IUnitOfWork _unitOfWork;
    
    public CreateOrderCommandHandler(
        IOrderRepository orderRepository,
        IProductRepository productRepository,
        IUnitOfWork unitOfWork)
    {
        _orderRepository = orderRepository;
        _productRepository = productRepository;
        _unitOfWork = unitOfWork;
    }
    
    public async Task<CreateOrderResult> Handle(
        CreateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = Order.Create(request.CustomerId);
        
        foreach (var item in request.Items)
        {
            var product = await _productRepository.GetByIdAsync(
                item.ProductId, cancellationToken);
                
            if (product is null)
                throw new NotFoundException($"ไม่พบสินค้า ID: {item.ProductId}");
                
            if (!product.IsInStock(item.Quantity))
                throw new BusinessRuleException($"สินค้า {product.Name} มีไม่เพียงพอ");
                
            order.AddItem(
                product.Id.ToString(),
                product.Name,
                product.Price,
                item.Quantity);
                
            product.ReduceStock(item.Quantity);
            await _productRepository.UpdateAsync(product, cancellationToken);
        }
        
        await _orderRepository.AddAsync(order, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);
        
        return new CreateOrderResult(
            order.Id,
            order.GetTotalAmount(),
            order.Status.ToString());
    }
}

// Application/Orders/Queries/GetOrderQuery.cs
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;

public record OrderDto(
    Guid Id,
    string CustomerId,
    DateTime OrderDate,
    string Status,
    decimal TotalAmount,
    List<OrderItemDto> Items);

public record OrderItemDto(
    Guid Id,
    string ProductId,
    string ProductName,
    decimal UnitPrice,
    int Quantity,
    decimal Subtotal);

public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    private readonly IOrderRepository _orderRepository;
    
    public GetOrderQueryHandler(IOrderRepository orderRepository)
    {
        _orderRepository = orderRepository;
    }
    
    public async Task<OrderDto?> Handle(
        GetOrderQuery request, CancellationToken cancellationToken)
    {
        var order = await _orderRepository.GetByIdAsync(
            request.OrderId, cancellationToken);
            
        if (order is null) return null;
        
        return new OrderDto(
            order.Id,
            order.CustomerId,
            order.OrderDate,
            order.Status.ToString(),
            order.GetTotalAmount(),
            order.Items.Select(i => new OrderItemDto(
                i.Id,
                i.ProductId,
                i.ProductName,
                i.UnitPrice,
                i.Quantity,
                i.GetSubtotal())).ToList());
    }
}
```

### Infrastructure Layer

```csharp
// Infrastructure/Persistence/AppDbContext.cs
namespace ECommerce.Infrastructure.Persistence;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<Product> Products => Set<Product>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(modelBuilder);
    }
}

// Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.CustomerId).IsRequired().HasMaxLength(100);
        builder.Property(o => o.Status).HasConversion<string>();
        builder.HasMany(o => o.Items)
               .WithOne()
               .HasForeignKey(i => i.OrderId)
               .OnDelete(DeleteBehavior.Cascade);
        builder.Ignore(o => o.DomainEvents);
    }
}

// Infrastructure/DependencyInjection.cs
namespace ECommerce.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services, 
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)));
                
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddTransient<IEmailService, EmailService>();
        
        return services;
    }
}
```

### Presentation Layer

```csharp
// Presentation/Program.cs
using ECommerce.Application;
using ECommerce.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// เพิ่ม Application layer
builder.Services.AddApplication();

// เพิ่ม Infrastructure layer
builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();

// Application/DependencyInjection.cs
namespace ECommerce.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg => 
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
            
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        
        // เพิ่ม Pipeline behaviors
        services.AddTransient(typeof(IPipelineBehavior<,>), 
            typeof(ValidationBehavior<,>));
        services.AddTransient(typeof(IPipelineBehavior<,>), 
            typeof(LoggingBehavior<,>));
            
        return services;
    }
}
```

### ตัวอย่างการรัน (Program สมบูรณ์)

```csharp
// ConsoleApp สำหรับทดสอบ Clean Architecture
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        // สำหรับ demo ใช้ In-Memory
        services.AddDbContext<AppDbContext>(opt => 
            opt.UseInMemoryDatabase("ECommerceDb"));
            
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddMediatR(cfg => 
            cfg.RegisterServicesFromAssembly(typeof(CreateOrderCommand).Assembly));
    })
    .Build();

using var scope = host.Services.CreateScope();
var mediator = scope.ServiceProvider.GetRequiredService<IMediator>();

// สร้าง Product ก่อน (สำหรับ demo)
var productRepo = scope.ServiceProvider.GetRequiredService<IProductRepository>();
var product1 = Product.Create("iPhone 15 Pro", "สมาร์ทโฟน Apple", 45000m, 10);
var product2 = Product.Create("AirPods Pro", "หูฟังไร้สาย", 9000m, 20);

// สั่งซื้อ
var command = new CreateOrderCommand(
    CustomerId: "customer-001",
    Items: new List<CreateOrderItemDto>
    {
        new(product1.Id, 1),
        new(product2.Id, 2)
    });

var result = await mediator.Send(command);

Console.WriteLine($"สร้างคำสั่งซื้อสำเร็จ:");
Console.WriteLine($"  Order ID: {result.OrderId}");
Console.WriteLine($"  ยอดรวม: {result.TotalAmount:N2} บาท");
Console.WriteLine($"  สถานะ: {result.Status}");

// ดึงข้อมูลคำสั่งซื้อ
var order = await mediator.Send(new GetOrderQuery(result.OrderId));
Console.WriteLine($"\nรายละเอียดคำสั่งซื้อ:");
Console.WriteLine($"  ลูกค้า: {order.CustomerId}");
Console.WriteLine($"  วันที่: {order.OrderDate:dd/MM/yyyy}");
foreach (var item in order.Items)
{
    Console.WriteLine($"  - {item.ProductName}: {item.Quantity} x {item.UnitPrice:N2} = {item.Subtotal:N2}");
}
```

---

## Exercises

1. **Exercise 1**: เพิ่ม `Customer` entity ใน Domain layer พร้อม validation rules
   - Customer ต้องมี Email ที่ถูกต้อง
   - Customer ต้องมี Name ที่มีความยาว 2-100 ตัวอักษร

2. **Exercise 2**: สร้าง `CancelOrderCommand` สำหรับยกเลิกคำสั่งซื้อ
   - ยกเลิกได้เฉพาะ Order สถานะ Pending หรือ Confirmed
   - ต้องเพิ่ม Stock กลับเมื่อยกเลิก

3. **Exercise 3**: เพิ่ม `ValidationBehavior` ใน MediatR pipeline
   - ใช้ FluentValidation
   - Validate ทุก Command ก่อนส่งถึง Handler

4. **Exercise 4**: สร้าง Repository pattern ด้วย Generic Repository
   - `IRepository<T>` base interface
   - Implement `EfRepository<T>` สำหรับ EF Core

5. **Exercise 5**: เพิ่ม Domain Event handling
   - เมื่อยืนยัน Order ให้ส่ง Email แจ้งลูกค้า
   - ใช้ Domain Events + Event Handlers

---

## สรุป

Clean Architecture ช่วยให้เราสร้างระบบที่:
- **แยกส่วนรับผิดชอบ** ชัดเจน แต่ละ layer มีหน้าที่ของตัวเอง
- **ทดสอบได้ง่าย** เพราะ Business logic ไม่ผูกกับ infrastructure
- **ยืดหยุ่น** เปลี่ยน database หรือ UI ได้โดยไม่กระทบ core logic
- **บำรุงรักษาได้** โค้ดอ่านเข้าใจง่าย และแก้ไขในที่เดียว

กฎสำคัญ: **Dependencies ต้องชี้เข้าสู่ Domain เท่านั้น**

---

## Part ถัดไป

ใน Part 72 เราจะเรียนรู้เรื่อง **CQRS Pattern** ซึ่งเป็น pattern ที่นิยมใช้ร่วมกับ Clean Architecture เพื่อแยก Read และ Write operations ออกจากกัน และใช้งาน MediatR library สำหรับ implementation

---

*Part 71/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
