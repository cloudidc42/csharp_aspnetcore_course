# Part 086: Azure Fundamentals กับ .NET

## เนื้อหาใน Part นี้
- Azure App Service
- Azure SQL Database
- Azure Blob Storage
- Azure Key Vault (secrets)
- Azure Application Insights
- โปรแกรมตัวอย่าง: Deploy to Azure

---

## 1. Azure App Service

Azure App Service เป็น PaaS สำหรับ host web applications, REST APIs, และ mobile backends

### Deploy ASP.NET Core ไป App Service

```bash
# Login to Azure
az login

# Create resource group
az group create --name myapp-rg --location eastasia

# Create App Service Plan
az appservice plan create \
  --name myapp-plan \
  --resource-group myapp-rg \
  --sku B2 \
  --is-linux

# Create Web App
az webapp create \
  --name myapp-api \
  --resource-group myapp-rg \
  --plan myapp-plan \
  --runtime "DOTNETCORE:9.0"

# Configure App Settings
az webapp config appsettings set \
  --name myapp-api \
  --resource-group myapp-rg \
  --settings \
    ASPNETCORE_ENVIRONMENT=Production \
    ConnectionStrings__DefaultConnection="Server=myserver.database.windows.net;..."

# Deploy from local zip
dotnet publish -c Release -o ./publish
cd ./publish
zip -r ../deploy.zip .
az webapp deploy \
  --name myapp-api \
  --resource-group myapp-rg \
  --src-path ../deploy.zip
```

### App Service Configuration ใน appsettings.json

```json
{
  "AzureAppService": {
    "DiagnosticServicesEndpoint": "https://eastasia-0.in.applicationinsights.azure.com/",
    "AzureStorageEnabled": "true"
  }
}
```

### Deployment Slots

```bash
# Create staging slot
az webapp deployment slot create \
  --name myapp-api \
  --resource-group myapp-rg \
  --slot staging

# Deploy to staging
az webapp deploy \
  --name myapp-api \
  --resource-group myapp-rg \
  --slot staging \
  --src-path deploy.zip

# Swap staging -> production
az webapp deployment slot swap \
  --name myapp-api \
  --resource-group myapp-rg \
  --slot staging \
  --target-slot production
```

---

## 2. Azure SQL Database

### การตั้งค่า Connection

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

// Connection string ปกติ
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(60);
        }));
```

### Managed Identity (ไม่ต้องใช้ Password)

```csharp
// เชื่อม Azure SQL ด้วย Managed Identity
builder.Services.AddDbContext<AppDbContext>(options =>
{
    var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
    
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(maxRetryCount: 3);
    });
});

// หรือใช้ Azure.Identity token
public class AzureSqlTokenProvider
{
    private readonly TokenCredential _credential;
    
    public AzureSqlTokenProvider()
    {
        _credential = new DefaultAzureCredential();
    }
    
    public async Task<string> GetAccessTokenAsync()
    {
        var tokenRequestContext = new TokenRequestContext(
            new[] { "https://database.windows.net/.default" });
        var token = await _credential.GetTokenAsync(tokenRequestContext);
        return token.Token;
    }
}

// Custom DbContext สำหรับ Managed Identity
public class AppDbContext : DbContext
{
    private readonly AzureSqlTokenProvider _tokenProvider;
    
    public AppDbContext(
        DbContextOptions<AppDbContext> options,
        AzureSqlTokenProvider tokenProvider) : base(options)
    {
        _tokenProvider = tokenProvider;
    }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        var connection = Database.GetDbConnection() as SqlConnection;
        if (connection != null)
        {
            connection.AccessToken = _tokenProvider.GetAccessTokenAsync().GetAwaiter().GetResult();
        }
    }
}
```

### Azure SQL Best Practices

```csharp
// Elastic Scale (Read-only replica)
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity",
    "ReadOnlyConnection": "Server=myserver.database.windows.net;Database=mydb;ApplicationIntent=ReadOnly;Authentication=Active Directory Managed Identity"
  }
}

// DI setup สำหรับ read/write splitting
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("ReadOnlyConnection")));
```

---

## 3. Azure Blob Storage

### การติดตั้ง

```bash
dotnet add package Azure.Storage.Blobs
dotnet add package Azure.Identity
```

### BlobStorageService

```csharp
// Services/BlobStorageService.cs
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Storage.Sas;

