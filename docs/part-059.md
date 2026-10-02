# Part 059: JWT Authentication ใน ASP.NET Core

## เนื้อหาใน Part นี้
- JWT Structure (Header.Payload.Signature)
- สร้าง JWT Token
- Validate JWT
- Refresh Tokens
- Claims-based Authorization
- โปรแกรมตัวอย่าง: Secure API with JWT

---

## JWT คืออะไร

**JWT (JSON Web Token)** คือ open standard (RFC 7519) สำหรับส่ง information อย่างปลอดภัยระหว่าง parties ในรูปแบบ JSON

### ทำไมใช้ JWT?

| Feature | Session (Cookie) | JWT |
|---------|------------------|-----|
| State | Server-side | Stateless |
| Storage | Server memory/DB | Client |
| Scale | ยาก (sticky session) | ง่าย |
| Mobile | ไม่เหมาะ | เหมาะ |
| Microservices | ยาก | ง่าย |
| Revoke | ง่าย | ยาก |

---

## JWT Structure

JWT มีรูปแบบ: `xxxxx.yyyyy.zzzzz`

### 1. Header

```json
{
  "alg": "HS256",    // Algorithm: HMAC SHA256
  "typ": "JWT"       // Type
}
```

Encoded (Base64Url):
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

### 2. Payload (Claims)

```json
{
  "sub": "user-id-123",           // Subject (User ID)
  "email": "user@example.com",
  "name": "สมชาย ใจดี",
  "role": ["User", "Manager"],
  "iat": 1698835200,              // Issued At (Unix timestamp)
  "exp": 1698838800,              // Expiration (Unix timestamp)
  "iss": "https://myapp.com",     // Issuer
  "aud": "https://myapp.com"      // Audience
}
```

Encoded:
```
eyJzdWIiOiJ1c2VyLWlkLTEyMyIsImVtYWlsIjoiLi4uIn0
```

### 3. Signature

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key
)
```

**ข้อสำคัญ**: JWT ถูก encode ไม่ใช่ encrypt! ใครก็อ่าน Payload ได้ แต่แก้ไขไม่ได้โดยไม่รู้ secret key

---

## Setup Project

```bash
dotnet new webapi -n JwtApi
cd JwtApi
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 9.0.0
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 9.0.0
dotnet add package System.IdentityModel.Tokens.Jwt --version 8.2.0
```

---

## JwtSettings Configuration

```csharp
// Settings/JwtSettings.cs
namespace JwtApi.Settings;

public class JwtSettings
{
    public const string SectionName = "JwtSettings";
    
    public string SecretKey { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int AccessTokenExpirationMinutes { get; set; } = 60;
    public int RefreshTokenExpirationDays { get; set; } = 7;
}
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Data Source=jwt_api.db"
  },
  "JwtSettings": {
    "SecretKey": "super-secret-key-minimum-32-characters-long-for-hs256",
    "Issuer": "https://api.myapp.com",
    "Audience": "https://myapp.com",
    "AccessTokenExpirationMinutes": 15,
    "RefreshTokenExpirationDays": 7
  }
}
```

---

## Models

```csharp
// Models/User.cs
namespace JwtApi.Models;

public class User
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string Email { get; set; } = string.Empty;
    public string Username { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
    
    public List<UserRole> UserRoles { get; set; } = [];
    public List<RefreshToken> RefreshTokens { get; set; } = [];
    
    public string FullName => $"{FirstName} {LastName}".Trim();
}
```

```csharp
// Models/Role.cs
namespace JwtApi.Models;

public class Role
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    
    public List<UserRole> UserRoles { get; set; } = [];
}

public class UserRole
{
    public Guid UserId { get; set; }
    public int RoleId { get; set; }
    public DateTime AssignedAt { get; set; } = DateTime.UtcNow;
    
    public User User { get; set; } = null!;
    public Role Role { get; set; } = null!;
}
```

```csharp
// Models/RefreshToken.cs
namespace JwtApi.Models;

