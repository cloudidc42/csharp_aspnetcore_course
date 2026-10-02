# Part 058: Authentication พื้นฐาน

## เนื้อหาใน Part นี้
- Authentication vs Authorization
- Cookie Authentication
- JWT Tokens เบื้องต้น
- Claims
- ASP.NET Core Identity เบื้องต้น
- โปรแกรมตัวอย่าง: Login/Register API

---

## Authentication vs Authorization

### Authentication (การพิสูจน์ตัวตน)

ตรวจสอบว่า "คุณเป็นใคร?" ตัวอย่าง:
- Login ด้วย username/password
- Google Sign-In
- ใช้ fingerprint

### Authorization (การกำหนดสิทธิ์)

ตรวจสอบว่า "คุณทำอะไรได้บ้าง?" ตัวอย่าง:
- User ปกติดูข้อมูลได้ แต่ลบไม่ได้
- Admin ทำได้ทุกอย่าง
- Manager แก้ไขได้แต่ลบไม่ได้

```
Request → Authentication Middleware → Authorization Middleware → Controller
           (ฉันคือใคร?)              (ฉันทำได้ไหม?)
```

---

## ASP.NET Core Identity คืออะไร

ASP.NET Core Identity คือระบบ membership สำเร็จรูปที่มี:
- User management (สร้าง, แก้ไข, ลบ users)
- Password hashing ที่ปลอดภัย
- Roles and Claims
- External login providers (Google, Facebook)
- Token generation
- Lockout

---

## Models

```csharp
// Models/ApplicationUser.cs
using Microsoft.AspNetCore.Identity;

namespace AuthApp.Models;

public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime? DateOfBirth { get; set; }
    public string? ProfileImageUrl { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
    public bool IsActive { get; set; } = true;
    
    public string FullName => $"{FirstName} {LastName}".Trim();
}
```

---

## ติดตั้ง Identity

### NuGet Packages

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 9.0.0
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 9.0.0
dotnet add package System.IdentityModel.Tokens.Jwt --version 8.2.0
```

### DbContext กับ Identity

```csharp
// Data/ApplicationDbContext.cs
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using AuthApp.Models;

namespace AuthApp.Data;

public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);  // ต้องเรียก base ด้วย
        
        // Customize Identity tables
        builder.Entity<ApplicationUser>(e =>
        {
            e.Property(u => u.FirstName).HasMaxLength(100);
            e.Property(u => u.LastName).HasMaxLength(100);
        });
        
        // Seed roles
        builder.Entity<Microsoft.AspNetCore.Identity.IdentityRole>().HasData(
            new Microsoft.AspNetCore.Identity.IdentityRole
            {
                Id = "1",
                Name = "Admin",
                NormalizedName = "ADMIN"
            },
            new Microsoft.AspNetCore.Identity.IdentityRole
            {
                Id = "2",
                Name = "User",
                NormalizedName = "USER"
            },
            new Microsoft.AspNetCore.Identity.IdentityRole
            {
                Id = "3",
                Name = "Manager",
                NormalizedName = "MANAGER"
            }
        );
    }
}
```

---

## Program.cs Setup

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using AuthApp.Data;
using AuthApp.Models;
using AuthApp.Services;

var builder = WebApplication.CreateBuilder(args);

// 1. Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite(
        builder.Configuration.GetConnectionString("Default") 
        ?? "Data Source=auth.db"));

// 2. Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Password requirements
    options.Password.RequiredLength = 8;
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = false;
    
    // Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;
    
    // User settings
    options.User.RequireUniqueEmail = true;
    options.SignIn.RequireConfirmedEmail = false;  // dev mode
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// 3. JWT Authentication
var jwtSettings = builder.Configuration.GetSection("JwtSettings");
var secretKey = jwtSettings["SecretKey"] 
    ?? throw new InvalidOperationException("JWT SecretKey not configured");

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(secretKey)),
        ValidateIssuer = true,
        ValidIssuer = jwtSettings["Issuer"],
        ValidateAudience = true,
        ValidAudience = jwtSettings["Audience"],
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero
    };
});

builder.Services.AddAuthorization();

// 4. Services
builder.Services.AddScoped<ITokenService, TokenService>();
builder.Services.AddScoped<IAuthService, AuthService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    
    // Auto-apply migrations in development
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await context.Database.MigrateAsync();
}

app.UseHttpsRedirection();
app.UseAuthentication();  // ต้องมาก่อน Authorization
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## appsettings.json

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=auth.db"
  },
  "JwtSettings": {
    "SecretKey": "your-super-secret-key-that-is-at-least-32-characters-long-for-security",
    "Issuer": "https://yourapp.com",
    "Audience": "https://yourapp.com",
    "ExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 7
  }
}
```

