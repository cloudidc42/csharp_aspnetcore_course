# Part 055: EF Core: Migrations

## เนื้อหาใน Part นี้
- Add-Migration
- Update-Database
- Remove-Migration
- Migration history
- Seeding data
- Applying migrations in code
- โปรแกรมตัวอย่าง: Schema evolution

---

## Migration คืออะไร

**Migration** คือไฟล์ C# ที่บันทึกการเปลี่ยนแปลง database schema เมื่อเวลาผ่านไป เปรียบเสมือน version control สำหรับ database schema

### ทำไมต้องใช้ Migrations?

```
v1: สร้าง Customers table
v2: เพิ่ม PhoneNumber column ใน Customers
v3: สร้าง Orders table
v4: เพิ่ม Index บน Email
```

Migrations ช่วยให้:
1. ติดตามการเปลี่ยนแปลง schema ใน Git
2. Apply การเปลี่ยนแปลงไปยัง environments ต่างๆ ได้อัตโนมัติ
3. Roll back การเปลี่ยนแปลงได้
4. ทำงานร่วมกันเป็นทีมได้ดีขึ้น

---

## ติดตั้ง EF Core Tools

### Global Tool (แนะนำ)

```bash
# ติดตั้ง
dotnet tool install --global dotnet-ef

# อัปเดต
dotnet tool update --global dotnet-ef

# ตรวจสอบ version
dotnet ef --version
```

### Local Tool (สำหรับ project-specific version)

```bash
# สร้าง tool manifest
dotnet new tool-manifest

# ติดตั้ง locally
dotnet tool install dotnet-ef

# ใช้งาน
dotnet dotnet-ef migrations add ...
```

### Package ที่ต้องการ

```bash
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.0
```

---

## Add-Migration

### คำสั่งพื้นฐาน

```bash
# สร้าง migration
dotnet ef migrations add <MigrationName>

# ตัวอย่าง
dotnet ef migrations add InitialCreate
dotnet ef migrations add AddPhoneNumberToCustomers
dotnet ef migrations add CreateOrdersTable
```

### Options

```bash
# ระบุ DbContext ที่ต้องการ (ถ้ามีหลาย DbContext)
dotnet ef migrations add InitialCreate --context BlogDbContext

# ระบุ project
dotnet ef migrations add InitialCreate --project src/MyApp.Data

# ระบุ startup project
dotnet ef migrations add InitialCreate --startup-project src/MyApp.Api

# ระบุ output directory
dotnet ef migrations add InitialCreate --output-dir Migrations/Blog
```

### ตัวอย่าง Migration ที่สร้างขึ้น

```csharp
// Migrations/20241001120000_InitialCreate.cs
using System;
using Microsoft.EntityFrameworkCore.Migrations;

#nullable disable

namespace MyApp.Migrations
{
    /// <inheritdoc />
    public partial class InitialCreate : Migration
    {
        /// <inheritdoc />
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "Customers",
                columns: table => new
                {
                    Id = table.Column<int>(type: "INTEGER", nullable: false)
                        .Annotation("Sqlite:Autoincrement", true),
                    FirstName = table.Column<string>(
                        type: "TEXT", maxLength: 100, nullable: false),
                    LastName = table.Column<string>(
                        type: "TEXT", maxLength: 100, nullable: false),
                    Email = table.Column<string>(
                        type: "TEXT", maxLength: 200, nullable: false),
                    CreatedAt = table.Column<DateTime>(type: "TEXT", nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Customers", x => x.Id);
                });

            migrationBuilder.CreateIndex(
                name: "IX_Customers_Email",
                table: "Customers",
                column: "Email",
                unique: true);
        }

        /// <inheritdoc />
        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(name: "Customers");
        }
    }
}
```

### Migration Snapshot

EF Core สร้าง `ModelSnapshot.cs` ที่บันทึก model ล่าสุด ใช้เปรียบเทียบเมื่อสร้าง migration ใหม่

```csharp
// Migrations/BlogDbContextModelSnapshot.cs
// ไฟล์นี้ถูกสร้างอัตโนมัติ ไม่ต้องแก้ไขด้วยตนเอง
[DbContext(typeof(BlogDbContext))]
partial class BlogDbContextModelSnapshot : ModelSnapshot
{
    protected override void BuildModel(ModelBuilder modelBuilder)
    {
        // ...snapshot of current model
    }
}
```

---

## Update-Database

