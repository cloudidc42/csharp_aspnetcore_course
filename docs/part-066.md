# Part 066: API Documentation กับ Swagger

## เนื้อหาใน Part นี้
- Swashbuckle และ OpenAPI
- XML Comments สำหรับ documentation
- API Versioning
- Authentication ใน Swagger UI
- Custom Filters
- โปรแกรมตัวอย่าง: Documented API

---

## 1. Swashbuckle และ OpenAPI

**Swashbuckle** เป็น library ที่ generate OpenAPI/Swagger documentation จาก ASP.NET Core controllers

### การติดตั้ง

```bash
dotnet add package Swashbuckle.AspNetCore
# หรือใน .NET 9 มี Microsoft.AspNetCore.OpenApi มาพร้อมแล้ว
dotnet add package Microsoft.AspNetCore.OpenApi
```

### การตั้งค่าพื้นฐาน

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "API documentation สำหรับระบบ My App",
        TermsOfService = new Uri("https://example.com/terms"),
        Contact = new OpenApiContact
        {
            Name = "Support Team",
            Email = "support@example.com",
            Url = new Uri("https://example.com/contact")
        },
        License = new OpenApiLicense
        {
            Name = "MIT License",
            Url = new Uri("https://opensource.org/licenses/MIT")
        }
    });
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        options.RoutePrefix = string.Empty; // Swagger UI ที่ root URL
        options.DocumentTitle = "My API Documentation";
        options.DisplayRequestDuration();
        options.EnableDeepLinking();
        options.EnableFilter();
        options.ShowExtensions();
    });
}

app.Run();
```

---

## 2. XML Comments

XML Comments ช่วยให้ Swagger แสดง documentation ที่ละเอียดขึ้น

### เปิดใช้งาน XML Documentation

```xml
<!-- MyApi.csproj -->
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>  <!-- ซ่อน warning สำหรับ missing XML comments -->
</PropertyGroup>
```

```csharp
// Program.cs
builder.Services.AddSwaggerGen(options =>
{
    // เพิ่ม XML comments
    var xmlFilename = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFilename);
    options.IncludeXmlComments(xmlPath);
});
```

### การเขียน XML Comments

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;

namespace SwaggerDemo.Controllers;

/// <summary>
/// API สำหรับจัดการ Products
/// </summary>
[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service)
    {
        _service = service;
    }

    /// <summary>
    /// ดึงรายการสินค้าทั้งหมด
    /// </summary>
    /// <param name="page">หน้าที่ต้องการ (เริ่มจาก 1)</param>
    /// <param name="pageSize">จำนวนรายการต่อหน้า (สูงสุด 100)</param>
    /// <param name="search">คำค้นหา (ชื่อสินค้า)</param>
    /// <returns>รายการสินค้าพร้อม pagination info</returns>
    /// <response code="200">ดึงข้อมูลสำเร็จ</response>
    /// <response code="400">Parameter ไม่ถูกต้อง</response>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<ProductDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<PagedResult<ProductDto>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        [FromQuery] string? search = null)
    {
        if (page < 1) return BadRequest("page ต้องมากกว่า 0");
        if (pageSize < 1 || pageSize > 100) return BadRequest("pageSize ต้องอยู่ระหว่าง 1-100");

        var result = await _service.GetAllAsync(page, pageSize, search);
        return Ok(result);
    }

    /// <summary>
    /// ดึงข้อมูลสินค้าตาม ID
    /// </summary>
    /// <param name="id">ID ของสินค้า</param>
    /// <returns>ข้อมูลสินค้า</returns>
    /// <response code="200">พบสินค้า</response>
    /// <response code="404">ไม่พบสินค้า</response>
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product == null) return NotFound($"ไม่พบสินค้า ID: {id}");
        return Ok(product);
    }

    /// <summary>
    /// สร้างสินค้าใหม่
    /// </summary>
    /// <param name="request">ข้อมูลสินค้าที่ต้องการสร้าง</param>
    /// <returns>สินค้าที่สร้างแล้ว</returns>
    /// <remarks>
    /// ตัวอย่าง request body:
    ///
    ///     POST /api/v1/products
    ///     {
    ///         "name": "iPhone 16 Pro",
    ///         "description": "Apple iPhone 16 Pro 256GB",
    ///         "price": 39900.00,
    ///         "stock": 100,
    ///         "categoryId": 1
    ///     }
    ///
    /// </remarks>
    /// <response code="201">สร้างสินค้าสำเร็จ</response>
    /// <response code="400">ข้อมูลไม่ถูกต้อง</response>
    /// <response code="401">ไม่ได้รับการ authenticate</response>
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductRequest request)
    {
        var product = await _service.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    /// <summary>
    /// อัปเดทสินค้า
    /// </summary>
    /// <param name="id">ID ของสินค้าที่ต้องการอัปเดท</param>
    /// <param name="request">ข้อมูลที่ต้องการอัปเดท</param>
    /// <response code="200">อัปเดทสำเร็จ</response>
    /// <response code="404">ไม่พบสินค้า</response>
    [HttpPut("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> Update(int id, [FromBody] UpdateProductRequest request)
    {
        var product = await _service.UpdateAsync(id, request);
        if (product == null) return NotFound();
        return Ok(product);
    }

    /// <summary>
    /// ลบสินค้า (Soft delete)
    /// </summary>
    /// <param name="id">ID ของสินค้าที่ต้องการลบ</param>
    /// <response code="204">ลบสำเร็จ</response>
    /// <response code="404">ไม่พบสินค้า</response>
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        if (!deleted) return NotFound();
        return NoContent();
    }
}
```

