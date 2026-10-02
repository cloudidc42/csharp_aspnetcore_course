# Part 098: Native AOT Compilation

## เนื้อหาใน Part นี้
- AOT vs JIT: ความแตกต่างและ trade-offs
- Native AOT limitations
- Tree Shaking และ Trimming
- ASP.NET Core Minimal API + AOT
- Benchmarks และ deployment size

---

## 1. AOT vs JIT

### JIT (Just-In-Time) - วิธีเดิม
```
Source Code → IL (CIL) → [ Runtime ] → Native Code
                                ↑
                          JIT Compiler รันตอน startup
```

### AOT (Ahead-Of-Time) - วิธีใหม่
```
Source Code → IL (CIL) → [ Build Time ] → Native Binary
                               ↑
                         AOT Compiler รันตอน build
```

### เปรียบเทียบ

| Feature | JIT | AOT |
|---------|-----|-----|
| Startup time | ช้า (warm-up) | เร็วมาก |
| Memory | มากกว่า (JIT metadata) | น้อยกว่า |
| Binary size | เล็ก (IL + runtime) | ใหญ่ขึ้น (รวม runtime) |
| Reflection | เต็มรูปแบบ | จำกัด |
| Dynamic code | รองรับ | ไม่รองรับ |
| Deployment | ต้องการ .NET runtime | self-contained |
| Platform | cross-platform IL | platform-specific |

---

## 2. Enable Native AOT

```xml
<!-- MyApi.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    
    <!-- Enable Native AOT -->
    <PublishAot>true</PublishAot>
    
    <!-- Optional: Trim unused code -->
    <TrimmerRootDescriptor>TrimmerRoots.xml</TrimmerRootDescriptor>
    
    <!-- Optimize for size vs speed -->
    <OptimizationPreference>Speed</OptimizationPreference>
    <!-- หรือ -->
    <!-- <OptimizationPreference>Size</OptimizationPreference> -->
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.*" />
  </ItemGroup>
</Project>
```

```bash
# Build as Native AOT
dotnet publish -r linux-x64 -c Release

# หรือ Windows
dotnet publish -r win-x64 -c Release

# ผลลัพธ์: self-contained executable ไม่ต้องการ .NET runtime
```

---

## 3. Minimal API ที่ Compatible กับ AOT

```csharp
// Program.cs - AOT-compatible Minimal API
using System.Text.Json.Serialization;

var builder = WebApplication.CreateSlimBuilder(args);  // "Slim" = AOT-friendly

// ใช้ Source Generated JSON Serialization (ไม่ใช้ reflection)
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

var app = builder.Build();

var products = new[]
{
    new Product(Guid.NewGuid(), "Laptop", 25000m),
    new Product(Guid.NewGuid(), "Mouse", 599m),
    new Product(Guid.NewGuid(), "Keyboard", 1299m)
};

app.MapGet("/products", () => products);
app.MapGet("/products/{id:guid}", (Guid id) =>
    products.FirstOrDefault(p => p.Id == id) is { } product
        ? Results.Ok(product)
        : Results.NotFound());

app.MapPost("/products", (Product product) =>
    Results.Created($"/products/{product.Id}", product));

app.Run();

public record Product(Guid Id, string Name, decimal Price);

// AOT ต้องการ JsonSerializerContext แทน reflection
[JsonSerializable(typeof(Product))]
[JsonSerializable(typeof(Product[]))]
[JsonSerializable(typeof(IEnumerable<Product>))]
internal partial class AppJsonSerializerContext : JsonSerializerContext { }
```

---

## 4. AOT Limitations

