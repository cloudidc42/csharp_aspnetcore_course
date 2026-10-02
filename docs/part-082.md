# Part 082: Message Queue กับ RabbitMQ

## เนื้อหาใน Part นี้
- Message Queue คืออะไร
- RabbitMQ concepts (Exchange, Queue, Binding)
- MassTransit library
- Publishing messages
- Consuming messages
- Saga pattern
- โปรแกรมตัวอย่าง: Order processing

---

## 1. Message Queue คืออะไร

Message Queue เป็น middleware ที่ช่วยให้ระบบสื่อสารกันแบบ asynchronous โดยมี broker รับส่งข้อความ

### ทำไมต้องใช้ Message Queue?

```
ปัญหาของ Direct HTTP Call:
Service A ──HTTP──► Service B  (ถ้า B down, A ก็ fail)

แก้ด้วย Message Queue:
Service A ──► [Queue] ──► Service B  (ถ้า B down, message ยังอยู่ใน queue)
```

**ประโยชน์:**
- **Decoupling**: Producer และ Consumer ไม่รู้จักกัน
- **Reliability**: ข้อความไม่หายแม้ consumer ล่ม
- **Scalability**: เพิ่ม consumers ได้ตามต้องการ
- **Load Leveling**: Queue รับ burst ของ messages แล้วค่อยๆ ส่งให้ consumer
- **Async Processing**: Producer ไม่ต้องรอ consumer

---

## 2. RabbitMQ Concepts

### Exchange, Queue, Binding

```
Publisher ──► Exchange ──binding──► Queue ──► Consumer
```

