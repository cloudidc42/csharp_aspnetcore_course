# Part 025: LINQ ขั้นสูง

## เนื้อหาใน Part นี้
- Join และ GroupJoin
- SelectMany - Flatten nested collections
- Aggregate - Custom aggregation
- Distinct, Except, Intersect, Union
- Zip
- Let, Into ใน Query Syntax
- ToList, ToArray, ToDictionary, ToLookup, ToHashSet
- Deferred vs Immediate Execution
- โปรแกรมตัวอย่าง: Product Inventory LINQ Queries

---

## 1. Join

Join เชื่อม 2 sequences ด้วย key ที่ตรงกัน (เหมือน INNER JOIN ใน SQL)

```csharp
// Data
var customers = new[]
{
    new { Id = 1, Name = "Alice", City = "Bangkok" },
    new { Id = 2, Name = "Bob", City = "Chiang Mai" },
    new { Id = 3, Name = "Charlie", City = "Phuket" },
    new { Id = 4, Name = "Diana", City = "Bangkok" },
};

var orders = new[]
{
    new { OrderId = 101, CustomerId = 1, Amount = 500m, Product = "Laptop" },
    new { OrderId = 102, CustomerId = 2, Amount = 200m, Product = "Mouse" },
    new { OrderId = 103, CustomerId = 1, Amount = 350m, Product = "Keyboard" },
    new { OrderId = 104, CustomerId = 3, Amount = 800m, Product = "Monitor" },
    new { OrderId = 105, CustomerId = 1, Amount = 150m, Product = "USB Hub" },
    new { OrderId = 106, CustomerId = 2, Amount = 1200m, Product = "Tablet" },
};

// Method Syntax Join
var customerOrders = customers
    .Join(
        orders,
        c => c.Id,          // key จาก customers
        o => o.CustomerId,   // key จาก orders
        (c, o) => new        // result selector
        {
            c.Name,
            c.City,
            o.OrderId,
            o.Product,
            o.Amount
        }
    )
    .OrderBy(x => x.Name)
    .ThenBy(x => x.OrderId);

foreach (var r in customerOrders)
    Console.WriteLine($"{r.Name} ({r.City}): Order #{r.OrderId} - {r.Product} {r.Amount:C}");

// Query Syntax Join
var queryJoin =
    from c in customers
    join o in orders on c.Id equals o.CustomerId
    orderby c.Name, o.OrderId
    select new { c.Name, o.Product, o.Amount };

// Join หลาย keys (Composite Key Join)
var sales = new[]
{
    new { Year = 2024, Month = 1, Region = "North", Amount = 100000m },
    new { Year = 2024, Month = 1, Region = "South", Amount = 85000m },
    new { Year = 2024, Month = 2, Region = "North", Amount = 120000m },
};

var targets = new[]
{
    new { Year = 2024, Month = 1, Region = "North", Target = 90000m },
    new { Year = 2024, Month = 1, Region = "South", Target = 80000m },
    new { Year = 2024, Month = 2, Region = "North", Target = 110000m },
};

var performance = sales.Join(
    targets,
    s => new { s.Year, s.Month, s.Region },  // composite key
    t => new { t.Year, t.Month, t.Region },
    (s, t) => new
    {
        s.Year, s.Month, s.Region,
        s.Amount, t.Target,
        Achievement = s.Amount / t.Target * 100
    }
);

foreach (var p in performance)
    Console.WriteLine($"{p.Year}/{p.Month:D2} {p.Region}: {p.Amount:C} / {p.Target:C} = {p.Achievement:F1}%");
```

---

## 2. GroupJoin (Left Outer Join)

GroupJoin รวม sequence แบบ LEFT JOIN ให้ทุก element ฝั่งซ้ายปรากฏ แม้ไม่มี match