### Model Documentation

```csharp
// DTOs/ProductDto.cs
using System.ComponentModel.DataAnnotations;

namespace SwaggerDemo.DTOs;

/// <summary>
/// ข้อมูลสินค้า
/// </summary>
public class ProductDto
{
    /// <summary>ID ของสินค้า</summary>
    /// <example>1</example>
    public int Id { get; set; }

    /// <summary>ชื่อสินค้า</summary>
    /// <example>iPhone 16 Pro</example>
    public string Name { get; set; } = string.Empty;

    /// <summary>รายละเอียดสินค้า</summary>
    /// <example>Apple iPhone 16 Pro 256GB Natural Titanium</example>
    public string Description { get; set; } = string.Empty;

    /// <summary>ราคา (บาท)</summary>
    /// <example>39900.00</example>
    public decimal Price { get; set; }

    /// <summary>จำนวนสินค้าในคลัง</summary>
    /// <example>50</example>
    public int Stock { get; set; }

    /// <summary>สถานะ (active/inactive)</summary>
    /// <example>true</example>
    public bool IsActive { get; set; }

    /// <summary>วันที่สร้าง</summary>
    /// <example>2024-01-15T10:30:00Z</example>
    public DateTime CreatedAt { get; set; }
}

/// <summary>
/// ข้อมูลสำหรับสร้างสินค้าใหม่
/// </summary>
public class CreateProductRequest
{
    /// <summary>ชื่อสินค้า (จำเป็น, 3-200 ตัวอักษร)</summary>
    /// <example>Samsung Galaxy S25</example>
    [Required(ErrorMessage = "ชื่อสินค้าจำเป็น")]
    [StringLength(200, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;

    /// <summary>รายละเอียดสินค้า</summary>
    /// <example>Samsung Galaxy S25 256GB Phantom Black</example>
    [StringLength(2000)]
    public string? Description { get; set; }

    /// <summary>ราคา (ต้องมากกว่า 0)</summary>
    /// <example>29900.00</example>
    [Required]
    [Range(0.01, 10000000)]
    public decimal Price { get; set; }

    /// <summary>จำนวนสินค้า (ต้องไม่ติดลบ)</summary>
    /// <example>100</example>
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }

    /// <summary>ID ของหมวดหมู่</summary>
    /// <example>1</example>
    [Required]
    public int CategoryId { get; set; }
}
```

