# Part 046: MVC - Controllers

## เนื้อหาใน Part นี้
- Controller class structure
- Action methods
- HTTP verbs: GET, POST, PUT, DELETE
- Route attributes
- Model binding
- ActionResult types
- โปรแกรมตัวอย่าง: Products controller

---

## 1. Controller คืออะไร?

Controller คือ class ที่รับ HTTP requests และส่งต่อไปยัง business logic จากนั้น return response กลับไปยัง client

```
HTTP Request
     │
     ▼
  Routing ──────────► Controller ──────────► Service/Repository
     │                    │                       │
     │                    │ ◄──────────────────────┘
     │                    │  return data
     │                    │
     ▼                    ▼
  Client ◄────────── ActionResult
                    (View/JSON/Redirect)
```

---

## 2. สร้าง Controller

### 2.1 Web API Controller

```csharp
// Controllers/ProductsController.cs
using Microsoft.AspNetCore.Mvc;

[ApiController]                          // เปิดใช้ API behaviors
[Route("api/[controller]")]             // Route: api/products
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    // Action methods จะอยู่ที่นี่
}
```

**[ApiController] เปิดใช้ features:**
- Automatic 400 responses สำหรับ validation errors
- Binding source inference (ไม่ต้อง `[FromBody]` เสมอ)
- Problem details response format
- Automatic model state validation

### 2.2 MVC Controller (สำหรับ Views)

```csharp
// Controllers/HomeController.cs
public class HomeController : Controller  // ใช้ Controller แทน ControllerBase
{
    public IActionResult Index()
    {
        return View();  // return Razor view
    }
    
    public IActionResult About()
    {
        ViewData["Title"] = "About Us";
        return View();
    }
}
```

### 2.3 ControllerBase vs Controller

| Feature | ControllerBase | Controller |
|---------|---------------|-----------|
| JSON results | ✅ | ✅ |
| View results | ❌ | ✅ |
| ViewData/ViewBag | ❌ | ✅ |
| TempData | ❌ | ✅ |
| เหมาะกับ | API | MVC |

---

## 3. HTTP Verbs

### 3.1 GET - ดึงข้อมูล

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    
    public ProductsController(IProductService service)
    {
        _service = service;
    }

    // GET api/products
    [HttpGet]
    public async Task<ActionResult<List<ProductDto>>> GetAll(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10,
        [FromQuery] string? search = null,
        [FromQuery] string? sortBy = null)
    {
        var result = await _service.GetAllAsync(page, pageSize, search, sortBy);
        return Ok(result);
    }

    // GET api/products/5
    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null)
            return NotFound(new { Message = $"Product {id} not found" });
        return Ok(product);
    }

    // GET api/products/sku/ABC123
    [HttpGet("sku/{sku}")]
    public async Task<ActionResult<ProductDto>> GetBySku(string sku)
    {
        var product = await _service.GetBySkuAsync(sku);
        return product is null ? NotFound() : Ok(product);
    }

    // GET api/products/categories
    [HttpGet("categories")]
    public async Task<ActionResult<List<string>>> GetCategories()
    {
        var categories = await _service.GetCategoriesAsync();
        return Ok(categories);
    }
}
```

### 3.2 POST - สร้างข้อมูล

```csharp
// POST api/products
[HttpPost]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductRequest request)
{
    // ถ้าใช้ [ApiController] จะ validate model state อัตโนมัติ
    // ไม่จำเป็นต้องเช็ค ModelState.IsValid เอง
    
    var product = await _service.CreateAsync(request);
    
    // Return 201 Created พร้อม Location header
    return CreatedAtAction(
        nameof(GetById),        // action name
        new { id = product.Id }, // route values
        product);               // response body
}

