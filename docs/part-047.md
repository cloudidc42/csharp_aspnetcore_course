# Part 047: MVC - Views และ Razor

## เนื้อหาใน Part นี้
- Razor syntax (@)
- Layouts
- Partial views
- Tag Helpers
- View Components
- HTML Helpers
- โปรแกรมตัวอย่าง: Product list page

---

## 1. Razor View คืออะไร?

Razor เป็น templating engine ของ ASP.NET Core ที่ผสม C# code กับ HTML ไฟล์ Razor มีนามสกุล `.cshtml`

```
Controller → return View(model) → Razor Engine → HTML → Browser
```

---

## 2. Razor Syntax

### 2.1 @ - Razor directive

```cshtml
@* นี่คือ comment ใน Razor *@

@* ใช้ @ เพื่อ insert C# expression *@
<p>วันที่วันนี้: @DateTime.Now.ToString("dd/MM/yyyy")</p>
<p>สวัสดี @Model.UserName!</p>

@* คำนวณ *@
<p>ราคารวม: @(Model.Price * Model.Quantity).ToString("N2")</p>

@* ternary expression *@
<p>สถานะ: @(Model.IsActive ? "ใช้งาน" : "ปิดใช้งาน")</p>
```

### 2.2 @{} - Code Block

```cshtml
@{
    // C# code block
    var greeting = "สวัสดี";
    var today = DateTime.Now;
    var products = Model.Products.Where(p => p.IsActive).ToList();
    
    // กำหนด ViewData
    ViewData["Title"] = "รายการสินค้า";
}

<h1>@greeting</h1>
<p>วันที่: @today.ToShortDateString()</p>
<p>พบสินค้า @products.Count รายการ</p>
```

### 2.3 Control Flow

```cshtml
@* if/else *@
@if (Model.IsAdmin)
{
    <button class="btn btn-danger">ลบ</button>
}
else
{
    <span class="text-muted">ไม่มีสิทธิ์ลบ</span>
}

@* switch *@
@switch (Model.Status)
{
    case "Active":
        <span class="badge bg-success">ใช้งาน</span>
        break;
    case "Inactive":
        <span class="badge bg-secondary">ปิดใช้งาน</span>
        break;
    default:
        <span class="badge bg-warning">ไม่ทราบ</span>
        break;
}

@* for loop *@
@for (int i = 0; i < Model.Items.Count; i++)
{
    <tr>
        <td>@(i + 1)</td>
        <td>@Model.Items[i].Name</td>
    </tr>
}

@* foreach *@
@foreach (var product in Model.Products)
{
    <div class="product-card">
        <h3>@product.Name</h3>
        <p>ราคา: @product.Price.ToString("N2") บาท</p>
    </div>
}

@* while *@
@{
    int count = 0;
}
@while (count < 5)
{
    <p>รายการ @count</p>
    count++;
}
```

### 2.4 @model Directive

```cshtml
@* กำหนด model type สำหรับ view *@
@model ProductListViewModel

@* ใช้ Model (strongly typed) *@
<h1>@Model.Title</h1>
<p>พบสินค้า @Model.TotalCount รายการ</p>

@foreach (var product in Model.Products)
{
    <p>@product.Name - @product.Price.ToString("C")</p>
}
```

### 2.5 @using, @inject Directives

```cshtml
@using MyApp.Models
@using MyApp.Services

@inject IProductService ProductService
@inject IConfiguration Configuration

@{
    var maxItems = Configuration.GetValue<int>("MaxItems");
    var featured = await ProductService.GetFeaturedAsync();
}

<p>Max items: @maxItems</p>
```

### 2.6 HTML Encoding

```cshtml
@* Razor encode HTML automatically (XSS protection) *@
@{
    var userInput = "<script>alert('XSS')</script>";
}

@* Encoded (safe) - แสดงเป็น text ไม่ใช่ HTML *@
<p>@userInput</p>

@* Unencoded (dangerous - ใช้เมื่อแน่ใจว่าปลอดภัย) *@
@Html.Raw(userInput)

@* ตัวอย่าง safe - content ที่ trust ได้ *@
@{
    var trustedHtml = "<strong>Bold text</strong>";
}
@Html.Raw(trustedHtml)
```

