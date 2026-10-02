# Part 72: CQRS Pattern

## เนื้อหาใน Part นี้
- Command Query Responsibility Segregation (CQRS)
- Commands vs Queries
- MediatR library
- Command handlers
- Query handlers
- Notifications/Events
- โปรแกรมตัวอย่าง: Order management with CQRS

---

## CQRS คืออะไร?

**CQRS** ย่อมาจาก **Command Query Responsibility Segregation** คือ pattern การออกแบบที่แยก การเขียนข้อมูล (Command) และการอ่านข้อมูล (Query) ออกจากกันอย่างชัดเจน

### แนวคิดหลัก

```
Traditional API:
┌─────────────────────────┐
│   Single Model / Service │
│  (Read + Write ปนกัน)   │
└─────────────────────────┘

CQRS:
┌─────────────┐    ┌─────────────┐
│  Write Side  │    │  Read Side  │
│  (Commands)  │    │  (Queries)  │
│  - Update DB │    │  - Read DB  │
│  - Validate  │    │  - Return   │
│  - Emit Events│   │    DTOs     │
└─────────────┘    └─────────────┘
```

### ทำไมต้องแยก Read กับ Write?

1. **Read/Write มีความต้องการต่างกัน**
   - Read: ต้องการความเร็ว, ข้อมูลหลายตาราง, aggregate data
   - Write: ต้องการ validation เข้มข้น, transaction, domain rules

2. **Scale แยกกันได้**
   - Read มักถูกเรียกบ่อยกว่า Write มาก
   - สามารถ scale read replicas แยกต่างหาก

3. **Optimize แยกกันได้**
   - Read: ใช้ read-optimized views, caching
   - Write: ใช้ normalized model, full validation

---

## Commands vs Queries

### Commands (การเขียน/เปลี่ยนแปลง)

Command คือ request ที่ต้องการ **เปลี่ยนแปลงสถานะ** ของระบบ

```csharp
// ✅ Commands - เปลี่ยนแปลงข้อมูล, ไม่ต้อง return ข้อมูลมาก
public record CreateOrderCommand(
    string CustomerId,
    List<OrderItemRequest> Items) : IRequest<Guid>;

public record UpdateOrderStatusCommand(
    Guid OrderId,
    OrderStatus NewStatus) : IRequest<bool>;

public record CancelOrderCommand(
    Guid OrderId,
    string CancellationReason) : IRequest;

// Command naming convention: [Action][Entity]Command
// CreateOrder, UpdateOrderStatus, CancelOrder, DeleteProduct
```

### Queries (การอ่าน)

Query คือ request ที่ต้องการ **ดึงข้อมูล** โดยไม่เปลี่ยนแปลงสถานะ

```csharp
// ✅ Queries - ดึงข้อมูล, return ข้อมูลที่ต้องการ
public record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderDetailDto?>;

public record GetOrdersByCustomerQuery(
    string CustomerId,
    int Page = 1,
    int PageSize = 10) : IRequest<PagedResult<OrderSummaryDto>>;

public record SearchOrdersQuery(
    string? SearchTerm,
    OrderStatus? Status,
    DateTime? FromDate,
    DateTime? ToDate,
    int Page = 1,
    int PageSize = 20) : IRequest<PagedResult<OrderSummaryDto>>;

// Query naming convention: Get[Entity]By[Criteria]Query
// GetOrderById, GetOrdersByCustomer, SearchProducts
```

### กฎสำคัญ

```csharp
// ❌ ผิด: Query ไม่ควรแก้ไขข้อมูล
public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await _db.Orders.FindAsync(request.OrderId);
        
        // ❌ ไม่ควรแก้ไขใน Query!
        order.LastViewedAt = DateTime.UtcNow;
        await _db.SaveChangesAsync();
        
        return MapToDto(order);
    }
}

// ✅ ถูก: Query อ่านอย่างเดียว
public class GetOrderQueryHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await _db.Orders
            .AsNoTracking() // ใช้ AsNoTracking สำหรับ query
            .FirstOrDefaultAsync(o => o.Id == request.OrderId, ct);
            
        return order is null ? null : MapToDto(order);
    }
}
```

