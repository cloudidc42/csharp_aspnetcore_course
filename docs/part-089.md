# Part 089: gRPC ใน .NET

## เนื้อหาใน Part นี้
- gRPC vs REST
- Protocol Buffers
- Service definition
- Streaming (unary, server, client, bidirectional)
- gRPC-Web
- โปรแกรมตัวอย่าง: gRPC product service

---

## 1. gRPC vs REST

gRPC เป็น modern high-performance RPC framework จาก Google

### เปรียบเทียบ

```
REST:
- Protocol: HTTP/1.1 หรือ HTTP/2
- Format: JSON/XML (text)
- Schema: Optional (OpenAPI/Swagger)
- Streaming: Limited (SSE, WebSocket แยกต่างหาก)
- Language: Any
- Use case: Public APIs, web browsers

gRPC:
- Protocol: HTTP/2 เท่านั้น
- Format: Protocol Buffers (binary)
- Schema: Required (.proto file)
- Streaming: Built-in (4 types)
- Language: Code generated for any language
- Use case: Microservices communication, high-performance
```

**ข้อดีของ gRPC:**
- เร็วกว่า REST 5-10x (binary protocol)
- Strongly typed
- Code generation
- Built-in streaming
- HTTP/2 multiplexing

**ข้อเสียของ gRPC:**
- ไม่ friendly กับ browser โดยตรง
- Debugging ยากกว่า (binary)
- ต้อง define schema ก่อน

---

## 2. Protocol Buffers

Protocol Buffers (protobuf) เป็น language สำหรับ define data structures และ services

### .proto file syntax

```protobuf
// product.proto
syntax = "proto3";

option csharp_namespace = "ProductService.Protos";

package product;

import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

// Service definition
service ProductService {
  // Unary RPC
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc CreateProduct (CreateProductRequest) returns (ProductResponse);
  rpc UpdateProduct (UpdateProductRequest) returns (ProductResponse);
  rpc DeleteProduct (DeleteProductRequest) returns (DeleteProductResponse);
  
  // Server streaming
  rpc ListProducts (ListProductsRequest) returns (stream ProductResponse);
  
  // Client streaming
  rpc BulkCreateProducts (stream CreateProductRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming
  rpc SyncProducts (stream SyncRequest) returns (stream SyncResponse);
}

// Messages
message GetProductRequest {
  string id = 1;
}

message CreateProductRequest {
  string name = 1;
  string description = 2;
  double price = 3;
  string category = 4;
  int32 stock = 5;
}

message UpdateProductRequest {
  string id = 1;
  google.protobuf.StringValue name = 2;     // Optional field
  google.protobuf.StringValue description = 3;
  google.protobuf.DoubleValue price = 4;
  google.protobuf.Int32Value stock = 5;
}

message DeleteProductRequest {
  string id = 1;
}

message DeleteProductResponse {
  bool success = 1;
  string message = 2;
}

message ProductResponse {
  string id = 1;
  string name = 2;
  string description = 3;
  double price = 4;
  string category = 5;
  int32 stock = 6;
  bool is_active = 7;
  google.protobuf.Timestamp created_at = 8;
  google.protobuf.Timestamp updated_at = 9;
}

message ListProductsRequest {
  string category = 1;
  int32 page = 2;
  int32 page_size = 3;
  string sort_by = 4;
  bool sort_descending = 5;
}

message BulkCreateResponse {
  int32 created_count = 1;
  int32 failed_count = 2;
  repeated string errors = 3;
}

message SyncRequest {
  string product_id = 1;
  oneof action {
    CreateProductRequest create = 2;
    UpdateProductRequest update = 3;
    DeleteProductRequest delete = 4;
  }
}

message SyncResponse {
  string product_id = 1;
  bool success = 2;
  string error = 3;
}
```

---

## 3. Server Implementation

### การตั้งค่าโปรเจกต์

```bash
dotnet new web -n ProductGrpcService
cd ProductGrpcService
dotnet add package Grpc.AspNetCore
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools
```

