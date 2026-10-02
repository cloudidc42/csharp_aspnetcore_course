# Part 014: Constructors

## เนื้อหาใน Part นี้
- Default constructor
- Parameterized constructors
- Constructor overloading
- Constructor chaining ด้วย this()
- Object initializers
- Static constructor
- Destructor/Finalizer
- โปรแกรมตัวอย่าง: Employee class

---

## 1. Constructor คืออะไร?

Constructor คือ method พิเศษที่ถูกเรียกอัตโนมัติเมื่อสร้าง object ด้วย `new` keyword

**กฎของ Constructor:**
- ชื่อต้องเหมือนกับชื่อ class
- ไม่มี return type (ไม่เขียน void ด้วย)
- สามารถมีได้หลาย constructor (overloading)
- ถ้าไม่ประกาศ compiler สร้าง default constructor ให้อัตโนมัติ

```csharp
public class Car
{
    public string Make { get; set; }
    public string Model { get; set; }
    public int Year { get; set; }

    // Constructor
    public Car(string make, string model, int year)
    {
        Make = make;
        Model = model;
        Year = year;
        Console.WriteLine($"สร้าง Car: {year} {make} {model}");
    }
}

// เมื่อสร้าง object constructor ถูกเรียก
var car = new Car("Toyota", "Camry", 2024);
// Output: สร้าง Car: 2024 Toyota Camry
```

---

## 2. Default Constructor

```csharp
// ถ้าไม่ประกาศ constructor ใดเลย
// compiler สร้าง default (parameterless) constructor ให้
public class SimpleClass
{
    public string Name { get; set; } = "";
    // compiler สร้าง: public SimpleClass() { }
}

var obj = new SimpleClass(); // ใช้ default constructor

// ถ้าประกาศ constructor อื่นแล้ว
// default constructor จะไม่ถูกสร้างโดยอัตโนมัติ
public class NoDefault
{
    public string Name { get; set; }

    public NoDefault(string name) // มี explicit constructor
    {
        Name = name;
    }
}

// var bad = new NoDefault(); // Error! ไม่มี parameterless constructor
var good = new NoDefault("สมชาย"); // OK

// ถ้าต้องการทั้ง parameterless และ parameterized
public class WithBoth
{
    public string Name { get; set; } = "Unknown";
    public int Age { get; set; }

    // Explicit default constructor
    public WithBoth()
    {
        Console.WriteLine("Default constructor called");
    }

    public WithBoth(string name, int age)
    {
        Name = name;
        Age = age;
        Console.WriteLine($"Parameterized constructor: {name}, {age}");
    }
}

var a = new WithBoth();            // Default constructor
var b = new WithBoth("สมชาย", 30); // Parameterized constructor
```

---

## 3. Parameterized Constructors

```csharp
public class Rectangle
{
    public double Width { get; }
    public double Height { get; }

    // Constructor ที่รับ 2 parameters
    public Rectangle(double width, double height)
    {
        if (width <= 0)
            throw new ArgumentOutOfRangeException(nameof(width), "Width must be positive");
        if (height <= 0)
            throw new ArgumentOutOfRangeException(nameof(height), "Height must be positive");

        Width = width;
        Height = height;
    }

    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);

    public override string ToString()
        => $"Rectangle({Width} x {Height})";
}

// Optional parameters ใน constructor
public class Circle
{
    public double Radius { get; }
    public string Color { get; }
    public bool IsFilled { get; }

    // Constructor พร้อม optional parameters
    public Circle(double radius, string color = "Black", bool isFilled = true)
    {
        if (radius <= 0)
            throw new ArgumentOutOfRangeException(nameof(radius));
        Radius = radius;
        Color = color;
        IsFilled = isFilled;
    }

    public double Area => Math.PI * Radius * Radius;
}

// การใช้งาน
var r1 = new Rectangle(5, 3);
Console.WriteLine($"{r1}: Area={r1.Area}, Perimeter={r1.Perimeter}");

var c1 = new Circle(5);                      // radius=5, color="Black", filled=true
var c2 = new Circle(3, "Red");               // radius=3, color="Red", filled=true
var c3 = new Circle(4, "Blue", false);       // กำหนดทุก parameter
var c4 = new Circle(radius: 6, isFilled: false); // Named arguments

// Constructor พร้อม validation ที่ครอบคลุม
public class Person
{
    public string FirstName { get; }
    public string LastName { get; }
    public DateTime BirthDate { get; }
    public string Email { get; }

    public Person(string firstName, string lastName,
        DateTime birthDate, string email)
    {
        if (string.IsNullOrWhiteSpace(firstName))
            throw new ArgumentException("First name is required.", nameof(firstName));
        if (string.IsNullOrWhiteSpace(lastName))
            throw new ArgumentException("Last name is required.", nameof(lastName));
        if (birthDate > DateTime.Today)
            throw new ArgumentException("Birth date cannot be in the future.", nameof(birthDate));
        if (birthDate < DateTime.Today.AddYears(-150))
            throw new ArgumentException("Birth date is too far in the past.", nameof(birthDate));
        if (!email.Contains('@'))
            throw new ArgumentException("Invalid email format.", nameof(email));

        FirstName = firstName.Trim();
        LastName = lastName.Trim();
        BirthDate = birthDate.Date;
        Email = email.ToLower().Trim();
    }

    public string FullName => $"{FirstName} {LastName}";

    public int Age
    {
        get
        {
            var today = DateTime.Today;
            int age = today.Year - BirthDate.Year;
            if (BirthDate.Date > today.AddYears(-age))
                age--;
            return age;
        }
    }
}
```