public interface IBlobStorageService
{
    Task<string> UploadAsync(Stream content, string fileName, string contentType, CancellationToken ct = default);
    Task<Stream?> DownloadAsync(string blobName, CancellationToken ct = default);
    Task<bool> DeleteAsync(string blobName, CancellationToken ct = default);
    Task<string> GetDownloadUrlAsync(string blobName, TimeSpan expiry);
    Task<IEnumerable<BlobItemInfo>> ListAsync(string? prefix = null);
}

public class BlobStorageService : IBlobStorageService
{
    private readonly BlobContainerClient _containerClient;
    private readonly ILogger<BlobStorageService> _logger;

    public BlobStorageService(
        BlobServiceClient blobServiceClient,
        IConfiguration config,
        ILogger<BlobStorageService> logger)
    {
        var containerName = config["Azure:BlobStorage:ContainerName"] ?? "uploads";
        _containerClient = blobServiceClient.GetBlobContainerClient(containerName);
        _logger = logger;
    }

    public async Task<string> UploadAsync(
        Stream content, 
        string fileName, 
        string contentType,
        CancellationToken ct = default)
    {
        // สร้าง container ถ้ายังไม่มี
        await _containerClient.CreateIfNotExistsAsync(
            PublicAccessType.None, cancellationToken: ct);

        // สร้าง unique name
        var blobName = $"{DateTime.UtcNow:yyyy/MM/dd}/{Guid.NewGuid()}-{fileName}";
        var blobClient = _containerClient.GetBlobClient(blobName);

        var uploadOptions = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders
            {
                ContentType = contentType
            },
            Metadata = new Dictionary<string, string>
            {
                { "OriginalFileName", fileName },
                { "UploadedAt", DateTime.UtcNow.ToString("O") }
            }
        };

        await blobClient.UploadAsync(content, uploadOptions, ct);
        
        _logger.LogInformation("Uploaded blob: {BlobName}", blobName);
        return blobName;
    }

    public async Task<Stream?> DownloadAsync(string blobName, CancellationToken ct = default)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        
        if (!await blobClient.ExistsAsync(ct))
            return null;

        var download = await blobClient.DownloadStreamingAsync(cancellationToken: ct);
        return download.Value.Content;
    }

    public async Task<bool> DeleteAsync(string blobName, CancellationToken ct = default)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        var response = await blobClient.DeleteIfExistsAsync(cancellationToken: ct);
        return response.Value;
    }

    public async Task<string> GetDownloadUrlAsync(string blobName, TimeSpan expiry)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        
        // Check if using managed identity (no account key)
        if (_containerClient.CanGenerateSasUri)
        {
            var sasUri = blobClient.GenerateSasUri(BlobSasPermissions.Read, 
                DateTimeOffset.UtcNow.Add(expiry));
            return sasUri.ToString();
        }
        
        // Fallback: return public URL (ถ้า container เป็น public)
        return blobClient.Uri.ToString();
    }

    public async Task<IEnumerable<BlobItemInfo>> ListAsync(string? prefix = null)
    {
        var blobs = new List<BlobItemInfo>();
        
        await foreach (var blobItem in _containerClient.GetBlobsAsync(prefix: prefix))
        {
            blobs.Add(new BlobItemInfo(
                blobItem.Name,
                blobItem.Properties.ContentLength ?? 0,
                blobItem.Properties.ContentType ?? "application/octet-stream",
                blobItem.Properties.LastModified?.DateTime ?? DateTime.MinValue
            ));
        }
        
        return blobs;
    }
}

public record BlobItemInfo(
    string Name,
    long Size,
    string ContentType,
    DateTime LastModified);
```

### Registration

```csharp
// Program.cs
using Azure.Identity;
using Azure.Storage.Blobs;

// ด้วย Managed Identity (แนะนำสำหรับ production)
builder.Services.AddSingleton(new BlobServiceClient(
    new Uri($"https://{builder.Configuration["Azure:BlobStorage:AccountName"]}.blob.core.windows.net"),
    new DefaultAzureCredential()));

// ด้วย Connection String (development)
builder.Services.AddSingleton(
    new BlobServiceClient(builder.Configuration["Azure:BlobStorage:ConnectionString"]));

