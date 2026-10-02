# Part 083: Docker กับ .NET

## เนื้อหาใน Part นี้
- Docker คืออะไร
- Dockerfile สำหรับ .NET
- Multi-stage build
- docker-compose
- Container networking
- โปรแกรมตัวอย่าง: Containerize ASP.NET Core API

---

## 1. Docker คืออะไร

Docker เป็น platform สำหรับสร้าง, รัน และจัดการ containers

### ทำไมต้องใช้ Docker?

```
ปัญหาเดิม: "It works on my machine!" 
สาเหตุ: version ของ runtime, OS, dependencies ต่างกัน

Docker แก้ปัญหา:
Developer Machine ──Dockerfile──► Docker Image ──► Container
                                       │
                          Staging Server──► Container (เหมือนกัน)
                                           │
                          Production──────► Container (เหมือนกัน)
```

### Key Concepts

- **Image**: Blueprint ของ container (read-only)
- **Container**: Instance ที่รันจาก Image (writable layer บนสุด)
- **Dockerfile**: Script สำหรับสร้าง Image
- **Registry**: ที่เก็บ Images (Docker Hub, Azure Container Registry, etc.)
- **Volume**: Persistent storage สำหรับ container
- **Network**: Virtual network ระหว่าง containers

---

## 2. Dockerfile สำหรับ .NET

### Basic Dockerfile

```dockerfile
# Dockerfile พื้นฐาน (ไม่แนะนำสำหรับ production)
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o /app/publish
EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Multi-stage Build (แนะนำ)

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy project files ก่อน (ทำให้ layer cache ทำงานได้ดี)
COPY ["src/MyApp.API/MyApp.API.csproj", "src/MyApp.API/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj", "src/MyApp.Application/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj", "src/MyApp.Domain/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]

# Restore dependencies
RUN dotnet restore "src/MyApp.API/MyApp.API.csproj"

# Copy ทุกอย่าง
COPY . .

# Build
WORKDIR "/src/src/MyApp.API"
RUN dotnet build "MyApp.API.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "MyApp.API.csproj" -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# Stage 3: Final image (เล็กกว่ามาก เพราะไม่มี SDK)
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app

# Security: run as non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

EXPOSE 8080
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

### Docker Images สำหรับ .NET

```
mcr.microsoft.com/dotnet/sdk:9.0          - Full SDK (สำหรับ build)
mcr.microsoft.com/dotnet/aspnet:9.0       - ASP.NET Core runtime
mcr.microsoft.com/dotnet/runtime:9.0      - .NET runtime (console apps)
mcr.microsoft.com/dotnet/runtime-deps:9.0 - Dependencies only (สำหรับ AOT)
```

### .dockerignore

```
# .dockerignore
**/.git
**/.vs
**/bin
**/obj
**/.idea
**/TestResults
**/node_modules
*.user
*.suo
.env
.env.*
```

---

## 3. Dockerfile Best Practices

### Layer Caching

```dockerfile
# ไม่ดี - Copy ทุกอย่างก่อน แล้ว restore
# ทุกครั้งที่โค้ดเปลี่ยน ต้อง restore ใหม่
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