---

## 4. Constructor Overloading

```csharp
public class DatabaseConnection
{
    public string Host { get; }
    public int Port { get; }
    public string DatabaseName { get; }
    public string Username { get; }
    private string Password { get; }
    public int ConnectionTimeout { get; }
    public bool UseSsl { get; }

    // Constructor 1: ขั้นต่ำ (localhost, default port)
    public DatabaseConnection(string databaseName, string username, string password)
        : this("localhost", 5432, databaseName, username, password) { }

    // Constructor 2: กำหนด host และ port
    public DatabaseConnection(string host, int port, string databaseName,
        string username, string password)
        : this(host, port, databaseName, username, password, 30, false) { }

    // Constructor 3: กำหนดทุก parameter
    public DatabaseConnection(string host, int port, string databaseName,
        string username, string password, int timeout, bool useSsl)
    {
        Host = host ?? throw new ArgumentNullException(nameof(host));
        Port = port > 0 && port <= 65535 ? port
            : throw new ArgumentOutOfRangeException(nameof(port));
        DatabaseName = databaseName ?? throw new ArgumentNullException(nameof(databaseName));
        Username = username ?? throw new ArgumentNullException(nameof(username));
        Password = password ?? throw new ArgumentNullException(nameof(password));
        ConnectionTimeout = timeout > 0 ? timeout : 30;
        UseSsl = useSsl;
    }

    public override string ToString()
        => $"postgresql://{Username}@{Host}:{Port}/{DatabaseName}" +
           $" (timeout={ConnectionTimeout}s, ssl={UseSsl})";
}

// การใช้งาน
var db1 = new DatabaseConnection("mydb", "admin", "secret");
var db2 = new DatabaseConnection("db.example.com", 5432, "proddb", "user", "pass");
var db3 = new DatabaseConnection("db.example.com", 5432, "proddb", "user", "pass", 60, true);

Console.WriteLine(db1);
Console.WriteLine(db2);
Console.WriteLine(db3);
```

---

## 5. Constructor Chaining ด้วย this()

Constructor chaining ช่วยหลีกเลี่ยงการ duplicate code