builder.Services.AddScoped<IBlobStorageService, BlobStorageService>();
```

### Upload Endpoint

```csharp
app.MapPost("/api/files/upload", async (
    IFormFile file,
    IBlobStorageService blobStorage,
    HttpContext context) =>
{
    if (file.Length == 0)
        return Results.BadRequest("File is empty");

    if (file.Length > 10 * 1024 * 1024) // 10MB limit
        return Results.BadRequest("File too large (max 10MB)");

    var allowedTypes = new[] { "image/jpeg", "image/png", "image/gif", "application/pdf" };
    if (!allowedTypes.Contains(file.ContentType))
        return Results.BadRequest("File type not allowed");

    using var stream = file.OpenReadStream();
    var blobName = await blobStorage.UploadAsync(stream, file.FileName, file.ContentType);
    
    var downloadUrl = await blobStorage.GetDownloadUrlAsync(blobName, TimeSpan.FromHours(1));
    
    return Results.Ok(new { BlobName = blobName, DownloadUrl = downloadUrl });
})
.DisableAntiforgery()
.RequireAuthorization();
```

---

## 4. Azure Key Vault

Key Vault เก็บ secrets, keys, และ certificates อย่างปลอดภัย

### การติดตั้ง

```bash
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Azure.Identity
```

### Integration กับ ASP.NET Core Configuration

```csharp
// Program.cs - เพิ่ม Key Vault เป็น configuration provider
using Azure.Identity;
using Azure.Extensions.AspNetCore.Configuration.Secrets;

var builder = WebApplication.CreateBuilder(args);

// เพิ่ม Key Vault หลังจาก appsettings
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUrl = builder.Configuration["Azure:KeyVault:Url"];
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUrl!),
        new DefaultAzureCredential(),
        new KeyVaultSecretManager());
}

// ใช้งานเหมือน config ปกติ
// Secret ชื่อ "MyApp--ConnectionString" ใน Key Vault
// จะอ่านได้เป็น Configuration["MyApp:ConnectionString"]
```

### Custom Secret Manager

```csharp
// KeyVaultSecretManager ที่ custom prefix
public class PrefixKeyVaultSecretManager : KeyVaultSecretManager
{
    private readonly string _prefix;
    
    public PrefixKeyVaultSecretManager(string prefix)
    {
        _prefix = $"{prefix}-";
    }
    
    public override bool Load(SecretProperties secret)
    {
        // เฉพาะ secrets ที่เริ่มด้วย prefix
        return secret.Name.StartsWith(_prefix);
    }
    
    public override string GetKey(KeyVaultSecret secret)
    {
        // "MyApp-ConnectionStrings--DefaultConnection" → "ConnectionStrings:DefaultConnection"
        return secret.Name[_prefix.Length..].Replace("--", ConfigurationPath.KeyDelimiter);
    }
}

// ใช้งาน
builder.Configuration.AddAzureKeyVault(
    new Uri(keyVaultUrl!),
    new DefaultAzureCredential(),
    new PrefixKeyVaultSecretManager("MyApp"));
```

### ใช้ Key Vault โดยตรง

```csharp
// Services/KeyVaultService.cs
using Azure.Security.KeyVault.Secrets;

public class KeyVaultService
{
    private readonly SecretClient _secretClient;

    public KeyVaultService(IConfiguration config)
    {
        var keyVaultUrl = config["Azure:KeyVault:Url"]!;
        _secretClient = new SecretClient(
            new Uri(keyVaultUrl),
            new DefaultAzureCredential());
    }

    public async Task<string?> GetSecretAsync(string secretName)
    {
        try
        {
            var secret = await _secretClient.GetSecretAsync(secretName);
            return secret.Value.Value;
        }
        catch (Azure.RequestFailedException ex) when (ex.Status == 404)
        {
            return null;
        }
    }

    public async Task SetSecretAsync(string secretName, string secretValue)
    {
        await _secretClient.SetSecretAsync(secretName, secretValue);
    }

    public async Task DeleteSecretAsync(string secretName)
    {
        var operation = await _secretClient.StartDeleteSecretAsync(secretName);
        await operation.WaitForCompletionAsync();
    }
}
```

---

## 5. Azure Application Insights

Application Insights ให้ monitoring, logging, และ distributed tracing

### การติดตั้ง

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet add package Microsoft.ApplicationInsights.WorkerService
```

### Configuration