**Exchange Types:**
- **Direct**: Route ตาม routing key ที่ตรงกัน
- **Fanout**: Broadcast ไปทุก queue ที่ bind
- **Topic**: Route ด้วย pattern matching (*.orange.*, lazy.#)
- **Headers**: Route ตาม message headers

```
Direct Exchange:
Publisher ──[routing_key="order.created"]──► Exchange
                                              ├──[order.created]──► OrderQueue
                                              └──[payment.processed]──► PaymentQueue

Fanout Exchange:
Publisher ──► Exchange ──► QueueA (Email Service)
                       ──► QueueB (SMS Service)
                       ──► QueueC (Analytics Service)

Topic Exchange:
Publisher ──[routing_key="order.created.eu"]──► Exchange
                                                 ├──[order.#]──► AllOrdersQueue
                                                 ├──[*.created.*]──► NewItemsQueue
                                                 └──[order.*.eu]──► EuropeOrdersQueue
```

### Dead Letter Queue

```
Normal Queue ──(message fails)──► Dead Letter Exchange ──► Dead Letter Queue
                                                            └──► Manual Review
```

---

## 3. MassTransit Library

MassTransit เป็น abstraction layer เหนือ message brokers (RabbitMQ, Azure Service Bus, etc.)

### การติดตั้ง

```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.EntityFrameworkCore
```

### Configuration พื้นฐาน

```csharp
// Program.cs
using MassTransit;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddMassTransit(x =>
{
    // Register consumers
    x.AddConsumer<OrderCreatedConsumer>();
    x.AddConsumer<PaymentProcessedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMQ:Host"], "/", h =>
        {
            h.Username(builder.Configuration["RabbitMQ:Username"]!);
            h.Password(builder.Configuration["RabbitMQ:Password"]!);
        });

        // Retry policy
        cfg.UseMessageRetry(r => r.Exponential(5, 
            TimeSpan.FromSeconds(1), 
            TimeSpan.FromSeconds(30), 
            TimeSpan.FromSeconds(5)));

        // Auto-configure endpoints from consumer registrations
        cfg.ConfigureEndpoints(context);
    });
});
```

---

## 4. Publishing Messages

### Message Types

```csharp
// Messages ใน MassTransit แบ่งเป็น 2 ประเภท:

// 1. Event: สิ่งที่เกิดขึ้นแล้ว (Past tense)
public record OrderCreated
{
    public Guid OrderId { get; init; }
    public Guid CustomerId { get; init; }
    public decimal TotalAmount { get; init; }
    public List<OrderItemDto> Items { get; init; } = new();
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}

// 2. Command: สิ่งที่ต้องทำ (Imperative)
public record ProcessPayment
{
    public Guid OrderId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; } = "THB";
    public Guid CustomerId { get; init; }
}

public record OrderItemDto
{
    public Guid ProductId { get; init; }
    public string ProductName { get; init; } = string.Empty;
    public decimal UnitPrice { get; init; }
    public int Quantity { get; init; }
}
```

### Publishing Events

```csharp
// ด้วย IPublishEndpoint (for events - 1 to many)
public class OrderService
{
    private readonly IPublishEndpoint _publishEndpoint;
    private readonly IOrderRepository _repository;

    public OrderService(IPublishEndpoint publishEndpoint, IOrderRepository repository)
    {
        _publishEndpoint = publishEndpoint;
        _repository = repository;
    }

    public async Task<Guid> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct = default)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = request.CustomerId,
            Items = request.Items.Select(i => new OrderItem
            {
                ProductId = i.ProductId,
                ProductName = i.ProductName,
                UnitPrice = i.UnitPrice,
                Quantity = i.Quantity
            }).ToList(),
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        await _repository.AddAsync(order, ct);
        await _repository.SaveChangesAsync(ct);

        // Publish event หลัง save สำเร็จ
        await _publishEndpoint.Publish(new OrderCreated
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            TotalAmount = order.Items.Sum(i => i.UnitPrice * i.Quantity),
            Items = order.Items.Select(i => new OrderItemDto
            {
                ProductId = i.ProductId,
                ProductName = i.ProductName,
                UnitPrice = i.UnitPrice,
                Quantity = i.Quantity
            }).ToList()
        }, ct);

        return order.Id;
    }
}

// ด้วย ISendEndpointProvider (for commands - 1 to 1)
public class PaymentOrchestrator
{
    private readonly ISendEndpointProvider _sendEndpointProvider;

    public PaymentOrchestrator(ISendEndpointProvider sendEndpointProvider)
    {
        _sendEndpointProvider = sendEndpointProvider;
    }

    public async Task SendPaymentCommandAsync(Guid orderId, decimal amount, Guid customerId)
    {
        var endpoint = await _sendEndpointProvider.GetSendEndpoint(
            new Uri("queue:process-payment"));
        
        await endpoint.Send(new ProcessPayment
        {
            OrderId = orderId,
            Amount = amount,
            CustomerId = customerId
        });
    }
}
```

### Transactional Outbox Pattern

```csharp
// ป้องกันปัญหา: save DB สำเร็จ แต่ publish fail (หรือกลับกัน)
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UsePostgres(); // หรือ UseSqlServer()
        o.UseBusOutbox();
    });
    
    // ...rest of config
});

// ใช้งาน - message จะ save ใน DB ก่อน แล้ว background service จะส่งให้
public class OrderService
{
    private readonly AppDbContext _dbContext;
    private readonly IPublishEndpoint _publishEndpoint;

    public async Task CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
    {
        await using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);

        var order = new Order { /* ... */ };
        _dbContext.Orders.Add(order);

        // นี้จะ save ลง outbox table แทน publish ตรงๆ
        await _publishEndpoint.Publish(new OrderCreated { OrderId = order.Id }, ct);

        await _dbContext.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct); // Outbox message saved atomically

        // Background service จะ deliver message จาก outbox
    }
}
```

---

## 5. Consuming Messages

### Basic Consumer

```csharp
// Consumer สำหรับ OrderCreated event
public class OrderCreatedConsumer : IConsumer<OrderCreated>
{
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<OrderCreatedConsumer> _logger;

    public OrderCreatedConsumer(
        IInventoryService inventoryService, 
        ILogger<OrderCreatedConsumer> logger)
    {
        _inventoryService = inventoryService;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var message = context.Message;
        _logger.LogInformation("Processing OrderCreated: {OrderId}", message.OrderId);

        try
        {
            foreach (var item in message.Items)
            {
                await _inventoryService.ReserveStockAsync(
                    item.ProductId, 
                    item.Quantity, 
                    context.CancellationToken);
            }

            _logger.LogInformation("Stock reserved for order {OrderId}", message.OrderId);
        }
        catch (InsufficientStockException ex)
        {
            _logger.LogWarning("Insufficient stock for order {OrderId}: {Message}", 
                message.OrderId, ex.Message);

            // Publish compensation event
            await context.Publish(new OrderInventoryFailed
            {
                OrderId = message.OrderId,
                Reason = ex.Message
            });
        }
    }
}

// Registration
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>(cfg =>
    {
        cfg.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(5)));
        cfg.UseInMemoryOutbox(); // เพื่อ exactly-once processing
    });
});
```

### Consumer Definition (สำหรับ configuration)

```csharp
public class OrderCreatedConsumerDefinition : ConsumerDefinition<OrderCreatedConsumer>
{
    protected override void ConfigureConsumer(
        IReceiveEndpointConfigurator endpointConfigurator,
        IConsumerConfigurator<OrderCreatedConsumer> consumerConfigurator,
        IRegistrationContext context)
    {
        endpointConfigurator.UseMessageRetry(r =>
        {
            r.Incremental(3, TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(10));
            r.Handle<DbUpdateException>();
            r.Handle<HttpRequestException>();
            r.Ignore<ArgumentException>();
        });

        endpointConfigurator.UseInMemoryOutbox(context);
        
        // Dead letter queue
        endpointConfigurator.BindDeadLetterQueue("order-created-dlq");
    }
}

// Registration จะ auto-discover ConsumerDefinition
builder.Services.AddMassTransit(x =>
{
    x.AddConsumers(Assembly.GetExecutingAssembly());
    // ...
});
```

### Batch Consumer

```csharp
// รับ messages เป็น batch
public class OrderAnalyticsConsumer : IConsumer<Batch<OrderCreated>>
{
    private readonly IAnalyticsRepository _analyticsRepo;

    public OrderAnalyticsConsumer(IAnalyticsRepository analyticsRepo)
    {
        _analyticsRepo = analyticsRepo;
    }

    public async Task Consume(ConsumeContext<Batch<OrderCreated>> context)
    {
        var orders = context.Message.Select(x => new OrderAnalytics
        {
            OrderId = x.Message.OrderId,
            CustomerId = x.Message.CustomerId,
            Amount = x.Message.TotalAmount,
            CreatedAt = x.Message.CreatedAt
        }).ToList();

        await _analyticsRepo.BulkInsertAsync(orders);
        
        Console.WriteLine($"Processed batch of {context.Message.Length} orders");
    }
}

// Registration
x.AddConsumer<OrderAnalyticsConsumer>(cfg =>
{
    cfg.Options<BatchOptions>(options => options
        .SetMessageLimit(100)        // batch สูงสุด 100 messages
        .SetTimeLimit(TimeSpan.FromSeconds(10))  // หรือรอ 10 วินาที
        .SetConcurrencyLimit(5));    // 5 batches พร้อมกัน
});
```

---

## 6. Saga Pattern

Saga จัดการ long-running business processes ที่ span หลาย services

### State Machine Saga

```csharp
// Order Saga State
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    
    public Guid OrderId { get; set; }
    public Guid CustomerId { get; set; }
    public decimal Amount { get; set; }
    
    // Tracking ว่า step ไหนสำเร็จแล้ว
    public bool InventoryReserved { get; set; }
    public bool PaymentProcessed { get; set; }
    
    public DateTime CreatedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
}

// Order Saga State Machine
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    // States
    public State WaitingForInventory { get; private set; } = null!;
    public State WaitingForPayment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    // Events
    public Event<OrderCreated> OrderCreated { get; private set; } = null!;
    public Event<InventoryReserved> InventoryReserved { get; private set; } = null!;
    public Event<InventoryReservationFailed> InventoryFailed { get; private set; } = null!;
    public Event<PaymentProcessed> PaymentProcessed { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = null!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        // Correlate events ด้วย OrderId
        Event(() => OrderCreated, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryFailed, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentProcessed, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(ctx => ctx.Message.OrderId));

        // Initial state -> WaitingForInventory
        Initially(
            When(OrderCreated)
                .Then(ctx =>
                {
                    ctx.Saga.OrderId = ctx.Message.OrderId;
                    ctx.Saga.CustomerId = ctx.Message.CustomerId;
                    ctx.Saga.Amount = ctx.Message.TotalAmount;
                    ctx.Saga.CreatedAt = DateTime.UtcNow;
                })
                .Publish(ctx => new ReserveInventory
                {
                    OrderId = ctx.Message.OrderId,
                    Items = ctx.Message.Items
                })
                .TransitionTo(WaitingForInventory)
        );

        // Inventory step
        During(WaitingForInventory,
            When(InventoryReserved)
                .Then(ctx =>
                {
                    ctx.Saga.InventoryReserved = true;
                })
                .Publish(ctx => new ProcessPayment
                {
                    OrderId = ctx.Saga.OrderId,
                    Amount = ctx.Saga.Amount,
                    CustomerId = ctx.Saga.CustomerId
                })
                .TransitionTo(WaitingForPayment),

            When(InventoryFailed)
                .Publish(ctx => new CancelOrder
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason = "Inventory unavailable"
                })
                .TransitionTo(Failed)
        );

        // Payment step
        During(WaitingForPayment,
            When(PaymentProcessed)
                .Then(ctx =>
                {
                    ctx.Saga.PaymentProcessed = true;
                    ctx.Saga.CompletedAt = DateTime.UtcNow;
                })
                .Publish(ctx => new OrderCompleted
                {
                    OrderId = ctx.Saga.OrderId,
                    CustomerId = ctx.Saga.CustomerId
                })
                .TransitionTo(Completed),

            When(PaymentFailed)
                .Publish(ctx => new ReleaseInventory
                {
                    OrderId = ctx.Saga.OrderId
                })
                .Publish(ctx => new CancelOrder
                {
                    OrderId = ctx.Saga.OrderId,
                    Reason = "Payment failed"
                })
                .TransitionTo(Failed)
        );
    }
}

// Messages สำหรับ Saga
public record ReserveInventory
{
    public Guid OrderId { get; init; }
    public List<OrderItemDto> Items { get; init; } = new();
}

public record InventoryReserved { public Guid OrderId { get; init; } }
public record InventoryReservationFailed { public Guid OrderId { get; init; } public string Reason { get; init; } = string.Empty; }
public record ProcessPayment { public Guid OrderId { get; init; } public decimal Amount { get; init; } public Guid CustomerId { get; init; } }
public record PaymentProcessed { public Guid OrderId { get; init; } public string TransactionId { get; init; } = string.Empty; }
public record PaymentFailed { public Guid OrderId { get; init; } public string Reason { get; init; } = string.Empty; }
public record ReleaseInventory { public Guid OrderId { get; init; } }
public record CancelOrder { public Guid OrderId { get; init; } public string Reason { get; init; } = string.Empty; }
public record OrderCompleted { public Guid OrderId { get; init; } public Guid CustomerId { get; init; } }
```

### Saga Registration

```csharp
builder.Services.AddMassTransit(x =>
{
    // Add Saga
    x.AddSagaStateMachine<OrderStateMachine, OrderState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;
            r.AddDbContext<DbContext, SagaDbContext>((provider, options) =>
            {
                options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"));
            });
        });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost");
        cfg.ConfigureEndpoints(context);
    });
});

// DbContext สำหรับ Saga
public class SagaDbContext : SagaDbContext<OrderState>
{
    public SagaDbContext(DbContextOptions<SagaDbContext> options) : base(options) { }

    protected override IEnumerable<ISagaClassMap> Configurations
    {
        get { yield return new OrderStateMap(); }
    }
}

public class OrderStateMap : SagaClassMap<OrderState>
{
    protected override void Configure(EntityTypeBuilder<OrderState> entity, ModelBuilder model)
    {
        entity.Property(x => x.CurrentState).HasMaxLength(64);
        entity.Property(x => x.Amount).HasColumnType("decimal(18,2)");
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Order Processing System

### โครงสร้างโปรเจกต์

```
OrderProcessing/
├── src/
│   ├── OrderService/
│   │   ├── Controllers/
│   │   ├── Messages/
│   │   ├── Sagas/
│   │   └── Program.cs
│   ├── InventoryService/
│   │   ├── Consumers/
│   │   └── Program.cs
│   └── PaymentService/
│       ├── Consumers/
│       └── Program.cs
└── docker-compose.yml
```

### docker-compose.yml

```yaml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 30s
      retries: 3

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: orderprocessing
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"

  order-service:
    build: ./src/OrderService
    environment:
      - RabbitMQ__Host=rabbitmq
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=orderprocessing;Username=postgres;Password=postgres
    depends_on:
      rabbitmq:
        condition: service_healthy
      postgres:
        condition: service_started
    ports:
      - "5001:8080"

  inventory-service:
    build: ./src/InventoryService
    environment:
      - RabbitMQ__Host=rabbitmq
    depends_on:
      rabbitmq:
        condition: service_healthy
    ports:
      - "5002:8080"

  payment-service:
    build: ./src/PaymentService
    environment:
      - RabbitMQ__Host=rabbitmq
    depends_on:
      rabbitmq:
        condition: service_healthy
    ports:
      - "5003:8080"
```

### InventoryService Consumer

```csharp
// InventoryService/Consumers/ReserveInventoryConsumer.cs
public class ReserveInventoryConsumer : IConsumer<ReserveInventory>
{
    private readonly IInventoryRepository _repo;
    private readonly ILogger<ReserveInventoryConsumer> _logger;

    public ReserveInventoryConsumer(
        IInventoryRepository repo, 
        ILogger<ReserveInventoryConsumer> logger)
    {
        _repo = repo;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<ReserveInventory> context)
    {
        _logger.LogInformation("Reserving inventory for order {OrderId}", context.Message.OrderId);

        var insufficientItems = new List<string>();

        foreach (var item in context.Message.Items)
        {
            var inventory = await _repo.GetByProductIdAsync(item.ProductId);
            
            if (inventory == null || inventory.AvailableStock < item.Quantity)
            {
                insufficientItems.Add($"Product {item.ProductId}: requested {item.Quantity}");
            }
        }

        if (insufficientItems.Any())
        {
            await context.Publish(new InventoryReservationFailed
            {
                OrderId = context.Message.OrderId,
                Reason = $"Insufficient stock: {string.Join(", ", insufficientItems)}"
            });
            return;
        }

        // Reserve stock
        foreach (var item in context.Message.Items)
        {
            await _repo.ReserveStockAsync(item.ProductId, item.Quantity);
        }

        await context.Publish(new InventoryReserved
        {
            OrderId = context.Message.OrderId
        });

        _logger.LogInformation("Inventory reserved for order {OrderId}", context.Message.OrderId);
    }
}
```

### PaymentService Consumer

```csharp
// PaymentService/Consumers/ProcessPaymentConsumer.cs
public class ProcessPaymentConsumer : IConsumer<ProcessPayment>
{
    private readonly IPaymentGateway _paymentGateway;
    private readonly ILogger<ProcessPaymentConsumer> _logger;

    public ProcessPaymentConsumer(
        IPaymentGateway paymentGateway,
        ILogger<ProcessPaymentConsumer> logger)
    {
        _paymentGateway = paymentGateway;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<ProcessPayment> context)
    {
        _logger.LogInformation("Processing payment for order {OrderId}", context.Message.OrderId);

        try
        {
            var result = await _paymentGateway.ChargeAsync(
                customerId: context.Message.CustomerId,
                amount: context.Message.Amount,
                currency: context.Message.Currency,
                idempotencyKey: context.Message.OrderId.ToString());

            if (result.Success)
            {
                await context.Publish(new PaymentProcessed
                {
                    OrderId = context.Message.OrderId,
                    TransactionId = result.TransactionId
                });
            }
            else
            {
                await context.Publish(new PaymentFailed
                {
                    OrderId = context.Message.OrderId,
                    Reason = result.ErrorMessage
                });
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Payment failed for order {OrderId}", context.Message.OrderId);
            
            await context.Publish(new PaymentFailed
            {
                OrderId = context.Message.OrderId,
                Reason = "Payment processing error"
            });
        }
    }
}
```

### OrderService Program.cs (สมบูรณ์)

```csharp
// OrderService/Program.cs
using MassTransit;
using Microsoft.EntityFrameworkCore;
using OrderService.Sagas;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<OrderDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// MassTransit
builder.Services.AddMassTransit(x =>
{
    // Saga
    x.AddSagaStateMachine<OrderStateMachine, OrderState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Optimistic;
            r.AddDbContext<DbContext, OrderDbContext>((provider, options) =>
            {
                options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"));
            });
        });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration["RabbitMQ:Host"] ?? "localhost", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        // Global retry
        cfg.UseMessageRetry(r => r.Interval(3, 1000));
        
        cfg.ConfigureEndpoints(context);
    });
});

