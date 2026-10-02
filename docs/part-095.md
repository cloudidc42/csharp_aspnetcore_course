# Part 095: Real-world Project: SaaS Starter

## เนื้อหาใน Part นี้
- Multi-tenancy architecture
- Subscription management
- Feature flags
- Analytics tracking
- Complete SaaS template

---

## 1. Multi-Tenancy Overview

SaaS application ต้องรองรับ multiple tenants บน infrastructure เดียวกัน

### Multi-tenancy Strategies
```
1. Database per Tenant   - Isolation สูง, ค่าใช้จ่ายสูง
2. Schema per Tenant     - Balance ระหว่าง isolation กับ cost
3. Shared Database       - Cost-effective, ต้องระวัง data leak
```

ใช้ **Shared Database with Tenant ID** + Row-Level Security

---

## 2. Tenant Domain

```csharp
// Domain/Entities/Tenant.cs
public class Tenant
{
    public Guid Id { get; private set; }
    public string Slug { get; private set; } = string.Empty;
    public string Name { get; private set; } = string.Empty;
    public string? LogoUrl { get; private set; }
    public string PlanId { get; private set; } = string.Empty;
    public TenantStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? TrialEndsAt { get; private set; }

    private Tenant() { }

    public static Tenant Create(string name, string slug, string planId, int trialDays = 14)
    {
        return new Tenant
        {
            Id = Guid.NewGuid(),
            Name = name,
            Slug = slug.ToLower().Replace(" ", "-"),
            PlanId = planId,
            Status = TenantStatus.Trial,
            TrialEndsAt = DateTime.UtcNow.AddDays(trialDays),
            CreatedAt = DateTime.UtcNow
        };
    }

    public void Activate() => Status = TenantStatus.Active;
    public void Suspend(string reason) => Status = TenantStatus.Suspended;
    public bool IsTrialExpired() => Status == TenantStatus.Trial && TrialEndsAt < DateTime.UtcNow;
}

public enum TenantStatus { Trial, Active, Suspended, Cancelled }

// Domain/Entities/Subscription.cs
public class Subscription
{
    public Guid Id { get; private set; }
    public Guid TenantId { get; private set; }
    public string PlanId { get; private set; } = string.Empty;
    public SubscriptionStatus Status { get; private set; }
    public string? ExternalSubscriptionId { get; private set; }
    public DateTime StartDate { get; private set; }
    public DateTime? EndDate { get; private set; }
    public DateTime? NextBillingDate { get; private set; }
    public decimal MonthlyAmount { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private Subscription() { }

    public static Subscription Create(Guid tenantId, string planId, decimal amount)
    {
        return new Subscription
        {
            Id = Guid.NewGuid(),
            TenantId = tenantId,
            PlanId = planId,
            Status = SubscriptionStatus.Active,
            StartDate = DateTime.UtcNow,
            NextBillingDate = DateTime.UtcNow.AddMonths(1),
            MonthlyAmount = amount,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void Cancel(DateTime? endDate = null)
    {
        Status = SubscriptionStatus.Cancelled;
        EndDate = endDate ?? DateTime.UtcNow;
    }

    public bool IsActive() => Status == SubscriptionStatus.Active && 
                               (EndDate == null || EndDate > DateTime.UtcNow);
}

public enum SubscriptionStatus { Active, PastDue, Cancelled, Paused }
```

---

## 3. Tenant Context

