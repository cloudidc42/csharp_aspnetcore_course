# Part 041: ASP.NET Core - เริ่มต้นโปรเจค

## เนื้อหาใน Part นี้
- การสร้างโปรเจค ASP.NET Core ด้วย `dotnet new`
- โครงสร้างโปรเจค ASP.NET Core
- ไฟล์ launchSettings.json
- ไฟล์ appsettings.json
- การรัน Development Server
- Swagger/OpenAPI
- โปรแกรมตัวอย่าง: สร้าง API โปรเจคแรก

---

## 1. ASP.NET Core คืออะไร?

ASP.NET Core คือ Framework สำหรับสร้าง Web Application และ Web API บนแพลตฟอร์ม .NET ที่พัฒนาโดย Microsoft มีคุณสมบัติเด่นดังนี้:

- **Cross-platform**: รันได้บน Windows, Linux, macOS
- **High Performance**: เร็วกว่า ASP.NET เดิมหลายเท่า
- **Open Source**: พัฒนาแบบ Open Source บน GitHub
- **Modular**: เลือกใช้ middleware เฉพาะที่ต้องการ
- **Dependency Injection**: มี DI container ในตัว
- **Minimal API**: รองรับการสร้าง API แบบ minimal (ตั้งแต่ .NET 6)

```
ASP.NET Core สถาปัตยกรรม:
┌─────────────────────────────────────────┐
│              Client (Browser/App)        │
└─────────────────┬───────────────────────┘
                  │ HTTP Request
┌─────────────────▼───────────────────────┐
│           Kestrel Web Server            │
├─────────────────────────────────────────┤
│          Middleware Pipeline            │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │   Auth   │ │ Routing  │ │  MVC    │ │
│  └──────────┘ └──────────┘ └─────────┘ │
├─────────────────────────────────────────┤
│         Business Logic / Services       │
├─────────────────────────────────────────┤
│         Data Access / Database          │
└─────────────────────────────────────────┘
```

---

## 2. การติดตั้ง .NET SDK

ก่อนเริ่มต้น ต้องติดตั้ง .NET SDK ก่อน:

```bash
# ตรวจสอบเวอร์ชัน .NET ที่ติดตั้ง
dotnet --version

# แสดง SDK ทั้งหมดที่ติดตั้ง
dotnet --list-sdks

# แสดง Runtime ทั้งหมด
dotnet --list-runtimes
```

ผลลัพธ์ที่คาดหวัง:
```
9.0.100
```

---

## 3. การสร้างโปรเจค ASP.NET Core

### 3.1 dotnet new webapi

สร้างโปรเจค Web API:

```bash
# สร้าง Web API project (Minimal API style)
dotnet new webapi -n MyFirstApi

# สร้าง Web API แบบ Controller-based
dotnet new webapi -n MyFirstApi --use-controllers

# สร้าง Web API โดยไม่มี HTTPS
dotnet new webapi -n MyFirstApi --no-https

# ดู template ที่มีทั้งหมด
dotnet new list
```

### 3.2 dotnet new mvc

สร้างโปรเจค MVC (Model-View-Controller):

```bash
# สร้าง MVC project
dotnet new mvc -n MyMvcApp

# สร้าง MVC พร้อม Authentication
dotnet new mvc -n MyMvcApp --auth Individual

# สร้าง Razor Pages project
dotnet new webapp -n MyWebApp
```

### 3.3 Template ที่ใช้บ่อย

| Template | Short Name | คำอธิบาย |
|----------|-----------|----------|
| ASP.NET Core Web App (Razor Pages) | webapp | Web App แบบ Razor Pages |
| ASP.NET Core Web App (MVC) | mvc | Web App แบบ MVC |
| ASP.NET Core Web API | webapi | REST API |
| ASP.NET Core Minimal API | webapi | Minimal API style |
| Blazor Web App | blazor | Blazor Application |
| gRPC Service | grpc | gRPC Service |