---

## 3. API Versioning

```bash
dotnet add package Asp.Versioning.Mvc
dotnet add package Asp.Versioning.Mvc.ApiExplorer
```

### การตั้งค่า API Versioning

```csharp
// Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),    // /api/v1/products
        new HeaderApiVersionReader("X-Api-Version"),  // Header
        new QueryStringApiVersionReader("api-version") // ?api-version=1.0
    );
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddSwaggerGen(options =>
{
    // Swagger docs สำหรับแต่ละ version
    var provider = builder.Services.BuildServiceProvider()
        .GetRequiredService<IApiVersionDescriptionProvider>();

    foreach (var description in provider.ApiVersionDescriptions)
    {
        options.SwaggerDoc(description.GroupName, new OpenApiInfo
        {
            Title = $"My API {description.ApiVersion}",
            Version = description.ApiVersion.ToString(),
            Description = description.IsDeprecated
                ? "⚠️ API version นี้ถูก deprecated แล้ว"
                : "API documentation"
        });
    }
});

// Swagger UI แสดงทุก version
app.UseSwaggerUI(options =>
{
    var provider = app.Services.GetRequiredService<IApiVersionDescriptionProvider>();
    foreach (var description in provider.ApiVersionDescriptions.Reverse())
    {
        options.SwaggerEndpoint(
            $"/swagger/{description.GroupName}/swagger.json",
            $"API {description.ApiVersion}");
    }
});
```

### Controllers พร้อม Versioning

```csharp
// Controllers/V1/ProductsController.cs
using Asp.Versioning;
using Microsoft.AspNetCore.Mvc;

namespace SwaggerDemo.Controllers.V1;

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    /// <summary>Get all products (v1)</summary>
    [HttpGet]
    public ActionResult<List<ProductDtoV1>> GetAll()
    {
        return Ok(new List<ProductDtoV1>());
    }
}

// Controllers/V2/ProductsController.cs
namespace SwaggerDemo.Controllers.V2;

[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    /// <summary>Get all products (v2) - พร้อม enhanced data</summary>
    [HttpGet]
    public ActionResult<List<ProductDtoV2>> GetAll()
    {
        return Ok(new List<ProductDtoV2>());
    }
}

// Deprecated version
[ApiController]
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/[controller]")]
public class OldController : ControllerBase
{
    // ...
}
```

---

## 4. Authentication ใน Swagger

### JWT Bearer Authentication

```csharp
// Program.cs
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo { Title = "Secure API", Version = "v1" });

    // เพิ่ม JWT Security Definition
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "กรอก JWT token: eyJhbGci..."
    });

    // เพิ่ม Security Requirement สำหรับทุก endpoint
    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

### OAuth2 / OpenID Connect

```csharp
options.AddSecurityDefinition("oauth2", new OpenApiSecurityScheme
{
    Type = SecuritySchemeType.OAuth2,
    Flows = new OpenApiOAuthFlows
    {
        AuthorizationCode = new OpenApiOAuthFlow
        {
            AuthorizationUrl = new Uri("https://auth.example.com/authorize"),
            TokenUrl = new Uri("https://auth.example.com/token"),
            Scopes = new Dictionary<string, string>
            {
                { "read:products", "อ่านข้อมูลสินค้า" },
                { "write:products", "แก้ไขข้อมูลสินค้า" }
            }
        }
    }
});

// Swagger UI configuration
app.UseSwaggerUI(options =>
{
    options.OAuthClientId("swagger-ui");
    options.OAuthClientSecret("");
    options.OAuthUsePkce();
    options.OAuthScopes("read:products");
});
```

### ApiKey Authentication

```csharp
options.AddSecurityDefinition("ApiKey", new OpenApiSecurityScheme
{
    Name = "X-Api-Key",
    Type = SecuritySchemeType.ApiKey,
    In = ParameterLocation.Header,
    Description = "API Key สำหรับ authentication"
});

