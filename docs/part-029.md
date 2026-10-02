# Part 029: Lambda Expressions

## เนื้อหาใน Part นี้
- Lambda Syntax: Expression vs Statement
- Closure และการ Capture Variables
- Lambda กับ LINQ
- Expression Trees เบื้องต้น
- Anonymous Methods
- Method Groups
- โปรแกรมตัวอย่าง: Sorting Strategies

---

## 1. Lambda Syntax

Lambda Expression คือ anonymous function ที่เขียนสั้นกว่า method ปกติ

```csharp
// ===== Expression Lambda (single expression) =====
// parameters => expression
Func<int, int> square = x => x * x;
Func<int, int, int> add = (x, y) => x + y;
Func<string, string> upper = s => s.ToUpper();
Func<double, double, double> hypotenuse = (a, b) => Math.Sqrt(a * a + b * b);

Console.WriteLine(square(5));           // 25
Console.WriteLine(add(3, 4));           // 7
Console.WriteLine(upper("hello"));      // HELLO
Console.WriteLine(hypotenuse(3, 4));    // 5

// ===== Statement Lambda (block body) =====
// parameters => { statements }
Func<int, string> classify = n =>
{
    if (n < 0) return "negative";
    if (n == 0) return "zero";
    if (n < 10) return "small";
    return "large";
};

Console.WriteLine(classify(-5));  // negative
Console.WriteLine(classify(0));   // zero
Console.WriteLine(classify(7));   // small
Console.WriteLine(classify(100)); // large

// ===== No Parameters =====
Func<DateTime> now = () => DateTime.Now;
Action greet = () => Console.WriteLine("Hello!");
Func<int> getAnswer = () => 42;

greet();
Console.WriteLine(now());

// ===== Multiple Statements in Lambda =====
Action<List<int>> printAndSort = list =>
{
    Console.WriteLine($"Before: {string.Join(", ", list)}");
    list.Sort();
    Console.WriteLine($"After:  {string.Join(", ", list)}");
};

var nums = new List<int> { 5, 2, 8, 1, 9 };
printAndSort(nums);

// ===== Lambda Type Inference =====
// ส่วนใหญ่ C# infer type ได้เอง
var multiply = (int a, int b) => a * b; // C# 10+ explicit type
var numbers = new[] { 1, 2, 3, 4, 5 };
var evens = numbers.Where(n => n % 2 == 0); // type inferred

// ===== Natural Lambda Type (C# 10+) =====
var subtract = (int a, int b) => a - b;  // Func<int, int, int>
var writeLog = (string msg) => Console.WriteLine($"[LOG] {msg}"); // Action<string>
```

---

## 2. Closure - การ Capture Variables

Closure คือ lambda ที่ "จำ" (capture) variables จาก scope ภายนอก