---

## 4. โครงสร้างโปรเจค ASP.NET Core

หลังจากสร้างโปรเจค Web API จะได้โครงสร้างดังนี้:

### 4.1 โครงสร้าง Minimal API (dotnet new webapi)

```
MyFirstApi/
├── MyFirstApi.csproj          # Project file
├── Program.cs                  # Entry point + Application setup
├── appsettings.json           # Configuration
├── appsettings.Development.json  # Dev-specific config
├── Properties/
│   └── launchSettings.json    # Launch profiles
└── obj/                       # Build output (auto-generated)
    └── ...
```

### 4.2 โครงสร้าง MVC (dotnet new mvc)

```
MyMvcApp/
├── MyMvcApp.csproj
├── Program.cs
├── appsettings.json
├── appsettings.Development.json
├── Controllers/               # Controller classes
│   └── HomeController.cs
├── Models/                    # Data models
│   └── ErrorViewModel.cs
├── Views/                     # Razor views
│   ├── Home/
│   │   ├── Index.cshtml
│   │   └── Privacy.cshtml
│   ├── Shared/
│   │   ├── _Layout.cshtml
│   │   └── _ValidationScriptsPartial.cshtml
│   ├── _ViewImports.cshtml
│   └── _ViewStart.cshtml
├── wwwroot/                   # Static files
│   ├── css/
│   ├── js/
│   └── lib/
└── Properties/
    └── launchSettings.json
```

### 4.3 ไฟล์ .csproj

```xml
<!-- MyFirstApi.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="9.0.0" />
  </ItemGroup>

</Project>
```

**คำอธิบาย:**
- `TargetFramework`: เวอร์ชัน .NET ที่ใช้
- `Nullable`: เปิดใช้ Nullable Reference Types
- `ImplicitUsings`: เพิ่ม using statements อัตโนมัติ
- `PackageReference`: NuGet packages ที่ต้องการ

---

## 5. launchSettings.json

ไฟล์นี้กำหนดวิธีการ launch application ในระหว่าง development:

```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:7001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

**คำอธิบายแต่ละ Profile:**

| ฟิลด์ | คำอธิบาย |
|-------|----------|
| `commandName` | วิธีการ run (Project, IISExpress, Executable) |
| `launchBrowser` | เปิด browser อัตโนมัติหรือไม่ |
| `launchUrl` | URL ที่เปิดเมื่อ launch |
| `applicationUrl` | URL ที่ application รัน |
| `environmentVariables` | Environment variables ที่ตั้งค่า |

---

## 6. appsettings.json

ไฟล์ configuration หลักของ application:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb;Trusted_Connection=True;"
  },
  "AppSettings": {
    "ApiKey": "your-api-key-here",
    "MaxItemsPerPage": 50,
    "EnableFeatureX": true
  }
}
```

### appsettings.Development.json

การตั้งค่าเฉพาะสำหรับ Development environment:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information",
      "Microsoft.AspNetCore": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb_Dev;Trusted_Connection=True;"
  }
}
```

**หมายเหตุ**: ค่าใน `appsettings.Development.json` จะ override ค่าใน `appsettings.json` เมื่อรันใน Development environment

---

## 7. การรัน Development Server

```bash
# รัน development server
dotnet run

# รันพร้อมระบุ profile
dotnet run --launch-profile https

# รันแบบ watch (auto-reload เมื่อโค้ดเปลี่ยน)
dotnet watch run

# รันบน port ที่กำหนด
dotnet run --urls "http://localhost:3000"

# Build โปรเจค
dotnet build