options.AddSecurityRequirement(new OpenApiSecurityRequirement
{
    {
        new OpenApiSecurityScheme
        {
            Reference = new OpenApiReference
            {
                Type = ReferenceType.SecurityScheme,
                Id = "ApiKey"
            }
        },
        Array.Empty<string>()
    }
});
```

---

## 5. Custom Filters

### Operation Filter - เพิ่ม custom headers

```csharp
// Filters/AddCorrelationIdFilter.cs
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

namespace SwaggerDemo.Filters;

public class AddCorrelationIdHeaderFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters ??= new List<OpenApiParameter>();

        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Correlation-Id",
            In = ParameterLocation.Header,
            Required = false,
            Schema = new OpenApiSchema
            {
                Type = "string",
                Format = "uuid",
                Example = new Microsoft.OpenApi.Any.OpenApiString(Guid.NewGuid().ToString())
            },
            Description = "Correlation ID สำหรับ request tracing"
        });
    }
}
```

### Document Filter - เพิ่ม custom info

```csharp
// Filters/AddServerEnvironmentFilter.cs
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

namespace SwaggerDemo.Filters;

public class AddServerEnvironmentFilter : IDocumentFilter
{
    private readonly IWebHostEnvironment _env;

    public AddServerEnvironmentFilter(IWebHostEnvironment env)
    {
        _env = env;
    }

    public void Apply(OpenApiDocument swaggerDoc, DocumentFilterContext context)
    {
        // เพิ่ม server info
        swaggerDoc.Servers = new List<OpenApiServer>
        {
            new() { Url = "https://api.example.com", Description = "Production" },
            new() { Url = "https://staging-api.example.com", Description = "Staging" },
            new() { Url = "https://localhost:5001", Description = "Local" }
        };

        // เพิ่ม environment tag ใน info
        swaggerDoc.Info.Description +=
            $"\n\n**Current Environment:** {_env.EnvironmentName}";
    }
}
```

### Schema Filter - custom schema display

```csharp
// Filters/EnumSchemaFilter.cs
using Microsoft.OpenApi.Models;
using Microsoft.OpenApi.Any;
using Swashbuckle.AspNetCore.SwaggerGen;

namespace SwaggerDemo.Filters;

public class EnumSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        if (context.Type.IsEnum)
        {
            schema.Enum.Clear();
            var enumNames = Enum.GetNames(context.Type);
            var enumValues = Enum.GetValues(context.Type).Cast<int>();

            foreach (var (name, value) in enumNames.Zip(enumValues))
            {
                schema.Enum.Add(new OpenApiString($"{value} = {name}"));
            }

            schema.Type = "string";
            schema.Description += $"\n\nValues: {string.Join(", ", enumNames)}";
        }
    }
}
```

### ลงทะเบียน Filters

```csharp
builder.Services.AddSwaggerGen(options =>
{
    options.OperationFilter<AddCorrelationIdHeaderFilter>();
    options.DocumentFilter<AddServerEnvironmentFilter>();
    options.SchemaFilter<EnumSchemaFilter>();
});
```

---

## โปรแกรมตัวอย่าง: Documented API

ระบบ API สำหรับ e-commerce ที่มี documentation ครบถ้วน

### Models และ DTOs

```csharp
// Models/Product.cs
namespace DocumentedApi.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public ProductStatus Status { get; set; }
    public int CategoryId { get; set; }
    public Category? Category { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}

/// <summary>
/// สถานะของสินค้า
/// </summary>
public enum ProductStatus
{
    /// <summary>สินค้าพร้อมขาย</summary>
    Active = 1,

    /// <summary>สินค้าหมดสต็อก</summary>
    OutOfStock = 2,

    /// <summary>สินค้าถูกระงับ</summary>
    Suspended = 3,

    /// <summary>สินค้าถูกลบ</summary>
    Deleted = 4
}

