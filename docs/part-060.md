# Part 060: Authorization และ Roles

## เนื้อหาใน Part นี้
- [Authorize] Attribute
- Role-based Authorization
- Policy-based Authorization
- Resource-based Authorization
- Authorization Middleware
- โปรแกรมตัวอย่าง: Multi-role System

---

## Authorization คืออะไร

**Authorization** คือกระบวนการกำหนดว่า user ที่ผ่าน authentication แล้วสามารถทำอะไรได้บ้าง

### ประเภทของ Authorization ใน ASP.NET Core

| ประเภท | ตัวอย่าง |
|--------|---------|
| Role-based | "Admin เท่านั้นที่ลบ user ได้" |
| Claims-based | "user ที่มี claim 'department:HR' เท่านั้น" |
| Policy-based | "ต้องอายุมากกว่า 18 ปี" |
| Resource-based | "Owner เท่านั้นที่แก้ไข post ของตัวเองได้" |

---

## [Authorize] Attribute

### ใช้งานพื้นฐาน

```csharp
// ต้อง authenticate (login) เท่านั้น
[Authorize]
public IActionResult GetUserData() { ... }

// ต้องเป็น Admin role
[Authorize(Roles = "Admin")]
public IActionResult AdminAction() { ... }

// Admin หรือ Manager (OR condition)
[Authorize(Roles = "Admin,Manager")]
public IActionResult ManageAction() { ... }

// ต้องเป็นทั้ง Admin และ SuperUser (AND condition)
[Authorize(Roles = "Admin")]
[Authorize(Roles = "SuperUser")]
public IActionResult SuperAdminAction() { ... }

// ใช้ Policy
[Authorize(Policy = "AtLeast18")]
public IActionResult AdultContent() { ... }

// AllowAnonymous - bypass authorize ทั้งหมด
[AllowAnonymous]
public IActionResult PublicContent() { ... }
```

### ระดับ Controller

```csharp
// ทุก action ใน controller ต้อง authorize
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class ProductController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() { ... }  // ต้อง login
    
    [HttpGet("{id}")]
    [AllowAnonymous]  // ยกเว้น action นี้
    public IActionResult GetById(int id) { ... }  // ทุกคนดูได้
    
    [HttpPost]
    [Authorize(Roles = "Admin,Manager")]  // เพิ่ม requirement
    public IActionResult Create() { ... }  // ต้อง Admin หรือ Manager
    
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    public IActionResult Delete(int id) { ... }  // Admin เท่านั้น
}
```

---

## Role-based Authorization

### Setup Roles

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // ไม่ต้องกำหนดเพิ่มเติมสำหรับ role-based
    // เพียงแค่มี Claims ใน JWT ที่ถูกต้อง
});
```

### Claims ใน JWT

```csharp
// ใน Token generation
claims.Add(new Claim(ClaimTypes.Role, "Admin"));
claims.Add(new Claim(ClaimTypes.Role, "Manager"));
// หรือแบบ short form
claims.Add(new Claim("role", "User"));
```

### ตรวจสอบ Role ใน Code

```csharp
[HttpGet]
[Authorize]
public IActionResult GetAction()
{
    // วิธีที่ 1: IsInRole
    if (!User.IsInRole("Admin"))
        return Forbid();
    
    // วิธีที่ 2: FindAll
    var roles = User.FindAll(ClaimTypes.Role)
        .Select(c => c.Value)
        .ToList();
    
    // วิธีที่ 3: HasClaim
    if (!User.HasClaim(c => c.Type == ClaimTypes.Role && c.Value == "Admin"))
        return Forbid();
    
    return Ok("Admin action");
}
```

---

## Policy-based Authorization

Policy คือ rule ที่ซับซ้อนกว่า role-based

### สร้าง Simple Policy

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    // Policy ที่ต้อง login และเป็น Admin
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));
    
    // Policy ที่ต้องมี claim "department" เป็น "IT"
    options.AddPolicy("ITDepartment", policy =>
        policy.RequireClaim("department", "IT"));
    
    // Policy ที่ต้องมีทั้ง 2 roles
    options.AddPolicy("AdminManager", policy =>
        policy.RequireRole("Admin")
              .RequireRole("Manager"));
    
    // Policy ที่ต้อง authenticated เท่านั้น
    options.AddPolicy("AuthenticatedUser", policy =>
        policy.RequireAuthenticatedUser());
    
    // Policy ที่ใช้ custom requirement
    options.AddPolicy("MinimumAge18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
    
    // Policy ที่รวม role และ claim
    options.AddPolicy("SeniorStaff", policy =>
    {
        policy.RequireRole("Manager", "Admin");
        policy.RequireClaim("yearsOfService");
    });
});
```