```csharp
public class Address
{
    public string Street { get; }
    public string City { get; }
    public string Province { get; }
    public string PostalCode { get; }
    public string Country { get; }

    // Main constructor - มี logic ทั้งหมด
    public Address(string street, string city, string province,
        string postalCode, string country)
    {
        Street = street?.Trim() ?? throw new ArgumentNullException(nameof(street));
        City = city?.Trim() ?? throw new ArgumentNullException(nameof(city));
        Province = province?.Trim() ?? throw new ArgumentNullException(nameof(province));
        PostalCode = postalCode?.Trim() ?? throw new ArgumentNullException(nameof(postalCode));
        Country = country?.Trim() ?? "Thailand";
    }

    // Convenience constructors - chain ไปหา main constructor
    public Address(string street, string city, string province, string postalCode)
        : this(street, city, province, postalCode, "Thailand") { }

    public Address(string street, string city, string postalCode)
        : this(street, city, city, postalCode) { } // ใช้ city เป็น province

    public override string ToString()
        => $"{Street}, {City}, {Province} {PostalCode}, {Country}";
}

// ตัวอย่างที่ซับซ้อนขึ้น - Order class
public class Order
{
    public int OrderId { get; }
    public string CustomerId { get; }
    public DateTime OrderDate { get; }
    public List<OrderLine> Lines { get; }
    public string Status { get; private set; }
    public string? Notes { get; set; }

    private static int _nextOrderId = 1000;

    // Primary constructor
    public Order(string customerId, DateTime orderDate, List<OrderLine>? lines = null,
        string status = "Pending", string? notes = null)
    {
        if (string.IsNullOrWhiteSpace(customerId))
            throw new ArgumentException("Customer ID required.");

        OrderId = _nextOrderId++;
        CustomerId = customerId;
        OrderDate = orderDate;
        Lines = lines ?? new List<OrderLine>();
        Status = status;
        Notes = notes;
    }

    // Convenience: order สำหรับวันนี้
    public Order(string customerId)
        : this(customerId, DateTime.Now) { }

    // Convenience: order พร้อม lines
    public Order(string customerId, List<OrderLine> lines)
        : this(customerId, DateTime.Now, lines) { }

    public decimal Total => Lines.Sum(l => l.LineTotal);

    public void AddLine(string product, int qty, decimal price)
        => Lines.Add(new OrderLine(product, qty, price));

    public void UpdateStatus(string newStatus) => Status = newStatus;
}

public class OrderLine
{
    public string ProductName { get; }
    public int Quantity { get; }
    public decimal UnitPrice { get; }
    public decimal LineTotal => Quantity * UnitPrice;

    public OrderLine(string product, int qty, decimal price)
    {
        ProductName = product;
        Quantity = qty;
        UnitPrice = price;
    }
}
```

---

## 6. Object Initializers

Object initializers ทำให้สร้าง object และกำหนดค่าได้ใน expression เดียว

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public double GPA { get; set; }
    public List<string> Courses { get; set; } = new();
    public Address? HomeAddress { get; set; }
}

// Object initializer
var student = new Student
{
    Id = 1,
    Name = "สมชาย ใจดี",
    Email = "somchai@uni.edu",
    GPA = 3.75,
    Courses = new List<string> { "Math", "Physics", "Computer" },
    HomeAddress = new Address("123 Main St", "Bangkok", "10100")
    {
        // Address ก็ใช้ object initializer ได้
    }
};

// Object initializer กับ anonymous types
var anonymous = new
{
    Name = "สมชาย",
    Age = 30,
    Score = 95.5
};
Console.WriteLine($"{anonymous.Name}: {anonymous.Score}");

// Collection initializer
var students = new List<Student>
{
    new Student { Id = 1, Name = "สมชาย", GPA = 3.5 },
    new Student { Id = 2, Name = "สมหญิง", GPA = 3.8 },
    new Student { Id = 3, Name = "สมศรี", GPA = 3.2 }
};

// Dictionary initializer
var grades = new Dictionary<string, string>
{
    { "A", "ดีเยี่ยม" },
    { "B", "ดี" },
    { "C", "พอใช้" },
    ["D"] = "ผ่าน",   // index initializer syntax
    ["F"] = "ตก"
};

// Object initializer + Constructor
public class Config
{
    public string Server { get; set; } = "";
    public int Port { get; set; } = 80;
    public bool Secure { get; set; }
    public TimeSpan Timeout { get; set; } = TimeSpan.FromSeconds(30);
}

var config = new Config
{
    Server = "api.example.com",
    Port = 443,
    Secure = true,
    Timeout = TimeSpan.FromMinutes(2)
};
```

---

## 7. Static Constructor

Static constructor เรียกครั้งเดียวก่อน class ถูกใช้งานครั้งแรก

```csharp
public class AppSettings
{
    // Static properties
    public static string ConnectionString { get; private set; } = "";
    public static string ApiKey { get; private set; } = "";
    public static int MaxRetries { get; private set; }
    public static Dictionary<string, string> Config { get; } = new();

