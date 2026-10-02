# Part 054: EF Core: CRUD Operations

## เนื้อหาใน Part นี้
- Add, AddRange
- Update, UpdateRange
- Remove, RemoveRange
- SaveChanges และ SaveChangesAsync
- Transactions
- Optimistic Concurrency
- โปรแกรมตัวอย่าง: Customer management

---

## Change Tracking

EF Core ใช้ **Change Tracker** ติดตามการเปลี่ยนแปลงของ entities:

| State | ความหมาย |
|-------|-----------|
| `Detached` | ไม่ได้ถูก track (ยังไม่ได้เพิ่มใน context) |
| `Added` | ใหม่ รอ INSERT |
| `Unchanged` | โหลดมาจาก DB แต่ยังไม่มีการเปลี่ยนแปลง |
| `Modified` | มีการเปลี่ยนแปลง รอ UPDATE |
| `Deleted` | รอ DELETE |

```csharp
// ตรวจสอบ state
var product = new Product { Name = "Test" };
var state1 = context.Entry(product).State;  // Detached

context.Products.Add(product);
var state2 = context.Entry(product).State;  // Added

await context.SaveChangesAsync();
var state3 = context.Entry(product).State;  // Unchanged

product.Name = "Updated";
var state4 = context.Entry(product).State;  // Modified

context.Products.Remove(product);
var state5 = context.Entry(product).State;  // Deleted
```

---

## Add Operations

### เพิ่มทีละ entity

```csharp
// วิธีที่ 1: context.Entity.Add()
var customer = new Customer
{
    Name = "สมชาย ใจดี",
    Email = "somchai@example.com"
};
context.Customers.Add(customer);
await context.SaveChangesAsync();
// customer.Id ถูก set อัตโนมัติหลัง SaveChanges

// วิธีที่ 2: context.Add() - สำหรับทุก entity type
context.Add(customer);

// วิธีที่ 3: Entry().State
context.Entry(customer).State = EntityState.Added;
```

### เพิ่มหลาย entities

```csharp
// AddRange
var customers = new List<Customer>
{
    new() { Name = "สมชาย", Email = "a@test.com" },
    new() { Name = "สมหญิง", Email = "b@test.com" },
    new() { Name = "มานี", Email = "c@test.com" }
};
context.Customers.AddRange(customers);
await context.SaveChangesAsync();

// หรือด้วย params array
context.Customers.AddRange(
    new Customer { Name = "Test1", Email = "t1@test.com" },
    new Customer { Name = "Test2", Email = "t2@test.com" }
);
```

### เพิ่มพร้อม related data

```csharp
// EF Core จะ INSERT related entities ด้วย
var order = new Order
{
    OrderNumber = "ORD-001",
    CustomerId = customerId,
    Items =
    [
        new OrderItem { ProductId = 1, Quantity = 2, UnitPrice = 100m },
        new OrderItem { ProductId = 2, Quantity = 1, UnitPrice = 200m }
    ]
};
context.Orders.Add(order);  // EF Core เพิ่ม OrderItems อัตโนมัติ
await context.SaveChangesAsync();
```

---

## Update Operations

### Update tracked entity

```csharp
// 1. ดึง entity มาก่อน (tracked)
var customer = await context.Customers.FindAsync(customerId);
if (customer != null)
{
    // แก้ไข properties
    customer.Name = "ชื่อใหม่";
    customer.Email = "newemail@example.com";
    
    // EF Core รู้อัตโนมัติว่า Modified
    await context.SaveChangesAsync();  // SQL: UPDATE Customers SET ...
}
```

### Update untracked entity

```csharp
// entity ที่ได้มาจากที่อื่น (เช่น API request)
var customerFromRequest = new Customer
{
    Id = 5,  // ต้องมี PK
    Name = "Updated Name",
    Email = "updated@example.com"
};

// วิธีที่ 1: Attach + Entry().State
context.Customers.Update(customerFromRequest);
// หรือ: context.Update(customerFromRequest);
// EF Core mark ว่า Modified ทุก property
await context.SaveChangesAsync();
```

### Update บาง fields เท่านั้น