---

## 3. Layouts

Layout คือ template หลักที่ Views ต่างๆ share กัน (เหมือน master page)

### 3.1 _Layout.cshtml

```cshtml
@* Views/Shared/_Layout.cshtml *@
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - My App</title>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
    @* RenderSection สำหรับ page-specific CSS *@
    @await RenderSectionAsync("Styles", required: false)
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container-fluid">
                <a class="navbar-brand" asp-controller="Home" asp-action="Index">My App</a>
                <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target=".navbar-collapse">
                    <span class="navbar-toggler-icon"></span>
                </button>
                <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
                    <ul class="navbar-nav flex-grow-1">
                        <li class="nav-item">
                            <a class="nav-link" asp-controller="Home" asp-action="Index">หน้าหลัก</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link" asp-controller="Products" asp-action="Index">สินค้า</a>
                        </li>
                    </ul>
                </div>
            </div>
        </nav>
    </header>

    <div class="container">
        <main role="main" class="pb-3">
            @* เนื้อหาของแต่ละ page จะถูก render ที่นี่ *@
            @RenderBody()
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; @DateTime.Now.Year - My App
        </div>
    </footer>

    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>
    @* RenderSection สำหรับ page-specific scripts *@
    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

### 3.2 _ViewStart.cshtml

```cshtml
@* Views/_ViewStart.cshtml - กำหนด default layout *@
@{
    Layout = "_Layout";
}
```

### 3.3 View ที่ใช้ Layout

```cshtml
@* Views/Products/Index.cshtml *@
@model ProductListViewModel

@* กำหนด Layout (หรือ inherit จาก _ViewStart) *@
@{
    Layout = "_Layout";  // หรือไม่ต้องกำหนดถ้า _ViewStart ตั้งไว้แล้ว
    ViewData["Title"] = "รายการสินค้า";
}

@* Optional sections *@
@section Styles {
    <link rel="stylesheet" href="~/css/products.css" />
}

<h1 class="display-4">@Model.Title</h1>
<p>พบสินค้า @Model.TotalCount รายการ</p>

@* เนื้อหาหลัก *@
<div class="row">
    @foreach (var product in Model.Products)
    {
        <div class="col-md-4">
            <div class="card">
                <div class="card-body">
                    <h5 class="card-title">@product.Name</h5>
                    <p class="card-text">@product.Description</p>
                    <p class="card-text">
                        <strong>ราคา: @product.Price.ToString("N2") บาท</strong>
                    </p>
                    <a asp-action="Details" asp-route-id="@product.Id" 
                       class="btn btn-primary">ดูรายละเอียด</a>
                </div>
            </div>
        </div>
    }
</div>

@section Scripts {
    <script src="~/js/products.js"></script>
    <script>
        console.log('Products loaded: @Model.TotalCount');
    </script>
}
```

### 3.4 Nested Layouts

```cshtml
@* Views/Shared/_AdminLayout.cshtml - Layout สำหรับ admin *@
@{
    Layout = "_Layout";  // Inherit จาก main layout
}

<div class="admin-sidebar">
    <ul>
        <li><a asp-controller="Admin" asp-action="Dashboard">Dashboard</a></li>
        <li><a asp-controller="Admin" asp-action="Users">Users</a></li>
        <li><a asp-controller="Admin" asp-action="Reports">Reports</a></li>
    </ul>
</div>

<div class="admin-content">
    @RenderBody()
</div>

@* Admin view ใช้ admin layout *@
@{
    Layout = "_AdminLayout";
}
```

---

## 4. Partial Views

Partial views คือ views ที่สามารถ reuse ได้ใน views อื่น

### 4.1 สร้าง Partial View

```cshtml
@* Views/Shared/_ProductCard.cshtml *@
@model ProductCardViewModel