# ดี - Copy .csproj ก่อน แล้ว restore
# Layer restore จะ cache ไว้ถ้า .csproj ไม่เปลี่ยน
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["MyApp.API.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish
```

### Health Check ใน Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

### Environment Variables

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=publish /app/publish .

# Default environment
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080

EXPOSE 8080
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

---

## 4. docker-compose

### Basic docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp;Username=postgres;Password=postgres
      - Redis__ConnectionString=redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - backend

volumes:
  postgres_data:
  redis_data:

networks:
  backend:
    driver: bridge
```

### docker-compose.override.yml (Development)

```yaml
# docker-compose.override.yml - ใช้สำหรับ development
version: '3.8'

services:
  api:
    build:
      target: build  # ใช้ build stage แทน final
    volumes:
      - .:/src
      - /src/bin
      - /src/obj
    command: dotnet watch run --project src/MyApp.API/MyApp.API.csproj
    ports:
      - "5000:8080"
      - "5001:443"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1

  db:
    ports:
      - "5432:5432"  # Expose ให้ access จาก host ได้

  redis:
    ports:
      - "6379:6379"
```

### docker-compose.prod.yml (Production)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api:
    image: myregistry.azurecr.io/myapp:${TAG:-latest}
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    secrets:
      - db_password
      - jwt_secret

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
```

---

## 5. Container Networking

### Network Types

```yaml
# Bridge Network (default) - containers คุยกันได้ด้วย service name
networks:
  backend:
    driver: bridge

# Host Network - container ใช้ network ของ host โดยตรง
services:
  myapp:
    network_mode: "host"

# Overlay Network - สำหรับ Docker Swarm/multi-host
networks:
  overlay_net:
    driver: overlay
    attachable: true
```

### Service Discovery ใน docker-compose

```yaml
# Services คุยกันด้วย service name
services:
  api:
    # สามารถ connect ไป db ด้วย hostname "db"
    environment:
      - DB_HOST=db
      - REDIS_HOST=redis

  db:
    image: postgres:16

  redis:
    image: redis:7
```

### Expose vs Ports

```yaml
services:
  api:
    ports:
      - "5000:8080"  # host:container - accessible จาก outside
    expose:
      - "8080"  # เฉพาะ internal network ไม่ accessible จาก host

  db:
    # ไม่มี ports = accessible เฉพาะ containers ใน same network
    expose:
      - "5432"
```

---

## 6. Volume Management

```yaml
services:
  db:
    volumes:
      # Named volume (managed by Docker)
      - postgres_data:/var/lib/postgresql/data
      
      # Bind mount (map host path)
      - ./init-scripts:/docker-entrypoint-initdb.d
      
      # tmpfs mount (in-memory)
      - type: tmpfs
        target: /tmp

  api:
    volumes:
      # Read-only bind mount
      - ./config/appsettings.json:/app/appsettings.json:ro

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/postgres  # Custom path on host
```

---

## 7. โปรแกรมตัวอย่าง: Containerize ASP.NET Core API

### โปรเจกต์: Product Catalog API

```csharp
// ProductCatalog/Program.cs
using Microsoft.EntityFrameworkCore;
using StackExchange.Redis;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<ProductDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Redis Cache
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect(builder.Configuration["Redis:ConnectionString"] ?? "localhost:6379"));

builder.Services.AddScoped<ICacheService, RedisCacheService>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();

// Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Health checks
builder.Services.AddHealthChecks()
    .AddNpgsql(
        builder.Configuration.GetConnectionString("DefaultConnection")!,
        name: "database")
    .AddRedis(
        builder.Configuration["Redis:ConnectionString"] ?? "localhost:6379",
        name: "redis");

var app = builder.Build();

// Migrate database on startup
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<ProductDbContext>();
    await db.Database.MigrateAsync();
    await SeedData.SeedAsync(db);
}

app.UseSwagger();
app.UseSwaggerUI();

// Endpoints
var productsGroup = app.MapGroup("/api/products").WithTags("Products");

productsGroup.MapGet("/", async (
    IProductService service,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 10,
    [FromQuery] string? category = null) =>
{
    var products = await service.GetProductsAsync(page, pageSize, category);
    return Results.Ok(products);
});