```xml
<!-- ProductGrpcService.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <!-- กำหนด .proto files -->
    <Protobuf Include="Protos\product.proto" GrpcServices="Server" />
    <Protobuf Include="Protos\inventory.proto" GrpcServices="Server" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.60.0" />
    <PackageReference Include="Google.Protobuf" Version="3.26.0" />
    <PackageReference Include="Grpc.Tools" Version="2.60.0" PrivateAssets="All" />
  </ItemGroup>
</Project>
```

### Service Implementation

```csharp
// Services/ProductGrpcService.cs
using Grpc.Core;
using ProductService.Protos;
using Google.Protobuf.WellKnownTypes;

public class ProductGrpcService : ProductService.ProductServiceBase
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductGrpcService> _logger;

    public ProductGrpcService(
        IProductRepository repository,
        ILogger<ProductGrpcService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    // ─── Unary RPC ────────────────────────────────────────────

    public override async Task<ProductResponse> GetProduct(
        GetProductRequest request, 
        ServerCallContext context)
    {
        _logger.LogInformation("GetProduct: {Id}", request.Id);
        
        if (!Guid.TryParse(request.Id, out var id))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, 
                $"Invalid product ID: {request.Id}"));
        }
        
        var product = await _repository.GetByIdAsync(id, context.CancellationToken);
        
        if (product == null)
        {
            throw new RpcException(new Status(
                StatusCode.NotFound, 
                $"Product {request.Id} not found"));
        }
        
        return MapToResponse(product);
    }

    public override async Task<ProductResponse> CreateProduct(
        CreateProductRequest request,
        ServerCallContext context)
    {
        // Validate
        if (string.IsNullOrWhiteSpace(request.Name))
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, 
                "Product name is required"));
        }
        
        if (request.Price <= 0)
        {
            throw new RpcException(new Status(
                StatusCode.InvalidArgument, 
                "Price must be positive"));
        }

        var product = await _repository.CreateAsync(new CreateProductDto
        {
            Name = request.Name,
            Description = request.Description,
            Price = (decimal)request.Price,
            Category = request.Category,
            Stock = request.Stock
        }, context.CancellationToken);

        _logger.LogInformation("Created product: {Id}", product.Id);
        return MapToResponse(product);
    }

    public override async Task<ProductResponse> UpdateProduct(
        UpdateProductRequest request,
        ServerCallContext context)
    {
        if (!Guid.TryParse(request.Id, out var id))
        {
            throw new RpcException(new Status(StatusCode.InvalidArgument, "Invalid ID"));
        }

        var updateDto = new UpdateProductDto
        {
            Name = request.Name?.Value,
            Description = request.Description?.Value,
            Price = request.Price.HasValue ? (decimal?)request.Price.Value : null,
            Stock = request.Stock?.Value
        };

        var product = await _repository.UpdateAsync(id, updateDto, context.CancellationToken);
        
        if (product == null)
        {
            throw new RpcException(new Status(StatusCode.NotFound, $"Product {request.Id} not found"));
        }

        return MapToResponse(product);
    }

    // ─── Server Streaming RPC ────────────────────────────────

    public override async Task ListProducts(
        ListProductsRequest request,
        IServerStreamWriter<ProductResponse> responseStream,
        ServerCallContext context)
    {
        _logger.LogInformation("Streaming products - category: {Category}", request.Category);
        
        var products = _repository.GetProductsStreamAsync(
            request.Category,
            request.Page,
            request.PageSize,
            context.CancellationToken);
        
        await foreach (var product in products)
        {
            if (context.CancellationToken.IsCancellationRequested)
                break;
            
            await responseStream.WriteAsync(MapToResponse(product));
            
            // Simulate processing delay
            await Task.Delay(10, context.CancellationToken);
        }
    }

    // ─── Client Streaming RPC ────────────────────────────────

    public override async Task<BulkCreateResponse> BulkCreateProducts(
        IAsyncStreamReader<CreateProductRequest> requestStream,
        ServerCallContext context)
    {
        var created = 0;
        var failed = 0;
        var errors = new List<string>();

        await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
        {
            try
            {
                await _repository.CreateAsync(new CreateProductDto
                {
                    Name = request.Name,
                    Description = request.Description,
                    Price = (decimal)request.Price,
                    Category = request.Category,
                    Stock = request.Stock
                }, context.CancellationToken);
                
                created++;
            }
            catch (Exception ex)
            {
                failed++;
                errors.Add($"Failed to create '{request.Name}': {ex.Message}");
                _logger.LogError(ex, "Failed to create product: {Name}", request.Name);
            }
        }

        return new BulkCreateResponse
        {
            CreatedCount = created,
            FailedCount = failed,
            Errors = { errors }
        };
    }

    // ─── Bidirectional Streaming RPC ─────────────────────────

    public override async Task SyncProducts(
        IAsyncStreamReader<SyncRequest> requestStream,
        IServerStreamWriter<SyncResponse> responseStream,
        ServerCallContext context)
    {
        await foreach (var request in requestStream.ReadAllAsync(context.CancellationToken))
        {
            SyncResponse response;
            
            try
            {
                switch (request.ActionCase)
                {
                    case SyncRequest.ActionOneofCase.Create:
                        await _repository.CreateAsync(
                            MapCreateDto(request.Create), 
                            context.CancellationToken);
                        response = new SyncResponse 
                        { 
                            ProductId = request.ProductId, 
                            Success = true 
                        };
                        break;
                    
                    case SyncRequest.ActionOneofCase.Update:
                        await _repository.UpdateAsync(
                            Guid.Parse(request.ProductId),
                            MapUpdateDto(request.Update),
                            context.CancellationToken);
                        response = new SyncResponse 
                        { 
                            ProductId = request.ProductId, 
                            Success = true 
                        };
                        break;
                    
                    case SyncRequest.ActionOneofCase.Delete:
                        await _repository.DeleteAsync(
                            Guid.Parse(request.ProductId),
                            context.CancellationToken);
                        response = new SyncResponse 
                        { 
                            ProductId = request.ProductId, 
                            Success = true 
                        };
                        break;
                    
                    default:
                        response = new SyncResponse
                        {
                            ProductId = request.ProductId,
                            Success = false,
                            Error = "Unknown action"
                        };
                        break;
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Sync failed for product: {Id}", request.ProductId);
                response = new SyncResponse
                {
                    ProductId = request.ProductId,
                    Success = false,
                    Error = ex.Message
                };
            }
            
            await responseStream.WriteAsync(response);
        }
    }

    // ─── Helpers ─────────────────────────────────────────────

    private static ProductResponse MapToResponse(Product product) => new()
    {
        Id = product.Id.ToString(),
        Name = product.Name,
        Description = product.Description ?? "",
        Price = (double)product.Price,
        Category = product.Category,
        Stock = product.Stock,
        IsActive = product.IsActive,
        CreatedAt = Timestamp.FromDateTime(product.CreatedAt.ToUniversalTime()),
        UpdatedAt = product.UpdatedAt.HasValue
            ? Timestamp.FromDateTime(product.UpdatedAt.Value.ToUniversalTime())
            : null
    };
    
    private static CreateProductDto MapCreateDto(CreateProductRequest req) => new()
    {
        Name = req.Name,
        Description = req.Description,
        Price = (decimal)req.Price,
        Category = req.Category,
        Stock = req.Stock
    };
    
    private static UpdateProductDto MapUpdateDto(UpdateProductRequest req) => new()
    {
        Name = req.Name?.Value,
        Price = req.Price.HasValue ? (decimal?)req.Price.Value : null,
        Stock = req.Stock?.Value
    };
}
```