# Publish โปรเจค
dotnet publish -c Release -o ./publish
```

### ผลลัพธ์เมื่อรัน

```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:7001
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /path/to/MyFirstApi
```

---

## 8. Swagger/OpenAPI

Swagger UI ช่วยให้สามารถทดสอบ API ได้โดยตรงจาก browser

### 8.1 การตั้งค่า Swagger ใน Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม services
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My First API",
        Version = "v1",
        Description = "API สำหรับ demo ใน Part 041",
        Contact = new OpenApiContact
        {
            Name = "Developer",
            Email = "dev@example.com"
        }
    });
});

var app = builder.Build();

// ใช้ Swagger เฉพาะ Development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1");
        options.RoutePrefix = string.Empty; // เปิด Swagger ที่ root URL
    });
}

app.Run();
```

### 8.2 เข้าถึง Swagger UI

เมื่อรันในโหมด Development:
- Swagger UI: `https://localhost:7001/swagger`
- OpenAPI JSON: `https://localhost:7001/swagger/v1/swagger.json`

### 8.3 ตัวอย่าง Swagger ที่สมบูรณ์

```csharp
// เพิ่ม XML documentation
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Products API",
        Version = "v1"
    });

    // เพิ่ม XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);

    // เพิ่ม JWT Authentication ใน Swagger
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey
    });
});
```

---

## 9. โปรแกรมตัวอย่าง: สร้าง API โปรเจคแรก

มาสร้าง API โปรเจคแรกกัน! เราจะสร้าง API สำหรับจัดการรายการหนังสือ

### ขั้นตอนที่ 1: สร้างโปรเจค

```bash
dotnet new webapi -n BookApi
cd BookApi
dotnet add package Microsoft.AspNetCore.OpenApi
```

### ขั้นตอนที่ 2: สร้าง Model

สร้างไฟล์ `Models/Book.cs`:

```csharp
namespace BookApi.Models;

/// <summary>
/// Model สำหรับหนังสือ
/// </summary>
public class Book
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Author { get; set; } = string.Empty;
    public string ISBN { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int PublishedYear { get; set; }
    public bool IsAvailable { get; set; } = true;
}

/// <summary>
/// Request model สำหรับสร้าง/แก้ไขหนังสือ
/// </summary>
public class BookRequest
{
    public string Title { get; set; } = string.Empty;
    public string Author { get; set; } = string.Empty;
    public string ISBN { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int PublishedYear { get; set; }
}
```

### ขั้นตอนที่ 3: สร้าง Program.cs