---

## MediatR Library

MediatR เป็น library ยอดนิยมสำหรับ implement CQRS และ Mediator pattern ใน .NET

### ติดตั้ง

```bash
dotnet add package MediatR
# สำหรับ ASP.NET Core
dotnet add package MediatR.Extensions.Microsoft.DependencyInjection
```

### ลงทะเบียนใน DI Container

```csharp
// Program.cs หรือ ServiceCollectionExtensions.cs
builder.Services.AddMediatR(cfg => 
{
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    
    // เพิ่ม Pipeline Behaviors (middleware สำหรับ MediatR)
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(PerformanceBehavior<,>));
});
```

### IRequest และ IRequestHandler

```csharp
// สร้าง Command/Query ที่ implement IRequest<TResponse>
public record PlaceOrderCommand(
    string CustomerId,
    List<OrderLineItem> Items,
    string ShippingAddress) : IRequest<PlaceOrderResult>;

// สร้าง Handler ที่ implement IRequestHandler<TRequest, TResponse>
public class PlaceOrderCommandHandler 
    : IRequestHandler<PlaceOrderCommand, PlaceOrderResult>
{
    private readonly IOrderRepository _orderRepository;
    private readonly IInventoryService _inventoryService;
    private readonly IPublisher _publisher;
    
    public PlaceOrderCommandHandler(
        IOrderRepository orderRepository,
        IInventoryService inventoryService,
        IPublisher publisher)
    {
        _orderRepository = orderRepository;
        _inventoryService = inventoryService;
        _publisher = publisher;
    }
    
    public async Task<PlaceOrderResult> Handle(
        PlaceOrderCommand request, CancellationToken cancellationToken)
    {
        // 1. ตรวจสอบ inventory
        foreach (var item in request.Items)
        {
            var available = await _inventoryService
                .CheckAvailabilityAsync(item.ProductId, item.Quantity, cancellationToken);
                
            if (!available)
                throw new InsufficientStockException(item.ProductId);
        }
        
        // 2. สร้าง Order
        var order = Order.Create(request.CustomerId, request.ShippingAddress);
        foreach (var item in request.Items)
            order.AddItem(item.ProductId, item.ProductName, item.Price, item.Quantity);
            
        // 3. บันทึก
        await _orderRepository.AddAsync(order, cancellationToken);
        
        // 4. Publish notification
        await _publisher.Publish(
            new OrderPlacedNotification(order.Id, request.CustomerId), 
            cancellationToken);
        
        return new PlaceOrderResult(order.Id, order.GetTotalAmount());
    }
}
```

---

## Command Handlers

### Command Handler แบบสมบูรณ์

```csharp
// Commands/UpdateProductCommand.cs
public record UpdateProductCommand(
    Guid ProductId,
    string Name,
    string Description,
    decimal Price,
    int StockQuantity) : IRequest<UpdateProductResult>;

public record UpdateProductResult(bool Success, string Message);

// Commands/UpdateProductCommandHandler.cs
public class UpdateProductCommandHandler 
    : IRequestHandler<UpdateProductCommand, UpdateProductResult>
{
    private readonly IProductRepository _productRepository;
    private readonly ILogger<UpdateProductCommandHandler> _logger;
    
    public UpdateProductCommandHandler(
        IProductRepository productRepository,
        ILogger<UpdateProductCommandHandler> logger)
    {
        _productRepository = productRepository;
        _logger = logger;
    }
    
    public async Task<UpdateProductResult> Handle(
        UpdateProductCommand request, CancellationToken cancellationToken)
    {
        _logger.LogInformation(
            "กำลังอัพเดทสินค้า ID: {ProductId}", request.ProductId);
        
        var product = await _productRepository
            .GetByIdAsync(request.ProductId, cancellationToken);
            
        if (product is null)
        {
            _logger.LogWarning("ไม่พบสินค้า ID: {ProductId}", request.ProductId);
            return new UpdateProductResult(false, $"ไม่พบสินค้า ID: {request.ProductId}");
        }
        
        product.Update(request.Name, request.Description, request.Price);
        
        await _productRepository.UpdateAsync(product, cancellationToken);
        
        _logger.LogInformation("อัพเดทสินค้าสำเร็จ ID: {ProductId}", request.ProductId);
        return new UpdateProductResult(true, "อัพเดทสินค้าสำเร็จ");
    }
}

// Validation สำหรับ Command
public class UpdateProductCommandValidator : AbstractValidator<UpdateProductCommand>
{
    public UpdateProductCommandValidator()
    {
        RuleFor(x => x.ProductId).NotEmpty();
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200)
            .WithMessage("ชื่อสินค้าต้องมีความยาวไม่เกิน 200 ตัวอักษร");
        RuleFor(x => x.Price)
            .GreaterThan(0)
            .WithMessage("ราคาสินค้าต้องมากกว่า 0");
        RuleFor(x => x.StockQuantity)
            .GreaterThanOrEqualTo(0)
            .WithMessage("จำนวนสินค้าต้องไม่ติดลบ");
    }
}
```

