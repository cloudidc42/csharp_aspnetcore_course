# Part 036: Tuples และ ValueTuples

## เนื้อหาใน Part นี้
- System.Tuple (เก่า)
- ValueTuple (C# 7+)
- Named elements
- Destructuring
- Tuples ใน return values
- Pattern matching กับ tuples
- โปรแกรมตัวอย่าง: Multiple return values

---

## 1. System.Tuple (แบบเก่า)

```csharp
using System;
using System.Collections.Generic;

class OldTupleExample
{
    static void Main()
    {
        // สร้าง Tuple (แบบเก่า)
        Tuple<int, string> person = new Tuple<int, string>(1, "สมชาย");
        
        // เข้าถึงด้วย Item1, Item2, ...
        Console.WriteLine($"Id: {person.Item1}, Name: {person.Item2}");
        
        // Tuple.Create (helper)
        var point = Tuple.Create(10.5, 20.3);
        Console.WriteLine($"X: {point.Item1}, Y: {point.Item2}");
        
        // Tuple 3 elements
        var employee = Tuple.Create("สมหญิง", "IT", 50000.0);
        Console.WriteLine($"{employee.Item1} ฝ่าย {employee.Item2} เงินเดือน {employee.Item3:N0}");
        
        // ใช้เป็น return type
        var result = GetMinMax(new[] { 5, 2, 8, 1, 9, 3 });
        Console.WriteLine($"Min: {result.Item1}, Max: {result.Item2}");
        
        // ปัญหาของ System.Tuple
        // 1. เข้าถึงด้วย Item1, Item2 ไม่ readable
        // 2. เป็น reference type (allocates on heap)
        // 3. Immutable - ดี แต่ verbose
        
        // Nested Tuples (เก่า)
        var nested = Tuple.Create(1, Tuple.Create("hello", true));
        Console.WriteLine($"{nested.Item1}, {nested.Item2.Item1}, {nested.Item2.Item2}");
    }
    
    static Tuple<int, int> GetMinMax(IEnumerable<int> numbers)
    {
        int min = int.MaxValue, max = int.MinValue;
        foreach (var n in numbers)
        {
            if (n < min) min = n;
            if (n > max) max = n;
        }
        return Tuple.Create(min, max);
    }
}
```

---

## 2. ValueTuple (C# 7+)

ValueTuple เป็น **value type** (struct) ที่เร็วกว่าและ readable กว่า

```csharp
using System;
using System.Collections.Generic;

class ValueTupleExample
{
    static void Main()
    {
        // 3 วิธีสร้าง ValueTuple
        
        // 1. Syntax น้อย
        (int, string) person1 = (1, "สมชาย");
        
        // 2. Named elements
        (int Id, string Name) person2 = (1, "สมชาย");
        
        // 3. ValueTuple.Create
        var person3 = ValueTuple.Create(1, "สมชาย");
        
        // เข้าถึงด้วยชื่อหรือ Item1, Item2
        Console.WriteLine($"Id: {person2.Id}, Name: {person2.Name}");
        Console.WriteLine($"Id: {person2.Item1}, Name: {person2.Item2}"); // ก็ได้
        
        // ValueTuple เป็น value type
        var original = (X: 10, Y: 20);
        var copy = original;
        copy.X = 99; // ไม่กระทบ original
        Console.WriteLine($"Original: ({original.X}, {original.Y})");
        Console.WriteLine($"Copy: ({copy.X}, {copy.Y})");
        
        // Mutable (ต่างจาก record)
        var mutable = (Name: "สมชาย", Age: 25);
        mutable.Age = 26; // ได้!
        Console.WriteLine($"After change: {mutable.Name}, {mutable.Age}");
        
        // ขนาดต่างกัน
        Console.WriteLine($"Tuple<int,string> size: reference type");
        Console.WriteLine($"(int, string) size: {System.Runtime.InteropServices.Marshal.SizeOf<(int, string)>()} bytes... wait, tuples with reference types differ");
        
        // Equality
        var t1 = (1, "hello");
        var t2 = (1, "hello");
        Console.WriteLine($"t1 == t2: {t1 == t2}"); // true - structural equality
        
        // Max 8 elements (สร้าง nested สำหรับมากกว่า)
        var big = (1, 2, 3, 4, 5, 6, 7, 8);
        Console.WriteLine($"Element 8: {big.Item8}");
    }
}
```

---

## 3. Named Elements

```csharp
using System;

class NamedElementsExample
{
    static void Main()
    {
        // Named elements ทำให้ readable
        var color = (Red: 255, Green: 128, Blue: 0);
        Console.WriteLine($"RGB({color.Red}, {color.Green}, {color.Blue})");
        
        // Named elements ใน return type
        var stats = CalculateStats(new double[] { 1, 2, 3, 4, 5 });
        Console.WriteLine($"Min={stats.Min}, Max={stats.Max}, Avg={stats.Average:F2}");
        
        // Named elements ใน variable declaration
        (double Min, double Max, double Average) result = CalculateStats(new double[] { 10, 20, 30 });
        
        // Inferred names (C# 7.1+)
        int userId = 42;
        string userName = "สมชาย";
        var user = (userId, userName); // names ถูก infer อัตโนมัติ
        Console.WriteLine($"User {user.userId}: {user.userName}");
        
        // ใน LINQ
        var people = new[]
        {
            new { Name = "สมชาย", Age = 25 },
            new { Name = "สมหญิง", Age = 30 },
            new { Name = "วิชัย", Age = 22 }
        };
        
        var query = people
            .Select(p => (p.Name, p.Age, IsAdult: p.Age >= 18))
            .Where(t => t.IsAdult)
            .OrderBy(t => t.Age);
        
        foreach (var (name, age, isAdult) in query)
        {
            Console.WriteLine($"  {name}: {age} ปี (IsAdult: {isAdult})");
        }
        
        // Named tuples ใน Dictionary
        var locations = new Dictionary<string, (double Lat, double Lng)>
        {
            ["กรุงเทพ"] = (13.7563, 100.5018),
            ["เชียงใหม่"] = (18.7883, 98.9853),
            ["ภูเก็ต"] = (7.8804, 98.3923)
        };
        
        foreach (var (city, coords) in locations)
        {
            Console.WriteLine($"{city}: ({coords.Lat:F4}, {coords.Lng:F4})");
        }
    }
    
    static (double Min, double Max, double Average, double StdDev) CalculateStats(double[] data)
    {
        double min = double.MaxValue, max = double.MinValue, sum = 0;
        
        foreach (double d in data)
        {
            if (d < min) min = d;
            if (d > max) max = d;
            sum += d;
        }
        
        double avg = sum / data.Length;
        double variance = data.Average(d => Math.Pow(d - avg, 2));
        
        return (min, max, avg, Math.Sqrt(variance));
    }
}
```

---

## 4. Destructuring

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

class DestructuringExample
{
    static void Main()
    {
        // Basic destructuring
        var (id, name) = GetUser(1);
        Console.WriteLine($"User: {id} - {name}");
        
        // Destructuring กับ named tuple
        (int UserId, string UserName, string Email) = GetUserDetails(1);
        Console.WriteLine($"Details: {UserId}, {UserName}, {Email}");
        
        // Ignore elements ด้วย _
        var (_, price, _) = GetProductInfo();
        Console.WriteLine($"Price only: {price:N2}");
        
        // Destructuring ใน foreach
        var orders = new List<(int Id, string Product, decimal Price)>
        {
            (1, "iPhone 16", 49900),
            (2, "MacBook Pro", 89900),
            (3, "iPad Air", 29900)
        };
        
        decimal total = 0;
        foreach (var (orderId, product, orderPrice) in orders)
        {
            Console.WriteLine($"  [{orderId}] {product}: {orderPrice:N0}");
            total += orderPrice;
        }
        Console.WriteLine($"Total: {total:N0}");
        
        // Swap ด้วย tuple
        int x = 10, y = 20;
        Console.WriteLine($"Before: x={x}, y={y}");
        (x, y) = (y, x);
        Console.WriteLine($"After swap: x={x}, y={y}");
        
        // Destructuring ใน LINQ with Select
        var processed = orders
            .Select(o => (o.Id, o.Product, Discounted: o.Price * 0.9m))
            .ToList();
        
        foreach (var (orderId, product, discounted) in processed)
        {
            Console.WriteLine($"  [{orderId}] {product}: {discounted:N0} (ลด 10%)");
        }
        
        // Deconstruct custom class (ต้อง define Deconstruct method)
        var point = new Point2D(10, 20);
        var (px, py) = point;
        Console.WriteLine($"Point: ({px}, {py})");
        
        // Nested destructuring
        var data = ((1, "A"), (2, "B"));
        var ((n1, s1), (n2, s2)) = data;
        Console.WriteLine($"{n1}:{s1}, {n2}:{s2}");
    }
    
    static (int Id, string Name) GetUser(int id) => (id, $"User_{id}");
    
    static (int UserId, string UserName, string Email) GetUserDetails(int id) =>
        (id, $"User_{id}", $"user{id}@example.com");
    
    static (string Name, decimal Price, int Stock) GetProductInfo() =>
        ("iPhone 16", 49900, 10);
}

// Custom Deconstruct
public class Point2D
{
    public double X { get; }
    public double Y { get; }
    
    public Point2D(double x, double y) { X = x; Y = y; }
    
    // Define Deconstruct method เพื่อ support destructuring
    public void Deconstruct(out double x, out double y)
    {
        x = X;
        y = Y;
    }
}

// Extension method Deconstruct
public static class KeyValuePairExtensions
{
    public static void Deconstruct<TKey, TValue>(
        this KeyValuePair<TKey, TValue> kv,
        out TKey key,
        out TValue value)
    {
        key = kv.Key;
        value = kv.Value;
    }
}
```

---

## 5. Tuples ใน Return Values

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.RegularExpressions;

class TupleReturnValues
{
    // Pattern: return multiple values
    static (bool IsValid, string Message) ValidateAge(int age)
    {
        if (age < 0) return (false, "อายุต้องไม่ติดลบ");
        if (age > 150) return (false, "อายุไม่น่าจะเกิน 150 ปี");
        return (true, "ถูกต้อง");
    }
    
    // Pattern: return result + metadata
    static (T? Value, bool Found, TimeSpan SearchTime) SearchDatabase<T>(string query)
    {
        var start = DateTime.Now;
        // จำลองการค้นหา
        System.Threading.Thread.Sleep(10);
        T? result = default;
        bool found = query.Length > 3;
        return (result, found, DateTime.Now - start);
    }
    
    // Pattern: return split result
    static (string[] Valid, string[] Invalid) SplitByValidity(IEnumerable<string> emails)
    {
        var emailRegex = new Regex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$");
        var (valid, invalid) = emails.Partition(e => emailRegex.IsMatch(e));
        return (valid.ToArray(), invalid.ToArray());
    }
    
    // Pattern: TryGet pattern
    static bool TryParseDate(string input, out (int Year, int Month, int Day) date)
    {
        date = default;
        if (!DateTime.TryParse(input, out DateTime dt)) return false;
        date = (dt.Year, dt.Month, dt.Day);
        return true;
    }
    
    // Pattern: Pagination result
    static (IEnumerable<T> Items, int TotalCount, int Page, int PageSize, int TotalPages)
        Paginate<T>(IEnumerable<T> source, int page, int pageSize)
    {
        var list = source.ToList();
        int total = list.Count;
        var items = list.Skip((page - 1) * pageSize).Take(pageSize);
        int totalPages = (int)Math.Ceiling((double)total / pageSize);
        return (items, total, page, pageSize, totalPages);
    }
    
    static void Main()
    {
        // ValidateAge
        var (isValid, message) = ValidateAge(25);
        Console.WriteLine($"Valid: {isValid}, Message: {message}");
        
        var (isValid2, message2) = ValidateAge(-5);
        Console.WriteLine($"Valid: {isValid2}, Message: {message2}");
        
        // SplitByValidity
        var emails = new[] { "valid@test.com", "bad-email", "another@test.com", "wrong" };
        var (valid, invalid) = SplitByValidity(emails);
        Console.WriteLine($"\nValid emails: {string.Join(", ", valid)}");
        Console.WriteLine($"Invalid emails: {string.Join(", ", invalid)}");
        
        // TryParseDate
        if (TryParseDate("2024-03-15", out var date))
        {
            Console.WriteLine($"\nParsed: Year={date.Year}, Month={date.Month}, Day={date.Day}");
        }
        
        // Pagination
        var numbers = Enumerable.Range(1, 100);
        var (items, total, page, size, totalPages) = Paginate(numbers, page: 3, pageSize: 10);
        Console.WriteLine($"\nPage {page}/{totalPages}: [{string.Join(", ", items)}]");
        Console.WriteLine($"Total: {total} items, {size} per page");
    }
}

// Extension method helper
public static class EnumerableExtensions
{
    public static (IEnumerable<T> First, IEnumerable<T> Second) Partition<T>(
        this IEnumerable<T> source,
        Func<T, bool> predicate)
    {
        var list = source.ToList();
        return (list.Where(predicate), list.Where(x => !predicate(x)));
    }
}
```

---

## 6. Pattern Matching กับ Tuples

```csharp
using System;
using System.Collections.Generic;

class TuplePatternMatching
{
    // State machine ด้วย tuple pattern
    enum TrafficLight { Red, Yellow, Green }
    
    static string GetTrafficAction(TrafficLight current) =>
        current switch
        {
            TrafficLight.Red => "หยุด",
            TrafficLight.Yellow => "ระวัง",
            TrafficLight.Green => "ผ่านได้",
            _ => "ไม่รู้จัก"
        };
    
    // State transition กับ tuple
    static TrafficLight NextLight(TrafficLight current) =>
        current switch
        {
            TrafficLight.Red => TrafficLight.Green,
            TrafficLight.Green => TrafficLight.Yellow,
            TrafficLight.Yellow => TrafficLight.Red,
            _ => current
        };
    
    // Rock Paper Scissors
    enum Hand { Rock, Paper, Scissors }
    
    static string PlayGame(Hand player1, Hand player2) =>
        (player1, player2) switch
        {
            (Hand.Rock, Hand.Scissors) => "ผู้เล่น 1 ชนะ",
            (Hand.Paper, Hand.Rock) => "ผู้เล่น 1 ชนะ",
            (Hand.Scissors, Hand.Paper) => "ผู้เล่น 1 ชนะ",
            (Hand.Scissors, Hand.Rock) => "ผู้เล่น 2 ชนะ",
            (Hand.Rock, Hand.Paper) => "ผู้เล่น 2 ชนะ",
            (Hand.Paper, Hand.Scissors) => "ผู้เล่น 2 ชนะ",
            _ => "เสมอ"
        };
    
    // Complex tuple patterns
    static string ClassifyPoint(double x, double y) =>
        (x, y) switch
        {
            (0, 0) => "จุดกำเนิด",
            (0, _) => "แกน Y",
            (_, 0) => "แกน X",
            ( > 0, > 0) => "Quadrant I (บวก/บวก)",
            ( < 0, > 0) => "Quadrant II (ลบ/บวก)",
            ( < 0, < 0) => "Quadrant III (ลบ/ลบ)",
            ( > 0, < 0) => "Quadrant IV (บวก/ลบ)",
            _ => "ไม่รู้"
        };
    
    // Role-based action
    record User(string Name, bool IsAdmin, bool IsModerator);
    
    static string GetUserAction(User user, string resource) =>
        (user.IsAdmin, user.IsModerator, resource) switch
        {
            (true, _, _) => $"{user.Name}: เข้าถึงได้ทุกอย่าง",
            (false, true, "comments") => $"{user.Name}: จัดการ comments ได้",
            (false, true, "users") => $"{user.Name}: ดู users ได้แต่แก้ไขไม่ได้",
            (false, false, "profile") => $"{user.Name}: แก้ไข profile ตัวเองได้",
            _ => $"{user.Name}: ไม่มีสิทธิ์"
        };
    
    static void Main()
    {
        // Traffic light
        Console.WriteLine("Traffic Light:");
        var light = TrafficLight.Red;
        for (int i = 0; i < 6; i++)
        {
            Console.WriteLine($"  {light}: {GetTrafficAction(light)}");
            light = NextLight(light);
        }
        
        // Rock Paper Scissors
        Console.WriteLine("\nRock Paper Scissors:");
        Console.WriteLine(PlayGame(Hand.Rock, Hand.Scissors));
        Console.WriteLine(PlayGame(Hand.Paper, Hand.Rock));
        Console.WriteLine(PlayGame(Hand.Rock, Hand.Rock));
        
        // Classify points
        Console.WriteLine("\nClassify Points:");
        Console.WriteLine($"(0, 0): {ClassifyPoint(0, 0)}");
        Console.WriteLine($"(5, 3): {ClassifyPoint(5, 3)}");
        Console.WriteLine($"(-2, 4): {ClassifyPoint(-2, 4)}");
        Console.WriteLine($"(-1, -1): {ClassifyPoint(-1, -1)}");
        
        // User permissions
        Console.WriteLine("\nUser Permissions:");
        var admin = new User("Admin", IsAdmin: true, IsModerator: false);
        var mod = new User("Mod", IsAdmin: false, IsModerator: true);
        var user = new User("User", IsAdmin: false, IsModerator: false);
        
        Console.WriteLine(GetUserAction(admin, "anything"));
        Console.WriteLine(GetUserAction(mod, "comments"));
        Console.WriteLine(GetUserAction(user, "profile"));
        Console.WriteLine(GetUserAction(user, "admin"));
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Multiple Return Values

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;
using System.Text.Json;

// HTTP Response wrapper
public record HttpResult<T>(
    bool Success,
    T? Data,
    string? ErrorMessage,
    int StatusCode
)
{
    public static HttpResult<T> Ok(T data, int statusCode = 200) =>
        new(true, data, null, statusCode);
    
    public static HttpResult<T> Fail(string error, int statusCode = 500) =>
        new(false, default, error, statusCode);
}

// Complex calculation results
public static class StatisticsCalculator
{
    public static (
        double Mean,
        double Median,
        double Mode,
        double StdDev,
        double Min,
        double Max,
        double Range,
        double Q1,
        double Q3,
        double IQR
    ) FullStatistics(IEnumerable<double> data)
    {
        var sorted = data.OrderBy(x => x).ToList();
        int n = sorted.Count;
        
        if (n == 0)
            return (0, 0, 0, 0, 0, 0, 0, 0, 0, 0);
        
        double mean = sorted.Average();
        
        double median = n % 2 == 0
            ? (sorted[n / 2 - 1] + sorted[n / 2]) / 2.0
            : sorted[n / 2];
        
        double mode = sorted
            .GroupBy(x => x)
            .OrderByDescending(g => g.Count())
            .First().Key;
        
        double variance = sorted.Average(x => Math.Pow(x - mean, 2));
        double stdDev = Math.Sqrt(variance);
        
        double min = sorted.First();
        double max = sorted.Last();
        double range = max - min;
        
        double q1 = GetPercentile(sorted, 25);
        double q3 = GetPercentile(sorted, 75);
        double iqr = q3 - q1;
        
        return (mean, median, mode, stdDev, min, max, range, q1, q3, iqr);
    }
    
    private static double GetPercentile(List<double> sorted, double percentile)
    {
        double position = (percentile / 100) * (sorted.Count - 1);
        int lower = (int)position;
        double fraction = position - lower;
        
        if (lower >= sorted.Count - 1)
            return sorted.Last();
        
        return sorted[lower] + fraction * (sorted[lower + 1] - sorted[lower]);
    }
}

// Parser with error reporting
public class Parser
{
    public static (bool Success, object? Value, string? Error, int Position) Parse(
        string input,
        string pattern)
    {
        if (string.IsNullOrEmpty(input))
            return (false, null, "Input ว่างเปล่า", 0);
        
        var regex = new System.Text.RegularExpressions.Regex(pattern);
        var match = regex.Match(input);
        
        if (!match.Success)
            return (false, null, $"ไม่ตรงกับ pattern: {pattern}", match.Index);
        
        return (true, match.Value, null, match.Index);
    }
}

// File operation results
public class FileOperations
{
    public static async Task<(bool Success, long BytesRead, int LineCount, string[] FirstLines, string? Error)>
        ReadFileStatsAsync(string path)
    {
        try
        {
            var content = await System.IO.File.ReadAllTextAsync(path);
            var lines = content.Split('\n');
            return (
                Success: true,
                BytesRead: System.Text.Encoding.UTF8.GetByteCount(content),
                LineCount: lines.Length,
                FirstLines: lines.Take(3).ToArray(),
                Error: null
            );
        }
        catch (Exception ex)
        {
            return (false, 0, 0, Array.Empty<string>(), ex.Message);
        }
    }
}

// Geographic calculations
public static class GeoCalculator
{
    public static (double Distance, double Bearing, string Direction) CalculateBearing(
        double lat1, double lon1,
        double lat2, double lon2)
    {
        const double R = 6371; // Earth radius km
        
        double dLat = ToRad(lat2 - lat1);
        double dLon = ToRad(lon2 - lon1);
        
        double a = Math.Sin(dLat / 2) * Math.Sin(dLat / 2) +
                   Math.Cos(ToRad(lat1)) * Math.Cos(ToRad(lat2)) *
                   Math.Sin(dLon / 2) * Math.Sin(dLon / 2);
        
        double c = 2 * Math.Atan2(Math.Sqrt(a), Math.Sqrt(1 - a));
        double distance = R * c;
        
        double y = Math.Sin(ToRad(lon2 - lon1)) * Math.Cos(ToRad(lat2));
        double x = Math.Cos(ToRad(lat1)) * Math.Sin(ToRad(lat2)) -
                   Math.Sin(ToRad(lat1)) * Math.Cos(ToRad(lat2)) * Math.Cos(ToRad(lon2 - lon1));
        
        double bearing = (ToDeg(Math.Atan2(y, x)) + 360) % 360;
        
        string direction = bearing switch
        {
            >= 337.5 or < 22.5 => "เหนือ",
            >= 22.5 and < 67.5 => "ตะวันออกเฉียงเหนือ",
            >= 67.5 and < 112.5 => "ตะวันออก",
            >= 112.5 and < 157.5 => "ตะวันออกเฉียงใต้",
            >= 157.5 and < 202.5 => "ใต้",
            >= 202.5 and < 247.5 => "ตะวันตกเฉียงใต้",
            >= 247.5 and < 292.5 => "ตะวันตก",
            _ => "ตะวันตกเฉียงเหนือ"
        };
        
        return (distance, bearing, direction);
    }
    
    private static double ToRad(double deg) => deg * Math.PI / 180;
    private static double ToDeg(double rad) => rad * 180 / Math.PI;
}

// Main Program
class Program
{
    static void Main()
    {
        Console.WriteLine("===== Tuples Demo: Multiple Return Values =====\n");
        
        // Statistics
        var data = new double[] { 2, 4, 4, 4, 5, 5, 7, 9, 10, 10, 12, 15 };
        var stats = StatisticsCalculator.FullStatistics(data);
        
        Console.WriteLine("--- Statistics ---");
        Console.WriteLine($"Data: [{string.Join(", ", data)}]");
        Console.WriteLine($"Mean:   {stats.Mean:F2}");
        Console.WriteLine($"Median: {stats.Median:F2}");
        Console.WriteLine($"Mode:   {stats.Mode:F2}");
        Console.WriteLine($"StdDev: {stats.StdDev:F2}");
        Console.WriteLine($"Min:    {stats.Min:F2}, Max: {stats.Max:F2}");
        Console.WriteLine($"Q1:     {stats.Q1:F2}, Q3: {stats.Q3:F2}");
        Console.WriteLine($"IQR:    {stats.IQR:F2}");
        
        // Geographic
        Console.WriteLine("\n--- Geographic Calculation ---");
        var (dist, bearing, dir) = GeoCalculator.CalculateBearing(
            13.7563, 100.5018,  // กรุงเทพ
            18.7883, 98.9853    // เชียงใหม่
        );
        Console.WriteLine($"กรุงเทพ -> เชียงใหม่:");
        Console.WriteLine($"  ระยะทาง: {dist:F0} km");
        Console.WriteLine($"  ทิศทาง: {bearing:F1}° ({dir})");
        
        // Pattern matching
        Console.WriteLine("\n--- Pattern Matching ---");
        var points = new[] { (0.0, 0.0), (1.0, 2.0), (-3.0, 4.0), (5.0, -6.0), (-7.0, -8.0) };
        
        foreach (var (x, y) in points)
        {
            string quadrant = (x, y) switch
            {
                (0, 0) => "จุดกำเนิด",
                (0, _) => "แกน Y",
                (_, 0) => "แกน X",
                (> 0, > 0) => "Q1",
                (< 0, > 0) => "Q2",
                (< 0, < 0) => "Q3",
                (> 0, < 0) => "Q4",
                _ => "?"
            };
            Console.WriteLine($"  ({x}, {y}) -> {quadrant}");
        }
        
        // Destructuring showcase
        Console.WriteLine("\n--- Destructuring ---");
        var orders = new List<(int Id, string Product, decimal Price, int Qty)>
        {
            (1, "Coffee", 45, 3),
            (2, "Tea", 35, 2),
            (3, "Cake", 120, 1)
        };
        
        decimal grandTotal = 0;
        foreach (var (id, product, price, qty) in orders)
        {
            decimal subtotal = price * qty;
            grandTotal += subtotal;
            Console.WriteLine($"  [{id}] {product}: {price:N0} x {qty} = {subtotal:N0}");
        }
        Console.WriteLine($"  รวมทั้งหมด: {grandTotal:N0} บาท");
    }
}
```

---

## Exercises

### Exercise 1: Color Mixer
```csharp
// TODO: สร้าง color mixer ด้วย tuples
public static class ColorMixer
{
    // Mix สีสอง RGBA
    public static (byte R, byte G, byte B, byte A) Mix(
        (byte R, byte G, byte B, byte A) color1,
        (byte R, byte G, byte B, byte A) color2,
        double ratio = 0.5)
    {
        throw new NotImplementedException();
    }
    
    // แปลงระหว่าง RGB และ HSL
    public static (double H, double S, double L) RgbToHsl(byte r, byte g, byte b)
    {
        throw new NotImplementedException();
    }
    
    public static (byte R, byte G, byte B) HslToRgb(double h, double s, double l)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 2: Matrix Operations
```csharp
// TODO: Matrix operations ที่ return tuples
public static class MatrixOps
{
    // LU Decomposition ของ matrix 2x2
    // Return (L matrix, U matrix, success)
    public static ((double, double, double, double) L, (double, double, double, double) U, bool Success)
        LUDecompose2x2(double a11, double a12, double a21, double a22)
    {
        throw new NotImplementedException();
    }
    
    // Solve 2x2 linear system Ax = b
    // Return (x1, x2, bool HasSolution)
    public static (double X1, double X2, bool HasSolution)
        Solve2x2(double a11, double a12, double a21, double a22, double b1, double b2)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: Schedule Builder
```csharp
// TODO: Time slot allocator
public class ScheduleBuilder
{
    // หา time slots ที่ว่างระหว่าง busy slots
    public IEnumerable<(TimeSpan Start, TimeSpan End, TimeSpan Duration)> FindFreeSlots(
        IEnumerable<(TimeSpan Start, TimeSpan End)> busySlots,
        TimeSpan workStart,
        TimeSpan workEnd,
        TimeSpan minDuration)
    {
        throw new NotImplementedException();
    }
}
```

---

## สรุป

✅ **System.Tuple** เป็น reference type แบบเก่า ใช้ Item1, Item2 - ไม่ readable

✅ **ValueTuple** (C# 7+) เป็น value type เร็วกว่า รองรับ named elements

✅ **Named elements** ทำให้ code readable: (int Id, string Name) แทน (int, string)

✅ **Destructuring** ช่วย extract ค่าได้สะดวก: var (id, name) = GetUser()

✅ ใช้ **\_** เพื่อ ignore elements ที่ไม่ต้องการ

✅ Tuples เหมาะสำหรับ **multiple return values** โดยไม่ต้องสร้าง class ใหม่

✅ **Pattern matching กับ tuples** ทำ state machines และ rule engines ได้ดี

✅ ข้อเสีย: ถ้า tuple ใหญ่ ควรสร้าง class/record แทน

---

## Part ถัดไป

➡️ **Part 037**: Pattern Matching ขั้นสูง (C# 9-13) - switch expressions, positional patterns, list patterns

---

*Part 036/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