### ใช้งาน Policy

```csharp
[Authorize(Policy = "AdminOnly")]
public IActionResult AdminAction() { ... }

[Authorize(Policy = "ITDepartment")]
public IActionResult ITAction() { ... }
```

---

## Custom Authorization Requirements

สำหรับ logic ที่ซับซ้อน เช่น "อายุมากกว่า 18 ปี":

```csharp
// Requirements/MinimumAgeRequirement.cs
using Microsoft.AspNetCore.Authorization;

namespace AuthApp.Requirements;

// 1. Requirement - defines what we need
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    
    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// 2. Handler - handles the requirement
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        // ดึง dateOfBirth จาก claims
        var dateOfBirthClaim = context.User
            .FindFirst(c => c.Type == "dateOfBirth");
        
        if (dateOfBirthClaim == null)
        {
            context.Fail();  // ไม่มี claim = ไม่ผ่าน
            return Task.CompletedTask;
        }
        
        if (!DateTime.TryParse(dateOfBirthClaim.Value, out var dateOfBirth))
        {
            context.Fail();
            return Task.CompletedTask;
        }
        
        var age = DateTime.Today.Year - dateOfBirth.Year;
        if (dateOfBirth.Date > DateTime.Today.AddYears(-age))
            age--;
        
        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);  // ผ่าน
        else
            context.Fail();
        
        return Task.CompletedTask;
    }
}
```

### Register Custom Handler

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AtLeast18", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
    
    options.AddPolicy("AtLeast21", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(21)));
});

// Register handler
builder.Services.AddScoped<IAuthorizationHandler, MinimumAgeHandler>();
```

---

## Employee Permission System

```csharp
// Requirements/PermissionRequirement.cs
using Microsoft.AspNetCore.Authorization;

namespace AuthApp.Requirements;

public class PermissionRequirement : IAuthorizationRequirement
{
    public string Permission { get; }
    
    public PermissionRequirement(string permission)
    {
        Permission = permission;
    }
}

public class PermissionHandler : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        // Admin ผ่านทุก permission อัตโนมัติ
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        // ตรวจสอบ permission claim
        var hasPermission = context.User.HasClaim(
            c => c.Type == "permission" && c.Value == requirement.Permission);
        
        if (hasPermission)
            context.Succeed(requirement);
        
        return Task.CompletedTask;
    }
}

// Extension สำหรับสร้าง permissions ง่ายขึ้น
public static class PermissionPolicies
{
    public const string ReadProducts = "products:read";
    public const string WriteProducts = "products:write";
    public const string DeleteProducts = "products:delete";
    public const string ManageUsers = "users:manage";
    public const string ViewReports = "reports:view";
    
    public static void AddPermissionPolicies(AuthorizationOptions options)
    {
        options.AddPolicy(ReadProducts, p => 
            p.Requirements.Add(new PermissionRequirement(ReadProducts)));
        options.AddPolicy(WriteProducts, p => 
            p.Requirements.Add(new PermissionRequirement(WriteProducts)));
        options.AddPolicy(DeleteProducts, p => 
            p.Requirements.Add(new PermissionRequirement(DeleteProducts)));
        options.AddPolicy(ManageUsers, p => 
            p.Requirements.Add(new PermissionRequirement(ManageUsers)));
        options.AddPolicy(ViewReports, p => 
            p.Requirements.Add(new PermissionRequirement(ViewReports)));
    }
}
```

---

## Resource-based Authorization

เมื่อ authorization ขึ้นอยู่กับ resource ที่กำลัง access

```csharp
// Requirements/DocumentAuthorizationRequirement.cs
using Microsoft.AspNetCore.Authorization;