### Apply Migrations

```bash
# Apply ทุก pending migrations
dotnet ef database update

# Apply ถึง migration ที่กำหนด
dotnet ef database update 20241001120000_InitialCreate

# Apply migration ก่อนหน้า (roll back 1 migration)
dotnet ef database update PreviousMigrationName

# Roll back ทั้งหมด (empty database แต่ยังมี schema)
dotnet ef database update 0
```

### Script SQL

```bash
# สร้าง SQL script แทน apply โดยตรง (สำหรับ Production)
dotnet ef migrations script

# Script ตั้งแต่ migration หนึ่งถึงอีกอัน
dotnet ef migrations script FromMigration ToMigration

# Idempotent script (ตรวจสอบก่อน apply)
dotnet ef migrations script --idempotent
```

ตัวอย่าง SQL script ที่สร้าง:

```sql
-- Script ที่ EF Core สร้าง
BEGIN TRANSACTION;
GO

CREATE TABLE [Customers] (
    [Id] int NOT NULL IDENTITY,
    [FirstName] nvarchar(100) NOT NULL,
    [LastName] nvarchar(100) NOT NULL,
    [Email] nvarchar(200) NOT NULL,
    [CreatedAt] datetime2 NOT NULL,
    CONSTRAINT [PK_Customers] PRIMARY KEY ([Id])
);
GO

CREATE UNIQUE INDEX [IX_Customers_Email] ON [Customers] ([Email]);
GO

INSERT INTO [__EFMigrationsHistory] ([MigrationId], [ProductVersion])
VALUES (N'20241001120000_InitialCreate', N'9.0.0');
GO

COMMIT;
GO
```

---

## Remove-Migration

```bash
# ลบ migration ล่าสุด (ต้องยัง NOT apply ใน database)
dotnet ef migrations remove

# Force remove (แม้ว่าจะ apply แล้ว - อันตราย!)
dotnet ef migrations remove --force
```

**คำเตือน**: ถ้า migration ถูก apply ใน database แล้ว ต้อง roll back ก่อน:

```bash
# 1. Roll back database
dotnet ef database update PreviousMigration

# 2. แล้วค่อย remove migration
dotnet ef migrations remove
```

---

## Migration History

EF Core ใช้ตาราง `__EFMigrationsHistory` เก็บประวัติ migrations ที่ถูก apply

```sql
-- ดู migration history
SELECT * FROM __EFMigrationsHistory;
```

| MigrationId | ProductVersion |
|-------------|----------------|
| 20241001120000_InitialCreate | 9.0.0 |
| 20241002090000_AddPhoneNumber | 9.0.0 |
| 20241003150000_CreateOrders | 9.0.0 |

### ดู Migrations ด้วย CLI

```bash
# List migrations
dotnet ef migrations list
```

```
20241001120000_InitialCreate (Applied)
20241002090000_AddPhoneNumber (Applied)
20241003150000_CreateOrders (Pending)
```

---

## Custom Migration Operations

### เพิ่ม SQL ใน Migration

```csharp
public partial class AddStoredProcedure : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // สร้าง stored procedure
        migrationBuilder.Sql(@"
            CREATE PROCEDURE GetCustomerOrders
                @CustomerId INT
            AS
            BEGIN
                SELECT * FROM Orders WHERE CustomerId = @CustomerId
                ORDER BY OrderDate DESC
            END");
        
        // สร้าง view
        migrationBuilder.Sql(@"
            CREATE VIEW CustomerOrderSummary AS
            SELECT 
                c.Id,
                c.FirstName + ' ' + c.LastName AS FullName,
                COUNT(o.Id) AS OrderCount,
                SUM(o.Total) AS TotalSpent
            FROM Customers c
            LEFT JOIN Orders o ON o.CustomerId = c.Id
            GROUP BY c.Id, c.FirstName, c.LastName");
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("DROP PROCEDURE IF EXISTS GetCustomerOrders");
        migrationBuilder.Sql("DROP VIEW IF EXISTS CustomerOrderSummary");
    }
}
```

### Data Migration (เปลี่ยน data format)