```csharp
// ===== ตัวอย่าง Closure พื้นฐาน =====
int multiplier = 3;
Func<int, int> triple = n => n * multiplier; // capture 'multiplier'

Console.WriteLine(triple(5)); // 15

multiplier = 10; // เปลี่ยน captured variable
Console.WriteLine(triple(5)); // 50! (ค่าเปลี่ยนตามด้วย)

// ===== Counter สร้างด้วย Closure =====
Func<int> CreateCounter(int start = 0)
{
    int count = start;
    return () => ++count; // capture 'count'
}

var counter1 = CreateCounter();
var counter2 = CreateCounter(10);

Console.WriteLine(counter1()); // 1
Console.WriteLine(counter1()); // 2
Console.WriteLine(counter1()); // 3
Console.WriteLine(counter2()); // 11 (แยกกัน)
Console.WriteLine(counter2()); // 12
Console.WriteLine(counter1()); // 4

// ===== Classic Closure Bug (loop capture) =====
// Bug: ทุก lambda capture ตัวแปรเดียวกัน
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.Write($"{i} ")); // capture variable i, not value!
}

foreach (var action in actions)
    action(); // 5 5 5 5 5 (ไม่ใช่ 0 1 2 3 4!)

Console.WriteLine();

// Fix: สร้าง local copy
var fixedActions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    int copy = i; // สร้าง copy ใหม่ทุก iteration
    fixedActions.Add(() => Console.Write($"{copy} "));
}
foreach (var action in fixedActions)
    action(); // 0 1 2 3 4

// ===== Closure เก็บ State =====
// Bank account สร้างด้วย Closure (Functional approach)
(Func<decimal> GetBalance, Action<decimal> Deposit, Action<decimal> Withdraw)
CreateBankAccount(decimal initialBalance)
{
    decimal balance = initialBalance;
    var transactionLog = new List<(DateTime, string, decimal)>();

    return (
        GetBalance: () => balance,
        Deposit: amount =>
        {
            balance += amount;
            transactionLog.Add((DateTime.Now, "Deposit", amount));
            Console.WriteLine($"Deposited {amount:C}, Balance: {balance:C}");
        },
        Withdraw: amount =>
        {
            if (amount > balance)
                throw new InvalidOperationException("Insufficient funds");
            balance -= amount;
            transactionLog.Add((DateTime.Now, "Withdraw", amount));
            Console.WriteLine($"Withdrew {amount:C}, Balance: {balance:C}");
        }
    );
}

var (getBalance, deposit, withdraw) = CreateBankAccount(1000m);
Console.WriteLine($"Balance: {getBalance():C}"); // ฿1,000.00
deposit(500m);    // Deposited ฿500.00, Balance: ฿1,500.00
withdraw(200m);   // Withdrew ฿200.00, Balance: ฿1,300.00
Console.WriteLine($"Balance: {getBalance():C}"); // ฿1,300.00

// ===== Memoization ด้วย Closure =====
Func<TInput, TOutput> Memoize<TInput, TOutput>(Func<TInput, TOutput> func)
    where TInput : notnull
{
    var cache = new Dictionary<TInput, TOutput>(); // captured by closure
    return input =>
    {
        if (!cache.TryGetValue(input, out TOutput? result))
        {
            result = func(input);
            cache[input] = result;
        }
        return result;
    };
}

// Fibonacci with memoization
Func<int, long> fib = null!;
fib = Memoize<int, long>(n =>
    n <= 1 ? n : fib(n - 1) + fib(n - 2));

Console.WriteLine(fib(40)); // เร็วมากเพราะ cache

// ===== Partial Application ด้วย Closure =====
Func<int, int, int> power = (base_, exp) =>
    (int)Math.Pow(base_, exp);

Func<int, int> square2 = n => power(n, 2); // partial application
Func<int, int> cube = n => power(n, 3);
Func<int, int> power10 = n => power(10, n);

Console.WriteLine(square2(5));   // 25
Console.WriteLine(cube(3));      // 27
Console.WriteLine(power10(4));   // 10000
```

---

## 3. Lambda กับ LINQ

```csharp
// Lambda เป็นหัวใจของ LINQ Method Syntax
var products = new[]
{
    new { Name = "Laptop", Price = 35000m, Category = "Tech", Rating = 4.5 },
    new { Name = "Phone", Price = 15000m, Category = "Tech", Rating = 4.2 },
    new { Name = "Chair", Price = 4500m, Category = "Home", Rating = 4.0 },
    new { Name = "Desk", Price = 8000m, Category = "Home", Rating = 4.3 },
    new { Name = "Headphone", Price = 2500m, Category = "Tech", Rating = 4.7 },
};

// Where: กรอง
var techProducts = products.Where(p => p.Category == "Tech");

// Select: แปลง
var names = products.Select(p => p.Name);
var priceTag = products.Select(p => $"{p.Name}: {p.Price:C}");

// OrderBy: เรียง
var byPrice = products.OrderBy(p => p.Price);
var byRatingDesc = products.OrderByDescending(p => p.Rating);

// GroupBy: จัดกลุ่ม
var byCategory = products.GroupBy(p => p.Category);

// Aggregate: สรุป
decimal totalValue = products.Sum(p => p.Price);
double avgRating = products.Average(p => p.Rating);
var mostExpensive = products.MaxBy(p => p.Price);
var cheapest = products.MinBy(p => p.Price);

// Complex chaining
var report = products
    .Where(p => p.Price > 3000)
    .GroupBy(p => p.Category)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count(),
        AvgPrice = g.Average(p => p.Price),
        AvgRating = g.Average(p => p.Rating),
        TopProduct = g.MaxBy(p => p.Rating)?.Name
    })
    .OrderByDescending(x => x.AvgRating);

foreach (var r in report)
    Console.WriteLine($"{r.Category}: {r.Count} products, " +
        $"Avg: {r.AvgPrice:C}, Rating: {r.AvgRating:F1}, Top: {r.TopProduct}");

// ===== Custom Comparers ด้วย Lambda =====
var people = new[]
{
    new { Name = "Charlie", Age = 30 },
    new { Name = "Alice", Age = 25 },
    new { Name = "Bob", Age = 30 },
};

// Multi-key sort
var sorted = people
    .OrderBy(p => p.Age)
    .ThenBy(p => p.Name)
    .ToList();

// Custom sort ด้วย Comparer.Create
var customSorted = people.OrderBy(p => p, Comparer<object>.Create((a, b) =>
{
    dynamic da = a, db = b;
    int ageCompare = da.Age.CompareTo(db.Age);
    return ageCompare != 0 ? ageCompare : da.Name.CompareTo(db.Name);
}));
```