// API
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Migrate database
using (var scope = app.Services.CreateScope())
{
    await scope.ServiceProvider.GetRequiredService<OrderDbContext>().Database.MigrateAsync();
}

app.UseSwagger();
app.UseSwaggerUI();

// Endpoints
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    IBus bus,
    OrderDbContext db) =>
{
    var orderId = Guid.NewGuid();
    
    db.Orders.Add(new Order
    {
        Id = orderId,
        CustomerId = request.CustomerId,
        Status = "Pending",
        CreatedAt = DateTime.UtcNow
    });
    await db.SaveChangesAsync();

    // Start the saga by publishing OrderCreated event
    await bus.Publish(new OrderCreated
    {
        OrderId = orderId,
        CustomerId = request.CustomerId,
        TotalAmount = request.Items.Sum(i => i.Price * i.Quantity),
        Items = request.Items.Select(i => new OrderItemDto
        {
            ProductId = i.ProductId,
            ProductName = i.ProductName,
            UnitPrice = i.Price,
            Quantity = i.Quantity
        }).ToList()
    });

    return Results.Accepted($"/api/orders/{orderId}", new { OrderId = orderId });
});

app.MapGet("/api/orders/{id:guid}", async (Guid id, OrderDbContext db) =>
{
    var order = await db.Orders.FindAsync(id);
    return order is null ? Results.NotFound() : Results.Ok(order);
});

