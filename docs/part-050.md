# Part 050: REST API เบื้องต้น

## เนื้อหาใน Part นี้
- REST principles
- HTTP status codes
- JSON serialization/deserialization
- System.Text.Json vs Newtonsoft.Json
- Controller-based API
- Minimal API
- API versioning เบื้องต้น
- โปรแกรมตัวอย่าง: Complete CRUD API

---

## 1. REST คืออะไร?

REST (Representational State Transfer) เป็น architectural style สำหรับ web services มีหลักการ 6 ข้อ:

### 1.1 REST Principles

```
1. Stateless        - Server ไม่เก็บ state ของ client
2. Client-Server    - แยก client กับ server ออกจากกัน
3. Cacheable        - Response สามารถ cache ได้
4. Uniform Interface - ใช้ URL และ HTTP methods สม่ำเสมอ
5. Layered System   - มีหลาย layers (proxy, gateway, etc.)
6. Code on Demand   - Server ส่ง executable code ให้ client (optional)
```

### 1.2 RESTful URL Design

```
Resource: Users

Collection:       GET    /api/users         → ดึง users ทั้งหมด
                  POST   /api/users         → สร้าง user ใหม่

Single Resource:  GET    /api/users/5       → ดึง user ID 5
                  PUT    /api/users/5       → แก้ไข user ID 5 (ทั้งหมด)
                  PATCH  /api/users/5       → แก้ไข user ID 5 (บางส่วน)
                  DELETE /api/users/5       → ลบ user ID 5

Nested Resource:  GET    /api/users/5/orders       → orders ของ user 5
                  POST   /api/users/5/orders       → สร้าง order ให้ user 5
                  GET    /api/users/5/orders/10    → order 10 ของ user 5

Filter/Search:    GET    /api/users?role=admin&active=true
Pagination:       GET    /api/users?page=2&pageSize=10
Sort:             GET    /api/users?sortBy=name&desc=false
```

### 1.3 ตัวอย่าง RESTful vs Non-RESTful

```
❌ Non-RESTful (RPC style)
GET  /getUser?id=5
POST /createNewUser
POST /deleteUser?id=5
POST /updateUserEmail

✅ RESTful
GET    /api/users/5
POST   /api/users
DELETE /api/users/5
PATCH  /api/users/5/email
```

---

## 2. HTTP Status Codes

### 2.1 2xx - Success

```
200 OK              → สำเร็จ (GET, PUT, PATCH)
201 Created         → สร้างสำเร็จ (POST)
202 Accepted        → รับคำสั่งแล้ว แต่ยังไม่เสร็จ (async operations)
204 No Content      → สำเร็จแต่ไม่มี body (DELETE)
206 Partial Content → ส่งข้อมูลบางส่วน (Range request)
```

### 2.2 3xx - Redirection

```
301 Moved Permanently  → URL เปลี่ยนถาวร
302 Found              → Redirect ชั่วคราว
304 Not Modified       → ข้อมูลไม่เปลี่ยน (ใช้ cache ได้)
307 Temporary Redirect → เหมือน 302 แต่คง HTTP method
308 Permanent Redirect → เหมือน 301 แต่คง HTTP method
```

### 2.3 4xx - Client Errors

```
400 Bad Request           → Request ผิดรูปแบบ
401 Unauthorized          → ต้อง authenticate ก่อน
403 Forbidden             → ไม่มีสิทธิ์ (แม้ authenticated แล้ว)
404 Not Found             → ไม่พบ resource
405 Method Not Allowed    → HTTP method ไม่รองรับ
406 Not Acceptable        → Content type ไม่รองรับ
409 Conflict              → ข้อมูลขัดแย้ง (duplicate)
410 Gone                  → Resource ถูกลบถาวรแล้ว
415 Unsupported Media Type → Content-Type ไม่รองรับ
422 Unprocessable Entity  → Validation error
429 Too Many Requests     → Rate limit exceeded
```

### 2.4 5xx - Server Errors

```
500 Internal Server Error → Server error ทั่วไป
501 Not Implemented       → Feature ยังไม่ได้ implement
502 Bad Gateway           → Gateway/proxy error
503 Service Unavailable   → Service ไม่พร้อม (maintenance)
504 Gateway Timeout       → Gateway timeout
```

### 2.5 การเลือก Status Code ที่เหมาะสม