```csharp
// GroupJoin = Left Outer Join
var allCustomerOrders = customers
    .GroupJoin(
        orders,
        c => c.Id,
        o => o.CustomerId,
        (c, orderGroup) => new
        {
            Customer = c.Name,
            Orders = orderGroup.ToList(),
            OrderCount = orderGroup.Count(),
            TotalAmount = orderGroup.Sum(o => o.Amount)
        }
    )
    .OrderByDescending(x => x.TotalAmount);

foreach (var r in allCustomerOrders)
{
    Console.WriteLine($"\n{r.Customer}: {r.OrderCount} orders, Total: {r.TotalAmount:C}");
    foreach (var o in r.Orders)
        Console.WriteLine($"  - {o.Product}: {o.Amount:C}");
}
// Diana จะปรากฏด้วยแม้ไม่มี order

// Left Outer Join ด้วย GroupJoin + SelectMany
var leftJoin = customers
    .GroupJoin(
        orders,
        c => c.Id,
        o => o.CustomerId,
        (c, orderGroup) => new { Customer = c, Orders = orderGroup }
    )
    .SelectMany(
        x => x.Orders.DefaultIfEmpty(),
        (x, o) => new
        {
            CustomerName = x.Customer.Name,
            Product = o?.Product ?? "No orders",
            Amount = o?.Amount ?? 0
        }
    );

foreach (var r in leftJoin)
    Console.WriteLine($"{r.CustomerName}: {r.Product} ({r.Amount:C})");
```

---

## 3. SelectMany - Flatten Nested Collections

SelectMany ใช้ "flatten" หรือ project หลาย collections เป็น sequence เดียว

```csharp
// SelectMany พื้นฐาน
var classrooms = new[]
{
    new { Class = "A", Students = new[] { "Alice", "Alex", "Amy" } },
    new { Class = "B", Students = new[] { "Bob", "Betty" } },
    new { Class = "C", Students = new[] { "Charlie", "Clara", "Carl" } },
};

// ดึงนักเรียนทั้งหมดเป็น flat list
var allStudents = classrooms.SelectMany(c => c.Students);
Console.WriteLine(string.Join(", ", allStudents));
// Alice, Alex, Amy, Bob, Betty, Charlie, Clara, Carl

// SelectMany พร้อม result selector
var studentInfo = classrooms
    .SelectMany(
        c => c.Students,
        (c, student) => new { c.Class, Student = student }
    );

foreach (var s in studentInfo)
    Console.WriteLine($"Class {s.Class}: {s.Student}");

// SelectMany กับ Optional (Null handling)
var data = new[]
{
    new { Name = "Alice", Tags = new[] { "C#", "Azure", "SQL" } },
    new { Name = "Bob", Tags = new[] { "Python", "ML" } },
    new { Name = "Charlie", Tags = new[] { "C#", "Docker" } },
};

// ดึง tags ทั้งหมด
var allTags = data.SelectMany(d => d.Tags);
Console.WriteLine(string.Join(", ", allTags));

// หานักเรียนที่รู้ C#
var csharpDevs = data
    .Where(d => d.Tags.Contains("C#"))
    .Select(d => d.Name);
Console.WriteLine("C# developers: " + string.Join(", ", csharpDevs));

// Flatten ซ้อนกัน
var matrix = new[]
{
    new[] { 1, 2, 3 },
    new[] { 4, 5, 6 },
    new[] { 7, 8, 9 }
};

var flat = matrix.SelectMany(row => row);
Console.WriteLine(string.Join(", ", flat)); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// Cartesian Product ด้วย SelectMany
var colors = new[] { "Red", "Green", "Blue" };
var sizes = new[] { "S", "M", "L" };

var combinations = colors.SelectMany(
    c => sizes,
    (color, size) => $"{color}-{size}"
);

foreach (string combo in combinations)
    Console.Write($"{combo} ");
// Red-S Red-M Red-L Green-S ...
```

---

## 4. Aggregate

Aggregate คือ custom fold/reduce operation

