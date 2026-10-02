# Part 092: Observability: Metrics, Tracing, Logging

## เนื้อหาใน Part นี้
- OpenTelemetry
- Distributed tracing
- Prometheus metrics
- Grafana dashboards
- Health monitoring
- โปรแกรมตัวอย่าง: Observable API

---

## 1. Observability คืออะไร

Observability = ความสามารถในการเข้าใจสถานะของระบบจากภายนอก โดยไม่ต้อง modify code

**3 Pillars of Observability:**
```
1. Logs    - Events ที่เกิดขึ้น (structured text)
2. Metrics - Numbers ที่วัดได้ (CPU, memory, request count)
3. Traces  - ติดตาม request journey ข้าม services
```

### OpenTelemetry

OpenTelemetry เป็น standard open-source framework สำหรับ telemetry collection

```
Application
    │
    └── OpenTelemetry SDK
            │
            ├── Logs ──────► Log Backend (Elasticsearch, Loki)
            ├── Metrics ───► Metric Backend (Prometheus)
            └── Traces ────► Trace Backend (Jaeger, Zipkin, Tempo)
```

---

## 2. การติดตั้ง OpenTelemetry

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Exporter.Otlp
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore
dotnet add package OpenTelemetry.Exporter.Console
```

### Program.cs

```csharp
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// Service resource info
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService(
        serviceName: "order-api",
        serviceVersion: "1.0.0",
        serviceInstanceId: Environment.MachineName)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName,
        ["team.name"] = "platform"
    });

// ─── OpenTelemetry Setup ──────────────────────────────────────

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation(options =>
            {
                options.RecordException = true;
                options.EnrichWithHttpRequest = (activity, request) =>
                {
                    activity.SetTag("http.client_ip", request.HttpContext.Connection.RemoteIpAddress?.ToString());
                };
            })
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation(options =>
            {
                options.SetDbStatementForText = true;
            })
            .AddSource("OrderService.*")  // Custom activities
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317");
            });
        
        if (builder.Environment.IsDevelopment())
        {
            tracing.AddConsoleExporter();
        }
    })
    .WithMetrics(metrics =>
    {
        metrics
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddMeter("OrderService.*")  // Custom meters
            .AddPrometheusExporter()
            .AddOtlpExporter(options =>
            {
                options.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317");
            });
    });

// Logging
builder.Logging.AddOpenTelemetry(logging =>
{
    logging
        .SetResourceBuilder(resourceBuilder)
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317");
        });
    
    if (builder.Environment.IsDevelopment())
    {
        logging.AddConsoleExporter();
    }
});

var app = builder.Build();

// Expose Prometheus metrics endpoint
app.MapPrometheusScrapingEndpoint("/metrics");

app.Run();
```

---

## 3. Distributed Tracing

### Custom Activity (Span)

```csharp
using System.Diagnostics;

// Define ActivitySource สำหรับ service
public static class Telemetry
{
    public static readonly ActivitySource ActivitySource = new("OrderService.Api", "1.0.0");
}

// ใช้งานใน service
public class OrderService
{
    private static readonly ActivitySource _activitySource = new("OrderService.Api");
    private readonly IOrderRepository _repository;
    private readonly IInventoryClient _inventoryClient;

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
    {
        // สร้าง span
        using var activity = _activitySource.StartActivity("CreateOrder");
        activity?.SetTag("order.customer_id", request.CustomerId.ToString());
        activity?.SetTag("order.item_count", request.Items.Count);
        
        try
        {
            // Check inventory ใน sub-span
            using var inventoryActivity = _activitySource.StartActivity("CheckInventory");
            
            foreach (var item in request.Items)
            {
                inventoryActivity?.SetTag("product.id", item.ProductId.ToString());
                
                var available = await _inventoryClient.CheckAvailabilityAsync(
                    item.ProductId, item.Quantity, ct);
                
                if (!available)
                {
                    inventoryActivity?.SetStatus(ActivityStatusCode.Error, "Inventory not available");
                    throw new InsufficientInventoryException(item.ProductId);
                }
            }
            
            // Create order ใน sub-span
            using var createActivity = _activitySource.StartActivity("PersistOrder");
            var order = await _repository.CreateAsync(request, ct);
            
            createActivity?.SetTag("order.id", order.Id.ToString());
            
            // Set final status
            activity?.SetTag("order.id", order.Id.ToString());
            activity?.SetStatus(ActivityStatusCode.Ok);
            
            return order;
        }
        catch (Exception ex)
        {
            // Record exception ใน trace
            activity?.RecordException(ex);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            throw;
        }
    }
}
```

### Propagating Context

```csharp
// HTTP Client - auto-propagated ด้วย AddHttpClientInstrumentation