```csharp
// วิธีที่ 1: Attach แล้ว mark specific properties
var customerToUpdate = new Customer { Id = 5 };
context.Customers.Attach(customerToUpdate);  // Unchanged state

customerToUpdate.Name = "Updated Name";  // Modified state
// Email ยังเป็น Unchanged

await context.SaveChangesAsync();
// SQL: UPDATE Customers SET Name = 'Updated Name' WHERE Id = 5

// วิธีที่ 2: ใช้ Entry().Property().IsModified
var customer2 = new Customer { Id = 5, Name = "New Name", Email = "old@email.com" };
context.Customers.Attach(customer2);
context.Entry(customer2).Property(c => c.Name).IsModified = true;
// Email จะไม่ถูก update
await context.SaveChangesAsync();
```

### Bulk Update (EF Core 7+)

```csharp
// ExecuteUpdateAsync - อัปเดตหลาย rows โดยไม่ต้อง load
int rowsUpdated = await context.Products
    .Where(p => p.CategoryId == 1)
    .ExecuteUpdateAsync(setter => setter
        .SetProperty(p => p.IsActive, false)
        .SetProperty(p => p.UpdatedAt, DateTime.UtcNow));

Console.WriteLine($"Updated {rowsUpdated} products");
```

---

## Remove Operations

### ลบ tracked entity

```csharp
var customer = await context.Customers.FindAsync(customerId);
if (customer != null)
{
    context.Customers.Remove(customer);
    await context.SaveChangesAsync();
}
```

### ลบ untracked entity

```csharp
// ถ้ามีแค่ Id
var customerToDelete = new Customer { Id = customerId };
context.Customers.Remove(customerToDelete);
await context.SaveChangesAsync();
```

### RemoveRange

```csharp
// ลบหลาย entities
var oldOrders = await context.Orders
    .Where(o => o.OrderDate < DateTime.UtcNow.AddYears(-1))
    .ToListAsync();
    
context.Orders.RemoveRange(oldOrders);
await context.SaveChangesAsync();
```

### Bulk Delete (EF Core 7+)

```csharp
// ExecuteDeleteAsync - ลบโดยไม่ต้อง load entities
int rowsDeleted = await context.Orders
    .Where(o => o.Status == OrderStatus.Cancelled && 
                o.OrderDate < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync();

Console.WriteLine($"Deleted {rowsDeleted} old cancelled orders");
```

### Soft Delete Pattern

```csharp
// แทนที่จะลบจริง เปลี่ยน flag
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public bool IsDeleted { get; set; }        // Soft delete flag
    public DateTime? DeletedAt { get; set; }
}

// Global query filter - ซ่อน deleted records อัตโนมัติ
modelBuilder.Entity<Customer>()
    .HasQueryFilter(c => !c.IsDeleted);

// Soft delete method
public async Task SoftDeleteCustomerAsync(int customerId)
{
    await context.Customers
        .Where(c => c.Id == customerId)
        .ExecuteUpdateAsync(s => s
            .SetProperty(c => c.IsDeleted, true)
            .SetProperty(c => c.DeletedAt, DateTime.UtcNow));
}

// Query รวม deleted (IgnoreQueryFilters)
var allIncludingDeleted = await context.Customers
    .IgnoreQueryFilters()
    .ToListAsync();
```

---

## SaveChanges

```csharp
// Synchronous (ไม่แนะนำ - block thread)
context.SaveChanges();

// Asynchronous (แนะนำ)
await context.SaveChangesAsync();

// ดักจับ errors
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Conflict กับ concurrent update
    Console.WriteLine("Concurrency conflict: " + ex.Message);
}
catch (DbUpdateException ex)
{
    // Database error (FK violation, unique constraint, etc.)
    Console.WriteLine("DB error: " + ex.InnerException?.Message);
}

// SaveChanges returns จำนวน rows ที่ affected
int rowsAffected = await context.SaveChangesAsync();
Console.WriteLine($"{rowsAffected} rows saved");
```

### Intercepting SaveChanges