namespace AuthApp.Requirements;

public enum DocumentOperation
{
    Read,
    Update,
    Delete,
    Share
}

public class DocumentAuthorizationHandler 
    : AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Document document)
    {
        var userId = context.User.FindFirst(
            System.Security.Claims.ClaimTypes.NameIdentifier)?.Value;
        
        if (userId == null)
        {
            context.Fail();
            return Task.CompletedTask;
        }
        
        // Admin ทำได้ทุกอย่าง
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        // Owner ทำได้ทุกอย่าง
        if (document.OwnerId == userId)
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        // Shared users อ่านได้อย่างเดียว
        if (requirement.Name == nameof(DocumentOperation.Read) &&
            document.SharedWithUserIds.Contains(userId))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }
        
        return Task.CompletedTask;
    }
}

// Document model
public class Document
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public string OwnerId { get; set; } = string.Empty;
    public List<string> SharedWithUserIds { get; set; } = [];
    public bool IsPublic { get; set; }
}

// Operations
public static class DocumentOperations
{
    public static readonly OperationAuthorizationRequirement Read =
        new() { Name = nameof(DocumentOperation.Read) };
    public static readonly OperationAuthorizationRequirement Update =
        new() { Name = nameof(DocumentOperation.Update) };
    public static readonly OperationAuthorizationRequirement Delete =
        new() { Name = nameof(DocumentOperation.Delete) };
    public static readonly OperationAuthorizationRequirement Share =
        new() { Name = nameof(DocumentOperation.Share) };
}
```

### ใช้งาน Resource-based Authorization

```csharp
// Controllers/DocumentController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace AuthApp.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class DocumentController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;
    private readonly IDocumentService _documentService;
    
    public DocumentController(
        IAuthorizationService authorizationService,
        IDocumentService documentService)
    {
        _authorizationService = authorizationService;
        _documentService = documentService;
    }
    
    [HttpGet("{id}")]
    public async Task<IActionResult> GetDocument(int id)
    {
        var document = await _documentService.GetDocumentAsync(id);
        if (document == null) return NotFound();
        
        // Resource-based authorization check
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Read);
        
        if (!authResult.Succeeded)
            return Forbid();
        
        return Ok(document);
    }
    
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateDocument(int id, [FromBody] UpdateDocumentRequest request)
    {
        var document = await _documentService.GetDocumentAsync(id);
        if (document == null) return NotFound();
        
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Update);
        
        if (!authResult.Succeeded)
            return Forbid();
        
        await _documentService.UpdateDocumentAsync(id, request);
        return Ok();
    }
    
    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteDocument(int id)
    {
        var document = await _documentService.GetDocumentAsync(id);
        if (document == null) return NotFound();
        
        var authResult = await _authorizationService.AuthorizeAsync(
            User, document, DocumentOperations.Delete);
        
        if (!authResult.Succeeded)
            return Forbid();
        
        await _documentService.DeleteDocumentAsync(id);
        return NoContent();
    }
}
```

---

## โปรแกรมตัวอย่าง: Multi-role System

### Models

```csharp
// Models สำหรับ Multi-role System
namespace MultiRoleApp.Models;

public class AppUser
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public string Email { get; set; } = string.Empty;
    public string Username { get; set; } = string.Empty;
    public string Department { get; set; } = string.Empty;
    public bool IsActive { get; set; } = true;
    
    public List<string> Roles { get; set; } = [];
    public List<string> Permissions { get; set; } = [];
}