```csharp
var numbers = new[] { 1, 2, 3, 4, 5 };

// Aggregate พื้นฐาน: sum
int sum = numbers.Aggregate((acc, n) => acc + n); // 15

// Aggregate ด้วย seed
int sumFrom100 = numbers.Aggregate(100, (acc, n) => acc + n); // 115

// Aggregate ด้วย result selector
string result = numbers.Aggregate(
    new System.Text.StringBuilder(),
    (sb, n) => { sb.Append(n); return sb; },
    sb => sb.ToString()
);
Console.WriteLine(result); // 12345

// ตัวอย่างจริง: Running total
var expenses = new[] { 100m, 250m, 75m, 300m, 150m };
var runningTotal = expenses.Aggregate(
    new List<(decimal Expense, decimal Total)>(),
    (acc, e) =>
    {
        decimal total = acc.LastOrDefault().Total + e;
        acc.Add((e, total));
        return acc;
    }
);

decimal runningSum = 0;
foreach (var (expense, total) in runningTotal)
    Console.WriteLine($"Expense: {expense:C}, Running Total: {total:C}");

// Factorial
int n = 10;
long factorial = Enumerable.Range(1, n).Aggregate(1L, (acc, i) => acc * i);
Console.WriteLine($"{n}! = {factorial}"); // 3628800

// Most frequent word
string text = "the quick brown fox the fox jumps the";
string mostFrequent = text.Split(' ')
    .GroupBy(w => w)
    .Aggregate(
        (maxGroup: (IGrouping<string, string>?)null, maxCount: 0),
        (acc, g) => g.Count() > acc.maxCount
            ? (g, g.Count())
            : acc,
        acc => acc.maxGroup?.Key ?? ""
    );
Console.WriteLine($"Most frequent: {mostFrequent}"); // the
```

---

## 5. Set Operations: Distinct, Except, Intersect, Union

```csharp
var set1 = new[] { 1, 2, 3, 4, 5, 3, 2 };
var set2 = new[] { 3, 4, 5, 6, 7 };

// Distinct: ลบ duplicates
var unique = set1.Distinct();
Console.WriteLine(string.Join(", ", unique)); // 1, 2, 3, 4, 5

// Union: รวม (ไม่ซ้ำ)
var union = set1.Union(set2);
Console.WriteLine(string.Join(", ", union.OrderBy(x => x))); // 1, 2, 3, 4, 5, 6, 7

// Intersect: ตัดกัน
var intersect = set1.Intersect(set2);
Console.WriteLine(string.Join(", ", intersect)); // 3, 4, 5

// Except: ผลต่าง (มีใน set1 แต่ไม่มีใน set2)
var except = set1.Except(set2);
Console.WriteLine(string.Join(", ", except)); // 1, 2

// Distinct กับ Complex Objects
var products = new[]
{
    new { Id = 1, Name = "Apple" },
    new { Id = 2, Name = "Banana" },
    new { Id = 1, Name = "Apple" }, // duplicate
    new { Id = 3, Name = "Cherry" }
};

// DistinctBy (C# 10+ / .NET 6+)
var distinctProducts = products.DistinctBy(p => p.Id);
foreach (var p in distinctProducts)
    Console.WriteLine($"{p.Id}: {p.Name}");

// UnionBy, IntersectBy, ExceptBy (C# 10+)
var newProducts = new[]
{
    new { Id = 2, Name = "Banana" }, // same id
    new { Id = 4, Name = "Durian" }  // new
};

var unionById = products.UnionBy(newProducts, p => p.Id);
// Id: 1, 2, 3, 4 (no duplicates by Id)

// ตัวอย่างจริง: หา common skills
var teamA = new[] { "C#", "Azure", "SQL", "JavaScript" };
var teamB = new[] { "Python", "C#", "Docker", "SQL" };

var commonSkills = teamA.Intersect(teamB);
var allSkills = teamA.Union(teamB);
var aOnly = teamA.Except(teamB);
var bOnly = teamB.Except(teamA);

Console.WriteLine("Common: " + string.Join(", ", commonSkills)); // C#, SQL
Console.WriteLine("All: " + string.Join(", ", allSkills));
Console.WriteLine("A only: " + string.Join(", ", aOnly));
Console.WriteLine("B only: " + string.Join(", ", bOnly));
```

---

## 6. Zip

Zip รวม 2 sequences เข้าด้วยกัน element by element

