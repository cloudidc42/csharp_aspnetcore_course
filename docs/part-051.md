# Part 051: Entity Framework Core เบื้องต้น

## เนื้อหาใน Part นี้
- ORM คืออะไรและทำไมต้องใช้
- ติดตั้ง EF Core 9
- DbContext และการตั้งค่า
- DbSet&lt;T&gt;
- Code First vs Database First
- Migration เบื้องต้น
- โปรแกรมตัวอย่าง: Blog database

---

## ORM คืออะไร

**ORM (Object-Relational Mapper)** คือเครื่องมือที่ช่วยแปลงข้อมูลระหว่างโลกของ Object-Oriented Programming (OOP) กับโลกของ Relational Database โดยอัตโนมัติ

### ปัญหาที่ ORM แก้ไข

เมื่อเราเขียนโปรแกรม C# เราจัดการข้อมูลในรูปแบบ Object แต่ Database จัดเก็บข้อมูลในรูปแบบ Table/Row ซึ่งทำให้เกิดช่องว่างที่เรียกว่า **Object-Relational Impedance Mismatch**

**โดยไม่มี ORM:**

```csharp
// ต้องเขียน SQL เองทั้งหมด
using var connection = new SqlConnection(connectionString);
connection.Open();

var command = new SqlCommand(
    "SELECT Id, Title, Content, CreatedAt FROM Posts WHERE Id = @id", 
    connection);
command.Parameters.AddWithValue("@id", postId);

using var reader = command.ExecuteReader();
if (reader.Read())
{
    var post = new Post
    {
        Id = reader.GetInt32(0),
        Title = reader.GetString(1),
        Content = reader.GetString(2),
        CreatedAt = reader.GetDateTime(3)
    };
    return post;
}
```

**ด้วย ORM (EF Core):**

```csharp
// สะอาด อ่านง่าย และปลอดภัย
var post = await context.Posts.FindAsync(postId);
```

### ข้อดีของ ORM

1. **ลดโค้ดซ้ำซ้อน**: ไม่ต้องเขียน SQL boilerplate code
2. **Type Safety**: Compiler ตรวจสอบ type ให้
3. **Maintainability**: เปลี่ยน database schema ง่ายขึ้น
4. **Database Agnostic**: เปลี่ยน database provider ได้โดยไม่ต้องแก้ logic
5. **Security**: ป้องกัน SQL Injection อัตโนมัติ

### ข้อเสียของ ORM

1. **Performance overhead**: มี overhead เพิ่มเติมจาก query generation
2. **Learning curve**: ต้องเรียนรู้ API ของ ORM
3. **Complex queries**: บางกรณี SQL ตรงๆ ทำได้ดีกว่า

---

## Entity Framework Core คืออะไร

**Entity Framework Core (EF Core)** คือ ORM อย่างเป็นทางการของ Microsoft สำหรับ .NET รองรับ database หลายตัว:

- SQL Server
- PostgreSQL
- MySQL/MariaDB
- SQLite
- Oracle
- และอื่นๆ

EF Core 9 มาพร้อมกับ .NET 9 และรองรับ C# 13

---

## ติดตั้ง EF Core

### สร้าง Project ใหม่

```bash
dotnet new console -n BlogApp
cd BlogApp
```

### ติดตั้ง NuGet Packages

สำหรับ SQL Server:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.0
```

สำหรับ SQLite (เหมาะสำหรับ development):

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 9.0.0
```

### ตรวจสอบ .csproj

หลังติดตั้งแล้ว ไฟล์ `.csproj` ควรมีลักษณะนี้:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="9.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

</Project>
```

---

## Entities (Model Classes)

**Entity** คือ C# class ที่แสดงถึง table ใน database

```csharp
// Models/Blog.cs
namespace BlogApp.Models;

public class Blog
{
    public int Id { get; set; }           // Primary Key
    public string Name { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    // Navigation property - one Blog has many Posts
    public List<Post> Posts { get; set; } = [];
}
```

```csharp
// Models/Post.cs
namespace BlogApp.Models;

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime PublishedAt { get; set; }
    public bool IsPublished { get; set; }
    
    // Foreign Key
    public int BlogId { get; set; }
    
    // Navigation property
    public Blog Blog { get; set; } = null!;
    