public class Resource
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Type { get; set; } = string.Empty;
    public string OwnerId { get; set; } = string.Empty;
    public bool IsPublic { get; set; }
    public List<string> AllowedUserIds { get; set; } = [];
    public List<string> AllowedRoles { get; set; } = [];
}
```

### Policy Definitions

```csharp
// Authorization/Policies.cs
using Microsoft.AspNetCore.Authorization;
using MultiRoleApp.Requirements;

namespace MultiRoleApp.Authorization;

public static class AppPolicies
{
    public const string AdminOnly = "AdminOnly";
    public const string ManageUsers = "ManageUsers";
    public const string ViewReports = "ViewReports";
    public const string PublishContent = "PublishContent";
    public const string SameDepartment = "SameDepartment";
    public const string ActiveUser = "ActiveUser";
    
    public static void Configure(AuthorizationOptions options)
    {
        // Simple role-based
        options.AddPolicy(AdminOnly, p => p.RequireRole("Admin"));
        
        // Multi-role
        options.AddPolicy(ManageUsers, p => 
            p.RequireRole("Admin", "HR"));
        
        // Claim-based
        options.AddPolicy(ViewReports, p => 
            p.RequireClaim("permission", "reports:view"));
        
        // Combined
        options.AddPolicy(PublishContent, p =>
        {
            p.RequireAuthenticatedUser();
            p.RequireRole("Editor", "Admin");
            p.RequireClaim("permission", "content:publish");
        });
        
        // Custom requirement
        options.AddPolicy(SameDepartment, p =>
            p.Requirements.Add(new SameDepartmentRequirement()));
        
        // Active user check
        options.AddPolicy(ActiveUser, p =>
            p.Requirements.Add(new ActiveUserRequirement()));
    }
}
```

### Custom Requirements

```csharp
// Requirements/ActiveUserRequirement.cs
using Microsoft.AspNetCore.Authorization;

namespace MultiRoleApp.Requirements;

public class ActiveUserRequirement : IAuthorizationRequirement { }

public class ActiveUserHandler : AuthorizationHandler<ActiveUserRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ActiveUserRequirement requirement)
    {
        // ตรวจสอบ isActive claim
        var isActiveClaim = context.User.FindFirst("isActive")?.Value;
        
        if (bool.TryParse(isActiveClaim, out var isActive) && isActive)
            context.Succeed(requirement);
        else
            context.Fail(new AuthorizationFailureReason(this, "User account is inactive"));
        
        return Task.CompletedTask;
    }
}

public class SameDepartmentRequirement : IAuthorizationRequirement { }

public class SameDepartmentHandler : AuthorizationHandler<SameDepartmentRequirement, AppUser>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameDepartmentRequirement requirement,
        AppUser resource)
    {
        var userDepartment = context.User.FindFirst("department")?.Value;
        
        if (context.User.IsInRole("Admin") || 
            userDepartment == resource.Department)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}
```

### Controllers

```csharp
// Controllers/AdminController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using MultiRoleApp.Authorization;
using MultiRoleApp.Services;

namespace MultiRoleApp.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class AdminController : ControllerBase
{
    private readonly IUserService _userService;
    
    public AdminController(IUserService userService)
    {
        _userService = userService;
    }
    
    // Admin เท่านั้น
    [HttpGet("dashboard")]
    [Authorize(Policy = AppPolicies.AdminOnly)]
    public IActionResult GetDashboard()
    {
        return Ok(new 
        { 
            Message = "Admin Dashboard",
            Timestamp = DateTime.UtcNow 
        });
    }
    
    // Admin หรือ HR
    [HttpGet("users")]
    [Authorize(Policy = AppPolicies.ManageUsers)]
    public async Task<IActionResult> GetAllUsers()
    {
        var users = await _userService.GetAllUsersAsync();
        return Ok(users);
    }
    
    // Claim-based
    [HttpGet("reports")]
    [Authorize(Policy = AppPolicies.ViewReports)]
    public IActionResult GetReports()
    {
        return Ok(new { Report = "Financial Summary", Data = new[] { 100, 200, 300 } });
    }
    
