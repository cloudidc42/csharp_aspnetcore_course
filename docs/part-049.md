# Part 049: Routing ใน ASP.NET Core

## เนื้อหาใน Part นี้
- Convention-based routing
- Attribute routing
- Route constraints
- Route parameters
- URL generation
- Area routing
- โปรแกรมตัวอย่าง: RESTful routes

---

## 1. Routing คืออะไร?

Routing คือกระบวนการที่ ASP.NET Core จับคู่ HTTP request URL กับ endpoint (controller action หรือ Minimal API handler)

```
HTTP Request: GET /api/products/5?includeDetails=true
                │
                ▼
         Route Matching
         ┌──────────────────────────────┐
         │ Pattern: /api/products/{id}  │
         │ Controller: ProductsController│
         │ Action: GetById              │
         │ id = 5                       │
         │ includeDetails = true        │
         └──────────────────────────────┘
                │
                ▼
         ProductsController.GetById(5, true)
```

---

## 2. Routing Middleware

```csharp
var app = builder.Build();

// UseRouting - วิเคราะห์ URL และเลือก endpoint
app.UseRouting();

// Middleware ระหว่างนี้สามารถใช้ routing information ได้
app.UseAuthentication();
app.UseAuthorization();

// MapControllers/MapGet etc. - ลงทะเบียน endpoints
app.MapControllers();
```

ใน .NET 6+ `UseRouting()` และ `UseEndpoints()` ถูกเรียกอัตโนมัติหากไม่ได้กำหนดเอง

---

## 3. Convention-based Routing

Convention-based routing ใช้ pattern template ที่กำหนดไว้ล่วงหน้า

### 3.1 Default Convention Route

```csharp
// MVC default route
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// Route จะ match:
// /              → HomeController.Index()
// /Home          → HomeController.Index()
// /Home/Index    → HomeController.Index()
// /Products      → ProductsController.Index()
// /Products/Details/5 → ProductsController.Details(5)
// /Blog/Post/hello-world → BlogController.Post("hello-world")
```

### 3.2 Multiple Routes

```csharp
// ลงทะเบียนหลาย routes (ลำดับสำคัญ - ตรวจสอบจากบนลงล่าง)
app.MapControllerRoute(
    name: "blog",
    pattern: "blog/{year:int}/{month:int}/{slug}",
    defaults: new { controller = "Blog", action = "Post" });

app.MapControllerRoute(
    name: "admin",
    pattern: "admin/{controller=Dashboard}/{action=Index}/{id?}",
    constraints: new { controller = new RegexRouteConstraint("^(?!api)") });

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

### 3.3 Route Defaults

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller}/{action}/{id?}",
    defaults: new { controller = "Home", action = "Index" });

// เทียบเท่ากับ
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

---

## 4. Attribute Routing

Attribute routing กำหนด routes โดยตรงบน controller และ action methods

### 4.1 Route บน Controller

```csharp
// Route prefix จาก class
[Route("api/products")]
[ApiController]
public class ProductsController : ControllerBase
{
    // GET api/products
    [HttpGet]
    public IActionResult GetAll() => Ok();

    // GET api/products/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id) => Ok();

    // POST api/products
    [HttpPost]
    public IActionResult Create() => Ok();
}

// ใช้ [controller] token
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    // Route จะเป็น api/orders
}

// ใช้ [action] token
[Route("[controller]/[action]")]
public class ReportsController : Controller
{
    // GET /Reports/Monthly
    public IActionResult Monthly() => View();
    
    // GET /Reports/Weekly
    public IActionResult Weekly() => View();
}
```

### 4.2 Route บน Action Method

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    // GET api/users
    [HttpGet]
    public IActionResult GetAll() => Ok();

    // GET api/users/5
    [HttpGet("{id:int}")]
    public IActionResult GetById(int id) => Ok();

    // GET api/users/email/user@example.com
    [HttpGet("email/{email}")]
    public IActionResult GetByEmail(string email) => Ok();

    // GET api/users/5/orders
    [HttpGet("{id:int}/orders")]
    public IActionResult GetUserOrders(int id) => Ok();

    // GET api/users/5/orders/10
    [HttpGet("{userId:int}/orders/{orderId:int}")]
    public IActionResult GetUserOrder(int userId, int orderId) => Ok();

    // หลาย routes สำหรับ action เดียวกัน
    [HttpGet("")]
    [HttpGet("list")]
    [HttpGet("all")]
    public IActionResult GetAllUsers() => Ok();

    // Override class-level route
    [HttpGet("~/api/v2/members/{id}")]  // ~ = absolute URL
    public IActionResult GetMember(int id) => Ok();
}
```

