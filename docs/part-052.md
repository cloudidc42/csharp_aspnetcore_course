# Part 052: EF Core: Entities และ Relationships

## เนื้อหาใน Part นี้
- Entity configuration แบบต่างๆ
- Primary keys รูปแบบต่างๆ
- One-to-Many relationships
- Many-to-Many relationships
- One-to-One relationships
- Owned entities
- โปรแกรมตัวอย่าง: E-commerce schema

---

## Entity Configuration

มี 2 วิธีหลักในการ configure entities ใน EF Core:

### 1. Data Annotations (Attributes)

ใช้ attribute ตรงบน class หรือ property

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

[Table("Products")]
public class Product
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    
    [Required]
    [MaxLength(200)]
    [Column("ProductName")]
    public string Name { get; set; } = string.Empty;
    
    [Required]
    [Column(TypeName = "decimal(18,2)")]
    public decimal Price { get; set; }
    
    [MaxLength(500)]
    public string? Description { get; set; }
    
    [NotMapped]  // property นี้จะไม่ถูก map กับ column ใน database
    public string DisplayName => $"{Name} (${Price})";
    
    [Timestamp]  // สำหรับ concurrency control
    public byte[]? RowVersion { get; set; }
}
```

### 2. Fluent API (ใน OnModelCreating)

กำหนด configuration ใน `OnModelCreating` method ของ DbContext

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Product>(entity =>
    {
        // Table name
        entity.ToTable("Products");
        
        // Primary key
        entity.HasKey(p => p.Id);
        
        // Properties
        entity.Property(p => p.Name)
              .IsRequired()
              .HasMaxLength(200)
              .HasColumnName("ProductName");
              
        entity.Property(p => p.Price)
              .IsRequired()
              .HasColumnType("decimal(18,2)");
              
        entity.Property(p => p.Description)
              .HasMaxLength(500);
              
        // ไม่ map property นี้กับ database
        entity.Ignore(p => p.DisplayName);
        
        // Index
        entity.HasIndex(p => p.Name).IsUnique();
        
        // RowVersion สำหรับ concurrency
        entity.Property(p => p.RowVersion)
              .IsRowVersion();
    });
}
```

### IEntityTypeConfiguration&lt;T&gt; Pattern

แยก configuration ออกเป็นไฟล์แยก:

```csharp
// Data/Configurations/ProductConfiguration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");
        builder.HasKey(p => p.Id);
        
        builder.Property(p => p.Name)
               .IsRequired()
               .HasMaxLength(200);
               
        builder.Property(p => p.Price)
               .HasColumnType("decimal(18,2)");
               
        builder.HasIndex(p => p.Name).IsUnique();
    }
}

// DbContext - ใช้ ApplyConfigurationsFromAssembly
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Apply configuration จาก assembly อัตโนมัติ
    modelBuilder.ApplyConfigurationsFromAssembly(
        typeof(BlogDbContext).Assembly);
}
```

---

## Primary Keys

### Auto-increment Integer (ค่าเริ่มต้น)

```csharp
public class Product
{
    public int Id { get; set; }  // EF Core รู้โดยอัตโนมัติว่านี่คือ PK
    // หรือชื่อ ProductId ก็ได้
}
```

### GUID Primary Key

```csharp
public class Order
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public string OrderNumber { get; set; } = string.Empty;
}

// Configuration
modelBuilder.Entity<Order>(e =>
{
    e.HasKey(o => o.Id);
    e.Property(o => o.Id).ValueGeneratedOnAdd();
});
```

### Composite Primary Key

```csharp
public class OrderItem
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

// Composite PK ต้องกำหนดด้วย Fluent API เท่านั้น
modelBuilder.Entity<OrderItem>(e =>
{
    e.HasKey(oi => new { oi.OrderId, oi.ProductId });
});
```

### String Primary Key

```csharp
public class Country
{
    [Key]
    [MaxLength(2)]
    public string Code { get; set; } = string.Empty;  // "TH", "US", etc.
    public string Name { get; set; } = string.Empty;
}
```

### Custom Value Generation