    // ต้องเป็น Admin เท่านั้น (ไม่ใช้ policy)
    [HttpPost("users/{id}/deactivate")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> DeactivateUser(string id)
    {
        await _userService.DeactivateUserAsync(id);
        return Ok(new { Message = $"User {id} deactivated" });
    }
    
    // ตรวจสอบ authorization ใน code
    [HttpPost("users/{id}/assign-role")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> AssignRole(
        string id,
        [FromBody] AssignRoleRequest request)
    {
        // ตรวจสอบว่า admin กำลัง assign role สูงกว่าตัวเอง
        if (request.Role == "SuperAdmin" && !User.IsInRole("SuperAdmin"))
            return Forbid();
        
        await _userService.AssignRoleAsync(id, request.Role);
        return Ok(new { Message = $"Role {request.Role} assigned to user {id}" });
    }
}

public record AssignRoleRequest(string Role);
```

```csharp
// Controllers/ContentController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using MultiRoleApp.Authorization;

namespace MultiRoleApp.Controllers;

[ApiController]
[Route("api/[controller]")]
public class ContentController : ControllerBase
{
    private readonly IAuthorizationService _authService;
    
    public ContentController(IAuthorizationService authService)
    {
        _authService = authService;
    }
    
    // ทุกคนดูได้
    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetPublicContent()
    {
        return Ok(new[] { "Article 1", "Article 2", "Article 3" });
    }
    
    // Login แล้วดูได้
    [HttpGet("premium")]
    [Authorize]
    public IActionResult GetPremiumContent()
    {
        return Ok(new[] { "Premium Article 1", "Premium Article 2" });
    }
    
    // ต้องมี permission
    [HttpPost]
    [Authorize(Policy = AppPolicies.PublishContent)]
    public IActionResult CreateContent([FromBody] CreateContentRequest request)
    {
        return CreatedAtAction(nameof(GetPublicContent), new { id = 1 }, request);
    }
    
    // Resource-based authorization
    [HttpDelete("{id}")]
    [Authorize]
    public async Task<IActionResult> DeleteContent(int id)
    {
        // ดึง content
        var content = GetContentById(id);
        if (content == null) return NotFound();
        
        // ตรวจสอบ ownership หรือ Admin
        var userId = User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value;
        if (content.OwnerId != userId && !User.IsInRole("Admin"))
            return Forbid();
        
        return NoContent();
    }
    
    private static ContentItem? GetContentById(int id)
    {
        // Mock data
        return id == 1 ? new ContentItem(1, "Test", "owner-123") : null;
    }
}

public record CreateContentRequest(string Title, string Body);
public record ContentItem(int Id, string Title, string OwnerId);
```

### Program.cs สมบูรณ์

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using MultiRoleApp.Authorization;
using MultiRoleApp.Requirements;
using MultiRoleApp.Services;

var builder = WebApplication.CreateBuilder(args);

// Authentication
var secretKey = builder.Configuration["JwtSettings:SecretKey"]!;
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(secretKey)),
            ValidateIssuer = false,
            ValidateAudience = false,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });

// Authorization with Policies
builder.Services.AddAuthorization(options =>
{
    AppPolicies.Configure(options);
    
    // Default policy - ต้อง authenticate
    options.DefaultPolicy = new Microsoft.AspNetCore.Authorization.AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    
    // Fallback policy - ต้อง authenticate (สำหรับ endpoints ที่ไม่มี attribute)
    // options.FallbackPolicy = options.DefaultPolicy;
});

// Register Authorization Handlers
builder.Services.AddScoped<
    Microsoft.AspNetCore.Authorization.IAuthorizationHandler, 
    ActiveUserHandler>();
builder.Services.AddScoped<
    Microsoft.AspNetCore.Authorization.IAuthorizationHandler, 
    MinimumAgeHandler>();

// Services
builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## Global Authorization (Fallback Policy)

```csharp
// ตั้ง Global Authorization - ทุก endpoint ต้อง authenticate
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// Endpoint ที่ต้องการ public ต้องใส่ [AllowAnonymous] เอง
[AllowAnonymous]
public IActionResult PublicEndpoint() { ... }
```

---

## Custom Authorization Middleware

```csharp
// Middleware/ApiKeyAuthMiddleware.cs
namespace MultiRoleApp.Middleware;

public class ApiKeyAuthMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _config;
    private const string ApiKeyHeader = "X-API-Key";
    
    public ApiKeyAuthMiddleware(RequestDelegate next, IConfiguration config)
    {
        _next = next;
        _config = config;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        // เฉพาะ /api/external/* ต้องใช้ API Key
        if (!context.Request.Path.StartsWithSegments("/api/external"))
        {
            await _next(context);
            return;
        }
        
        if (!context.Request.Headers.TryGetValue(ApiKeyHeader, out var apiKeyValue))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsJsonAsync(new { Message = "API Key is required" });
            return;
        }
        
        var validApiKeys = _config.GetSection("ApiKeys").Get<string[]>() ?? [];
        
        if (!validApiKeys.Contains(apiKeyValue.ToString()))
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsJsonAsync(new { Message = "Invalid API Key" });
            return;
        }
        
