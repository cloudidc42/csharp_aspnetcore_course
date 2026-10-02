# Part 024: LINQ พื้นฐาน

## เนื้อหาใน Part นี้
- LINQ คืออะไร และทำงานอย่างไร
- Query Syntax vs Method Syntax
- Where, Select, OrderBy, ThenBy
- GroupBy
- First, FirstOrDefault, Single, Last
- Count, Sum, Average, Min, Max
- Any, All, Contains
- Take, Skip, TakeWhile, SkipWhile
- โปรแกรมตัวอย่าง: Student Grades Analysis

---

## 1. LINQ คืออะไร?

LINQ (Language Integrated Query) คือชุด methods และ syntax สำหรับ query ข้อมูลใน C# รองรับทั้ง in-memory collections, databases (EF Core), XML และอื่นๆ

```csharp
// ===== ก่อนมี LINQ =====
List<int> numbers = new List<int> { 5, 2, 8, 1, 9, 3, 7, 4, 6 };

// หาเลขคู่ > 4 แล้วเรียง
List<int> result = new List<int>();
foreach (int n in numbers)
{
    if (n % 2 == 0 && n > 4)
        result.Add(n);
}
result.Sort();

// ===== ด้วย LINQ =====
var linqResult = numbers
    .Where(n => n % 2 == 0 && n > 4)
    .OrderBy(n => n)
    .ToList();

Console.WriteLine(string.Join(", ", linqResult)); // 6, 8

// LINQ ทำงานบน IEnumerable<T>
// หลักการ: Deferred Execution - execute เมื่อ iterate จริง
```

---

## 2. Query Syntax vs Method Syntax

C# LINQ รองรับ 2 syntax ที่ให้ผลเหมือนกัน

```csharp
// Data
var students = new[]
{
    new { Name = "Alice", Score = 85, Subject = "Math" },
    new { Name = "Bob", Score = 72, Subject = "Science" },
    new { Name = "Charlie", Score = 90, Subject = "Math" },
    new { Name = "Diana", Score = 68, Subject = "Science" },
    new { Name = "Eve", Score = 95, Subject = "Math" },
};

// ===== Query Syntax (SQL-like) =====
var queryResult =
    from s in students
    where s.Score >= 80
    orderby s.Score descending
    select new { s.Name, s.Score };

// ===== Method Syntax (Fluent) =====
var methodResult = students
    .Where(s => s.Score >= 80)
    .OrderByDescending(s => s.Score)
    .Select(s => new { s.Name, s.Score });

// ทั้งสองให้ผลเหมือนกัน
foreach (var s in queryResult)
    Console.WriteLine($"{s.Name}: {s.Score}");
// Eve: 95
// Charlie: 90
// Alice: 85

// Query Syntax ที่ซับซ้อนกว่า
var complexQuery =
    from s in students
    where s.Score >= 70
    let grade = s.Score >= 90 ? "A" : s.Score >= 80 ? "B" : "C"
    orderby grade, s.Name
    select new { s.Name, s.Score, Grade = grade };

// ส่วนใหญ่นิยม Method Syntax เพราะ:
// - ใช้ร่วมกับ methods อื่นๆ ได้ดี
// - IDE support ดีกว่า
// - บาง operations มีเฉพาะ method syntax (Count, Sum, ฯลฯ)
```

---

## 3. Where - กรองข้อมูล