<div class="card h-100 shadow-sm">
    @if (Model.ImageUrl != null)
    {
        <img src="@Model.ImageUrl" class="card-img-top" alt="@Model.Name">
    }
    <div class="card-body">
        <h5 class="card-title">@Model.Name</h5>
        <p class="card-text text-muted small">@Model.Category</p>
        <p class="card-text">@Model.Description</p>
    </div>
    <div class="card-footer d-flex justify-content-between align-items-center">
        <span class="fs-5 fw-bold text-primary">@Model.Price.ToString("N2") ฿</span>
        <div>
            @if (Model.Stock > 0)
            {
                <button class="btn btn-primary btn-sm" 
                        onclick="addToCart(@Model.Id)">
                    เพิ่มในตะกร้า
                </button>
            }
            else
            {
                <span class="badge bg-secondary">สินค้าหมด</span>
            }
        </div>
    </div>
</div>
```

### 4.2 ใช้ Partial View

```cshtml
@* ใช้ Tag Helper *@
<partial name="_ProductCard" model="@product" />

@* ใช้ HTML Helper *@
@await Html.PartialAsync("_ProductCard", product)
@Html.Partial("_ProductCard", product)

@* ใช้ RenderPartialAsync (ดีกว่าสำหรับ performance) *@
@{ await Html.RenderPartialAsync("_ProductCard", product); }

@* ตัวอย่างใน foreach *@
@foreach (var product in Model.Products)
{
    <div class="col-md-4 mb-4">
        <partial name="_ProductCard" model="@new ProductCardViewModel { 
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            Stock = product.Stock,
            Category = product.Category
        }" />
    </div>
}
```

### 4.3 Passing ViewData ไปยัง Partial

```cshtml
@* ส่ง ViewData เพิ่มเติม *@
<partial name="_Pagination" 
         model="@Model.Pagination" 
         view-data="@(new ViewDataDictionary(ViewData) { { "Action", "Index" } })" />
```

---

## 5. Tag Helpers

Tag Helpers ช่วยให้ HTML cleaner กว่า HTML Helpers

### 5.1 Built-in Tag Helpers

```cshtml
@* Anchor Tag Helper *@
<a asp-controller="Products" asp-action="Details" asp-route-id="@product.Id">
    @product.Name
</a>
@* แปลงเป็น: <a href="/Products/Details/5">Product Name</a> *@

@* Form Tag Helper *@
<form asp-controller="Products" asp-action="Create" method="post">
    @Html.AntiForgeryToken()
    
    @* Label Tag Helper *@
    <label asp-for="Name" class="form-label">ชื่อสินค้า</label>
    
    @* Input Tag Helper *@
    <input asp-for="Name" class="form-control" placeholder="ชื่อสินค้า" />
    
    @* Validation Tag Helper *@
    <span asp-validation-for="Name" class="text-danger"></span>
    
    @* Select Tag Helper *@
    <select asp-for="CategoryId" asp-items="@Model.Categories" class="form-select">
        <option value="">-- เลือกหมวดหมู่ --</option>
    </select>
    
    @* Textarea Tag Helper *@
    <textarea asp-for="Description" class="form-control" rows="4"></textarea>
    
    <button type="submit" class="btn btn-primary">บันทึก</button>
</form>
```

### 5.2 Environment Tag Helper

```cshtml
@* แสดงเฉพาะใน Development *@
<environment include="Development">
    <link rel="stylesheet" href="~/css/site.css" />
</environment>

@* แสดงใน Production และ Staging *@
<environment exclude="Development">
    <link rel="stylesheet" 
          href="https://cdn.example.com/bootstrap.min.css"
          asp-fallback-href="~/lib/bootstrap/dist/css/bootstrap.min.css"
          asp-fallback-test-class="sr-only"
          asp-fallback-test-property="position"
          asp-fallback-test-value="absolute" />
</environment>
```

### 5.3 Cache Tag Helper

```cshtml
@* Cache content เป็นเวลา 10 นาที *@
<cache expires-after="@TimeSpan.FromMinutes(10)">
    <p>เวลาที่ cache: @DateTime.Now.ToLongTimeString()</p>
    @* Expensive content ที่ต้องการ cache *@
    @await Component.InvokeAsync("FeaturedProducts")
</cache>