### Command ที่ไม่ return ค่า

```csharp
// Command ที่ไม่ต้อง return ค่า
public record DeleteProductCommand(Guid ProductId) : IRequest;

public class DeleteProductCommandHandler : IRequestHandler<DeleteProductCommand>
{
    private readonly IProductRepository _productRepository;
    
    public DeleteProductCommandHandler(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }
    
    public async Task Handle(DeleteProductCommand request, CancellationToken cancellationToken)
    {
        var product = await _productRepository
            .GetByIdAsync(request.ProductId, cancellationToken)
            ?? throw new NotFoundException($"ไม่พบสินค้า ID: {request.ProductId}");
            
        await _productRepository.DeleteAsync(product, cancellationToken);
    }
}
```

---

## Query Handlers

### Query Handler พร้อม Pagination

```csharp
// Queries/GetProductsQuery.cs
public record GetProductsQuery(
    string? SearchTerm = null,
    decimal? MinPrice = null,
    decimal? MaxPrice = null,
    bool? InStockOnly = null,
    int Page = 1,
    int PageSize = 20,
    string SortBy = "Name",
    bool SortDescending = false) 
    : IRequest<PagedResult<ProductSummaryDto>>;

public record ProductSummaryDto(
    Guid Id,
    string Name,
    decimal Price,
    int StockQuantity,
    bool IsAvailable);

public record PagedResult<T>(
    IEnumerable<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}

// Queries/GetProductsQueryHandler.cs
public class GetProductsQueryHandler 
    : IRequestHandler<GetProductsQuery, PagedResult<ProductSummaryDto>>
{
    private readonly AppDbContext _context;
    
    public GetProductsQueryHandler(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<PagedResult<ProductSummaryDto>> Handle(
        GetProductsQuery request, CancellationToken cancellationToken)
    {
        var query = _context.Products
            .AsNoTracking()
            .Where(p => p.IsActive);
            
        // Apply filters
        if (!string.IsNullOrWhiteSpace(request.SearchTerm))
            query = query.Where(p => 
                p.Name.Contains(request.SearchTerm) || 
                p.Description.Contains(request.SearchTerm));
                
        if (request.MinPrice.HasValue)
            query = query.Where(p => p.Price >= request.MinPrice.Value);
            
        if (request.MaxPrice.HasValue)
            query = query.Where(p => p.Price <= request.MaxPrice.Value);
            
        if (request.InStockOnly == true)
            query = query.Where(p => p.StockQuantity > 0);
            
        // Apply sorting
        query = request.SortBy.ToLower() switch
        {
            "price" => request.SortDescending 
                ? query.OrderByDescending(p => p.Price)
                : query.OrderBy(p => p.Price),
            "stock" => request.SortDescending
                ? query.OrderByDescending(p => p.StockQuantity)
                : query.OrderBy(p => p.StockQuantity),
            _ => request.SortDescending
                ? query.OrderByDescending(p => p.Name)
                : query.OrderBy(p => p.Name)
        };
        
        // Count total
        var totalCount = await query.CountAsync(cancellationToken);
        
        // Apply pagination and project to DTO
        var items = await query
            .Skip((request.Page - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductSummaryDto(
                p.Id,
                p.Name,
                p.Price,
                p.StockQuantity,
                p.StockQuantity > 0))
            .ToListAsync(cancellationToken);
            
        return new PagedResult<ProductSummaryDto>(
            items, totalCount, request.Page, request.PageSize);
    }
}
```