```csharp
// Override SaveChangesAsync เพื่อ auto-set timestamps
public class AppDbContext : DbContext
{
    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        var entries = ChangeTracker.Entries()
            .Where(e => e.State == EntityState.Added || 
                        e.State == EntityState.Modified);
        
        foreach (var entry in entries)
        {
            if (entry.Entity is IAuditableEntity auditable)
            {
                if (entry.State == EntityState.Added)
                    auditable.CreatedAt = DateTime.UtcNow;
                    
                auditable.UpdatedAt = DateTime.UtcNow;
            }
        }
        
        return await base.SaveChangesAsync(cancellationToken);
    }
}

public interface IAuditableEntity
{
    DateTime CreatedAt { get; set; }
    DateTime UpdatedAt { get; set; }
}
```

---

## Transactions

### Default Transaction

EF Core wrap ทุก `SaveChanges()` call ใน transaction อัตโนมัติ

```csharp
// สอง operations นี้อยู่ใน transaction เดียวกัน
context.Customers.Add(newCustomer);
context.Orders.Add(newOrder);
await context.SaveChangesAsync();  // COMMIT ทั้งคู่หรือ ROLLBACK ทั้งคู่
```

### Explicit Transaction

```csharp
// เมื่อต้องการ control transaction เอง
await using var transaction = await context.Database.BeginTransactionAsync();
try
{
    // Operation 1
    var customer = new Customer { Name = "สมชาย", Email = "s@test.com" };
    context.Customers.Add(customer);
    await context.SaveChangesAsync();
    
    // Operation 2 (ใช้ Id ที่เพิ่งสร้าง)
    var order = new Order 
    { 
        CustomerId = customer.Id, 
        OrderNumber = "ORD-001" 
    };
    context.Orders.Add(order);
    await context.SaveChangesAsync();
    
    // ถ้าทุกอย่างผ่าน -> COMMIT
    await transaction.CommitAsync();
    Console.WriteLine("Transaction committed successfully");
}
catch (Exception ex)
{
    // ถ้ามี error -> ROLLBACK
    await transaction.RollbackAsync();
    Console.WriteLine($"Transaction rolled back: {ex.Message}");
    throw;
}
```

### Savepoints (EF Core 5+)

```csharp
await using var transaction = await context.Database.BeginTransactionAsync();
try
{
    // Phase 1
    context.Customers.Add(customer1);
    await context.SaveChangesAsync();
    
    // Create savepoint
    await transaction.CreateSavepointAsync("AfterCustomer");
    
    try
    {
        // Phase 2 - อาจ fail
        context.Orders.Add(order1);
        await context.SaveChangesAsync();
    }
    catch
    {
        // Rollback แค่ phase 2 ไม่ใช่ทั้งหมด
        await transaction.RollbackToSavepointAsync("AfterCustomer");
    }
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
}
```

### Multiple DbContexts ใน Transaction เดียว

```csharp
// ใช้ DbContext อื่นใน transaction เดียวกัน
await using var transaction = await context1.Database.BeginTransactionAsync();

context1.Products.Add(product);
await context1.SaveChangesAsync();

// context2 join transaction เดียวกัน
await context2.Database.UseTransactionAsync(transaction.GetDbTransaction());
context2.Logs.Add(log);
await context2.SaveChangesAsync();

await transaction.CommitAsync();
```

---

## Optimistic Concurrency

Optimistic Concurrency ป้องกันการ overwrite ข้อมูลจาก concurrent users

### RowVersion (Timestamp)

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    
    // Concurrency token - byte[] สำหรับ SQL Server
    [Timestamp]
    public byte[]? RowVersion { get; set; }
}

// หรือด้วย Fluent API
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion();
```

### ConcurrencyCheck Attribute

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    [ConcurrencyCheck]
    public decimal Price { get; set; }  // ตรวจสอบว่า Price ไม่เปลี่ยน
}

// Fluent API
modelBuilder.Entity<Product>()
    .Property(p => p.Price)
    .IsConcurrencyToken();
```

### จัดการ Concurrency Conflict