```csharp
// Infrastructure/Multitenancy/ITenantContext.cs
public interface ITenantContext
{
    Guid TenantId { get; }
    string TenantSlug { get; }
    string PlanId { get; }
    bool HasFeature(string featureKey);
}

// Infrastructure/Multitenancy/TenantContext.cs
public class TenantContext : ITenantContext
{
    public Guid TenantId { get; private set; }
    public string TenantSlug { get; private set; } = string.Empty;
    public string PlanId { get; private set; } = string.Empty;
    private readonly HashSet<string> _features = new();

    public void SetTenant(Tenant tenant, IEnumerable<string> features)
    {
        TenantId = tenant.Id;
        TenantSlug = tenant.Slug;
        PlanId = tenant.PlanId;
        _features.UnionWith(features);
    }

    public bool HasFeature(string featureKey) => _features.Contains(featureKey);
}

// Infrastructure/Multitenancy/TenantMiddleware.cs
public class TenantMiddleware
{
    private readonly RequestDelegate _next;

    public TenantMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(
        HttpContext context,
        ITenantRepository tenantRepo,
        IPlanFeatureService planFeatureService,
        TenantContext tenantContext)
    {
        // Resolve tenant from subdomain or header
        string? tenantSlug = null;

        // Check custom header (for API clients)
        if (context.Request.Headers.TryGetValue("X-Tenant", out var headerValue))
        {
            tenantSlug = headerValue.FirstOrDefault();
        }
        // Check subdomain
        else if (context.Request.Host.HasValue)
        {
            var host = context.Request.Host.Host;
            var parts = host.Split('.');
            if (parts.Length >= 3)  // subdomain.domain.tld
                tenantSlug = parts[0];
        }
        // Check JWT claim
        else if (context.User.Identity?.IsAuthenticated == true)
        {
            tenantSlug = context.User.FindFirst("tenant_slug")?.Value;
        }

        if (tenantSlug != null)
        {
            var tenant = await tenantRepo.GetBySlugAsync(tenantSlug, context.RequestAborted);
            if (tenant != null && tenant.Status != TenantStatus.Suspended)
            {
                var features = await planFeatureService.GetFeaturesAsync(tenant.PlanId);
                tenantContext.SetTenant(tenant, features);
            }
        }

        await _next(context);
    }
}
```

---

## 4. Multi-tenant DbContext

```csharp
// Infrastructure/Data/SaasDbContext.cs
public class SaasDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public SaasDbContext(DbContextOptions<SaasDbContext> options, ITenantContext tenantContext)
        : base(options)
    {
        _tenantContext = tenantContext;
    }

    public DbSet<Tenant> Tenants => Set<Tenant>();
    public DbSet<TenantUser> TenantUsers => Set<TenantUser>();
    public DbSet<Subscription> Subscriptions => Set<Subscription>();
    public DbSet<Project> Projects => Set<Project>();
    public DbSet<UsageEvent> UsageEvents => Set<UsageEvent>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Auto-apply tenant filter to all ITenantEntity
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                var method = GetType()
                    .GetMethod(nameof(ApplyTenantFilter), BindingFlags.NonPublic | BindingFlags.Static)!
                    .MakeGenericMethod(entityType.ClrType);
                method.Invoke(null, [modelBuilder, _tenantContext]);
            }
        }

        base.OnModelCreating(modelBuilder);
    }

    private static void ApplyTenantFilter<T>(ModelBuilder modelBuilder, ITenantContext tenantContext)
        where T : class, ITenantEntity
    {
        modelBuilder.Entity<T>().HasQueryFilter(e => e.TenantId == tenantContext.TenantId);
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Auto-set TenantId on new entities
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.TenantId = _tenantContext.TenantId;
        }
        return base.SaveChangesAsync(ct);
    }
}

public interface ITenantEntity
{
    Guid TenantId { get; set; }
}

// Example entity
public class Project : ITenantEntity
{
    public Guid Id { get; set; }
    public Guid TenantId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

---

## 5. Feature Flags

```csharp
// Application/Services/FeatureFlagService.cs
public interface IFeatureFlagService
{
    Task<bool> IsEnabledAsync(string featureKey, Guid? tenantId = null);
    Task<T?> GetValueAsync<T>(string featureKey, T? defaultValue = default);
}

public class FeatureFlagService : IFeatureFlagService
{
    private readonly IFeatureFlagRepository _repo;
    private readonly ITenantContext _tenantContext;
    private readonly ICacheService _cache;

    public FeatureFlagService(
        IFeatureFlagRepository repo,
        ITenantContext tenantContext,
        ICacheService cache)
    {
        _repo = repo;
        _tenantContext = tenantContext;
        _cache = cache;
    }