```csharp
var names = new[] { "Alice", "Bob", "Charlie" };
var scores = new[] { 85, 92, 78 };
var grades = new[] { "B", "A", "C" };

// Zip 2 sequences
var combined = names.Zip(scores, (n, s) => new { Name = n, Score = s });
foreach (var x in combined)
    Console.WriteLine($"{x.Name}: {x.Score}");

// Zip 3 sequences (C# 9+ / .NET 5+)
var tripleZip = names.Zip(scores).Zip(grades,
    (pair, g) => new { Name = pair.First, Score = pair.Second, Grade = g });

// หรือใช้ Enumerable.Zip ใหม่
var triple = names.Zip(scores, grades);
foreach (var (name, score, grade) in triple)
    Console.WriteLine($"{name}: {score} ({grade})");

// ตัวอย่าง: Compare ข้อมูล 2 ชุด
var expected = new[] { 100, 200, 300, 400 };
var actual = new[] { 98, 205, 300, 395 };

var comparison = expected.Zip(actual, (e, a) => new
{
    Expected = e,
    Actual = a,
    Diff = a - e,
    WithinTolerance = Math.Abs(a - e) <= 5
});

foreach (var c in comparison)
    Console.WriteLine($"Expected: {c.Expected}, Actual: {c.Actual}, " +
        $"Diff: {c.Diff:+#;-#;0}, OK: {c.WithinTolerance}");

// Zip กับ index
var indexed = names.Zip(Enumerable.Range(1, names.Length),
    (name, idx) => $"{idx}. {name}");
foreach (string s in indexed)
    Console.WriteLine(s);
```

---

## 7. Let และ Into ใน Query Syntax

```csharp
var students = new[]
{
    new { Name = "Alice", Math = 85.0, Science = 92.0, English = 78.0 },
    new { Name = "Bob", Math = 70.0, Science = 68.0, English = 82.0 },
    new { Name = "Charlie", Math = 95.0, Science = 88.0, English = 91.0 },
};

// let: คำนวณค่าชั่วคราวใน query
var withAverage =
    from s in students
    let avg = (s.Math + s.Science + s.English) / 3
    let grade = avg >= 90 ? "A" : avg >= 80 ? "B" : avg >= 70 ? "C" : "D"
    where avg >= 75
    orderby avg descending
    select new { s.Name, Average = avg, Grade = grade };

foreach (var x in withAverage)
    Console.WriteLine($"{x.Name}: {x.Average:F2} ({x.Grade})");

// into: ใช้กับ group และ join
var products = new[]
{
    new { Category = "Electronics", Name = "Phone", Price = 15000m },
    new { Category = "Electronics", Name = "Laptop", Price = 35000m },
    new { Category = "Furniture", Name = "Chair", Price = 3500m },
    new { Category = "Furniture", Name = "Desk", Price = 8000m },
};

// into กับ group
var grouped =
    from p in products
    group p by p.Category into g
    where g.Count() > 1
    orderby g.Key
    select new
    {
        Category = g.Key,
        Count = g.Count(),
        TotalValue = g.Sum(p => p.Price),
        Items = g.Select(p => p.Name).ToList()
    };

foreach (var g in grouped)
{
    Console.WriteLine($"{g.Category}: {g.Count} items, {g.TotalValue:C}");
    Console.WriteLine($"  [{string.Join(", ", g.Items)}]");
}

// Continuation query (into กับ select)
var topExpensive =
    from p in products
    orderby p.Price descending
    select p
    into expensiveFirst
    take 3
    select new { expensiveFirst.Name, expensiveFirst.Price };

foreach (var p in topExpensive)
    Console.WriteLine($"{p.Name}: {p.Price:C}");
```

---

## 8. Conversion Methods