        await _next(context);
    }
}

// Extension method
public static class ApiKeyAuthMiddlewareExtensions
{
    public static IApplicationBuilder UseApiKeyAuth(this IApplicationBuilder app)
        => app.UseMiddleware<ApiKeyAuthMiddleware>();
}

// Program.cs
app.UseApiKeyAuth();
app.UseAuthentication();
app.UseAuthorization();
```

---

## Authorization ใน Minimal APIs

```csharp
// Minimal API endpoints
app.MapGet("/api/products", async (ProductService service) =>
{
    return await service.GetAllAsync();
})
.RequireAuthorization();  // ต้อง authenticate

app.MapPost("/api/products", async (CreateProductRequest req, ProductService service) =>
{
    return await service.CreateAsync(req);
})
.RequireAuthorization("AdminOnly");  // ต้องเป็น Admin

app.MapGet("/api/public", () => "Public content")
.AllowAnonymous();

app.MapGet("/api/reports", () => "Reports")
.RequireAuthorization(policy =>
{
    policy.RequireRole("Admin", "Manager");
    policy.RequireClaim("department");
});
```

---

## Testing Authorization

```csharp
// Tests/AuthorizationTests.cs
using Microsoft.AspNetCore.Mvc.Testing;
using System.Net.Http.Headers;
using System.Net;
using Xunit;

namespace MultiRoleApp.Tests;

