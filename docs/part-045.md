# Part 045: Configuration และ appsettings

## เนื้อหาใน Part นี้
- appsettings.json structure
- IConfiguration
- Environment-specific config
- User Secrets
- Environment Variables
- Options pattern (IOptions<T>)
- โปรแกรมตัวอย่าง: Database config

---

## 1. Configuration System ใน ASP.NET Core

ASP.NET Core มีระบบ configuration ที่ยืดหยุ่นมาก รองรับ sources หลายชนิด:

```
Configuration Sources (ลำดับ priority จากต่ำสูง):
1. appsettings.json
2. appsettings.{Environment}.json
3. User Secrets (Development only)
4. Environment Variables
5. Command Line Arguments
```

ค่าที่อยู่ใน source ที่มี priority สูงกว่าจะ **override** ค่าในระดับต่ำกว่า

```csharp
// Program.cs - Configuration sources ถูกตั้งค่าอัตโนมัติโดย CreateBuilder
var builder = WebApplication.CreateBuilder(args);

// builder.Configuration มีข้อมูลจากทุก source
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var apiKey = builder.Configuration["AppSettings:ApiKey"];
```

---

## 2. appsettings.json

### 2.1 โครงสร้างพื้นฐาน

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyAppDb;User=sa;Password=P@ssword123;",
    "RedisConnection": "localhost:6379,password=redis123"
  },
  "AppSettings": {
    "ApplicationName": "My Application",
    "Version": "1.0.0",
    "MaxUploadSizeMb": 10,
    "EnableFeatureFlags": true,
    "SupportedLanguages": ["th", "en", "ja"],
    "AdminEmails": ["admin@example.com", "support@example.com"]
  },
  "Jwt": {
    "Issuer": "https://myapp.com",
    "Audience": "https://myapp.com",
    "SecretKey": "REPLACE-WITH-STRONG-SECRET-KEY-IN-PRODUCTION",
    "ExpiryMinutes": 60,
    "RefreshTokenExpiryDays": 7
  },
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "UseSsl": true,
    "FromEmail": "noreply@example.com",
    "FromName": "My App"
  }
}
```

### 2.2 appsettings.Development.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "System": "Information",
      "Microsoft": "Information",
      "Microsoft.AspNetCore": "Debug"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyAppDb_Dev;Trusted_Connection=True;",
    "RedisConnection": "localhost:6379"
  },
  "AppSettings": {
    "EnableFeatureFlags": true
  }
}
```

### 2.3 appsettings.Production.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Error"
    }
  },
  "AppSettings": {
    "EnableFeatureFlags": false
  }
}
```

---

## 3. IConfiguration

### 3.1 อ่าน Configuration

```csharp
var builder = WebApplication.CreateBuilder(args);
var config = builder.Configuration;

// อ่านค่าโดยตรง
var appName = config["AppSettings:ApplicationName"];
var maxSize = config["AppSettings:MaxUploadSizeMb"];

// อ่านค่าพร้อม default
var port = config.GetValue<int>("Port", defaultValue: 5000);
var timeout = config.GetValue<TimeSpan>("Timeout", TimeSpan.FromSeconds(30));
var debugMode = config.GetValue<bool>("Debug", false);

// อ่าน connection string (shortcut method)
var connStr = config.GetConnectionString("DefaultConnection");

// อ่าน section
var jwtSection = config.GetSection("Jwt");
var jwtIssuer = jwtSection["Issuer"];
var jwtAudience = jwtSection.GetValue<string>("Audience");

// อ่าน array
var languages = config.GetSection("AppSettings:SupportedLanguages").Get<string[]>();
var adminEmails = config.GetSection("AppSettings:AdminEmails").Get<List<string>>();

// อ่าน object
public class JwtSettings
{
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public string SecretKey { get; set; } = string.Empty;
    public int ExpiryMinutes { get; set; } = 60;
}