```csharp
// สิ่งที่ทำไม่ได้ใน AOT:

// 1. Dynamic Type Loading
// Assembly.LoadFrom("plugin.dll");  // ❌ ไม่รองรับ

// 2. Full Reflection (บางอย่างยังทำได้)
// var type = Type.GetType("MyApp.SomeClass");  // ❌ อาจ trim ออก
// Activator.CreateInstance("MyApp.SomeClass");  // ❌

// 3. Dynamic Code Generation
// Expression trees ที่ Compile() → ❌
// Emit IL ใน runtime → ❌

// 4. Unsafe Reflection
// typeof(SomeClass).GetMethod("PrivateMethod").Invoke(obj, args);  // ❌ อาจล้มเหลว

// สิ่งที่ยังทำได้:
// typeof(Product).GetProperties()  // ✅ ถ้าไม่ถูก trim
// [DynamicallyAccessedMembers] attribute บอก trimmer ไม่ให้ trim

// วิธีแก้ปัญหา: ใช้ DynamicallyAccessedMembers
public static void PrintProperties<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicProperties)] T>(T obj)
{
    foreach (var prop in typeof(T).GetProperties())
    {
        Console.WriteLine($"{prop.Name}: {prop.GetValue(obj)}");
    }
}

// หรือใช้ Source Generator แทน reflection
// ดีกว่า: ไม่ต้องการ reflection เลย
public static void PrintProperties(Product product)
{
    Console.WriteLine($"Id: {product.Id}");
    Console.WriteLine($"Name: {product.Name}");
    Console.WriteLine($"Price: {product.Price}");
}
```

---

## 5. Trimming Configuration

```csharp
// TrimmerRoots.xml - บอก trimmer อย่า trim types เหล่านี้
```

```xml
<linker>
  <assembly fullname="MyApi">
    <!-- Keep specific types -->
    <type fullname="MyApi.Models.*" preserve="all" />
    
    <!-- Keep specific methods -->
    <type fullname="MyApi.Services.ProductService">
      <method signature="System.Threading.Tasks.Task GetAllAsync()" />
    </type>
  </assembly>
</linker>
```

```csharp
// ใน code: ใช้ attributes เพื่อ preserve types
[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.All)]
public class ReflectiveService
{
    // trimmer จะเก็บ members ทั้งหมดของ class นี้ไว้
}

// RequiresUnreferencedCode - warn ว่า code นี้ไม่ compatible กับ trimming
[RequiresUnreferencedCode("This method uses reflection")]
public object CreateInstance(string typeName)
{
    var type = Type.GetType(typeName);
    return Activator.CreateInstance(type!)!;
}

// RequiresDynamicCode - warn ว่าต้องการ dynamic code generation
[RequiresDynamicCode("Compiles expressions at runtime")]
public Func<T, TResult> Compile<T, TResult>(Expression<Func<T, TResult>> expr)
{
    return expr.Compile();
}
```

---

## 6. JSON Serialization ใน AOT

```csharp
// ❌ ไม่ใช้ใน AOT: reflection-based serialization
var json = JsonSerializer.Serialize(product);  // อาจล้มเหลวหรือ trim incorrectly

// ✅ ใช้ใน AOT: source-generated context
var json = JsonSerializer.Serialize(product, AppJsonSerializerContext.Default.Product);
var product = JsonSerializer.Deserialize(json, AppJsonSerializerContext.Default.Product);

// รองรับ collection types
[JsonSerializable(typeof(List<Product>))]
[JsonSerializable(typeof(PagedResult<Product>))]
[JsonSerializable(typeof(ApiResponse<Product>))]
// รองรับ nested types
[JsonSerializable(typeof(OrderDto))]
[JsonSerializable(typeof(OrderItemDto))]
// รองรับ primitives (มักไม่จำเป็นต้องเพิ่ม)
[JsonSerializable(typeof(int))]
[JsonSerializable(typeof(string))]
internal partial class AppJsonSerializerContext : JsonSerializerContext { }

// DTO types ต้อง compatible กับ AOT
public record OrderDto
{
    public Guid Id { get; init; }
    public string OrderNumber { get; init; } = string.Empty;
    public List<OrderItemDto> Items { get; init; } = [];
    public decimal Total { get; init; }
    public string Status { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; }
}

public record OrderItemDto
{
    public Guid ProductId { get; init; }
    public string ProductName { get; init; } = string.Empty;
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

// ใน endpoint
app.MapGet("/orders/{id:guid}", (Guid id, IOrderService svc) =>
{
    var order = svc.GetById(id);
    // ASP.NET Core ใช้ Context โดยอัตโนมัติถ้า configure ไว้แล้ว
    return order == null ? Results.NotFound() : Results.Ok(order);
});
```