```csharp
var numbers = Enumerable.Range(1, 20).ToList();

// Where พื้นฐาน
var evens = numbers.Where(n => n % 2 == 0);
var over10 = numbers.Where(n => n > 10);

// Where ด้วยหลายเงื่อนไข
var special = numbers.Where(n => n % 2 == 0 && n > 5 && n < 15);
Console.WriteLine(string.Join(", ", special)); // 6, 8, 10, 12, 14

// Where กับ index
var atEvenPositions = numbers.Where((n, i) => i % 2 == 0);
Console.WriteLine(string.Join(", ", atEvenPositions)); // 1, 3, 5, 7, 9, 11, 13, 15, 17, 19

// Where กับ Object
var products = new[]
{
    new { Name = "Laptop", Price = 25000m, Category = "Electronics", InStock = true },
    new { Name = "Chair", Price = 3500m, Category = "Furniture", InStock = false },
    new { Name = "Phone", Price = 15000m, Category = "Electronics", InStock = true },
    new { Name = "Desk", Price = 8000m, Category = "Furniture", InStock = true },
    new { Name = "Tablet", Price = 12000m, Category = "Electronics", InStock = true },
};

var availableElectronics = products
    .Where(p => p.Category == "Electronics" && p.InStock && p.Price < 20000);

foreach (var p in availableElectronics)
    Console.WriteLine($"{p.Name}: {p.Price:C}");
// Phone: ฿15,000.00
// Tablet: ฿12,000.00
```

---

## 4. Select - แปลงข้อมูล (Projection)

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };

// Select พื้นฐาน
var doubled = numbers.Select(n => n * 2);
var asStrings = numbers.Select(n => n.ToString());

// Select ด้วย index
var indexed = numbers.Select((n, i) => $"{i}: {n}");
foreach (var s in indexed)
    Console.WriteLine(s);
// 0: 1
// 1: 2
// ...

// Select เป็น Anonymous Type
var products = new[]
{
    new { Name = "Apple", Price = 35m, Qty = 100 },
    new { Name = "Banana", Price = 15m, Qty = 200 },
    new { Name = "Cherry", Price = 120m, Qty = 50 }
};

var summary = products.Select(p => new
{
    p.Name,
    p.Price,
    TotalValue = p.Price * p.Qty,
    IsExpensive = p.Price > 50
});

foreach (var s in summary)
    Console.WriteLine($"{s.Name}: {s.TotalValue:C} (Expensive: {s.IsExpensive})");

// Select กับ Record
public record ProductDto(string Name, decimal Price);
var dtos = products.Select(p => new ProductDto(p.Name, p.Price));

// Select กับ Null Handling
var names = new string?[] { "Alice", null, "Bob", null, "Charlie" };
var validNames = names
    .Where(n => n != null)
    .Select(n => n!.ToUpper());
// หรือ
var validNames2 = names
    .OfType<string>() // กรอง null ออก
    .Select(n => n.ToUpper());
```

---

## 5. OrderBy, OrderByDescending, ThenBy

```csharp
var students = new[]
{
    new { Name = "Charlie", Score = 85, Age = 20 },
    new { Name = "Alice", Score = 85, Age = 22 },
    new { Name = "Bob", Score = 92, Age = 19 },
    new { Name = "Diana", Score = 78, Age = 21 },
    new { Name = "Eve", Score = 92, Age = 20 },
};

// OrderBy: เรียง ascending
var byScore = students.OrderBy(s => s.Score);

// OrderByDescending: เรียง descending
var byScoreDesc = students.OrderByDescending(s => s.Score);

// ThenBy: เรียงรองลงมา
var byScoreAndName = students
    .OrderByDescending(s => s.Score)
    .ThenBy(s => s.Name);

foreach (var s in byScoreAndName)
    Console.WriteLine($"{s.Name}: {s.Score}");
// Bob: 92
// Eve: 92
// Charlie: 85
// Alice: 85 (ชื่อ Alice มาก่อน Charlie เพราะ ThenBy Name)
// Diana: 78

// ThenByDescending
var complex = students
    .OrderByDescending(s => s.Score)
    .ThenBy(s => s.Name)
    .ThenByDescending(s => s.Age);

// OrderBy ด้วย string
var names = new[] { "Charlie", "alice", "Bob", "DIANA" };
var alphabetical = names.OrderBy(n => n, StringComparer.OrdinalIgnoreCase);
// alice, Bob, Charlie, DIANA