    // Tags collection (many-to-many)
    public List<Tag> Tags { get; set; } = [];
}
```

```csharp
// Models/Tag.cs
namespace BlogApp.Models;

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    
    public List<Post> Posts { get; set; } = [];
}
```

---

## DbContext

**DbContext** คือ class หลักของ EF Core ทำหน้าที่เป็นสะพานเชื่อมระหว่าง Entity Classes กับ Database

```csharp
// Data/BlogDbContext.cs
using Microsoft.EntityFrameworkCore;
using BlogApp.Models;

namespace BlogApp.Data;

public class BlogDbContext : DbContext
{
    // DbSet properties แทน tables ใน database
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
    public DbSet<Tag> Tags { get; set; }
    
    // Constructor สำหรับรับ options
    public BlogDbContext(DbContextOptions<BlogDbContext> options) 
        : base(options)
    {
    }
    
    // Constructor ไม่มี options (สำหรับ override OnConfiguring)
    public BlogDbContext()
    {
    }
    
    // กำหนด connection string ใน OnConfiguring
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder.UseSqlite("Data Source=blog.db");
        }
    }
    
    // กำหนด model configuration ใน OnModelCreating
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Configure Blog entity
        modelBuilder.Entity<Blog>(entity =>
        {
            entity.HasKey(b => b.Id);
            entity.Property(b => b.Name)
                  .IsRequired()
                  .HasMaxLength(200);
            entity.Property(b => b.Url)
                  .IsRequired()
                  .HasMaxLength(500);
        });
        
        // Configure Post entity
        modelBuilder.Entity<Post>(entity =>
        {
            entity.HasKey(p => p.Id);
            entity.Property(p => p.Title)
                  .IsRequired()
                  .HasMaxLength(300);
            
            // One Blog -> Many Posts
            entity.HasOne(p => p.Blog)
                  .WithMany(b => b.Posts)
                  .HasForeignKey(p => p.BlogId)
                  .OnDelete(DeleteBehavior.Cascade);
        });
        
        // Configure Tag entity
        modelBuilder.Entity<Tag>(entity =>
        {
            entity.HasKey(t => t.Id);
            entity.Property(t => t.Name)
                  .IsRequired()
                  .HasMaxLength(50);
            
            // Many Posts <-> Many Tags
            entity.HasMany(t => t.Posts)
                  .WithMany(p => p.Tags)
                  .UsingEntity("PostTags");
        });
    }
}
```

---

## DbSet&lt;T&gt;

`DbSet<T>` แทน collection ของ entities ที่เชื่อมกับ table ใน database

```csharp
// ตัวอย่างการใช้งาน DbSet
using BlogApp.Data;
using BlogApp.Models;
using Microsoft.EntityFrameworkCore;

var context = new BlogDbContext();

// เพิ่มข้อมูล
var blog = new Blog { Name = "My Blog", Url = "https://myblog.com" };
context.Blogs.Add(blog);  // ใช้ DbSet.Add()
await context.SaveChangesAsync();

// ดึงข้อมูลทั้งหมด
var allBlogs = await context.Blogs.ToListAsync();

// ค้นหาด้วย Primary Key
var foundBlog = await context.Blogs.FindAsync(1);

// Query ด้วย LINQ
var activePosts = await context.Posts
    .Where(p => p.IsPublished)
    .OrderByDescending(p => p.PublishedAt)
    .ToListAsync();
```

---

## Code First vs Database First

### Code First Approach

เขียน C# classes ก่อน แล้วให้ EF Core สร้าง database schema ให้

**ข้อดี:**
- Database schema อยู่ใน code (version control ได้)
- ง่ายต่อการ migrate
- เหมาะกับ project ใหม่

**ข้อเสีย:**
- ต้องออกแบบ schema ผ่าน code อาจซับซ้อน

```csharp
// 1. สร้าง Entity classes
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

// 2. สร้าง DbContext
public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder options)
        => options.UseSqlite("Data Source=app.db");
}

// 3. สร้าง Migration และ Database
// dotnet ef migrations add InitialCreate
// dotnet ef database update
```

### Database First Approach

มี database อยู่แล้ว แล้วให้ EF Core สร้าง C# classes จาก database

```bash
# Scaffold จาก existing database
dotnet ef dbcontext scaffold "Data Source=existing.db" \
    Microsoft.EntityFrameworkCore.Sqlite \
    --output-dir Models \
    --context-dir Data \
    --context ExistingDbContext