```csharp
public class Product
{
    public int Id { get; set; }
    public string Sku { get; set; } = string.Empty;  // "PRD-00001"
}

modelBuilder.Entity<Product>(e =>
{
    // DatabaseGenerated.Identity: ให้ DB สร้างค่า
    e.Property(p => p.Id)
     .ValueGeneratedOnAdd();
     
    // DatabaseGenerated.Computed: คำนวณจาก DB
    // e.Property(p => p.Sku).ValueGeneratedOnAddOrUpdate();
    
    // ไม่ให้ DB สร้างค่า (ต้อง set เอง)
    // e.Property(p => p.Id).ValueGeneratedNever();
});
```

---

## One-to-Many Relationships

หนึ่ง Blog มีได้หลาย Post

```csharp
// ฝั่ง "One"
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    // Navigation property - collection
    public List<Post> Posts { get; set; } = [];
}

// ฝั่ง "Many"
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    
    // Foreign Key
    public int BlogId { get; set; }
    
    // Navigation property - reference
    public Blog Blog { get; set; } = null!;
}
```

### Configuration

```csharp
modelBuilder.Entity<Post>(e =>
{
    // HasOne: Post มี Blog หนึ่งอัน
    // WithMany: Blog มีได้หลาย Posts
    // HasForeignKey: FK คือ BlogId
    e.HasOne(p => p.Blog)
     .WithMany(b => b.Posts)
     .HasForeignKey(p => p.BlogId)
     .OnDelete(DeleteBehavior.Cascade);  // ลบ Blog -> ลบ Posts ด้วย
});
```

### Delete Behaviors

```csharp
// Cascade: ลบ parent -> ลบ children ด้วย (default)
.OnDelete(DeleteBehavior.Cascade)

// SetNull: ลบ parent -> set FK ของ children เป็น null
.OnDelete(DeleteBehavior.SetNull)

// Restrict: ไม่ยอมลบ parent ถ้ามี children อยู่
.OnDelete(DeleteBehavior.Restrict)

// NoAction: ไม่ทำอะไร (อาจ error ถ้า DB enforce FK)
.OnDelete(DeleteBehavior.NoAction)
```

### Optional Relationship (Nullable FK)

```csharp
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    
    // Optional: post ไม่จำเป็นต้องมี category
    public int? CategoryId { get; set; }
    public Category? Category { get; set; }
}

modelBuilder.Entity<Post>(e =>
{
    e.HasOne(p => p.Category)
     .WithMany(c => c.Posts)
     .HasForeignKey(p => p.CategoryId)
     .IsRequired(false);  // Optional relationship
});
```

---

## Many-to-Many Relationships

Product กับ Category (product อยู่ได้หลาย category, category มีได้หลาย product)

### EF Core 5+ (Implicit Join Table)

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    public List<Category> Categories { get; set; } = [];
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    public List<Product> Products { get; set; } = [];
}

// EF Core สร้าง join table อัตโนมัติ (ชื่อ CategoryProduct)
modelBuilder.Entity<Product>()
    .HasMany(p => p.Categories)
    .WithMany(c => c.Products);
```

### Explicit Join Entity (มี extra properties)

เมื่อ join table มี extra fields เช่น วันที่เพิ่ม:

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    public List<ProductCategory> ProductCategories { get; set; } = [];
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    public List<ProductCategory> ProductCategories { get; set; } = [];
}

// Join entity
public class ProductCategory
{
    public int ProductId { get; set; }
    public int CategoryId { get; set; }
    public DateTime AddedAt { get; set; } = DateTime.UtcNow;
    public string AddedBy { get; set; } = string.Empty;
    
    // Navigation properties
    public Product Product { get; set; } = null!;
    public Category Category { get; set; } = null!;
}

// Configuration
modelBuilder.Entity<ProductCategory>(e =>
{
    e.HasKey(pc => new { pc.ProductId, pc.CategoryId });
    
    e.HasOne(pc => pc.Product)
     .WithMany(p => p.ProductCategories)
     .HasForeignKey(pc => pc.ProductId);
     
    e.HasOne(pc => pc.Category)
     .WithMany(c => c.ProductCategories)
     .HasForeignKey(pc => pc.CategoryId);
});
```