// เรียงตาม computed value
var scores = new[] { 45, 92, 67, 83, 71 };
var byDistance = scores.OrderBy(s => Math.Abs(s - 70)); // ใกล้ 70 ที่สุด
```

---

## 6. GroupBy

```csharp
var orders = new[]
{
    new { OrderId = 1, Customer = "Alice", Amount = 500m, Month = 1 },
    new { OrderId = 2, Customer = "Bob", Amount = 300m, Month = 1 },
    new { OrderId = 3, Customer = "Alice", Amount = 700m, Month = 2 },
    new { OrderId = 4, Customer = "Charlie", Amount = 200m, Month = 1 },
    new { OrderId = 5, Customer = "Bob", Amount = 900m, Month = 2 },
    new { OrderId = 6, Customer = "Alice", Amount = 400m, Month = 2 },
};

// GroupBy พื้นฐาน
var byCustomer = orders.GroupBy(o => o.Customer);

foreach (var group in byCustomer)
{
    Console.WriteLine($"\nCustomer: {group.Key}");
    Console.WriteLine($"  Orders: {group.Count()}");
    Console.WriteLine($"  Total: {group.Sum(o => o.Amount):C}");
    foreach (var order in group)
        Console.WriteLine($"  - Order #{order.OrderId}: {order.Amount:C}");
}

// GroupBy พร้อม projection
var customerStats = orders
    .GroupBy(o => o.Customer)
    .Select(g => new
    {
        Customer = g.Key,
        OrderCount = g.Count(),
        TotalAmount = g.Sum(o => o.Amount),
        AverageAmount = g.Average(o => o.Amount),
        MaxOrder = g.Max(o => o.Amount)
    })
    .OrderByDescending(s => s.TotalAmount);

foreach (var stat in customerStats)
    Console.WriteLine($"{stat.Customer}: {stat.OrderCount} orders, Total: {stat.TotalAmount:C}");

// GroupBy หลาย keys (Composite Key)
var byMonthAndCustomer = orders
    .GroupBy(o => new { o.Month, o.Customer })
    .Select(g => new
    {
        g.Key.Month,
        g.Key.Customer,
        Total = g.Sum(o => o.Amount)
    })
    .OrderBy(x => x.Month).ThenBy(x => x.Customer);

foreach (var x in byMonthAndCustomer)
    Console.WriteLine($"Month {x.Month}, {x.Customer}: {x.Total:C}");

// Query Syntax GroupBy
var grouped =
    from o in orders
    group o by o.Customer into g
    select new
    {
        Customer = g.Key,
        Total = g.Sum(o => o.Amount)
    };
```

---

## 7. First, FirstOrDefault, Single, Last

```csharp
var numbers = new[] { 3, 1, 4, 1, 5, 9, 2, 6, 5, 3 };

// First: หา element แรก (throw ถ้า empty)
int first = numbers.First();                    // 3
int firstOver5 = numbers.First(n => n > 5);   // 9

// FirstOrDefault: หา element แรก (คืน default ถ้าไม่พบ)
int? firstOver20 = numbers.FirstOrDefault(n => n > 20);  // null (int? default)
int firstOver20Int = numbers.FirstOrDefault(n => n > 20, -1); // -1 (C# 10+)

// Last: หา element สุดท้าย
int last = numbers.Last();                    // 3
int lastUnder5 = numbers.Last(n => n < 5);   // 3

// LastOrDefault
int? lastOver20 = numbers.LastOrDefault(n => n > 20);

// Single: หา element ที่ unique (throw ถ้ามีมากกว่า 1 หรือ empty)
try
{
    int unique = numbers.Single(n => n == 10); // throws: ไม่มี
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}

var onlyOneList = new[] { 42 };
int single = onlyOneList.Single(); // 42