@* Cache ตาม condition *@
<cache enabled="@(User.IsInRole("Admin") == false)"
       expires-after="@TimeSpan.FromMinutes(5)">
    @* content *@
</cache>
```

### 5.4 Image Tag Helper

```cshtml
@* เพิ่ม cache-busting hash ใน URL *@
<img src="~/images/logo.png" 
     asp-append-version="true" 
     alt="Logo" />
@* แปลงเป็น: <img src="/images/logo.png?v=abc123" alt="Logo" /> *@
```

### 5.5 Script/Link Tag Helper

```cshtml
@* Version query string สำหรับ cache busting *@
<link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
<script src="~/js/site.js" asp-append-version="true"></script>
```

---

## 6. View Components

View Components มีประสิทธิภาพมากกว่า Partial Views เพราะมี business logic แยกออกมา

### 6.1 สร้าง View Component

```csharp
// ViewComponents/CartSummaryViewComponent.cs
using Microsoft.AspNetCore.Mvc;

public class CartSummaryViewComponent : ViewComponent
{
    private readonly ICartService _cartService;

    public CartSummaryViewComponent(ICartService cartService)
    {
        _cartService = cartService;
    }

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var cartItems = await _cartService.GetCartItemsAsync(HttpContext);
        var viewModel = new CartSummaryViewModel
        {
            ItemCount = cartItems.Count,
            TotalAmount = cartItems.Sum(i => i.Price * i.Quantity)
        };
        return View(viewModel);
    }
}

// View Component View: Views/Shared/Components/CartSummary/Default.cshtml
```

### 6.2 View Component View

```cshtml
@* Views/Shared/Components/CartSummary/Default.cshtml *@
@model CartSummaryViewModel

<div class="cart-summary" id="cart-summary">
    <a href="/cart" class="btn btn-outline-primary">
        🛒 ตะกร้า
        @if (Model.ItemCount > 0)
        {
            <span class="badge bg-danger">@Model.ItemCount</span>
        }
        <span class="cart-total ms-2">@Model.TotalAmount.ToString("N2") ฿</span>
    </a>
</div>
```

### 6.3 ใช้ View Component

```cshtml
@* ใน view *@
@await Component.InvokeAsync("CartSummary")

@* Tag Helper syntax (ต้อง add to _ViewImports.cshtml) *@
<vc:cart-summary />

@* พร้อม parameters *@
@await Component.InvokeAsync("ProductList", new { category = "Electronics", count = 6 })
<vc:product-list category="Electronics" count="6" />
```

### 6.4 View Component พร้อม Parameters

```csharp
// ViewComponents/ProductListViewComponent.cs
public class ProductListViewComponent : ViewComponent
{
    private readonly IProductService _productService;

    public ProductListViewComponent(IProductService productService)
    {
        _productService = productService;
    }

    public async Task<IViewComponentResult> InvokeAsync(
        string? category = null, 
        int count = 6,
        string? sortBy = null)
    {
        var products = await _productService.GetFeaturedAsync(category, count, sortBy);
        return View(products);
    }
}
```

---

## 7. HTML Helpers

```cshtml
@* HTML Helpers - เก่ากว่า Tag Helpers แต่ยังใช้ได้ *@