### Program.cs

```csharp
// Program.cs
using Microsoft.AspNetCore.Server.Kestrel.Core;

var builder = WebApplication.CreateBuilder(args);

// Add gRPC
builder.Services.AddGrpc(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaxReceiveMessageSize = 5 * 1024 * 1024;  // 5MB
    options.MaxSendMessageSize = 5 * 1024 * 1024;
});

// gRPC reflection (สำหรับ tooling เช่น grpcurl, Postman)
builder.Services.AddGrpcReflection();

// Services
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddScoped<IProductRepository, ProductRepository>();

// Kestrel - HTTP/2 สำหรับ gRPC
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http2;  // gRPC requires HTTP/2
    });
    
    options.ListenAnyIP(5000, listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;  // REST + gRPC-Web
    });
});

var app = builder.Build();

// Map gRPC services
app.MapGrpcService<ProductGrpcService>();

// gRPC reflection ใน development
if (app.Environment.IsDevelopment())
{
    app.MapGrpcReflectionService();
}

app.Run();
```

---

## 4. Client Implementation

```bash
# Client project
dotnet new console -n ProductClient
dotnet add package Grpc.Net.Client
dotnet add package Google.Protobuf
dotnet add package Grpc.Tools
```

```csharp
// Client/Program.cs
using Grpc.Net.Client;
using Grpc.Core;
using ProductService.Protos;

// Create channel
using var channel = GrpcChannel.ForAddress("https://localhost:5001", new GrpcChannelOptions
{
    MaxReceiveMessageSize = 5 * 1024 * 1024
});

var client = new ProductService.ProductServiceClient(channel);

// Unary call
Console.WriteLine("=== Unary Call ===");
try
{
    var product = await client.GetProductAsync(new GetProductRequest
    {
        Id = "some-product-id"
    });
    Console.WriteLine($"Product: {product.Name} - ${product.Price}");
}
catch (RpcException ex) when (ex.StatusCode == StatusCode.NotFound)
{
    Console.WriteLine($"Product not found: {ex.Status.Detail}");
}

// Create product
Console.WriteLine("\n=== Create Product ===");
var newProduct = await client.CreateProductAsync(new CreateProductRequest
{
    Name = "Test Product",
    Description = "This is a test product",
    Price = 99.99,
    Category = "Electronics",
    Stock = 100
});
Console.WriteLine($"Created: {newProduct.Id} - {newProduct.Name}");

// Server streaming
Console.WriteLine("\n=== Server Streaming ===");
using var listCall = client.ListProducts(new ListProductsRequest
{
    Category = "Electronics",
    Page = 1,
    PageSize = 10
});

await foreach (var p in listCall.ResponseStream.ReadAllAsync())
{
    Console.WriteLine($"  - {p.Name}: ${p.Price} (stock: {p.Stock})");
}

// Client streaming (bulk create)
Console.WriteLine("\n=== Client Streaming (Bulk Create) ===");
using var bulkCall = client.BulkCreateProducts();

for (int i = 0; i < 10; i++)
{
    await bulkCall.RequestStream.WriteAsync(new CreateProductRequest
    {
        Name = $"Bulk Product {i + 1}",
        Price = 10.00 + i,
        Category = "Test",
        Stock = 50
    });
    
    Console.WriteLine($"Sent product {i + 1}/10");
}

await bulkCall.RequestStream.CompleteAsync();
var bulkResult = await bulkCall.ResponseAsync;
Console.WriteLine($"Bulk created: {bulkResult.CreatedCount} success, {bulkResult.FailedCount} failed");

// Bidirectional streaming
Console.WriteLine("\n=== Bidirectional Streaming ===");
using var syncCall = client.SyncProducts();

var receiveTask = Task.Run(async () =>
{
    await foreach (var response in syncCall.ResponseStream.ReadAllAsync())
    {
        Console.WriteLine($"Sync result for {response.ProductId}: {(response.Success ? "OK" : response.Error)}");
    }
});

// Send multiple operations
await syncCall.RequestStream.WriteAsync(new SyncRequest
{
    ProductId = Guid.NewGuid().ToString(),
    Create = new CreateProductRequest { Name = "Sync Product 1", Price = 50.0, Category = "Test", Stock = 10 }
});

await syncCall.RequestStream.WriteAsync(new SyncRequest
{
    ProductId = newProduct.Id,
    Update = new UpdateProductRequest
    {
        Id = newProduct.Id,
        Price = Google.Protobuf.WellKnownTypes.DoubleValue.Of(199.99)
    }
});

await syncCall.RequestStream.CompleteAsync();
await receiveTask;

Console.WriteLine("Done!");
```