---

## 7. Complete AOT-Compatible API

```csharp
// Program.cs
using System.Text.Json.Serialization;
using Microsoft.AspNetCore.Http.HttpResults;

var builder = WebApplication.CreateSlimBuilder(args);

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonSerializerContext.Default);
});

// Dependency injection ทำงานได้ปกติใน AOT
builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();

var app = builder.Build();

// Health check endpoint
app.MapGet("/health", () => Results.Ok(new { status = "healthy", timestamp = DateTime.UtcNow }));

// Products endpoints
var productsApi = app.MapGroup("/api/products");

productsApi.MapGet("/", async (IProductService svc) =>
{
    var products = await svc.GetAllAsync();
    return Results.Ok(products);
});

productsApi.MapGet("/{id:guid}", async Task<Results<Ok<ProductDto>, NotFound>>
    (Guid id, IProductService svc) =>
{
    var product = await svc.GetByIdAsync(id);
    return product is null ? TypedResults.NotFound() : TypedResults.Ok(product);
});

productsApi.MapPost("/", async Task<Results<Created<ProductDto>, ValidationProblem>>
    (CreateProductRequest request, IProductService svc) =>
{
    // Manual validation (ไม่ใช้ DataAnnotations validation ที่ต้องการ reflection)
    var errors = new Dictionary<string, string[]>();
    if (string.IsNullOrWhiteSpace(request.Name))
        errors["name"] = ["Name is required"];
    if (request.Price <= 0)
        errors["price"] = ["Price must be positive"];
    
    if (errors.Any())
        return TypedResults.ValidationProblem(errors);
    
    var product = await svc.CreateAsync(request);
    return TypedResults.Created($"/api/products/{product.Id}", product);
});

productsApi.MapDelete("/{id:guid}", async Task<Results<NoContent, NotFound>>
    (Guid id, IProductService svc) =>
{
    var deleted = await svc.DeleteAsync(id);
    return deleted ? TypedResults.NoContent() : TypedResults.NotFound();
});

app.Run();

// DTOs
public record ProductDto(Guid Id, string Name, decimal Price, int Stock, bool IsActive);
public record CreateProductRequest(string Name, decimal Price, int InitialStock = 0);

// Service interface ปกติ
public interface IProductService
{
    Task<IReadOnlyList<ProductDto>> GetAllAsync();
    Task<ProductDto?> GetByIdAsync(Guid id);
    Task<ProductDto> CreateAsync(CreateProductRequest request);
    Task<bool> DeleteAsync(Guid id);
}

// In-memory implementation for demo
public class InMemoryProductRepository : IProductRepository
{
    private readonly List<ProductDto> _products =
    [
        new(Guid.NewGuid(), "Laptop Pro", 35000m, 10, true),
        new(Guid.NewGuid(), "Wireless Mouse", 899m, 50, true),
        new(Guid.NewGuid(), "USB-C Hub", 1299m, 30, true)
    ];

    public Task<IReadOnlyList<ProductDto>> GetAllAsync() =>
        Task.FromResult<IReadOnlyList<ProductDto>>(_products.AsReadOnly());

    public Task<ProductDto?> GetByIdAsync(Guid id) =>
        Task.FromResult(_products.FirstOrDefault(p => p.Id == id));

    public Task<ProductDto> AddAsync(ProductDto product)
    {
        _products.Add(product);
        return Task.FromResult(product);
    }

    public Task<bool> RemoveAsync(Guid id)
    {
        var item = _products.FirstOrDefault(p => p.Id == id);
        if (item == null) return Task.FromResult(false);
        _products.Remove(item);
        return Task.FromResult(true);
    }
}

// JSON Context - ต้องครอบคลุม types ทั้งหมดที่ serialize/deserialize
[JsonSerializable(typeof(ProductDto))]
[JsonSerializable(typeof(List<ProductDto>))]
[JsonSerializable(typeof(IReadOnlyList<ProductDto>))]
[JsonSerializable(typeof(CreateProductRequest))]
[JsonSerializable(typeof(object))]  // for health check
internal partial class AppJsonSerializerContext : JsonSerializerContext { }
```