---

## One-to-One Relationships

User กับ UserProfile

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    
    // Navigation property
    public UserProfile? Profile { get; set; }
}

public class UserProfile
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public DateTime? DateOfBirth { get; set; }
    public string? AvatarUrl { get; set; }
    
    // FK to User
    public int UserId { get; set; }
    public User User { get; set; } = null!;
}

// Configuration
modelBuilder.Entity<UserProfile>(e =>
{
    e.HasKey(up => up.Id);
    
    e.HasOne(up => up.User)
     .WithOne(u => u.Profile)
     .HasForeignKey<UserProfile>(up => up.UserId)
     .OnDelete(DeleteBehavior.Cascade);
});
```

### Table Splitting (One entity แต่ 2 tables)

```csharp
// ฝั่ง Principal
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public decimal Total { get; set; }
    
    public OrderDetails? Details { get; set; }
}

// ฝั่ง Dependent
public class OrderDetails
{
    public int Id { get; set; }
    public string ShippingAddress { get; set; } = string.Empty;
    public string BillingAddress { get; set; } = string.Empty;
    public string Notes { get; set; } = string.Empty;
}

modelBuilder.Entity<Order>()
    .HasOne(o => o.Details)
    .WithOne()
    .HasForeignKey<OrderDetails>(d => d.Id);
```

---

## Owned Entities

Owned entity คือ entity ที่ถูก "เป็นเจ้าของ" โดย entity อื่น จะถูก store ใน table เดียวกันกับ owner

```csharp
// Value Object / Owned Type
public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string Province { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    public string Country { get; set; } = "Thailand";
}

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    
    // Owned entity - จะถูก store ใน Customers table
    public Address ShippingAddress { get; set; } = new();
    public Address BillingAddress { get; set; } = new();
}

// Configuration
modelBuilder.Entity<Customer>(e =>
{
    e.OwnsOne(c => c.ShippingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("ShippingStreet");
        addr.Property(a => a.City).HasColumnName("ShippingCity");
        addr.Property(a => a.Province).HasColumnName("ShippingProvince");
        addr.Property(a => a.PostalCode).HasColumnName("ShippingPostalCode");
        addr.Property(a => a.Country).HasColumnName("ShippingCountry");
    });
    
    e.OwnsOne(c => c.BillingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("BillingStreet");
        addr.Property(a => a.City).HasColumnName("BillingCity");
        addr.Property(a => a.Province).HasColumnName("BillingProvince");
        addr.Property(a => a.PostalCode).HasColumnName("BillingPostalCode");
        addr.Property(a => a.Country).HasColumnName("BillingCountry");
    });
});
```

### Owned Collection (EF Core 7+)

```csharp
public class BlogPost
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    
    // Collection ของ owned entities
    public List<Revision> Revisions { get; set; } = [];
}

public class Revision
{
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public string Author { get; set; } = string.Empty;
}

modelBuilder.Entity<BlogPost>()
    .OwnsMany(p => p.Revisions, rev =>
    {
        rev.ToTable("PostRevisions");
        rev.WithOwner().HasForeignKey("PostId");
    });
```

---

## โปรแกรมตัวอย่าง: E-commerce Schema

### Models

```csharp
// Models/Category.cs
namespace ECommerceApp.Models;

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    public string? Description { get; set; }
    public int? ParentCategoryId { get; set; }
    
    // Self-referential relationship
    public Category? ParentCategory { get; set; }
    public List<Category> SubCategories { get; set; } = [];
    
    public List<ProductCategory> ProductCategories { get; set; } = [];
}
```

```csharp
// Models/Product.cs
namespace ECommerceApp.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Sku { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal BasePrice { get; set; }
    public int StockQuantity { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public List<ProductCategory> ProductCategories { get; set; } = [];
    public List<ProductImage> Images { get; set; } = [];
    public List<OrderItem> OrderItems { get; set; } = [];
}
```

```csharp
// Models/ProductImage.cs
namespace ECommerceApp.Models;

public class ProductImage
{
    public int Id { get; set; }
    public string Url { get; set; } = string.Empty;
    public string? AltText { get; set; }
    public bool IsPrimary { get; set; }
    public int SortOrder { get; set; }
    
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
}
```

```csharp
// Models/Customer.cs
namespace ECommerceApp.Models;

public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string Province { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    public string Country { get; set; } = "Thailand";
    
    public string FullAddress => 
        $"{Street}, {City}, {Province} {PostalCode}, {Country}";
}

public class Customer
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public DateTime RegisteredAt { get; set; } = DateTime.UtcNow;
    