public class RefreshToken
{
    public int Id { get; set; }
    public string Token { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime ExpiresAt { get; set; }
    public bool IsRevoked { get; set; }
    public string? RevokedReason { get; set; }
    public DateTime? RevokedAt { get; set; }
    public string? ReplacedByToken { get; set; }
    public string? CreatedByIp { get; set; }
    
    public Guid UserId { get; set; }
    public User User { get; set; } = null!;
    
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    public bool IsActive => !IsRevoked && !IsExpired;
}
```

---

## DbContext

```csharp
// Data/JwtDbContext.cs
using Microsoft.EntityFrameworkCore;
using JwtApi.Models;

namespace JwtApi.Data;

public class JwtDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Role> Roles { get; set; }
    public DbSet<UserRole> UserRoles { get; set; }
    public DbSet<RefreshToken> RefreshTokens { get; set; }
    
    public JwtDbContext(DbContextOptions<JwtDbContext> options) : base(options) { }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>(e =>
        {
            e.HasKey(u => u.Id);
            e.Property(u => u.Email).IsRequired().HasMaxLength(200);
            e.HasIndex(u => u.Email).IsUnique();
            e.Property(u => u.Username).IsRequired().HasMaxLength(100);
            e.HasIndex(u => u.Username).IsUnique();
            e.Ignore(u => u.FullName);
        });
        
        modelBuilder.Entity<Role>(e =>
        {
            e.HasKey(r => r.Id);
            e.Property(r => r.Name).IsRequired().HasMaxLength(50);
            e.HasIndex(r => r.Name).IsUnique();
            
            e.HasData(
                new Role { Id = 1, Name = "Admin", Description = "Full access" },
                new Role { Id = 2, Name = "User", Description = "Basic access" },
                new Role { Id = 3, Name = "Manager", Description = "Management access" }
            );
        });
        
        modelBuilder.Entity<UserRole>(e =>
        {
            e.HasKey(ur => new { ur.UserId, ur.RoleId });
            
            e.HasOne(ur => ur.User)
             .WithMany(u => u.UserRoles)
             .HasForeignKey(ur => ur.UserId);
             
            e.HasOne(ur => ur.Role)
             .WithMany(r => r.UserRoles)
             .HasForeignKey(ur => ur.RoleId);
        });
        