```csharp
// Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    options.EnableAdaptiveSampling = true;
    options.EnableQuickPulseMetricStream = true;
});

// Custom telemetry
builder.Services.AddSingleton<ITelemetryInitializer, UserTelemetryInitializer>();
```

### Custom TelemetryInitializer

```csharp
// Telemetry/UserTelemetryInitializer.cs
using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.Extensibility;

public class UserTelemetryInitializer : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public UserTelemetryInitializer(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Initialize(ITelemetry telemetry)
    {
        var context = _httpContextAccessor.HttpContext;
        if (context == null) return;

        var userId = context.User?.FindFirst("sub")?.Value;
        if (userId != null)
        {
            telemetry.Context.User.Id = userId;
            telemetry.Context.User.AuthenticatedUserId = userId;
        }

        var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault();
        if (correlationId != null)
        {
            telemetry.Context.GlobalProperties["CorrelationId"] = correlationId;
        }

        telemetry.Context.GlobalProperties["Environment"] = 
            Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Unknown";
    }
}
```

### Custom Tracking

```csharp
// Services/OrderService.cs
public class OrderService
{
    private readonly TelemetryClient _telemetry;
    private readonly IOrderRepository _repository;

    public OrderService(TelemetryClient telemetry, IOrderRepository repository)
    {
        _telemetry = telemetry;
        _repository = repository;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // Track custom event
        _telemetry.TrackEvent("OrderCreated", new Dictionary<string, string>
        {
            { "CustomerId", request.CustomerId.ToString() },
            { "ItemCount", request.Items.Count.ToString() }
        }, new Dictionary<string, double>
        {
            { "OrderValue", (double)request.Items.Sum(i => i.Price * i.Quantity) }
        });

        // Track metric
        _telemetry.TrackMetric("OrderValue", 
            (double)request.Items.Sum(i => i.Price * i.Quantity));

        // Custom dependency tracking
        var stopwatch = Stopwatch.StartNew();
        Order? order = null;
        
        try
        {
            order = await _repository.CreateAsync(request);
            
            _telemetry.TrackDependency(
                "Database", "Orders", "CreateOrder",
                DateTime.UtcNow.Subtract(stopwatch.Elapsed),
                stopwatch.Elapsed,
                success: true);
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex, new Dictionary<string, string>
            {
                { "CustomerId", request.CustomerId.ToString() }
            });
            
            _telemetry.TrackDependency(
                "Database", "Orders", "CreateOrder",
                DateTime.UtcNow.Subtract(stopwatch.Elapsed),
                stopwatch.Elapsed,
                success: false);
            
            throw;
        }
        
        return order;
    }
}
```

### Kusto Query Language (KQL) สำหรับ query logs

```kql
// Application Insights Queries

// Top 10 slow requests
requests
| where timestamp > ago(1h)
| where success == true
| project timestamp, name, duration, url
| order by duration desc
| take 10

// Error rate over time
requests
| where timestamp > ago(1d)
| summarize 
    totalCount = count(),
    failedCount = countif(success == false)
    by bin(timestamp, 5m)
| project timestamp, errorRate = (failedCount * 100.0) / totalCount

// Custom events
customEvents
| where name == "OrderCreated"
| project timestamp, toreal(customMeasurements.OrderValue)
| summarize totalRevenue = sum(toreal(customMeasurements_OrderValue))
    by bin(timestamp, 1h)
```

---

## 6. โปรแกรมตัวอย่าง: Deploy to Azure

### Bicep Infrastructure as Code

```bicep
// infra/main.bicep
param location string = resourceGroup().location
param appName string
param environment string = 'prod'

var resourcePrefix = '${appName}-${environment}'

// Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: '${resourcePrefix}-logs'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// Application Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: '${resourcePrefix}-insights'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: '${resourcePrefix}-kv'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
  }
}

// Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: replace('${resourcePrefix}sa', '-', '')
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: '${resourcePrefix}-plan'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
  }
  properties: {
    reserved: true  // Linux
  }
}

// App Service
resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: '${resourcePrefix}-app'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|9.0'
      alwaysOn: true
      minTlsVersion: '1.2'
      appSettings: [
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
        {
          name: 'Azure__KeyVault__Url'
          value: keyVault.properties.vaultUri
        }
        {
          name: 'Azure__BlobStorage__AccountName'
          value: storageAccount.name
        }
      ]
    }
  }
}

// Grant App Service access to Key Vault
resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: keyVault
  name: guid(keyVault.id, appService.id, 'KeyVaultSecretsUser')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Grant App Service access to Storage
resource storageRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, appService.id, 'StorageBlobDataContributor')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

output appServiceUrl string = 'https://${appService.properties.defaultHostName}'
output keyVaultUrl string = keyVault.properties.vaultUri
```