```csharp
var data = new[] { 1, 2, 3, 4, 5 };

// ToList: แปลงเป็น List<T> (Immediate execution)
List<int> list = data.Where(n => n > 2).ToList();

// ToArray: แปลงเป็น T[]
int[] arr = data.Where(n => n % 2 == 0).ToArray();

// ToHashSet: แปลงเป็น HashSet<T> (ลบ duplicate)
HashSet<int> set = new[] { 1, 2, 2, 3, 3, 3 }.ToHashSet();

// ToDictionary: แปลงเป็น Dictionary<K, V>
var products = new[]
{
    new { Id = 1, Name = "Apple", Price = 35m },
    new { Id = 2, Name = "Banana", Price = 15m },
    new { Id = 3, Name = "Cherry", Price = 120m }
};

// key selector
Dictionary<int, string> idToName = products.ToDictionary(p => p.Id, p => p.Name);

// key and value selector
Dictionary<string, decimal> nameToPrices = products.ToDictionary(p => p.Name, p => p.Price);

// ใช้ whole object
Dictionary<int, object> idToProduct = products.ToDictionary(p => p.Id, p => (object)p);

// ToLookup: คล้าย Dictionary แต่รองรับหลาย values ต่อ key
// (และ key ไม่ต้อง unique)
var orders = new[]
{
    new { CustomerId = 1, Amount = 100m },
    new { CustomerId = 2, Amount = 200m },
    new { CustomerId = 1, Amount = 300m },
    new { CustomerId = 3, Amount = 150m },
    new { CustomerId = 2, Amount = 250m },
};

// ToLookup: อ่านได้หลายครั้งโดยไม่ต้อง query ซ้ำ
ILookup<int, decimal> customerOrders = orders.ToLookup(o => o.CustomerId, o => o.Amount);

foreach (int customerId in customerOrders.Select(g => g.Key))
{
    var amounts = customerOrders[customerId];
    Console.WriteLine($"Customer {customerId}: {string.Join(", ", amounts)}");
}

// Key ที่ไม่มีจะคืน empty (ไม่ throw)
var unknown = customerOrders[999]; // empty, ไม่ throw!
Console.WriteLine($"Customer 999 orders: {unknown.Count()}"); // 0

// Cast and OfType
object[] mixed = { 1, "hello", 2, "world", 3, 4.5 };

// OfType: กรองตาม type (ปลอดภัย)
var ints = mixed.OfType<int>();         // 1, 2
var strings = mixed.OfType<string>();   // hello, world

// Cast: แปลง type (throw ถ้า cast ไม่ได้)
var onlyInts = mixed.Where(x => x is int).Cast<int>(); // 1, 2
```

---

## 9. Deferred vs Immediate Execution

```csharp
// ===== Deferred Execution =====
// LINQ query สร้างแค่ "blueprint" - ไม่ execute จนกว่าจะ iterate

var numbers = new List<int> { 1, 2, 3, 4, 5 };

// สร้าง query (ยังไม่ execute)
var query = numbers.Where(n => n > 2);
Console.WriteLine("Query created, not executed yet");

// Execute เมื่อ iterate
numbers.Add(6); // เพิ่มหลังสร้าง query
numbers.Add(7);

foreach (int n in query) // Execute ณ จุดนี้
    Console.Write($"{n} "); // 3, 4, 5, 6, 7 (รวมที่เพิ่มมาทีหลัง!)

// ===== Immediate Execution =====
// ToList, ToArray, ToDictionary, Count, Sum, First, etc.
// Execute ทันทีและ cache ผลลัพธ์

var snapshot = numbers.Where(n => n > 2).ToList(); // Execute ทันที
numbers.Add(8);
numbers.Add(9);

Console.WriteLine(string.Join(", ", snapshot)); // 3, 4, 5, 6, 7 (ไม่มี 8, 9)

// ตัวอย่างการ debug deferred execution
int callCount = 0;
var expensiveQuery = numbers.Where(n =>
{
    callCount++;
    return n > 3;
});

Console.WriteLine($"Before iterate: {callCount}"); // 0

// iterate ครั้งที่ 1
var count = expensiveQuery.Count();
Console.WriteLine($"After Count(): {callCount}"); // N (executed)

// iterate ครั้งที่ 2 (execute อีกครั้ง!)
var list = expensiveQuery.ToList();
Console.WriteLine($"After ToList(): {callCount}"); // 2N (executed again!)

// แก้ปัญหา: cache ด้วย ToList()
var cached = expensiveQuery.ToList(); // execute ครั้งเดียว
int count2 = cached.Count;
var list2 = cached; // ใช้ cached list
```

### 9.1 Performance Implications