    public async Task<bool> IsEnabledAsync(string featureKey, Guid? tenantId = null)
    {
        var tid = tenantId ?? _tenantContext.TenantId;
        var cacheKey = $"ff:{tid}:{featureKey}";

        var cached = await _cache.GetAsync<bool?>(cacheKey);
        if (cached.HasValue) return cached.Value;

        // Check tenant-specific override first
        var tenantFlag = await _repo.GetTenantFlagAsync(tid, featureKey);
        if (tenantFlag != null)
        {
            await _cache.SetAsync(cacheKey, tenantFlag.IsEnabled, TimeSpan.FromMinutes(5));
            return tenantFlag.IsEnabled;
        }

        // Check plan-level feature
        var planEnabled = _tenantContext.HasFeature(featureKey);
        await _cache.SetAsync(cacheKey, planEnabled, TimeSpan.FromMinutes(5));
        return planEnabled;
    }

    public async Task<T?> GetValueAsync<T>(string featureKey, T? defaultValue = default)
    {
        var flag = await _repo.GetFlagWithValueAsync(_tenantContext.TenantId, featureKey);
        if (flag?.Value == null) return defaultValue;

        try
        {
            return JsonSerializer.Deserialize<T>(flag.Value);
        }
        catch
        {
            return defaultValue;
        }
    }
}

// Domain/Entities/FeatureFlag.cs
public class FeatureFlag
{
    public string Key { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public bool IsEnabled { get; set; }
    public string? Value { get; set; }  // JSON value
    public Guid? TenantId { get; set; }  // null = global flag
}

// Usage in controller/service:
public class ReportService
{
    private readonly IFeatureFlagService _featureFlags;

    public async Task<Report?> GenerateAdvancedReportAsync(Guid tenantId)
    {
        if (!await _featureFlags.IsEnabledAsync("advanced-reports"))
            throw new FeatureNotAvailableException("Advanced reports require Pro plan");

        var maxRows = await _featureFlags.GetValueAsync("report-max-rows", defaultValue: 1000);
        // Generate report with maxRows limit
        return null;
    }
}
```

---

## 6. Usage Analytics

```csharp
// Application/Services/AnalyticsService.cs
public class AnalyticsService : IAnalyticsService
{
    private readonly IUsageEventRepository _repo;
    private readonly ITenantContext _tenantContext;

    public AnalyticsService(IUsageEventRepository repo, ITenantContext tenantContext)
    {
        _repo = repo;
        _tenantContext = tenantContext;
    }

    public async Task TrackAsync(string eventName, Dictionary<string, object>? properties = null)
    {
        var ev = new UsageEvent
        {
            TenantId = _tenantContext.TenantId,
            EventName = eventName,
            Properties = properties != null
                ? JsonSerializer.Serialize(properties)
                : null,
            OccurredAt = DateTime.UtcNow
        };

        await _repo.SaveAsync(ev, CancellationToken.None);
    }

    public async Task<UsageSummary> GetSummaryAsync(Guid tenantId, DateTime from, DateTime to)
    {
        var events = await _repo.GetEventsAsync(tenantId, from, to);

        return new UsageSummary
        {
            TenantId = tenantId,
            Period = new DateRange(from, to),
            EventCounts = events
                .GroupBy(e => e.EventName)
                .ToDictionary(g => g.Key, g => g.Count()),
            DailyActive = events
                .Select(e => e.OccurredAt.Date)
                .Distinct()
                .Count()
        };
    }
}

public class UsageEvent : ITenantEntity
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public Guid TenantId { get; set; }
    public string EventName { get; set; } = string.Empty;
    public string? Properties { get; set; }
    public DateTime OccurredAt { get; set; }
}

public record UsageSummary(
    Guid TenantId,
    DateRange Period,
    Dictionary<string, int> EventCounts,
    int DailyActive);

public record DateRange(DateTime From, DateTime To);
```

---

## 7. Subscription Plans

```csharp
// Infrastructure/Services/PlanFeatureService.cs
public class PlanFeatureService : IPlanFeatureService
{
    private static readonly Dictionary<string, PlanDefinition> Plans = new()
    {
        ["free"] = new PlanDefinition("free", "Free", 0,
            Features: ["basic-reports", "up-to-5-users"]),

        ["starter"] = new PlanDefinition("starter", "Starter", 29,
            Features: ["basic-reports", "up-to-25-users", "api-access", "email-support"]),

        ["pro"] = new PlanDefinition("pro", "Pro", 99,
            Features: ["basic-reports", "advanced-reports", "unlimited-users",
                      "api-access", "priority-support", "custom-integrations",
                      "data-export", "audit-logs"]),

        ["enterprise"] = new PlanDefinition("enterprise", "Enterprise", 299,
            Features: ["basic-reports", "advanced-reports", "unlimited-users",
                      "api-access", "dedicated-support", "custom-integrations",
                      "data-export", "audit-logs", "sso", "custom-domain",
                      "sla-guarantee"])
    };

