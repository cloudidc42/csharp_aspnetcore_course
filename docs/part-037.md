# Part 037: Pattern Matching ขั้นสูง (C# 9-13)

## เนื้อหาใน Part นี้
- is expression patterns
- switch expressions
- Positional patterns
- List patterns (C# 11)
- Extended property patterns
- Guard clauses กับ patterns
- โปรแกรมตัวอย่าง: Shape Calculator

---

## 1. is Expression Patterns

```csharp
using System;
using System.Collections.Generic;

class IsPatternExamples
{
    static void Main()
    {
        // Type pattern (C# 7+)
        object obj = "Hello";
        if (obj is string s)
        {
            Console.WriteLine($"String: {s.ToUpper()}");
        }
        
        // Null check
        string? name = null;
        if (name is null)
            Console.WriteLine("name is null");
        
        if (name is not null)
            Console.WriteLine("name has value");
        
        // Constant pattern
        int x = 42;
        if (x is 42)
            Console.WriteLine("x is 42");
        
        // Relational patterns (C# 9+)
        int age = 25;
        bool isAdult = age is >= 18;
        bool isChild = age is < 13;
        bool isTeenOrAdult = age is >= 13 and <= 18;
        
        Console.WriteLine($"Adult: {isAdult}, Child: {isChild}");
        
        // Logical patterns
        char c = 'A';
        bool isUpperLetter = c is >= 'A' and <= 'Z';
        bool isVowel = c is 'A' or 'E' or 'I' or 'O' or 'U';
        bool isNotDigit = c is not (>= '0' and <= '9');
        
        Console.WriteLine($"Upper: {isUpperLetter}, Vowel: {isVowel}");
        
        // Property pattern (C# 8+)
        var person = new Person("สมชาย", 25, "Bangkok");
        
        if (person is { Age: >= 18, City: "Bangkok" })
            Console.WriteLine($"{person.Name} เป็นผู้ใหญ่ในกรุงเทพ");
        
        // var pattern - จับค่า
        if (GetMaybeNull() is var result)
        {
            Console.WriteLine($"Result (may be null): {result ?? "null"}");
        }
        
        // Discard pattern
        if (GetObject() is { } notNull)
            Console.WriteLine($"Not null: {notNull}");
        
        // Combining patterns
        object value = 42;
        bool isSmallPositiveInt = value is int n and (> 0 and < 100);
        Console.WriteLine($"Small positive int: {isSmallPositiveInt}, n={( value is int nv ? nv : 0)}");
    }
    
    static string? GetMaybeNull() => DateTime.Now.Second % 2 == 0 ? "value" : null;
    static object? GetObject() => "not null";
}

public record Person(string Name, int Age, string City);
```

---

## 2. switch Expressions

### Basic switch Expression

```csharp
using System;

class SwitchExpressionBasics
{
    // switch expression แทน switch statement
    static string GetDayType(DayOfWeek day) =>
        day switch
        {
            DayOfWeek.Saturday or DayOfWeek.Sunday => "วันหยุด",
            DayOfWeek.Monday or DayOfWeek.Friday => "วันทำงาน (ต้น/ปลายสัปดาห์)",
            _ => "วันทำงานปกติ"
        };
    
    // กับ type patterns
    static double GetArea(object shape) =>
        shape switch
        {
            Circle c => Math.PI * c.Radius * c.Radius,
            Rectangle r => r.Width * r.Height,
            Triangle t => 0.5 * t.Base * t.Height,
            null => throw new ArgumentNullException(nameof(shape)),
            _ => throw new ArgumentException($"ไม่รู้จัก shape: {shape.GetType().Name}")
        };
    
    // กับ property patterns
    static decimal CalculateDiscount(Customer customer) =>
        customer switch
        {
            { Type: CustomerType.VIP, YearsOfMembership: >= 5 } => 0.30m,
            { Type: CustomerType.VIP } => 0.20m,
            { Type: CustomerType.Regular, YearsOfMembership: >= 2 } => 0.10m,
            { TotalPurchases: >= 50000 } => 0.05m,
            _ => 0m
        };
    
    static void Main()
    {
        foreach (DayOfWeek day in Enum.GetValues<DayOfWeek>())
        {
            Console.WriteLine($"{day}: {GetDayType(day)}");
        }
        
        Console.WriteLine($"\nCircle area: {GetArea(new Circle(5)):F2}");
        Console.WriteLine($"Rectangle area: {GetArea(new Rectangle(4, 6)):F2}");
        
        var customers = new[]
        {
            new Customer("VIP Member 5yr", CustomerType.VIP, 5, 100000),
            new Customer("VIP Member 1yr", CustomerType.VIP, 1, 10000),
            new Customer("Regular 3yr", CustomerType.Regular, 3, 5000),
            new Customer("Big Spender", CustomerType.Regular, 1, 60000),
            new Customer("New Customer", CustomerType.Regular, 0, 1000)
        };
        
        foreach (var c in customers)
        {
            Console.WriteLine($"{c.Name}: {CalculateDiscount(c) * 100}% discount");
        }
    }
}

record Circle(double Radius);
record Rectangle(double Width, double Height);
record Triangle(double Base, double Height);
enum CustomerType { Regular, VIP, Premium }
record Customer(string Name, CustomerType Type, int YearsOfMembership, decimal TotalPurchases);
```

### Advanced switch Expressions

```csharp
using System;
using System.Collections.Generic;

class AdvancedSwitchExpressions
{
    // Nested property patterns
    record Address(string Street, string City, string Country);
    record Order(int Id, decimal Amount, Address DeliveryAddress, bool IsPaid);
    
    static decimal CalculateShipping(Order order) =>
        order switch
        {
            { IsPaid: false } => throw new InvalidOperationException("ต้องชำระเงินก่อน"),
            { DeliveryAddress.Country: "TH", Amount: >= 1000 } => 0,    // ฟรีส่ง
            { DeliveryAddress.Country: "TH" } => 50,                      // ในประเทศ
            { DeliveryAddress.Country: "SG" or "MY" } => 150,            // ASEAN
            { DeliveryAddress.Country: var c } when IsEuCountry(c) => 500, // EU
            _ => 800                                                        // อื่นๆ
        };
    
    static bool IsEuCountry(string country) =>
        new[] { "DE", "FR", "IT", "ES", "NL" }.Contains(country);
    
    // Recursive patterns
    abstract record Expr;
    record Num(double Value) : Expr;
    record Add(Expr Left, Expr Right) : Expr;
    record Mul(Expr Left, Expr Right) : Expr;
    record Neg(Expr Operand) : Expr;
    
    static double Evaluate(Expr expr) => expr switch
    {
        Num(var v) => v,
        Add(var left, var right) => Evaluate(left) + Evaluate(right),
        Mul(var left, var right) => Evaluate(left) * Evaluate(right),
        Neg(var operand) => -Evaluate(operand),
        _ => throw new ArgumentException("Unknown expression")
    };
    
    static string PrintExpr(Expr expr) => expr switch
    {
        Num(var v) => v.ToString(),
        Add(var l, var r) => $"({PrintExpr(l)} + {PrintExpr(r)})",
        Mul(var l, var r) => $"({PrintExpr(l)} * {PrintExpr(r)})",
        Neg(var e) => $"(-{PrintExpr(e)})",
        _ => "?"
    };
    
    static void Main()
    {
        // Shipping
        var orders = new[]
        {
            new Order(1, 1500, new Address("Main St", "Bangkok", "TH"), true),
            new Order(2, 500, new Address("Street", "Bangkok", "TH"), true),
            new Order(3, 200, new Address("Ave", "Singapore", "SG"), true),
            new Order(4, 1000, new Address("Road", "Berlin", "DE"), true),
            new Order(5, 100, new Address("Blvd", "New York", "US"), true),
        };
        
        Console.WriteLine("Shipping costs:");
        foreach (var order in orders)
        {
            Console.WriteLine($"  Order {order.Id} to {order.DeliveryAddress.Country}: " +
                $"{CalculateShipping(order):N0} บาท");
        }
        
        // Expression evaluation: (2 + 3) * -(4 + 1)
        Expr e = new Mul(
            new Add(new Num(2), new Num(3)),
            new Neg(new Add(new Num(4), new Num(1)))
        );
        
        Console.WriteLine($"\nExpression: {PrintExpr(e)}");
        Console.WriteLine($"Result: {Evaluate(e)}");
    }
}
```

---

## 3. Positional Patterns

```csharp
using System;
using System.Collections.Generic;

class PositionalPatterns
{
    // Positional pattern กับ Deconstruct
    record Point(double X, double Y);
    record Rectangle(Point TopLeft, Point BottomRight);
    
    static string ClassifyPoint(Point p) =>
        p switch
        {
            (0, 0) => "Origin",
            (var x, 0) => $"On X-axis at {x}",
            (0, var y) => $"On Y-axis at {y}",
            ( > 0, > 0) => "Quadrant I",
            ( < 0, > 0) => "Quadrant II",
            ( < 0, < 0) => "Quadrant III",
            ( > 0, < 0) => "Quadrant IV",
            _ => "Unknown"
        };
    
    // กับ tuple
    static string GetBMICategory(double bmi) =>
        bmi switch
        {
            < 18.5 => "น้ำหนักน้อย",
            >= 18.5 and < 25 => "ปกติ",
            >= 25 and < 30 => "น้ำหนักเกิน",
            >= 30 => "โรคอ้วน",
            _ => "ไม่ทราบ"
        };
    
    // กับ class ที่มี Deconstruct
    record DateRange(DateTime Start, DateTime End)
    {
        public void Deconstruct(out int days, out bool isWeekend)
        {
            days = (int)(End - Start).TotalDays;
            isWeekend = Start.DayOfWeek == DayOfWeek.Saturday ||
                        Start.DayOfWeek == DayOfWeek.Sunday;
        }
    }
    
    static string ClassifyDateRange(DateRange range) =>
        range switch
        {
            (1, true) => "วันหยุดสุดสัปดาห์ 1 วัน",
            (2, true) => "สุดสัปดาห์เต็มวัน",
            ( > 7, _) => "มากกว่า 1 สัปดาห์",
            (var days, false) => $"วันทำงาน {days} วัน",
            _ => "ไม่ทราบ"
        };
    
    static void Main()
    {
        var points = new[]
        {
            new Point(0, 0), new Point(3, 0), new Point(0, -2),
            new Point(1, 2), new Point(-1, 3), new Point(-2, -1), new Point(3, -2)
        };
        
        Console.WriteLine("Point Classifications:");
        foreach (var p in points)
        {
            Console.WriteLine($"  ({p.X}, {p.Y}): {ClassifyPoint(p)}");
        }
        
        Console.WriteLine("\nBMI Categories:");
        foreach (var bmi in new[] { 16.5, 22.0, 27.5, 35.0 })
        {
            Console.WriteLine($"  BMI {bmi}: {GetBMICategory(bmi)}");
        }
    }
}
```

---

## 4. List Patterns (C# 11)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

class ListPatterns
{
    static void Main()
    {
        // Basic list patterns
        int[] empty = Array.Empty<int>();
        int[] one = { 1 };
        int[] two = { 1, 2 };
        int[] many = { 1, 2, 3, 4, 5 };
        
        Console.WriteLine(DescribeArray(empty));
        Console.WriteLine(DescribeArray(one));
        Console.WriteLine(DescribeArray(two));
        Console.WriteLine(DescribeArray(many));
        
        // Matching specific patterns
        var sequences = new[]
        {
            new[] { 1, 2, 3 },
            new[] { 0, 1, 2 },
            new[] { 1, 2, 3, 4 },
            new[] { 1, 1, 1 },
            new[] { 3, 2, 1 }
        };
        
        foreach (var seq in sequences)
        {
            Console.WriteLine($"[{string.Join(",", seq)}]: {AnalyzeSequence(seq)}");
        }
        
        // List pattern ใน real-world
        var lines = new[]
        {
            new[] { "GET", "/api/users", "200" },
            new[] { "POST", "/api/users", "201" },
            new[] { "DELETE", "/api/users/1", "204" },
            new[] { "GET", "/api/unknown", "404" },
            new[] { "internal error" }
        };
        
        Console.WriteLine("\nLog parsing:");
        foreach (var line in lines)
        {
            string description = line switch
            {
                ["GET", var path, "200"] => $"GET {path} OK",
                ["POST", var path, "201"] => $"POST {path} Created",
                ["DELETE", _, "204"] => "DELETE success",
                [_, _, var status] when int.Parse(status) >= 400 => $"Error: {status}",
                [var msg] => $"Single: {msg}",
                _ => "Unknown format"
            };
            Console.WriteLine($"  {description}");
        }
    }
    
    static string DescribeArray(int[] arr) => arr switch
    {
        [] => "Empty",
        [var single] => $"Single: {single}",
        [var first, var second] => $"Two: {first}, {second}",
        [var first, .., var last] => $"Many: starts={first}, ends={last}",
        _ => "Unknown"
    };
    
    static string AnalyzeSequence(int[] arr) => arr switch
    {
        [1, 2, 3] => "ลำดับ 1-2-3 พอดี",
        [0, ..] => "เริ่มด้วย 0",
        [.., 1] => "จบด้วย 1",
        [var f, var s, ..] when f == s => $"สองตัวแรกเท่ากัน ({f})",
        [var f, .., var l] when f > l => "ลดลง (first > last)",
        _ => "อื่นๆ"
    };
}
```

---

## 5. Extended Property Patterns (C# 10)

```csharp
using System;
using System.Collections.Generic;

class ExtendedPropertyPatterns
{
    record Address(string City, string Country, string PostalCode);
    record Person(string Name, int Age, Address HomeAddress, Person? Spouse = null);
    record Company(string Name, Address HeadOffice, List<Person> Employees);
    
    static void Main()
    {
        var people = new[]
        {
            new Person("สมชาย", 35, new Address("Bangkok", "TH", "10100")),
            new Person("John", 28, new Address("London", "UK", "SW1A1AA")),
            new Person("Marie", 42, new Address("Paris", "FR", "75001")),
            new Person("สมหญิง", 30, new Address("Chiang Mai", "TH", "50000"))
        };
        
        foreach (var person in people)
        {
            string category = person switch
            {
                // Extended property pattern: NestedProp.SubProp
                { HomeAddress.Country: "TH", HomeAddress.City: "Bangkok" } =>
                    $"คนกรุงเทพ",
                
                { HomeAddress.Country: "TH" } =>
                    $"คนไทย (ต่างจังหวัด)",
                
                { HomeAddress.Country: "UK" or "FR", Age: >= 30 } =>
                    $"ชาวยุโรปอายุ >= 30",
                
                { HomeAddress.Country: var country, Age: < 30 } =>
                    $"คนหนุ่มสาวจาก {country}",
                
                _ => "อื่นๆ"
            };
            
            Console.WriteLine($"{person.Name}: {category}");
        }
        
        // กับ nullable
        var withSpouse = new Person("วิชัย", 40,
            new Address("Bangkok", "TH", "10200"),
            Spouse: new Person("สุดา", 38, new Address("Bangkok", "TH", "10200"))
        );
        
        string maritalStatus = withSpouse switch
        {
            { Spouse: { Age: > 40 } } => "แต่งงาน คู่สมรสอายุ > 40",
            { Spouse: { Name: var spouseName } } => $"แต่งงานกับ {spouseName}",
            { Spouse: null } => "โสด",
            _ => "ไม่ทราบ"
        };
        
        Console.WriteLine($"\n{withSpouse.Name}: {maritalStatus}");
        
        // JSON-like object patterns
        var config = new Dictionary<string, object>
        {
            ["debug"] = true,
            ["maxRetries"] = 3,
            ["environment"] = "production"
        };
        
        bool isProductionDebug = config.TryGetValue("environment", out var env) &&
                                  config.TryGetValue("debug", out var debug) &&
                                  env is "production" && debug is true;
        
        Console.WriteLine($"\nProduction with debug: {isProductionDebug}");
    }
}
```

---

## 6. Guard Clauses กับ Patterns

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

class GuardClauses
{
    record Product(string Name, decimal Price, int Stock, string Category, DateTime ExpiryDate);
    record Order(Customer Customer, List<(Product Product, int Qty)> Items, string PromoCode);
    record Customer(string Name, bool IsPremium, int LoyaltyPoints, DateTime MemberSince);
    
    // guard clause ด้วย when
    static decimal CalculateOrderTotal(Order order)
    {
        decimal subtotal = order.Items.Sum(i => i.Product.Price * i.Qty);
        
        decimal discount = (order, subtotal) switch
        {
            // Premium + promo code + ยอดสูง
            ({ Customer.IsPremium: true }, _) when order.PromoCode == "VIP20" => 0.20m,
            
            // Premium + ยอดมาก
            ({ Customer.IsPremium: true }, >= 5000) => 0.15m,
            
            // Premium
            ({ Customer.IsPremium: true }, _) => 0.10m,
            
            // Regular + promo
            (_, _) when order.PromoCode == "SAVE10" => 0.10m,
            
            // High value
            (_, >= 10000) => 0.08m,
            (_, >= 5000) => 0.05m,
            
            _ => 0m
        };
        
        return subtotal * (1 - discount);
    }
    
    // Pattern matching สำหรับ validation
    static (bool Valid, string Message) ValidateProduct(Product product) =>
        product switch
        {
            { Price: <= 0 } => (false, "ราคาต้องมากกว่า 0"),
            { Stock: < 0 } => (false, "สต็อกต้องไม่ติดลบ"),
            { ExpiryDate: var exp } when exp < DateTime.Today => (false, "สินค้าหมดอายุแล้ว"),
            { Name: null or "" } => (false, "ชื่อสินค้าต้องไม่ว่าง"),
            { Category: "fresh" } when product.Stock == 0 => (false, "สินค้าสดต้องมีสต็อก"),
            _ => (true, "ถูกต้อง")
        };
    
    // Complex state machine
    enum OrderState { Pending, Confirmed, Processing, Shipped, Delivered, Cancelled, Refunded }
    
    static (OrderState Next, string Message) TransitionOrder(
        OrderState current,
        string action,
        bool hasInventory = true,
        bool isPaid = false) =>
        (current, action) switch
        {
            (OrderState.Pending, "confirm") when isPaid && hasInventory =>
                (OrderState.Confirmed, "Order confirmed"),
            
            (OrderState.Pending, "confirm") when !isPaid =>
                (OrderState.Pending, "กรุณาชำระเงินก่อน"),
            
            (OrderState.Pending, "confirm") when !hasInventory =>
                (OrderState.Pending, "สินค้าหมดสต็อก"),
            
            (OrderState.Confirmed, "process") =>
                (OrderState.Processing, "กำลังเตรียมสินค้า"),
            
            (OrderState.Processing, "ship") =>
                (OrderState.Shipped, "จัดส่งแล้ว"),
            
            (OrderState.Shipped, "deliver") =>
                (OrderState.Delivered, "ส่งถึงแล้ว"),
            
            (OrderState.Pending or OrderState.Confirmed, "cancel") =>
                (OrderState.Cancelled, "ยกเลิกแล้ว"),
            
            (OrderState.Delivered, "refund") =>
                (OrderState.Refunded, "คืนเงินแล้ว"),
            
            _ => (current, $"ไม่สามารถ {action} ที่ state {current}")
        };
    
    static void Main()
    {
        // Order calculation
        var customer1 = new Customer("VIP", true, 1000, new DateTime(2020, 1, 1));
        var product1 = new Product("iPhone", 49900, 5, "electronics", DateTime.Now.AddYears(1));
        
        var order = new Order(
            customer1,
            new List<(Product, int)> { (product1, 2) },
            "VIP20"
        );
        
        Console.WriteLine($"Order total: {CalculateOrderTotal(order):N0} บาท");
        
        // Validation
        Console.WriteLine("\nProduct Validation:");
        var products = new[]
        {
            new Product("Milk", 45, 10, "fresh", DateTime.Now.AddDays(7)),
            new Product("Expired", 30, 5, "fresh", DateTime.Now.AddDays(-1)),
            new Product("", 0, -1, "electronics", DateTime.Now.AddYears(1)),
        };
        
        foreach (var p in products)
        {
            var (valid, msg) = ValidateProduct(p);
            Console.WriteLine($"  {(string.IsNullOrEmpty(p.Name) ? "(empty)" : p.Name)}: " +
                $"{(valid ? "✅" : "❌")} {msg}");
        }
        
        // State machine
        Console.WriteLine("\nOrder State Machine:");
        var state = OrderState.Pending;
        var actions = new[]
        {
            ("confirm", false, false),  // ไม่ได้ชำระ
            ("confirm", true, true),    // ชำระแล้ว มีสต็อก
            ("process", true, true),
            ("ship", true, true),
            ("deliver", true, true)
        };
        
        foreach (var (action, paid, inventory) in actions)
        {
            var (next, msg) = TransitionOrder(state, action, inventory, paid);
            Console.WriteLine($"  [{state}] + '{action}' -> [{next}]: {msg}");
            state = next;
        }
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Shape Calculator

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// Shape hierarchy
public abstract record Shape
{
    public abstract double Area { get; }
    public abstract double Perimeter { get; }
    public string Color { get; init; } = "White";
}

public record Circle(double Radius) : Shape
{
    public override double Area => Math.PI * Radius * Radius;
    public override double Perimeter => 2 * Math.PI * Radius;
}

public record Rectangle(double Width, double Height) : Shape
{
    public override double Area => Width * Height;
    public override double Perimeter => 2 * (Width + Height);
    public bool IsSquare => Math.Abs(Width - Height) < 1e-10;
}

public record Triangle(double SideA, double SideB, double SideC) : Shape
{
    private double S => (SideA + SideB + SideC) / 2;
    public override double Area => Math.Sqrt(S * (S - SideA) * (S - SideB) * (S - SideC));
    public override double Perimeter => SideA + SideB + SideC;
    
    public bool IsEquilateral => Math.Abs(SideA - SideB) < 1e-10 && Math.Abs(SideB - SideC) < 1e-10;
    public bool IsIsosceles => Math.Abs(SideA - SideB) < 1e-10 || Math.Abs(SideB - SideC) < 1e-10
                              || Math.Abs(SideA - SideC) < 1e-10;
    public bool IsRight => Math.Abs(SideA * SideA + SideB * SideB - SideC * SideC) < 1e-10 ||
                           Math.Abs(SideA * SideA + SideC * SideC - SideB * SideB) < 1e-10 ||
                           Math.Abs(SideB * SideB + SideC * SideC - SideA * SideA) < 1e-10;
}

public record Ellipse(double SemiMajor, double SemiMinor) : Shape
{
    public override double Area => Math.PI * SemiMajor * SemiMinor;
    // Ramanujan approximation
    public override double Perimeter => Math.PI * (3 * (SemiMajor + SemiMinor) -
        Math.Sqrt((3 * SemiMajor + SemiMinor) * (SemiMajor + 3 * SemiMinor)));
}

public record RegularPolygon(int Sides, double SideLength) : Shape
{
    public override double Area =>
        (Sides * SideLength * SideLength) / (4 * Math.Tan(Math.PI / Sides));
    public override double Perimeter => Sides * SideLength;
}

// Shape Calculator with Pattern Matching
public class ShapeCalculator
{
    // Describe shape
    public static string Describe(Shape shape) => shape switch
    {
        Circle { Radius: 0 } => "จุด (ไม่ใช่ circle)",
        Circle c when c.Radius < 0 => "Invalid circle",
        Circle { Radius: var r } => $"วงกลม รัศมี {r:F2}",
        
        Rectangle { IsSquare: true, Width: var s } => $"สี่เหลี่ยมจัตุรัส ด้าน {s:F2}",
        Rectangle { Width: var w, Height: var h } => $"สี่เหลี่ยมผืนผ้า {w:F2}x{h:F2}",
        
        Triangle { IsEquilateral: true, SideA: var s } => $"สามเหลี่ยมด้านเท่า ด้าน {s:F2}",
        Triangle { IsRight: true } => "สามเหลี่ยมมุมฉาก",
        Triangle { IsIsosceles: true } => "สามเหลี่ยมหน้าจั่ว",
        Triangle => "สามเหลี่ยมด้านไม่เท่า",
        
        RegularPolygon { Sides: 3 } => "สามเหลี่ยมด้านเท่า",
        RegularPolygon { Sides: 4 } => "สี่เหลี่ยมจัตุรัส",
        RegularPolygon { Sides: 5 } => "ห้าเหลี่ยมด้านเท่า",
        RegularPolygon { Sides: 6 } => "หกเหลี่ยมด้านเท่า",
        RegularPolygon { Sides: var n } => $"{n}-เหลี่ยมด้านเท่า",
        
        _ => $"รูปทรง: {shape.GetType().Name}"
    };
    
    // Classify by size
    public static string ClassifyByArea(Shape shape) =>
        shape.Area switch
        {
            0 => "ไม่มีพื้นที่",
            > 0 and <= 10 => "เล็กมาก",
            > 10 and <= 100 => "เล็ก",
            > 100 and <= 1000 => "กลาง",
            > 1000 and <= 10000 => "ใหญ่",
            _ => "ใหญ่มาก"
        };
    
    // Calculate relationship between shapes
    public static string CompareShapes(Shape s1, Shape s2) =>
        (s1.Area - s2.Area) switch
        {
            0 => "พื้นที่เท่ากัน",
            > 0 and <= 1 => $"{s1.GetType().Name} ใหญ่กว่าเล็กน้อย",
            > 1 => $"{s1.GetType().Name} ใหญ่กว่า {s1.Area / s2.Area:F1}x",
            < 0 and >= -1 => $"{s2.GetType().Name} ใหญ่กว่าเล็กน้อย",
            _ => $"{s2.GetType().Name} ใหญ่กว่า {s2.Area / s1.Area:F1}x"
        };
    
    // Check containment
    public static bool Contains(Shape outer, Shape inner) =>
        (outer, inner) switch
        {
            (Circle c1, Circle c2) => c1.Radius >= c2.Radius,
            (Rectangle r1, Rectangle r2) => r1.Width >= r2.Width && r1.Height >= r2.Height,
            (Circle c, Rectangle r) =>
                c.Radius >= Math.Sqrt(r.Width * r.Width + r.Height * r.Height) / 2,
            (Rectangle r, Circle c) =>
                r.Width >= 2 * c.Radius && r.Height >= 2 * c.Radius,
            _ => outer.Area >= inner.Area // approximation
        };
    
    // List pattern: จัดกลุ่ม shapes
    public static Dictionary<string, List<Shape>> GroupShapes(IEnumerable<Shape> shapes)
    {
        return shapes.GroupBy(s => s switch
        {
            Circle => "วงกลม",
            Rectangle { IsSquare: true } => "สี่เหลี่ยมจัตุรัส",
            Rectangle => "สี่เหลี่ยมผืนผ้า",
            Triangle => "สามเหลี่ยม",
            RegularPolygon => "รูปหลายเหลี่ยมด้านเท่า",
            _ => "อื่นๆ"
        })
        .ToDictionary(g => g.Key, g => g.ToList());
    }
}

// Main Program
class Program
{
    static void Main()
    {
        Console.WriteLine("===== Shape Calculator with Pattern Matching =====\n");
        
        var shapes = new List<Shape>
        {
            new Circle(5) { Color = "Red" },
            new Circle(0),
            new Rectangle(4, 4) { Color = "Blue" },
            new Rectangle(6, 3) { Color = "Green" },
            new Triangle(3, 4, 5) { Color = "Yellow" },      // Right triangle
            new Triangle(5, 5, 5) { Color = "Purple" },      // Equilateral
            new Triangle(5, 5, 3) { Color = "Orange" },      // Isosceles
            new RegularPolygon(6, 4) { Color = "Pink" },
            new Ellipse(5, 3)
        };
        
        Console.WriteLine("--- Shape Descriptions ---");
        foreach (var shape in shapes)
        {
            Console.WriteLine($"  {ShapeCalculator.Describe(shape)}");
            Console.WriteLine($"    Area={shape.Area:F2}, Perimeter={shape.Perimeter:F2}");
            Console.WriteLine($"    Size: {ShapeCalculator.ClassifyByArea(shape)}");
        }
        
        Console.WriteLine("\n--- Shape Comparison ---");
        Console.WriteLine(ShapeCalculator.CompareShapes(shapes[0], shapes[3]));
        Console.WriteLine(ShapeCalculator.CompareShapes(shapes[3], shapes[0]));
        
        Console.WriteLine("\n--- Containment Check ---");
        var bigCircle = new Circle(10);
        var smallCircle = new Circle(5);
        var bigRect = new Rectangle(20, 15);
        var smallRect = new Rectangle(8, 6);
        
        Console.WriteLine($"Circle(10) contains Circle(5): {ShapeCalculator.Contains(bigCircle, smallCircle)}");
        Console.WriteLine($"Rect(20,15) contains Rect(8,6): {ShapeCalculator.Contains(bigRect, smallRect)}");
        Console.WriteLine($"Circle(10) contains Rect(8,6): {ShapeCalculator.Contains(bigCircle, smallRect)}");
        
        Console.WriteLine("\n--- Grouped by Type ---");
        var groups = ShapeCalculator.GroupShapes(shapes);
        foreach (var (type, list) in groups)
        {
            Console.WriteLine($"  {type}: {list.Count} shape(s)");
            foreach (var s in list)
            {
                Console.WriteLine($"    - {ShapeCalculator.Describe(s)} (Area={s.Area:F2})");
            }
        }
        
        // Statistics
        Console.WriteLine("\n--- Statistics ---");
        var totalArea = shapes.Where(s => s.Area > 0).Sum(s => s.Area);
        var avgArea = shapes.Where(s => s.Area > 0).Average(s => s.Area);
        var largest = shapes.MaxBy(s => s.Area);
        
        Console.WriteLine($"Total area: {totalArea:F2}");
        Console.WriteLine($"Average area: {avgArea:F2}");
        Console.WriteLine($"Largest: {ShapeCalculator.Describe(largest!)} ({largest!.Area:F2})");
    }
}
```

---

## Exercises

### Exercise 1: Expression Evaluator
```csharp
// TODO: สร้าง mathematical expression evaluator ด้วย pattern matching

abstract record MathExpr;
record NumberExpr(double Value) : MathExpr;
record BinaryExpr(MathExpr Left, string Op, MathExpr Right) : MathExpr;
record UnaryExpr(string Op, MathExpr Operand) : MathExpr;
record FuncExpr(string Name, MathExpr Argument) : MathExpr;

public class ExprEvaluator
{
    public double Evaluate(MathExpr expr)
    {
        // Handle: +, -, *, /, ^ (power)
        // Handle unary: -, abs
        // Handle functions: sin, cos, tan, sqrt, log
        throw new NotImplementedException();
    }
    
    public string ToInfix(MathExpr expr)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 2: HTTP Router
```csharp
// TODO: สร้าง simple HTTP router ด้วย pattern matching
record HttpRequest(string Method, string Path, Dictionary<string, string> Headers);

public class Router
{
    public string Route(HttpRequest request)
    {
        // Match: Method + Path pattern
        // Support: /api/users/{id} parameters
        // Return: handler description
        throw new NotImplementedException();
    }
}

// Expected:
// GET /api/users -> "List users"
// GET /api/users/123 -> "Get user 123"
// POST /api/users -> "Create user"
// DELETE /api/users/123 -> "Delete user 123"
```

### Exercise 3: JSON-like Tree Processor
```csharp
// TODO: Process tree structure ด้วย recursive patterns

abstract record JsonNode;
record JsonNull : JsonNode;
record JsonBool(bool Value) : JsonNode;
record JsonNumber(double Value) : JsonNode;
record JsonString(string Value) : JsonNode;
record JsonArray(List<JsonNode> Items) : JsonNode;
record JsonObject(Dictionary<string, JsonNode> Properties) : JsonNode;

public class JsonProcessor
{
    // แปลง JsonNode เป็น string
    public string Stringify(JsonNode node) { throw new NotImplementedException(); }
    
    // หา path ใน JSON tree
    public JsonNode? Get(JsonNode root, string path) { throw new NotImplementedException(); }
    
    // นับ total nodes
    public int CountNodes(JsonNode root) { throw new NotImplementedException(); }
}
```

---

## สรุป

✅ **is expression** ใช้ pattern matching inline: `if (x is int n)`, `if (x is > 5)`

✅ **switch expression** (C# 8+) compact กว่า switch statement มาก

✅ **Positional patterns** ใช้ Deconstruct method: `(var x, var y) =>`

✅ **List patterns** (C# 11) match arrays/lists: `[first, .., last]`

✅ **Extended property patterns** (C# 10): `{ Nested.Prop: value }`

✅ **Guard clauses** ด้วย `when` เพิ่มเงื่อนไขใน patterns

✅ **Logical patterns**: `and`, `or`, `not` รวม patterns หลายอัน

✅ Pattern matching ทำให้โค้ดสั้น readable และ exhaustive checking

---

## Part ถัดไป

➡️ **Part 038**: Records (C# 9+) - Immutable data models, value equality, with expressions

---

*Part 037/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