    public Address DefaultShippingAddress { get; set; } = new();
    
    public List<Order> Orders { get; set; } = [];
    
    public string FullName => $"{FirstName} {LastName}";
}
```

```csharp
// Models/Order.cs
namespace ECommerceApp.Models;

public enum OrderStatus
{
    Pending,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public DateTime OrderDate { get; set; } = DateTime.UtcNow;
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public decimal SubTotal { get; set; }
    public decimal ShippingCost { get; set; }
    public decimal Tax { get; set; }
    public decimal Total => SubTotal + ShippingCost + Tax;
    
    // FK
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!;
    
    // Owned entity - shipping address (ณ เวลาสั่ง)
    public Address ShippingAddress { get; set; } = new();
    
    // One-to-Many
    public List<OrderItem> Items { get; set; } = [];
    
    // One-to-One
    public Payment? Payment { get; set; }
}
```

```csharp
// Models/OrderItem.cs
namespace ECommerceApp.Models;

public class OrderItem
{
    public int Id { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal Discount { get; set; }
    public decimal LineTotal => (UnitPrice - Discount) * Quantity;
    
    public int OrderId { get; set; }
    public Order Order { get; set; } = null!;
    
    public int ProductId { get; set; }
    public Product Product { get; set; } = null!;
}
```

```csharp
// Models/Payment.cs
namespace ECommerceApp.Models;

public enum PaymentMethod
{
    CreditCard,
    DebitCard,
    BankTransfer,
    PromptPay,
    COD
}

public class Payment
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public PaymentMethod Method { get; set; }
    public DateTime PaidAt { get; set; }
    public string? TransactionId { get; set; }
    public bool IsSuccessful { get; set; }
    
    public int OrderId { get; set; }
    public Order Order { get; set; } = null!;
}
```

### DbContext

```csharp
// Data/ECommerceDbContext.cs
using Microsoft.EntityFrameworkCore;
using ECommerceApp.Models;

namespace ECommerceApp.Data;

public class ECommerceDbContext : DbContext
{
    public DbSet<Category> Categories { get; set; }
    public DbSet<Product> Products { get; set; }
    public DbSet<ProductCategory> ProductCategories { get; set; }
    public DbSet<ProductImage> ProductImages { get; set; }
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    public DbSet<Payment> Payments { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlite("Data Source=ecommerce.db");
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Category - Self-referential
        modelBuilder.Entity<Category>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.Name).IsRequired().HasMaxLength(100);
            e.Property(c => c.Slug).IsRequired().HasMaxLength(100);
            e.HasIndex(c => c.Slug).IsUnique();
            