---

## Pipeline Behaviors

Pipeline Behaviors คือ middleware สำหรับ MediatR ที่ทำงานก่อน/หลัง handler

```csharp
// Common/Behaviors/LoggingBehavior.cs
public class LoggingBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;
    
    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }
    
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        
        _logger.LogInformation("กำลังประมวลผล {RequestName}", requestName);
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var response = await next();
            stopwatch.Stop();
            
            _logger.LogInformation(
                "ประมวลผล {RequestName} สำเร็จ ใช้เวลา {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);
                
            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            _logger.LogError(ex, 
                "เกิดข้อผิดพลาดใน {RequestName} หลังจาก {ElapsedMs}ms",
                requestName, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}

// Common/Behaviors/ValidationBehavior.cs
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }
    
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any()) return await next();
        
        var context = new ValidationContext<TRequest>(request);
        
        var validationResults = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, cancellationToken)));
            
        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();
            
        if (failures.Count != 0)
            throw new ValidationException(failures);
            
        return await next();
    }
}

// Common/Behaviors/PerformanceBehavior.cs
public class PerformanceBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly Stopwatch _timer;
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;
    
    private const int SlowRequestThresholdMs = 500;
    
    public PerformanceBehavior(ILogger<PerformanceBehavior<TRequest, TResponse>> logger)
    {
        _timer = new Stopwatch();
        _logger = logger;
    }
    
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        _timer.Start();
        var response = await next();
        _timer.Stop();
        
        var elapsed = _timer.ElapsedMilliseconds;
        
        if (elapsed > SlowRequestThresholdMs)
        {
            _logger.LogWarning(
                "คำขอช้า: {RequestName} ใช้เวลา {ElapsedMs}ms. Request: {@Request}",
                typeof(TRequest).Name, elapsed, request);
        }
        
        return response;
    }
}
```

---

## Notifications/Events

Notifications คือ messages ที่ส่งให้ handlers หลาย handler พร้อมกัน (fan-out)

```csharp
// Notifications/OrderPlacedNotification.cs
public record OrderPlacedNotification(
    Guid OrderId,
    string CustomerId,
    decimal TotalAmount) : INotification;

// Handlers สำหรับ Notification (สามารถมีได้หลาย handler)
public class SendOrderConfirmationEmailHandler 
    : INotificationHandler<OrderPlacedNotification>
{
    private readonly IEmailService _emailService;
    private readonly ICustomerRepository _customerRepository;
    
    public SendOrderConfirmationEmailHandler(
        IEmailService emailService,
        ICustomerRepository customerRepository)
    {
        _emailService = emailService;
        _customerRepository = customerRepository;
    }
    
    public async Task Handle(
        OrderPlacedNotification notification, CancellationToken cancellationToken)
    {
        var customer = await _customerRepository
            .GetByIdAsync(notification.CustomerId, cancellationToken);
            
        if (customer?.Email != null)
        {
            await _emailService.SendAsync(
                customer.Email,
                "ยืนยันการสั่งซื้อ",
                $"คำสั่งซื้อของคุณ #{notification.OrderId} " +
                $"ยอดรวม {notification.TotalAmount:N2} บาท ได้รับการยืนยันแล้ว",
                cancellationToken);
        }
    }
}

public class UpdateInventoryHandler 
    : INotificationHandler<OrderPlacedNotification>
{
    private readonly IInventoryService _inventoryService;
    
    public UpdateInventoryHandler(IInventoryService inventoryService)
    {
        _inventoryService = inventoryService;
    }
    
    public async Task Handle(
        OrderPlacedNotification notification, CancellationToken cancellationToken)
    {
        await _inventoryService.ReserveItemsAsync(
            notification.OrderId, cancellationToken);
    }
}

public class LogOrderActivityHandler 
    : INotificationHandler<OrderPlacedNotification>
{
    private readonly IActivityLogger _activityLogger;
    
    public LogOrderActivityHandler(IActivityLogger activityLogger)
    {
        _activityLogger = activityLogger;
    }
    
    public async Task Handle(
        OrderPlacedNotification notification, CancellationToken cancellationToken)
    {
        await _activityLogger.LogAsync(
            "ORDER_PLACED",
            $"CustomerId: {notification.CustomerId}, " +
            $"OrderId: {notification.OrderId}, " +
            $"Total: {notification.TotalAmount}",
            cancellationToken);
    }
}
```