---

## 8. Docker กับ Native AOT

```dockerfile
# Dockerfile สำหรับ Native AOT
# Multi-stage build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build

# Install AOT prerequisites for Linux
RUN apt-get update && apt-get install -y clang zlib1g-dev

WORKDIR /src
COPY ["MyApi.csproj", "."]
RUN dotnet restore

COPY . .
# Build as Native AOT targeting Linux x64
RUN dotnet publish -r linux-x64 -c Release -o /app/publish \
    --self-contained true

# Final image - ไม่ต้องการ .NET runtime!
FROM mcr.microsoft.com/dotnet/runtime-deps:9.0
WORKDIR /app
COPY --from=build /app/publish .

# Binary เดียวที่รัน app ทั้งหมด
ENTRYPOINT ["./MyApi"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_ENVIRONMENT=Production
```

---

## 9. Benchmarks

```
=== Startup Time ===
JIT Mode:    450ms cold start
AOT Mode:    25ms cold start  (18x faster!)

=== Memory Usage (idle) ===
JIT Mode:    ~80MB
AOT Mode:    ~15MB  (5x less!)

=== Binary Size ===
JIT Mode:    .dll = 512KB + .NET Runtime = 180MB (shared)
AOT Mode:    executable = 12MB (self-contained, no runtime needed)

=== Request Throughput (after warmup) ===
JIT Mode:    ~95,000 req/s
AOT Mode:    ~92,000 req/s  (comparable after warmup)

=== Cold Start (เช่น serverless) ===
JIT Mode:    450ms
AOT Mode:    25ms  → ประหยัด cost ใน pay-per-use serverless
```

---

## Exercises / Project Tasks

### Exercise 1: Port REST API to AOT
นำ API ที่สร้างก่อนหน้าแปลงเป็น AOT-compatible:
1. เปลี่ยนเป็น `CreateSlimBuilder`
2. เพิ่ม `JsonSerializerContext` สำหรับทุก DTO
3. แก้ warning ที่เกี่ยวกับ trimming
4. Build และทดสอบ

### Exercise 2: Serverless Function
สร้าง AWS Lambda function ด้วย Native AOT:
- Cold start time < 50ms
- Memory <= 128MB
- ลด cost เมื่อเทียบกับ JIT

### Exercise 3: Benchmark Comparison
วัด performance:
- Startup time: JIT vs AOT
- Memory: JIT vs AOT
- Throughput: JIT vs AOT ด้วย BenchmarkDotNet

### Exercise 4: gRPC + AOT
สร้าง gRPC service ที่ AOT-compatible:
- Protobuf serialization รองรับ AOT โดยธรรมชาติ
- ทดสอบ all 4 streaming types

---

## สรุป

| ใช้ AOT เมื่อ | ใช้ JIT เมื่อ |
|--------------|--------------|
| Startup time สำคัญ (serverless, containers) | ใช้ reflection มาก |
| Memory จำกัด | ใช้ dynamic code generation |
| Self-contained deployment | Library compatibility ไม่ชัดเจน |
| Microservices scale-out บ่อย | Development iteration เร็ว |

- **AOT เหมาะมาก** สำหรับ serverless, microservices ที่ scale down/up บ่อย
- **JIT เหมาะกว่า** สำหรับ monolith ที่มี complex reflection/dynamic code
- ใช้ **Source Generators** แทน reflection เพื่อ AOT compatibility

---

## Part ถัดไป

**Part 099: Interview Preparation & Career Guide** - เตรียมตัวสัมภาษณ์งาน C#/.NET

---

*Part 098/100 | Phase 7/7: ระดับโลก | หลักสูตร C# และ ASP.NET Core*