```csharp
public partial class SplitFullName : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // เพิ่ม columns ใหม่
        migrationBuilder.AddColumn<string>(
            name: "FirstName",
            table: "Customers",
            type: "TEXT",
            maxLength: 100,
            nullable: false,
            defaultValue: "");
        
        migrationBuilder.AddColumn<string>(
            name: "LastName",
            table: "Customers",
            type: "TEXT",
            maxLength: 100,
            nullable: false,
            defaultValue: "");
        
        // Migrate ข้อมูลจาก FullName -> FirstName + LastName
        migrationBuilder.Sql(@"
            UPDATE Customers 
            SET 
                FirstName = CASE 
                    WHEN instr(FullName, ' ') > 0 
                    THEN substr(FullName, 1, instr(FullName, ' ') - 1)
                    ELSE FullName
                END,
                LastName = CASE 
                    WHEN instr(FullName, ' ') > 0 
                    THEN substr(FullName, instr(FullName, ' ') + 1)
                    ELSE ''
                END");
        
        // ลบ column เก่า
        migrationBuilder.DropColumn(name: "FullName", table: "Customers");
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "FullName",
            table: "Customers",
            nullable: false,
            defaultValue: "");
        
        migrationBuilder.Sql(
            "UPDATE Customers SET FullName = FirstName || ' ' || LastName");
        
        migrationBuilder.DropColumn(name: "FirstName", table: "Customers");
        migrationBuilder.DropColumn(name: "LastName", table: "Customers");
    }
}
```

---

## Seeding Data

### HasData (ใน OnModelCreating)

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Seed data - ต้องมี hardcoded IDs
    modelBuilder.Entity<Category>().HasData(
        new Category { Id = 1, Name = "Electronics", Slug = "electronics" },
        new Category { Id = 2, Name = "Clothing", Slug = "clothing" },
        new Category { Id = 3, Name = "Food", Slug = "food" }
    );
    
    modelBuilder.Entity<Tag>().HasData(
        new Tag { Id = 1, Name = "C#", Slug = "csharp" },
        new Tag { Id = 2, Name = ".NET", Slug = "dotnet" },
        new Tag { Id = 3, Name = "EF Core", Slug = "efcore" }
    );
}
```

เมื่อ seed ใน migration จะได้:

```csharp
// Migrations/...AddSeedData.cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.InsertData(
        table: "Categories",
        columns: new[] { "Id", "Name", "Slug" },
        values: new object[,]
        {
            { 1, "Electronics", "electronics" },
            { 2, "Clothing", "clothing" },
            { 3, "Food", "food" }
        });
}
```

### Custom Seed (ใน Program.cs)

เหมาะสำหรับ seed data จำนวนมาก หรือ data ที่ซับซ้อน:

```csharp
// Services/DataSeeder.cs
using Microsoft.EntityFrameworkCore;

public class DataSeeder
{
    private readonly AppDbContext _context;
    private readonly ILogger<DataSeeder> _logger;
    
    public DataSeeder(AppDbContext context, ILogger<DataSeeder> logger)
    {
        _context = context;
        _logger = logger;
    }
    
    public async Task SeedAsync()
    {
        await SeedCategoriesAsync();
        await SeedProductsAsync();
        await SeedAdminUserAsync();
    }
    
    private async Task SeedCategoriesAsync()
    {
        if (await _context.Categories.AnyAsync())
        {
            _logger.LogInformation("Categories already seeded, skipping...");
            return;
        }
        
        var categories = new[]
        {
            new Category { Name = "Electronics", Slug = "electronics" },
            new Category { Name = "Clothing", Slug = "clothing" },
            new Category { Name = "Books", Slug = "books" },
            new Category { Name = "Sports", Slug = "sports" }
        };
        
        _context.Categories.AddRange(categories);
        await _context.SaveChangesAsync();
        
        _logger.LogInformation("Seeded {Count} categories", categories.Length);
    }
    
    private async Task SeedProductsAsync()
    {
        if (await _context.Products.AnyAsync())
        {
            _logger.LogInformation("Products already seeded, skipping...");
            return;
        }
        
        // ดึง categories ที่เพิ่งสร้าง
        var electronics = await _context.Categories
            .FirstAsync(c => c.Slug == "electronics");
        
        var products = Enumerable.Range(1, 20).Select(i => new Product
        {
            Name = $"Product {i}",
            Sku = $"PRD-{i:D5}",
            Price = Random.Shared.Next(100, 10000),
            StockQuantity = Random.Shared.Next(0, 100),
            CategoryId = electronics.Id
        }).ToList();
        
        _context.Products.AddRange(products);
        await _context.SaveChangesAsync();
        
        _logger.LogInformation("Seeded {Count} products", products.Count);
    }
    