---

## Claims คืออะไร

**Claims** คือ key-value pairs ที่บอกข้อมูลเกี่ยวกับ user ที่ผ่านการยืนยันแล้ว

```csharp
// ตัวอย่าง Claims ที่พบบ่อย
var claims = new[]
{
    new Claim(ClaimTypes.NameIdentifier, user.Id),   // "sub" - User ID
    new Claim(ClaimTypes.Email, user.Email!),          // Email
    new Claim(ClaimTypes.Name, user.UserName!),        // Username
    new Claim("FirstName", user.FirstName),            // Custom claim
    new Claim(ClaimTypes.Role, "Admin"),               // Role
    new Claim("permission", "product:write"),          // Custom permission
};
```

### อ่าน Claims ใน Controller

```csharp
[HttpGet("me")]
[Authorize]
public IActionResult GetCurrentUser()
{
    var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
    var email = User.FindFirstValue(ClaimTypes.Email);
    var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value);
    
    return Ok(new { UserId = userId, Email = email, Roles = roles });
}
```

---

## Token Service

```csharp
// Services/ITokenService.cs
using AuthApp.Models;

namespace AuthApp.Services;

public interface ITokenService
{
    string GenerateAccessToken(ApplicationUser user, IList<string> roles);
    string GenerateRefreshToken();
    System.Security.Claims.ClaimsPrincipal? ValidateToken(string token);
}
```

```csharp
// Services/TokenService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.IdentityModel.Tokens;
using AuthApp.Models;

namespace AuthApp.Services;

public class TokenService : ITokenService
{
    private readonly IConfiguration _config;
    
    public TokenService(IConfiguration config)
    {
        _config = config;
    }
    
    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var jwtSettings = _config.GetSection("JwtSettings");
        var secretKey = jwtSettings["SecretKey"]!;
        var expirationMinutes = int.Parse(jwtSettings["ExpirationMinutes"] ?? "60");
        
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(ClaimTypes.Name, user.UserName!),
            new("firstName", user.FirstName),
            new("lastName", user.LastName),
        };
        
        // เพิ่ม roles
        foreach (var role in roles)
            claims.Add(new Claim(ClaimTypes.Role, role));
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        
        var token = new JwtSecurityToken(
            issuer: jwtSettings["Issuer"],
            audience: jwtSettings["Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(expirationMinutes),
            signingCredentials: credentials
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    public string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
    
    public ClaimsPrincipal? ValidateToken(string token)
    {
        var jwtSettings = _config.GetSection("JwtSettings");
        var secretKey = jwtSettings["SecretKey"]!;
        
        try
        {
            var tokenHandler = new JwtSecurityTokenHandler();
            var key = Encoding.UTF8.GetBytes(secretKey);
            
            var principal = tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = jwtSettings["Issuer"],
                ValidateAudience = true,
                ValidAudience = jwtSettings["Audience"],
                ValidateLifetime = false  // ไม่ check lifetime ตอน validate (สำหรับ refresh)
            }, out var validatedToken);
            
            if (validatedToken is not JwtSecurityToken jwtToken ||
                !jwtToken.Header.Alg.Equals(
                    SecurityAlgorithms.HmacSha256,
                    StringComparison.InvariantCultureIgnoreCase))
                return null;
            
            return principal;
        }
        catch
        {
            return null;
        }
    }
}
```

---

## Auth DTOs

```csharp
// DTOs/AuthDtos.cs
namespace AuthApp.DTOs;

public record RegisterRequest(
    string FirstName,
    string LastName,
    string Email,
    string Password,
    string ConfirmPassword);

public record LoginRequest(
    string Email,
    string Password,
    bool RememberMe = false);

public record AuthResponse(
    string AccessToken,
    string RefreshToken,
    DateTime ExpiresAt,
    UserInfo User);

public record UserInfo(
    string Id,
    string Email,
    string FullName,
    List<string> Roles);

public record RefreshTokenRequest(
    string AccessToken,
    string RefreshToken);

public record ChangePasswordRequest(
    string CurrentPassword,
    string NewPassword,
    string ConfirmNewPassword);

public record ForgotPasswordRequest(string Email);

public record ResetPasswordRequest(
    string Email,
    string Token,
    string NewPassword);
```