```csharp
// ===== ปัญหาที่พบบ่อย: Multiple Enumeration =====
IEnumerable<int> GetNumbers()
{
    Console.WriteLine("Generating...");
    for (int i = 0; i < 5; i++)
    {
        Console.WriteLine($"  Yielding {i}");
        yield return i;
    }
}

var query = GetNumbers().Where(n => n > 2);

// Execute ครั้งที่ 1
Console.WriteLine("First Count:");
int c = query.Count(); // Generates + filters

// Execute ครั้งที่ 2 (Generates อีกครั้ง!)
Console.WriteLine("Second Iteration:");
foreach (int n in query) Console.WriteLine(n);

// แก้ปัญหา
var cached = GetNumbers().Where(n => n > 2).ToList(); // Generate ครั้งเดียว

// ===== Lazy Loading Pattern =====
// ดีสำหรับ: pipeline ที่ filter มากก่อน transform
// (ไม่ต้องโหลด/transform ข้อมูลที่ถูก filter ออก)

// ตัวอย่าง: อ่านไฟล์ขนาดใหญ่
IEnumerable<string> ReadLargeFile(string path)
{
    // ในจริงจะใช้ File.ReadLines() ซึ่ง lazy
    yield return "line1: data";
    yield return "line2: error";
    yield return "line3: data";
    yield return "line4: error";
}

// Process เฉพาะ error lines โดยไม่โหลดทั้งไฟล์
var errors = ReadLargeFile("dummy.txt")
    .Where(line => line.Contains("error"))
    .Take(10) // เอาแค่ 10 บรรทัดแรก - หยุดอ่านไฟล์เมื่อได้พอ
    .ToList();
```

---