// SingleOrDefault: คืน default ถ้า empty, throw ถ้ามีมากกว่า 1
int? singleOrDefault = numbers.SingleOrDefault(n => n == 99); // null
try
{
    int duplicate = numbers.Single(n => n == 5); // throws: มี 5 สองตัว
}
catch (InvalidOperationException) { }

// ElementAt: เข้าถึงด้วย index
int third = numbers.ElementAt(2); // 4

// ElementAtOrDefault: คืน default ถ้า index เกิน range
int? outOfRange = numbers.ElementAtOrDefault(100); // null

// ตัวอย่างจริง - หา product ราคาต่ำสุด
var products = new[]
{
    new { Name = "A", Price = 100m },
    new { Name = "B", Price = 50m },
    new { Name = "C", Price = 200m }
};

var cheapest = products.OrderBy(p => p.Price).First();
Console.WriteLine($"Cheapest: {cheapest.Name} at {cheapest.Price}"); // B at 50
```

---

## 8. Count, Sum, Average, Min, Max

```csharp
var scores = new[] { 85, 92, 67, 78, 95, 88, 71, 84, 90, 73 };

// Count
int total = scores.Count();                  // 10
int highCount = scores.Count(s => s >= 85);  // 4

// Sum
int sumAll = scores.Sum();                        // 823
double sumHigh = scores.Where(s => s >= 85).Sum(); // ใช้ Where ก่อน

// Average
double avg = scores.Average();               // 82.3
double avgHigh = scores.Where(s => s >= 85).Average();

// Min, Max
int min = scores.Min();                      // 67
int max = scores.Max();                      // 95
int maxOver80 = scores.Where(s => s > 80).Max(); // 95

// MinBy, MaxBy (C# 10+ / .NET 6+)
var students = new[]
{
    new { Name = "Alice", Score = 85 },
    new { Name = "Bob", Score = 92 },
    new { Name = "Charlie", Score = 67 }
};

var topStudent = students.MaxBy(s => s.Score);
var lowestStudent = students.MinBy(s => s.Score);
Console.WriteLine($"Top: {topStudent?.Name}");    // Bob
Console.WriteLine($"Lowest: {lowestStudent?.Name}"); // Charlie

// Aggregate: custom aggregation
int product = scores.Aggregate(1, (acc, n) => acc * n);
string csv = scores.Aggregate("", (acc, n) => acc + (acc == "" ? "" : ",") + n);

// การคำนวณ Statistics
double mean = scores.Average();
double variance = scores.Average(s => Math.Pow(s - mean, 2));
double stdDev = Math.Sqrt(variance);

Console.WriteLine($"Mean: {mean:F2}");
Console.WriteLine($"StdDev: {stdDev:F2}");
Console.WriteLine($"Min: {scores.Min()}, Max: {scores.Max()}");
Console.WriteLine($"Range: {scores.Max() - scores.Min()}");
```

---

## 9. Any, All, Contains

```csharp
var numbers = new[] { 2, 4, 6, 8, 10 };

// Any: มีอย่างน้อย 1 ตัวที่ตรงเงื่อนไข
bool hasOdd = numbers.Any(n => n % 2 != 0);  // false
bool hasEven = numbers.Any(n => n % 2 == 0); // true
bool isEmpty = numbers.Any();                  // true (มีข้อมูล)

// All: ทุกตัวต้องตรงเงื่อนไข
bool allEven = numbers.All(n => n % 2 == 0); // true
bool allUnder20 = numbers.All(n => n < 20);  // true
bool allOver5 = numbers.All(n => n > 5);     // false (2, 4 ไม่ตรง)

// Contains: มีค่านี้หรือไม่
bool has6 = numbers.Contains(6);             // true
bool has7 = numbers.Contains(7);             // false

// Contains ด้วย Custom Equality
var words = new[] { "Hello", "World", "LINQ" };
bool hasHello = words.Contains("hello", StringComparer.OrdinalIgnoreCase); // true