---

## 5. Route Constraints

### 5.1 Built-in Constraints

```csharp
// Type constraints
[HttpGet("{id:int}")]           // int
[HttpGet("{id:long}")]          // long
[HttpGet("{id:decimal}")]       // decimal
[HttpGet("{id:double}")]        // double
[HttpGet("{id:float}")]         // float
[HttpGet("{id:bool}")]          // bool
[HttpGet("{id:guid}")]          // GUID
[HttpGet("{id:datetime}")]      // DateTime

// String constraints
[HttpGet("{name:alpha}")]        // letters only (a-z, A-Z)
[HttpGet("{name:minlength(3)}")] // minimum length 3
[HttpGet("{name:maxlength(20)}")] // maximum length 20
[HttpGet("{name:length(3,20)}")] // length between 3-20

// Numeric constraints
[HttpGet("{id:min(1)}")]        // minimum value 1
[HttpGet("{id:max(100)}")]      // maximum value 100
[HttpGet("{id:range(1,100)}")] // value between 1-100

// Regex constraint
[HttpGet("{code:regex(^[A-Z]{{2}}[0-9]{{3}}$)}")]  // ตัวอักษร 2 ตัว + เลข 3 ตัว

// Required constraint
[HttpGet("{id:required}")]      // ไม่สามารถเป็นค่าว่างได้
```

### 5.2 รวม Constraints หลายตัว

```csharp
// รวม constraints ด้วย :
[HttpGet("{id:int:min(1):max(1000)}")]  // int ระหว่าง 1-1000
[HttpGet("{name:alpha:minlength(3):maxlength(20)}")]  // letters, length 3-20

// ตัวอย่างจริง
[HttpGet("products/{id:int:min(1)}")]
public IActionResult GetProduct(int id) { ... }

[HttpGet("users/{username:alpha:minlength(3)}")]
public IActionResult GetUserByUsername(string username) { ... }
```

### 5.3 Custom Route Constraints

```csharp
// สร้าง custom constraint
public class EvenNumberConstraint : IRouteConstraint
{
    public bool Match(
        HttpContext? httpContext,
        IRouter? route,
        string routeKey,
        RouteValueDictionary values,
        RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            if (int.TryParse(value?.ToString(), out int number))
                return number % 2 == 0;
        }
        return false;
    }
}

// ลงทะเบียน
builder.Services.Configure<RouteOptions>(options =>
{
    options.ConstraintMap.Add("even", typeof(EvenNumberConstraint));
});

// ใช้งาน
[HttpGet("{id:even}")]
public IActionResult GetEvenItem(int id) => Ok(id);
```

---

## 6. Route Parameters

### 6.1 Optional Parameters

```csharp
// Optional parameter ใช้ ?
[HttpGet("{id?}")]
public IActionResult Get(int? id = null)
{
    if (id.HasValue)
        return Ok($"Item {id}");
    return Ok("All items");
}

// Optional ใน convention route
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");  // id optional
```

### 6.2 Catch-all Parameters

```csharp
// {**path} หรือ {*path} - จับทุกอย่างหลัง /
[HttpGet("files/{**path}")]
public IActionResult GetFile(string path)
{
    // path = "images/2024/photo.jpg"
    // URL: /files/images/2024/photo.jpg
    return Ok(path);
}

// Minimal API
app.MapGet("/docs/{**path}", (string path) => path);
// จะ match /docs/, /docs/guide, /docs/api/v1/reference
```

### 6.3 Route Values

```csharp
[HttpGet("{category}/{subcategory?}")]
public IActionResult Browse(string category, string? subcategory = null)
{
    // /shop/electronics        → category="electronics", subcategory=null
    // /shop/electronics/phones → category="electronics", subcategory="phones"
    return Ok(new { category, subcategory });
}
```