```csharp
public async Task UpdateProductPriceAsync(int productId, decimal newPrice)
{
    const int maxRetries = 3;
    var retries = 0;
    
    while (retries < maxRetries)
    {
        try
        {
            var product = await context.Products.FindAsync(productId);
            if (product == null) throw new NotFoundException($"Product {productId} not found");
            
            product.Price = newPrice;
            await context.SaveChangesAsync();
            
            Console.WriteLine($"Price updated successfully to {newPrice}");
            return;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            retries++;
            
            if (retries >= maxRetries)
                throw new Exception($"Failed after {maxRetries} retries", ex);
            
            // Reload current values
            foreach (var entry in ex.Entries)
            {
                var dbValues = await entry.GetDatabaseValuesAsync();
                
                if (dbValues == null)
                {
                    // Record ถูกลบไปแล้ว
                    throw new NotFoundException("Record was deleted by another user");
                }
                
                // Strategy: Client Wins (ใช้ค่าของ client)
                // entry.OriginalValues.SetValues(dbValues);
                
                // Strategy: Database Wins (ใช้ค่าจาก database)
                entry.CurrentValues.SetValues(dbValues);
                entry.OriginalValues.SetValues(dbValues);
                
                Console.WriteLine($"Retry {retries}: Conflict detected, retrying...");
            }
        }
    }
}
```

---

## โปรแกรมตัวอย่าง: Customer Management

```csharp
// Models/Customer.cs
namespace CustomerApp.Models;

public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public CustomerStatus Status { get; set; } = CustomerStatus.Active;
    public decimal TotalPurchases { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    public bool IsDeleted { get; set; }
    
    [Timestamp]
    public byte[]? RowVersion { get; set; }
    
    public List<CustomerNote> Notes { get; set; } = [];
    
    public string FullName => $"{FirstName} {LastName}";
}

public enum CustomerStatus { Active, Inactive, Blocked, VIP }

public class CustomerNote
{
    public int Id { get; set; }
    public string Content { get; set; } = string.Empty;
    public string AuthorName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
}
```

```csharp
// Data/CustomerDbContext.cs
using Microsoft.EntityFrameworkCore;
using CustomerApp.Models;
using System.ComponentModel.DataAnnotations;

namespace CustomerApp.Data;

public class CustomerDbContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<CustomerNote> CustomerNotes { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlite("Data Source=customers.db");
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
            e.Property(c => c.TotalPurchases).HasColumnType("decimal(18,2)");
            e.Property(c => c.RowVersion).IsRowVersion();
            e.HasQueryFilter(c => !c.IsDeleted);
            e.Ignore(c => c.FullName);
        });
        
        modelBuilder.Entity<CustomerNote>(e =>
        {
            e.HasKey(n => n.Id);
            e.Property(n => n.Content).IsRequired();
            e.Property(n => n.AuthorName).IsRequired().HasMaxLength(100);
            
            e.HasOne(n => n.Customer)
             .WithMany(c => c.Notes)
             .HasForeignKey(n => n.CustomerId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
    
    public override async Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        // Auto-update UpdatedAt
        foreach (var entry in ChangeTracker.Entries<Customer>())
        {
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

```csharp
// Services/CustomerService.cs
using Microsoft.EntityFrameworkCore;
using CustomerApp.Data;
using CustomerApp.Models;

namespace CustomerApp.Services;

public class CustomerService
{
    private readonly CustomerDbContext _context;
    
    public CustomerService(CustomerDbContext context)
    {
        _context = context;
    }
    
    // CREATE
    public async Task<Customer> CreateCustomerAsync(
        string firstName,
        string lastName,
        string email,
        string? phone = null)
    {
        // Check duplicate email
        if (await _context.Customers.AnyAsync(c => c.Email == email))
            throw new InvalidOperationException($"Email '{email}' already exists");
        
        var customer = new Customer
        {
            FirstName = firstName,
            LastName = lastName,
            Email = email,
            PhoneNumber = phone
        };
        
        _context.Customers.Add(customer);
        await _context.SaveChangesAsync();
        
        return customer;
    }
    
    // BULK CREATE
    public async Task<List<Customer>> CreateCustomersAsync(
        List<(string First, string Last, string Email)> customers)
    {
        var emails = customers.Select(c => c.Email).ToList();
        var existing = await _context.Customers
            .Where(c => emails.Contains(c.Email))
            .Select(c => c.Email)
            .ToListAsync();
        
        if (existing.Any())
            throw new InvalidOperationException(
                $"Emails already exist: {string.Join(", ", existing)}");
        
        var newCustomers = customers.Select(c => new Customer
        {
            FirstName = c.First,
            LastName = c.Last,
            Email = c.Email
        }).ToList();
        
        _context.Customers.AddRange(newCustomers);
        await _context.SaveChangesAsync();
        
        return newCustomers;
    }
    