// ตัวอย่างจริง - Validation
var cart = new[]
{
    new { ProductId = 1, Quantity = 2, Price = 100m },
    new { ProductId = 2, Quantity = 0, Price = 50m },  // ของขาด
    new { ProductId = 3, Quantity = 1, Price = 200m },
};

bool hasOutOfStock = cart.Any(item => item.Quantity == 0);
bool allInStock = cart.All(item => item.Quantity > 0);
bool hasExpensive = cart.Any(item => item.Price > 150);

Console.WriteLine($"Has out-of-stock: {hasOutOfStock}"); // true
Console.WriteLine($"All in stock: {allInStock}");         // false

// Permission check
string[] userRoles = { "User", "Editor" };
string[] requiredRoles = { "Admin", "Editor" };

bool hasPermission = requiredRoles.Any(r => userRoles.Contains(r));
Console.WriteLine($"Has permission: {hasPermission}"); // true
```

---

## 10. Take, Skip, TakeWhile, SkipWhile

```csharp
var numbers = Enumerable.Range(1, 20).ToList();

// Take: เอา N ตัวแรก
var first5 = numbers.Take(5);
Console.WriteLine(string.Join(", ", first5)); // 1, 2, 3, 4, 5

// Take ด้วย Range (C# 8+)
var middle = numbers.Take(5..10);              // index 5 ถึง 9
var last3 = numbers.Take(^3..);               // 3 ตัวสุดท้าย
Console.WriteLine(string.Join(", ", last3));   // 18, 19, 20

// Skip: ข้าม N ตัวแรก
var skip5 = numbers.Skip(5);
Console.WriteLine(string.Join(", ", skip5.Take(5))); // 6, 7, 8, 9, 10

// Pagination
int pageSize = 5;
int pageNumber = 2; // 0-based: หน้าที่ 3
var page = numbers.Skip(pageNumber * pageSize).Take(pageSize);
Console.WriteLine($"Page {pageNumber + 1}: {string.Join(", ", page)}"); // Page 3: 11, 12, 13, 14, 15

// TakeWhile: เอาตราบเท่าที่เงื่อนไขเป็น true
var whileUnder10 = numbers.TakeWhile(n => n < 10);
Console.WriteLine(string.Join(", ", whileUnder10)); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// SkipWhile: ข้ามตราบเท่าที่เงื่อนไขเป็น true
var afterFirst10 = numbers.SkipWhile(n => n <= 10);
Console.WriteLine(string.Join(", ", afterFirst10.Take(5))); // 11, 12, 13, 14, 15

// Chunk (C# 6+ / .NET 6+): แบ่งเป็นก้อนๆ
var chunks = numbers.Chunk(4);
foreach (int[] chunk in chunks)
    Console.WriteLine(string.Join(", ", chunk));
// 1, 2, 3, 4
// 5, 6, 7, 8
// ...
```

---

## 11. Chaining LINQ Operations

```csharp
// LINQ เก่งที่สุดเมื่อ chain หลาย operations
var data = new[]
{
    new { Name = "Alice", Dept = "HR", Salary = 45000m, Years = 3 },
    new { Name = "Bob", Dept = "IT", Salary = 65000m, Years = 7 },
    new { Name = "Charlie", Dept = "IT", Salary = 55000m, Years = 4 },
    new { Name = "Diana", Dept = "HR", Salary = 50000m, Years = 5 },
    new { Name = "Eve", Dept = "IT", Salary = 72000m, Years = 10 },
    new { Name = "Frank", Dept = "Finance", Salary = 60000m, Years = 6 },
    new { Name = "Grace", Dept = "HR", Salary = 42000m, Years = 2 },
};

// Complex query: IT employees with > 5 years, ranked by salary
var itSeniors = data
    .Where(e => e.Dept == "IT" && e.Years > 5)
    .OrderByDescending(e => e.Salary)
    .Select((e, i) => new
    {
        Rank = i + 1,
        e.Name,
        e.Salary,
        e.Years,
        Bonus = e.Salary * 0.1m
    });