// DTOs/PagedResult.cs
/// <summary>
/// ผลลัพธ์แบบ pagination
/// </summary>
/// <typeparam name="T">ประเภทของ item</typeparam>
public class PagedResult<T>
{
    /// <summary>รายการ items ในหน้านี้</summary>
    public List<T> Items { get; set; } = new();

    /// <summary>จำนวนทั้งหมด</summary>
    /// <example>100</example>
    public int TotalCount { get; set; }

    /// <summary>หน้าปัจจุบัน</summary>
    /// <example>1</example>
    public int Page { get; set; }

    /// <summary>จำนวนรายการต่อหน้า</summary>
    /// <example>20</example>
    public int PageSize { get; set; }

    /// <summary>จำนวนหน้าทั้งหมด</summary>
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);

    /// <summary>มีหน้าถัดไปหรือไม่</summary>
    public bool HasNextPage => Page < TotalPages;

    /// <summary>มีหน้าก่อนหน้าหรือไม่</summary>
    public bool HasPreviousPage => Page > 1;
}
```

### Complete Program.cs

```csharp
// Program.cs
using System.Reflection;
using Asp.Versioning;
using Microsoft.OpenApi.Models;
using DocumentedApi.Filters;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// API Versioning
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
})
.AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Swagger
builder.Services.AddSwaggerGen(options =>
{
    // API Info สำหรับ v1
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "E-Commerce API",
        Version = "v1",
        Description = """
            ## API สำหรับระบบ E-Commerce

            ### Features
            - จัดการสินค้า (Products)
            - จัดการหมวดหมู่ (Categories)
            - จัดการคำสั่งซื้อ (Orders)

            ### Authentication
            ใช้ JWT Bearer token ในการ authenticate
            ขอ token ได้ที่ POST /api/v1/auth/login
            """,
        Contact = new OpenApiContact
        {
            Name = "API Support",
            Email = "api@example.com"
        }
    });

    // API Info สำหรับ v2
    options.SwaggerDoc("v2", new OpenApiInfo
    {
        Title = "E-Commerce API",
        Version = "v2",
        Description = "V2 - เพิ่ม GraphQL support และ enhanced filtering"
    });

    // XML Comments
    var xmlFilename = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    options.IncludeXmlComments(Path.Combine(AppContext.BaseDirectory, xmlFilename));

    // JWT Authentication
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Description = "กรอก JWT token โดยไม่ต้องมี 'Bearer' นำหน้า"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });

    // Custom Filters
    options.OperationFilter<AddCorrelationIdHeaderFilter>();
    options.DocumentFilter<AddServerEnvironmentFilter>();
    options.SchemaFilter<EnumSchemaFilter>();
});

var app = builder.Build();

if (app.Environment.IsDevelopment() || app.Environment.IsStaging())
{
    app.UseSwagger(options =>
    {
        options.RouteTemplate = "api-docs/{documentName}/swagger.json";
    });

    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/api-docs/v2/swagger.json", "E-Commerce API v2");
        options.SwaggerEndpoint("/api-docs/v1/swagger.json", "E-Commerce API v1");
        options.RoutePrefix = "api-docs";
        options.DocumentTitle = "E-Commerce API Documentation";
        options.DisplayRequestDuration();
        options.EnableDeepLinking();
        options.EnableFilter();
        options.DefaultModelsExpandDepth(2);
        options.DefaultModelRendering(Swashbuckle.AspNetCore.SwaggerUI.ModelRendering.Example);
        options.ConfigObject.AdditionalItems["syntaxHighlight"] = new { activated = false };
    });
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

### Documented Controller