var jwtSettings = config.GetSection("Jwt").Get<JwtSettings>();
```

### 3.2 IConfiguration ใน Service

```csharp
// Inject IConfiguration ใน service
public class EmailService : IEmailService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<EmailService> _logger;

    public EmailService(IConfiguration configuration, ILogger<EmailService> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        var host = _configuration["Email:SmtpHost"]!;
        var port = _configuration.GetValue<int>("Email:SmtpPort");
        var useSsl = _configuration.GetValue<bool>("Email:UseSsl");
        var fromEmail = _configuration["Email:FromEmail"]!;
        
        _logger.LogInformation("Sending email to {To} via {Host}:{Port}", to, host, port);
        
        // ส่ง email...
    }
}
```

### 3.3 Hot Reload Configuration

```csharp
// IOptionsMonitor สำหรับ hot reload (ไม่ต้อง restart app)
public class DynamicSettingsService
{
    private readonly IOptionsMonitor<AppSettings> _optionsMonitor;
    
    public DynamicSettingsService(IOptionsMonitor<AppSettings> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
        
        // Register callback เมื่อ config เปลี่ยน
        _optionsMonitor.OnChange(settings =>
        {
            Console.WriteLine($"Settings changed: {settings.ApplicationName}");
        });
    }
    
    public string GetCurrentAppName() => _optionsMonitor.CurrentValue.ApplicationName;
}
```

---

## 4. Environment-specific Configuration

### 4.1 Environment ที่รองรับ

```
Development  - สำหรับ local development
Staging      - สำหรับ staging/UAT
Production   - สำหรับ production
```

### 4.2 ตั้งค่า Environment

```bash
# Linux/macOS
export ASPNETCORE_ENVIRONMENT=Production

# Windows Command Prompt
set ASPNETCORE_ENVIRONMENT=Production

# Windows PowerShell
$env:ASPNETCORE_ENVIRONMENT = "Production"