            e.HasOne(c => c.ParentCategory)
             .WithMany(c => c.SubCategories)
             .HasForeignKey(c => c.ParentCategoryId)
             .OnDelete(DeleteBehavior.Restrict);
        });
        
        // Product
        modelBuilder.Entity<Product>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Name).IsRequired().HasMaxLength(200);
            e.Property(p => p.Sku).IsRequired().HasMaxLength(50);
            e.HasIndex(p => p.Sku).IsUnique();
            e.Property(p => p.BasePrice).HasColumnType("decimal(18,2)");
        });
        
        // Product-Category Many-to-Many
        modelBuilder.Entity<ProductCategory>(e =>
        {
            e.HasKey(pc => new { pc.ProductId, pc.CategoryId });
            
            e.HasOne(pc => pc.Product)
             .WithMany(p => p.ProductCategories)
             .HasForeignKey(pc => pc.ProductId);
             
            e.HasOne(pc => pc.Category)
             .WithMany(c => c.ProductCategories)
             .HasForeignKey(pc => pc.CategoryId);
        });
        
        // ProductImage
        modelBuilder.Entity<ProductImage>(e =>
        {
            e.HasKey(pi => pi.Id);
            e.Property(pi => pi.Url).IsRequired();
            
            e.HasOne(pi => pi.Product)
             .WithMany(p => p.Images)
             .HasForeignKey(pi => pi.ProductId)
             .OnDelete(DeleteBehavior.Cascade);
        });
        
        // Customer with Owned Address
        modelBuilder.Entity<Customer>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.Email).IsRequired().HasMaxLength(200);
            e.HasIndex(c => c.Email).IsUnique();
            
            e.OwnsOne(c => c.DefaultShippingAddress, addr =>
            {
                addr.Property(a => a.Street).HasMaxLength(200);
                addr.Property(a => a.City).HasMaxLength(100);
                addr.Property(a => a.Province).HasMaxLength(100);
                addr.Property(a => a.PostalCode).HasMaxLength(10);
                addr.Property(a => a.Country).HasMaxLength(50).HasDefaultValue("Thailand");
            });
            
            e.Ignore(c => c.FullName);
        });
        
        // Order
        modelBuilder.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.Property(o => o.OrderNumber).IsRequired().HasMaxLength(50);
            e.HasIndex(o => o.OrderNumber).IsUnique();
            e.Property(o => o.SubTotal).HasColumnType("decimal(18,2)");
            e.Property(o => o.ShippingCost).HasColumnType("decimal(18,2)");
            e.Property(o => o.Tax).HasColumnType("decimal(18,2)");
            e.Ignore(o => o.Total);  // computed property
            
            e.HasOne(o => o.Customer)
             .WithMany(c => c.Orders)
             .HasForeignKey(o => o.CustomerId)
             .OnDelete(DeleteBehavior.Restrict);
             
            e.OwnsOne(o => o.ShippingAddress);
        });
        
        // OrderItem
        modelBuilder.Entity<OrderItem>(e =>
        {
            e.HasKey(oi => oi.Id);
            e.Property(oi => oi.UnitPrice).HasColumnType("decimal(18,2)");
            e.Property(oi => oi.Discount).HasColumnType("decimal(18,2)");
            e.Ignore(oi => oi.LineTotal);
            
            e.HasOne(oi => oi.Order)
             .WithMany(o => o.Items)
             .HasForeignKey(oi => oi.OrderId)
             .OnDelete(DeleteBehavior.Cascade);
             
            e.HasOne(oi => oi.Product)
             .WithMany(p => p.OrderItems)
             .HasForeignKey(oi => oi.ProductId)
             .OnDelete(DeleteBehavior.Restrict);
        });
        
        // Payment (One-to-One with Order)
        modelBuilder.Entity<Payment>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Amount).HasColumnType("decimal(18,2)");
            
            e.HasOne(p => p.Order)
             .WithOne(o => o.Payment)
             .HasForeignKey<Payment>(p => p.OrderId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

### Program.cs - สร้างข้อมูลตัวอย่าง

```csharp
// Program.cs
using ECommerceApp.Data;
using ECommerceApp.Models;
using Microsoft.EntityFrameworkCore;

Console.WriteLine("=== E-Commerce Database Setup ===\n");

await using var context = new ECommerceDbContext();
await context.Database.EnsureDeletedAsync();
await context.Database.EnsureCreatedAsync();

// 1. สร้าง Categories
Console.WriteLine("Creating categories...");
var electronics = new Category { Name = "Electronics", Slug = "electronics" };
var phones = new Category 
{ 
    Name = "Smartphones", 
    Slug = "smartphones",
    ParentCategory = electronics 
};
var laptops = new Category 
{ 
    Name = "Laptops", 
    Slug = "laptops",
    ParentCategory = electronics 
};
var clothing = new Category { Name = "Clothing", Slug = "clothing" };

context.Categories.AddRange(electronics, phones, laptops, clothing);
await context.SaveChangesAsync();

// 2. สร้าง Products
Console.WriteLine("Creating products...");
var iPhone = new Product
{
    Name = "iPhone 16 Pro",
    Sku = "APL-IP16PRO",
    BasePrice = 45900m,
    StockQuantity = 50,
    Description = "Apple iPhone 16 Pro 256GB"
};

var macbook = new Product
{
    Name = "MacBook Pro 14\"",
    Sku = "APL-MBP14",
    BasePrice = 74900m,
    StockQuantity = 20,
    Description = "Apple MacBook Pro M4"
};

// เพิ่ม images
iPhone.Images.Add(new ProductImage 
{ 
    Url = "https://example.com/iphone16-1.jpg", 
    IsPrimary = true,
    AltText = "iPhone 16 Pro - Front" 
});
iPhone.Images.Add(new ProductImage 
{ 
    Url = "https://example.com/iphone16-2.jpg", 
    AltText = "iPhone 16 Pro - Back" 
});

context.Products.AddRange(iPhone, macbook);
await context.SaveChangesAsync();

// 3. กำหนด categories ให้ products
Console.WriteLine("Assigning categories...");
context.ProductCategories.AddRange(
    new ProductCategory { ProductId = iPhone.Id, CategoryId = phones.Id },
    new ProductCategory { ProductId = iPhone.Id, CategoryId = electronics.Id },
    new ProductCategory { ProductId = macbook.Id, CategoryId = laptops.Id },
    new ProductCategory { ProductId = macbook.Id, CategoryId = electronics.Id }
);
await context.SaveChangesAsync();

// 4. สร้าง Customer
Console.WriteLine("Creating customer...");
var customer = new Customer
{
    FirstName = "สมชาย",
    LastName = "ใจดี",
    Email = "somchai@example.com",
    PhoneNumber = "081-234-5678",
    DefaultShippingAddress = new Address
    {
        Street = "123 ถ.สุขุมวิท",
        City = "กรุงเทพมหานคร",
        Province = "กรุงเทพมหานคร",
        PostalCode = "10110"
    }
};
context.Customers.Add(customer);
await context.SaveChangesAsync();

// 5. สร้าง Order
Console.WriteLine("Creating order...");
var order = new Order
{
    OrderNumber = "ORD-20241001-001",
    CustomerId = customer.Id,
    Status = OrderStatus.Confirmed,
    SubTotal = 45900m + 74900m,
    ShippingCost = 0m,
    Tax = (45900m + 74900m) * 0.07m,
    ShippingAddress = new Address
    {
        Street = customer.DefaultShippingAddress.Street,
        City = customer.DefaultShippingAddress.City,
        Province = customer.DefaultShippingAddress.Province,
        PostalCode = customer.DefaultShippingAddress.PostalCode
    },
    Items =
    [
        new OrderItem
        {
            ProductId = iPhone.Id,
            Quantity = 1,
            UnitPrice = iPhone.BasePrice,
            Discount = 0m
        },
        new OrderItem
        {
            ProductId = macbook.Id,
            Quantity = 1,
            UnitPrice = macbook.BasePrice,
            Discount = 0m
        }
    ]
};
context.Orders.Add(order);
await context.SaveChangesAsync();

// 6. เพิ่ม Payment
Console.WriteLine("Adding payment...");
var payment = new Payment
{
    OrderId = order.Id,
    Amount = order.SubTotal + order.ShippingCost + order.Tax,
    Method = PaymentMethod.PromptPay,
    PaidAt = DateTime.UtcNow,
    TransactionId = "TXN-123456789",
    IsSuccessful = true
};
context.Payments.Add(payment);
await context.SaveChangesAsync();

// 7. Query ข้อมูล
Console.WriteLine("\n=== Order Summary ===");
var orderDetails = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(oi => oi.Product)
    .Include(o => o.Payment)
    .FirstOrDefaultAsync(o => o.OrderNumber == "ORD-20241001-001");

if (orderDetails != null)
{
    Console.WriteLine($"Order: {orderDetails.OrderNumber}");
    Console.WriteLine($"Customer: {orderDetails.Customer.FullName}");
    Console.WriteLine($"Status: {orderDetails.Status}");
    Console.WriteLine($"\nItems:");
    foreach (var item in orderDetails.Items)
    {
        Console.WriteLine($"  - {item.Product.Name} x{item.Quantity} = ฿{item.LineTotal:N0}");
    }
    Console.WriteLine($"\nSubtotal: ฿{orderDetails.SubTotal:N0}");
    Console.WriteLine($"Shipping: ฿{orderDetails.ShippingCost:N0}");
    Console.WriteLine($"Tax (7%): ฿{orderDetails.Tax:N0}");
    Console.WriteLine($"Total: ฿{orderDetails.Total:N0}");
    
    if (orderDetails.Payment != null)
    {
        Console.WriteLine($"\nPayment: {orderDetails.Payment.Method} - {orderDetails.Payment.TransactionId}");
    }
    
    Console.WriteLine($"\nShip to: {orderDetails.ShippingAddress.FullAddress}");
}

// 8. Query Categories with Products
Console.WriteLine("\n=== Electronics Category ===");
var electronicsCategory = await context.Categories
    .Include(c => c.SubCategories)
    .Include(c => c.ProductCategories)
        .ThenInclude(pc => pc.Product)
    .FirstOrDefaultAsync(c => c.Slug == "electronics");

if (electronicsCategory != null)
{
    Console.WriteLine($"Category: {electronicsCategory.Name}");
    Console.WriteLine($"Sub-categories: {string.Join(", ", electronicsCategory.SubCategories.Select(s => s.Name))}");
    Console.WriteLine($"Direct products: {electronicsCategory.ProductCategories.Count}");
    foreach (var pc in electronicsCategory.ProductCategories)
    {
        Console.WriteLine($"  - {pc.Product.Name} (฿{pc.Product.BasePrice:N0})");
    }
}

Console.WriteLine("\n=== Done! ===");
```

---

## Exercises

### แบบฝึกหัดที่ 1: เพิ่ม Review System
เพิ่ม `Review` entity ที่มี:
- Customer review ให้ Product (One-to-Many)
- Rating (1-5 ดาว)
- Comment
- Verified purchase flag

```csharp
// TODO: สร้าง Review entity
public class Review
{
    // เพิ่ม properties...
}

// TODO: เพิ่มใน Product
// public List<Review> Reviews { get; set; } = [];

// TODO: Configure ใน OnModelCreating
```

### แบบฝึกหัดที่ 2: Discount System
สร้าง discount system:
- `Discount` entity (code, percentage/amount, valid dates, max uses)
- เชื่อมกับ `Order`

### แบบฝึกหัดที่ 3: Inventory Tracking
เพิ่มระบบ track inventory:
- `InventoryLog` entity บันทึกการเพิ่ม/ลดสินค้า
- แต่ละ log มี ProductId, Quantity (+/-), Reason, CreatedAt

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **Entity Configuration** ทั้ง Data Annotations และ Fluent API
2. **Primary Keys** หลายรูปแบบ: int, GUID, Composite, String
3. **One-to-Many**: ใช้ `HasOne().WithMany()`
4. **Many-to-Many**: ทั้งแบบ implicit และ explicit join table
5. **One-to-One**: ใช้ `HasOne().WithOne()`
6. **Owned Entities**: store ใน table เดียวกับ owner
7. **Delete Behaviors**: Cascade, SetNull, Restrict, NoAction

---

## Part ถัดไป

ใน **Part 053** เราจะเรียนรู้เกี่ยวกับ **EF Core: Querying** อย่างละเอียด:
- LINQ กับ EF Core
- Eager Loading ด้วย Include/ThenInclude
- Lazy Loading
- Explicit Loading
- Raw SQL queries
- การ filter, sort, paginate

---

*Part 052/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