```csharp
// Controllers/V1/ProductsController.cs
using Asp.Versioning;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using DocumentedApi.DTOs;

namespace DocumentedApi.Controllers.V1;

/// <summary>
/// API สำหรับจัดการสินค้า
/// </summary>
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
[Produces("application/json")]
[Authorize]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service)
    {
        _service = service;
    }

    /// <summary>ค้นหาสินค้า</summary>
    /// <param name="query">พารามิเตอร์สำหรับค้นหาและกรองสินค้า</param>
    /// <returns>รายการสินค้าพร้อม pagination</returns>
    /// <response code="200">ดึงข้อมูลสำเร็จ</response>
    [HttpGet]
    [AllowAnonymous]
    [ProducesResponseType(typeof(PagedResult<ProductSummaryDto>), 200)]
    public async Task<ActionResult<PagedResult<ProductSummaryDto>>> Search(
        [FromQuery] ProductSearchQuery query)
    {
        var result = await _service.SearchAsync(query);
        return Ok(result);
    }

    /// <summary>ดูรายละเอียดสินค้า</summary>
    /// <param name="id">ID ของสินค้า</param>
    [HttpGet("{id:int}", Name = "GetProductById")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(ProductDetailDto), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<ProductDetailDto>> GetById(
        /// <example>1</example>
        int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product == null) return NotFound(new { message = $"ไม่พบสินค้า ID: {id}" });
        return Ok(product);
    }

    /// <summary>สร้างสินค้าใหม่ (ต้อง login)</summary>
    [HttpPost]
    [ProducesResponseType(typeof(ProductDetailDto), 201)]
    [ProducesResponseType(typeof(ValidationProblemDetails), 400)]
    [ProducesResponseType(401)]
    [ProducesResponseType(403)]
    public async Task<ActionResult<ProductDetailDto>> Create(
        [FromBody] CreateProductRequest request)
    {
        var product = await _service.CreateAsync(request);
        return CreatedAtRoute("GetProductById", new { id = product.Id }, product);
    }

    /// <summary>อัปเดทสินค้า (ต้อง login)</summary>
    [HttpPatch("{id:int}")]
    [ProducesResponseType(typeof(ProductDetailDto), 200)]
    [ProducesResponseType(400)]
    [ProducesResponseType(401)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<ProductDetailDto>> Update(
        int id,
        [FromBody] UpdateProductRequest request)
    {
        var product = await _service.UpdateAsync(id, request);
        if (product == null) return NotFound();
        return Ok(product);
    }

    /// <summary>ลบสินค้า (Soft delete, ต้อง login)</summary>
    [HttpDelete("{id:int}")]
    [ProducesResponseType(204)]
    [ProducesResponseType(401)]
    [ProducesResponseType(404)]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        if (!deleted) return NotFound();
        return NoContent();
    }
}
```

---

## Exercises

### Exercise 1: ReDoc Integration
เพิ่ม ReDoc เป็นอีกหนึ่ง documentation UI นอกจาก Swagger UI

### Exercise 2: Custom Examples
สร้าง IExamplesProvider สำหรับ request/response examples ที่ซับซ้อน

### Exercise 3: Deprecation Notice
สร้าง Operation Filter ที่เพิ่ม deprecation notice สำหรับ endpoints ที่มี `[Obsolete]` attribute

### Exercise 4: Export Documentation
สร้าง endpoint ที่ download Swagger JSON หรือ generate Postman collection

### Exercise 5: Conditional Documentation
ซ่อน/แสดง endpoints บางอย่างตาม environment หรือ role ของ user

---

## สรุป

- **Swashbuckle** สร้าง OpenAPI documentation จาก code
- **XML Comments** ให้ documentation ที่ละเอียดและมี examples
- **API Versioning** ช่วยจัดการ breaking changes
- เพิ่ม **authentication** ใน Swagger UI เพื่อทดสอบ secured endpoints
- **Custom Filters** ช่วยปรับแต่ง documentation ให้ตรงกับความต้องการ
- ใช้ `[ProducesResponseType]` ระบุ response types ทุกกรณี

---

## Part ถัดไป

**Part 067: File Upload/Download** - เรียนรู้การจัดการไฟล์ใน ASP.NET Core

---

*Part 066/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