    // Static constructor - ไม่มี access modifier, ไม่มี parameters
    static AppSettings()
    {
        Console.WriteLine("AppSettings static constructor called");

        // โหลดค่าจาก environment หรือ config file
        ConnectionString = Environment.GetEnvironmentVariable("DB_CONNECTION")
            ?? "Server=localhost;Database=mydb;";
        ApiKey = Environment.GetEnvironmentVariable("API_KEY")
            ?? "default-api-key";
        MaxRetries = int.TryParse(
            Environment.GetEnvironmentVariable("MAX_RETRIES"), out int retries)
            ? retries : 3;

        // Initialize static collections
        Config["env"] = Environment.GetEnvironmentVariable("APP_ENV") ?? "Development";
        Config["version"] = "1.0.0";
        Config["appName"] = "MyApplication";
    }

    // ไม่สามารถ new ได้ - static class หรือ private constructor
    private AppSettings() { }
}

// Logger class ที่ใช้ static constructor
public class Logger
{
    private static readonly string _logDirectory;
    private static readonly string _logFile;

    static Logger()
    {
        _logDirectory = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "logs");
        Directory.CreateDirectory(_logDirectory);
        _logFile = Path.Combine(_logDirectory,
            $"app-{DateTime.Now:yyyy-MM-dd}.log");
        Console.WriteLine($"Logger initialized: {_logFile}");
    }

    public static void Log(string message, string level = "INFO")
    {
        string entry = $"[{DateTime.Now:HH:mm:ss.fff}] [{level}] {message}";
        Console.WriteLine(entry);
        // File.AppendAllText(_logFile, entry + Environment.NewLine);
    }
}

// การใช้งาน
Logger.Log("Application started");    // static constructor ถูกเรียกก่อน
Logger.Log("Processing request", "DEBUG");
Console.WriteLine($"ENV: {AppSettings.Config["env"]}");
```

---

## 8. Destructor และ Finalizer

```csharp
// Destructor (Finalizer) - ถูกเรียกโดย Garbage Collector
// ไม่แนะนำให้ใช้บ่อย - ใช้ IDisposable แทน

public class ResourceHolder
{
    private readonly string _resourceName;
    private bool _isReleased = false;

    public ResourceHolder(string name)
    {
        _resourceName = name;
        Console.WriteLine($"Acquired resource: {name}");
    }

    // Destructor/Finalizer - ถูกเรียกโดย GC ไม่ใช่โปรแกรมเมอร์
    ~ResourceHolder()
    {
        Console.WriteLine($"Finalizer called for: {_resourceName}");
        // ทำความสะอาด unmanaged resources ที่ไม่มีการเรียก Dispose
        ReleaseResource();
    }

    private void ReleaseResource()
    {
        if (!_isReleased)
        {
            Console.WriteLine($"Releasing resource: {_resourceName}");
            _isReleased = true;
        }
    }
}

// วิธีที่ดีกว่า: IDisposable Pattern
public class ManagedResource : IDisposable
{
    private bool _disposed = false;
    private readonly string _name;

    public ManagedResource(string name)
    {
        _name = name;
        Console.WriteLine($"Opening: {name}");
    }

    // Public Dispose method
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this); // บอก GC ไม่ต้องเรียก finalizer
    }

    // Protected virtual Dispose สำหรับ inheritance
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // ปล่อย managed resources
                Console.WriteLine($"Releasing managed resources: {_name}");
            }
            // ปล่อย unmanaged resources (ถ้ามี)
            Console.WriteLine($"Closing: {_name}");
            _disposed = true;
        }
    }

    // Finalizer เป็น safety net
    ~ManagedResource()
    {
        Dispose(false);
    }

    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        Console.WriteLine($"Working with: {_name}");
    }
}

// การใช้งาน IDisposable ที่ถูกต้อง
// วิธีที่ 1: using statement (แนะนำ)
using (var resource = new ManagedResource("FileStream"))
{
    resource.DoWork();
} // Dispose() ถูกเรียกอัตโนมัติ