---

## 4. Expression Trees เบื้องต้น

Expression Trees แทน code เป็น data structure ที่สามารถวิเคราะห์ได้

```csharp
using System.Linq.Expressions;

// ===== Lambda as Expression =====
// Expression<Func<T>> แทน Func<T> โดยเก็บ tree structure
Expression<Func<int, int>> squareExpr = x => x * x;
Func<int, int> squareFunc = x => x * x;

// squareFunc: เรียกใช้ได้ทันที
Console.WriteLine(squareFunc(5)); // 25

// squareExpr: ดู structure ได้
Console.WriteLine(squareExpr); // x => (x * x)
Console.WriteLine(squareExpr.Body.GetType()); // BinaryExpression
Console.WriteLine(squareExpr.NodeType); // Lambda

// Compile expression เป็น function
Func<int, int> compiled = squareExpr.Compile();
Console.WriteLine(compiled(5)); // 25

// ===== ใช้ Expression ใน Entity Framework =====
// EF Core รับ Expression<Func<T, bool>> ไม่ใช่ Func<T, bool>
// เพราะ EF แปลง Expression เป็น SQL ได้

// ตัวอย่าง (สมมติ):
// Expression<Func<Product, bool>> filter = p => p.Price > 1000;
// dbContext.Products.Where(filter).ToList(); // แปลงเป็น SQL: WHERE Price > 1000

// ===== สร้าง Expression Tree ด้วย API =====
// สร้าง: x => x * x  โดยไม่ใช้ lambda syntax
ParameterExpression param = Expression.Parameter(typeof(int), "x");
BinaryExpression body = Expression.Multiply(param, param);
var lambda = Expression.Lambda<Func<int, int>>(body, param);

Func<int, int> dynamicSquare = lambda.Compile();
Console.WriteLine(dynamicSquare(7)); // 49

// ===== Expression Tree ใช้ทำ Type-safe Property =====
// แทนที่ magic string "PropertyName"
public static string GetPropertyName<T>(Expression<Func<T, object>> expr)
{
    if (expr.Body is MemberExpression member)
        return member.Member.Name;
    if (expr.Body is UnaryExpression unary && unary.Operand is MemberExpression m)
        return m.Member.Name;
    throw new ArgumentException("Invalid expression");
}

class Product { public string Name { get; set; } = ""; public decimal Price { get; set; } }

// Type-safe property name
string propName = GetPropertyName<Product>(p => p.Name);   // "Name"
string priceProp = GetPropertyName<Product>(p => p.Price); // "Price"

// ===== Dynamic Filter Builder =====
public static Expression<Func<T, bool>> BuildFilter<T>(
    string propertyName, object value)
{
    var param = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(param, propertyName);
    var constant = Expression.Constant(value, property.Type);
    var equal = Expression.Equal(property, constant);
    return Expression.Lambda<Func<T, bool>>(equal, param);
}

// สร้าง filter แบบ dynamic
var filter = BuildFilter<Product>("Name", "Laptop");
Func<Product, bool> compiledFilter = filter.Compile();

var products2 = new List<Product>
{
    new() { Name = "Laptop", Price = 35000 },
    new() { Name = "Phone", Price = 15000 }
};

var laptops = products2.Where(compiledFilter).ToList();
Console.WriteLine(laptops.Count); // 1
```

---

## 5. Anonymous Methods (เก่า แต่ควรรู้)