---

## 7. URL Generation

### 7.1 ใน Controller (Redirect)

```csharp
// RedirectToAction
return RedirectToAction("Index");
return RedirectToAction("Index", "Home");
return RedirectToAction("Details", new { id = product.Id });
return RedirectToAction("Details", "Products", new { id = 5 });

// RedirectToRoute
return RedirectToRoute("default", new { controller = "Products", action = "Index" });

// CreatedAtAction (201)
return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);

// CreatedAtRoute
return CreatedAtRoute("GetProduct", new { id = product.Id }, product);
```

### 7.2 ใน Razor Views (Tag Helpers)

```cshtml
@* asp-controller / asp-action *@
<a asp-controller="Products" asp-action="Details" asp-route-id="5">รายละเอียด</a>
@* → /Products/Details/5 *@

@* asp-route-* สำหรับ route values *@
<a asp-action="Search" 
   asp-route-q="laptop"
   asp-route-category="Electronics"
   asp-route-page="2">ค้นหา</a>
@* → /Products/Search?q=laptop&category=Electronics&page=2 *@

@* asp-route (named route) *@
<a asp-route="GetProduct" asp-route-id="5">Product</a>

@* Form action *@
<form asp-controller="Account" asp-action="Login" method="post">
    ...
</form>
```

### 7.3 IUrlHelper

```csharp
// ใน controller
public class ProductsController : Controller
{
    public IActionResult GetUrl()
    {
        // สร้าง URL
        var url = Url.Action("Details", "Products", new { id = 5 });
        // → /Products/Details/5

        var absoluteUrl = Url.Action("Details", "Products", 
            new { id = 5 }, Request.Scheme);
        // → https://localhost:7001/Products/Details/5

        var routeUrl = Url.RouteUrl("GetProduct", new { id = 5 });
        
        return Ok(new { url, absoluteUrl });
    }
}

// ใน service (inject IHttpContextAccessor + LinkGenerator)
public class NotificationService
{
    private readonly LinkGenerator _linkGenerator;

    public NotificationService(LinkGenerator linkGenerator)
    {
        _linkGenerator = linkGenerator;
    }

    public string GetProductUrl(HttpContext context, int productId)
    {
        return _linkGenerator.GetPathByAction(
            context,
            action: "Details",
            controller: "Products",
            values: new { id = productId }) ?? "/";
    }
}
```

---

## 8. Area Routing

Areas ใช้สำหรับจัดกลุ่ม controllers ที่มี functionality ใกล้เคียงกัน เช่น Admin area

### 8.1 โครงสร้าง Area

```
Areas/
├── Admin/
│   ├── Controllers/
│   │   ├── DashboardController.cs
│   │   └── UsersController.cs
│   └── Views/
│       ├── Dashboard/
│       │   └── Index.cshtml
│       └── Users/
│           └── Index.cshtml
├── Customer/
│   ├── Controllers/
│   │   └── ProfileController.cs
│   └── Views/
│       └── Profile/
│           └── Index.cshtml
Controllers/
Views/
```

### 8.2 Area Controller

```csharp
// Areas/Admin/Controllers/DashboardController.cs
[Area("Admin")]
[Route("admin/[controller]/[action]")]
[Authorize(Roles = "Admin")]
public class DashboardController : Controller
{
    public IActionResult Index() => View();
    public IActionResult Reports() => View();
    public IActionResult Settings() => View();
}

// Areas/Admin/Controllers/UsersController.cs
[Area("Admin")]
[Route("admin/users")]
[Authorize(Roles = "Admin")]
public class UsersController : Controller
{
    [HttpGet("")]
    public IActionResult Index() => View();

    [HttpGet("{id:int}")]
    public IActionResult Details(int id) => View();

    [HttpGet("create")]
    public IActionResult Create() => View();

    [HttpPost("create")]
    [ValidateAntiForgeryToken]
    public IActionResult Create(CreateUserViewModel model) => View();
}
```

### 8.3 ลงทะเบียน Area Routes