// วิธีที่ 2: using declaration (C# 8+)
using var resource2 = new ManagedResource("NetworkStream");
resource2.DoWork();
// Dispose() ถูกเรียกเมื่อออกจาก scope
```

---

## 9. โปรแกรมตัวอย่าง: Employee Class

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace EmployeeExample
{
    /// <summary>
    /// ประเภทพนักงาน
    /// </summary>
    public enum EmployeeType
    {
        FullTime,
        PartTime,
        Contract,
        Intern
    }

    /// <summary>
    /// แผนกงาน
    /// </summary>
    public enum Department
    {
        Engineering,
        Marketing,
        Sales,
        HumanResources,
        Finance,
        Operations,
        Management
    }

    /// <summary>
    /// ที่อยู่
    /// </summary>
    public class Address
    {
        public string Street { get; init; }
        public string City { get; init; }
        public string Province { get; init; }
        public string PostalCode { get; init; }

        // Primary constructor
        public Address(string street, string city, string province, string postalCode)
        {
            Street = street;
            City = city;
            Province = province;
            PostalCode = postalCode;
        }

        // Default (กรุงเทพ)
        public Address() : this("ไม่ระบุ", "กรุงเทพมหานคร", "กรุงเทพมหานคร", "10000") { }

        public override string ToString()
            => $"{Street}, {City}, {Province} {PostalCode}";
    }

    /// <summary>
    /// ข้อมูลพนักงาน
    /// </summary>
    public class Employee : IDisposable
    {
        // Static
        private static int _nextEmployeeId = 1000;
        private static int _totalEmployees = 0;
        private bool _disposed = false;

        // Properties - init only
        public int EmployeeId { get; init; }
        public string NationalId { get; init; }
        public DateTime StartDate { get; init; }

        // Properties - settable
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public DateTime BirthDate { get; set; }
        public string Email { get; set; }
        public string? Phone { get; set; }
        public Department Department { get; set; }
        public EmployeeType EmployeeType { get; set; }
        public Address? HomeAddress { get; set; }

        // Private backing field
        private decimal _baseSalary;
        public decimal BaseSalary
        {
            get => _baseSalary;
            set
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException(nameof(value), "Salary cannot be negative");
                _baseSalary = value;
            }
        }

        // Performance records
        private readonly List<PerformanceRecord> _performanceHistory = new();

        // Computed
        public string FullName => $"{FirstName} {LastName}";
        public int Age => CalculateAge(BirthDate);
        public int YearsOfService => CalculateAge(StartDate);
        public static int TotalEmployees => _totalEmployees;

        public decimal AnnualSalary
        {
            get
            {
                return EmployeeType switch
                {
                    EmployeeType.FullTime => _baseSalary * 12,
                    EmployeeType.PartTime => _baseSalary * 12 * 0.5m,
                    EmployeeType.Contract => _baseSalary * 12,
                    EmployeeType.Intern => _baseSalary * 12 * 0.7m,
                    _ => _baseSalary * 12
                };
            }
        }

        public decimal BonusEligible
        {
            get
            {
                if (_performanceHistory.Count == 0) return 0;
                double avgRating = _performanceHistory
                    .TakeLast(2)
                    .Average(p => p.Rating);
                return avgRating switch
                {
                    >= 4.5 => _baseSalary * 3,  // 3 เดือน
                    >= 3.5 => _baseSalary * 2,  // 2 เดือน
                    >= 2.5 => _baseSalary,       // 1 เดือน
                    _ => 0
                };
            }
        }

        // --- Constructors ---

        // Full constructor
        public Employee(string nationalId, string firstName, string lastName,
            DateTime birthDate, string email, Department department,
            decimal baseSalary, DateTime startDate,
            EmployeeType type = EmployeeType.FullTime)
        {
            // Validation
            if (string.IsNullOrWhiteSpace(nationalId))
                throw new ArgumentException("National ID is required.");
            if (string.IsNullOrWhiteSpace(firstName))
                throw new ArgumentException("First name is required.");
            if (string.IsNullOrWhiteSpace(lastName))
                throw new ArgumentException("Last name is required.");
            if (birthDate > DateTime.Today.AddYears(-18))
                throw new ArgumentException("Employee must be at least 18 years old.");
            if (!email.Contains('@'))
                throw new ArgumentException("Invalid email format.");
            if (baseSalary < 0)
                throw new ArgumentException("Salary cannot be negative.");
            if (startDate > DateTime.Today)
                throw new ArgumentException("Start date cannot be in the future.");

            // Assignment
            EmployeeId = _nextEmployeeId++;
            NationalId = nationalId;
            FirstName = firstName.Trim();
            LastName = lastName.Trim();
            BirthDate = birthDate.Date;
            Email = email.ToLower().Trim();
            Department = department;
            _baseSalary = baseSalary;
            StartDate = startDate.Date;
            EmployeeType = type;

            _totalEmployees++;
        }

        // Convenience: เริ่มงานวันนี้
        public Employee(string nationalId, string firstName, string lastName,
            DateTime birthDate, string email, Department department, decimal baseSalary)
            : this(nationalId, firstName, lastName, birthDate, email,
                  department, baseSalary, DateTime.Today) { }

        // Static constructor
        static Employee()
        {
            Console.WriteLine("Employee system initialized");
        }

        // --- Methods ---

        public void AddPerformanceReview(int year, double rating, string comments)
        {
            if (rating < 1 || rating > 5)
                throw new ArgumentOutOfRangeException(nameof(rating), "Rating must be 1-5");

            _performanceHistory.Add(new PerformanceRecord(year, rating, comments));
        }

        public double GetAverageRating()
        {
            if (_performanceHistory.Count == 0) return 0;
            return _performanceHistory.Average(p => p.Rating);
        }

        public void Promote(decimal salaryIncrease, Department? newDepartment = null)
        {
            BaseSalary += salaryIncrease;
            if (newDepartment.HasValue)
                Department = newDepartment.Value;
            Console.WriteLine($"Promoted {FullName}: +{salaryIncrease:N0} บาท → {_baseSalary:N0} บาท/เดือน");
        }

        // Private helper
        private static int CalculateAge(DateTime date)
        {
            var today = DateTime.Today;
            int age = today.Year - date.Year;
            if (date.Date > today.AddYears(-age)) age--;
            return age;
        }

        // IDisposable
        public void Dispose()
        {
            if (!_disposed)
            {
                _totalEmployees--;
                _disposed = true;
                GC.SuppressFinalize(this);
            }
        }

        ~Employee()
        {
            Dispose();
        }

        public override string ToString()
            => $"[{EmployeeId}] {FullName} | {Department} | {EmployeeType} | {_baseSalary:N0} บาท/เดือน";

        public string GetDetailReport()
        {
            string performanceSummary = _performanceHistory.Count > 0
                ? $"{GetAverageRating():F1}/5.0 ({_performanceHistory.Count} reviews)"
                : "ยังไม่มีการประเมิน";

            return $"""
                === Employee Profile ===
                ID         : {EmployeeId}
                ชื่อ-นามสกุล: {FullName}
                บัตรประชาชน : {NationalId}
                อายุ        : {Age} ปี (เกิด {BirthDate:dd/MM/yyyy})
                Email      : {Email}
                โทรศัพท์    : {Phone ?? "ไม่ระบุ"}
                แผนก       : {Department}
                ประเภท     : {EmployeeType}
                เงินเดือน  : {_baseSalary:N0} บาท/เดือน
                เงินปี      : {AnnualSalary:N0} บาท/ปี
                โบนัส      : {BonusEligible:N0} บาท
                เริ่มงาน    : {StartDate:dd/MM/yyyy} ({YearsOfService} ปี)
                ที่อยู่     : {HomeAddress?.ToString() ?? "ไม่ระบุ"}
                ผลงาน      : {performanceSummary}
                """;
        }
    }

    public record PerformanceRecord(int Year, double Rating, string Comments);

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("=== ระบบข้อมูลพนักงาน ===\n");

            // สร้างพนักงาน
            var emp1 = new Employee(
                "1234567890123",
                "สมชาย", "ใจดี",
                new DateTime(1990, 5, 15),
                "somchai@company.com",
                Department.Engineering,
                55000m,
                new DateTime(2020, 1, 1));

            emp1.Phone = "0812345678";
            emp1.HomeAddress = new Address("123/45 ถ.สุขุมวิท", "กรุงเทพมหานคร",
                "กรุงเทพมหานคร", "10110");
            emp1.EmployeeType = EmployeeType.FullTime;

            // เพิ่มประวัติประเมิน
            emp1.AddPerformanceReview(2022, 4.0, "ทำงานได้ดี");
            emp1.AddPerformanceReview(2023, 4.5, "ผลงานดีเยี่ยม");
            emp1.AddPerformanceReview(2024, 4.8, "เกินคาด");

            // แสดงรายละเอียด
            Console.WriteLine(emp1.GetDetailReport());

            // เลื่อนตำแหน่ง
            emp1.Promote(10000m, Department.Management);
            Console.WriteLine($"\nหลังเลื่อนตำแหน่ง: {emp1}");

            Console.WriteLine($"\nพนักงานทั้งหมด: {Employee.TotalEmployees} คน");

            // ทดสอบ validation
            Console.WriteLine("\n=== ทดสอบ Validation ===");
            try
            {
                var invalid = new Employee("", "Test", "User",
                    DateTime.Today, "test@email.com",
                    Department.Sales, 30000m);
            }
            catch (ArgumentException ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }

            // IDisposable
            using var tempEmp = new Employee(
                "9999999999999", "ชั่วคราว", "พนักงาน",
                new DateTime(1995, 1, 1), "temp@company.com",
                Department.Operations, 25000m);
            Console.WriteLine($"\nสร้างพนักงานชั่วคราว: {tempEmp.FullName}");
            Console.WriteLine($"พนักงานทั้งหมด: {Employee.TotalEmployees} คน");
            // Dispose() ถูกเรียกอัตโนมัติเมื่อจบ using block
        }
    }
}
```