### Full Azure Program.cs

```csharp
// Program.cs
using Azure.Identity;
using Azure.Storage.Blobs;
using Microsoft.ApplicationInsights.AspNetCore.Extensions;

var builder = WebApplication.CreateBuilder(args);

// ─── Azure Configuration ───────────────────────────────────────

// Key Vault (production only)
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUrl = builder.Configuration["Azure:KeyVault:Url"]!;
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUrl),
        new DefaultAzureCredential());
}

// ─── Services ─────────────────────────────────────────────────

// Application Insights
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
    options.EnableAdaptiveSampling = false;  // Disable sampling ใน production
});

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection"),
        sql => sql.EnableRetryOnFailure(3)));

// Blob Storage
builder.Services.AddSingleton(sp =>
{
    var accountName = builder.Configuration["Azure:BlobStorage:AccountName"];
    return new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        new DefaultAzureCredential());
});
builder.Services.AddScoped<IBlobStorageService, BlobStorageService>();

// Health checks
builder.Services.AddHealthChecks()
    .AddSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        tags: new[] { "ready" })
    .AddAzureBlobStorage(
        builder.Configuration["Azure:BlobStorage:ConnectionString"] ?? "",
        tags: new[] { "ready" });

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapHealthChecks("/health/live");
app.MapHealthChecks("/health/ready");

// File upload endpoint
app.MapPost("/api/files", async (IFormFile file, IBlobStorageService blobService) =>
{
    using var stream = file.OpenReadStream();
    var blobName = await blobService.UploadAsync(stream, file.FileName, file.ContentType);
    return Results.Ok(new { BlobName = blobName });
})
.DisableAntiforgery()
.RequireAuthorization();

app.Run();
```

### GitHub Actions Deploy to Azure

```yaml
# .github/workflows/deploy-azure.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '9.0.x'
    
    - name: Build
      run: dotnet publish src/MyApp.API -c Release -o ./publish
    
    - name: Login to Azure
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}
    
    - name: Deploy Infra (Bicep)
      run: |
        az deployment group create \
          --resource-group myapp-rg \
          --template-file infra/main.bicep \
          --parameters appName=myapp environment=prod
    
    - name: Deploy to App Service
      uses: azure/webapps-deploy@v3
      with:
        app-name: myapp-prod-app
        package: ./publish
    
    - name: Verify deployment
      run: |
        sleep 30
        curl -f https://myapp-prod-app.azurewebsites.net/health || exit 1
```

---

## Exercises / Project Tasks

### Exercise 1: App Service Deployment
Deploy ASP.NET Core API ไป Azure App Service:
- Setup deployment slot
- Configure App Settings
- Test slot swap

### Exercise 2: Blob Storage Integration
เพิ่ม file upload/download:
- Upload image ไป Blob Storage
- Generate SAS URL สำหรับ download
- Implement file listing

### Exercise 3: Key Vault Integration
ย้าย secrets ไป Key Vault:
- Connection strings
- JWT secrets
- API keys
- ใช้ Managed Identity (ไม่มี password)

### Exercise 4: Application Insights
เพิ่ม observability:
- Custom events
- Custom metrics
- Exception tracking
- Dashboard ใน Azure portal

---

## สรุป

- **App Service** เป็น PaaS ที่ง่ายที่สุดในการ deploy .NET apps
- **Azure SQL Database** รองรับ retry logic สำหรับ transient failures
- **Blob Storage** เหมาะสำหรับ unstructured data เช่น images, documents
- **Key Vault** เก็บ secrets อย่างปลอดภัย ไม่ต้อง hardcode ใน code
- **Managed Identity** ให้ Azure services คุยกันโดยไม่ต้องใช้ passwords
- **Application Insights** ให้ monitoring ครบวงจรสำหรับ production apps
- **Bicep** ช่วยจัดการ infrastructure as code

---

## Part ถัดไป

**Part 087: Performance Optimization** - เรียนรู้การ optimize .NET applications ให้เร็วและประหยัดทรัพยากร

---

*Part 086/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