```csharp
// ลงทะเบียน area routes ก่อน default route
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Dashboard}/{action=Index}/{id?}");

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// หรือ MapAreaControllerRoute
app.MapAreaControllerRoute(
    name: "admin",
    areaName: "Admin",
    pattern: "admin/{controller=Dashboard}/{action=Index}/{id?}");
```

### 8.4 Links ไปยัง Area

```cshtml
@* ลิงก์ไปยัง Admin area *@
<a asp-area="Admin" asp-controller="Dashboard" asp-action="Index">Admin Dashboard</a>
@* → /admin/Dashboard/Index *@

<a asp-area="Admin" asp-controller="Users" asp-action="Details" asp-route-id="5">
    User 5
</a>

@* ใน area view - กลับไป main area *@
<a asp-area="" asp-controller="Home" asp-action="Index">กลับหน้าหลัก</a>
```

---

## 9. Minimal API Routing

```csharp
// Basic routes
app.MapGet("/", () => "Hello");
app.MapPost("/items", (Item item) => Results.Created($"/items/{item.Id}", item));
app.MapPut("/items/{id}", (int id, Item item) => Results.Ok(item));
app.MapDelete("/items/{id}", (int id) => Results.NoContent());

// Route groups
var v1 = app.MapGroup("/api/v1").WithTags("v1");
var v2 = app.MapGroup("/api/v2").WithTags("v2");

v1.MapGet("/products", () => "Products v1");
v2.MapGet("/products", () => "Products v2");

// Nested groups
var api = app.MapGroup("/api");
var users = api.MapGroup("/users").RequireAuthorization();
var admin = api.MapGroup("/admin").RequireAuthorization("AdminOnly");

users.MapGet("/", () => "Get users");
users.MapGet("/{id:int}", (int id) => $"User {id}");
admin.MapGet("/stats", () => "Admin stats");

// Route constraints ใน Minimal API
app.MapGet("/items/{id:int:min(1)}", (int id) => $"Item {id}");
app.MapGet("/users/{name:alpha:minlength(3)}", (string name) => $"User {name}");

// Catch-all
app.MapGet("/files/{**path}", (string path) => $"File: {path}");
```

---

## 10. โปรแกรมตัวอย่าง: RESTful Routes

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Title = "E-Commerce API",
        Version = "v1"
    });
});

// ลงทะเบียน services
builder.Services.AddScoped<ICategoryService, CategoryService>();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IReviewService, ReviewService>();
builder.Services.AddSingleton<IDataStore, InMemoryDataStore>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();

// Minimal API routes (alternative)
var apiV2 = app.MapGroup("/api/v2").WithTags("v2 (Minimal API)");
apiV2.MapGet("/health", () => new { Status = "OK", Version = "2.0" });

app.Run();

// ============================================
// MODELS
// ============================================

public record Category(int Id, string Name, string? Description, bool IsActive);
public record Product(int Id, string Name, decimal Price, int CategoryId, int Stock, bool IsActive);
public record Review(int Id, int ProductId, int UserId, int Rating, string Comment, DateTime CreatedAt);

// ============================================
// CONTROLLERS
// ============================================

// Categories API
[ApiController]
[Route("api/[controller]")]
public class CategoriesController : ControllerBase
{
    private readonly ICategoryService _service;

    public CategoriesController(ICategoryService service) => _service = service;

    // GET /api/categories
    [HttpGet]
    public async Task<ActionResult<List<Category>>> GetAll()
        => Ok(await _service.GetAllAsync());

    // GET /api/categories/5
    [HttpGet("{id:int}", Name = "GetCategory")]
    public async Task<ActionResult<Category>> GetById(int id)
    {
        var cat = await _service.GetByIdAsync(id);
        return cat is null ? NotFound() : Ok(cat);
    }

    // GET /api/categories/5/products
    [HttpGet("{id:int}/products")]
    public async Task<ActionResult<List<Product>>> GetProducts(int id)
    {
        var cat = await _service.GetByIdAsync(id);
        if (cat is null) return NotFound($"Category {id} not found");
        var products = await _service.GetProductsAsync(id);
        return Ok(products);
    }