```csharp
[ApiController]
[Route("api/[controller]")]
public class ItemsController : ControllerBase
{
    // GET → 200 OK (หรือ 404 ถ้าไม่พบ)
    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        var item = _service.GetById(id);
        return item is null ? NotFound() : Ok(item);  // 404 หรือ 200
    }

    // POST → 201 Created
    [HttpPost]
    public IActionResult Create(CreateItemDto dto)
    {
        var item = _service.Create(dto);
        return CreatedAtAction(nameof(Get), new { id = item.Id }, item);  // 201
    }

    // PUT → 200 OK หรือ 204 No Content
    [HttpPut("{id}")]
    public IActionResult Update(int id, UpdateItemDto dto)
    {
        var item = _service.Update(id, dto);
        return item is null ? NotFound() : Ok(item);  // 404 หรือ 200
        // หรือ return NoContent(); ถ้าไม่ต้องการ return body
    }

    // DELETE → 204 No Content
    [HttpDelete("{id}")]
    public IActionResult Delete(int id)
    {
        _service.Delete(id);
        return NoContent();  // 204
    }

    // Validation error → 400 Bad Request
    [HttpPost("validate")]
    public IActionResult Validate(ItemDto dto)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);  // 400

        return Ok();
    }

    // Duplicate → 409 Conflict
    [HttpPost("create-unique")]
    public IActionResult CreateUnique(ItemDto dto)
    {
        if (_service.Exists(dto.Name))
            return Conflict(new { Message = "Item already exists" });  // 409

        return CreatedAtAction(nameof(Get), new { id = 1 }, dto);
    }
}
```

---

## 3. JSON Serialization

### 3.1 System.Text.Json (Built-in .NET)

```csharp
// ใช้งานพื้นฐาน
using System.Text.Json;

var product = new Product { Name = "iPhone", Price = 39900 };

// Serialize
string json = JsonSerializer.Serialize(product);
// {"Name":"iPhone","Price":39900}

// Deserialize
var deserialized = JsonSerializer.Deserialize<Product>(json);

// Options
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,    // camelCase
    WriteIndented = true,                                  // pretty print
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    NumberHandling = JsonNumberHandling.AllowReadingFromString
};

string prettyJson = JsonSerializer.Serialize(product, options);
```

### 3.2 ตั้งค่า JSON ใน ASP.NET Core

```csharp
// Program.cs
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        // CamelCase property names
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        
        // ไม่ส่ง null values
        options.JsonSerializerOptions.DefaultIgnoreCondition =
            JsonIgnoreCondition.WhenWritingNull;
        
        // อ่าน enum เป็น string
        options.JsonSerializerOptions.Converters.Add(
            new JsonStringEnumConverter());
        
        // Pretty print ใน development
        options.JsonSerializerOptions.WriteIndented =
            builder.Environment.IsDevelopment();
        
        // Allow trailing commas
        options.JsonSerializerOptions.AllowTrailingCommas = true;
    });

// หรือสำหรับ Minimal API
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
```

### 3.3 JSON Attributes

```csharp
using System.Text.Json.Serialization;

public class UserDto
{
    // กำหนดชื่อ JSON property
    [JsonPropertyName("user_id")]
    public int Id { get; set; }

    // ไม่ส่ง property นี้เมื่อ serialize
    [JsonIgnore]
    public string PasswordHash { get; set; } = string.Empty;

    // ไม่ส่งเมื่อ value เป็น null
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]
    public string? MiddleName { get; set; }

    // ไม่ส่งเมื่อ value เป็น default
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
    public int? Age { get; set; }

    // อ่านได้อย่างเดียว (write-only)
    [JsonInclude]
    private string InternalNote { get; set; } = string.Empty;

    // Order ของ serialization
    [JsonPropertyOrder(1)]
    public string FirstName { get; set; } = string.Empty;

    [JsonPropertyOrder(2)]
    public string LastName { get; set; } = string.Empty;
}

// Custom converter
public class DateOnlyJsonConverter : JsonConverter<DateOnly>
{
    public override DateOnly Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        => DateOnly.Parse(reader.GetString()!);

    public override void Write(Utf8JsonWriter writer, DateOnly value, JsonSerializerOptions options)
        => writer.WriteStringValue(value.ToString("yyyy-MM-dd"));
}
```

### 3.4 System.Text.Json vs Newtonsoft.Json

| Feature | System.Text.Json | Newtonsoft.Json |
|---------|-----------------|----------------|
| Performance | เร็วกว่า ~2x | ช้ากว่า |
| Built-in | ✅ .NET 5+ | ❌ ต้องติดตั้ง |
| Features | น้อยกว่า | มากกว่า |
| Custom Converters | ยากกว่า | ง่ายกว่า |
| Circular Reference | ต้องตั้งค่า | จัดการได้ง่าย |
| Dynamic/ExpandoObject | ❌ | ✅ |
| JObject/JToken | ❌ | ✅ |