@* BeginForm *@
@using (Html.BeginForm("Create", "Products", FormMethod.Post, new { @class = "product-form" }))
{
    @Html.AntiForgeryToken()
    
    <div class="form-group">
        @Html.LabelFor(m => m.Name, "ชื่อสินค้า")
        @Html.TextBoxFor(m => m.Name, new { @class = "form-control" })
        @Html.ValidationMessageFor(m => m.Name, "", new { @class = "text-danger" })
    </div>
    
    <div class="form-group">
        @Html.LabelFor(m => m.Price, "ราคา")
        @Html.TextBoxFor(m => m.Price, "{0:N2}", new { @class = "form-control", type = "number", step = "0.01" })
        @Html.ValidationMessageFor(m => m.Price, "", new { @class = "text-danger" })
    </div>
    
    <div class="form-group">
        @Html.LabelFor(m => m.CategoryId, "หมวดหมู่")
        @Html.DropDownListFor(m => m.CategoryId, Model.Categories, "-- เลือกหมวดหมู่ --", 
            new { @class = "form-select" })
    </div>
    
    <input type="submit" value="บันทึก" class="btn btn-primary" />
}
```

---

## 8. _ViewImports.cshtml

```cshtml
@* Views/_ViewImports.cshtml - shared imports สำหรับทุก views *@
@using MyApp
@using MyApp.Models
@using MyApp.ViewModels
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, MyApp  @* เพิ่ม custom tag helpers *@
```

---

## 9. โปรแกรมตัวอย่าง: Product List Page

### Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม MVC
builder.Services.AddControllersWithViews();

// Register services
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddSingleton<IProductRepository, InMemoryProductRepository>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
    app.UseExceptionHandler("/Home/Error");

app.UseStaticFiles();
app.UseRouting();
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

### ProductsController.cs (MVC)

```csharp
public class ProductsController : Controller
{
    private readonly IProductService _service;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(IProductService service, ILogger<ProductsController> logger)
    {
        _service = service;
        _logger = logger;
    }

    // GET /Products
    public async Task<IActionResult> Index(
        string? search, string? category, int page = 1)
    {
        var filter = new ProductFilter { Search = search, Category = category, Page = page };
        var result = await _service.GetAllAsync(filter);
        var categories = await _service.GetCategoriesAsync();

        var viewModel = new ProductListViewModel
        {
            Products = result.Data.Select(p => new ProductCardViewModel
            {
                Id = p.Id, Name = p.Name, Description = p.Description,
                Price = p.Price, Stock = p.Stock, Category = p.Category
            }).ToList(),
            Categories = categories,
            CurrentCategory = category,
            SearchTerm = search,
            TotalCount = result.TotalCount,
            CurrentPage = page,
            TotalPages = result.TotalPages
        };

        return View(viewModel);
    }

    // GET /Products/Details/5
    public async Task<IActionResult> Details(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null) return NotFound();
        return View(product);
    }

    // GET /Products/Create
    public async Task<IActionResult> Create()
    {
        var categories = await _service.GetCategoriesAsync();
        var viewModel = new CreateProductViewModel
        {
            Categories = categories.Select(c => new SelectListItem(c, c)).ToList()
        };
        return View(viewModel);
    }

    // POST /Products/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(CreateProductViewModel model)
    {
        if (!ModelState.IsValid)
        {
            model.Categories = (await _service.GetCategoriesAsync())
                .Select(c => new SelectListItem(c, c)).ToList();
            return View(model);
        }

        var request = new CreateProductRequest
        {
            Name = model.Name,
            Description = model.Description,
            Price = model.Price,
            Stock = model.Stock,
            Category = model.Category,
            Sku = model.Sku
        };

        var product = await _service.CreateAsync(request);
        TempData["Success"] = $"สร้างสินค้า '{product.Name}' สำเร็จ";
        return RedirectToAction(nameof(Index));
    }

    // GET /Products/Edit/5
    public async Task<IActionResult> Edit(int id)
    {
        var product = await _service.GetByIdAsync(id);
        if (product is null) return NotFound();

        var categories = await _service.GetCategoriesAsync();
        var viewModel = new EditProductViewModel
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            Price = product.Price,
            Stock = product.Stock,
            Category = product.Category,
            Categories = categories.Select(c => new SelectListItem(c, c)).ToList()
        };
        return View(viewModel);
    }

    // POST /Products/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(int id, EditProductViewModel model)
    {
        if (id != model.Id) return BadRequest();
        if (!ModelState.IsValid)
        {
            model.Categories = (await _service.GetCategoriesAsync())
                .Select(c => new SelectListItem(c, c)).ToList();
            return View(model);
        }

        var request = new UpdateProductRequest
        {
            Name = model.Name,
            Description = model.Description,
            Price = model.Price,
            Stock = model.Stock,
            Category = model.Category
        };

        var updated = await _service.UpdateAsync(id, request);
        if (updated is null) return NotFound();

        TempData["Success"] = $"แก้ไขสินค้า '{updated.Name}' สำเร็จ";
        return RedirectToAction(nameof(Index));
    }
}
```

### Views/Products/Index.cshtml

```cshtml
@model ProductListViewModel
@{
    ViewData["Title"] = "รายการสินค้า";
}