```csharp
// Anonymous Methods (C# 2.0) - เก่ากว่า lambda แต่ยังใช้อยู่บ้าง
Func<int, int> doubler = delegate(int n) { return n * 2; };
Action<string> printer = delegate(string s) { Console.WriteLine(s); };
Func<int, bool> isEven = delegate(int n) { return n % 2 == 0; };

Console.WriteLine(doubler(5));  // 10
printer("Hello");               // Hello
Console.WriteLine(isEven(4));   // true

// Anonymous Method ใช้ใน Event Handler (เก่า)
var button = new System.Windows.Forms.Button();
// button.Click += delegate(object sender, EventArgs e) { ... };

// ===== Lambda vs Anonymous Method =====
// Lambda:
Func<int, int> lambdaDouble = n => n * 2;
// Anonymous Method:
Func<int, int> anonDouble = delegate(int n) { return n * 2; };

// ทั้งสองเหมือนกัน แต่ Lambda:
// 1. สั้นกว่า
// 2. ใช้เป็น Expression Tree ได้
// 3. ไม่ต้องระบุ type (infer ได้)

// Anonymous method ที่ไม่ใช้ parameter (ต่างจาก lambda)
// สัญลักษณ์พิเศษ: ไม่ต้องใส่ parameter list ถ้าไม่ใช้
Action noParamAnon = delegate { Console.WriteLine("No params!"); };
// Lambda ต้องมี ():
Action noParamLambda = () => Console.WriteLine("No params!");
```

---

## 6. Method Groups

Method Group คือการอ้างถึง method โดยตรงโดยไม่ต้องใช้ lambda wrapper

```csharp
// ===== Method Groups =====
// แทนที่ n => n.ToString() ด้วย ToString.Method Group
var numbers = new[] { 1, 2, 3, 4, 5 };

// Lambda:
var strings1 = numbers.Select(n => n.ToString());
// Method Group (สั้นกว่า):
// var strings2 = numbers.Select(n => n.ToString()); // ทำได้ถ้าใช้ instance method

// Static method reference
var doubled = numbers.Select(n => n * 2).Select(Console.WriteLine);
// เหมือน: .Select(n => Console.WriteLine(n))

// ===== Method Group ใน Practice =====
List<string> names = new() { "Alice", "Bob", "Charlie" };

// Lambda
names.ForEach(n => Console.WriteLine(n));

// Method Group (สั้นกว่า!)
names.ForEach(Console.WriteLine);

// Sort ด้วย Method Group
names.Sort(string.Compare);    // static method
// หรือ
names.Sort(StringComparer.Ordinal.Compare);

// Filter ด้วย instance method
string[] words = { "hello", "world", "linq", "is", "great" };

// Lambda
var longWords1 = words.Where(w => w.Length > 4);

// Method Group ไม่ได้เสมอ (ต้องการ instance)
// ถ้า predicate เป็น instance method ใช้ได้
bool IsLongWord(string w) => w.Length > 4;
var longWords2 = words.Where(IsLongWord); // Local function as method group

// ===== delegate กับ Method Group =====
Func<double, double> sqrt = Math.Sqrt;      // static
Func<string, string> upper = string.Intern; // static
Action<object?> print = Console.WriteLine;  // static

Console.WriteLine(sqrt(16));   // 4
print("Hello via method group");

// Instance method group
var sb = new System.Text.StringBuilder();
Action<string> appendLine = sb.AppendLine;
appendLine("Line 1");
appendLine("Line 2");
Console.WriteLine(sb.ToString());
```

---