---

## Auth Service

```csharp
// Services/IAuthService.cs
using AuthApp.DTOs;
using AuthApp.Models;

namespace AuthApp.Services;

public interface IAuthService
{
    Task<(bool Success, string? Error, AuthResponse? Response)> RegisterAsync(
        RegisterRequest request);
    Task<(bool Success, string? Error, AuthResponse? Response)> LoginAsync(
        LoginRequest request);
    Task<(bool Success, string? Error, AuthResponse? Response)> RefreshTokenAsync(
        RefreshTokenRequest request);
    Task<bool> LogoutAsync(string userId);
    Task<ApplicationUser?> GetUserByIdAsync(string userId);
}
```

```csharp
// Services/AuthService.cs
using Microsoft.AspNetCore.Identity;
using AuthApp.Data;
using AuthApp.DTOs;
using AuthApp.Models;
using Microsoft.EntityFrameworkCore;

namespace AuthApp.Services;

public class AuthService : IAuthService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly ITokenService _tokenService;
    private readonly ApplicationDbContext _context;
    private readonly ILogger<AuthService> _logger;
    private readonly IConfiguration _config;
    
    public AuthService(
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        ITokenService tokenService,
        ApplicationDbContext context,
        ILogger<AuthService> logger,
        IConfiguration config)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _tokenService = tokenService;
        _context = context;
        _logger = logger;
        _config = config;
    }
    
    public async Task<(bool, string?, AuthResponse?)> RegisterAsync(RegisterRequest request)
    {
        // Validate
        if (request.Password != request.ConfirmPassword)
            return (false, "Passwords do not match", null);
        
        // Check existing user
        var existingUser = await _userManager.FindByEmailAsync(request.Email);
        if (existingUser != null)
            return (false, "Email already in use", null);
        
        // Create user
        var user = new ApplicationUser
        {
            UserName = request.Email,
            Email = request.Email,
            FirstName = request.FirstName,
            LastName = request.LastName,
            EmailConfirmed = true  // Skip email confirmation for now
        };
        
        var createResult = await _userManager.CreateAsync(user, request.Password);
        if (!createResult.Succeeded)
        {
            var errors = string.Join(", ", createResult.Errors.Select(e => e.Description));
            return (false, errors, null);
        }
        
        // Assign default role
        await _userManager.AddToRoleAsync(user, "User");
        
        // Generate tokens
        var roles = await _userManager.GetRolesAsync(user);
        var accessToken = _tokenService.GenerateAccessToken(user, roles);
        var refreshToken = _tokenService.GenerateRefreshToken();
        
        // Save refresh token
        user.RefreshToken = refreshToken;
        user.RefreshTokenExpiry = DateTime.UtcNow.AddDays(
            int.Parse(_config["JwtSettings:RefreshTokenExpirationDays"] ?? "7"));
        await _userManager.UpdateAsync(user);
        
        _logger.LogInformation("New user registered: {Email}", user.Email);
        
        var expMinutes = int.Parse(_config["JwtSettings:ExpirationMinutes"] ?? "60");
        return (true, null, new AuthResponse(
            accessToken,
            refreshToken,
            DateTime.UtcNow.AddMinutes(expMinutes),
            new UserInfo(user.Id, user.Email!, user.FullName, [.. roles])));
    }
    
    public async Task<(bool, string?, AuthResponse?)> LoginAsync(LoginRequest request)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);
        if (user == null)
            return (false, "Invalid email or password", null);
        
        if (!user.IsActive)
            return (false, "Account is disabled", null);
        
        // Check lockout
        if (await _userManager.IsLockedOutAsync(user))
            return (false, "Account is locked. Try again later.", null);
        
        // Verify password
        var passwordValid = await _userManager.CheckPasswordAsync(user, request.Password);
        if (!passwordValid)
        {
            await _userManager.AccessFailedAsync(user);
            return (false, "Invalid email or password", null);
        }
        
        // Reset failed attempts
        await _userManager.ResetAccessFailedCountAsync(user);
        
        // Update last login
        user.LastLoginAt = DateTime.UtcNow;
        
        // Generate tokens
        var roles = await _userManager.GetRolesAsync(user);
        var accessToken = _tokenService.GenerateAccessToken(user, roles);
        var refreshToken = _tokenService.GenerateRefreshToken();
        
        user.RefreshToken = refreshToken;
        user.RefreshTokenExpiry = DateTime.UtcNow.AddDays(
            int.Parse(_config["JwtSettings:RefreshTokenExpirationDays"] ?? "7"));
        
        await _userManager.UpdateAsync(user);
        
        _logger.LogInformation("User logged in: {Email}", user.Email);
        
        var expMinutes = int.Parse(_config["JwtSettings:ExpirationMinutes"] ?? "60");
        return (true, null, new AuthResponse(
            accessToken,
            refreshToken,
            DateTime.UtcNow.AddMinutes(expMinutes),
            new UserInfo(user.Id, user.Email!, user.FullName, [.. roles])));
    }
    
    public async Task<(bool, string?, AuthResponse?)> RefreshTokenAsync(
        RefreshTokenRequest request)
    {
        var principal = _tokenService.ValidateToken(request.AccessToken);
        if (principal == null)
            return (false, "Invalid access token", null);
        
        var userId = principal.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value;
        if (userId == null)
            return (false, "Invalid token claims", null);
        
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null || 
            user.RefreshToken != request.RefreshToken ||
            user.RefreshTokenExpiry <= DateTime.UtcNow)
            return (false, "Invalid or expired refresh token", null);
        
        var roles = await _userManager.GetRolesAsync(user);
        var newAccessToken = _tokenService.GenerateAccessToken(user, roles);
        var newRefreshToken = _tokenService.GenerateRefreshToken();
        
        user.RefreshToken = newRefreshToken;
        user.RefreshTokenExpiry = DateTime.UtcNow.AddDays(7);
        await _userManager.UpdateAsync(user);
        
        var expMinutes = int.Parse(_config["JwtSettings:ExpirationMinutes"] ?? "60");
        return (true, null, new AuthResponse(
            newAccessToken,
            newRefreshToken,
            DateTime.UtcNow.AddMinutes(expMinutes),
            new UserInfo(user.Id, user.Email!, user.FullName, [.. roles])));
    }
    
    public async Task<bool> LogoutAsync(string userId)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null) return false;
        
        user.RefreshToken = null;
        user.RefreshTokenExpiry = null;
        await _userManager.UpdateAsync(user);
        
        return true;
    }
    
    public async Task<ApplicationUser?> GetUserByIdAsync(string userId)
    {
        return await _userManager.FindByIdAsync(userId);
    }
}
```