```csharp
// ใช้ Newtonsoft.Json (ถ้าต้องการ compatibility)
dotnet add package Microsoft.AspNetCore.Mvc.NewtonsoftJson

builder.Services.AddControllers()
    .AddNewtonsoftJson(options =>
    {
        options.SerializerSettings.ContractResolver =
            new CamelCasePropertyNamesContractResolver();
        options.SerializerSettings.NullValueHandling = NullValueHandling.Ignore;
        options.SerializerSettings.Converters.Add(new StringEnumConverter());
        options.SerializerSettings.ReferenceLoopHandling = ReferenceLoopHandling.Ignore;
    });
```

---

## 4. API Response Standards

### 4.1 Consistent Response Format

```csharp
// Response wrapper
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T? Data { get; set; }
    public string? Message { get; set; }
    public List<string> Errors { get; set; } = [];
    public object? Meta { get; set; }

    public static ApiResponse<T> Ok(T data, string? message = null) =>
        new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string error) =>
        new() { Success = false, Errors = [error] };

    public static ApiResponse<T> Fail(List<string> errors) =>
        new() { Success = false, Errors = errors };
}

// Paged response
public class PagedApiResponse<T> : ApiResponse<List<T>>
{
    public PaginationMeta Pagination { get; set; } = new();
}

public class PaginationMeta
{
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public int TotalPages { get; set; }
    public bool HasNextPage { get; set; }
    public bool HasPreviousPage { get; set; }
}
```

---

## 5. API Versioning เบื้องต้น

### 5.1 วิธีการ Versioning

```
1. URL Path:      /api/v1/users, /api/v2/users
2. Query String:  /api/users?version=1
3. Header:        X-API-Version: 1
4. Content-Type:  application/vnd.myapp.v1+json
```

### 5.2 URL Path Versioning (แนะนำ)

```csharp
// ติดตั้ง package
// dotnet add package Asp.Versioning.Mvc

builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = new UrlSegmentApiVersionReader();
}).AddMvc();

// Controller v1
[ApiVersion(1)]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new { Version = "1.0", Users = new[] { "User1" } });
}

// Controller v2 (เพิ่ม features)
[ApiVersion(2)]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new { 
        Version = "2.0", 
        Users = new[] { "User1", "User2" },
        Meta = new { Total = 2 }
    });
}
```

### 5.3 Manual Path Versioning (ง่ายกว่า)

```csharp
// สำหรับโปรเจคขนาดเล็ก
[ApiController]
[Route("api/v1/[controller]")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new[] { "Product1" });
}

[ApiController]
[Route("api/v2/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new
    {
        Data = new[] { "Product1", "Product2" },
        Total = 2,
        Version = "2.0"
    });
}
```

---

## 6. โปรแกรมตัวอย่าง: Complete CRUD API

มาสร้าง Complete CRUD API ที่รวมทุกอย่างที่เรียนมา:

```csharp
// Program.cs
using System.Text.Json;
using System.Text.Json.Serialization;

var builder = WebApplication.CreateBuilder(args);

// Controllers พร้อม JSON options
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        options.JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
        options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
        options.JsonSerializerOptions.WriteIndented = builder.Environment.IsDevelopment();
    });

// Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new Microsoft.OpenApi.Models.OpenApiInfo
    {
        Title = "Task Management API",
        Version = "v1",
        Description = "Complete CRUD API ตัวอย่างสำหรับ Part 050"
    });
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
        policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader());
});

// Services
builder.Services.AddSingleton<ITaskRepository, InMemoryTaskRepository>();
builder.Services.AddScoped<ITaskService, TaskService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors();
app.MapControllers();

// Health check
app.MapGet("/health", () => new
{
    Status = "Healthy",
    Timestamp = DateTime.UtcNow,
    Version = "1.0.0"
});

app.Run();

// ============================================
// DOMAIN MODELS
// ============================================

public enum TaskStatus { Todo, InProgress, Done, Cancelled }
public enum TaskPriority { Low, Medium, High, Critical }

public class TaskItem
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public TaskStatus Status { get; set; } = TaskStatus.Todo;
    public TaskPriority Priority { get; set; } = TaskPriority.Medium;
    public string? AssignedTo { get; set; }
    public DateTime? DueDate { get; set; }
    public List<string> Tags { get; set; } = [];
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    public bool IsDeleted { get; set; }
}

// ============================================
// DTOs
// ============================================

public class TaskDto
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public string Status { get; set; } = string.Empty;
    public string Priority { get; set; } = string.Empty;
    public string? AssignedTo { get; set; }
    public DateTime? DueDate { get; set; }
    public List<string> Tags { get; set; } = [];
    public DateTime CreatedAt { get; set; }
    public bool IsOverdue { get; set; }
}

public class CreateTaskRequest
{
    [Required(ErrorMessage = "กรุณาระบุชื่องาน")]
    [StringLength(200, MinimumLength = 3, ErrorMessage = "ชื่องานต้องมีความยาว 3-200 ตัวอักษร")]
    public string Title { get; set; } = string.Empty;

    [StringLength(2000, ErrorMessage = "คำอธิบายต้องไม่เกิน 2000 ตัวอักษร")]
    public string? Description { get; set; }

    public TaskPriority Priority { get; set; } = TaskPriority.Medium;

    public string? AssignedTo { get; set; }

    [DataType(DataType.DateTime)]
    public DateTime? DueDate { get; set; }

    public List<string> Tags { get; set; } = [];
}

public class UpdateTaskRequest : CreateTaskRequest
{
    public TaskStatus Status { get; set; } = TaskStatus.Todo;
}

public class PatchTaskRequest
{
    public string? Title { get; set; }
    public string? Description { get; set; }
    public TaskStatus? Status { get; set; }
    public TaskPriority? Priority { get; set; }
    public string? AssignedTo { get; set; }
    public DateTime? DueDate { get; set; }
}

public class TaskFilter
{
    public string? Search { get; set; }
    public TaskStatus? Status { get; set; }
    public TaskPriority? Priority { get; set; }
    public string? AssignedTo { get; set; }
    public string? Tag { get; set; }
    public bool? IsOverdue { get; set; }
    public int Page { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    public string SortBy { get; set; } = "createdAt";
    public bool Desc { get; set; } = true;
}

// ============================================
// CONTROLLER
// ============================================

[ApiController]
[Route("api/v1/[controller]")]
[Produces("application/json")]
public class TasksController : ControllerBase
{
    private readonly ITaskService _service;
    private readonly ILogger<TasksController> _logger;

    public TasksController(ITaskService service, ILogger<TasksController> logger)
    {
        _service = service;
        _logger = logger;
    }

    /// <summary>ดึงรายการงานทั้งหมดพร้อม filtering และ pagination</summary>
    [HttpGet]
    [ProducesResponseType(typeof(ApiResponse<List<TaskDto>>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<List<TaskDto>>>> GetAll(
        [FromQuery] TaskFilter filter)
    {
        var (tasks, total) = await _service.GetAllAsync(filter);
        
        var response = new ApiResponse<List<TaskDto>>
        {
            Success = true,
            Data = tasks,
            Meta = new PaginationMeta
            {
                Page = filter.Page,
                PageSize = filter.PageSize,
                TotalCount = total,
                TotalPages = (int)Math.Ceiling((double)total / filter.PageSize),
                HasNextPage = filter.Page < (int)Math.Ceiling((double)total / filter.PageSize),
                HasPreviousPage = filter.Page > 1
            }
        };
        
        return Ok(response);
    }

    /// <summary>ดึงงานตาม ID</summary>
    [HttpGet("{id:int}", Name = "GetTask")]
    [ProducesResponseType(typeof(ApiResponse<TaskDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<TaskDto>>> GetById(int id)
    {
        var task = await _service.GetByIdAsync(id);
        
        if (task is null)
            return NotFound(ApiResponse<TaskDto>.Fail($"ไม่พบงาน ID: {id}"));
        
        return Ok(ApiResponse<TaskDto>.Ok(task));
    }

    /// <summary>สร้างงานใหม่</summary>
    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<TaskDto>), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ApiResponse<TaskDto>>> Create(
        [FromBody] CreateTaskRequest request)
    {
        var task = await _service.CreateAsync(request);
        _logger.LogInformation("Task created: {Id} - {Title}", task.Id, task.Title);
        
        return CreatedAtRoute(
            "GetTask",
            new { id = task.Id },
            ApiResponse<TaskDto>.Ok(task, "สร้างงานสำเร็จ"));
    }

    /// <summary>แก้ไขงาน (ทั้งหมด)</summary>
    [HttpPut("{id:int}")]
    [ProducesResponseType(typeof(ApiResponse<TaskDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<TaskDto>>> Update(
        int id, [FromBody] UpdateTaskRequest request)
    {
        var task = await _service.UpdateAsync(id, request);
        
        if (task is null)
            return NotFound(ApiResponse<TaskDto>.Fail($"ไม่พบงาน ID: {id}"));
        
        return Ok(ApiResponse<TaskDto>.Ok(task, "แก้ไขงานสำเร็จ"));
    }

    /// <summary>แก้ไขงาน (บางส่วน)</summary>
    [HttpPatch("{id:int}")]
    [ProducesResponseType(typeof(ApiResponse<TaskDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<TaskDto>>> Patch(
        int id, [FromBody] PatchTaskRequest request)
    {
        var task = await _service.PatchAsync(id, request);
        
        if (task is null)
            return NotFound(ApiResponse<TaskDto>.Fail($"ไม่พบงาน ID: {id}"));
        
        return Ok(ApiResponse<TaskDto>.Ok(task, "อัพเดทงานสำเร็จ"));
    }

    /// <summary>ลบงาน (soft delete)</summary>
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        
        if (!deleted)
            return NotFound(ApiResponse<object>.Fail($"ไม่พบงาน ID: {id}"));
        
        _logger.LogInformation("Task deleted: {Id}", id);
        return NoContent();
    }

    /// <summary>เปลี่ยน status ของงาน</summary>
    [HttpPatch("{id:int}/status")]
    [ProducesResponseType(typeof(ApiResponse<TaskDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ApiResponse<TaskDto>>> UpdateStatus(
        int id, [FromBody] UpdateStatusRequest request)
    {
        var task = await _service.UpdateStatusAsync(id, request.Status);
        
        if (task is null)
            return NotFound(ApiResponse<TaskDto>.Fail($"ไม่พบงาน ID: {id}"));
        
        return Ok(ApiResponse<TaskDto>.Ok(task, $"อัพเดทสถานะเป็น '{request.Status}' สำเร็จ"));
    }

    /// <summary>ดึงสถิติงาน</summary>
    [HttpGet("stats")]
    [ProducesResponseType(typeof(ApiResponse<TaskStats>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<TaskStats>>> GetStats()
    {
        var stats = await _service.GetStatsAsync();
        return Ok(ApiResponse<TaskStats>.Ok(stats));
    }

    /// <summary>ค้นหางาน</summary>
    [HttpGet("search")]
    [ProducesResponseType(typeof(ApiResponse<List<TaskDto>>), StatusCodes.Status200OK)]
    public async Task<ActionResult<ApiResponse<List<TaskDto>>>> Search(
        [FromQuery] string q,
        [FromQuery] int limit = 10)
    {
        if (string.IsNullOrWhiteSpace(q))
            return BadRequest(ApiResponse<List<TaskDto>>.Fail("กรุณาระบุคำค้นหา"));
        
        var tasks = await _service.SearchAsync(q, limit);
        return Ok(ApiResponse<List<TaskDto>>.Ok(tasks));
    }

    /// <summary>Bulk operations</summary>
    [HttpPost("bulk/complete")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    public async Task<IActionResult> BulkComplete([FromBody] List<int> ids)
    {
        if (!ids.Any())
            return BadRequest(new { Message = "กรุณาระบุ IDs" });
        
        var count = await _service.BulkUpdateStatusAsync(ids, TaskStatus.Done);
        return Ok(new { Message = $"อัพเดท {count} งานเป็น Done สำเร็จ" });
    }
}

public class UpdateStatusRequest
{
    [Required]
    public TaskStatus Status { get; set; }
}

// ============================================
// MINIMAL API (v2)
// ============================================

// ตัวอย่าง Minimal API v2 (เพิ่มใน Program.cs)
// var v2 = app.MapGroup("/api/v2/tasks").WithTags("Tasks v2");
// 
// v2.MapGet("/", async (ITaskService service, [AsParameters] TaskFilter filter) =>
// {
//     var (tasks, total) = await service.GetAllAsync(filter);
//     return Results.Ok(new { tasks, total, version = "2.0" });
// });
// 
// v2.MapGet("/{id:int}", async (int id, ITaskService service) =>
// {
//     var task = await service.GetByIdAsync(id);
//     return task is null ? Results.NotFound() : Results.Ok(task);
// });

// ============================================
// SERVICE LAYER
// ============================================

public class TaskStats
{
    public int Total { get; set; }
    public int Todo { get; set; }
    public int InProgress { get; set; }
    public int Done { get; set; }
    public int Cancelled { get; set; }
    public int Overdue { get; set; }
    public Dictionary<string, int> ByPriority { get; set; } = [];
}

public interface ITaskService
{
    Task<(List<TaskDto> Tasks, int Total)> GetAllAsync(TaskFilter filter);
    Task<TaskDto?> GetByIdAsync(int id);
    Task<TaskDto> CreateAsync(CreateTaskRequest request);
    Task<TaskDto?> UpdateAsync(int id, UpdateTaskRequest request);
    Task<TaskDto?> PatchAsync(int id, PatchTaskRequest request);
    Task<bool> DeleteAsync(int id);
    Task<TaskDto?> UpdateStatusAsync(int id, TaskStatus status);
    Task<TaskStats> GetStatsAsync();
    Task<List<TaskDto>> SearchAsync(string query, int limit);
    Task<int> BulkUpdateStatusAsync(List<int> ids, TaskStatus status);
}

public interface ITaskRepository
{
    Task<(List<TaskItem> Tasks, int Total)> GetAllAsync(TaskFilter filter);
    Task<TaskItem?> GetByIdAsync(int id);
    Task<TaskItem> CreateAsync(TaskItem task);
    Task<TaskItem?> UpdateAsync(TaskItem task);
    Task<bool> DeleteAsync(int id);
    Task<List<TaskItem>> GetAllRawAsync();
    Task<int> BulkUpdateStatusAsync(List<int> ids, TaskStatus status);
}

public class TaskService : ITaskService
{
    private readonly ITaskRepository _repository;

    public TaskService(ITaskRepository repository) => _repository = repository;

    public async Task<(List<TaskDto> Tasks, int Total)> GetAllAsync(TaskFilter filter)
    {
        var (tasks, total) = await _repository.GetAllAsync(filter);
        return (tasks.Select(ToDto).ToList(), total);
    }

    public async Task<TaskDto?> GetByIdAsync(int id)
    {
        var task = await _repository.GetByIdAsync(id);
        return task is null ? null : ToDto(task);
    }

    public async Task<TaskDto> CreateAsync(CreateTaskRequest request)
    {
        var task = new TaskItem
        {
            Title = request.Title,
            Description = request.Description,
            Priority = request.Priority,
            AssignedTo = request.AssignedTo,
            DueDate = request.DueDate,
            Tags = request.Tags
        };
        var created = await _repository.CreateAsync(task);
        return ToDto(created);
    }

    public async Task<TaskDto?> UpdateAsync(int id, UpdateTaskRequest request)
    {
        var existing = await _repository.GetByIdAsync(id);
        if (existing is null) return null;

        existing.Title = request.Title;
        existing.Description = request.Description;
        existing.Status = request.Status;
        existing.Priority = request.Priority;
        existing.AssignedTo = request.AssignedTo;
        existing.DueDate = request.DueDate;
        existing.Tags = request.Tags;
        existing.UpdatedAt = DateTime.UtcNow;

        var updated = await _repository.UpdateAsync(existing);
        return updated is null ? null : ToDto(updated);
    }

    public async Task<TaskDto?> PatchAsync(int id, PatchTaskRequest request)
    {
        var existing = await _repository.GetByIdAsync(id);
        if (existing is null) return null;

        if (request.Title != null) existing.Title = request.Title;
        if (request.Description != null) existing.Description = request.Description;
        if (request.Status.HasValue) existing.Status = request.Status.Value;
        if (request.Priority.HasValue) existing.Priority = request.Priority.Value;
        if (request.AssignedTo != null) existing.AssignedTo = request.AssignedTo;
        if (request.DueDate.HasValue) existing.DueDate = request.DueDate.Value;
        existing.UpdatedAt = DateTime.UtcNow;

        var updated = await _repository.UpdateAsync(existing);
        return updated is null ? null : ToDto(updated);
    }

    public Task<bool> DeleteAsync(int id) => _repository.DeleteAsync(id);

    public async Task<TaskDto?> UpdateStatusAsync(int id, TaskStatus status)
    {
        var task = await _repository.GetByIdAsync(id);
        if (task is null) return null;

        task.Status = status;
        task.UpdatedAt = DateTime.UtcNow;
        var updated = await _repository.UpdateAsync(task);
        return updated is null ? null : ToDto(updated);
    }

    public async Task<TaskStats> GetStatsAsync()
    {
        var tasks = await _repository.GetAllRawAsync();
        var now = DateTime.UtcNow;

        return new TaskStats
        {
            Total = tasks.Count,
            Todo = tasks.Count(t => t.Status == TaskStatus.Todo),
            InProgress = tasks.Count(t => t.Status == TaskStatus.InProgress),
            Done = tasks.Count(t => t.Status == TaskStatus.Done),
            Cancelled = tasks.Count(t => t.Status == TaskStatus.Cancelled),
            Overdue = tasks.Count(t => t.DueDate < now && t.Status != TaskStatus.Done),
            ByPriority = tasks
                .GroupBy(t => t.Priority.ToString())
                .ToDictionary(g => g.Key, g => g.Count())
        };
    }

    public async Task<List<TaskDto>> SearchAsync(string query, int limit)
    {
        var all = await _repository.GetAllRawAsync();
        return all
            .Where(t => t.Title.Contains(query, StringComparison.OrdinalIgnoreCase)
                     || (t.Description?.Contains(query, StringComparison.OrdinalIgnoreCase) ?? false)
                     || t.Tags.Any(tag => tag.Contains(query, StringComparison.OrdinalIgnoreCase)))
            .Take(limit)
            .Select(ToDto)
            .ToList();
    }

    public Task<int> BulkUpdateStatusAsync(List<int> ids, TaskStatus status)
        => _repository.BulkUpdateStatusAsync(ids, status);

    private static TaskDto ToDto(TaskItem t) => new()
    {
        Id = t.Id,
        Title = t.Title,
        Description = t.Description,
        Status = t.Status.ToString(),
        Priority = t.Priority.ToString(),
        AssignedTo = t.AssignedTo,
        DueDate = t.DueDate,
        Tags = t.Tags,
        CreatedAt = t.CreatedAt,
        IsOverdue = t.DueDate.HasValue && t.DueDate < DateTime.UtcNow
            && t.Status != TaskStatus.Done
    };
}

public class InMemoryTaskRepository : ITaskRepository
{
    private readonly List<TaskItem> _tasks;
    private int _nextId = 6;
    private readonly object _lock = new();

    public InMemoryTaskRepository()
    {
        _tasks = new List<TaskItem>
        {
            new() { Id=1, Title="Setup project", Status=TaskStatus.Done, Priority=TaskPriority.High,
                    Tags=["setup","backend"], CreatedAt=DateTime.UtcNow.AddDays(-10) },
            new() { Id=2, Title="Design database schema", Status=TaskStatus.Done, Priority=TaskPriority.High,
                    Tags=["database","design"], CreatedAt=DateTime.UtcNow.AddDays(-8) },
            new() { Id=3, Title="Implement authentication", Status=TaskStatus.InProgress, 
                    Priority=TaskPriority.Critical, AssignedTo="john",
                    DueDate=DateTime.UtcNow.AddDays(3), Tags=["auth","security"],
                    CreatedAt=DateTime.UtcNow.AddDays(-5) },
            new() { Id=4, Title="Write API documentation", Status=TaskStatus.Todo,
                    Priority=TaskPriority.Medium, AssignedTo="jane",
                    DueDate=DateTime.UtcNow.AddDays(7), Tags=["docs","api"],
                    CreatedAt=DateTime.UtcNow.AddDays(-3) },
            new() { Id=5, Title="Deploy to staging", Status=TaskStatus.Todo,
                    Priority=TaskPriority.Low, DueDate=DateTime.UtcNow.AddDays(-2),
                    Tags=["deploy","ops"], CreatedAt=DateTime.UtcNow.AddDays(-1) }
        };
    }

    public Task<(List<TaskItem> Tasks, int Total)> GetAllAsync(TaskFilter filter)
    {
        var query = _tasks.Where(t => !t.IsDeleted).AsQueryable();

        if (!string.IsNullOrWhiteSpace(filter.Search))
            query = query.Where(t => t.Title.Contains(filter.Search, StringComparison.OrdinalIgnoreCase));
        if (filter.Status.HasValue)
            query = query.Where(t => t.Status == filter.Status.Value);
        if (filter.Priority.HasValue)
            query = query.Where(t => t.Priority == filter.Priority.Value);
        if (!string.IsNullOrWhiteSpace(filter.AssignedTo))
            query = query.Where(t => t.AssignedTo?.Equals(filter.AssignedTo, StringComparison.OrdinalIgnoreCase) == true);
        if (!string.IsNullOrWhiteSpace(filter.Tag))
            query = query.Where(t => t.Tags.Contains(filter.Tag, StringComparer.OrdinalIgnoreCase));
        if (filter.IsOverdue.HasValue && filter.IsOverdue.Value)
            query = query.Where(t => t.DueDate < DateTime.UtcNow && t.Status != TaskStatus.Done);

        query = filter.SortBy.ToLower() switch
        {
            "title" => filter.Desc ? query.OrderByDescending(t => t.Title) : query.OrderBy(t => t.Title),
            "priority" => filter.Desc ? query.OrderByDescending(t => t.Priority) : query.OrderBy(t => t.Priority),
            "duedate" => filter.Desc ? query.OrderByDescending(t => t.DueDate) : query.OrderBy(t => t.DueDate),
            _ => filter.Desc ? query.OrderByDescending(t => t.CreatedAt) : query.OrderBy(t => t.CreatedAt)
        };

        var total = query.Count();
        var tasks = query.Skip((filter.Page - 1) * filter.PageSize).Take(filter.PageSize).ToList();
        return Task.FromResult((tasks, total));
    }

    public Task<TaskItem?> GetByIdAsync(int id)
        => Task.FromResult(_tasks.FirstOrDefault(t => t.Id == id && !t.IsDeleted));

    public Task<TaskItem> CreateAsync(TaskItem task)
    {
        lock (_lock)
        {
            task.Id = _nextId++;
            _tasks.Add(task);
            return Task.FromResult(task);
        }
    }

    public Task<TaskItem?> UpdateAsync(TaskItem task)
    {
        lock (_lock)
        {
            var idx = _tasks.FindIndex(t => t.Id == task.Id);
            if (idx < 0) return Task.FromResult<TaskItem?>(null);
            _tasks[idx] = task;
            return Task.FromResult<TaskItem?>(task);
        }
    }

    public Task<bool> DeleteAsync(int id)
    {
        lock (_lock)
        {
            var task = _tasks.FirstOrDefault(t => t.Id == id && !t.IsDeleted);
            if (task is null) return Task.FromResult(false);
            task.IsDeleted = true;
            return Task.FromResult(true);
        }
    }

    public Task<List<TaskItem>> GetAllRawAsync()
        => Task.FromResult(_tasks.Where(t => !t.IsDeleted).ToList());

    public Task<int> BulkUpdateStatusAsync(List<int> ids, TaskStatus status)
    {
        lock (_lock)
        {
            var count = 0;
            foreach (var id in ids)
            {
                var task = _tasks.FirstOrDefault(t => t.Id == id && !t.IsDeleted);
                if (task != null)
                {
                    task.Status = status;
                    task.UpdatedAt = DateTime.UtcNow;
                    count++;
                }
            }
            return Task.FromResult(count);
        }
    }
}
```