public class AuthorizationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    
    public AuthorizationTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory;
    }
    
    [Fact]
    public async Task AdminEndpoint_WithoutToken_Returns401()
    {
        var client = _factory.CreateClient();
        
        var response = await client.GetAsync("/api/admin/dashboard");
        
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
    
    [Fact]
    public async Task AdminEndpoint_WithUserToken_Returns403()
    {
        var client = _factory.CreateClient();
        var token = GenerateTestToken(roles: ["User"]);
        
        client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        var response = await client.GetAsync("/api/admin/dashboard");
        
        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }
    
    [Fact]
    public async Task AdminEndpoint_WithAdminToken_Returns200()
    {
        var client = _factory.CreateClient();
        var token = GenerateTestToken(roles: ["Admin"]);
        
        client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);
        
        var response = await client.GetAsync("/api/admin/dashboard");
        
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
    
    [Fact]
    public async Task PublicEndpoint_WithoutToken_Returns200()
    {
        var client = _factory.CreateClient();
        
        var response = await client.GetAsync("/api/content");
        
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
    
    private static string GenerateTestToken(string[] roles)
    {
        // สร้าง test JWT token
        // ใช้ System.IdentityModel.Tokens.Jwt
        var handler = new System.IdentityModel.Tokens.Jwt.JwtSecurityTokenHandler();
        var claims = new System.Security.Claims.Claim[]
        {
            new("sub", "test-user-id"),
            new("email", "test@example.com"),
        }.Concat(roles.Select(r => 
            new System.Security.Claims.Claim(
                System.Security.Claims.ClaimTypes.Role, r)))
        .ToArray();
        
        var tokenDescriptor = new Microsoft.IdentityModel.Tokens.SecurityTokenDescriptor
        {
            Subject = new System.Security.Claims.ClaimsIdentity(claims),
            Expires = DateTime.UtcNow.AddHours(1),
            SigningCredentials = new Microsoft.IdentityModel.Tokens.SigningCredentials(
                new Microsoft.IdentityModel.Tokens.SymmetricSecurityKey(
                    System.Text.Encoding.UTF8.GetBytes("test-secret-key-32-characters-long!")),
                Microsoft.IdentityModel.Tokens.SecurityAlgorithms.HmacSha256)
        };
        
        var token = handler.CreateToken(tokenDescriptor);
        return handler.WriteToken(token);
    }
}
```

---

## สรุป Permission Matrix

```
Role/Permission   | Admin | Manager | Editor | User
------------------|-------|---------|--------|------
View Content      |  ✓    |   ✓     |   ✓    |  ✓
Create Content    |  ✓    |   ✓     |   ✓    |  ✗
Edit Own Content  |  ✓    |   ✓     |   ✓    |  ✓
Edit All Content  |  ✓    |   ✓     |   ✗    |  ✗
Delete Content    |  ✓    |   ✓     |   ✗    |  ✗
View Reports      |  ✓    |   ✓     |   ✗    |  ✗
Manage Users      |  ✓    |   ✗     |   ✗    |  ✗
System Settings   |  ✓    |   ✗     |   ✗    |  ✗
```

---

## Exercises

### แบบฝึกหัดที่ 1: Department-based Access
สร้าง system ที่:
- HR department ดู/แก้ไข employee records ของทุกคน
- Manager ดู/แก้ไขแค่ employee ในทีมตัวเอง
- Employee ดูแค่ข้อมูลตัวเอง

### แบบฝึกหัดที่ 2: Time-based Policy
สร้าง policy ที่:
- ทำ sensitive operations ได้แค่ในเวลาทำการ (08:00-18:00)
- นอกเวลาต้องมี special override permission

### แบบฝึกหัดที่ 3: IP Whitelist
สร้าง middleware ที่:
- Admin endpoints เข้าถึงได้จาก IP whitelist เท่านั้น
- ถ้า IP ไม่ใช่ whitelist return 403 แม้จะมี valid token

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **[Authorize] Attribute**: ทั้ง role, policy, และ custom
2. **Role-based Authorization**: ใช้ Roles ใน claims
3. **Policy-based Authorization**: กฎที่ซับซ้อนกว่า role
4. **Custom Requirements & Handlers**: Logic ที่กำหนดเอง
5. **Resource-based Authorization**: ตรวจสอบบน resource จริง
6. **Authorization Middleware**: Custom logic ก่อน endpoint
7. **Testing**: ทดสอบ authorization ด้วย WebApplicationFactory

---

## Phase 4 สรุป (Part 051-060)

เราได้เรียนรู้ใน Phase 4 ครึ่งแรก:
- **EF Core**: ORM พื้นฐาน, Entities, Querying, CRUD, Migrations, Performance
- **Repository Pattern**: Clean architecture กับ EF Core
- **Authentication**: Cookie, JWT, Identity
- **Authorization**: Roles, Policies, Custom requirements

## Part ถัดไป

ใน **Part 061** เราจะเรียนรู้ **ASP.NET Core Web API ขั้นสูง**:
- REST API best practices
- API Versioning
- Response caching
- Rate limiting
- API documentation ด้วย OpenAPI/Swagger

---

*Part 060/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
