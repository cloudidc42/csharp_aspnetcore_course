# Part 020: Namespaces และ Using

## เนื้อหาใน Part นี้
- Namespace declaration และโครงสร้าง
- Nested namespaces
- using directive
- using alias
- global using (C# 10+)
- File-scoped namespaces (C# 10+)
- โปรแกรมตัวอย่าง: Library organization

---

## 1. Namespace คืออะไร?

Namespace คือวิธีจัด organize โค้ดและป้องกัน name conflicts โดยการจัดกลุ่ม types ที่เกี่ยวข้องกัน

```csharp
// ไม่มี namespace - global namespace (ไม่แนะนำ)
class MyClass { }

// มี namespace
namespace MyApp.Models
{
    class User { }
    class Product { }
}

namespace MyApp.Services
{
    class UserService { }
    class ProductService { }
}

// ใช้งาน fully qualified name
var user = new MyApp.Models.User();
var service = new MyApp.Services.UserService();

// หรือใช้ using เพื่อไม่ต้องพิมพ์ full name
using MyApp.Models;
using MyApp.Services;

var user2 = new User();          // ไม่ต้องระบุ namespace
var service2 = new UserService();
```

---

## 2. Namespace Declaration

```csharp
// Block-scoped namespace (ก่อน C# 10)
namespace MyCompany.MyApp.Domain
{
    public class Customer
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
    }

    public class Order
    {
        public int Id { get; set; }
        public int CustomerId { get; set; }
    }
}

// File-scoped namespace (C# 10+) - แนะนำให้ใช้
// ไม่ต้อง indent ทั้งไฟล์
namespace MyCompany.MyApp.Domain;

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
}

// Naming Convention
// Organization.Product.Feature
// Company.Application.Layer
// ตัวอย่าง:
// Microsoft.EntityFrameworkCore.Design
// MyCompany.ECommerceApp.Infrastructure.Data
// Acme.Billing.Services
```

---

## 3. Nested Namespaces

```csharp
// วิธีที่ 1: Nested namespace แบบ block
namespace MyApp
{
    namespace Data
    {
        namespace Entities
        {
            public class UserEntity { }
        }

        namespace Repositories
        {
            public class UserRepository { }
        }
    }

    namespace Services
    {
        public class UserService { }
    }
}

// วิธีที่ 2: Dotted notation (แนะนำ - เหมือนกันแต่สะอาดกว่า)
namespace MyApp.Data.Entities
{
    public class ProductEntity { }
}

namespace MyApp.Data.Repositories
{
    public class ProductRepository { }
}

namespace MyApp.Services
{
    public class ProductService { }
}

// ตัวอย่าง namespace โครงสร้างจริง
// MyShop/
// ├── MyShop.Domain/
// │   ├── Entities/ (Customer, Order, Product)
// │   ├── Events/ (OrderCreatedEvent, etc.)
// │   └── ValueObjects/ (Money, Address)
// ├── MyShop.Application/
// │   ├── Commands/ (CreateOrderCommand)
// │   ├── Queries/ (GetOrderQuery)
// │   └── Services/ (OrderService, CartService)
// ├── MyShop.Infrastructure/
// │   ├── Data/ (DbContext, Repositories)
// │   ├── Email/ (EmailService)
// │   └── Payment/ (PaymentGateway)
// └── MyShop.API/
//     ├── Controllers/ (OrdersController)
//     └── Middleware/
```

---

## 4. using Directives

```csharp
// ไฟล์: Controllers/ProductController.cs
using System;                        // ใช้ DateTime, Console, etc.
using System.Collections.Generic;    // ใช้ List<T>, Dictionary<T,K>
using System.Linq;                   // ใช้ LINQ
using System.Threading.Tasks;        // ใช้ Task, async/await
using Microsoft.AspNetCore.Mvc;      // ใช้ Controller, IActionResult
using MyApp.Services;                // ใช้ ProductService
using MyApp.Models;                  // ใช้ Product, ProductDto

// ด้วย using ไม่ต้องเขียน namespace ทุกครั้ง
public class ProductController
{
    private readonly ProductService _service;

    public ProductController(ProductService service)
    {
        _service = service;
    }

    public List<Product> GetAll()
    {
        return _service.GetProducts()
            .Where(p => p.IsActive)
            .OrderBy(p => p.Name)
            .ToList();
    }
}

// Using ใน method scope - สำหรับ IDisposable
void ReadFile(string path)
{
    using var reader = new System.IO.StreamReader(path);
    string content = reader.ReadToEnd();
    Console.WriteLine(content);
}

// Using statement (block-based)
void WriteFile(string path, string content)
{
    using (var writer = new System.IO.StreamWriter(path))
    {
        writer.Write(content);
    } // Dispose() เรียกอัตโนมัติ
}
```

---

## 5. using Aliases

```csharp
// Alias สำหรับ namespace - ป้องกัน conflicts
using Forms = System.Windows.Forms;
using Drawing = System.Drawing;
using MyControl = MyApp.Controls;

// ใช้งาน
// var btn = new Forms.Button();
// var color = new Drawing.Color();

// Alias สำหรับ type
using IntList = System.Collections.Generic.List<int>;
using StringDict = System.Collections.Generic.Dictionary<string, string>;
using ProductPair = (string Name, decimal Price);

var numbers = new IntList { 1, 2, 3, 4, 5 };
var config = new StringDict { ["server"] = "localhost" };
ProductPair product = ("Laptop", 35000m);

Console.WriteLine($"Numbers: {string.Join(", ", numbers)}");
Console.WriteLine($"Server: {config["server"]}");
Console.WriteLine($"Product: {product.Name} = {product.Price:N0}฿");

// Alias เมื่อมีชื่อซ้ำกัน
using SystemTimer = System.Timers.Timer;
using ThreadingTimer = System.Threading.Timer;

// var t1 = new SystemTimer(1000);
// var t2 = new ThreadingTimer(null, null, 0, 1000);

// Type alias (C# 12+) - ทรงพลังกว่า
using NumberList = List<int>;                        // Generic alias
using Coordinates = (double Lat, double Lon);       // Tuple alias
using HttpHeaders = Dictionary<string, string[]>;   // Complex type alias

Coordinates loc = (13.756, 100.502);
Console.WriteLine($"Location: {loc.Lat}°N, {loc.Lon}°E");
```

---

## 6. global using (C# 10+)

global using ทำให้ using มีผลกับทุกไฟล์ใน project

```csharp
// GlobalUsings.cs - ไฟล์พิเศษสำหรับ global usings
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;

// ใน .csproj ก็กำหนดได้:
// <ImplicitUsings>enable</ImplicitUsings>
// จะ auto-add usings ที่ใช้บ่อยสำหรับ project type

// เมื่อใช้ global using แล้ว ไม่ต้อง using อีกในทุกไฟล์
// Program.cs - ไม่ต้อง using System; อีก
var list = new List<int> { 1, 2, 3 }; // ใช้ได้เลย
Console.WriteLine(list.Sum());

// Custom global using สำหรับโปรเจค
// GlobalUsings.cs

global using MyApp.Domain.Entities;
global using MyApp.Application.Services;
global using MyApp.Infrastructure.Data;

// ดี: ลด boilerplate ในทุกไฟล์
// ระวัง: อย่า global using ทุกอย่าง อาจทำให้ confused

// global using alias
global using Logger = Microsoft.Extensions.Logging.ILogger;
global using Config = Microsoft.Extensions.Configuration.IConfiguration;
```

---

## 7. File-Scoped Namespaces (C# 10+)

```csharp
// ก่อน C# 10 - ต้อง indent ทั้งไฟล์
namespace MyApp.Services
{
    public class UserService
    {
        public string GetUser() => "User";
    }

    public class AuthService
    {
        public bool Login() => true;
    }
}

// C# 10+ File-scoped namespace (แนะนำ)
// ใช้ ; แทน { }
// ทั้งไฟล์อยู่ใน namespace เดียวกัน
namespace MyApp.Services;  // ไม่มี indentation

public class UserService
{
    public string GetUser() => "User";
}

public class AuthService
{
    public bool Login() => true;
}

// ประโยชน์:
// - ลด indentation level 1 ระดับ
// - โค้ดอ่านง่ายขึ้น
// - ไม่สามารถมีหลาย namespace ในไฟล์เดียว (บังคับ 1 file = 1 namespace)
```

---

## 8. Namespace Organization Patterns

```csharp
// Pattern 1: Layer-based
namespace MyApp.Domain
namespace MyApp.Application
namespace MyApp.Infrastructure
namespace MyApp.Presentation

// Pattern 2: Feature-based
namespace MyApp.Features.Orders
namespace MyApp.Features.Products
namespace MyApp.Features.Users

// Pattern 3: Clean Architecture
namespace MyApp.Domain.Entities
namespace MyApp.Domain.ValueObjects
namespace MyApp.Domain.Events
namespace MyApp.Domain.Repositories

namespace MyApp.Application.Commands
namespace MyApp.Application.Queries
namespace MyApp.Application.Handlers
namespace MyApp.Application.DTOs

namespace MyApp.Infrastructure.Persistence
namespace MyApp.Infrastructure.External
namespace MyApp.Infrastructure.Identity

namespace MyApp.API.Controllers
namespace MyApp.API.Middleware
namespace MyApp.API.Filters

// ตัวอย่าง namespace ใน .NET BCL
// System - core types
// System.Collections.Generic - generic collections
// System.Linq - query operators
// System.Threading - threading primitives
// System.IO - file I/O
// System.Text.Json - JSON serialization
// System.Net.Http - HTTP client
// Microsoft.AspNetCore.Mvc - ASP.NET Core MVC
// Microsoft.EntityFrameworkCore - EF Core
```

---

## 9. Namespace Conflicts และ Resolution

```csharp
// เมื่อมีชื่อ class เดียวกันใน 2 namespaces
namespace Graphics
{
    public class Color { public string Name { get; set; } = ""; }
}

namespace Painting
{
    public class Color { public int RGB { get; set; } }
}

// ใช้ fully qualified name
var gc = new Graphics.Color { Name = "Red" };
var pc = new Painting.Color { RGB = 0xFF0000 };

// หรือใช้ alias
using GColor = Graphics.Color;
using PColor = Painting.Color;

var gc2 = new GColor { Name = "Blue" };
var pc2 = new PColor { RGB = 0x0000FF };

// extern alias สำหรับ assembly conflicts
// (ใช้น้อยมาก - สำหรับ assembly ที่มี namespace เดียวกัน)
// extern alias Lib1;
// extern alias Lib2;
// var obj1 = new Lib1::SharedNamespace.SharedClass();
```

---

## 10. โปรแกรมตัวอย่าง: Library Organization System

```csharp
// ============================================================
// โครงสร้างไฟล์:
// LibrarySystem/
// ├── GlobalUsings.cs
// ├── Domain/
// │   ├── Entities/
// │   │   ├── Book.cs
// │   │   ├── Member.cs
// │   │   └── Loan.cs
// │   ├── ValueObjects/
// │   │   └── ISBN.cs
// │   └── Enums/
// │       └── BookStatus.cs
// ├── Application/
// │   ├── Services/
// │   │   ├── BookService.cs
// │   │   └── LoanService.cs
// │   └── DTOs/
// │       └── BookDto.cs
// └── Infrastructure/
//     └── Repositories/
//         └── InMemoryBookRepository.cs
// ============================================================

// --- GlobalUsings.cs ---
global using System;
global using System.Collections.Generic;
global using System.Linq;

// --- Domain/Enums/BookStatus.cs ---
namespace LibrarySystem.Domain.Enums;

public enum BookStatus
{
    Available,
    CheckedOut,
    Reserved,
    Lost,
    Damaged
}

// --- Domain/ValueObjects/ISBN.cs ---
namespace LibrarySystem.Domain.ValueObjects;

public readonly struct ISBN
{
    private readonly string _value;

    public ISBN(string value)
    {
        string cleaned = value.Replace("-", "").Replace(" ", "");
        if (!IsValid(cleaned))
            throw new ArgumentException($"ISBN ไม่ถูกต้อง: {value}");
        _value = cleaned;
    }

    private static bool IsValid(string isbn)
    {
        if (isbn.Length == 10) return IsValidISBN10(isbn);
        if (isbn.Length == 13) return IsValidISBN13(isbn);
        return false;
    }

    private static bool IsValidISBN10(string isbn)
    {
        int sum = 0;
        for (int i = 0; i < 9; i++)
        {
            if (!char.IsDigit(isbn[i])) return false;
            sum += (10 - i) * (isbn[i] - '0');
        }
        char last = isbn[9];
        int lastVal = last == 'X' ? 10 : (last - '0');
        sum += lastVal;
        return sum % 11 == 0;
    }

    private static bool IsValidISBN13(string isbn)
    {
        if (!isbn.All(char.IsDigit)) return false;
        int sum = 0;
        for (int i = 0; i < 12; i++)
            sum += (isbn[i] - '0') * (i % 2 == 0 ? 1 : 3);
        int check = (10 - sum % 10) % 10;
        return check == isbn[12] - '0';
    }

    public string Formatted => _value.Length == 13
        ? $"{_value[..3]}-{_value[3]}-{_value[4..9]}-{_value[9..12]}-{_value[12]}"
        : _value;

    public override string ToString() => Formatted;
    public override bool Equals(object? obj) => obj is ISBN other && _value == other._value;
    public override int GetHashCode() => _value.GetHashCode();
    public static bool operator ==(ISBN a, ISBN b) => a.Equals(b);
    public static bool operator !=(ISBN a, ISBN b) => !a.Equals(b);
}

// --- Domain/Entities/Book.cs ---
namespace LibrarySystem.Domain.Entities;

using LibrarySystem.Domain.Enums;
using LibrarySystem.Domain.ValueObjects;

public class Book
{
    private static int _nextId = 1;

    public int Id { get; init; }
    public ISBN ISBN { get; init; }
    public string Title { get; set; }
    public string Author { get; set; }
    public string Publisher { get; set; }
    public int PublicationYear { get; set; }
    public string Category { get; set; }
    public BookStatus Status { get; internal set; }
    public DateTime AddedDate { get; init; }
    public int TotalLoans { get; private set; }

    public bool IsAvailable => Status == BookStatus.Available;

    public Book(ISBN isbn, string title, string author,
        string publisher, int year, string category = "General")
    {
        Id = _nextId++;
        ISBN = isbn;
        Title = title ?? throw new ArgumentNullException(nameof(title));
        Author = author ?? throw new ArgumentNullException(nameof(author));
        Publisher = publisher ?? throw new ArgumentNullException(nameof(publisher));
        PublicationYear = year;
        Category = category;
        Status = BookStatus.Available;
        AddedDate = DateTime.Now;
    }

    internal void IncrementLoanCount() => TotalLoans++;

    public override string ToString()
        => $"[{Id}] \"{Title}\" by {Author} | ISBN: {ISBN} | {Status}";
}

// --- Domain/Entities/Member.cs ---
namespace LibrarySystem.Domain.Entities;

public class Member
{
    private static int _nextId = 1000;

    public int Id { get; init; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string PhoneNumber { get; set; }
    public DateTime MemberSince { get; init; }
    public bool IsActive { get; set; }
    public int MaxLoansAllowed { get; set; } = 5;

    public Member(string name, string email, string phone = "")
    {
        Id = _nextId++;
        Name = name ?? throw new ArgumentNullException(nameof(name));
        Email = email ?? throw new ArgumentNullException(nameof(email));
        PhoneNumber = phone;
        MemberSince = DateTime.Now;
        IsActive = true;
    }

    public override string ToString()
        => $"[{Id}] {Name} <{Email}> | สมาชิกตั้งแต่: {MemberSince:dd/MM/yyyy}";
}

// --- Domain/Entities/Loan.cs ---
namespace LibrarySystem.Domain.Entities;

public class Loan
{
    private static int _nextId = 1;

    public int Id { get; init; }
    public int BookId { get; init; }
    public int MemberId { get; init; }
    public DateTime LoanDate { get; init; }
    public DateTime DueDate { get; init; }
    public DateTime? ReturnDate { get; private set; }
    public bool IsReturned => ReturnDate.HasValue;
    public bool IsOverdue => !IsReturned && DateTime.Now > DueDate;

    public int DaysOverdue => IsOverdue
        ? (int)(DateTime.Now - DueDate).TotalDays
        : 0;

    public decimal OverdueFee => DaysOverdue * 5m; // 5 บาท/วัน

    public Loan(int bookId, int memberId, int loanDays = 14)
    {
        Id = _nextId++;
        BookId = bookId;
        MemberId = memberId;
        LoanDate = DateTime.Now;
        DueDate = DateTime.Now.AddDays(loanDays);
    }

    public void Return()
    {
        if (IsReturned)
            throw new InvalidOperationException("หนังสือถูกคืนแล้ว");
        ReturnDate = DateTime.Now;
    }

    public override string ToString()
    {
        string status = IsReturned ? $"คืนแล้ว ({ReturnDate:dd/MM/yyyy})"
            : IsOverdue ? $"เกินกำหนด {DaysOverdue} วัน (ค่าปรับ {OverdueFee:N0}฿)"
            : $"ต้องคืน {DueDate:dd/MM/yyyy}";
        return $"Loan#{Id}: Book#{BookId} → Member#{MemberId} | {status}";
    }
}

// --- Application/DTOs/BookDto.cs ---
namespace LibrarySystem.Application.DTOs;

public class BookDto
{
    public int Id { get; set; }
    public string ISBN { get; set; } = "";
    public string Title { get; set; } = "";
    public string Author { get; set; } = "";
    public string Category { get; set; } = "";
    public string Status { get; set; } = "";
    public int TotalLoans { get; set; }
}

// --- Infrastructure/Repositories/InMemoryBookRepository.cs ---
namespace LibrarySystem.Infrastructure.Repositories;

using LibrarySystem.Domain.Entities;
using LibrarySystem.Domain.Enums;
using LibrarySystem.Domain.ValueObjects;

public class InMemoryBookRepository
{
    private readonly List<Book> _books = new();
    private readonly List<Loan> _loans = new();
    private readonly List<Member> _members = new();

    public void AddBook(Book book) => _books.Add(book);
    public void AddMember(Member member) => _members.Add(member);

    public Book? FindBookById(int id) => _books.FirstOrDefault(b => b.Id == id);
    public Book? FindBookByISBN(ISBN isbn) => _books.FirstOrDefault(b => b.ISBN == isbn);
    public Member? FindMemberById(int id) => _members.FirstOrDefault(m => m.Id == id);

    public IEnumerable<Book> GetAvailableBooks()
        => _books.Where(b => b.IsAvailable).OrderBy(b => b.Title);

    public IEnumerable<Book> SearchBooks(string query)
    {
        string q = query.ToLower();
        return _books.Where(b =>
            b.Title.ToLower().Contains(q) ||
            b.Author.ToLower().Contains(q) ||
            b.Category.ToLower().Contains(q));
    }

    public Loan? BorrowBook(int bookId, int memberId)
    {
        var book = FindBookById(bookId);
        if (book == null || !book.IsAvailable) return null;

        var member = FindMemberById(memberId);
        if (member == null || !member.IsActive) return null;

        int activeLoans = _loans.Count(l => l.MemberId == memberId && !l.IsReturned);
        if (activeLoans >= member.MaxLoansAllowed) return null;

        var loan = new Loan(bookId, memberId);
        _loans.Add(loan);
        book.Status = BookStatus.CheckedOut;
        book.IncrementLoanCount();
        return loan;
    }

    public bool ReturnBook(int loanId)
    {
        var loan = _loans.FirstOrDefault(l => l.Id == loanId);
        if (loan == null || loan.IsReturned) return false;

        loan.Return();
        var book = FindBookById(loan.BookId);
        if (book != null) book.Status = BookStatus.Available;
        return true;
    }

    public IEnumerable<Loan> GetActiveLoans()
        => _loans.Where(l => !l.IsReturned);

    public IEnumerable<Loan> GetOverdueLoans()
        => _loans.Where(l => l.IsOverdue);

    public IEnumerable<Loan> GetMemberLoanHistory(int memberId)
        => _loans.Where(l => l.MemberId == memberId);

    public (int total, int available, int checkedOut) GetBookStats()
    {
        int total = _books.Count;
        int available = _books.Count(b => b.Status == BookStatus.Available);
        return (total, available, total - available);
    }
}

// --- Application/Services/LibraryService.cs ---
namespace LibrarySystem.Application.Services;

using LibrarySystem.Infrastructure.Repositories;
using LibrarySystem.Domain.Entities;
using LibrarySystem.Domain.ValueObjects;
using LibrarySystem.Application.DTOs;

public class LibraryService
{
    private readonly InMemoryBookRepository _repo;

    public LibraryService(InMemoryBookRepository repo)
    {
        _repo = repo;
    }

    public void RegisterBook(string isbnStr, string title, string author,
        string publisher, int year, string category = "General")
    {
        var isbn = new ISBN(isbnStr);
        var book = new Book(isbn, title, author, publisher, year, category);
        _repo.AddBook(book);
        Console.WriteLine($"✅ เพิ่มหนังสือ: {book}");
    }

    public void RegisterMember(string name, string email, string phone = "")
    {
        var member = new Member(name, email, phone);
        _repo.AddMember(member);
        Console.WriteLine($"✅ เพิ่มสมาชิก: {member}");
    }

    public bool BorrowBook(int bookId, int memberId)
    {
        var loan = _repo.BorrowBook(bookId, memberId);
        if (loan != null)
        {
            Console.WriteLine($"📚 {loan}");
            return true;
        }
        Console.WriteLine($"❌ ยืมหนังสือไม่สำเร็จ (Book#{bookId}, Member#{memberId})");
        return false;
    }

    public bool ReturnBook(int loanId)
    {
        bool success = _repo.ReturnBook(loanId);
        Console.WriteLine(success
            ? $"↩️ คืนหนังสือ Loan#{loanId} สำเร็จ"
            : $"❌ คืนหนังสือไม่สำเร็จ Loan#{loanId}");
        return success;
    }

    public void PrintAvailableBooks()
    {
        Console.WriteLine("\n=== หนังสือที่พร้อมยืม ===");
        var books = _repo.GetAvailableBooks().ToList();
        if (!books.Any())
        {
            Console.WriteLine("  ไม่มีหนังสือที่พร้อมให้ยืม");
            return;
        }
        foreach (var book in books)
            Console.WriteLine($"  {book}");
    }

    public void PrintOverdueLoans()
    {
        Console.WriteLine("\n=== หนังสือเกินกำหนด ===");
        var overdue = _repo.GetOverdueLoans().ToList();
        if (!overdue.Any())
        {
            Console.WriteLine("  ไม่มีหนังสือเกินกำหนด 🎉");
            return;
        }
        foreach (var loan in overdue)
            Console.WriteLine($"  ⚠️ {loan}");
    }

    public void PrintStats()
    {
        var (total, available, checkedOut) = _repo.GetBookStats();
        int activeLoans = _repo.GetActiveLoans().Count();
        int overdueLoans = _repo.GetOverdueLoans().Count();

        Console.WriteLine($"""

            === สถิติห้องสมุด ===
            หนังสือทั้งหมด    : {total} เล่ม
            พร้อมให้ยืม       : {available} เล่ม
            ถูกยืมออกไป      : {checkedOut} เล่ม
            การยืมที่ active  : {activeLoans} รายการ
            เกินกำหนดคืน     : {overdueLoans} รายการ
            """);
    }

    public void SearchBooks(string query)
    {
        Console.WriteLine($"\n=== ค้นหา: '{query}' ===");
        var results = _repo.SearchBooks(query).ToList();
        Console.WriteLine($"พบ {results.Count} รายการ:");
        foreach (var book in results)
            Console.WriteLine($"  {book}");
    }

    public IEnumerable<BookDto> GetAllBooksAsDto()
    {
        return _repo.GetAvailableBooks().Select(b => new BookDto
        {
            Id = b.Id,
            ISBN = b.ISBN.ToString(),
            Title = b.Title,
            Author = b.Author,
            Category = b.Category,
            Status = b.Status.ToString(),
            TotalLoans = b.TotalLoans
        });
    }
}

// --- Program.cs ---
using LibrarySystem.Application.Services;
using LibrarySystem.Infrastructure.Repositories;

Console.WriteLine("=== ระบบห้องสมุด ===\n");

var repo = new InMemoryBookRepository();
var library = new LibraryService(repo);

// เพิ่มหนังสือ
library.RegisterBook("9780132350884", "Clean Code", "Robert C. Martin",
    "Prentice Hall", 2008, "Programming");
library.RegisterBook("9780201633610", "Design Patterns", "Gang of Four",
    "Addison-Wesley", 1994, "Programming");
library.RegisterBook("9781491950357", "Learning ASP.NET Core",
    "Various Authors", "O'Reilly", 2023, "Web Development");
library.RegisterBook("9780136468684", "C# in Depth",
    "Jon Skeet", "Manning", 2019, "Programming");
library.RegisterBook("9780321125217", "Domain-Driven Design",
    "Eric Evans", "Addison-Wesley", 2003, "Architecture");

// เพิ่มสมาชิก
library.RegisterMember("สมชาย ใจดี", "somchai@email.com", "0812345678");
library.RegisterMember("สมหญิง รักดี", "somying@email.com", "0823456789");

Console.WriteLine();

// ยืมหนังสือ
library.BorrowBook(1, 1000); // Clean Code → สมชาย
library.BorrowBook(2, 1000); // Design Patterns → สมชาย
library.BorrowBook(3, 1001); // ASP.NET Core → สมหญิง

// แสดงหนังสือที่พร้อมยืม
library.PrintAvailableBooks();

// คืนหนังสือ
Console.WriteLine("\n=== คืนหนังสือ ===");
library.ReturnBook(1); // คืน loan#1 (Clean Code)

// ค้นหาหนังสือ
library.SearchBooks("clean");
library.SearchBooks("programming");

// สถิติ
library.PrintStats();
library.PrintOverdueLoans();
```

---

## Exercises

### Exercise 1: API Project Structure
```csharp
// TODO: สร้างโครงสร้าง namespace สำหรับ REST API:
// MyApi.Domain.*
// MyApi.Application.*
// MyApi.Infrastructure.*
// MyApi.API.*
// สร้าง GlobalUsings.cs และ file-scoped namespaces ทุกไฟล์
// แต่ละ namespace มีอย่างน้อย 2 types

// GlobalUsings.cs
global using System;
// TODO: เพิ่ม global usings ที่เหมาะสม
```

### Exercise 2: Plugin System
```csharp
// TODO: สร้าง plugin system ที่ใช้ namespace:
// MyPluginSystem.Core (IPlugin interface)
// MyPluginSystem.Plugins.TextProcessor
// MyPluginSystem.Plugins.ImageProcessor
// MyPluginSystem.Host (PluginLoader, PluginManager)
// ใช้ using aliases เพื่อป้องกัน naming conflicts

namespace MyPluginSystem.Core;

public interface IPlugin
{
    // TODO: Define plugin contract
}
```

### Exercise 3: Refactor Namespaces
```csharp
// TODO: ไฟล์ต่อไปนี้มีปัญหา namespace ที่ต้องแก้:
// 1. ทุกอย่างอยู่ใน global namespace
// 2. ชื่อ class ซ้ำกัน
// 3. ไม่มี using ที่ถูกต้อง
// Refactor ให้ถูกต้อง:
// - สร้าง namespaces ที่เหมาะสม
// - ใช้ file-scoped namespaces
// - ใช้ global using
// - แก้ naming conflicts

class Database { }   // ซ้ำกับ Microsoft.Data.Sqlite.Database
class Logger { }     // ซ้ำกับ Microsoft.Extensions.Logging.Logger
class User { }
class Order { }
class Product { }
```

---

## สรุป

✅ Namespace จัด organize โค้ดและป้องกัน name conflicts  
✅ File-scoped namespace (C# 10+) ลด indentation ทำให้โค้ดสะอาดกว่า  
✅ `using` directive ลดการพิมพ์ fully qualified names  
✅ `using` alias แก้ปัญหา name conflicts และทำให้โค้ดอ่านง่าย  
✅ `global using` (C# 10+) ใช้ร่วมกันทุกไฟล์ใน project  
✅ Namespace convention: Organization.Product.Layer หรือ Organization.Product.Feature  
✅ แยก concerns ด้วย namespace ช่วยให้โค้ด maintainable  
✅ 1 file = 1 namespace เป็น convention ที่ดี (บังคับโดย file-scoped namespace)  

## Part ถัดไป
**Part 021: Exception Handling** - เรียนรู้ try-catch-finally, custom exceptions, exception hierarchy และ best practices

---
*Part 020/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