---

## โปรแกรมตัวอย่าง: Order Management with CQRS

```csharp
// ===== Models =====
public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }

public class OrderItem
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Guid OrderId { get; set; }
    public string ProductId { get; set; } = "";
    public string ProductName { get; set; } = "";
    public decimal UnitPrice { get; set; }
    public int Quantity { get; set; }
    public decimal Subtotal => UnitPrice * Quantity;
}

public class Order
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string CustomerId { get; set; } = "";
    public DateTime OrderDate { get; set; } = DateTime.UtcNow;
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public List<OrderItem> Items { get; set; } = new();
    public decimal TotalAmount => Items.Sum(i => i.Subtotal);
}

// ===== Commands =====
public record CreateOrderCommand(
    string CustomerId,
    List<OrderItemRequest> Items) : IRequest<CreateOrderResult>;

public record OrderItemRequest(string ProductId, string ProductName, decimal Price, int Quantity);
public record CreateOrderResult(Guid OrderId, decimal Total, string Status);

public record ConfirmOrderCommand(Guid OrderId) : IRequest<bool>;
public record ShipOrderCommand(Guid OrderId, string TrackingNumber) : IRequest<bool>;
public record CancelOrderCommand(Guid OrderId, string Reason) : IRequest<bool>;

// ===== Queries =====
public record GetOrderByIdQuery(Guid OrderId) : IRequest<OrderDetailDto?>;
public record GetAllOrdersQuery(
    string? CustomerId = null, 
    OrderStatus? Status = null) : IRequest<List<OrderSummaryDto>>;

// ===== DTOs =====
public record OrderDetailDto(
    Guid Id, string CustomerId, DateTime OrderDate,
    string Status, decimal Total, List<OrderItemDto> Items);
    
public record OrderItemDto(
    string ProductName, decimal UnitPrice, int Quantity, decimal Subtotal);
    
public record OrderSummaryDto(
    Guid Id, string CustomerId, DateTime OrderDate, string Status, decimal Total);

// ===== Notifications =====
public record OrderStatusChangedNotification(
    Guid OrderId, string CustomerId, 
    OrderStatus OldStatus, OrderStatus NewStatus) : INotification;

// ===== Handlers Implementation =====
// In-Memory Repository สำหรับ demo
public class InMemoryOrderRepository
{
    private readonly Dictionary<Guid, Order> _orders = new();
    
    public Task<Order?> GetByIdAsync(Guid id) 
        => Task.FromResult(_orders.GetValueOrDefault(id));
        
    public Task<List<Order>> GetAllAsync(string? customerId, OrderStatus? status)
    {
        var orders = _orders.Values.AsEnumerable();
        if (customerId != null)
            orders = orders.Where(o => o.CustomerId == customerId);
        if (status != null)
            orders = orders.Where(o => o.Status == status);
        return Task.FromResult(orders.ToList());
    }
    
    public Task AddAsync(Order order)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }
    
    public Task UpdateAsync(Order order)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }
}

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, CreateOrderResult>
{
    private readonly InMemoryOrderRepository _repo;
    private readonly IPublisher _publisher;
    
    public CreateOrderHandler(InMemoryOrderRepository repo, IPublisher publisher)
    {
        _repo = repo;
        _publisher = publisher;
    }
    
    public async Task<CreateOrderResult> Handle(
        CreateOrderCommand request, CancellationToken ct)
    {
        var order = new Order
        {
            CustomerId = request.CustomerId,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                ProductName = i.ProductName,
                UnitPrice = i.Price,
                Quantity = i.Quantity
            }).ToList()
        };
        
        // กำหนด OrderId สำหรับ items
        foreach (var item in order.Items)
            item.OrderId = order.Id;
        
        await _repo.AddAsync(order);
        
        await _publisher.Publish(new OrderStatusChangedNotification(
            order.Id, order.CustomerId, 
            OrderStatus.Pending, OrderStatus.Pending), ct);
        
        return new CreateOrderResult(order.Id, order.TotalAmount, order.Status.ToString());
    }
}

public class ConfirmOrderHandler : IRequestHandler<ConfirmOrderCommand, bool>
{
    private readonly InMemoryOrderRepository _repo;
    private readonly IPublisher _publisher;
    
    public ConfirmOrderHandler(InMemoryOrderRepository repo, IPublisher publisher)
    {
        _repo = repo;
        _publisher = publisher;
    }
    
    public async Task<bool> Handle(ConfirmOrderCommand request, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(request.OrderId);
        if (order == null || order.Status != OrderStatus.Pending) return false;
        
        var oldStatus = order.Status;
        order.Status = OrderStatus.Confirmed;
        await _repo.UpdateAsync(order);
        
        await _publisher.Publish(new OrderStatusChangedNotification(
            order.Id, order.CustomerId, oldStatus, order.Status), ct);
        
        return true;
    }
}

public class GetOrderByIdHandler : IRequestHandler<GetOrderByIdQuery, OrderDetailDto?>
{
    private readonly InMemoryOrderRepository _repo;
    
    public GetOrderByIdHandler(InMemoryOrderRepository repo)
    {
        _repo = repo;
    }
    
    public async Task<OrderDetailDto?> Handle(GetOrderByIdQuery request, CancellationToken ct)
    {
        var order = await _repo.GetByIdAsync(request.OrderId);
        if (order == null) return null;
        
        return new OrderDetailDto(
            order.Id,
            order.CustomerId,
            order.OrderDate,
            order.Status.ToString(),
            order.TotalAmount,
            order.Items.Select(i => new OrderItemDto(
                i.ProductName, i.UnitPrice, i.Quantity, i.Subtotal)).ToList());
    }
}

public class GetAllOrdersHandler : IRequestHandler<GetAllOrdersQuery, List<OrderSummaryDto>>
{
    private readonly InMemoryOrderRepository _repo;
    
    public GetAllOrdersHandler(InMemoryOrderRepository repo)
    {
        _repo = repo;
    }
    
    public async Task<List<OrderSummaryDto>> Handle(GetAllOrdersQuery request, CancellationToken ct)
    {
        var orders = await _repo.GetAllAsync(request.CustomerId, request.Status);
        return orders.Select(o => new OrderSummaryDto(
            o.Id, o.CustomerId, o.OrderDate, o.Status.ToString(), o.TotalAmount
        )).ToList();
    }
}

// Notification Handler
public class OrderStatusChangedHandler : INotificationHandler<OrderStatusChangedNotification>
{
    public Task Handle(OrderStatusChangedNotification notification, CancellationToken ct)
    {
        Console.WriteLine($"[Event] คำสั่งซื้อ {notification.OrderId}: " +
            $"{notification.OldStatus} → {notification.NewStatus} " +
            $"(ลูกค้า: {notification.CustomerId})");
        return Task.CompletedTask;
    }
}

// ===== Program =====
var services = new ServiceCollection();

// ลงทะเบียน MediatR
services.AddMediatR(cfg => 
    cfg.RegisterServicesFromAssembly(typeof(CreateOrderHandler).Assembly));

// ลงทะเบียน Repository
services.AddSingleton<InMemoryOrderRepository>();

var sp = services.BuildServiceProvider();
var mediator = sp.GetRequiredService<IMediator>();

Console.WriteLine("=== Order Management System (CQRS Demo) ===\n");

// สร้างคำสั่งซื้อ
var createResult = await mediator.Send(new CreateOrderCommand(
    "customer-001",
    new List<OrderItemRequest>
    {
        new("p001", "MacBook Pro 16\"", 89000m, 1),
        new("p002", "Magic Mouse", 3500m, 2)
    }));

Console.WriteLine($"สร้างคำสั่งซื้อ: {createResult.OrderId}");
Console.WriteLine($"ยอดรวม: {createResult.Total:N2} บาท\n");

// ยืนยันคำสั่งซื้อ
var confirmed = await mediator.Send(new ConfirmOrderCommand(createResult.OrderId));
Console.WriteLine($"ยืนยันคำสั่งซื้อ: {(confirmed ? "สำเร็จ" : "ล้มเหลว")}\n");

// ดึงข้อมูลคำสั่งซื้อ
var orderDetail = await mediator.Send(new GetOrderByIdQuery(createResult.OrderId));
Console.WriteLine($"รายละเอียดคำสั่งซื้อ:");
Console.WriteLine($"  ID: {orderDetail.Id}");
Console.WriteLine($"  ลูกค้า: {orderDetail.CustomerId}");
Console.WriteLine($"  สถานะ: {orderDetail.Status}");
Console.WriteLine($"  รายการสินค้า:");
foreach (var item in orderDetail.Items)
    Console.WriteLine($"    - {item.ProductName}: {item.Quantity} x {item.UnitPrice:N2} = {item.Subtotal:N2}");
Console.WriteLine($"  ยอดรวม: {orderDetail.Total:N2} บาท");

// แสดงรายการคำสั่งซื้อทั้งหมด
var allOrders = await mediator.Send(new GetAllOrdersQuery());
Console.WriteLine($"\nคำสั่งซื้อทั้งหมด ({allOrders.Count} รายการ):");
foreach (var o in allOrders)
    Console.WriteLine($"  [{o.Status}] {o.Id} - {o.CustomerId} - {o.Total:N2} บาท");
```