@* Flash message *@
@if (TempData["Success"] != null)
{
    <div class="alert alert-success alert-dismissible fade show" role="alert">
        @TempData["Success"]
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
}

<div class="d-flex justify-content-between align-items-center mb-4">
    <h1 class="h2">รายการสินค้า</h1>
    <a asp-action="Create" class="btn btn-primary">+ เพิ่มสินค้า</a>
</div>

@* Search and Filter *@
<form asp-action="Index" method="get" class="mb-4">
    <div class="row g-3">
        <div class="col-md-6">
            <div class="input-group">
                <input type="text" name="search" value="@Model.SearchTerm"
                       class="form-control" placeholder="ค้นหาสินค้า..." />
                <button type="submit" class="btn btn-outline-secondary">ค้นหา</button>
            </div>
        </div>
        <div class="col-md-4">
            <select name="category" class="form-select" onchange="this.form.submit()">
                <option value="">-- ทุกหมวดหมู่ --</option>
                @foreach (var category in Model.Categories)
                {
                    <option value="@category" selected="@(Model.CurrentCategory == category)">
                        @category
                    </option>
                }
            </select>
        </div>
        @if (!string.IsNullOrWhiteSpace(Model.SearchTerm) || !string.IsNullOrWhiteSpace(Model.CurrentCategory))
        {
            <div class="col-md-2">
                <a asp-action="Index" class="btn btn-outline-danger w-100">ล้างตัวกรอง</a>
            </div>
        }
    </div>
</form>

<p class="text-muted">พบสินค้า @Model.TotalCount รายการ</p>

@* Product Cards *@
@if (Model.Products.Any())
{
    <div class="row row-cols-1 row-cols-md-3 g-4 mb-4">
        @foreach (var product in Model.Products)
        {
            <div class="col">
                <partial name="_ProductCard" model="product" />
            </div>
        }
    </div>

    @* Pagination *@
    @if (Model.TotalPages > 1)
    {
        <nav aria-label="Page navigation">
            <ul class="pagination justify-content-center">
                <li class="page-item @(Model.CurrentPage == 1 ? "disabled" : "")">
                    <a class="page-link" asp-action="Index"
                       asp-route-page="@(Model.CurrentPage - 1)"
                       asp-route-search="@Model.SearchTerm"
                       asp-route-category="@Model.CurrentCategory">
                        ก่อนหน้า
                    </a>
                </li>

                @for (int i = 1; i <= Model.TotalPages; i++)
                {
                    <li class="page-item @(i == Model.CurrentPage ? "active" : "")">
                        <a class="page-link" asp-action="Index"
                           asp-route-page="@i"
                           asp-route-search="@Model.SearchTerm"
                           asp-route-category="@Model.CurrentCategory">@i</a>
                    </li>
                }

                <li class="page-item @(Model.CurrentPage == Model.TotalPages ? "disabled" : "")">
                    <a class="page-link" asp-action="Index"
                       asp-route-page="@(Model.CurrentPage + 1)"
                       asp-route-search="@Model.SearchTerm"
                       asp-route-category="@Model.CurrentCategory">
                        ถัดไป
                    </a>
                </li>
            </ul>
        </nav>
    }
}
else
{
    <div class="text-center py-5">
        <p class="text-muted fs-5">ไม่พบสินค้าที่ตรงกับเงื่อนไข</p>
        <a asp-action="Create" class="btn btn-primary">เพิ่มสินค้าใหม่</a>
    </div>
}
```

### Views/Shared/_ProductCard.cshtml

```cshtml
@model ProductCardViewModel