    // READ
    public async Task<Customer?> GetCustomerByIdAsync(int id)
    {
        return await _context.Customers
            .Include(c => c.Notes.OrderByDescending(n => n.CreatedAt))
            .FirstOrDefaultAsync(c => c.Id == id);
    }
    
    public async Task<List<Customer>> GetCustomersAsync(
        CustomerStatus? status = null,
        string? searchTerm = null)
    {
        var query = _context.Customers.AsQueryable();
        
        if (status.HasValue)
            query = query.Where(c => c.Status == status);
        
        if (!string.IsNullOrWhiteSpace(searchTerm))
            query = query.Where(c => 
                c.FirstName.Contains(searchTerm) ||
                c.LastName.Contains(searchTerm) ||
                c.Email.Contains(searchTerm));
        
        return await query
            .OrderBy(c => c.LastName)
            .ThenBy(c => c.FirstName)
            .ToListAsync();
    }
    
    // UPDATE - basic
    public async Task UpdateCustomerAsync(
        int id,
        string firstName,
        string lastName,
        string? phone)
    {
        var customer = await _context.Customers.FindAsync(id)
            ?? throw new KeyNotFoundException($"Customer {id} not found");
        
        customer.FirstName = firstName;
        customer.LastName = lastName;
        customer.PhoneNumber = phone;
        
        await _context.SaveChangesAsync();
    }
    
    // UPDATE - status change
    public async Task ChangeStatusAsync(int id, CustomerStatus newStatus)
    {
        await _context.Customers
            .Where(c => c.Id == id)
            .ExecuteUpdateAsync(s => s
                .SetProperty(c => c.Status, newStatus)
                .SetProperty(c => c.UpdatedAt, DateTime.UtcNow));
    }
    
    // UPDATE with concurrency check
    public async Task UpdateCustomerWithConcurrencyAsync(
        int id,
        string firstName,
        string lastName,
        byte[] rowVersion)
    {
        var customer = await _context.Customers.FindAsync(id)
            ?? throw new KeyNotFoundException($"Customer {id} not found");
        
        // Set original RowVersion for concurrency check
        _context.Entry(customer).Property(c => c.RowVersion).OriginalValue = rowVersion;
        
        customer.FirstName = firstName;
        customer.LastName = lastName;
        
        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            throw new InvalidOperationException(
                "Customer was modified by another user. Please reload and try again.");
        }
    }
    
    // DELETE - soft delete
    public async Task DeleteCustomerAsync(int id)
    {
        var customer = await _context.Customers.FindAsync(id)
            ?? throw new KeyNotFoundException($"Customer {id} not found");
        
        customer.IsDeleted = true;
        await _context.SaveChangesAsync();
    }
    
    // DELETE - hard delete
    public async Task HardDeleteCustomerAsync(int id)
    {
        var customer = await _context.Customers
            .IgnoreQueryFilters()  // ดู deleted customers ด้วย
            .FirstOrDefaultAsync(c => c.Id == id)
            ?? throw new KeyNotFoundException($"Customer {id} not found");
        
        _context.Customers.Remove(customer);
        await _context.SaveChangesAsync();
    }
    
    // BULK DELETE
    public async Task<int> DeleteInactiveCustomersAsync(int daysInactive)
    {
        var cutoffDate = DateTime.UtcNow.AddDays(-daysInactive);
        
        return await _context.Customers
            .Where(c => c.Status == CustomerStatus.Inactive &&
                        c.UpdatedAt < cutoffDate)
            .ExecuteDeleteAsync();
    }
    
    // ADD NOTE
    public async Task AddNoteAsync(int customerId, string content, string author)
    {
        // Verify customer exists
        if (!await _context.Customers.AnyAsync(c => c.Id == customerId))
            throw new KeyNotFoundException($"Customer {customerId} not found");
        
        var note = new CustomerNote
        {
            CustomerId = customerId,
            Content = content,
            AuthorName = author
        };
        
        _context.CustomerNotes.Add(note);
        await _context.SaveChangesAsync();
    }
    
    // TRANSACTION EXAMPLE: Transfer customer data
    public async Task MergeCustomersAsync(int sourceId, int targetId)
    {
        await using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            var source = await _context.Customers
                .Include(c => c.Notes)
                .FirstOrDefaultAsync(c => c.Id == sourceId)
                ?? throw new KeyNotFoundException($"Source customer {sourceId} not found");
            
            var target = await _context.Customers
                .FirstOrDefaultAsync(c => c.Id == targetId)
                ?? throw new KeyNotFoundException($"Target customer {targetId} not found");
            
            // Transfer purchases
            target.TotalPurchases += source.TotalPurchases;
            
            // Move notes to target
            foreach (var note in source.Notes)
            {
                note.CustomerId = targetId;
                note.Content = $"[Merged from {source.FullName}] {note.Content}";
            }
            
            await _context.SaveChangesAsync();
            
            // Soft delete source
            source.IsDeleted = true;
            source.Status = CustomerStatus.Inactive;
            
            await _context.SaveChangesAsync();
            
            await transaction.CommitAsync();
            
            Console.WriteLine($"Merged {source.FullName} into {target.FullName}");
        }
        catch
        {
            await transaction.RollbackAsync();
            throw;
        }
    }
    
    // STATISTICS
    public async Task<CustomerStatsDto> GetStatsAsync()
    {
        var stats = await _context.Customers
            .GroupBy(c => c.Status)
            .Select(g => new { Status = g.Key, Count = g.Count() })
            .ToListAsync();
        
        return new CustomerStatsDto(
            Total: stats.Sum(s => s.Count),
            Active: stats.FirstOrDefault(s => s.Status == CustomerStatus.Active)?.Count ?? 0,
            VIP: stats.FirstOrDefault(s => s.Status == CustomerStatus.VIP)?.Count ?? 0,
            Inactive: stats.FirstOrDefault(s => s.Status == CustomerStatus.Inactive)?.Count ?? 0,
            Blocked: stats.FirstOrDefault(s => s.Status == CustomerStatus.Blocked)?.Count ?? 0
        );
    }
}