productsGroup.MapGet("/{id:guid}", async (Guid id, IProductService service) =>
{
    var product = await service.GetProductByIdAsync(id);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

productsGroup.MapPost("/", async (CreateProductRequest request, IProductService service) =>
{
    var product = await service.CreateProductAsync(request);
    return Results.Created($"/api/products/{product.Id}", product);
});

productsGroup.MapPut("/{id:guid}", async (Guid id, UpdateProductRequest request, IProductService service) =>
{
    var product = await service.UpdateProductAsync(id, request);
    return product is null ? Results.NotFound() : Results.Ok(product);
});

productsGroup.MapDelete("/{id:guid}", async (Guid id, IProductService service) =>
{
    var deleted = await service.DeleteProductAsync(id);
    return deleted ? Results.NoContent() : Results.NotFound();
});

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready", new Microsoft.AspNetCore.Diagnostics.HealthChecks.HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.Run();
```

### ProductService

```csharp
// Services/ProductService.cs
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly ICacheService _cache;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IProductRepository repository,
        ICacheService cache,
        ILogger<ProductService> logger)
    {
        _repository = repository;
        _cache = cache;
        _logger = logger;
    }

    public async Task<PagedResult<ProductDto>> GetProductsAsync(
        int page, int pageSize, string? category)
    {
        var cacheKey = $"products:page:{page}:size:{pageSize}:cat:{category}";
        
        var cached = await _cache.GetAsync<PagedResult<ProductDto>>(cacheKey);
        if (cached is not null) return cached;

        var result = await _repository.GetPagedAsync(page, pageSize, category);
        await _cache.SetAsync(cacheKey, result, TimeSpan.FromMinutes(5));
        
        return result;
    }

    public async Task<ProductDto?> GetProductByIdAsync(Guid id)
    {
        var cacheKey = $"product:{id}";
        
        var cached = await _cache.GetAsync<ProductDto>(cacheKey);
        if (cached is not null) return cached;

        var product = await _repository.GetByIdAsync(id);
        if (product is null) return null;

        var dto = MapToDto(product);
        await _cache.SetAsync(cacheKey, dto, TimeSpan.FromMinutes(10));
        
        return dto;
    }

    public async Task<ProductDto> CreateProductAsync(CreateProductRequest request)
    {
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            Category = request.Category,
            Stock = request.InitialStock,
            CreatedAt = DateTime.UtcNow
        };

        await _repository.AddAsync(product);
        await _repository.SaveChangesAsync();
        
        // Invalidate list cache
        await _cache.RemovePatternAsync("products:*");
        
        return MapToDto(product);
    }

    public async Task<ProductDto?> UpdateProductAsync(Guid id, UpdateProductRequest request)
    {
        var product = await _repository.GetByIdAsync(id);
        if (product is null) return null;

        product.Name = request.Name;
        product.Description = request.Description;
        product.Price = request.Price;
        product.UpdatedAt = DateTime.UtcNow;

        await _repository.SaveChangesAsync();
        
        // Invalidate cache
        await _cache.RemoveAsync($"product:{id}");
        await _cache.RemovePatternAsync("products:*");
        
        return MapToDto(product);
    }

    public async Task<bool> DeleteProductAsync(Guid id)
    {
        var product = await _repository.GetByIdAsync(id);
        if (product is null) return false;

        _repository.Remove(product);
        await _repository.SaveChangesAsync();
        
        await _cache.RemoveAsync($"product:{id}");
        await _cache.RemovePatternAsync("products:*");
        
        return true;
    }

    private static ProductDto MapToDto(Product p) => new(
        p.Id, p.Name, p.Description, p.Price, p.Category, p.Stock, p.CreatedAt);
}
```

### Dockerfile สมบูรณ์

```dockerfile
# Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy solution and project files
COPY ["ProductCatalog.sln", "./"]
COPY ["src/ProductCatalog.API/ProductCatalog.API.csproj", "src/ProductCatalog.API/"]

# Restore
RUN dotnet restore "src/ProductCatalog.API/ProductCatalog.API.csproj"

# Copy source
COPY . .

# Build
WORKDIR "/src/src/ProductCatalog.API"
RUN dotnet build "ProductCatalog.API.csproj" -c Release -o /app/build --no-restore

# Publish
FROM build AS publish
RUN dotnet publish "ProductCatalog.API.csproj" -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# Final
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final

# Install curl for healthcheck
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

WORKDIR /app
COPY --from=publish /app/publish .

# Set ownership
RUN chown -R appuser:appgroup /app

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080

EXPOSE 8080