    public Task<IEnumerable<string>> GetFeaturesAsync(string planId)
    {
        if (Plans.TryGetValue(planId, out var plan))
            return Task.FromResult<IEnumerable<string>>(plan.Features);
        return Task.FromResult<IEnumerable<string>>(Array.Empty<string>());
    }

    public PlanDefinition? GetPlan(string planId) =>
        Plans.TryGetValue(planId, out var plan) ? plan : null;
}

public record PlanDefinition(
    string Id,
    string Name,
    decimal MonthlyPrice,
    IReadOnlyList<string> Features);
```

---

## 8. Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, cfg) => cfg.ReadFrom.Configuration(ctx.Configuration));

builder.Services.AddDbContext<SaasDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = builder.Configuration["Redis:ConnectionString"]);

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));

// Multi-tenancy
builder.Services.AddScoped<TenantContext>();
builder.Services.AddScoped<ITenantContext>(sp => sp.GetRequiredService<TenantContext>());
builder.Services.AddScoped<ITenantRepository, TenantRepository>();
builder.Services.AddScoped<IPlanFeatureService, PlanFeatureService>();

// SaaS services
builder.Services.AddScoped<ISubscriptionService, SubscriptionService>();
builder.Services.AddScoped<IFeatureFlagService, FeatureFlagService>();
builder.Services.AddScoped<IAnalyticsService, AnalyticsService>();
builder.Services.AddScoped<IUsageEventRepository, UsageEventRepository>();

// Auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt =>
    {
        var jwtConfig = builder.Configuration.GetSection("Jwt");
        opt.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtConfig["SecretKey"]!)),
            ValidIssuer = jwtConfig["Issuer"],
            ValidAudience = jwtConfig["Audience"]
        };
    });

builder.Services.AddAuthorization(opt =>
{
    opt.AddPolicy("TenantAdmin", policy => policy.RequireRole("TenantAdmin", "SuperAdmin"));
    opt.AddPolicy("SuperAdmin", policy => policy.RequireRole("SuperAdmin"));
});

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddHealthChecks()
    .AddNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")!);

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseSerilogRequestLogging();
app.UseMiddleware<TenantMiddleware>();
app.UseAuthentication();
app.UseAuthorization();

app.MapTenantEndpoints();
app.MapSubscriptionEndpoints();
app.MapProjectEndpoints();
app.MapAnalyticsEndpoints();
app.MapHealthChecks("/health");

app.Run();
```

---

## Exercises / Project Tasks

### Exercise 1: Usage Limits
เพิ่ม per-plan usage limits:
- UsageLimitChecker middleware
- Check before expensive operations
- Return 429 when limit exceeded

### Exercise 2: Stripe Webhook
จัดการ Stripe webhooks:
- `customer.subscription.updated`
- `invoice.payment_failed`
- `customer.subscription.deleted`

### Exercise 3: Tenant Onboarding
สร้าง onboarding flow:
- Register → Create Tenant → Invite Team
- Send welcome email ด้วย email template
- Setup wizard API

### Exercise 4: Audit Logs
เพิ่ม audit trail:
- Log all changes to important entities
- AuditLog table with before/after JSON
- Admin API to view audit logs

---

## สรุป

- **Multi-tenancy** ด้วย Row-Level Security และ Global Query Filter
- **Feature Flags** ที่ยืดหยุ่น รองรับทั้ง plan-level และ tenant-specific
- **Usage Analytics** track events สำหรับ billing และ insights
- **Subscription Plans** กำหนด features ต่อ plan
- **TenantMiddleware** resolve tenant จาก subdomain/header/JWT

---

## Part ถัดไป

**Part 096: Advanced C# 13 Features** - เรียนรู้ features ใหม่ใน C# 13

---

*Part 095/100 | Phase 7/7: ระดับโลก | หลักสูตร C# และ ASP.NET Core*