---

## Exercises

### Exercise 1: Vehicle Class Hierarchy
```csharp
// TODO: สร้าง class Vehicle ที่มี constructors หลายแบบ:
// Vehicle(string make, string model) - ปีปัจจุบัน
// Vehicle(string make, string model, int year) - กำหนดปี
// Vehicle(string make, string model, int year, string color, decimal price)
// ใช้ constructor chaining
// Properties: Make, Model, Year, Color, Price, Age (computed)
// Method: ToString() ที่สวยงาม

public class Vehicle
{
    // TODO: Implement with constructor chaining
}
```

### Exercise 2: Matrix
```csharp
// TODO: สร้าง class Matrix ที่มี:
// Matrix(int rows, int cols) - matrix ว่าง
// Matrix(int[,] data) - จาก 2D array
// Matrix(int size) - square matrix (identity)
// Properties: Rows, Cols, IsSquare, Determinant (2x2 เท่านั้น)
// Methods: Add(Matrix), Multiply(Matrix), Transpose()
// Indexer: this[int row, int col]

public class Matrix
{
    // TODO: Implement
}
```

### Exercise 3: Configuration Builder
```csharp
// TODO: สร้าง class AppConfig ที่ใช้ Builder pattern:
// - Private constructor
// - Static Builder class ด้านใน
// - Builder methods: WithDatabase, WithServer, WithTimeout, WithAuth
// - Build() method คืน AppConfig
// ตัวอย่างการใช้:
// var config = AppConfig.Builder
//     .WithDatabase("localhost", "mydb")
//     .WithServer("0.0.0.0", 8080)
//     .WithTimeout(30)
//     .Build();

public class AppConfig
{
    // TODO: Implement Builder pattern
}
```

---

## สรุป

✅ Constructor เรียกอัตโนมัติเมื่อสร้าง object ด้วย `new`  
✅ Default constructor สร้างให้อัตโนมัติถ้าไม่มี explicit constructor  
✅ Constructor overloading ทำให้สร้าง object ด้วยวิธีต่างๆ ได้  
✅ Constructor chaining ด้วย `this()` หลีกเลี่ยง duplicate code  
✅ Object initializers ทำให้โค้ดอ่านง่ายขึ้น  
✅ Static constructor เรียกครั้งเดียวก่อน class ใช้งานครั้งแรก  
✅ Destructor/Finalizer ใช้สำหรับ unmanaged resources (นิยมใช้ IDisposable แทน)  
✅ `using` statement/declaration รับประกันการเรียก `Dispose()`  

## Part ถัดไป
**Part 015: Inheritance (การสืบทอด)** - เรียนรู้ base/derived classes, base keyword, method overriding vs hiding และ sealed class

---
*Part 014/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