### เพิ่ม Refresh Token Fields ใน ApplicationUser

```csharp
// Models/ApplicationUser.cs
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime? DateOfBirth { get; set; }
    public string? ProfileImageUrl { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
    public bool IsActive { get; set; } = true;
    
    // Refresh Token
    public string? RefreshToken { get; set; }
    public DateTime? RefreshTokenExpiry { get; set; }
    
    public string FullName => $"{FirstName} {LastName}".Trim();
}
```

---

## Auth Controller

```csharp
// Controllers/AuthController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using AuthApp.DTOs;
using AuthApp.Services;

namespace AuthApp.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;
    
    public AuthController(IAuthService authService)
    {
        _authService = authService;
    }
    
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);
        
        var (success, error, response) = await _authService.RegisterAsync(request);
        
        if (!success)
            return BadRequest(new { Message = error });
        
        return Ok(response);
    }
    
    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);
        
        var (success, error, response) = await _authService.LoginAsync(request);
        
        if (!success)
            return Unauthorized(new { Message = error });
        
        return Ok(response);
    }
    
    [HttpPost("refresh")]
    public async Task<IActionResult> RefreshToken([FromBody] RefreshTokenRequest request)
    {
        var (success, error, response) = await _authService.RefreshTokenAsync(request);
        
        if (!success)
            return Unauthorized(new { Message = error });
        
        return Ok(response);
    }
    
    [HttpPost("logout")]
    [Authorize]
    public async Task<IActionResult> Logout()
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId == null)
            return Unauthorized();
        
        await _authService.LogoutAsync(userId);
        
        return Ok(new { Message = "Logged out successfully" });
    }
    
    [HttpGet("me")]
    [Authorize]
    public IActionResult GetCurrentUser()
    {
        var claims = User.Claims.Select(c => new { c.Type, c.Value }).ToList();
        
        return Ok(new
        {
            UserId = User.FindFirstValue(ClaimTypes.NameIdentifier),
            Email = User.FindFirstValue(ClaimTypes.Email),
            Name = User.FindFirstValue(ClaimTypes.Name),
            FirstName = User.FindFirstValue("firstName"),
            Roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList(),
            Claims = claims
        });
    }
    
    [HttpGet("admin-only")]
    [Authorize(Roles = "Admin")]
    public IActionResult AdminOnly()
    {
        return Ok(new { Message = "You are an Admin!" });
    }
    
    [HttpGet("user-or-above")]
    [Authorize(Roles = "User,Admin,Manager")]
    public IActionResult UserOrAbove()
    {
        return Ok(new { Message = "Welcome authenticated user!" });
    }
}
```