public record CustomerStatsDto(
    int Total,
    int Active,
    int VIP,
    int Inactive,
    int Blocked);
```

```csharp
// Program.cs
using CustomerApp.Data;
using CustomerApp.Models;
using CustomerApp.Services;
using Microsoft.EntityFrameworkCore;

Console.WriteLine("=== Customer Management System ===\n");

await using var context = new CustomerDbContext();
await context.Database.EnsureDeletedAsync();
await context.Database.EnsureCreatedAsync();

var service = new CustomerService(context);

// 1. CREATE customers
Console.WriteLine("1. Creating customers...");
var c1 = await service.CreateCustomerAsync("สมชาย", "ใจดี", "somchai@test.com", "081-111-1111");
var c2 = await service.CreateCustomerAsync("สมหญิง", "รักดี", "somying@test.com", "082-222-2222");
var c3 = await service.CreateCustomerAsync("มานี", "แสงจันทร์", "manee@test.com");
Console.WriteLine($"   Created: {c1.FullName}, {c2.FullName}, {c3.FullName}");

// 2. BULK CREATE
Console.WriteLine("\n2. Bulk creating customers...");
var bulkCustomers = await service.CreateCustomersAsync([
    ("ปีติ", "ชื่นใจ", "piti@test.com"),
    ("วิไล", "สวยงาม", "wilai@test.com"),
    ("ประสิทธิ์", "เก่งกาจ", "prasit@test.com")
]);
Console.WriteLine($"   Created {bulkCustomers.Count} customers in bulk");

// 3. READ
Console.WriteLine("\n3. Reading customers...");
var allCustomers = await service.GetCustomersAsync();
Console.WriteLine($"   Total customers: {allCustomers.Count}");

var searchResult = await service.GetCustomersAsync(searchTerm: "สม");
Console.WriteLine($"   Search 'สม': {searchResult.Count} found");

// 4. UPDATE
Console.WriteLine("\n4. Updating customer...");
await service.UpdateCustomerAsync(c1.Id, "สมชาย", "ใจดีมาก", "081-999-9999");
var updated = await service.GetCustomerByIdAsync(c1.Id);
Console.WriteLine($"   Updated: {updated?.FullName}, Phone: {updated?.PhoneNumber}");