<div class="card h-100 shadow-sm hover-shadow">
    <div class="card-body">
        <div class="d-flex justify-content-between align-items-start">
            <h5 class="card-title mb-1">@Model.Name</h5>
            @if (Model.Stock == 0)
            {
                <span class="badge bg-danger">หมด</span>
            }
            else if (Model.Stock <= 5)
            {
                <span class="badge bg-warning text-dark">เหลือน้อย</span>
            }
        </div>
        <p class="text-muted small mb-2">@Model.Category</p>
        <p class="card-text text-truncate-3">@Model.Description</p>
    </div>
    <div class="card-footer bg-transparent d-flex justify-content-between align-items-center">
        <span class="fs-5 fw-bold text-primary">@Model.Price.ToString("N0") ฿</span>
        <div class="btn-group">
            <a asp-controller="Products" asp-action="Details" asp-route-id="@Model.Id"
               class="btn btn-sm btn-outline-secondary">รายละเอียด</a>
            <a asp-controller="Products" asp-action="Edit" asp-route-id="@Model.Id"
               class="btn btn-sm btn-outline-primary">แก้ไข</a>
        </div>
    </div>
</div>
```

### ViewModels

```csharp
// ViewModels/ProductListViewModel.cs
public class ProductListViewModel
{
    public List<ProductCardViewModel> Products { get; set; } = [];
    public List<string> Categories { get; set; } = [];
    public string? CurrentCategory { get; set; }
    public string? SearchTerm { get; set; }
    public int TotalCount { get; set; }
    public int CurrentPage { get; set; } = 1;
    public int TotalPages { get; set; }
    public bool HasNextPage => CurrentPage < TotalPages;
    public bool HasPreviousPage => CurrentPage > 1;
}

public class ProductCardViewModel
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public string Category { get; set; } = string.Empty;
    public string? ImageUrl { get; set; }
}

public class CreateProductViewModel
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

    [Required, RegularExpression(@"^[A-Z]{2}[0-9]{4}$")]
    public string Sku { get; set; } = string.Empty;

    public List<SelectListItem> Categories { get; set; } = [];
}

public class EditProductViewModel : CreateProductViewModel
{
    public int Id { get; set; }
}
```

---

## Exercises

### Exercise 1: Product Details Page
สร้างหน้า Details สำหรับแสดงรายละเอียดสินค้า:

```cshtml
@* ต้องมี:
- แสดงรายละเอียดสินค้าทั้งหมด
- Breadcrumb navigation
- Related products section (ใช้ View Component)
- Share buttons
- Back to list button
*@
```

### Exercise 2: Create/Edit Form
สร้าง form สำหรับ Create และ Edit product:

```cshtml
@* ต้องมี:
- ใช้ Tag Helpers ทั้งหมด
- Client-side validation
- Image preview เมื่อ input URL
- Character counter สำหรับ description
- Submit button พร้อม loading state
*@
```

### Exercise 3: Admin Layout
สร้าง admin layout แยกจาก main layout:

```cshtml
@* ต้องมี:
- Sidebar navigation
- Breadcrumb
- User info ใน header
- Statistics section (ใช้ View Component)
*@
```

---

## สรุป

✅ Razor syntax ใช้ `@` สำหรับ insert C# expressions และ `@{}` สำหรับ code blocks  
✅ Layout ใช้ `@RenderBody()` และ `@RenderSection()` สำหรับ define structure  
✅ `_ViewStart.cshtml` กำหนด default layout สำหรับทุก views  
✅ Partial views ใช้ `<partial name="..." model="..." />` หรือ `@await Html.PartialAsync()`  
✅ Tag Helpers (`asp-for`, `asp-controller`, `asp-action`) ทำให้ HTML สะอาดกว่า HTML Helpers  
✅ View Components มีประสิทธิภาพมากกว่า Partial Views เพราะมี business logic แยกออกมา  
✅ `_ViewImports.cshtml` ใช้ import namespaces และ add Tag Helpers  
✅ `TempData` ใช้ส่ง flash messages ระหว่าง redirects  

---

## Part ถัดไป

ใน **Part 048** เราจะเรียนรู้เรื่อง **MVC: Models และ ViewModels** อย่างละเอียด:
- Model classes
- Data Annotations validation
- ViewModel pattern
- Model validation
- Fluent Validation เบื้องต้น

---

*Part 047/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