// Manual propagation สำหรับ queue messages
public class OrderEventPublisher
{
    private readonly IBus _bus;
    private static readonly ActivitySource _activitySource = new("OrderService.Events");

    public async Task PublishOrderCreatedAsync(Order order, CancellationToken ct)
    {
        using var activity = _activitySource.StartActivity("PublishOrderCreated");
        
        // Capture current trace context
        var traceContext = new Dictionary<string, string>();
        
        if (Activity.Current != null)
        {
            Propagators.DefaultTextMapPropagator.Inject(
                new PropagationContext(Activity.Current.Context, Baggage.Current),
                traceContext,
                (dict, key, value) => dict[key] = value);
        }
        
        await _bus.Publish(new OrderCreatedEvent
        {
            OrderId = order.Id,
            TraceContext = traceContext  // Carry trace context ใน message
        });
    }
}

// Consumer - extract trace context
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        // Restore trace context
        var parentContext = Propagators.DefaultTextMapPropagator.Extract(
            default,
            context.Message.TraceContext,
            (dict, key) => dict.TryGetValue(key, out var value) ? new[] { value } : Array.Empty<string>());
        
        using var activity = Telemetry.ActivitySource.StartActivity(
            "ProcessOrderCreated",
            ActivityKind.Consumer,
            parentContext.ActivityContext);
        
        activity?.SetTag("order.id", context.Message.OrderId.ToString());
        
        // Process...
    }
}
```

---

## 4. Custom Metrics

```csharp
using System.Diagnostics.Metrics;

// Metrics definitions
public static class OrderMetrics
{
    private static readonly Meter _meter = new("OrderService.Api", "1.0.0");
    
    // Counter - นับจำนวน
    public static readonly Counter<long> OrdersCreated = 
        _meter.CreateCounter<long>("orders.created", "orders", "Total orders created");
    
    public static readonly Counter<long> OrdersFailed = 
        _meter.CreateCounter<long>("orders.failed", "orders", "Total orders that failed");
    
    // Histogram - วัด distribution
    public static readonly Histogram<double> OrderValue = 
        _meter.CreateHistogram<double>("orders.value", "THB", "Order value distribution");
    
    public static readonly Histogram<double> OrderProcessingTime = 
        _meter.CreateHistogram<double>("orders.processing_time", "ms", "Order processing duration");
    
    // Gauge - วัด current state
    public static readonly ObservableGauge<int> ActiveOrders = 
        _meter.CreateObservableGauge<int>("orders.active", GetActiveOrderCount, "orders", "Currently active orders");
    
    private static int GetActiveOrderCount()
    {
        // ดึงจาก database หรือ cache
        return _activeOrdersGauge;
    }
    
    private static int _activeOrdersGauge = 0;
    public static void SetActiveOrders(int count) => _activeOrdersGauge = count;
    
    // UpDownCounter - นับ +/-
    public static readonly UpDownCounter<int> QueuedMessages =
        _meter.CreateUpDownCounter<int>("orders.queue.length", "messages", "Messages in order queue");
}

// ใช้งานใน service
public class OrderService
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            var order = await ProcessOrderAsync(request, ct);
            
            stopwatch.Stop();
            
            // Record metrics
            OrderMetrics.OrdersCreated.Add(1, 
                new KeyValuePair<string, object?>("status", "success"),
                new KeyValuePair<string, object?>("customer_type", request.CustomerType));
            
            OrderMetrics.OrderValue.Record(
                (double)order.TotalAmount,
                new KeyValuePair<string, object?>("category", order.Category));
            
            OrderMetrics.OrderProcessingTime.Record(
                stopwatch.Elapsed.TotalMilliseconds,
                new KeyValuePair<string, object?>("operation", "create"));
            
            return order;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            
            OrderMetrics.OrdersFailed.Add(1,
                new KeyValuePair<string, object?>("error_type", ex.GetType().Name));
            
            throw;
        }
    }
}
```

---

## 5. Structured Logging

```csharp
using Serilog;
using Serilog.Sinks.OpenTelemetry;