    private async Task SeedAdminUserAsync()
    {
        if (await _context.Users.AnyAsync(u => u.Email == "admin@example.com"))
        {
            return;
        }
        
        var admin = new User
        {
            Email = "admin@example.com",
            Name = "System Admin",
            Role = "Admin"
        };
        
        _context.Users.Add(admin);
        await _context.SaveChangesAsync();
        
        _logger.LogInformation("Seeded admin user");
    }
}

// Program.cs
// Register seeder
builder.Services.AddScoped<DataSeeder>();

// Run seeder after build
var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var seeder = scope.ServiceProvider.GetRequiredService<DataSeeder>();
    await seeder.SeedAsync();
}
```

---

## Applying Migrations in Code

### EnsureCreated vs Migrate

```csharp
// EnsureCreated: สร้าง database schema โดยตรงจาก model
// ไม่สร้าง migration history table
// ไม่รองรับ incremental updates
// เหมาะสำหรับ testing เท่านั้น
await context.Database.EnsureCreatedAsync();

// Migrate: Apply pending migrations
// สร้าง migration history
// รองรับ incremental updates
// เหมาะสำหรับ production
await context.Database.MigrateAsync();
```

### Apply Migrations ใน Program.cs

```csharp
// Program.cs
var app = builder.Build();

// Apply migrations on startup
using (var scope = app.Services.CreateScope())
{
    var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();
    
    try
    {
        logger.LogInformation("Applying database migrations...");
        await context.Database.MigrateAsync();
        logger.LogInformation("Database migrations applied successfully");
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "An error occurred while applying migrations");
        throw;
    }
}

app.Run();
```

### ตรวจสอบ Pending Migrations

```csharp
// ตรวจสอบว่ามี pending migrations หรือไม่
var pendingMigrations = await context.Database.GetPendingMigrationsAsync();
if (pendingMigrations.Any())
{
    Console.WriteLine("Pending migrations:");
    foreach (var migration in pendingMigrations)
        Console.WriteLine($"  - {migration}");
}

// ดู applied migrations
var appliedMigrations = await context.Database.GetAppliedMigrationsAsync();
Console.WriteLine($"Applied migrations: {appliedMigrations.Count()}");

// ตรวจสอบว่า database มีอยู่แล้วหรือไม่
bool exists = await context.Database.CanConnectAsync();
```

---

## โปรแกรมตัวอย่าง: Schema Evolution

มาดู workflow ของการเพิ่ม feature โดยใช้ Migrations:

### Version 1: Schema เริ่มต้น

```csharp
// v1: Models/Customer.cs
public class Customer
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty;  // v1 ใช้ FullName
    public string Email { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

```bash
# สร้าง migration v1
dotnet ef migrations add InitialCreate_v1
dotnet ef database update
```

```csharp
// Migration: 20241001_InitialCreate_v1.cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.CreateTable(
        name: "Customers",
        columns: table => new
        {
            Id = table.Column<int>(nullable: false)
                .Annotation("Sqlite:Autoincrement", true),
            FullName = table.Column<string>(maxLength: 200, nullable: false),
            Email = table.Column<string>(maxLength: 200, nullable: false),
            CreatedAt = table.Column<DateTime>(nullable: false)
        },
        constraints: table => table.PrimaryKey("PK_Customers", x => x.Id));
    
    migrationBuilder.CreateIndex(
        name: "IX_Customers_Email",
        table: "Customers",
        column: "Email",
        unique: true);
}
```

### Version 2: แยก FullName เป็น FirstName + LastName

```csharp
// v2: Models/Customer.cs
public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;  // เปลี่ยน
    public string LastName { get; set; } = string.Empty;   // เพิ่ม
    public string Email { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

```bash
dotnet ef migrations add SplitFullName_v2
```

```csharp
// Migration: 20241002_SplitFullName_v2.cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    // เพิ่ม columns ใหม่
    migrationBuilder.AddColumn<string>(
        name: "FirstName", table: "Customers",
        maxLength: 100, nullable: false, defaultValue: "");
    
    migrationBuilder.AddColumn<string>(
        name: "LastName", table: "Customers",
        maxLength: 100, nullable: false, defaultValue: "");
    
    // Migrate ข้อมูล
    migrationBuilder.Sql(@"
        UPDATE Customers 
        SET FirstName = CASE 
                WHEN instr(FullName, ' ') > 0 
                THEN substr(FullName, 1, instr(FullName, ' ') - 1)
                ELSE FullName END,
            LastName = CASE 
                WHEN instr(FullName, ' ') > 0 
                THEN substr(FullName, instr(FullName, ' ') + 1)
                ELSE '' END");
    
    // ลบ column เก่า
    migrationBuilder.DropColumn(name: "FullName", table: "Customers");
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(
        name: "FullName", table: "Customers",
        maxLength: 200, nullable: false, defaultValue: "");
    
    migrationBuilder.Sql(
        "UPDATE Customers SET FullName = FirstName || ' ' || LastName");
    
    migrationBuilder.DropColumn(name: "FirstName", table: "Customers");
    migrationBuilder.DropColumn(name: "LastName", table: "Customers");
}
```

### Version 3: เพิ่ม Phone และ Address

```csharp
// v3: Models/Customer.cs
public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }  // เพิ่ม
    public string? Address { get; set; }       // เพิ่ม
    public CustomerTier Tier { get; set; } = CustomerTier.Regular;  // เพิ่ม
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