## 7. โปรแกรมตัวอย่าง: Sorting Strategies

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace SortingStrategies
{
    public record Product(string Name, decimal Price, int Stock, double Rating, string Category);

    // ===== Strategy Pattern ด้วย Lambda =====
    public class SortingStrategy<T>
    {
        public string Name { get; }
        public Func<IEnumerable<T>, IEnumerable<T>> Sort { get; }
        public string Description { get; }

        public SortingStrategy(string name, Func<IEnumerable<T>, IEnumerable<T>> sort,
            string description = "")
        {
            Name = name;
            Sort = sort;
            Description = description;
        }
    }

    public class ProductSorter
    {
        private readonly List<SortingStrategy<Product>> _strategies;

        public ProductSorter()
        {
            _strategies = new List<SortingStrategy<Product>>
            {
                // Price strategies
                new("price_asc",
                    products => products.OrderBy(p => p.Price),
                    "ราคาต่ำสุดก่อน"),

                new("price_desc",
                    products => products.OrderByDescending(p => p.Price),
                    "ราคาสูงสุดก่อน"),

                // Name
                new("name_asc",
                    products => products.OrderBy(p => p.Name),
                    "ชื่อ A-Z"),

                new("name_desc",
                    products => products.OrderByDescending(p => p.Name),
                    "ชื่อ Z-A"),

                // Rating
                new("rating_desc",
                    products => products.OrderByDescending(p => p.Rating),
                    "คะแนนสูงสุด"),

                // Composite: Best value (rating/price ratio)
                new("best_value",
                    products => products.OrderByDescending(p => p.Rating / (double)p.Price * 10000),
                    "คุ้มค่าที่สุด"),

                // Composite: Featured (rating > 4.5, in stock, by price)
                new("featured",
                    products => products
                        .OrderByDescending(p => p.Rating >= 4.5 ? 1 : 0)
                        .ThenByDescending(p => p.Stock > 0 ? 1 : 0)
                        .ThenBy(p => p.Price),
                    "แนะนำ (Rating > 4.5, มีสินค้า, ราคาน้อย)"),

                // Custom: Popular (high rating + enough stock)
                new("popular",
                    products => products
                        .Where(p => p.Stock > 10)
                        .OrderByDescending(p => p.Rating)
                        .Concat(products.Where(p => p.Stock <= 10).OrderBy(p => p.Stock)),
                    "ยอดนิยม (มีสต็อกเพียงพอ)"),
            };
        }

        public IEnumerable<SortingStrategy<Product>> GetStrategies() => _strategies;

        public IEnumerable<Product> Sort(IEnumerable<Product> products, string strategyName)
        {
            var strategy = _strategies.FirstOrDefault(s => s.Name == strategyName)
                ?? throw new ArgumentException($"Unknown strategy: {strategyName}");
            return strategy.Sort(products);
        }

        public void AddCustomStrategy(string name, Func<IEnumerable<Product>, IEnumerable<Product>> sort,
            string description = "")
        {
            _strategies.Add(new SortingStrategy<Product>(name, sort, description));
        }
    }

    // ===== Filter Builder ด้วย Lambda =====
    public class ProductFilter
    {
        private List<Func<Product, bool>> _filters = new();
        private string _description = "";

        public ProductFilter WithMinPrice(decimal minPrice)
        {
            _filters.Add(p => p.Price >= minPrice);
            _description += $" Price>={minPrice:C}";
            return this;
        }

        public ProductFilter WithMaxPrice(decimal maxPrice)
        {
            _filters.Add(p => p.Price <= maxPrice);
            _description += $" Price<={maxPrice:C}";
            return this;
        }

        public ProductFilter WithMinRating(double minRating)
        {
            _filters.Add(p => p.Rating >= minRating);
            _description += $" Rating>={minRating:F1}";
            return this;
        }

        public ProductFilter InStock()
        {
            _filters.Add(p => p.Stock > 0);
            _description += " InStock";
            return this;
        }

        public ProductFilter InCategory(params string[] categories)
        {
            var set = new HashSet<string>(categories, StringComparer.OrdinalIgnoreCase);
            _filters.Add(p => set.Contains(p.Category));
            _description += $" Category=[{string.Join(",", categories)}]";
            return this;
        }

        public ProductFilter WithCustomFilter(Func<Product, bool> filter, string desc = "custom")
        {
            _filters.Add(filter);
            _description += $" {desc}";
            return this;
        }

        public IEnumerable<Product> Apply(IEnumerable<Product> products)
        {
            // รวม filters ทั้งหมดด้วย AND
            Func<Product, bool> combined = p => _filters.All(f => f(p));
            return products.Where(combined);
        }

        public string GetDescription() => _description.Trim();

        // สร้าง Filter จาก Predicate
        public static ProductFilter Create(Func<Product, bool> predicate, string desc = "")
        {
            var filter = new ProductFilter();
            filter._filters.Add(predicate);
            filter._description = desc;
            return filter;
        }

        // Combine filters ด้วย OR
        public static ProductFilter Or(params ProductFilter[] filters)
        {
            var combined = new ProductFilter();
            combined._filters.Add(p => filters.Any(f => f.Apply(new[] { p }).Any()));
            combined._description = $"OR[{string.Join(", ", filters.Select(f => f.GetDescription()))}]";
            return combined;
        }
    }

    // ===== Analytics ด้วย Lambda Pipeline =====
    public class ProductAnalytics
    {
        private readonly List<Product> _products;

        public ProductAnalytics(List<Product> products) => _products = products;

        // Factory methods สร้าง computations ต่างๆ ด้วย lambda
        public Func<string, decimal> TotalValueByCategory =>
            category => _products
                .Where(p => p.Category.Equals(category, StringComparison.OrdinalIgnoreCase))
                .Sum(p => p.Price * p.Stock);

        public Func<double, List<Product>> GetProductsByMinRating =>
            minRating => _products
                .Where(p => p.Rating >= minRating)
                .OrderByDescending(p => p.Rating)
                .ToList();

        public Func<decimal, decimal, List<Product>> GetProductsInPriceRange =>
            (min, max) => _products
                .Where(p => p.Price >= min && p.Price <= max)
                .OrderBy(p => p.Price)
                .ToList();

        public Dictionary<string, Func<List<Product>>> CategoryReports =>
            _products.Select(p => p.Category)
                .Distinct()
                .ToDictionary(
                    category => category,
                    category => (Func<List<Product>>)(() =>
                        _products.Where(p => p.Category == category)
                                 .OrderByDescending(p => p.Rating)
                                 .ToList())
                );

        public void RunAllReports()
        {
            Console.WriteLine("=== Product Analytics ===\n");

            // Category totals
            var categories = _products.Select(p => p.Category).Distinct();
            Console.WriteLine("Inventory Value by Category:");
            foreach (var cat in categories)
                Console.WriteLine($"  {cat}: {TotalValueByCategory(cat):C}");

            // High rated
            Console.WriteLine("\nTop Rated Products (4.5+):");
            foreach (var p in GetProductsByMinRating(4.5))
                Console.WriteLine($"  {p.Name}: {p.Rating:F1} ⭐ - {p.Price:C}");

            // Price range
            Console.WriteLine("\nMid-range Products (1000-10000):");
            foreach (var p in GetProductsInPriceRange(1000m, 10000m))
                Console.WriteLine($"  {p.Name}: {p.Price:C}");
        }
    }

    class Program
    {
        static void Main()
        {
            var products = new List<Product>
            {
                new("Laptop Pro", 45000m, 5, 4.8, "Electronics"),
                new("Budget Phone", 5500m, 20, 4.0, "Electronics"),
                new("Wireless Mouse", 890m, 50, 4.3, "Electronics"),
                new("Mechanical Keyboard", 2800m, 15, 4.7, "Electronics"),
                new("Office Chair", 6500m, 8, 4.2, "Furniture"),
                new("Standing Desk", 18000m, 3, 4.6, "Furniture"),
                new("Bookshelf", 3200m, 12, 4.1, "Furniture"),
                new("T-Shirt", 350m, 100, 4.0, "Clothing"),
                new("Jeans", 1200m, 60, 4.3, "Clothing"),
                new("Sneakers", 3500m, 25, 4.5, "Clothing"),
                new("Headphones", 4500m, 30, 4.9, "Electronics"),
                new("Monitor 4K", 15000m, 7, 4.6, "Electronics"),
            };

            var sorter = new ProductSorter();
            var analytics = new ProductAnalytics(products);

            // Show all strategies
            Console.WriteLine("=== Available Sorting Strategies ===");
            foreach (var s in sorter.GetStrategies())
                Console.WriteLine($"  {s.Name,-15}: {s.Description}");

            // Demo: Sort by best value
            Console.WriteLine("\n=== Best Value Products ===");
            foreach (var p in sorter.Sort(products, "best_value").Take(5))
            {
                double valueScore = p.Rating / (double)p.Price * 10000;
                Console.WriteLine($"  {p.Name,-20} {p.Price,10:C} Rating: {p.Rating:F1} Score: {valueScore:F2}");
            }

            // Demo: Featured products
            Console.WriteLine("\n=== Featured Products ===");
            foreach (var p in sorter.Sort(products, "featured").Take(5))
                Console.WriteLine($"  {p.Name,-20} {p.Price,10:C} ⭐{p.Rating:F1} Stock: {p.Stock}");

            // Demo: Filter + Sort
            Console.WriteLine("\n=== Filtered & Sorted: Electronics, ≤10000, Rating≥4.5 ===");
            var filter = new ProductFilter()
                .InCategory("Electronics")
                .WithMaxPrice(10000m)
                .WithMinRating(4.5)
                .InStock();

            Console.WriteLine($"Filter: {filter.GetDescription()}");
            var filtered = filter.Apply(products).ToList();
            var sortedFiltered = sorter.Sort(filtered, "price_asc");
            foreach (var p in sortedFiltered)
                Console.WriteLine($"  {p.Name,-20} {p.Price,10:C} ⭐{p.Rating:F1}");

            // Custom strategy ด้วย lambda
            Console.WriteLine("\n=== Custom Sort: Best Electronics Under 5000 ===");
            sorter.AddCustomStrategy(
                "custom_electronics",
                prods => prods
                    .Where(p => p.Category == "Electronics" && p.Price < 5000)
                    .OrderByDescending(p => p.Rating * (1000 / (double)p.Price)),
                "Electronics ราคาดี คะแนนสูง"
            );

            foreach (var p in sorter.Sort(products, "custom_electronics"))
                Console.WriteLine($"  {p.Name,-20} {p.Price,10:C} ⭐{p.Rating:F1}");

            // Analytics
            Console.WriteLine();
            analytics.RunAllReports();

            // OR filter
            Console.WriteLine("\n=== OR Filter: Electronics OR Clothing ===");
            var electronicsFilter = new ProductFilter().InCategory("Electronics");
            var clothingFilter = new ProductFilter().InCategory("Clothing").WithMinRating(4.3);
            var combined = ProductFilter.Or(electronicsFilter, clothingFilter);
            foreach (var p in combined.Apply(products).OrderByDescending(p => p.Rating))
                Console.WriteLine($"  [{p.Category,-11}] {p.Name,-20} {p.Price,10:C} ⭐{p.Rating:F1}");

            // Closure example: สร้าง price tier lambda
            Console.WriteLine("\n=== Dynamic Price Tier Classifier ===");
            Func<decimal, string> CreatePriceClassifier(
                decimal budget, decimal midRange, decimal premium)
            {
                return price => price <= budget ? "Budget" :
                                price <= midRange ? "Mid-range" :
                                price <= premium ? "Premium" : "Luxury";
            }

            var classifier = CreatePriceClassifier(1000m, 5000m, 20000m);

            foreach (var p in products.OrderBy(p => p.Price))
                Console.WriteLine($"  {classifier(p.Price),-10} {p.Name,-20} {p.Price:C}");
        }
    }
}
```

---

## Exercises

### Exercise 1: Pipeline Builder
สร้าง text processing pipeline ด้วย lambda:
- `Transform(Func<string, string>)` เพิ่ม transformation
- `Filter(Func<string, bool>)` กรอง
- `Execute(string[])` ประมวลผล

```csharp
var pipeline = new TextPipeline()
    .Transform(s => s.Trim())
    .Transform(s => s.ToLower())
    .Filter(s => s.Length > 3)
    .Transform(s => s.Replace(" ", "_"));