// Saga state endpoint
app.MapGet("/api/orders/{id:guid}/saga-state", async (
    Guid id,
    OrderDbContext db) =>
{
    var sagaState = await db.Set<OrderState>().FindAsync(id);
    return sagaState is null 
        ? Results.NotFound() 
        : Results.Ok(new 
        { 
            sagaState.OrderId, 
            sagaState.CurrentState,
            sagaState.InventoryReserved,
            sagaState.PaymentProcessed,
            sagaState.CreatedAt,
            sagaState.CompletedAt
        });
});

app.Run();

public record CreateOrderRequest(
    Guid CustomerId,
    List<CreateOrderItemRequest> Items);

public record CreateOrderItemRequest(
    Guid ProductId,
    string ProductName,
    decimal Price,
    int Quantity);
```

---

## 8. Request/Response Pattern

```csharp
// ส่ง request และรอ response (synchronous ผ่าน queue)
public class OrderController : ControllerBase
{
    private readonly IRequestClient<CheckInventory> _checkInventoryClient;

    public OrderController(IRequestClient<CheckInventory> checkInventoryClient)
    {
        _checkInventoryClient = checkInventoryClient;
    }

    [HttpGet("check-availability/{productId:guid}/{quantity:int}")]
    public async Task<IActionResult> CheckAvailability(Guid productId, int quantity)
    {
        var response = await _checkInventoryClient.GetResponse<InventoryCheckResult>(
            new CheckInventory { ProductId = productId, Quantity = quantity },
            timeout: RequestTimeout.After(s: 5));

        return Ok(response.Message);
    }
}