    // POST /api/categories
    [HttpPost]
    public async Task<ActionResult<Category>> Create([FromBody] CreateCategoryRequest req)
    {
        var cat = await _service.CreateAsync(req);
        return CreatedAtRoute("GetCategory", new { id = cat.Id }, cat);
    }

    // PUT /api/categories/5
    [HttpPut("{id:int}")]
    public async Task<ActionResult<Category>> Update(int id, [FromBody] UpdateCategoryRequest req)
    {
        var cat = await _service.UpdateAsync(id, req);
        return cat is null ? NotFound() : Ok(cat);
    }

    // DELETE /api/categories/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        return deleted ? NoContent() : NotFound();
    }
}

// Products API with nested routes
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly IReviewService _reviewService;

    public ProductsController(IProductService productService, IReviewService reviewService)
    {
        _productService = productService;
        _reviewService = reviewService;
    }

    // GET /api/products
    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetAll(
        [FromQuery] int? categoryId,
        [FromQuery] decimal? minPrice,
        [FromQuery] decimal? maxPrice,
        [FromQuery] bool? inStock,
        [FromQuery] string? search,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10,
        [FromQuery] string? sortBy = "id",
        [FromQuery] bool desc = false)
    {
        var products = await _productService.GetAllAsync(
            categoryId, minPrice, maxPrice, inStock, search, page, pageSize, sortBy, desc);
        return Ok(products);
    }

    // GET /api/products/5
    [HttpGet("{id:int}", Name = "GetProduct")]
    [ProducesResponseType(typeof(Product), 200)]
    [ProducesResponseType(404)]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        return product is null ? NotFound($"Product {id} not found") : Ok(product);
    }

    // GET /api/products/by-category/electronics
    [HttpGet("by-category/{categoryName:alpha}")]
    public async Task<ActionResult<List<Product>>> GetByCategory(string categoryName)
    {
        var products = await _productService.GetByCategoryNameAsync(categoryName);
        return Ok(products);
    }

    // GET /api/products/price-range?min=100&max=1000
    [HttpGet("price-range")]
    public async Task<ActionResult<List<Product>>> GetByPriceRange(
        [FromQuery] decimal min = 0,
        [FromQuery] decimal max = decimal.MaxValue)
    {
        if (min > max) return BadRequest("min ต้องน้อยกว่าหรือเท่ากับ max");
        var products = await _productService.GetByPriceRangeAsync(min, max);
        return Ok(products);
    }

    // POST /api/products
    [HttpPost]
    [ProducesResponseType(typeof(Product), 201)]
    [ProducesResponseType(typeof(ValidationProblemDetails), 400)]
    public async Task<ActionResult<Product>> Create([FromBody] CreateProductRequest req)
    {
        var product = await _productService.CreateAsync(req);
        return CreatedAtRoute("GetProduct", new { id = product.Id }, product);
    }

    // PUT /api/products/5
    [HttpPut("{id:int}")]
    public async Task<ActionResult<Product>> Update(int id, [FromBody] UpdateProductRequest req)
    {
        var product = await _productService.UpdateAsync(id, req);
        return product is null ? NotFound() : Ok(product);
    }

    // PATCH /api/products/5/stock
    [HttpPatch("{id:int}/stock")]
    public async Task<ActionResult<Product>> UpdateStock(
        int id, [FromBody] UpdateStockRequest req)
    {
        if (req.Quantity < 0) return BadRequest("จำนวน stock ต้องไม่ติดลบ");
        var product = await _productService.UpdateStockAsync(id, req.Quantity);
        return product is null ? NotFound() : Ok(product);
    }

    // PATCH /api/products/5/activate
    [HttpPatch("{id:int}/activate")]
    public async Task<IActionResult> Activate(int id)
    {
        var success = await _productService.SetActiveAsync(id, true);
        return success ? NoContent() : NotFound();
    }

    // PATCH /api/products/5/deactivate
    [HttpPatch("{id:int}/deactivate")]
    public async Task<IActionResult> Deactivate(int id)
    {
        var success = await _productService.SetActiveAsync(id, false);
        return success ? NoContent() : NotFound();
    }

    // DELETE /api/products/5
    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _productService.DeleteAsync(id);
        return deleted ? NoContent() : NotFound();
    }

    // ==========================================
    // Nested Routes: /api/products/{id}/reviews
    // ==========================================

    // GET /api/products/5/reviews
    [HttpGet("{productId:int}/reviews")]
    public async Task<ActionResult<List<Review>>> GetReviews(
        int productId,
        [FromQuery] int? minRating,
        [FromQuery] int? maxRating)
    {
        var product = await _productService.GetByIdAsync(productId);
        if (product is null) return NotFound($"Product {productId} not found");

        var reviews = await _reviewService.GetByProductAsync(productId, minRating, maxRating);
        return Ok(reviews);
    }

    // GET /api/products/5/reviews/10
    [HttpGet("{productId:int}/reviews/{reviewId:int}")]
    public async Task<ActionResult<Review>> GetReview(int productId, int reviewId)
    {
        var review = await _reviewService.GetByIdAsync(reviewId);
        if (review is null || review.ProductId != productId) return NotFound();
        return Ok(review);
    }

    // POST /api/products/5/reviews
    [HttpPost("{productId:int}/reviews")]
    public async Task<ActionResult<Review>> CreateReview(
        int productId, [FromBody] CreateReviewRequest req)
    {
        var product = await _productService.GetByIdAsync(productId);
        if (product is null) return NotFound($"Product {productId} not found");

        if (req.Rating < 1 || req.Rating > 5)
            return BadRequest("Rating ต้องอยู่ระหว่าง 1-5");

        var review = await _reviewService.CreateAsync(productId, req);
        return CreatedAtAction(
            nameof(GetReview),
            new { productId, reviewId = review.Id },
            review);
    }

    // DELETE /api/products/5/reviews/10
    [HttpDelete("{productId:int}/reviews/{reviewId:int}")]
    public async Task<IActionResult> DeleteReview(int productId, int reviewId)
    {
        var review = await _reviewService.GetByIdAsync(reviewId);
        if (review is null || review.ProductId != productId) return NotFound();

        await _reviewService.DeleteAsync(reviewId);
        return NoContent();
    }
}