public enum CustomerTier { Regular, Silver, Gold, Platinum }
```

```bash
dotnet ef migrations add AddContactInfo_v3
dotnet ef database update
```

```csharp
// Migration: 20241003_AddContactInfo_v3.cs
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<string>(
        name: "PhoneNumber", table: "Customers",
        maxLength: 20, nullable: true);
    
    migrationBuilder.AddColumn<string>(
        name: "Address", table: "Customers",
        maxLength: 500, nullable: true);
    
    migrationBuilder.AddColumn<int>(
        name: "Tier", table: "Customers",
        nullable: false, defaultValue: 0);
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.DropColumn(name: "PhoneNumber", table: "Customers");
    migrationBuilder.DropColumn(name: "Address", table: "Customers");
    migrationBuilder.DropColumn(name: "Tier", table: "Customers");
}
```

### Version 4: เพิ่ม Orders Table

```csharp
// v4: Models/Order.cs
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
}

public enum OrderStatus { Pending, Confirmed, Shipped, Delivered, Cancelled }
```

```bash
dotnet ef migrations add AddOrdersTable_v4
dotnet ef database update
```

### Program.cs สำหรับทดสอบ

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;

// DbContext ที่ใช้ทดสอบ
public class SchemaEvolutionDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlite("Data Source=evolution.db")
               .LogTo(Console.WriteLine, LogLevel.Warning);
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Customer>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.FirstName).IsRequired().HasMaxLength(100);
            e.Property(c => c.LastName).IsRequired().HasMaxLength(100);
            e.Property(c => c.Email).IsRequired().HasMaxLength(200);
            e.HasIndex(c => c.Email).IsUnique();
        });
        
        modelBuilder.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.Property(o => o.Total).HasColumnType("decimal(18,2)");
            
            e.HasOne(o => o.Customer)
             .WithMany()
             .HasForeignKey(o => o.CustomerId);
        });
        
        // Seed data
        modelBuilder.Entity<Customer>().HasData(
            new Customer 
            { 
                Id = 1, 
                FirstName = "สมชาย", 
                LastName = "ใจดี",
                Email = "somchai@test.com",
                Tier = CustomerTier.Gold
            },
            new Customer 
            { 
                Id = 2, 
                FirstName = "สมหญิง", 
                LastName = "รักดี",
                Email = "somying@test.com" 
            }
        );
    }
}

// Main program
Console.WriteLine("=== Schema Evolution Demo ===\n");

await using var context = new SchemaEvolutionDbContext();

// Apply migrations
Console.WriteLine("Checking pending migrations...");
var pending = await context.Database.GetPendingMigrationsAsync();
Console.WriteLine($"Pending migrations: {pending.Count()}");

Console.WriteLine("\nApplying migrations...");
await context.Database.MigrateAsync();
Console.WriteLine("Migrations applied!");

// Show applied migrations
var applied = await context.Database.GetAppliedMigrationsAsync();
Console.WriteLine($"\nApplied migrations: {applied.Count()}");
foreach (var migration in applied)
    Console.WriteLine($"  ✓ {migration}");

// Use the migrated database
var customers = await context.Customers.ToListAsync();
Console.WriteLine($"\nCustomers in database: {customers.Count}");
foreach (var c in customers)
    Console.WriteLine($"  - {c.FirstName} {c.LastName} [{c.Tier}]");

// สร้าง Order
var order = new Order
{
    OrderNumber = "ORD-20241001-001",
    CustomerId = customers[0].Id,
    Total = 2500m,
    Status = OrderStatus.Confirmed
};
context.Orders.Add(order);
await context.SaveChangesAsync();
Console.WriteLine($"\nCreated order: {order.OrderNumber}");

// Query ข้อมูล
var customerWithOrders = await context.Customers
    .Include(c => context.Orders.Where(o => o.CustomerId == c.Id))
    .FirstOrDefaultAsync(c => c.Id == customers[0].Id);

Console.WriteLine("\n=== Done! ===");
```