// POST api/products/bulk
[HttpPost("bulk")]
public async Task<ActionResult<List<ProductDto>>> CreateBulk(
    [FromBody] List<CreateProductRequest> requests)
{
    if (!requests.Any())
        return BadRequest(new { Message = "กรุณาระบุสินค้าอย่างน้อย 1 รายการ" });
    
    if (requests.Count > 100)
        return BadRequest(new { Message = "สามารถสร้างได้ไม่เกิน 100 รายการต่อครั้ง" });
    
    var products = await _service.CreateBulkAsync(requests);
    return CreatedAtAction(nameof(GetAll), products);
}
```

### 3.3 PUT - แก้ไขข้อมูล (ทั้งหมด)

```csharp
// PUT api/products/5
[HttpPut("{id:int}")]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult<ProductDto>> Update(
    int id, 
    [FromBody] UpdateProductRequest request)
{
    var product = await _service.UpdateAsync(id, request);
    
    return product is null 
        ? NotFound(new { Message = $"Product {id} not found" })
        : Ok(product);
}
```

### 3.4 PATCH - แก้ไขบางส่วน

```csharp
// PATCH api/products/5
[HttpPatch("{id:int}")]
public async Task<ActionResult<ProductDto>> Patch(
    int id, 
    [FromBody] JsonPatchDocument<UpdateProductRequest> patchDoc)
{
    var product = await _service.GetByIdAsync(id);
    if (product is null) return NotFound();
    
    var updateRequest = new UpdateProductRequest
    {
        Name = product.Name,
        Price = product.Price,
        Stock = product.Stock
    };
    
    patchDoc.ApplyTo(updateRequest, ModelState);
    
    if (!ModelState.IsValid)
        return BadRequest(ModelState);
    
    var updated = await _service.UpdateAsync(id, updateRequest);
    return Ok(updated);
}
```

### 3.5 DELETE - ลบข้อมูล

```csharp
// DELETE api/products/5
[HttpDelete("{id:int}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> Delete(int id)
{
    var deleted = await _service.DeleteAsync(id);
    
    return deleted ? NoContent() : NotFound();
}

// DELETE api/products/bulk
[HttpDelete("bulk")]
public async Task<IActionResult> DeleteBulk([FromBody] List<int> ids)
{
    await _service.DeleteBulkAsync(ids);
    return NoContent();
}
```

---

## 4. Route Attributes

### 4.1 Route Templates

```csharp
// Route ที่ level class
[Route("api/[controller]")]           // [controller] = "Products"
[Route("api/v1/products")]            // hard-coded
[Route("api/v{version:apiVersion}/[controller]")] // versioned

// Route ที่ level action
[HttpGet]                              // GET api/products
[HttpGet("{id}")]                      // GET api/products/5
[HttpGet("{id:int}")]                 // GET api/products/5 (int constraint)
[HttpGet("{id:int:min(1)}")]          // GET api/products/5 (min value 1)
[HttpGet("{name:alpha}")]             // GET api/products/electronics (alpha only)
[HttpGet("{id}/{name}")]              // GET api/products/5/laptop
[HttpGet("search")]                   // GET api/products/search
[HttpGet("~/health")]                 // GET /health (override class route)
```

### 4.2 Route Constraints

```csharp
// int - integer
[HttpGet("{id:int}")]           // /products/5 ✅, /products/abc ❌

// long - long integer  
[HttpGet("{id:long}")]

// decimal
[HttpGet("{price:decimal}")]

// bool
[HttpGet("{active:bool}")]

// guid
[HttpGet("{id:guid}")]          // /products/a1b2c3d4-e5f6-...

// alpha - letters only
[HttpGet("{name:alpha}")]       // /products/laptop ✅, /products/123 ❌

// minlength/maxlength
[HttpGet("{code:minlength(3)}")]

// min/max
[HttpGet("{id:int:min(1):max(1000)}")]

// range
[HttpGet("{id:int:range(1,1000)}")]

// regex
[HttpGet("{code:regex(^[A-Z]{{2}}[0-9]{{3}}$)}")]  // AB123

// datetime
[HttpGet("{date:datetime}")]
```

### 4.3 Multiple Routes

```csharp
// Action หนึ่งตัวรับหลาย routes
[HttpGet("")]
[HttpGet("all")]
[HttpGet("list")]
public async Task<ActionResult<List<Product>>> GetAll() { ... }

// ใช้ Route attribute หลายตัว
[Route("api/products")]
[Route("api/items")]
[ApiController]
public class ProductsController : ControllerBase { ... }
```

---

## 5. Model Binding

Model binding คือกระบวนการที่ ASP.NET Core ดึงข้อมูลจาก request และแปลงเป็น parameter ของ action method

### 5.1 Binding Sources

```csharp
[HttpGet("{id}")]
public IActionResult GetById(
    [FromRoute] int id,          // จาก URL path: /products/5
    [FromQuery] string? sort,    // จาก query string: ?sort=name
    [FromHeader] string? auth,   // จาก HTTP header: X-Auth: token
    [FromBody] SearchFilter? filter, // จาก request body (JSON)
    [FromForm] IFormFile? file,  // จาก multipart form data
    [FromServices] ILogger<ProductsController> logger) // จาก DI
{
    // ...
}
```

### 5.2 Auto-binding ด้วย [ApiController]

```csharp
// เมื่อใช้ [ApiController] ไม่ต้องระบุ [FromBody] สำหรับ complex types
[HttpPost]
public async Task<IActionResult> Create(CreateProductRequest request)
// request ถูก bind จาก body อัตโนมัติ (เพราะ complex type)

[HttpGet("{id}")]
public async Task<IActionResult> Get(int id, string? search)
// id ถูก bind จาก route, search จาก query string อัตโนมัติ
```

### 5.3 Binding Complex Objects

```csharp
// Query string binding
// GET /products?filter.minPrice=100&filter.maxPrice=500&filter.category=Electronics
public IActionResult Search([FromQuery] ProductFilter filter)

public class ProductFilter
{
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public string? Category { get; set; }
    public bool? InStock { get; set; }
    public List<string> Tags { get; set; } = [];
}

// Multiple values: ?tags=electronics&tags=sale&tags=new
public IActionResult Search([FromQuery] string[] tags)
```

### 5.4 File Upload

```csharp
// Single file
[HttpPost("upload")]
[Consumes("multipart/form-data")]
public async Task<IActionResult> Upload(IFormFile file)
{
    if (file.Length == 0) return BadRequest("File is empty");
    if (file.Length > 10 * 1024 * 1024) return BadRequest("File too large (max 10MB)");
    
    var allowedTypes = new[] { "image/jpeg", "image/png", "application/pdf" };
    if (!allowedTypes.Contains(file.ContentType))
        return BadRequest("Unsupported file type");

    var fileName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
    var path = Path.Combine("uploads", fileName);
    
    await using var stream = System.IO.File.Create(path);
    await file.CopyToAsync(stream);
    
    return Ok(new { FileName = fileName, Size = file.Length });
}

// Multiple files
[HttpPost("upload-multiple")]
public async Task<IActionResult> UploadMultiple(List<IFormFile> files)
{
    var results = new List<object>();
    
    foreach (var file in files)
    {
        var fileName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
        // save file...
        results.Add(new { file.FileName, SavedAs = fileName });
    }
    
    return Ok(results);
}
```

---

## 6. ActionResult Types

### 6.1 HTTP Status Codes Methods

```csharp
// 2xx Success
return Ok(data);                    // 200 OK
return Created("/url", data);       // 201 Created
return CreatedAtAction("Get", new { id = 1 }, data); // 201 Created
return CreatedAtRoute("GetProduct", new { id = 1 }, data); // 201 Created
return Accepted();                  // 202 Accepted
return AcceptedAtAction("Check", new { id = jobId }); // 202
return NoContent();                 // 204 No Content

// 3xx Redirection
return Redirect("https://example.com");         // 302
return RedirectPermanent("https://example.com"); // 301
return RedirectToAction("Index", "Home");        // redirect to action
return RedirectToRoute("Default", new { controller = "Home" });
return LocalRedirect("/home");                   // redirect to local URL only (safe)

// 4xx Client Errors
return BadRequest();                // 400
return BadRequest(ModelState);      // 400 with validation errors
return BadRequest(new { Error = "Invalid input" }); // 400 with message
return Unauthorized();              // 401
return Forbid();                    // 403
return NotFound();                  // 404
return NotFound(new { Message = "Resource not found" }); // 404 with message
return Conflict();                  // 409
return UnprocessableEntity();       // 422
return StatusCode(429, new { Message = "Too many requests" }); // 429

// 5xx Server Errors
return StatusCode(500, new { Error = "Internal error" }); // 500
return Problem("Server error occurred");                    // 500 RFC 7807
```

### 6.2 IActionResult vs ActionResult<T>

```csharp
// IActionResult - ไม่ระบุ type (Swagger ไม่รู้ response type)
[HttpGet("{id}")]
public IActionResult GetById(int id)
{
    var product = _service.GetById(id);
    return product is null ? NotFound() : Ok(product);
}

// ActionResult<T> - ระบุ type (Swagger รู้ response type)
[HttpGet("{id}")]
public ActionResult<ProductDto> GetById(int id)
{
    var product = _service.GetById(id);
    return product is null ? NotFound() : product; // implicit conversion
}

// async version
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetByIdAsync(int id)
{
    var product = await _service.GetByIdAsync(id);
    if (product is null) return NotFound();
    return product; // implicit conversion to Ok(product)
}
```

### 6.3 ProducesResponseType Attributes

```csharp
// บอก Swagger ว่า endpoint return อะไรบ้าง
[HttpPost]
[ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(StatusCodes.Status409Conflict)]
[ProducesResponseType(StatusCodes.Status500InternalServerError)]
public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductRequest request)
{
    // ...
}
```

### 6.4 Problem Details (RFC 7807)

```csharp
// ProblemDetails response (standard error format)
return Problem(
    detail: "เกิดข้อผิดพลาดในการ process request",
    instance: HttpContext.Request.Path,
    statusCode: 500,
    title: "Internal Server Error",
    type: "https://tools.ietf.org/html/rfc7807"
);

// ValidationProblemDetails (422 with validation errors)
ModelState.AddModelError("Name", "Name is required");
ModelState.AddModelError("Price", "Price must be greater than 0");
return ValidationProblem(ModelState);

// Custom problem
var problem = new ProblemDetails
{
    Status = 409,
    Title = "Conflict",
    Detail = $"Product with SKU {sku} already exists",
    Instance = Request.Path
};
problem.Extensions["sku"] = sku;
return Conflict(problem);
```

---

## 7. โปรแกรมตัวอย่าง: Products Controller

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register services
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();

app.Run();

// ============================================
// MODELS
// ============================================

public record Product(
    int Id,
    string Name,
    string Description,
    decimal Price,
    int Stock,
    string Category,
    string Sku,
    bool IsActive,
    DateTime CreatedAt,
    DateTime UpdatedAt
);

public record ProductDto(
    int Id,
    string Name,
    string Description,
    decimal Price,
    int Stock,
    string Category,
    string Sku,
    bool IsActive
);

public class CreateProductRequest
{
    [Required(ErrorMessage = "ชื่อสินค้าจำเป็น")]
    [StringLength(200, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;

    [StringLength(2000)]
    public string Description { get; set; } = string.Empty;

    [Required]
    [Range(0.01, 1000000, ErrorMessage = "ราคาต้องมากกว่า 0")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "จำนวน stock ต้องไม่ติดลบ")]
    public int Stock { get; set; }

    [Required]
    public string Category { get; set; } = string.Empty;

    [Required]
    [RegularExpression(@"^[A-Z]{2}[0-9]{4}$", ErrorMessage = "SKU ต้องเป็นรูปแบบ XX0000")]
    public string Sku { get; set; } = string.Empty;
}

public class UpdateProductRequest
{
    [Required, StringLength(200, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;

    [StringLength(2000)]
    public string Description { get; set; } = string.Empty;

    [Required, Range(0.01, 1000000)]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue)]
    public int Stock { get; set; }

    [Required]
    public string Category { get; set; } = string.Empty;
}

public class ProductFilter
{
    public string? Search { get; set; }
    public string? Category { get; set; }
    public decimal? MinPrice { get; set; }
    public decimal? MaxPrice { get; set; }
    public bool? InStock { get; set; }
    public string? SortBy { get; set; }
    public bool SortDescending { get; set; }
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
}

public record PagedResult<T>(
    List<T> Data,
    int Page,
    int PageSize,
    int TotalCount
)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}

// ============================================
// CONTROLLER
// ============================================

[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductService service, ILogger<ProductsController> logger)
    {
        _service = service;
        _logger = logger;
    }

    /// <summary>ดึงรายการสินค้าทั้งหมดพร้อม filtering และ pagination</summary>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<ProductDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<ProductDto>>> GetAll([FromQuery] ProductFilter filter)
    {
        _logger.LogInformation("Getting products with filter: {@Filter}", filter);
        var result = await _service.GetAllAsync(filter);
        return Ok(result);
    }

    /// <summary>ดึงสินค้าตาม ID</summary>
    [HttpGet("{id:int}", Name = "GetProductById")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null)
        {
            _logger.LogWarning("Product {Id} not found", id);
            return NotFound(new { Message = $"ไม่พบสินค้า ID: {id}" });
        }
        return Ok(product);
    }

    /// <summary>ดึงสินค้าตาม SKU</summary>
    [HttpGet("sku/{sku}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetBySku(string sku)
    {
        var product = await _service.GetBySkuAsync(sku);
        return product is null ? NotFound() : Ok(product);
    }

    /// <summary>ดึงหมวดหมู่สินค้าทั้งหมด</summary>
    [HttpGet("categories")]
    [ProducesResponseType(typeof(List<string>), StatusCodes.Status200OK)]
    public async Task<ActionResult<List<string>>> GetCategories()
    {
        var categories = await _service.GetCategoriesAsync();
        return Ok(categories);
    }

    /// <summary>สร้างสินค้าใหม่</summary>
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductRequest request)
    {
        // ตรวจสอบ SKU ซ้ำ
        var existing = await _service.GetBySkuAsync(request.Sku);
        if (existing is not null)
        {
            return Conflict(new ProblemDetails
            {
                Status = 409,
                Title = "Conflict",
                Detail = $"สินค้า SKU '{request.Sku}' มีอยู่ในระบบแล้ว"
            });
        }

        var product = await _service.CreateAsync(request);
        _logger.LogInformation("Created product {Id}: {Name}", product.Id, product.Name);

        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    /// <summary>แก้ไขสินค้า</summary>
    [HttpPut("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> Update(
        int id,
        [FromBody] UpdateProductRequest request)
    {
        var product = await _service.UpdateAsync(id, request);

        if (product is null)
            return NotFound(new { Message = $"ไม่พบสินค้า ID: {id}" });

        _logger.LogInformation("Updated product {Id}", id);
        return Ok(product);
    }

    /// <summary>แก้ไขราคาสินค้า</summary>
    [HttpPatch("{id:int}/price")]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> UpdatePrice(
        int id,
        [FromBody] UpdatePriceRequest request)
    {
        if (request.Price <= 0)
            return BadRequest(new { Message = "ราคาต้องมากกว่า 0" });

        var product = await _service.UpdatePriceAsync(id, request.Price);
        return product is null ? NotFound() : Ok(product);
    }

    /// <summary>ลบสินค้า</summary>
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        if (!deleted)
            return NotFound(new { Message = $"ไม่พบสินค้า ID: {id}" });

        _logger.LogInformation("Deleted product {Id}", id);
        return NoContent();
    }

    /// <summary>ค้นหาสินค้า</summary>
    [HttpGet("search")]
    [ProducesResponseType(typeof(List<ProductDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<List<ProductDto>>> Search(
        [FromQuery] string q,
        [FromQuery] string? category = null)
    {
        if (string.IsNullOrWhiteSpace(q))
            return BadRequest(new { Message = "กรุณาระบุคำค้นหา" });

        var products = await _service.SearchAsync(q, category);
        return Ok(products);
    }
}

public class UpdatePriceRequest
{
    [Range(0.01, 1000000)]
    public decimal Price { get; set; }
}

// ============================================
// SERVICE & REPOSITORY
// ============================================

public interface IProductService
{
    Task<PagedResult<ProductDto>> GetAllAsync(ProductFilter filter);
    Task<ProductDto?> GetByIdAsync(int id);
    Task<ProductDto?> GetBySkuAsync(string sku);
    Task<List<string>> GetCategoriesAsync();
    Task<ProductDto> CreateAsync(CreateProductRequest request);
    Task<ProductDto?> UpdateAsync(int id, UpdateProductRequest request);
    Task<ProductDto?> UpdatePriceAsync(int id, decimal price);
    Task<bool> DeleteAsync(int id);
    Task<List<ProductDto>> SearchAsync(string query, string? category = null);
}

public interface IProductRepository
{
    Task<(List<Product> Products, int Total)> GetAllAsync(ProductFilter filter);
    Task<Product?> GetByIdAsync(int id);
    Task<Product?> GetBySkuAsync(string sku);
    Task<List<string>> GetCategoriesAsync();
    Task<Product> CreateAsync(Product product);
    Task<Product?> UpdateAsync(Product product);
    Task<bool> DeleteAsync(int id);
    Task<List<Product>> SearchAsync(string query, string? category = null);
}

public class ProductService : IProductService
{
    private readonly IProductRepository _repository;

    public ProductService(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<PagedResult<ProductDto>> GetAllAsync(ProductFilter filter)
    {
        var (products, total) = await _repository.GetAllAsync(filter);
        var dtos = products.Select(ToDto).ToList();
        return new PagedResult<ProductDto>(dtos, filter.Page, filter.PageSize, total);
    }

    public async Task<ProductDto?> GetByIdAsync(int id)
    {
        var product = await _repository.GetByIdAsync(id);
        return product is null ? null : ToDto(product);
    }

    public async Task<ProductDto?> GetBySkuAsync(string sku)
    {
        var product = await _repository.GetBySkuAsync(sku);
        return product is null ? null : ToDto(product);
    }

    public async Task<List<string>> GetCategoriesAsync()
        => await _repository.GetCategoriesAsync();

    public async Task<ProductDto> CreateAsync(CreateProductRequest request)
    {
        var product = new Product(
            Id: 0,
            Name: request.Name,
            Description: request.Description,
            Price: request.Price,
            Stock: request.Stock,
            Category: request.Category,
            Sku: request.Sku.ToUpper(),
            IsActive: true,
            CreatedAt: DateTime.UtcNow,
            UpdatedAt: DateTime.UtcNow
        );

        var created = await _repository.CreateAsync(product);
        return ToDto(created);
    }

    public async Task<ProductDto?> UpdateAsync(int id, UpdateProductRequest request)
    {
        var existing = await _repository.GetByIdAsync(id);
        if (existing is null) return null;

        var updated = existing with
        {
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            Stock = request.Stock,
            Category = request.Category,
            UpdatedAt = DateTime.UtcNow
        };

        var result = await _repository.UpdateAsync(updated);
        return result is null ? null : ToDto(result);
    }

    public async Task<ProductDto?> UpdatePriceAsync(int id, decimal price)
    {
        var existing = await _repository.GetByIdAsync(id);
        if (existing is null) return null;

        var updated = existing with { Price = price, UpdatedAt = DateTime.UtcNow };
        var result = await _repository.UpdateAsync(updated);
        return result is null ? null : ToDto(result);
    }

    public async Task<bool> DeleteAsync(int id)
        => await _repository.DeleteAsync(id);

    public async Task<List<ProductDto>> SearchAsync(string query, string? category = null)
    {
        var products = await _repository.SearchAsync(query, category);
        return products.Select(ToDto).ToList();
    }

    private static ProductDto ToDto(Product p) => new(
        p.Id, p.Name, p.Description, p.Price, p.Stock, p.Category, p.Sku, p.IsActive);
}

public class InMemoryProductRepository : IProductRepository
{
    private readonly List<Product> _products = new()
    {
        new(1, "iPhone 15 Pro", "Apple smartphone", 39900m, 50, "Electronics", "IP0001", true, DateTime.UtcNow.AddDays(-30), DateTime.UtcNow.AddDays(-5)),
        new(2, "MacBook Pro M3", "Apple laptop", 79900m, 20, "Electronics", "MB0001", true, DateTime.UtcNow.AddDays(-25), DateTime.UtcNow.AddDays(-3)),
        new(3, "Nike Air Max", "Running shoes", 4500m, 100, "Footwear", "NK0001", true, DateTime.UtcNow.AddDays(-20), DateTime.UtcNow.AddDays(-1)),
        new(4, "Programming C#", "C# book", 850m, 200, "Books", "BK0001", true, DateTime.UtcNow.AddDays(-15), DateTime.UtcNow),
        new(5, "Coffee Maker", "Automatic coffee machine", 3500m, 30, "Kitchen", "CF0001", true, DateTime.UtcNow.AddDays(-10), DateTime.UtcNow),
    };

    private int _nextId = 6;
    private readonly object _lock = new();

    public Task<(List<Product> Products, int Total)> GetAllAsync(ProductFilter filter)
    {
        var query = _products.AsQueryable();

        if (!string.IsNullOrWhiteSpace(filter.Search))
            query = query.Where(p => p.Name.Contains(filter.Search, StringComparison.OrdinalIgnoreCase));
        if (!string.IsNullOrWhiteSpace(filter.Category))
            query = query.Where(p => p.Category.Equals(filter.Category, StringComparison.OrdinalIgnoreCase));
        if (filter.MinPrice.HasValue)
            query = query.Where(p => p.Price >= filter.MinPrice.Value);
        if (filter.MaxPrice.HasValue)
            query = query.Where(p => p.Price <= filter.MaxPrice.Value);
        if (filter.InStock.HasValue)
            query = filter.InStock.Value ? query.Where(p => p.Stock > 0) : query.Where(p => p.Stock == 0);

        query = filter.SortBy?.ToLower() switch
        {
            "name" => filter.SortDescending ? query.OrderByDescending(p => p.Name) : query.OrderBy(p => p.Name),
            "price" => filter.SortDescending ? query.OrderByDescending(p => p.Price) : query.OrderBy(p => p.Price),
            _ => query.OrderBy(p => p.Id)
        };

        var total = query.Count();
        var products = query.Skip((filter.Page - 1) * filter.PageSize).Take(filter.PageSize).ToList();

        return Task.FromResult((products, total));
    }

    public Task<Product?> GetByIdAsync(int id)
        => Task.FromResult(_products.FirstOrDefault(p => p.Id == id));

    public Task<Product?> GetBySkuAsync(string sku)
        => Task.FromResult(_products.FirstOrDefault(p => p.Sku.Equals(sku, StringComparison.OrdinalIgnoreCase)));

    public Task<List<string>> GetCategoriesAsync()
        => Task.FromResult(_products.Select(p => p.Category).Distinct().OrderBy(c => c).ToList());

    public Task<Product> CreateAsync(Product product)
    {
        lock (_lock)
        {
            var newProduct = product with { Id = _nextId++ };
            _products.Add(newProduct);
            return Task.FromResult(newProduct);
        }
    }

    public Task<Product?> UpdateAsync(Product product)
    {
        lock (_lock)
        {
            var index = _products.FindIndex(p => p.Id == product.Id);
            if (index < 0) return Task.FromResult<Product?>(null);
            _products[index] = product;
            return Task.FromResult<Product?>(product);
        }
    }

    public Task<bool> DeleteAsync(int id)
    {
        lock (_lock)
        {
            var product = _products.FirstOrDefault(p => p.Id == id);
            if (product is null) return Task.FromResult(false);
            _products.Remove(product);
            return Task.FromResult(true);
        }
    }

    public Task<List<Product>> SearchAsync(string query, string? category = null)
    {
        var results = _products
            .Where(p => p.Name.Contains(query, StringComparison.OrdinalIgnoreCase)
                     || p.Description.Contains(query, StringComparison.OrdinalIgnoreCase))
            .Where(p => category == null || p.Category.Equals(category, StringComparison.OrdinalIgnoreCase))
            .ToList();
        return Task.FromResult(results);
    }
}
```

---

## Exercises

### Exercise 1: Order Controller
สร้าง `OrdersController` สำหรับจัดการ orders:

```csharp
// Endpoints ที่ต้องการ:
// GET /api/orders - ดึง orders ทั้งหมด (with pagination)
// GET /api/orders/{id} - ดึง order ตาม ID
// GET /api/orders/user/{userId} - ดึง orders ของ user
// POST /api/orders - สร้าง order ใหม่
// PUT /api/orders/{id}/status - เปลี่ยน status ของ order
// DELETE /api/orders/{id} - ยกเลิก order

public enum OrderStatus { Pending, Processing, Shipped, Delivered, Cancelled }
```

### Exercise 2: File Upload Controller
สร้าง controller สำหรับ file management:

```csharp
// Endpoints:
// POST /api/files/upload - upload file (ตรวจสอบ type และ size)
// POST /api/files/upload-multiple - upload หลายไฟล์
// GET /api/files/{filename} - download file
// DELETE /api/files/{filename} - ลบ file
// GET /api/files - list ไฟล์ทั้งหมด
```

### Exercise 3: เพิ่ม Logging
เพิ่ม structured logging ใน Products Controller:

```csharp
// ต้องการ log:
// - Request เริ่มต้น (method, path, user)
// - Operation สำเร็จ (id ที่ถูกสร้าง/แก้ไข/ลบ)
// - Operation ล้มเหลว (reason)
// - Performance (elapsed time สำหรับ operations ที่นานกว่า 100ms)
```

---

## สรุป

✅ Controller รับ HTTP requests และ return responses  
✅ `[ApiController]` เปิดใช้ automatic validation, binding inference และ problem details  
✅ `ControllerBase` สำหรับ API, `Controller` สำหรับ MVC (Views)  
✅ HTTP verbs: `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]`, `[HttpPatch]`  
✅ Route attributes: `[Route("api/[controller]")]`, `[HttpGet("{id:int}")]`  
✅ Model binding: `[FromRoute]`, `[FromQuery]`, `[FromBody]`, `[FromHeader]`, `[FromForm]`  
✅ ActionResult types: `Ok()`, `Created()`, `NoContent()`, `NotFound()`, `BadRequest()`, `Problem()`  
✅ `ActionResult<T>` ดีกว่า `IActionResult` เพราะ Swagger รู้ response type  
✅ `CreatedAtAction()` return 201 พร้อม Location header  

---

## Part ถัดไป

ใน **Part 047** เราจะเรียนรู้เรื่อง **MVC: Views และ Razor** อย่างละเอียด:
- Razor syntax
- Layouts
- Partial views
- Tag Helpers
- View Components

---

*Part 046/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