### Typed gRPC Client ใน ASP.NET Core

```csharp
// Registration ใน another service
builder.Services.AddGrpcClient<ProductService.ProductServiceClient>(options =>
{
    options.Address = new Uri(builder.Configuration["Services:ProductService:GrpcUrl"]!);
})
.ConfigureChannel(options =>
{
    options.MaxReceiveMessageSize = 5 * 1024 * 1024;
})
.AddCallCredentials((context, metadata) =>
{
    // Add authentication
    metadata.Add("Authorization", $"Bearer {GetToken()}");
    return Task.CompletedTask;
})
.AddRetryPolicy(new MethodConfig
{
    Names = { MethodName.Default },
    RetryPolicy = new RetryPolicy
    {
        MaxAttempts = 3,
        InitialBackoff = TimeSpan.FromMilliseconds(100),
        MaxBackoff = TimeSpan.FromSeconds(5),
        BackoffMultiplier = 2,
        RetryableStatusCodes = { StatusCode.Unavailable, StatusCode.DeadlineExceeded }
    }
});
```

---

## 5. Interceptors

```csharp
// Logging Interceptor (Server side)
public class LoggingInterceptor : Interceptor
{
    private readonly ILogger<LoggingInterceptor> _logger;

    public LoggingInterceptor(ILogger<LoggingInterceptor> logger)
    {
        _logger = logger;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var method = context.Method;
        var stopwatch = Stopwatch.StartNew();
        
        _logger.LogInformation("gRPC {Method} started", method);
        
        try
        {
            var response = await continuation(request, context);
            stopwatch.Stop();
            
            _logger.LogInformation("gRPC {Method} completed in {Elapsed}ms", 
                method, stopwatch.ElapsedMilliseconds);
            
            return response;
        }
        catch (RpcException ex)
        {
            stopwatch.Stop();
            _logger.LogError("gRPC {Method} failed: {Status} in {Elapsed}ms", 
                method, ex.Status, stopwatch.ElapsedMilliseconds);
            throw;
        }
    }
}

// Auth Interceptor (Client side)
public class AuthInterceptor : Interceptor
{
    private readonly ITokenProvider _tokenProvider;

    public AuthInterceptor(ITokenProvider tokenProvider)
    {
        _tokenProvider = tokenProvider;
    }

    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request,
        ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var token = _tokenProvider.GetToken();
        
        var newContext = context.WithCallOptions(
            context.Options.WithHeaders(
                new Metadata { { "Authorization", $"Bearer {token}" } }));
        
        return continuation(request, newContext);
    }
}

// Registration
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<LoggingInterceptor>();
});
```