Console.WriteLine("IT Senior Employees:");
foreach (var e in itSeniors)
    Console.WriteLine($"  {e.Rank}. {e.Name}: {e.Salary:C}, Bonus: {e.Bonus:C}");

// Department summary
var deptSummary = data
    .GroupBy(e => e.Dept)
    .Select(g => new
    {
        Department = g.Key,
        HeadCount = g.Count(),
        AvgSalary = g.Average(e => e.Salary),
        TotalPayroll = g.Sum(e => e.Salary),
        AvgYears = g.Average(e => e.Years)
    })
    .OrderByDescending(s => s.TotalPayroll);

Console.WriteLine("\nDepartment Summary:");
foreach (var d in deptSummary)
    Console.WriteLine($"  {d.Department}: {d.HeadCount} people, " +
        $"Avg: {d.AvgSalary:C}, Total: {d.TotalPayroll:C}");
```

---

## 12. โปรแกรมตัวอย่าง: Student Grades Analysis

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace StudentAnalysis
{
    public record Student(
        int Id,
        string Name,
        string Major,
        int Year,
        List<(string Subject, double Score)> Grades
    );

    public class GradeAnalyzer
    {
        private readonly List<Student> _students;

        public GradeAnalyzer(List<Student> students)
        {
            _students = students;
        }

        // คะแนนเฉลี่ยของแต่ละนักเรียน
        public IEnumerable<(string Name, double Average)> GetStudentAverages()
        {
            return _students
                .Select(s => (
                    s.Name,
                    Average: s.Grades.Average(g => g.Score)
                ))
                .OrderByDescending(x => x.Average);
        }

        // Top N นักเรียน
        public IEnumerable<Student> GetTopStudents(int n = 5)
        {
            return _students
                .OrderByDescending(s => s.Grades.Average(g => g.Score))
                .Take(n);
        }

        // นักเรียนที่ตกวิชาใดวิชาหนึ่ง (< 50)
        public IEnumerable<(string Name, IEnumerable<string> FailedSubjects)> GetFailingStudents()
        {
            return _students
                .Select(s => (
                    s.Name,
                    FailedSubjects: s.Grades
                        .Where(g => g.Score < 50)
                        .Select(g => g.Subject)
                ))
                .Where(x => x.FailedSubjects.Any());
        }

        // สถิติตาม Major
        public IEnumerable<object> GetMajorStats()
        {
            return _students
                .GroupBy(s => s.Major)
                .Select(g => new
                {
                    Major = g.Key,
                    StudentCount = g.Count(),
                    AverageScore = g.Average(s => s.Grades.Average(gr => gr.Score)),
                    TopStudent = g.MaxBy(s => s.Grades.Average(gr => gr.Score))?.Name,
                    PassRate = g.Count(s => s.Grades.All(gr => gr.Score >= 50)) * 100.0 / g.Count()
                })
                .OrderByDescending(m => m.AverageScore)
                .Cast<object>();
        }

        // คะแนนเฉลี่ยตามวิชา
        public IEnumerable<(string Subject, double Average, int StudentCount)> GetSubjectStats()
        {
            return _students
                .SelectMany(s => s.Grades.Select(g => (s.Name, g.Subject, g.Score)))
                .GroupBy(x => x.Subject)
                .Select(g => (
                    Subject: g.Key,
                    Average: g.Average(x => x.Score),
                    StudentCount: g.Count()
                ))
                .OrderBy(x => x.Subject);
        }

        // Distribution: Grade A/B/C/D/F
        public Dictionary<string, int> GetGradeDistribution()
        {
            return _students
                .SelectMany(s => s.Grades)
                .Select(g => g.Score >= 80 ? "A" :
                             g.Score >= 70 ? "B" :
                             g.Score >= 60 ? "C" :
                             g.Score >= 50 ? "D" : "F")
                .GroupBy(grade => grade)
                .ToDictionary(g => g.Key, g => g.Count());
        }

        // นักเรียนที่คะแนนดีขึ้นทุกวิชา (เรียง by subject)
        public IEnumerable<string> GetImprovedStudents()
        {
            return _students
                .Where(s => s.Grades.Count >= 2)
                .Where(s =>
                {
                    var scores = s.Grades.Select(g => g.Score).ToList();
                    return scores.Zip(scores.Skip(1), (a, b) => b > a).All(x => x);
                })
                .Select(s => s.Name);
        }

        // Report
        public void PrintReport()
        {
            Console.WriteLine("╔══════════════════════════════════════╗");
            Console.WriteLine("║      STUDENT GRADE REPORT            ║");
            Console.WriteLine("╚══════════════════════════════════════╝\n");

            // Overall stats
            var allScores = _students.SelectMany(s => s.Grades.Select(g => g.Score)).ToList();
            Console.WriteLine("Overall Statistics:");
            Console.WriteLine($"  Total Students: {_students.Count}");
            Console.WriteLine($"  Total Grades: {allScores.Count}");
            Console.WriteLine($"  Overall Average: {allScores.Average():F2}");
            Console.WriteLine($"  Highest Score: {allScores.Max():F0}");
            Console.WriteLine($"  Lowest Score: {allScores.Min():F0}");

            // Student averages
            Console.WriteLine("\nStudent Rankings:");
            int rank = 1;
            foreach (var (name, avg) in GetStudentAverages())
                Console.WriteLine($"  {rank++,2}. {name,-15} {avg:F2}");

            // Failing students
            var failing = GetFailingStudents().ToList();
            if (failing.Any())
            {
                Console.WriteLine("\nStudents with Failing Grades:");
                foreach (var (name, subjects) in failing)
                    Console.WriteLine($"  {name}: {string.Join(", ", subjects)}");
            }

            // Major stats
            Console.WriteLine("\nStats by Major:");
            foreach (dynamic m in GetMajorStats())
            {
                Console.WriteLine($"  {m.Major}:");
                Console.WriteLine($"    Students: {m.StudentCount}");
                Console.WriteLine($"    Average: {m.AverageScore:F2}");
                Console.WriteLine($"    Top Student: {m.TopStudent}");
                Console.WriteLine($"    Pass Rate: {m.PassRate:F1}%");
            }

            // Subject stats
            Console.WriteLine("\nStats by Subject:");
            foreach (var (subject, avg, count) in GetSubjectStats())
                Console.WriteLine($"  {subject,-15} Avg: {avg:F2} ({count} grades)");

            // Grade distribution
            Console.WriteLine("\nGrade Distribution:");
            var dist = GetGradeDistribution();
            foreach (var grade in new[] { "A", "B", "C", "D", "F" })
            {
                int count = dist.GetValueOrDefault(grade, 0);
                string bar = new string('█', count);
                Console.WriteLine($"  {grade}: {bar} ({count})");
            }
        }
    }

    class Program
    {
        static void Main()
        {
            var students = new List<Student>
            {
                new(1, "Alice", "Computer Science", 3, new()
                {
                    ("Math", 92), ("Programming", 95), ("Database", 88),
                    ("Networks", 85), ("AI", 90)
                }),
                new(2, "Bob", "Computer Science", 2, new()
                {
                    ("Math", 78), ("Programming", 82), ("Database", 75),
                    ("Networks", 80), ("AI", 77)
                }),
                new(3, "Charlie", "Mathematics", 4, new()
                {
                    ("Math", 98), ("Statistics", 95), ("Calculus", 92),
                    ("Linear Algebra", 89), ("Discrete Math", 94)
                }),
                new(4, "Diana", "Computer Science", 1, new()
                {
                    ("Math", 65), ("Programming", 70), ("Database", 45),
                    ("Networks", 60), ("AI", 55)
                }),
                new(5, "Eve", "Mathematics", 3, new()
                {
                    ("Math", 88), ("Statistics", 82), ("Calculus", 85),
                    ("Linear Algebra", 79), ("Discrete Math", 91)
                }),
                new(6, "Frank", "Physics", 2, new()
                {
                    ("Physics", 90), ("Math", 88), ("Chemistry", 75),
                    ("Lab", 82), ("Theory", 85)
                }),
                new(7, "Grace", "Computer Science", 3, new()
                {
                    ("Math", 55), ("Programming", 48), ("Database", 52),
                    ("Networks", 40), ("AI", 50)
                }),
            };

            var analyzer = new GradeAnalyzer(students);
            analyzer.PrintReport();

            // Extra queries
            Console.WriteLine("\n=== Extra Queries ===");

            // นักเรียนที่ได้ A ทุกวิชา
            var allA = students
                .Where(s => s.Grades.All(g => g.Score >= 80))
                .Select(s => s.Name);
            Console.WriteLine($"\nAll-A students: {string.Join(", ", allA)}");

            // วิชาที่ยากที่สุด (คะแนนเฉลี่ยต่ำสุด)
            var hardestSubject = students
                .SelectMany(s => s.Grades)
                .GroupBy(g => g.Subject)
                .MinBy(g => g.Average(x => x.Score));
            Console.WriteLine($"Hardest subject: {hardestSubject?.Key} " +
                $"(avg: {hardestSubject?.Average(x => x.Score):F2})");

            // แจกแจง students ตาม year
            var byYear = students
                .GroupBy(s => s.Year)
                .OrderBy(g => g.Key)
                .Select(g => $"Year {g.Key}: {string.Join(", ", g.Select(s => s.Name))}");

            Console.WriteLine("\nStudents by Year:");
            foreach (var line in byYear)
                Console.WriteLine($"  {line}");
        }
    }
}
```