        modelBuilder.Entity<RefreshToken>(e =>
        {
            e.HasKey(rt => rt.Id);
            e.HasIndex(rt => rt.Token).IsUnique();
            e.Ignore(rt => rt.IsExpired);
            e.Ignore(rt => rt.IsActive);
            
            e.HasOne(rt => rt.User)
             .WithMany(u => u.RefreshTokens)
             .HasForeignKey(rt => rt.UserId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

---

## JWT Service

```csharp
// Services/JwtService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using JwtApi.Data;
using JwtApi.Models;
using JwtApi.Settings;

namespace JwtApi.Services;

public class JwtService
{
    private readonly JwtSettings _settings;
    private readonly JwtDbContext _context;
    private readonly IHttpContextAccessor _httpContextAccessor;
    
    public JwtService(
        IOptions<JwtSettings> settings,
        JwtDbContext context,
        IHttpContextAccessor httpContextAccessor)
    {
        _settings = settings.Value;
        _context = context;
        _httpContextAccessor = httpContextAccessor;
    }
    
    // สร้าง Access Token
    public string GenerateAccessToken(User user, IEnumerable<string> roles)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_settings.SecretKey);
        
        var claims = new List<Claim>
        {
            // Standard Claims
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Name, user.Username),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, 
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
            
            // Custom Claims
            new("firstName", user.FirstName),
            new("lastName", user.LastName),
            new("userId", user.Id.ToString()),
        };
        
        // เพิ่ม roles
        foreach (var role in roles)
            claims.Add(new Claim(ClaimTypes.Role, role));
        
        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(claims),
            Expires = DateTime.UtcNow.AddMinutes(_settings.AccessTokenExpirationMinutes),
            Issuer = _settings.Issuer,
            Audience = _settings.Audience,
            SigningCredentials = new SigningCredentials(
                new SymmetricSecurityKey(key),
                SecurityAlgorithms.HmacSha256Signature)
        };
        
        var token = tokenHandler.CreateToken(tokenDescriptor);
        return tokenHandler.WriteToken(token);
    }
    
    // สร้าง Refresh Token และบันทึกใน DB
    public async Task<RefreshToken> GenerateRefreshTokenAsync(User user)
    {
        var refreshToken = new RefreshToken
        {
            Token = GenerateSecureToken(),
            UserId = user.Id,
            ExpiresAt = DateTime.UtcNow.AddDays(_settings.RefreshTokenExpirationDays),
            CreatedByIp = GetClientIpAddress()
        };
        
        _context.RefreshTokens.Add(refreshToken);
        await _context.SaveChangesAsync();
        
        // Cleanup old tokens (เก็บแค่ 5 tokens ล่าสุด)
        await CleanupOldTokensAsync(user.Id);
        
        return refreshToken;
    }
    
    // Validate Access Token
    public ClaimsPrincipal? ValidateAccessToken(string token, bool validateLifetime = true)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(_settings.SecretKey);
        
        try
        {
            var principal = tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = true,
                ValidIssuer = _settings.Issuer,
                ValidateAudience = true,
                ValidAudience = _settings.Audience,
                ValidateLifetime = validateLifetime,
                ClockSkew = TimeSpan.Zero
            }, out var validatedToken);
            
            // ตรวจสอบ algorithm
            if (validatedToken is not JwtSecurityToken jwtToken ||
                !jwtToken.Header.Alg.Equals(
                    SecurityAlgorithms.HmacSha256Signature,
                    StringComparison.OrdinalIgnoreCase))
                return null;
            
            return principal;
        }
        catch (SecurityTokenExpiredException)
        {
            // Token หมดอายุ - return principal เพื่อดู claims
            if (!validateLifetime)
            {
                try
                {
                    return tokenHandler.ValidateToken(token, new TokenValidationParameters
                    {
                        ValidateIssuerSigningKey = true,
                        IssuerSigningKey = new SymmetricSecurityKey(key),
                        ValidateIssuer = true,
                        ValidIssuer = _settings.Issuer,
                        ValidateAudience = true,
                        ValidAudience = _settings.Audience,
                        ValidateLifetime = false
                    }, out _);
                }
                catch { return null; }
            }
            return null;
        }
        catch
        {
            return null;
        }
    }
    
    // Refresh Access Token
    public async Task<(bool Success, string? Error, string? NewAccessToken, string? NewRefreshToken)>
        RefreshTokensAsync(string accessToken, string refreshToken)
    {
        // Validate access token (without lifetime check)
        var principal = ValidateAccessToken(accessToken, validateLifetime: false);
        if (principal == null)
            return (false, "Invalid access token", null, null);
        
        // Get user from claims
        var userIdClaim = principal.FindFirst(JwtRegisteredClaimNames.Sub)?.Value;
        if (!Guid.TryParse(userIdClaim, out var userId))
            return (false, "Invalid token claims", null, null);
        
        // Find refresh token in DB
        var storedToken = await _context.RefreshTokens
            .Include(rt => rt.User)
                .ThenInclude(u => u.UserRoles)
                    .ThenInclude(ur => ur.Role)
            .FirstOrDefaultAsync(rt => rt.Token == refreshToken);
        
        if (storedToken == null)
            return (false, "Refresh token not found", null, null);
        
        if (storedToken.UserId != userId)
            return (false, "Token mismatch", null, null);
        
        if (!storedToken.IsActive)
        {
            // Token ถูก revoke หรือหมดอายุ
            if (storedToken.IsRevoked)
            {
                // Possible token reuse attack - revoke all tokens for this user
                await RevokeAllUserTokensAsync(userId, "Suspicious activity");
                return (false, "Refresh token has been revoked", null, null);
            }
            return (false, "Refresh token has expired", null, null);
        }
        
        var user = storedToken.User;
        var roles = user.UserRoles.Select(ur => ur.Role.Name).ToList();
        
        // สร้าง tokens ใหม่
        var newAccessToken = GenerateAccessToken(user, roles);
        var newRefreshTokenModel = await GenerateRefreshTokenAsync(user);
        
        // Revoke old refresh token
        storedToken.IsRevoked = true;
        storedToken.RevokedAt = DateTime.UtcNow;
        storedToken.RevokedReason = "Replaced by new token";
        storedToken.ReplacedByToken = newRefreshTokenModel.Token;
        
        await _context.SaveChangesAsync();
        
        return (true, null, newAccessToken, newRefreshTokenModel.Token);
    }
    
    // Revoke Refresh Token
    public async Task<bool> RevokeRefreshTokenAsync(
        string token, 
        string reason = "User logout")
    {
        var storedToken = await _context.RefreshTokens
            .FirstOrDefaultAsync(rt => rt.Token == token);
        
        if (storedToken == null || !storedToken.IsActive)
            return false;
        
        storedToken.IsRevoked = true;
        storedToken.RevokedAt = DateTime.UtcNow;
        storedToken.RevokedReason = reason;
        
        await _context.SaveChangesAsync();
        return true;
    }
    
    // Revoke all tokens for a user
    public async Task RevokeAllUserTokensAsync(Guid userId, string reason)
    {
        var activeTokens = await _context.RefreshTokens
            .Where(rt => rt.UserId == userId && !rt.IsRevoked)
            .ToListAsync();
        
        foreach (var token in activeTokens)
        {
            token.IsRevoked = true;
            token.RevokedAt = DateTime.UtcNow;
            token.RevokedReason = reason;
        }
        
        await _context.SaveChangesAsync();
    }
    
    // Helper methods
    private static string GenerateSecureToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }
    
    private string? GetClientIpAddress()
    {
        return _httpContextAccessor.HttpContext?.Connection.RemoteIpAddress?.ToString();
    }
    
    private async Task CleanupOldTokensAsync(Guid userId)
    {
        var oldTokens = await _context.RefreshTokens
            .Where(rt => rt.UserId == userId && 
                   (rt.IsRevoked || rt.ExpiresAt < DateTime.UtcNow))
            .OrderBy(rt => rt.CreatedAt)
            .ToListAsync();
        
        if (oldTokens.Count > 5)
        {
            var toRemove = oldTokens.Take(oldTokens.Count - 5).ToList();
            _context.RefreshTokens.RemoveRange(toRemove);
            await _context.SaveChangesAsync();
        }
    }
}
```

---

## Auth Service

```csharp
// Services/AuthService.cs
using Microsoft.EntityFrameworkCore;
using JwtApi.Data;
using JwtApi.DTOs;
using JwtApi.Models;
using BC = BCrypt.Net.BCrypt;  // dotnet add package BCrypt.Net-Next

namespace JwtApi.Services;

public class AuthService
{
    private readonly JwtDbContext _context;
    private readonly JwtService _jwtService;
    private readonly ILogger<AuthService> _logger;
    
    public AuthService(
        JwtDbContext context,
        JwtService jwtService,
        ILogger<AuthService> logger)
    {
        _context = context;
        _jwtService = jwtService;
        _logger = logger;
    }
    
    public async Task<AuthResult> RegisterAsync(RegisterRequest request)
    {
        // Validate
        if (request.Password != request.ConfirmPassword)
            return AuthResult.Fail("Passwords do not match");
        
        // Check duplicates
        if (await _context.Users.AnyAsync(u => u.Email == request.Email))
            return AuthResult.Fail("Email already registered");
        
        if (await _context.Users.AnyAsync(u => u.Username == request.Username))
            return AuthResult.Fail("Username already taken");
        
        // Create user
        var user = new User
        {
            Email = request.Email.ToLower().Trim(),
            Username = request.Username.Trim(),
            FirstName = request.FirstName,
            LastName = request.LastName,
            PasswordHash = BC.HashPassword(request.Password)
        };
        
        _context.Users.Add(user);
        
        // Assign default role
        var userRole = await _context.Roles.FirstAsync(r => r.Name == "User");
        user.UserRoles.Add(new UserRole { UserId = user.Id, RoleId = userRole.Id });
        
        await _context.SaveChangesAsync();
        
        // Generate tokens
        var roles = new[] { userRole.Name };
        var accessToken = _jwtService.GenerateAccessToken(user, roles);
        var refreshToken = await _jwtService.GenerateRefreshTokenAsync(user);
        
        _logger.LogInformation("User registered: {Email}", user.Email);
        
        return AuthResult.Ok(accessToken, refreshToken.Token, user, roles);
    }
    
    public async Task<AuthResult> LoginAsync(LoginRequest request)
    {
        var user = await _context.Users
            .Include(u => u.UserRoles)
                .ThenInclude(ur => ur.Role)
            .FirstOrDefaultAsync(u => u.Email == request.EmailOrUsername.ToLower() ||
                                      u.Username == request.EmailOrUsername);
        
        if (user == null)
            return AuthResult.Fail("Invalid credentials");
        
        if (!user.IsActive)
            return AuthResult.Fail("Account is disabled");
        
        if (!BC.Verify(request.Password, user.PasswordHash))
            return AuthResult.Fail("Invalid credentials");
        
        // Update last login
        user.LastLoginAt = DateTime.UtcNow;
        await _context.SaveChangesAsync();
        
        var roles = user.UserRoles.Select(ur => ur.Role.Name).ToArray();
        var accessToken = _jwtService.GenerateAccessToken(user, roles);
        var refreshToken = await _jwtService.GenerateRefreshTokenAsync(user);
        
        _logger.LogInformation("User logged in: {Email}", user.Email);
        
        return AuthResult.Ok(accessToken, refreshToken.Token, user, roles);
    }
}

// Result types
public class AuthResult
{
    public bool Success { get; set; }
    public string? Error { get; set; }
    public string? AccessToken { get; set; }
    public string? RefreshToken { get; set; }
    public UserDto? User { get; set; }
    
    public static AuthResult Fail(string error) => 
        new() { Success = false, Error = error };
    
    public static AuthResult Ok(
        string accessToken, 
        string refreshToken, 
        User user, 
        IEnumerable<string> roles) => new()
    {
        Success = true,
        AccessToken = accessToken,
        RefreshToken = refreshToken,
        User = new UserDto(
            user.Id.ToString(),
            user.Email,
            user.Username,
            user.FullName,
            roles.ToList())
    };
}
```

---

## DTOs

```csharp
// DTOs/AuthDtos.cs
using System.ComponentModel.DataAnnotations;

namespace JwtApi.DTOs;

public record RegisterRequest(
    [Required][MaxLength(100)] string FirstName,
    [Required][MaxLength(100)] string LastName,
    [Required][EmailAddress] string Email,
    [Required][MaxLength(50)] string Username,
    [Required][MinLength(8)] string Password,
    [Required] string ConfirmPassword);

public record LoginRequest(
    [Required] string EmailOrUsername,
    [Required] string Password);

public record RefreshRequest(
    [Required] string AccessToken,
    [Required] string RefreshToken);

public record RevokeRequest([Required] string RefreshToken);

public record UserDto(
    string Id,
    string Email,
    string Username,
    string FullName,
    List<string> Roles);
```

---

## Auth Controller

```csharp
// Controllers/AuthController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using JwtApi.DTOs;
using JwtApi.Services;

namespace JwtApi.Controllers;

[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class AuthController : ControllerBase
{
    private readonly AuthService _authService;
    private readonly JwtService _jwtService;
    
    public AuthController(AuthService authService, JwtService jwtService)
    {
        _authService = authService;
        _jwtService = jwtService;
    }
    
    /// <summary>ลงทะเบียนผู้ใช้ใหม่</summary>
    [HttpPost("register")]
    [ProducesResponseType(typeof(object), 200)]
    [ProducesResponseType(400)]
    public async Task<IActionResult> Register([FromBody] RegisterRequest request)
    {
        var result = await _authService.RegisterAsync(request);
        
        if (!result.Success)
            return BadRequest(new { Message = result.Error });
        
        return Ok(new
        {
            AccessToken = result.AccessToken,
            RefreshToken = result.RefreshToken,
            ExpiresIn = 900,  // 15 minutes in seconds
            TokenType = "Bearer",
            User = result.User
        });
    }
    
    /// <summary>เข้าสู่ระบบ</summary>
    [HttpPost("login")]
    [ProducesResponseType(200)]
    [ProducesResponseType(401)]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        var result = await _authService.LoginAsync(request);
        
        if (!result.Success)
            return Unauthorized(new { Message = result.Error });
        
        return Ok(new
        {
            AccessToken = result.AccessToken,
            RefreshToken = result.RefreshToken,
            ExpiresIn = 900,
            TokenType = "Bearer",
            User = result.User
        });
    }
    
    /// <summary>ต่ออายุ Token</summary>
    [HttpPost("refresh")]
    [ProducesResponseType(200)]
    [ProducesResponseType(401)]
    public async Task<IActionResult> Refresh([FromBody] RefreshRequest request)
    {
        var (success, error, newAccess, newRefresh) = 
            await _jwtService.RefreshTokensAsync(
                request.AccessToken, 
                request.RefreshToken);
        
        if (!success)
            return Unauthorized(new { Message = error });
        
        return Ok(new
        {
            AccessToken = newAccess,
            RefreshToken = newRefresh,
            ExpiresIn = 900,
            TokenType = "Bearer"
        });
    }
    
    /// <summary>ออกจากระบบ (Revoke refresh token)</summary>
    [HttpPost("logout")]
    [Authorize]
    [ProducesResponseType(200)]
    public async Task<IActionResult> Logout([FromBody] RevokeRequest request)
    {
        await _jwtService.RevokeRefreshTokenAsync(request.RefreshToken);
        return Ok(new { Message = "Logged out successfully" });
    }
    
    /// <summary>ออกจากระบบทุก device</summary>
    [HttpPost("logout-all")]
    [Authorize]
    public async Task<IActionResult> LogoutAll()
    {
        var userId = User.FindFirstValue(JwtRegisteredClaimNames.Sub);
        if (!Guid.TryParse(userId, out var userGuid))
            return Unauthorized();
        
        await _jwtService.RevokeAllUserTokensAsync(userGuid, "User logged out all devices");
        return Ok(new { Message = "Logged out from all devices" });
    }
    
    /// <summary>ดูข้อมูล user ปัจจุบัน</summary>
    [HttpGet("me")]
    [Authorize]
    public IActionResult GetMe()
    {
        var claims = User.Claims
            .GroupBy(c => c.Type)
            .ToDictionary(g => g.Key, g => g.Count() == 1 ? g.First().Value : (object)g.Select(c => c.Value).ToList());
        
        return Ok(new
        {
            UserId = User.FindFirstValue(System.IdentityModel.Tokens.Jwt.JwtRegisteredClaimNames.Sub),
            Email = User.FindFirstValue(ClaimTypes.Email),
            Username = User.FindFirstValue(ClaimTypes.Name),
            FirstName = User.FindFirstValue("firstName"),
            LastName = User.FindFirstValue("lastName"),
            Roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList(),
            Claims = claims
        });
    }
}
```

---

## Protected Controller ตัวอย่าง

```csharp
// Controllers/SecureController.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;

namespace JwtApi.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]  // ต้อง authenticate ทุก action
public class SecureController : ControllerBase
{
    [HttpGet("public")]
    [AllowAnonymous]  // ยกเว้น action นี้
    public IActionResult PublicEndpoint()
    {
        return Ok(new { Message = "Anyone can access this" });
    }
    
    [HttpGet("protected")]
    public IActionResult ProtectedEndpoint()
    {
        var userId = User.FindFirstValue(
            System.IdentityModel.Tokens.Jwt.JwtRegisteredClaimNames.Sub);
        return Ok(new { Message = $"Hello user {userId}!" });
    }
    
    [HttpGet("admin")]
    [Authorize(Roles = "Admin")]  // เฉพาะ Admin
    public IActionResult AdminEndpoint()
    {
        return Ok(new { Message = "Admin access granted" });
    }
    
    [HttpGet("manager-or-admin")]
    [Authorize(Roles = "Admin,Manager")]  // Admin หรือ Manager
    public IActionResult ManagerEndpoint()
    {
        return Ok(new { Message = "Manager/Admin access granted" });
    }
    
    [HttpGet("user-info")]
    public IActionResult GetUserInfo()
    {
        // Access claims จาก JWT
        var userInfo = new
        {
            Id = User.FindFirstValue(
                System.IdentityModel.Tokens.Jwt.JwtRegisteredClaimNames.Sub),
            Email = User.FindFirstValue(ClaimTypes.Email),
            Roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value),
            IsAdmin = User.IsInRole("Admin"),
            IsManager = User.IsInRole("Manager"),
            TokenIssued = User.FindFirstValue(
                System.IdentityModel.Tokens.Jwt.JwtRegisteredClaimNames.Iat)
        };
        
        return Ok(userInfo);
    }
}
```

---

## Program.cs สมบูรณ์

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using Microsoft.OpenApi.Models;
using System.Text;
using JwtApi.Data;
using JwtApi.Services;
using JwtApi.Settings;

var builder = WebApplication.CreateBuilder(args);

// Settings
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));

// Database
builder.Services.AddDbContext<JwtDbContext>(options =>
    options.UseSqlite(
        builder.Configuration.GetConnectionString("Default") 
        ?? "Data Source=jwt_api.db"));

// JWT Authentication
var jwtSettings = builder.Configuration
    .GetSection(JwtSettings.SectionName)
    .Get<JwtSettings>()!;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings.SecretKey)),
            ValidateIssuer = true,
            ValidIssuer = jwtSettings.Issuer,
            ValidateAudience = true,
            ValidAudience = jwtSettings.Audience,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
        
        // Events สำหรับ custom handling
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                if (context.Exception is SecurityTokenExpiredException)
                    context.Response.Headers.Append("Token-Expired", "true");
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();
builder.Services.AddHttpContextAccessor();