---

## 6. gRPC-Web (Browser Support)

```csharp
// Program.cs - เพิ่ม gRPC-Web support
builder.Services.AddGrpc();
builder.Services.AddGrpcWeb(options => options.DefaultEnabled = true);

app.UseGrpcWeb();  // สำหรับ gRPC-Web
app.MapGrpcService<ProductGrpcService>().EnableGrpcWeb();
```

```typescript
// TypeScript/JavaScript client
import { GrpcWebFetchTransport } from "@protobuf-ts/grpcweb-transport";
import { ProductServiceClient } from "./generated/product.client";
import { GetProductRequest } from "./generated/product";

const transport = new GrpcWebFetchTransport({
    baseUrl: "https://api.myapp.com"
});

const client = new ProductServiceClient(transport);

// Unary call
const { response } = await client.getProduct({ id: "product-id" });
console.log(response.name);

// Server streaming
const stream = client.listProducts({ category: "Electronics", page: 1, pageSize: 10 });
for await (const product of stream.responses) {
    console.log(product.name);
}
```

---

## 7. โปรแกรมตัวอย่าง: gRPC Product Service (สมบูรณ์)

### โครงสร้างโปรเจกต์

```
ProductGrpcService/
├── Protos/
│   ├── product.proto
│   └── health.proto
├── Services/
│   ├── ProductGrpcService.cs
│   └── HealthGrpcService.cs
├── Data/
│   ├── AppDbContext.cs
│   └── ProductRepository.cs
├── Interceptors/
│   ├── LoggingInterceptor.cs
│   └── ValidationInterceptor.cs
├── Models/
│   ├── Product.cs
│   └── DTOs.cs
└── Program.cs
```

### health.proto

```protobuf
// Protos/health.proto
syntax = "proto3";

option csharp_namespace = "ProductService.Protos";

package health;

service HealthService {
  rpc Check (HealthCheckRequest) returns (HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}
```