string[] results = pipeline.Execute(inputs);
```

### Exercise 2: Event-driven Calculator
สร้าง calculator ที่:
- ใช้ `Dictionary<string, Func<double, double, double>>` สำหรับ operations
- เพิ่ม/ลบ operations ได้ dynamic
- Log ผลการคำนวณด้วย closures

### Exercise 3: Lazy Evaluation Chain
สร้าง lazy evaluation ด้วย lambda:
- แต่ละ step ทำงานเมื่อ request จริง
- Cache ผลลัพธ์ใน closure
- Support cancellation

---

## สรุป

- ✅ Lambda `x => expression` คือ Expression Lambda (return ทันที)
- ✅ Lambda `(x, y) => { statements }` คือ Statement Lambda
- ✅ Closure capture variables จาก outer scope - ระวัง loop capture bug!
- ✅ แก้ loop capture ด้วยการสร้าง `int copy = i` ทุก iteration
- ✅ `Func<T>` = function, `Action<T>` = procedure, `Predicate<T>` = bool function
- ✅ Lambda เป็นหัวใจของ LINQ method syntax
- ✅ `Expression<Func<T>>` เก็บ lambda เป็น data structure (ใช้ใน EF)
- ✅ Anonymous Methods (`delegate(x) { }`) เป็นรุ่นเก่าของ lambda
- ✅ Method Groups อ้างถึง method โดยตรงโดยไม่ต้องใช้ lambda
- ✅ Closure เหมาะสำหรับ Factory functions, Memoization, Partial Application

## Part ถัดไป
**Part 030** จะพูดถึง Nullable Types ขั้นสูง: Nullable\<T\>, null-conditional operators, null-coalescing, Nullable Reference Types (C# 8+) และ Pattern matching กับ null

---
*Part 029/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