### Rollback Demo

```csharp
// ทดสอบ rollback
Console.WriteLine("\n=== Rollback Demo ===");

// ดู migrations ปัจจุบัน
var currentMigrations = await context.Database.GetAppliedMigrationsAsync();
Console.WriteLine($"Current migrations: {currentMigrations.Count()}");

// ใน command line:
// dotnet ef database update AddContactInfo_v3  <- roll back ถึง v3
// dotnet ef database update InitialCreate_v1   <- roll back ถึง v1
// dotnet ef database update 0                  <- roll back ทั้งหมด
```

---

## Best Practices สำหรับ Migrations

### 1. ตั้งชื่อที่สื่อความหมาย

```bash
# ดี
dotnet ef migrations add AddEmailIndexToCustomers
dotnet ef migrations add CreateOrdersAndOrderItemsTables
dotnet ef migrations add MigrateFullNameToFirstLastName

# ไม่ดี
dotnet ef migrations add Update1
dotnet ef migrations add Fix
dotnet ef migrations add Changes
```

### 2. แยก migrations ตาม concern

```bash
# แยก table creation กับ data migration
dotnet ef migrations add CreateNewSchemaStructure
dotnet ef migrations add MigrateDataToNewSchema
dotnet ef migrations add RemoveOldColumns
```

### 3. Test migrations ก่อน deploy

```bash
# สร้าง SQL script ก่อน apply
dotnet ef migrations script --idempotent --output migration.sql

# Review script แล้ว apply กับ database
dotnet ef database update
```

### 4. อย่าแก้ migration ที่ apply แล้ว

ถ้าต้องการเปลี่ยนแปลง ให้สร้าง migration ใหม่

### 5. Backup ก่อน Apply ใน Production

```bash
# Backup database ก่อน
pg_dump mydb > backup.sql  # PostgreSQL
sqlcmd -S server -Q "BACKUP DATABASE mydb TO DISK='backup.bak'"  # SQL Server

# Apply migration
dotnet ef database update
```

---

## Exercises

### แบบฝึกหัดที่ 1: Migration Workflow
สร้าง project ใหม่แล้วทำ:
1. สร้าง `Student` entity กับ migration แรก
2. เพิ่ม `GradeLevel` field ใน migration ที่สอง
3. สร้าง `Course` entity และ many-to-many กับ `Student` ใน migration ที่สาม
4. Seed ข้อมูลตัวอย่าง

### แบบฝึกหัดที่ 2: Data Migration
มีข้อมูล `Address` เป็น string เดียวอยู่ใน `Customer`:
```
"123 Main St, Bangkok, 10110"
```
เขียน migration ที่แยกเป็น Street, City, PostalCode columns

### แบบฝึกหัดที่ 3: Rollback Safety
เขียน migration ที่เพิ่ม `IsVip` column พร้อม:
- `Up()`: เพิ่ม column, set ค่า VIP สำหรับ customers ที่สั่งซื้อมากกว่า 10 ครั้ง
- `Down()`: ลบ column

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Migrations** คือ version control สำหรับ database schema
2. **Add-Migration** สร้างไฟล์ migration จาก model changes
3. **Update-Database** apply migrations ไปยัง database
4. **Remove-Migration** ลบ migration ล่าสุดที่ยังไม่ apply
5. **Migration History** เก็บใน `__EFMigrationsHistory` table
6. **Seeding Data** ทั้ง HasData และ custom seeder
7. **Apply in Code** ด้วย `MigrateAsync()`

---

## Part ถัดไป

ใน **Part 056** เราจะเรียนรู้เกี่ยวกับ **EF Core: Performance** ที่สำคัญมากในการใช้งานจริง:
- AsNoTracking
- Projection
- Batch operations
- N+1 Problem
- Connection pooling

---

*Part 055/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