---

## Cookie Authentication (alternative)

สำหรับ web apps ที่ไม่ใช่ API:

```csharp
// Program.cs - Cookie Auth
builder.Services.AddAuthentication("Cookies")
    .AddCookie("Cookies", options =>
    {
        options.LoginPath = "/auth/login";
        options.LogoutPath = "/auth/logout";
        options.AccessDeniedPath = "/auth/access-denied";
        options.ExpireTimeSpan = TimeSpan.FromHours(1);
        options.SlidingExpiration = true;
    });

// Controller - Cookie Login
[HttpPost("login")]
public async Task<IActionResult> Login(LoginRequest request)
{
    var user = await _userManager.FindByEmailAsync(request.Email);
    if (user == null || !await _userManager.CheckPasswordAsync(user, request.Password))
        return Unauthorized();
    
    var claims = new List<Claim>
    {
        new(ClaimTypes.NameIdentifier, user.Id),
        new(ClaimTypes.Email, user.Email!),
        new(ClaimTypes.Name, user.FullName)
    };
    
    var roles = await _userManager.GetRolesAsync(user);
    claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));
    
    var identity = new ClaimsIdentity(claims, "Cookies");
    var principal = new ClaimsPrincipal(identity);
    
    await HttpContext.SignInAsync("Cookies", principal, new AuthenticationProperties
    {
        IsPersistent = request.RememberMe,
        ExpiresUtc = request.RememberMe 
            ? DateTimeOffset.UtcNow.AddDays(30)
            : DateTimeOffset.UtcNow.AddHours(1)
    });
    
    return Ok();
}

[HttpPost("logout")]
public async Task<IActionResult> Logout()
{
    await HttpContext.SignOutAsync("Cookies");
    return Ok();
}
```

---

## Testing the API

### Register

```http
POST /api/auth/register
Content-Type: application/json

{
  "firstName": "สมชาย",
  "lastName": "ใจดี",
  "email": "somchai@example.com",
  "password": "Password1!",
  "confirmPassword": "Password1!"
}
```

### Login

```http
POST /api/auth/login
Content-Type: application/json

{
  "email": "somchai@example.com",
  "password": "Password1!"
}
```

Response:
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "base64-encoded-refresh-token...",
  "expiresAt": "2024-10-01T13:00:00Z",
  "user": {
    "id": "abc123",
    "email": "somchai@example.com",
    "fullName": "สมชาย ใจดี",
    "roles": ["User"]
  }
}
```

### Use Token

```http
GET /api/auth/me
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## Exercises

### แบบฝึกหัดที่ 1: Email Verification
เพิ่มระบบ email verification:
1. ส่ง email confirmation เมื่อ register
2. Endpoint สำหรับ confirm email ด้วย token
3. ต้อง verify email ก่อน login ได้

### แบบฝึกหัดที่ 2: Password Reset
สร้าง forgot password flow:
1. `POST /auth/forgot-password` - ส่ง token ทาง email
2. `POST /auth/reset-password` - reset ด้วย token

### แบบฝึกหัดที่ 3: Two-Factor Authentication
เพิ่ม 2FA:
1. สร้าง TOTP secret สำหรับ user
2. Verify ด้วย 6-digit code ตอน login
3. Backup codes

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Authentication vs Authorization**: ใครคุณ vs ทำอะไรได้
2. **ASP.NET Core Identity**: User management สำเร็จรูป
3. **JWT Authentication**: Token-based auth สำหรับ APIs
4. **Cookie Authentication**: Session-based auth สำหรับ web
5. **Claims**: Key-value pairs บอกข้อมูล user
6. **Refresh Tokens**: ต่ออายุ session โดยไม่ต้อง login ใหม่

---

## Part ถัดไป

ใน **Part 059** เราจะเรียนรู้ **JWT Authentication** อย่างละเอียด:
- โครงสร้าง JWT (Header.Payload.Signature)
- สร้างและ validate JWT
- Refresh token strategy
- Claims-based authorization

---

*Part 058/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