// 5. CHANGE STATUS
Console.WriteLine("\n5. Changing status to VIP...");
await service.ChangeStatusAsync(c2.Id, CustomerStatus.VIP);
var vipCustomers = await service.GetCustomersAsync(status: CustomerStatus.VIP);
Console.WriteLine($"   VIP customers: {vipCustomers.Count}");

// 6. ADD NOTES
Console.WriteLine("\n6. Adding notes...");
await service.AddNoteAsync(c1.Id, "ลูกค้าประจำ ซื้อสินค้าทุกเดือน", "Admin");
await service.AddNoteAsync(c1.Id, "ชอบสินค้า Premium", "Sales");
var customerWithNotes = await service.GetCustomerByIdAsync(c1.Id);
Console.WriteLine($"   Notes for {customerWithNotes?.FullName}: {customerWithNotes?.Notes.Count} notes");

// 7. SOFT DELETE
Console.WriteLine("\n7. Soft deleting customer...");
await service.DeleteCustomerAsync(c3.Id);
var afterDelete = await service.GetCustomersAsync();
Console.WriteLine($"   Active customers after delete: {afterDelete.Count}");

// 8. TRANSACTION - Merge
Console.WriteLine("\n8. Merging customers...");
await service.MergeCustomersAsync(bulkCustomers[0].Id, bulkCustomers[1].Id);
var afterMerge = await service.GetCustomerByIdAsync(bulkCustomers[1].Id);
Console.WriteLine($"   {afterMerge?.FullName} notes: {afterMerge?.Notes.Count}");

// 9. STATISTICS
Console.WriteLine("\n9. Statistics:");
var stats = await service.GetStatsAsync();
Console.WriteLine($"   Total: {stats.Total}");
Console.WriteLine($"   Active: {stats.Active}");
Console.WriteLine($"   VIP: {stats.VIP}");
Console.WriteLine($"   Inactive: {stats.Inactive}");

// 10. DUPLICATE EMAIL check
Console.WriteLine("\n10. Testing duplicate email...");
try
{
    await service.CreateCustomerAsync("ทดสอบ", "ซ้ำ", "somchai@test.com");
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"   Expected error: {ex.Message}");
}

Console.WriteLine("\n=== Done! ===");
```

---

## Exercises

### แบบฝึกหัดที่ 1: Audit Log
สร้าง `AuditLog` entity และ interceptor ที่บันทึกทุก Insert, Update, Delete:

```csharp
public class AuditLog
{
    public int Id { get; set; }
    public string EntityName { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;  // "Created", "Updated", "Deleted"
    public string? OldValues { get; set; }   // JSON
    public string? NewValues { get; set; }   // JSON
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    public string? UserId { get; set; }
}
```

### แบบฝึกหัดที่ 2: Batch Operations
เขียน method ที่:
1. Import customers จาก CSV (List&lt;string[]&gt;)
2. ถ้า email ซ้ำ ให้ update แทน insert (Upsert)
3. Return สรุปว่า inserted กี่ rows, updated กี่ rows

### แบบฝึกหัดที่ 3: Concurrency Test
เขียน test ที่จำลอง concurrent updates:
1. ดึง customer มา 2 instances
2. ทั้งคู่พยายาม update phone number พร้อมกัน
3. จัดการ DbUpdateConcurrencyException ให้ถูกต้อง

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Change Tracker**: EF Core ติดตาม state ของ entities
2. **Add/AddRange**: เพิ่ม entities
3. **Update**: แก้ไข tracked หรือ untracked entities
4. **ExecuteUpdateAsync**: Bulk update ไม่ต้อง load entities
5. **Remove/RemoveRange**: ลบ entities
6. **ExecuteDeleteAsync**: Bulk delete ไม่ต้อง load entities
7. **Soft Delete**: ใช้ IsDeleted flag แทนการลบจริง
8. **Transactions**: จัดการ multi-step operations อย่างปลอดภัย
9. **Optimistic Concurrency**: ป้องกัน lost updates

---

## Part ถัดไป

ใน **Part 055** เราจะเรียนรู้เกี่ยวกับ **EF Core: Migrations** อย่างละเอียด:
- Add-Migration และ Update-Database
- Remove-Migration
- Migration history
- Seeding data
- Applying migrations ใน code

---

*Part 054/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