---

## Exercises

### Exercise 1: Product Catalog
ใช้ LINQ กับ products list:
- หาสินค้าราคาต่ำกว่า 1000 เรียงตามราคา
- คำนวณราคาเฉลี่ยตาม category
- หา 3 สินค้าที่ขายดีที่สุด

### Exercise 2: Text Analysis
วิเคราะห์ข้อความด้วย LINQ:
- นับคำทั้งหมด
- หา 10 คำที่ใช้บ่อยที่สุด
- หาประโยคที่ยาวที่สุด/สั้นที่สุด

### Exercise 3: Sales Report
ใช้ LINQ สร้าง report จากข้อมูลการขาย:
- ยอดขายรายเดือน
- Top 5 ลูกค้า
- สินค้าที่ขายได้มากที่สุด

---

## สรุป

- ✅ LINQ ช่วยให้ query ข้อมูลได้สะดวก อ่านง่าย และ type-safe
- ✅ Query Syntax (from...where...select) เหมือน SQL
- ✅ Method Syntax (fluent) ใช้งานง่าย เชื่อมต่อกับ IDE ดีกว่า
- ✅ `Where` กรอง, `Select` แปลง, `OrderBy/ThenBy` เรียง
- ✅ `GroupBy` จัดกลุ่ม ใช้คู่กับ aggregation ได้ดี
- ✅ `First/Last/Single` หา element เฉพาะ - `OrDefault` version ปลอดภัยกว่า
- ✅ `Count, Sum, Average, Min, Max` สำหรับ aggregation
- ✅ `Any/All/Contains` สำหรับ checking
- ✅ `Take/Skip` สำหรับ pagination
- ✅ LINQ ใช้ Deferred Execution - execute เมื่อ iterate จริง

## Part ถัดไป
**Part 025** จะพูดถึง LINQ ขั้นสูง: Join, GroupJoin, SelectMany, Aggregate, Set operations และ Deferred vs Immediate execution

---
*Part 024/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