// Program.cs - Serilog setup
builder.Host.UseSerilog((context, services, configuration) =>
{
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .Enrich.WithProperty("Application", "OrderService")
        .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter())
        .WriteTo.OpenTelemetry(options =>
        {
            options.Endpoint = context.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317";
            options.Protocol = OtlpProtocol.Grpc;
            options.ResourceAttributes = new Dictionary<string, object>
            {
                ["service.name"] = "order-api"
            };
        });
});

// ใช้งาน structured logging
public class OrderController : ControllerBase
{
    private readonly ILogger<OrderController> _logger;
    
    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
    {
        // Structured log - ใช้ property แทน string interpolation
        _logger.LogInformation(
            "Creating order for customer {CustomerId} with {ItemCount} items",
            request.CustomerId,
            request.Items.Count);
        
        try
        {
            var order = await _orderService.CreateOrderAsync(request);
            
            _logger.LogInformation(
                "Order {OrderId} created successfully with value {OrderValue:C}",
                order.Id,
                order.TotalAmount);
            
            return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
        }
        catch (InsufficientInventoryException ex)
        {
            _logger.LogWarning(
                ex,
                "Order creation failed due to insufficient inventory for product {ProductId}",
                ex.ProductId);
            
            return BadRequest(new { error = "Insufficient inventory" });
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Unexpected error creating order for customer {CustomerId}",
                request.CustomerId);
            
            return StatusCode(500, "An internal error occurred");
        }
    }
}
```

### Serilog Configuration

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.EntityFrameworkCore": "Warning"
      }
    },
    "Destructure": {
      "MaxDepth": 5,
      "MaxStringLength": 500
    }
  }
}
```

---

## 6. Health Monitoring

```csharp
// Health checks
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database",
        tags: new[] { "ready", "db" })
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"] ?? "localhost:6379",
        name: "cache",
        tags: new[] { "ready", "cache" })
    .AddRabbitMQ(
        rabbitMQConnectionString: builder.Configuration["RabbitMQ:Url"] ?? "amqp://localhost",
        name: "message-bus",
        tags: new[] { "ready", "messaging" })
    .AddUrlGroup(
        new Uri(builder.Configuration["Services:PaymentService:Url"] + "/health"),
        name: "payment-service",
        tags: new[] { "ready", "dependencies" })
    .AddCheck<DiskStorageHealthCheck>("disk", tags: new[] { "ready" })
    .AddCheck<MemoryHealthCheck>("memory", tags: new[] { "live" });

// Custom health check
public class DiskStorageHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        var drive = DriveInfo.GetDrives().FirstOrDefault(d => d.Name == "/");
        
        if (drive == null)
            return Task.FromResult(HealthCheckResult.Degraded("Cannot read disk info"));
        
        var freePercent = (double)drive.AvailableFreeSpace / drive.TotalSize * 100;
        
        var data = new Dictionary<string, object>
        {
            { "free_gb", Math.Round(drive.AvailableFreeSpace / 1024.0 / 1024 / 1024, 2) },
            { "total_gb", Math.Round(drive.TotalSize / 1024.0 / 1024 / 1024, 2) },
            { "free_percent", Math.Round(freePercent, 1) }
        };
        
        return freePercent switch
        {
            < 5 => Task.FromResult(HealthCheckResult.Unhealthy("Critical: Disk space < 5%", data: data)),
            < 15 => Task.FromResult(HealthCheckResult.Degraded($"Warning: Disk space {freePercent:F1}%", data: data)),
            _ => Task.FromResult(HealthCheckResult.Healthy($"Disk space: {freePercent:F1}%", data))
        };
    }
}

public class MemoryHealthCheck : IHealthCheck
{
    private const long MaxMemoryBytes = 512 * 1024 * 1024; // 512MB
    
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken ct = default)
    {
        var gcInfo = GC.GetGCMemoryInfo();
        var allocatedMB = GC.GetTotalMemory(false) / 1024.0 / 1024;
        
        var data = new Dictionary<string, object>
        {
            { "allocated_mb", Math.Round(allocatedMB, 2) },
            { "heap_size_mb", Math.Round(gcInfo.HeapSizeBytes / 1024.0 / 1024, 2) },
            { "gen0_count", GC.CollectionCount(0) },
            { "gen1_count", GC.CollectionCount(1) },
            { "gen2_count", GC.CollectionCount(2) }
        };
        
        if (GC.GetTotalMemory(false) > MaxMemoryBytes)
            return Task.FromResult(HealthCheckResult.Degraded(
                $"High memory usage: {allocatedMB:F0}MB", data: data));
        
        return Task.FromResult(HealthCheckResult.Healthy(
            $"Memory: {allocatedMB:F0}MB", data));
    }
}

// Map health endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

---

## 7. โปรแกรมตัวอย่าง: Observable API

### docker-compose.yml สำหรับ Observability Stack

```yaml
# docker-compose.observability.yml
version: '3.8'