// Request message
public record CheckInventory
{
    public Guid ProductId { get; init; }
    public int Quantity { get; init; }
}

// Response message
public record InventoryCheckResult
{
    public Guid ProductId { get; init; }
    public bool IsAvailable { get; init; }
    public int AvailableStock { get; init; }
}

// Responder ใน InventoryService
public class CheckInventoryConsumer : IConsumer<CheckInventory>
{
    private readonly IInventoryRepository _repo;

    public CheckInventoryConsumer(IInventoryRepository repo)
    {
        _repo = repo;
    }

    public async Task Consume(ConsumeContext<CheckInventory> context)
    {
        var inventory = await _repo.GetByProductIdAsync(context.Message.ProductId);
        
        await context.RespondAsync(new InventoryCheckResult
        {
            ProductId = context.Message.ProductId,
            IsAvailable = inventory?.AvailableStock >= context.Message.Quantity,
            AvailableStock = inventory?.AvailableStock ?? 0
        });
    }
}

// Registration
builder.Services.AddMassTransit(x =>
{
    x.AddRequestClient<CheckInventory>();
    // ...
});
```

---

## Exercises / Project Tasks

### Exercise 1: Basic Producer/Consumer
สร้างระบบ notification อย่างง่าย:
- UserRegistered event
- EmailNotificationConsumer
- SMSNotificationConsumer

### Exercise 2: Saga Pattern
ออกแบบ Saga สำหรับ Hotel Booking:
```
BookHotel →
  Reserve Room →
    Process Payment →
      Confirm Booking / Rollback