// ============================================
// REQUEST MODELS
// ============================================

public class CreateCategoryRequest
{
    [Required, StringLength(100)]
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
}

public class UpdateCategoryRequest : CreateCategoryRequest
{
    public bool IsActive { get; set; } = true;
}

public class CreateProductRequest
{
    [Required, StringLength(200)]
    public string Name { get; set; } = string.Empty;
    [Required, Range(0.01, 1000000)]
    public decimal Price { get; set; }
    [Required, Range(1, int.MaxValue)]
    public int CategoryId { get; set; }
    [Range(0, int.MaxValue)]
    public int Stock { get; set; }
}

public class UpdateProductRequest : CreateProductRequest { }

public class UpdateStockRequest
{
    [Range(0, int.MaxValue)]
    public int Quantity { get; set; }
}

public class CreateReviewRequest
{
    [Required, Range(1, 5)]
    public int Rating { get; set; }
    [Required, StringLength(1000, MinimumLength = 10)]
    public string Comment { get; set; } = string.Empty;
    public int UserId { get; set; } = 1; // จะมาจาก auth ใน production
}

// ============================================
// SERVICES (Interfaces)
// ============================================

public interface ICategoryService
{
    Task<List<Category>> GetAllAsync();
    Task<Category?> GetByIdAsync(int id);
    Task<List<Product>> GetProductsAsync(int categoryId);
    Task<Category> CreateAsync(CreateCategoryRequest req);
    Task<Category?> UpdateAsync(int id, UpdateCategoryRequest req);
    Task<bool> DeleteAsync(int id);
}

public interface IProductService
{
    Task<List<Product>> GetAllAsync(int? categoryId, decimal? minPrice, decimal? maxPrice,
        bool? inStock, string? search, int page, int pageSize, string? sortBy, bool desc);
    Task<Product?> GetByIdAsync(int id);
    Task<List<Product>> GetByCategoryNameAsync(string categoryName);
    Task<List<Product>> GetByPriceRangeAsync(decimal min, decimal max);
    Task<Product> CreateAsync(CreateProductRequest req);
    Task<Product?> UpdateAsync(int id, UpdateProductRequest req);
    Task<Product?> UpdateStockAsync(int id, int quantity);
    Task<bool> SetActiveAsync(int id, bool isActive);
    Task<bool> DeleteAsync(int id);
}