---

## Exercises

1. **Exercise 1**: สร้าง `UpdateOrderItemQuantityCommand` สำหรับเปลี่ยนจำนวนสินค้าในคำสั่งซื้อ

2. **Exercise 2**: สร้าง `GetOrderStatisticsQuery` ที่ return สถิติเช่น:
   - จำนวนคำสั่งซื้อตามสถานะ
   - ยอดขายรวม
   - คำสั่งซื้อเฉลี่ยต่อวัน

3. **Exercise 3**: Implement `CachingBehavior<TRequest, TResponse>` ที่ cache ผลลัพธ์ของ Query เป็นเวลา 5 นาที

4. **Exercise 4**: เพิ่ม `TransactionBehavior` ที่ wrap Command ด้วย database transaction

5. **Exercise 5**: ทดสอบ Handler ด้วย xUnit และ Moq

---

## สรุป

CQRS ช่วยให้เราแยก:
- **Commands**: เปลี่ยนแปลงข้อมูล, มี validation เข้มข้น, emit events
- **Queries**: อ่านข้อมูล, optimize สำหรับ read, ไม่ side effects

**MediatR** ทำให้ implement CQRS ง่ายขึ้นด้วย:
- `IRequest<T>` / `IRequestHandler<T>` สำหรับ Commands และ Queries
- `INotification` / `INotificationHandler` สำหรับ Events
- `IPipelineBehavior` สำหรับ cross-cutting concerns

---

## Part ถัดไป

ใน Part 73 เราจะเรียนรู้เรื่อง **Domain-Driven Design (DDD) เบื้องต้น** ซึ่งเป็นแนวทางการออกแบบระบบซับซ้อนโดยเน้นที่ Business Domain เป็นศูนย์กลาง

---

*Part 72/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