```

### Exercise 3: Dead Letter Queue Handling
เพิ่ม DLQ handling:
- Consumer fail 3 times → move to DLQ
- Monitoring endpoint สำหรับ DLQ messages
- Requeue mechanism

### Exercise 4: Request/Response
สร้าง pricing service ที่รับ request และตอบกลับราคา:
- ProductPricingRequest
- ProductPricingResponse
- Test timeout handling

---

## สรุป

- **Message Queue** แก้ปัญหา coupling และ reliability ใน distributed systems
- **RabbitMQ** ใช้ Exchange, Queue, Binding จัดการ routing
- **MassTransit** เป็น abstraction ที่ทำให้ใช้งาน RabbitMQ ง่ายขึ้น
- **Events vs Commands**: Events = สิ่งที่เกิดขึ้นแล้ว (1-to-many), Commands = สิ่งที่สั่งให้ทำ (1-to-1)
- **Saga Pattern** จัดการ long-running processes ด้วย State Machine
- **Transactional Outbox** ป้องกันข้อมูลสูญหายระหว่าง DB transaction กับ message publishing
- **Request/Response** pattern ให้ synchronous behavior ผ่าน queue

---

## Part ถัดไป

**Part 083: Docker กับ .NET** - เรียนรู้การ containerize แอปพลิเคชัน .NET ด้วย Docker

---

*Part 082/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