ENTRYPOINT ["dotnet", "ProductCatalog.API.dll"]
```

### docker-compose.yml สมบูรณ์

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: final
    image: product-catalog:latest
    container_name: product-catalog-api
    restart: unless-stopped
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Host=db;Port=5432;Database=productcatalog;Username=postgres;Password=${DB_PASSWORD}
      - Redis__ConnectionString=redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:16-alpine
    container_name: product-catalog-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: productcatalog
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: product-catalog-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --save 60 1 --loglevel warning
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    container_name: product-catalog-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### .env file

```env
# .env
DB_PASSWORD=MyStrongPassword123!
JWT_SECRET=my-very-secret-key-that-is-at-least-32-chars
ASPNETCORE_ENVIRONMENT=Production
```

### Makefile สำหรับ Docker commands

```makefile
# Makefile
.PHONY: build up down logs ps clean

build:
	docker compose build

up:
	docker compose up -d

down:
	docker compose down

logs:
	docker compose logs -f api

ps:
	docker compose ps

clean:
	docker compose down -v --remove-orphans
	docker image rm product-catalog:latest || true

shell:
	docker compose exec api /bin/bash

db-shell:
	docker compose exec db psql -U postgres productcatalog

migrate:
	docker compose exec api dotnet ef database update

restart:
	docker compose restart api
```

---

## 8. Docker Commands ที่ใช้บ่อย

```bash
# Build image
docker build -t myapp:latest .
docker build -t myapp:v1.0 --target build .  # Build specific stage

# Run container
docker run -d -p 5000:8080 --name myapp myapp:latest
docker run -d -p 5000:8080 -e ASPNETCORE_ENVIRONMENT=Development myapp:latest

# Container management
docker ps                    # List running containers
docker ps -a                 # List all containers
docker stop myapp
docker start myapp
docker rm myapp
docker logs myapp -f         # Follow logs
docker exec -it myapp bash   # Enter container

# Image management
docker images
docker rmi myapp:latest
docker pull mcr.microsoft.com/dotnet/aspnet:9.0

# docker-compose
docker compose up -d         # Start all services
docker compose down          # Stop and remove containers
docker compose logs -f       # Follow logs
docker compose ps            # List services
docker compose exec api bash # Enter service container
docker compose build --no-cache  # Rebuild without cache

# Clean up
docker system prune -f       # Remove unused containers, networks, images
docker volume prune -f       # Remove unused volumes
```

---

## Exercises / Project Tasks

### Exercise 1: Containerize API
เอา ASP.NET Core API ของคุณ:
- สร้าง Dockerfile แบบ multi-stage
- เพิ่ม healthcheck
- Run เป็น non-root user

### Exercise 2: Full Stack docker-compose
สร้าง docker-compose.yml ที่มี:
- ASP.NET Core API
- PostgreSQL database
- Redis cache
- nginx reverse proxy

### Exercise 3: Development vs Production
แยก configuration:
- docker-compose.yml (base)
- docker-compose.override.yml (development - hot reload)
- docker-compose.prod.yml (production)

### Exercise 4: Optimize Dockerfile
เปรียบเทียบ image size และ build time:
- ทำ layer caching ให้ดี
- ใช้ alpine base image
- Multi-stage build

---

## สรุป

- **Docker** ช่วยให้แอปพลิเคชัน run ได้เหมือนกันทุก environment
- **Multi-stage build** ลด image size โดยไม่เอา SDK ไปใน production image
- **Layer caching** ทำ build เร็วขึ้นด้วยการ copy .csproj แยกก่อน restore
- **docker-compose** จัดการหลาย containers พร้อมกัน
- **Networks** ใน docker-compose ทำให้ services คุยกันด้วย service name
- **Volumes** เก็บ persistent data ไว้แม้ container ถูก recreate
- **Health checks** ช่วยให้ orchestrator รู้ว่า container ready แล้ว

---

## Part ถัดไป

**Part 084: Kubernetes เบื้องต้น** - เรียนรู้การ deploy และจัดการ containers ด้วย Kubernetes

---

*Part 083/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