### ทดสอบ API

```bash
# รัน
dotnet run

# ดึงงานทั้งหมด
curl https://localhost:7001/api/v1/tasks

# ดึงงาน ID 1
curl https://localhost:7001/api/v1/tasks/1

# สร้างงานใหม่
curl -X POST https://localhost:7001/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "title": "เขียน unit tests",
    "description": "เขียน tests ให้ครอบคลุม 80%",
    "priority": "High",
    "assignedTo": "developer",
    "dueDate": "2024-12-31",
    "tags": ["testing", "quality"]
  }'

# แก้ไข status
curl -X PATCH https://localhost:7001/api/v1/tasks/1/status \
  -H "Content-Type: application/json" \
  -d '{"status": "InProgress"}'

# ค้นหา
curl "https://localhost:7001/api/v1/tasks/search?q=test&limit=5"

# สถิติ
curl https://localhost:7001/api/v1/tasks/stats

# Bulk complete
curl -X POST https://localhost:7001/api/v1/tasks/bulk/complete \
  -H "Content-Type: application/json" \
  -d '[1, 2, 3]'

# ลบ
curl -X DELETE https://localhost:7001/api/v1/tasks/5
```

---

## Exercises

### Exercise 1: เพิ่ม API versioning
เพิ่ม v2 ของ Tasks API ที่มี features เพิ่มเติม:

```csharp
// v2 เพิ่ม:
// 1. Response มี links (HATEOAS)
// 2. Filtering เพิ่ม dateFrom, dateTo
// 3. Bulk delete endpoint
// 4. Export tasks เป็น CSV
```

### Exercise 2: Rate Limiting
เพิ่ม rate limiting ให้กับ API:

```csharp
// ต้องการ:
// 1. Global: 100 requests/minute
// 2. /api/tasks POST: 10 requests/minute
// 3. /api/tasks/search: 30 requests/minute
// 4. Return 429 Too Many Requests เมื่อเกิน limit
// 5. Response headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
```

### Exercise 3: Error Handling
สร้าง global error handler ที่สมบูรณ์:

```csharp
// ต้องการ:
// 1. Catch ทุก exceptions ใน global middleware
// 2. Return consistent ApiResponse format
// 3. Log errors ด้วย correlation ID
// 4. Development: แสดง stack trace, Production: ซ่อน
// 5. Custom exceptions: NotFoundException, ValidationException, ConflictException
```

---

## สรุป

✅ REST เป็น architectural style ที่ใช้ URL + HTTP methods จัดการ resources  
✅ HTTP status codes: 2xx success, 3xx redirect, 4xx client error, 5xx server error  
✅ `System.Text.Json` built-in เร็วกว่า, `Newtonsoft.Json` features มากกว่า  
✅ JSON options: `PropertyNamingPolicy`, `DefaultIgnoreCondition`, `JsonStringEnumConverter`  
✅ `[JsonPropertyName]`, `[JsonIgnore]` ควบคุม JSON serialization ในระดับ property  
✅ Controller-based API เหมาะกับ complex logic, Minimal API เหมาะกับ simple endpoints  
✅ API versioning: URL path `/api/v1/`, query string `?version=1`, หรือ header  
✅ Consistent response format ช่วยให้ client integrate ง่ายขึ้น  
✅ `PagedApiResponse<T>` ช่วย standardize pagination responses  

---

## Part ถัดไป

ใน **Part 051** เราจะเรียนรู้เรื่อง **Entity Framework Core เบื้องต้น**:
- ORM concept
- DbContext
- Code-first migrations
- LINQ queries
- Repository pattern กับ EF Core

---

*Part 050/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