public interface IReviewService
{
    Task<List<Review>> GetByProductAsync(int productId, int? minRating, int? maxRating);
    Task<Review?> GetByIdAsync(int id);
    Task<Review> CreateAsync(int productId, CreateReviewRequest req);
    Task<bool> DeleteAsync(int id);
}

// ============================================
// DATA STORE
// ============================================

public interface IDataStore
{
    List<Category> Categories { get; }
    List<Product> Products { get; }
    List<Review> Reviews { get; }
}

public class InMemoryDataStore : IDataStore
{
    public List<Category> Categories { get; } = new()
    {
        new(1, "Electronics", "Electronic products", true),
        new(2, "Clothing", "Fashion and apparel", true),
        new(3, "Books", "Books and magazines", true),
        new(4, "Food", "Food and beverages", true)
    };

    public List<Product> Products { get; } = new()
    {
        new(1, "iPhone 15", 39900m, 1, 50, true),
        new(2, "Samsung TV", 25000m, 1, 20, true),
        new(3, "T-Shirt", 490m, 2, 200, true),
        new(4, "C# Book", 850m, 3, 100, true),
        new(5, "Coffee", 250m, 4, 500, true)
    };

    public List<Review> Reviews { get; } = new()
    {
        new(1, 1, 1, 5, "สินค้าดีมาก", DateTime.UtcNow.AddDays(-10)),
        new(2, 1, 2, 4, "ดีแต่ราคาสูง", DateTime.UtcNow.AddDays(-5)),
        new(3, 2, 1, 3, "ปานกลาง", DateTime.UtcNow.AddDays(-3))
    };
}
```

---

## Exercises

### Exercise 1: Blog Routing
สร้าง routes สำหรับ blog:

```
GET /blog                              → BlogController.Index
GET /blog/{year:int}/{month:int}/{slug} → BlogController.Post
GET /blog/category/{category}          → BlogController.Category
GET /blog/tag/{tag}                    → BlogController.Tag
GET /blog/author/{username:alpha}      → BlogController.Author
POST /blog/{id:int}/comment            → BlogController.AddComment
```

### Exercise 2: Admin Area
สร้าง Admin area ที่มี routes:

```
/admin                → Admin/Dashboard/Index
/admin/users          → Admin/Users/Index
/admin/users/{id}     → Admin/Users/Details
/admin/products       → Admin/Products/Index
/admin/reports        → Admin/Reports/Index
/admin/settings       → Admin/Settings/Index
```

### Exercise 3: API Versioning Routes
สร้าง routes สำหรับ API versioning:

```csharp
// ต้องการ:
// /api/v1/products → ProductsV1Controller
// /api/v2/products → ProductsV2Controller (เพิ่ม features)
// /api/products    → redirect ไป v2

// ใช้ route groups ใน Minimal API
// หรือ Area สำหรับ Controller-based
```

---

## สรุป

✅ Routing จับคู่ HTTP request URL กับ endpoint  
✅ Convention-based routing ใช้ pattern template เดียว map หลาย controllers  
✅ Attribute routing กำหนด routes โดยตรงบน controller/action  
✅ `[Route("api/[controller]")]` ใช้ token `[controller]`, `[action]`, `[area]`  
✅ Route constraints: `:int`, `:alpha`, `:min(1)`, `:regex(...)` ช่วยกรอง values  
✅ `{id?}` optional parameter, `{**path}` catch-all parameter  
✅ `asp-controller`, `asp-action`, `asp-route-*` สร้าง URL ใน Razor views  
✅ Areas ใช้จัดกลุ่ม controllers สำหรับ large applications  
✅ Minimal API ใช้ `app.MapGroup()` สำหรับ organize routes  

---

## Part ถัดไป

ใน **Part 050** เราจะเรียนรู้เรื่อง **REST API เบื้องต้น** อย่างละเอียด:
- REST principles
- HTTP status codes
- JSON serialization
- Controller-based API vs Minimal API
- API versioning เบื้องต้น

---

*Part 049/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