services:
  # Application
  api:
    build: .
    environment:
      - Otlp__Endpoint=http://otelcol:4317
    depends_on:
      - otelcol

  # OpenTelemetry Collector
  otelcol:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./config/otelcol.yaml:/etc/otelcol/config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    command: ["--config=/etc/otelcol/config.yaml"]

  # Prometheus (metrics)
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  # Grafana (dashboards)
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./config/grafana/datasources:/etc/grafana/provisioning/datasources
      - ./config/grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3000:3000"

  # Jaeger (distributed tracing)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "14250:14250"  # gRPC

  # Loki (logs)
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
```

### otelcol.yaml

```yaml
# config/otelcol.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1000
  
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write
  
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

### prometheus.yml

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'order-api'
    static_configs:
      - targets: ['api:8080']
    metrics_path: '/metrics'
    scheme: http

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### สรุป Program.cs ที่สมบูรณ์

```csharp
// Program.cs - Complete Observable API
using OpenTelemetry;
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Serilog
builder.Host.UseSerilog((context, services, config) =>
{
    config
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithProperty("Service", "order-api")
        .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter());
});

// OpenTelemetry
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService("order-api", serviceVersion: "1.0.0")
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName
    });

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation(o => o.RecordException = true)
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddSource("OrderService.*")
            .AddOtlpExporter(o =>
            {
                o.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317");
            });
    })
    .WithMetrics(metrics =>
    {
        metrics
            .SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation()
            .AddMeter("OrderService.*")
            .AddPrometheusExporter()
            .AddOtlpExporter(o =>
            {
                o.Endpoint = new Uri(builder.Configuration["Otlp:Endpoint"] ?? "http://localhost:4317");
            });
    });

// Health checks
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: new[] { "live" })
    .AddNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")!, tags: new[] { "ready" })
    .AddRedis(builder.Configuration["Redis:ConnectionString"] ?? "localhost:6379", tags: new[] { "ready" });

var app = builder.Build();

app.UseSerilogRequestLogging(options =>
{
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value ?? "anonymous");
        diagnosticContext.Set("ClientIP", httpContext.Connection.RemoteIpAddress?.ToString());
    };
});

app.MapPrometheusScrapingEndpoint("/metrics");
app.MapHealthChecks("/health/live", new() { Predicate = c => c.Tags.Contains("live") });
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });

app.Run();
```

---

## Exercises / Project Tasks

### Exercise 1: Add Tracing
เพิ่ม distributed tracing:
- Custom spans สำหรับ business operations
- Trace propagation ระหว่าง services
- View traces ใน Jaeger

### Exercise 2: Custom Metrics
สร้าง custom metrics:
- Request count per endpoint
- Error rate by status code
- Business metric (orders per minute)

### Exercise 3: Dashboards
สร้าง Grafana dashboard:
- Request rate
- Error rate (RED metrics)
- Latency percentiles (p50, p95, p99)

### Exercise 4: Alerting
ตั้ง Prometheus alerts:
- Error rate > 5% for 5 minutes
- P99 latency > 1 second
- Memory usage > 80%

---

## สรุป

- **Observability** ช่วยให้เข้าใจ production systems โดยไม่ต้อง modify code
- **OpenTelemetry** เป็น standard ที่ vendor-neutral สำหรับ telemetry
- **Distributed Tracing** ติดตาม request journey ข้าม multiple services
- **Metrics** วัดค่า system performance (request rate, error rate, latency)
- **Structured Logging** ทำให้ search และ analyze logs ได้ง่าย
- **Health Checks** บอก orchestrator ว่า service พร้อมรับ traffic หรือยัง
- **Grafana + Prometheus** คือ stack ยอดนิยมสำหรับ monitoring

---

## Part ถัดไป

**Part 093: Real-world Project: E-commerce API** - สร้าง complete e-commerce API

---

*Part 092/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