# ใน launchSettings.json (development เท่านั้น)
{
  "profiles": {
    "MyApp": {
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

### 4.3 ลำดับการ Load Configuration

```csharp
// ลำดับที่ ASP.NET Core load configuration:
var builder = WebApplication.CreateBuilder(args);

// เทียบเท่ากับการทำ:
// builder.Configuration
//     .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
//     .AddJsonFile($"appsettings.{env}.json", optional: true, reloadOnChange: true)
//     .AddUserSecrets<Program>(optional: true)  // Development only
//     .AddEnvironmentVariables()
//     .AddCommandLine(args);
```

### 4.4 Custom Environment

```csharp
// สร้าง custom environment "QA"
// appsettings.QA.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=qa-server;Database=MyAppDb_QA;..."
  },
  "AppSettings": {
    "EnableFeatureFlags": true
  }
}

// ตรวจสอบ
if (app.Environment.IsEnvironment("QA"))
{
    app.UseSwagger();
    // เพิ่ม diagnostics endpoints
}
```

---

## 5. User Secrets

User Secrets ใช้เก็บข้อมูลลับสำหรับ development โดยไม่ต้องเก็บไว้ใน source code

### 5.1 เริ่มต้นใช้ User Secrets

```bash
# Initialize user secrets
dotnet user-secrets init

# เพิ่ม secret
dotnet user-secrets set "Jwt:SecretKey" "my-super-secret-key-for-development"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;..."
dotnet user-secrets set "Email:SmtpPassword" "gmail-app-password"

# แสดง secrets ทั้งหมด
dotnet user-secrets list

# ลบ secret
dotnet user-secrets remove "Email:SmtpPassword"

# ล้าง secrets ทั้งหมด
dotnet user-secrets clear
```

### 5.2 ตำแหน่งที่เก็บ User Secrets

```
Windows:  %APPDATA%\Microsoft\UserSecrets\{user-secret-id}\secrets.json
Linux:    ~/.microsoft/usersecrets/{user-secret-id}/secrets.json
macOS:    ~/.microsoft/usersecrets/{user-secret-id}/secrets.json
```

### 5.3 ไฟล์ .csproj หลัง init

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <UserSecretsId>a1b2c3d4-e5f6-7890-abcd-ef1234567890</UserSecretsId>
  </PropertyGroup>
</Project>
```

### 5.4 secrets.json (อย่าเพิ่มใน .gitignore เพราะอยู่นอก project)

```json
{
  "Jwt": {
    "SecretKey": "my-super-secret-key-for-development-only"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyAppDb_Dev;Trusted_Connection=True;"
  },
  "Email": {
    "SmtpPassword": "development-app-password"
  }
}
```

---

## 6. Environment Variables

### 6.1 การใช้ Environment Variables

```bash
# ตั้งค่า environment variables
# ใช้ __ (double underscore) แทน : สำหรับ nested keys

# Linux/macOS
export ConnectionStrings__DefaultConnection="Server=prod-server;..."
export Jwt__SecretKey="production-secret-key"
export AppSettings__MaxUploadSizeMb=50

# Windows
set ConnectionStrings__DefaultConnection=Server=prod-server;...
set Jwt__SecretKey=production-secret-key

# Docker
docker run -e "ConnectionStrings__DefaultConnection=Server=db;..." myapp
```

### 6.2 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    image: myapp:latest
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyApp;User=sa;Password=${DB_PASSWORD}
      - Jwt__SecretKey=${JWT_SECRET}
      - Email__SmtpPassword=${SMTP_PASSWORD}
    ports:
      - "8080:80"
```

### 6.3 Prefix Filter

```csharp
// อ่าน environment variables เฉพาะที่ขึ้นต้นด้วย prefix
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");

// MYAPP_Jwt__SecretKey จะถูก map เป็น Jwt:SecretKey
// environment variable:  MYAPP_ConnectionStrings__DefaultConnection
// config key:           ConnectionStrings:DefaultConnection
```

---

## 7. Options Pattern (IOptions<T>)

Options pattern เป็นวิธีที่แนะนำสำหรับ bind configuration กับ strongly-typed class

### 7.1 IOptions<T> - ค่าคงที่

```csharp
// 1. สร้าง Options class
public class DatabaseOptions
{
    public const string SectionName = "Database";
    
    [Required]
    public string ConnectionString { get; set; } = string.Empty;
    
    [Range(1, 1000)]
    public int MaxPoolSize { get; set; } = 100;
    
    [Range(1, 3600)]
    public int CommandTimeoutSeconds { get; set; } = 30;
    
    public bool EnableRetry { get; set; } = true;
    public int MaxRetryCount { get; set; } = 3;
}

// 2. ลงทะเบียน
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));

// 3. ใช้งาน
public class DatabaseService
{
    private readonly DatabaseOptions _options;
    
    public DatabaseService(IOptions<DatabaseOptions> options)
    {
        _options = options.Value;
    }
    
    public string GetConnectionString() => _options.ConnectionString;
}
```

### 7.2 IOptionsSnapshot<T> - Reload per request

```csharp
// IOptionsSnapshot - reload ค่าใหม่ต่อ request (ถ้าไฟล์เปลี่ยน)
public class FeatureFlagService
{
    private readonly FeatureFlagOptions _options;
    
    public FeatureFlagService(IOptionsSnapshot<FeatureFlagOptions> options)
    {
        _options = options.Value;
    }
}
```

### 7.3 IOptionsMonitor<T> - Hot reload + Singleton

```csharp
// IOptionsMonitor - hot reload สำหรับ Singleton services
public class ConfigurationMonitorService : IHostedService
{
    private readonly IOptionsMonitor<AppSettings> _monitor;
    private IDisposable? _changeToken;
    
    public ConfigurationMonitorService(IOptionsMonitor<AppSettings> monitor)
    {
        _monitor = monitor;
    }
    
    public Task StartAsync(CancellationToken cancellationToken)
    {
        _changeToken = _monitor.OnChange(settings =>
        {
            Console.WriteLine($"Configuration changed! New version: {settings.Version}");
        });
        return Task.CompletedTask;
    }
    
    public Task StopAsync(CancellationToken cancellationToken)
    {
        _changeToken?.Dispose();
        return Task.CompletedTask;
    }
}
```

### 7.4 เปรียบเทียบ IOptions variants

| Interface | คำอธิบาย | Scoped/Singleton | Hot reload |
|-----------|---------|----------------|-----------|
| `IOptions<T>` | ค่าคงที่ตลอด | ทั้งคู่ | ❌ |
| `IOptionsSnapshot<T>` | ค่าใหม่ต่อ request | Scoped เท่านั้น | ✅ |
| `IOptionsMonitor<T>` | notify เมื่อเปลี่ยน | ทั้งคู่ | ✅ |

### 7.5 Options Validation

```csharp
// Data Annotations validation
public class SmtpOptions
{
    [Required]
    public string Host { get; set; } = string.Empty;
    
    [Range(1, 65535)]
    public int Port { get; set; } = 587;
    
    [Required, EmailAddress]
    public string FromEmail { get; set; } = string.Empty;
}

// ลงทะเบียนพร้อม validation
builder.Services
    .AddOptions<SmtpOptions>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations()
    .ValidateOnStart();  // Validate เมื่อ app start

// Custom validation
builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration("Jwt")
    .Validate(options =>
    {
        if (options.SecretKey.Length < 32)
            return false;  // Validation failed
        return true;
    }, "JWT Secret key must be at least 32 characters");
```

---

## 8. โปรแกรมตัวอย่าง: Database Config

```csharp
// Program.cs
using System.ComponentModel.DataAnnotations;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ลงทะเบียน Options พร้อม validation
builder.Services
    .AddOptions<DatabaseOptions>()
    .BindConfiguration(DatabaseOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services
    .AddOptions<JwtOptions>()
    .BindConfiguration(JwtOptions.SectionName)
    .Validate(opt => opt.SecretKey.Length >= 32, 
        "JWT SecretKey ต้องมีความยาวอย่างน้อย 32 ตัวอักษร")
    .ValidateOnStart();

builder.Services
    .AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SectionName)
    .ValidateDataAnnotations()
    .ValidateOnStart();

// ลงทะเบียน services
builder.Services.AddScoped<IDatabaseConnectionFactory, DatabaseConnectionFactory>();
builder.Services.AddScoped<IConfigurationDemoService, ConfigurationDemoService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// Endpoints
app.MapGet("/config/database", (IOptions<DatabaseOptions> options) =>
{
    var opts = options.Value;
    return Results.Ok(new
    {
        ConnectionString = MaskConnectionString(opts.ConnectionString),
        opts.MaxPoolSize,
        opts.CommandTimeoutSeconds,
        opts.EnableRetry,
        opts.MaxRetryCount
    });
})
.WithTags("Configuration")
.WithSummary("แสดง database configuration");

app.MapGet("/config/jwt", (IOptions<JwtOptions> options) =>
{
    var opts = options.Value;
    return Results.Ok(new
    {
        opts.Issuer,
        opts.Audience,
        SecretKey = "***REDACTED***",
        opts.ExpiryMinutes,
        opts.RefreshTokenExpiryDays
    });
})
.WithTags("Configuration")
.WithSummary("แสดง JWT configuration");

app.MapGet("/config/all", (
    IOptions<DatabaseOptions> dbOptions,
    IOptions<JwtOptions> jwtOptions,
    IOptions<EmailOptions> emailOptions,
    IWebHostEnvironment env) =>
{
    return Results.Ok(new
    {
        Environment = env.EnvironmentName,
        Database = new
        {
            ConnectionString = MaskConnectionString(dbOptions.Value.ConnectionString),
            dbOptions.Value.MaxPoolSize
        },
        Jwt = new
        {
            jwtOptions.Value.Issuer,
            jwtOptions.Value.Audience,
            SecretKey = "***REDACTED***"
        },
        Email = new
        {
            emailOptions.Value.Host,
            emailOptions.Value.Port,
            emailOptions.Value.FromEmail
        }
    });
})
.WithTags("Configuration")
.WithSummary("แสดงการตั้งค่าทั้งหมด");

// Test connection
app.MapGet("/config/test-connection", async (IDatabaseConnectionFactory factory) =>
{
    var result = await factory.TestConnectionAsync();
    return result.IsSuccess
        ? Results.Ok(new { Message = "เชื่อมต่อฐานข้อมูลสำเร็จ", result.ConnectionString })
        : Results.Problem(result.ErrorMessage, statusCode: 503);
})
.WithTags("Configuration")
.WithSummary("ทดสอบการเชื่อมต่อฐานข้อมูล");

app.Run();

// Helper function
static string MaskConnectionString(string connectionString)
{
    // ซ่อน password ใน connection string
    return System.Text.RegularExpressions.Regex.Replace(
        connectionString, 
        @"(Password|pwd)=([^;]*)", 
        "$1=***", 
        System.Text.RegularExpressions.RegexOptions.IgnoreCase);
}

// ============================================
// OPTIONS CLASSES
// ============================================

public class DatabaseOptions
{
    public const string SectionName = "Database";

    [Required(ErrorMessage = "ConnectionString is required")]
    public string ConnectionString { get; set; } = string.Empty;

    [Range(1, 1000, ErrorMessage = "MaxPoolSize ต้องอยู่ระหว่าง 1-1000")]
    public int MaxPoolSize { get; set; } = 100;

    [Range(1, 3600, ErrorMessage = "CommandTimeout ต้องอยู่ระหว่าง 1-3600 วินาที")]
    public int CommandTimeoutSeconds { get; set; } = 30;

    public bool EnableRetry { get; set; } = true;

    [Range(0, 10, ErrorMessage = "MaxRetryCount ต้องอยู่ระหว่าง 0-10")]
    public int MaxRetryCount { get; set; } = 3;

    public TimeSpan RetryDelay { get; set; } = TimeSpan.FromSeconds(1);
}

public class JwtOptions
{
    public const string SectionName = "Jwt";

    [Required]
    public string Issuer { get; set; } = string.Empty;

    [Required]
    public string Audience { get; set; } = string.Empty;

    [Required, MinLength(32)]
    public string SecretKey { get; set; } = string.Empty;

    [Range(1, 1440)]
    public int ExpiryMinutes { get; set; } = 60;

    [Range(1, 365)]
    public int RefreshTokenExpiryDays { get; set; } = 7;
}

public class EmailOptions
{
    public const string SectionName = "Email";

    [Required]
    public string Host { get; set; } = string.Empty;

    [Range(1, 65535)]
    public int Port { get; set; } = 587;

    public bool UseSsl { get; set; } = true;

    [Required, EmailAddress]
    public string FromEmail { get; set; } = string.Empty;

    [Required]
    public string FromName { get; set; } = string.Empty;

    public string? SmtpUsername { get; set; }
    public string? SmtpPassword { get; set; }
}

// ============================================
// SERVICES
// ============================================

public interface IDatabaseConnectionFactory
{
    Task<ConnectionTestResult> TestConnectionAsync();
}

public class DatabaseConnectionFactory : IDatabaseConnectionFactory
{
    private readonly DatabaseOptions _options;
    private readonly ILogger<DatabaseConnectionFactory> _logger;

    public DatabaseConnectionFactory(
        IOptions<DatabaseOptions> options,
        ILogger<DatabaseConnectionFactory> logger)
    {
        _options = options.Value;
        _logger = logger;
    }

    public async Task<ConnectionTestResult> TestConnectionAsync()
    {
        try
        {
            _logger.LogInformation("ทดสอบการเชื่อมต่อฐานข้อมูล...");
            
            // Simulate connection test
            await Task.Delay(100);
            
            // ใน production จะ test connection จริง:
            // using var connection = new SqlConnection(_options.ConnectionString);
            // await connection.OpenAsync();
            
            var maskedCs = MaskConnectionString(_options.ConnectionString);
            _logger.LogInformation("เชื่อมต่อสำเร็จ: {ConnectionString}", maskedCs);
            
            return ConnectionTestResult.Success(maskedCs);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ไม่สามารถเชื่อมต่อฐานข้อมูลได้");
            return ConnectionTestResult.Failure(ex.Message);
        }
    }

    private static string MaskConnectionString(string cs) =>
        System.Text.RegularExpressions.Regex.Replace(
            cs, @"(Password|pwd)=([^;]*)", "$1=***",
            System.Text.RegularExpressions.RegexOptions.IgnoreCase);
}

public record ConnectionTestResult(bool IsSuccess, string ConnectionString, string? ErrorMessage)
{
    public static ConnectionTestResult Success(string cs) => new(true, cs, null);
    public static ConnectionTestResult Failure(string error) => new(false, string.Empty, error);
}

public interface IConfigurationDemoService
{
    ConfigSummary GetSummary();
}

public class ConfigurationDemoService : IConfigurationDemoService
{
    private readonly IConfiguration _config;
    private readonly IOptionsSnapshot<DatabaseOptions> _dbOptions;
    private readonly IOptionsSnapshot<JwtOptions> _jwtOptions;

    public ConfigurationDemoService(
        IConfiguration config,
        IOptionsSnapshot<DatabaseOptions> dbOptions,
        IOptionsSnapshot<JwtOptions> jwtOptions)
    {
        _config = config;
        _dbOptions = dbOptions;
        _jwtOptions = jwtOptions;
    }

    public ConfigSummary GetSummary()
    {
        return new ConfigSummary(
            AppName: _config["AppSettings:ApplicationName"] ?? "Unknown",
            DatabasePoolSize: _dbOptions.Value.MaxPoolSize,
            JwtIssuer: _jwtOptions.Value.Issuer,
            Environment: Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown"
        );
    }
}

public record ConfigSummary(string AppName, int DatabasePoolSize, string JwtIssuer, string Environment);
```

### appsettings.json สำหรับโปรแกรม

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Database": {
    "ConnectionString": "Server=localhost;Database=MyAppDb;User=sa;Password=P@ssword123;TrustServerCertificate=True;",
    "MaxPoolSize": 100,
    "CommandTimeoutSeconds": 30,
    "EnableRetry": true,
    "MaxRetryCount": 3
  },
  "Jwt": {
    "Issuer": "https://myapp.com",
    "Audience": "https://myapp.com",
    "SecretKey": "my-32-character-long-secret-key-here!",
    "ExpiryMinutes": 60,
    "RefreshTokenExpiryDays": 7
  },
  "Email": {
    "Host": "smtp.gmail.com",
    "Port": 587,
    "UseSsl": true,
    "FromEmail": "noreply@myapp.com",
    "FromName": "My Application"
  },
  "AppSettings": {
    "ApplicationName": "My Application",
    "Version": "1.0.0"
  }
}
```

---

## 9. Configuration ขั้นสูง

### 9.1 Custom Configuration Source

```csharp
// สร้าง configuration source จากฐานข้อมูล
public class DatabaseConfigurationSource : IConfigurationSource
{
    private readonly string _connectionString;

    public DatabaseConfigurationSource(string connectionString)
    {
        _connectionString = connectionString;
    }

    public IConfigurationProvider Build(IConfigurationBuilder builder)
    {
        return new DatabaseConfigurationProvider(_connectionString);
    }
}

public class DatabaseConfigurationProvider : ConfigurationProvider
{
    private readonly string _connectionString;

    public DatabaseConfigurationProvider(string connectionString)
    {
        _connectionString = connectionString;
    }

    public override void Load()
    {
        // โหลด config จากฐานข้อมูล
        // Data = new Dictionary<string, string?>
        // {
        //     ["Feature:EnableNewUI"] = "true",
        //     ["Feature:MaxItems"] = "50"
        // };
    }
}

// ใช้งาน
builder.Configuration.Add(new DatabaseConfigurationSource(connectionString));
```

### 9.2 Reload Configuration

```csharp
// Configuration จาก JSON files จะ reload อัตโนมัติเมื่อไฟล์เปลี่ยน
// เพราะ reloadOnChange: true เป็น default

// ใช้ IOptionsMonitor เพื่อรับ notification
builder.Services.AddOptions<AppSettings>()
    .BindConfiguration("AppSettings");

// ใน service
public class MyService
{
    private readonly IOptionsMonitor<AppSettings> _monitor;
    
    public MyService(IOptionsMonitor<AppSettings> monitor)
    {
        _monitor = monitor;
        monitor.OnChange(newSettings => 
        {
            Console.WriteLine("Settings reloaded!");
        });
    }
}
```

---

## Exercises

### Exercise 1: Feature Flags System
สร้าง feature flags system โดยใช้ configuration:

```csharp
// appsettings.json
// {
//   "FeatureFlags": {
//     "EnableNewCheckout": true,
//     "EnableRecommendations": false,
//     "MaxCartItems": 20,
//     "EnableBetaFeatures": false
//   }
// }

public class FeatureFlagOptions
{
    public const string SectionName = "FeatureFlags";
    public bool EnableNewCheckout { get; set; }
    public bool EnableRecommendations { get; set; }
    public int MaxCartItems { get; set; } = 10;
    public bool EnableBetaFeatures { get; set; }
}

// สร้าง:
// 1. ลงทะเบียน FeatureFlagOptions
// 2. IFeatureFlagService interface
// 3. FeatureFlagService implementation (ใช้ IOptionsMonitor)
// 4. Endpoint GET /features ที่แสดง feature flags ทั้งหมด
// 5. Endpoint GET /features/{name} ที่ตรวจสอบ feature flag เดียว
```

### Exercise 2: Multi-Environment Database Config
สร้าง database configuration สำหรับหลาย environments:

```csharp
// ต้องการไฟล์:
// appsettings.json - base config
// appsettings.Development.json - ใช้ local SQL Server
// appsettings.Staging.json - ใช้ staging database
// appsettings.Production.json - ใช้ production database

// และ:
// 1. DatabaseOptions class พร้อม validation
// 2. IDatabaseFactory ที่สร้าง connection ตาม options
// 3. Endpoint ที่แสดง current database settings (masked)
// 4. Health check endpoint ที่ทดสอบ database connection
```

### Exercise 3: Secret Manager Integration
ฝึกใช้ User Secrets:

```bash
# ต้องการ:
# 1. Init user secrets ในโปรเจค
# 2. เพิ่ม secrets เหล่านี้:
dotnet user-secrets set "Jwt:SecretKey" "development-only-secret-key-32-chars"
dotnet user-secrets set "Email:SmtpPassword" "dev-gmail-app-password"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "local-dev-connection-string"

# 3. สร้าง endpoint ที่แสดงว่า secret ถูก load ถูกต้อง (ไม่แสดงค่าจริง)
# 4. แสดงว่าค่าที่ได้มาจาก source ใด (JSON file, User Secrets, Env Var)
```

---

## สรุป

✅ Configuration system รับข้อมูลจากหลาย sources: JSON files, User Secrets, Env Vars, Command Line  
✅ ลำดับ priority: Command Line > Env Vars > User Secrets > appsettings.{env}.json > appsettings.json  
✅ `IConfiguration` ใช้อ่าน configuration ด้วย key-based access  
✅ User Secrets ใช้เก็บ sensitive data ใน development โดยไม่ commit ลง source code  
✅ Environment Variables ใช้ `__` (double underscore) สำหรับ nested keys  
✅ Options pattern แนะนำให้ใช้แทน `IConfiguration` โดยตรง  
✅ `IOptions<T>` - ค่าคงที่, `IOptionsSnapshot<T>` - per request, `IOptionsMonitor<T>` - hot reload + singleton  
✅ `ValidateDataAnnotations()` และ `ValidateOnStart()` ช่วยตรวจสอบ configuration ตอน start  

---

## Part ถัดไป

ใน **Part 046** เราจะเรียนรู้เรื่อง **MVC: Controllers** อย่างละเอียด:
- Controller class structure
- Action methods
- HTTP verbs: GET, POST, PUT, DELETE
- Route attributes
- Model binding
- ActionResult types

---

*Part 045/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