// Services
builder.Services.AddScoped<JwtService>();
builder.Services.AddScoped<AuthService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

// Swagger with JWT support
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "JWT API", Version = "v1" });
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme.\r\n" +
                      "Enter 'Bearer' [space] and then your token.\r\n" +
                      "Example: 'Bearer eyJhbGci...'",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
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

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<JwtDbContext>();
    await db.Database.MigrateAsync();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## ทดสอบด้วย Swagger หรือ curl

```bash
# Register
curl -X POST https://localhost:5001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "สมชาย",
    "lastName": "ใจดี",
    "email": "somchai@test.com",
    "username": "somchai",
    "password": "Password1!",
    "confirmPassword": "Password1!"
  }'

# Login
curl -X POST https://localhost:5001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"emailOrUsername": "somchai", "password": "Password1!"}'

# Access protected endpoint
TOKEN="eyJhbGci..."
curl https://localhost:5001/api/auth/me \
  -H "Authorization: Bearer $TOKEN"

# Refresh token
curl -X POST https://localhost:5001/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"accessToken": "...", "refreshToken": "..."}'
```

---

## Exercises

### แบบฝึกหัดที่ 1: Token Blacklist
สร้าง token blacklist mechanism:
1. เมื่อ logout ให้เพิ่ม JWT ID (jti claim) ใน blacklist
2. Middleware ตรวจสอบ blacklist ทุก request

### แบบฝึกหัดที่ 2: Multiple Audiences
แก้ไข JWT service ให้รองรับ multiple audiences:
- Web frontend
- Mobile app
- Admin panel

### แบบฝึกหัดที่ 3: Rate Limiting
เพิ่ม rate limiting สำหรับ auth endpoints:
- Max 5 login attempts per minute per IP
- Max 3 register attempts per hour per IP

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **JWT Structure**: Header + Payload + Signature
2. **Access Token**: Short-lived token สำหรับ authentication
3. **Refresh Token**: Long-lived token สำหรับต่ออายุ
4. **Token Validation**: ตรวจสอบ signature, expiry, issuer, audience
5. **Refresh Strategy**: Rotate refresh tokens เมื่อใช้
6. **Security**: Revoke tokens เมื่อ suspicious activity

---

## Part ถัดไป

ใน **Part 060** เราจะเรียนรู้ **Authorization และ Roles** อย่างละเอียด:
- [Authorize] attribute ต่างๆ
- Role-based authorization
- Policy-based authorization
- Resource-based authorization
- Custom authorization middleware

---

*Part 059/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