## 10. โปรแกรมตัวอย่าง: Product Inventory LINQ Queries

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace ProductInventory
{
    public record Category(int Id, string Name, string Description);

    public record Product(
        int Id,
        string Name,
        int CategoryId,
        decimal Price,
        int Stock,
        string Supplier,
        DateTime LastRestocked,
        List<string> Tags
    );

    public record Sale(
        int ProductId,
        DateTime Date,
        int Quantity,
        decimal UnitPrice
    );

    public class InventorySystem
    {
        public List<Category> Categories { get; } = new();
        public List<Product> Products { get; } = new();
        public List<Sale> Sales { get; } = new();

        // === Basic Queries ===
        public IEnumerable<Product> GetLowStock(int threshold = 10) =>
            Products.Where(p => p.Stock <= threshold)
                    .OrderBy(p => p.Stock);

        public IEnumerable<Product> GetByCategory(string categoryName)
        {
            var category = Categories.FirstOrDefault(c =>
                c.Name.Equals(categoryName, StringComparison.OrdinalIgnoreCase));

            if (category == null) return Enumerable.Empty<Product>();

            return Products
                .Where(p => p.CategoryId == category.Id)
                .OrderBy(p => p.Name);
        }

        // === Join Queries ===
        public IEnumerable<object> GetProductsWithCategory() =>
            Products.Join(
                Categories,
                p => p.CategoryId,
                c => c.Id,
                (p, c) => (object)new
                {
                    p.Id, p.Name, Category = c.Name,
                    p.Price, p.Stock, p.Supplier
                }
            ).OrderBy(x => ((dynamic)x).Category)
             .ThenBy(x => ((dynamic)x).Name);

        // === Sales Analysis ===
        public IEnumerable<object> GetSalesByProduct()
        {
            return Sales
                .Join(Products, s => s.ProductId, p => p.Id,
                    (s, p) => new { s, p })
                .GroupBy(x => new { x.p.Id, x.p.Name })
                .Select(g => (object)new
                {
                    ProductId = g.Key.Id,
                    ProductName = g.Key.Name,
                    TotalQuantity = g.Sum(x => x.s.Quantity),
                    TotalRevenue = g.Sum(x => x.s.Quantity * x.s.UnitPrice),
                    TransactionCount = g.Count(),
                    AverageOrderSize = g.Average(x => x.s.Quantity)
                })
                .OrderByDescending(x => ((dynamic)x).TotalRevenue);
        }

        public IEnumerable<object> GetMonthlySalesReport()
        {
            return Sales
                .GroupBy(s => new { s.Date.Year, s.Date.Month })
                .Select(g => (object)new
                {
                    Year = g.Key.Year,
                    Month = g.Key.Month,
                    TotalRevenue = g.Sum(s => s.Quantity * s.UnitPrice),
                    TotalQuantity = g.Sum(s => s.Quantity),
                    TransactionCount = g.Count(),
                    UniqueProducts = g.Select(s => s.ProductId).Distinct().Count()
                })
                .OrderBy(x => ((dynamic)x).Year)
                .ThenBy(x => ((dynamic)x).Month);
        }

        // === Advanced Queries ===
        public IEnumerable<object> GetCategoryPerformance()
        {
            var salesByProduct = Sales.ToLookup(s => s.ProductId);

            return Products
                .Join(Categories, p => p.CategoryId, c => c.Id,
                    (p, c) => new { p, c })
                .GroupBy(x => x.c)
                .Select(g =>
                {
                    var products = g.Select(x => x.p).ToList();
                    var categorySales = products
                        .SelectMany(p => salesByProduct[p.Id])
                        .ToList();

                    return (object)new
                    {
                        Category = g.Key.Name,
                        ProductCount = products.Count,
                        TotalStock = products.Sum(p => p.Stock),
                        StockValue = products.Sum(p => p.Price * p.Stock),
                        TotalRevenue = categorySales.Sum(s => s.Quantity * s.UnitPrice),
                        TopProduct = products
                            .OrderByDescending(p => salesByProduct[p.Id].Sum(s => s.Quantity))
                            .FirstOrDefault()?.Name ?? "N/A"
                    };
                })
                .OrderByDescending(x => ((dynamic)x).TotalRevenue);
        }

        public IEnumerable<object> GetNeverSoldProducts()
        {
            var soldProductIds = Sales.Select(s => s.ProductId).ToHashSet();
            return Products
                .Where(p => !soldProductIds.Contains(p.Id))
                .Select(p => (object)new { p.Id, p.Name, p.Stock, p.Price });
        }

        public IEnumerable<object> GetTopSupplierProducts(int topN = 3)
        {
            return Products
                .GroupBy(p => p.Supplier)
                .Select(g => (object)new
                {
                    Supplier = g.Key,
                    Products = g.OrderByDescending(p => p.Price).Take(topN)
                                .Select(p => new { p.Name, p.Price, p.Stock })
                                .ToList(),
                    TotalStockValue = g.Sum(p => p.Price * p.Stock)
                })
                .OrderByDescending(x => ((dynamic)x).TotalStockValue);
        }

        public Dictionary<string, List<string>> GetTagCloud()
        {
            return Products
                .SelectMany(p => p.Tags.Select(tag => new { p.Name, Tag = tag }))
                .GroupBy(x => x.Tag)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(x => x.Name).OrderBy(n => n).ToList()
                );
        }

        public void PrintFullReport()
        {
            Console.WriteLine("════════════════════════════════════");
            Console.WriteLine("       INVENTORY REPORT             ");
            Console.WriteLine("════════════════════════════════════");

            Console.WriteLine($"\nTotal Products: {Products.Count}");
            Console.WriteLine($"Total Categories: {Categories.Count}");
            Console.WriteLine($"Total Sales Records: {Sales.Count}");

            decimal totalStockValue = Products.Sum(p => p.Price * p.Stock);
            Console.WriteLine($"Total Stock Value: {totalStockValue:C}");

            Console.WriteLine("\n── Low Stock Alert (≤10) ──");
            foreach (Product p in GetLowStock(10))
                Console.WriteLine($"  {p.Name}: {p.Stock} units left");

            Console.WriteLine("\n── Sales by Product ──");
            foreach (dynamic s in GetSalesByProduct())
                Console.WriteLine($"  {s.ProductName}: {s.TotalQuantity} units, {s.TotalRevenue:C}");

            Console.WriteLine("\n── Category Performance ──");
            foreach (dynamic c in GetCategoryPerformance())
                Console.WriteLine($"  {c.Category}: {c.ProductCount} products, Revenue: {c.TotalRevenue:C}");

            Console.WriteLine("\n── Never Sold Products ──");
            var neverSold = GetNeverSoldProducts().ToList();
            if (neverSold.Any())
                foreach (dynamic p in neverSold)
                    Console.WriteLine($"  {p.Name} (Stock: {p.Stock})");
            else
                Console.WriteLine("  All products have been sold!");
        }
    }

    class Program
    {
        static void Main()
        {
            var inventory = new InventorySystem();

            // เพิ่ม categories
            inventory.Categories.AddRange(new[]
            {
                new Category(1, "Electronics", "Electronic devices"),
                new Category(2, "Furniture", "Home and office furniture"),
                new Category(3, "Clothing", "Apparel and accessories"),
            });

            // เพิ่ม products
            inventory.Products.AddRange(new[]
            {
                new Product(1, "Laptop Pro", 1, 35000m, 15,
                    "TechCo", DateTime.Now.AddDays(-30), new() { "tech", "portable", "work" }),
                new Product(2, "Wireless Mouse", 1, 890m, 5,
                    "TechCo", DateTime.Now.AddDays(-10), new() { "tech", "peripheral" }),
                new Product(3, "Office Chair", 2, 5500m, 20,
                    "FurniCo", DateTime.Now.AddDays(-60), new() { "ergonomic", "work" }),
                new Product(4, "Standing Desk", 2, 12000m, 8,
                    "FurniCo", DateTime.Now.AddDays(-45), new() { "ergonomic", "work", "health" }),
                new Product(5, "T-Shirt", 3, 350m, 100,
                    "FashionCo", DateTime.Now.AddDays(-5), new() { "casual", "cotton" }),
                new Product(6, "Jeans", 3, 1200m, 50,
                    "FashionCo", DateTime.Now.AddDays(-15), new() { "casual", "denim" }),
                new Product(7, "Keyboard", 1, 1500m, 3,
                    "TechCo", DateTime.Now.AddDays(-20), new() { "tech", "peripheral", "mechanical" }),
                new Product(8, "Bookshelf", 2, 3200m, 0,
                    "FurniCo", DateTime.Now.AddDays(-90), new() { "storage" }),
            });

            // เพิ่ม sales
            var random = new Random(42);
            var dates = Enumerable.Range(0, 30)
                .Select(i => DateTime.Now.AddDays(-i))
                .ToArray();

            for (int i = 0; i < 50; i++)
            {
                int productId = random.Next(1, 8); // ไม่รวม product 8
                var product = inventory.Products.First(p => p.Id == productId);
                inventory.Sales.Add(new Sale(
                    productId,
                    dates[random.Next(dates.Length)],
                    random.Next(1, 10),
                    product.Price
                ));
            }

            inventory.PrintFullReport();

            // Extra: Tag cloud
            Console.WriteLine("\n── Tag Cloud ──");
            var tagCloud = inventory.GetTagCloud();
            foreach (var (tag, products) in tagCloud.OrderBy(x => x.Key))
                Console.WriteLine($"  #{tag}: {string.Join(", ", products)}");
        }
    }
}
```

---

## Exercises

### Exercise 1: Data Pipeline
สร้าง LINQ pipeline ที่:
- อ่านข้อมูล CSV (string array)
- Parse เป็น objects
- Filter, Transform, Aggregate
- Export กลับเป็น string

### Exercise 2: Graph Analysis
ใช้ LINQ วิเคราะห์ Social Network:
- หา mutual friends (Intersect)
- หา friends of friends (SelectMany)
- Suggest new friends (Except + Intersect)

### Exercise 3: Lazy Pipeline
สร้าง processing pipeline ที่:
- ใช้ Deferred Execution
- ประมวลผลไฟล์ขนาดใหญ่โดยไม่โหลดทั้งหมด
- ใช้ yield return ร่วมกับ LINQ

---

## สรุป

- ✅ `Join` เชื่อม 2 sequences ด้วย key (เหมือน INNER JOIN)
- ✅ `GroupJoin` ทำ LEFT OUTER JOIN - ทุก element ฝั่งซ้ายปรากฏ
- ✅ `SelectMany` flatten nested collections หรือ Cartesian product
- ✅ `Aggregate` ทำ custom fold/reduce operations
- ✅ `Distinct/Union/Intersect/Except` สำหรับ Set operations
- ✅ `DistinctBy/UnionBy/IntersectBy/ExceptBy` (C# 10+) ใช้ key selector
- ✅ `Zip` รวม 2+ sequences element by element
- ✅ `let` คำนวณค่าชั่วคราวใน Query Syntax
- ✅ `ToList/ToArray/ToDictionary/ToLookup/ToHashSet` บังคับ Immediate Execution
- ✅ Deferred Execution: query execute เมื่อ iterate จริง - ระวัง multiple enumeration
- ✅ ใช้ `ToList()` cache ผลลัพธ์เมื่อต้อง iterate หลายครั้ง

## Part ถัดไป
**Part 026** จะพูดถึง Exception Handling: try-catch-finally, Exception hierarchy, Custom exceptions, Exception filters และ Global exception handling

---
*Part 025/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