```csharp
// Program.cs
using BookApi.Models;
using Microsoft.AspNetCore.OpenApi;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

// === Services Configuration ===
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Book API",
        Version = "v1",
        Description = "API สำหรับจัดการรายการหนังสือ"
    });
});

var app = builder.Build();

// === Middleware Configuration ===
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// === In-Memory Data Store ===
var books = new List<Book>
{
    new Book { Id = 1, Title = "C# Programming", Author = "Andrew Troelsen", 
               ISBN = "978-1484268490", Price = 850m, PublishedYear = 2022 },
    new Book { Id = 2, Title = "ASP.NET Core in Action", Author = "Andrew Lock", 
               ISBN = "978-1617298301", Price = 920m, PublishedYear = 2023 },
    new Book { Id = 3, Title = "Clean Code", Author = "Robert C. Martin", 
               ISBN = "978-0132350884", Price = 650m, PublishedYear = 2008 }
};

int nextId = 4;

// === API Endpoints ===

// GET /api/books - ดึงหนังสือทั้งหมด
app.MapGet("/api/books", () =>
{
    return Results.Ok(books);
})
.WithName("GetAllBooks")
.WithSummary("ดึงรายการหนังสือทั้งหมด")
.WithTags("Books");

// GET /api/books/{id} - ดึงหนังสือตาม ID
app.MapGet("/api/books/{id:int}", (int id) =>
{
    var book = books.FirstOrDefault(b => b.Id == id);
    return book is null
        ? Results.NotFound(new { Message = $"ไม่พบหนังสือ ID: {id}" })
        : Results.Ok(book);
})
.WithName("GetBookById")
.WithSummary("ดึงหนังสือตาม ID")
.WithTags("Books");

// POST /api/books - สร้างหนังสือใหม่
app.MapPost("/api/books", (BookRequest request) =>
{
    if (string.IsNullOrWhiteSpace(request.Title))
    {
        return Results.BadRequest(new { Message = "กรุณาระบุชื่อหนังสือ" });
    }

    var newBook = new Book
    {
        Id = nextId++,
        Title = request.Title,
        Author = request.Author,
        ISBN = request.ISBN,
        Price = request.Price,
        PublishedYear = request.PublishedYear
    };

    books.Add(newBook);
    return Results.Created($"/api/books/{newBook.Id}", newBook);
})
.WithName("CreateBook")
.WithSummary("สร้างหนังสือใหม่")
.WithTags("Books");

// PUT /api/books/{id} - แก้ไขหนังสือ
app.MapPut("/api/books/{id:int}", (int id, BookRequest request) =>
{
    var book = books.FirstOrDefault(b => b.Id == id);
    if (book is null)
    {
        return Results.NotFound(new { Message = $"ไม่พบหนังสือ ID: {id}" });
    }

    book.Title = request.Title;
    book.Author = request.Author;
    book.ISBN = request.ISBN;
    book.Price = request.Price;
    book.PublishedYear = request.PublishedYear;

    return Results.Ok(book);
})
.WithName("UpdateBook")
.WithSummary("แก้ไขข้อมูลหนังสือ")
.WithTags("Books");

// DELETE /api/books/{id} - ลบหนังสือ
app.MapDelete("/api/books/{id:int}", (int id) =>
{
    var book = books.FirstOrDefault(b => b.Id == id);
    if (book is null)
    {
        return Results.NotFound(new { Message = $"ไม่พบหนังสือ ID: {id}" });
    }

    books.Remove(book);
    return Results.NoContent();
})
.WithName("DeleteBook")
.WithSummary("ลบหนังสือ")
.WithTags("Books");

// GET /api/books/search - ค้นหาหนังสือ
app.MapGet("/api/books/search", (string? title, string? author) =>
{
    var query = books.AsQueryable();

    if (!string.IsNullOrWhiteSpace(title))
        query = query.Where(b => b.Title.Contains(title, StringComparison.OrdinalIgnoreCase));

    if (!string.IsNullOrWhiteSpace(author))
        query = query.Where(b => b.Author.Contains(author, StringComparison.OrdinalIgnoreCase));

    return Results.Ok(query.ToList());
})
.WithName("SearchBooks")
.WithSummary("ค้นหาหนังสือ")
.WithTags("Books");

// GET / - Welcome endpoint
app.MapGet("/", () => new
{
    Message = "ยินดีต้อนรับสู่ Book API!",
    Version = "1.0.0",
    Documentation = "/swagger"
});

app.Run();
```

### ขั้นตอนที่ 4: รันและทดสอบ

```bash
dotnet run
```

เปิด browser ไปที่ `https://localhost:7001/swagger` จะเห็น Swagger UI

### ทดสอบด้วย curl

```bash
# ดึงหนังสือทั้งหมด
curl -X GET https://localhost:7001/api/books

# ดึงหนังสือ ID 1
curl -X GET https://localhost:7001/api/books/1

# สร้างหนังสือใหม่
curl -X POST https://localhost:7001/api/books \
  -H "Content-Type: application/json" \
  -d '{"title":"Design Patterns","author":"Gang of Four","isbn":"978-0201633610","price":780,"publishedYear":1994}'

# ค้นหาหนังสือ
curl -X GET "https://localhost:7001/api/books/search?title=C%23"

# ลบหนังสือ
curl -X DELETE https://localhost:7001/api/books/1
```

### ผลลัพธ์ GET /api/books