```

**ข้อดี:**
- ใช้กับ database ที่มีอยู่แล้ว
- DBA สามารถออกแบบ schema ได้อิสระ

**ข้อเสีย:**
- ต้อง re-scaffold เมื่อ schema เปลี่ยน
- Generated code อาจต้องแก้ไข

---

## Migration เบื้องต้น

Migration คือกลไกที่ใช้ติดตามการเปลี่ยนแปลง database schema และ apply การเปลี่ยนแปลงนั้นกับ database จริง

### ติดตั้ง EF Core Tools

```bash
dotnet tool install --global dotnet-ef
```

### Workflow ของ Migration

```
Entity Classes → Add-Migration → Migration File → Update-Database → Database
```

### คำสั่ง Migration

```bash
# สร้าง migration ใหม่
dotnet ef migrations add InitialCreate

# Apply migration ไปยัง database
dotnet ef database update

# ย้อน migration กลับไปจุดก่อนหน้า
dotnet ef database update PreviousMigrationName

# ลบ migration ล่าสุด (ยังไม่ได้ apply)
dotnet ef migrations remove

# ดู list ของ migrations
dotnet ef migrations list
```

### ตัวอย่าง Migration File ที่ถูกสร้างขึ้น

```csharp
// Migrations/20241001000000_InitialCreate.cs
using System;
using Microsoft.EntityFrameworkCore.Migrations;

#nullable disable

namespace BlogApp.Migrations
{
    public partial class InitialCreate : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "Blogs",
                columns: table => new
                {
                    Id = table.Column<int>(type: "INTEGER", nullable: false)
                        .Annotation("Sqlite:Autoincrement", true),
                    Name = table.Column<string>(type: "TEXT", maxLength: 200, nullable: false),
                    Url = table.Column<string>(type: "TEXT", maxLength: 500, nullable: false),
                    CreatedAt = table.Column<DateTime>(type: "TEXT", nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Blogs", x => x.Id);
                });

            migrationBuilder.CreateTable(
                name: "Posts",
                columns: table => new
                {
                    Id = table.Column<int>(type: "INTEGER", nullable: false)
                        .Annotation("Sqlite:Autoincrement", true),
                    Title = table.Column<string>(type: "TEXT", maxLength: 300, nullable: false),
                    Content = table.Column<string>(type: "TEXT", nullable: false),
                    PublishedAt = table.Column<DateTime>(type: "TEXT", nullable: false),
                    IsPublished = table.Column<bool>(type: "INTEGER", nullable: false),
                    BlogId = table.Column<int>(type: "INTEGER", nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Posts", x => x.Id);
                    table.ForeignKey(
                        name: "FK_Posts_Blogs_BlogId",
                        column: x => x.BlogId,
                        principalTable: "Blogs",
                        principalColumn: "Id",
                        onDelete: ReferentialAction.Cascade);
                });
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(name: "Posts");
            migrationBuilder.DropTable(name: "Blogs");
        }
    }
}
```

---

## โปรแกรมตัวอย่าง: Blog Database

มาสร้างโปรแกรม Blog ที่สมบูรณ์กัน:

### โครงสร้าง Project

```
BlogApp/
├── BlogApp.csproj
├── Program.cs
├── Models/
│   ├── Blog.cs
│   ├── Post.cs
│   └── Tag.cs
├── Data/
│   └── BlogDbContext.cs
└── Migrations/  (สร้างโดย EF Core)
```

### Models

```csharp
// Models/Blog.cs
namespace BlogApp.Models;

public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Url { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsActive { get; set; } = true;
    
    public List<Post> Posts { get; set; } = [];
    
    public override string ToString() => 
        $"Blog: {Name} ({Url}) - {Posts.Count} posts";
}
```

```csharp
// Models/Post.cs
namespace BlogApp.Models;

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public string Summary { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? PublishedAt { get; set; }
    public bool IsPublished { get; set; }
    public int ViewCount { get; set; }
    
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;
    
    public List<Tag> Tags { get; set; } = [];
    public List<Comment> Comments { get; set; } = [];
    
    public override string ToString() => 
        $"Post: {Title} [{(IsPublished ? "Published" : "Draft")}]";
}
```

```csharp
// Models/Tag.cs
namespace BlogApp.Models;