### Repository

```csharp
// Data/ProductRepository.cs
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default);
    IAsyncEnumerable<Product> GetProductsStreamAsync(string category, int page, int pageSize, CancellationToken ct = default);
    Task<Product> CreateAsync(CreateProductDto dto, CancellationToken ct = default);
    Task<Product?> UpdateAsync(Guid id, UpdateProductDto dto, CancellationToken ct = default);
    Task<bool> DeleteAsync(Guid id, CancellationToken ct = default);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;

    public ProductRepository(AppDbContext db)
    {
        _db = db;
    }

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await _db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
    }

    public async IAsyncEnumerable<Product> GetProductsStreamAsync(
        string category, int page, int pageSize,
        [System.Runtime.CompilerServices.EnumeratorCancellation] CancellationToken ct)
    {
        var query = _db.Products
            .AsNoTracking()
            .Where(p => p.IsActive);
        
        if (!string.IsNullOrEmpty(category))
            query = query.Where(p => p.Category == category);
        
        query = query
            .OrderBy(p => p.Name)
            .Skip((page - 1) * pageSize)
            .Take(pageSize);
        
        // Stream ทีละรายการ
        await foreach (var product in query.AsAsyncEnumerable().WithCancellation(ct))
        {
            yield return product;
        }
    }

    public async Task<Product> CreateAsync(CreateProductDto dto, CancellationToken ct)
    {
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = dto.Name,
            Description = dto.Description,
            Price = dto.Price,
            Category = dto.Category,
            Stock = dto.Stock,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };
        
        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
        return product;
    }

    public async Task<Product?> UpdateAsync(Guid id, UpdateProductDto dto, CancellationToken ct)
    {
        var product = await _db.Products.FindAsync(new object[] { id }, ct);
        if (product == null) return null;
        
        if (dto.Name != null) product.Name = dto.Name;
        if (dto.Description != null) product.Description = dto.Description;
        if (dto.Price.HasValue) product.Price = dto.Price.Value;
        if (dto.Stock.HasValue) product.Stock = dto.Stock.Value;
        product.UpdatedAt = DateTime.UtcNow;
        
        await _db.SaveChangesAsync(ct);
        return product;
    }

    public async Task<bool> DeleteAsync(Guid id, CancellationToken ct)
    {
        var product = await _db.Products.FindAsync(new object[] { id }, ct);
        if (product == null) return false;
        
        _db.Products.Remove(product);
        await _db.SaveChangesAsync(ct);
        return true;
    }
}
```

---

## Exercises / Project Tasks

### Exercise 1: Basic gRPC Service
สร้าง gRPC service สำหรับ User management:
- GetUser, CreateUser, UpdateUser
- ListUsers (server streaming)
- Service definition ใน .proto

### Exercise 2: Client Streaming
สร้าง log aggregation service:
- Client ส่ง log entries เป็น stream
- Server รวบรวมและ return summary

### Exercise 3: Bidirectional Streaming
สร้าง real-time chat service:
- Client ส่ง messages
- Server broadcast ไปทุก connected clients

### Exercise 4: Interceptors
เพิ่ม interceptors:
- Logging interceptor (method, duration, status)
- Auth interceptor (validate JWT)
- Retry interceptor (client-side)

---

## สรุป

- **gRPC** เร็วกว่า REST เพราะใช้ binary protocol (Protocol Buffers) บน HTTP/2
- **Protocol Buffers** บังคับ schema ทำให้ type-safe และ generate code ได้
- **4 streaming types**: Unary, Server streaming, Client streaming, Bidirectional
- **Interceptors** เหมือน middleware ของ gRPC สำหรับ cross-cutting concerns
- **gRPC-Web** ช่วยให้ browser ใช้ gRPC ได้ (ผ่าน proxy layer)
- **Code generation** ลด boilerplate และ ensure consistency ระหว่าง client/server

---

## Part ถัดไป

**Part 090: GraphQL ใน .NET** - เรียนรู้การสร้าง flexible APIs ด้วย GraphQL และ Hot Chocolate

---

*Part 089/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