```json
[
  {
    "id": 1,
    "title": "C# Programming",
    "author": "Andrew Troelsen",
    "isbn": "978-1484268490",
    "price": 850.00,
    "publishedYear": 2022,
    "isAvailable": true
  },
  {
    "id": 2,
    "title": "ASP.NET Core in Action",
    "author": "Andrew Lock",
    "isbn": "978-1617298301",
    "price": 920.00,
    "publishedYear": 2023,
    "isAvailable": true
  }
]
```

---

## 10. การจัดการ NuGet Packages

```bash
# เพิ่ม package
dotnet add package Newtonsoft.Json
dotnet add package Microsoft.EntityFrameworkCore

# ลบ package
dotnet remove package Newtonsoft.Json

# รายการ packages ที่ติดตั้ง
dotnet list package

# อัพเดท packages
dotnet add package Microsoft.EntityFrameworkCore --version 9.0.0

# Restore packages
dotnet restore
```

---

## 11. Hot Reload

.NET 9 รองรับ Hot Reload ทำให้ไม่ต้อง restart server เมื่อแก้โค้ด:

```bash
# รันพร้อม hot reload
dotnet watch run

# หรือ
dotnet watch
```

เมื่อบันทึกไฟล์ แอปจะ reload อัตโนมัติ (หรือ apply changes โดยไม่ restart)

---

## Exercises

### Exercise 1: สร้าง Todo API
สร้าง ASP.NET Core Web API สำหรับจัดการ Todo list โดยมี endpoints:
- `GET /api/todos` - ดึง todo ทั้งหมด
- `GET /api/todos/{id}` - ดึง todo ตาม ID  
- `POST /api/todos` - สร้าง todo ใหม่
- `PUT /api/todos/{id}` - แก้ไข todo
- `DELETE /api/todos/{id}` - ลบ todo
- `PATCH /api/todos/{id}/complete` - mark todo เป็น complete

```csharp
// โครงสร้าง Todo model
public class TodoItem
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public bool IsCompleted { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? CompletedAt { get; set; }
}
```

### Exercise 2: เพิ่ม Pagination
แก้ไข Book API เพื่อรองรับ pagination:
- `GET /api/books?page=1&pageSize=10` - ดึงหนังสือแบบ paged
- ผลลัพธ์ต้องมี metadata: total count, current page, total pages

```csharp
// ตัวอย่าง response
public class PagedResult<T>
{
    public List<T> Data { get; set; } = [];
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}
```

### Exercise 3: เพิ่ม Validation
เพิ่ม validation ให้กับ Book API:
- Title ต้องมีความยาว 1-200 ตัวอักษร
- Price ต้องมากกว่า 0
- PublishedYear ต้องอยู่ระหว่าง 1800 จนถึงปีปัจจุบัน
- ISBN ต้องมีรูปแบบถูกต้อง

---

## สรุป

✅ ASP.NET Core เป็น framework สำหรับสร้าง web application ที่ทำงานได้บนทุก platform  
✅ `dotnet new webapi` สร้าง Web API project, `dotnet new mvc` สร้าง MVC project  
✅ โครงสร้างโปรเจคประกอบด้วย: `Program.cs`, `appsettings.json`, `Properties/launchSettings.json`  
✅ `launchSettings.json` กำหนดวิธีการ launch ใน development  
✅ `appsettings.json` เก็บ configuration ของ application  
✅ `dotnet run` รัน dev server, `dotnet watch run` รันพร้อม hot reload  
✅ Swagger/OpenAPI ช่วยให้ทดสอบ API ได้จาก browser  
✅ Minimal API style ใช้ `app.MapGet/Post/Put/Delete` สร้าง endpoints  

---

## Part ถัดไป

ใน **Part 042** เราจะเรียนรู้เรื่อง **Program.cs และ Startup (Minimal API)** อย่างละเอียด:
- Minimal API pattern ใน .NET 6+
- การตั้งค่า services ด้วย `builder.Services`
- Middleware ต่างๆ ด้วย `app.Use*` และ `app.Map*`
- การจัดการ Environment
- Logging configuration

---

*Part 041/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