public class Tag
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Slug { get; set; } = string.Empty;
    
    public List<Post> Posts { get; set; } = [];
}
```

```csharp
// Models/Comment.cs
namespace BlogApp.Models;

public class Comment
{
    public int Id { get; set; }
    public string AuthorName { get; set; } = string.Empty;
    public string AuthorEmail { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsApproved { get; set; }
    
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
}
```

### DbContext

```csharp
// Data/BlogDbContext.cs
using Microsoft.EntityFrameworkCore;
using BlogApp.Models;

namespace BlogApp.Data;

public class BlogDbContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }
    public DbSet<Post> Posts { get; set; }
    public DbSet<Tag> Tags { get; set; }
    public DbSet<Comment> Comments { get; set; }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder
            .UseSqlite("Data Source=blog.db")
            .LogTo(Console.WriteLine, 
                   Microsoft.Extensions.Logging.LogLevel.Information);
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Blog configuration
        modelBuilder.Entity<Blog>(e =>
        {
            e.HasKey(b => b.Id);
            e.Property(b => b.Name).IsRequired().HasMaxLength(200);
            e.Property(b => b.Url).IsRequired().HasMaxLength(500);
            e.Property(b => b.Description).HasMaxLength(1000);
            e.HasIndex(b => b.Url).IsUnique();
        });
        
        // Post configuration
        modelBuilder.Entity<Post>(e =>
        {
            e.HasKey(p => p.Id);
            e.Property(p => p.Title).IsRequired().HasMaxLength(300);
            e.Property(p => p.Content).IsRequired();
            e.Property(p => p.Summary).HasMaxLength(500);
            
            e.HasOne(p => p.Blog)
             .WithMany(b => b.Posts)
             .HasForeignKey(p => p.BlogId)
             .OnDelete(DeleteBehavior.Cascade);
        });
        
        // Tag configuration
        modelBuilder.Entity<Tag>(e =>
        {
            e.HasKey(t => t.Id);
            e.Property(t => t.Name).IsRequired().HasMaxLength(50);
            e.Property(t => t.Slug).IsRequired().HasMaxLength(50);
            e.HasIndex(t => t.Slug).IsUnique();
            
            e.HasMany(t => t.Posts)
             .WithMany(p => p.Tags)
             .UsingEntity<Dictionary<string, object>>(
                 "PostTag",
                 j => j.HasOne<Post>().WithMany().HasForeignKey("PostId"),
                 j => j.HasOne<Tag>().WithMany().HasForeignKey("TagId")
             );
        });
        
        // Comment configuration
        modelBuilder.Entity<Comment>(e =>
        {
            e.HasKey(c => c.Id);
            e.Property(c => c.AuthorName).IsRequired().HasMaxLength(100);
            e.Property(c => c.AuthorEmail).HasMaxLength(200);
            e.Property(c => c.Content).IsRequired();
            
            e.HasOne(c => c.Post)
             .WithMany(p => p.Comments)
             .HasForeignKey(c => c.PostId)
             .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

### Program.cs

```csharp
// Program.cs
using BlogApp.Data;
using BlogApp.Models;
using Microsoft.EntityFrameworkCore;

Console.WriteLine("=== Blog Application with EF Core 9 ===\n");

// สร้าง Database และ apply migrations
await using var context = new BlogDbContext();
await context.Database.EnsureCreatedAsync();

// 1. สร้าง Tags
Console.WriteLine("1. Creating tags...");
var csharpTag = new Tag { Name = "C#", Slug = "csharp" };
var dotnetTag = new Tag { Name = ".NET", Slug = "dotnet" };
var efcoreTag = new Tag { Name = "EF Core", Slug = "efcore" };

context.Tags.AddRange(csharpTag, dotnetTag, efcoreTag);
await context.SaveChangesAsync();
Console.WriteLine($"   Created {await context.Tags.CountAsync()} tags");

// 2. สร้าง Blog
Console.WriteLine("\n2. Creating blog...");
var blog = new Blog
{
    Name = "Tech Blog Thailand",
    Url = "https://techblog.th",
    Description = "บล็อกเทคโนโลยีสำหรับนักพัฒนาไทย"
};
context.Blogs.Add(blog);
await context.SaveChangesAsync();
Console.WriteLine($"   Created blog: {blog.Name} (Id: {blog.Id})");

// 3. สร้าง Posts
Console.WriteLine("\n3. Creating posts...");
var post1 = new Post
{
    Title = "เริ่มต้นกับ EF Core 9",
    Content = "EF Core 9 มีฟีเจอร์ใหม่มากมาย...",
    Summary = "บทความแนะนำ EF Core 9",
    BlogId = blog.Id,
    IsPublished = true,
    PublishedAt = DateTime.UtcNow,
    Tags = [csharpTag, efcoreTag]
};

var post2 = new Post
{
    Title = "ASP.NET Core Web API",
    Content = "การสร้าง Web API ด้วย ASP.NET Core...",
    Summary = "บทความ Web API",
    BlogId = blog.Id,
    IsPublished = false,
    Tags = [csharpTag, dotnetTag]
};

context.Posts.AddRange(post1, post2);
await context.SaveChangesAsync();
Console.WriteLine($"   Created {await context.Posts.CountAsync()} posts");

// 4. เพิ่ม Comments
Console.WriteLine("\n4. Adding comments...");
var comment = new Comment
{
    AuthorName = "สมชาย ใจดี",
    AuthorEmail = "somchai@example.com",
    Content = "บทความดีมากครับ ขอบคุณ!",
    PostId = post1.Id,
    IsApproved = true
};
context.Comments.Add(comment);
await context.SaveChangesAsync();

// 5. Query ข้อมูล
Console.WriteLine("\n5. Querying data...");

// ดึง blogs พร้อม posts
var blogsWithPosts = await context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Tags)
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .ToListAsync();

foreach (var b in blogsWithPosts)
{
    Console.WriteLine($"\nBlog: {b.Name}");
    foreach (var p in b.Posts)
    {
        var tagNames = string.Join(", ", p.Tags.Select(t => t.Name));
        Console.WriteLine($"  - {p.Title} [Tags: {tagNames}] [Comments: {p.Comments.Count}]");
    }
}

// 6. ค้นหา published posts
Console.WriteLine("\n6. Published posts:");
var publishedPosts = await context.Posts
    .Where(p => p.IsPublished)
    .OrderByDescending(p => p.PublishedAt)
    .Select(p => new { p.Title, p.Summary, p.Blog.Name })
    .ToListAsync();

foreach (var p in publishedPosts)
{
    Console.WriteLine($"   [{p.Name}] {p.Title} - {p.Summary}");
}

// 7. อัปเดตข้อมูล
Console.WriteLine("\n7. Updating post2...");
post2.IsPublished = true;
post2.PublishedAt = DateTime.UtcNow;
await context.SaveChangesAsync();
Console.WriteLine($"   Post2 published: {post2.IsPublished}");

// 8. ลบข้อมูล
Console.WriteLine("\n8. Deleting a comment...");
context.Comments.Remove(comment);
await context.SaveChangesAsync();
Console.WriteLine($"   Comments remaining: {await context.Comments.CountAsync()}");

// 9. สถิติ
Console.WriteLine("\n9. Statistics:");
var stats = new
{
    Blogs = await context.Blogs.CountAsync(),
    Posts = await context.Posts.CountAsync(),
    PublishedPosts = await context.Posts.CountAsync(p => p.IsPublished),
    Tags = await context.Tags.CountAsync(),
    Comments = await context.Comments.CountAsync()
};
Console.WriteLine($"   Blogs: {stats.Blogs}");
Console.WriteLine($"   Total Posts: {stats.Posts}");
Console.WriteLine($"   Published: {stats.PublishedPosts}");
Console.WriteLine($"   Tags: {stats.Tags}");
Console.WriteLine($"   Comments: {stats.Comments}");

Console.WriteLine("\n=== Done! ===");
```

### การรันโปรแกรม

```bash
# สร้าง migration
dotnet ef migrations add InitialCreate

# Apply migration
dotnet ef database update

# รันโปรแกรม
dotnet run
```

### ผลลัพธ์ที่คาดหวัง

```
=== Blog Application with EF Core 9 ===

1. Creating tags...
   Created 3 tags

2. Creating blog...
   Created blog: Tech Blog Thailand (Id: 1)

3. Creating posts...
   Created 2 posts

4. Adding comments...

5. Querying data...

Blog: Tech Blog Thailand
  - เริ่มต้นกับ EF Core 9 [Tags: C#, EF Core] [Comments: 1]
  - ASP.NET Core Web API [Tags: C#, .NET] [Comments: 0]

6. Published posts:
   [Tech Blog Thailand] เริ่มต้นกับ EF Core 9 - บทความแนะนำ EF Core 9

7. Updating post2...
   Post2 published: True

8. Deleting a comment...
   Comments remaining: 0

9. Statistics:
   Blogs: 1
   Total Posts: 2
   Published: 2
   Tags: 3
   Comments: 0

=== Done! ===
```

---

## Logging และ Debugging

EF Core สามารถแสดง SQL queries ที่ถูก generate ได้:

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlite("Data Source=blog.db")
        // Log SQL queries ไปที่ Console
        .LogTo(Console.WriteLine, LogLevel.Information)
        // ให้แสดง parameter values
        .EnableSensitiveDataLogging()
        // ให้แสดง detailed errors
        .EnableDetailedErrors();
}
```

### Connection String ใน appsettings.json

สำหรับ ASP.NET Core Web Application:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=blog.db"
  }
}
```

```csharp
// Program.cs (ASP.NET Core)
builder.Services.AddDbContext<BlogDbContext>(options =>
    options.UseSqlite(
        builder.Configuration.GetConnectionString("DefaultConnection")));
```

---

## Exercises

### แบบฝึกหัดที่ 1: สร้าง Library System
สร้าง database สำหรับระบบห้องสมุดที่มี:
- `Book` (ISBN, Title, Author, Year, AvailableCopies)
- `Member` (MemberId, Name, Email, JoinDate)
- `Loan` (LoanId, BookId, MemberId, BorrowDate, DueDate, ReturnDate)

```csharp
// TODO: สร้าง Entity classes สำหรับ Library System
public class Book
{
    public int Id { get; set; }
    // เพิ่ม properties...
}

public class Member
{
    public int Id { get; set; }
    // เพิ่ม properties...
}

public class Loan
{
    public int Id { get; set; }
    // เพิ่ม properties...
}
```

### แบบฝึกหัดที่ 2: CRUD Operations
เขียนฟังก์ชัน CRUD สำหรับ Book:
- `AddBook(string isbn, string title, string author)`
- `GetBook(int id)`
- `UpdateBook(int id, string newTitle)`
- `DeleteBook(int id)`
- `SearchBooks(string keyword)` - ค้นหาจาก title หรือ author

### แบบฝึกหัดที่ 3: Statistics Query
เขียน query เพื่อหา:
1. สมาชิกที่ยืมหนังสือมากที่สุด 5 คน
2. หนังสือที่ถูกยืมมากที่สุด
3. หนังสือที่เกินกำหนดคืน (DueDate < today และ ReturnDate เป็น null)

---

## สรุป

ใน Part นี้เราได้เรียนรู้:

1. **ORM** คือเครื่องมือที่แปลงระหว่าง Object กับ Database automatically
2. **EF Core** คือ ORM ของ Microsoft สำหรับ .NET
3. **Entity** คือ C# class ที่แสดงถึง table ใน database
4. **DbContext** คือ class หลักที่จัดการ connection และ operations
5. **DbSet&lt;T&gt;** แทน table/collection ใน database
6. **Code First** เขียน class ก่อน แล้วสร้าง database
7. **Database First** scaffold จาก existing database
8. **Migration** คือกลไกติดตามและ apply การเปลี่ยนแปลง schema

---

## Part ถัดไป

ใน **Part 052** เราจะเรียนรู้เกี่ยวกับ **EF Core: Entities และ Relationships** โดยละเอียด:
- Entity configuration แบบต่างๆ
- Primary keys รูปแบบต่างๆ
- One-to-Many, Many-to-Many, One-to-One relationships
- Owned entities
- Complex type mappings

---

*Part 051/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
