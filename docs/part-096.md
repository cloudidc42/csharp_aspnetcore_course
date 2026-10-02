# Part 096: Advanced C# 13 Features

## เนื้อหาใน Part นี้
- params collections
- ref struct interfaces
- Overload resolution priority
- New lock type
- Primary constructors (advanced)
- Collection expressions (advanced)
- Interceptors (preview)

---

## 1. params Collections

C# 13 ขยาย `params` ให้รองรับ collection types ต่างๆ ไม่ใช่แค่ arrays

```csharp
// C# 12 และก่อน: params ใช้ได้แค่กับ array
public void LogMessages(params string[] messages)
{
    foreach (var msg in messages) Console.WriteLine(msg);
}

// C# 13: params รองรับ IEnumerable<T>, List<T>, Span<T>, และอื่นๆ
public void LogMessages(params IEnumerable<string> messages)
{
    foreach (var msg in messages) Console.WriteLine(msg);
}

public void ProcessItems(params List<int> items)
{
    Console.WriteLine($"Processing {items.Count} items");
}

public void SumSpan(params ReadOnlySpan<int> values)
{
    int total = 0;
    foreach (var v in values) total += v;
    Console.WriteLine($"Sum: {total}");
}

// ใช้งาน - syntax เหมือนเดิม
LogMessages("Hello", "World", "!");
ProcessItems(1, 2, 3, 4, 5);
SumSpan(10, 20, 30);

// ส่ง collection ได้โดยตรง
var list = new List<int> { 1, 2, 3 };
ProcessItems(list);  // ไม่ต้อง .ToArray() แล้ว!

// ประโยชน์: ลด allocations เมื่อใช้ Span
public static int Sum(params ReadOnlySpan<int> values)
{
    int total = 0;
    foreach (var v in values) total += v;
    return total;
}

// ไม่มี heap allocation!
int result = Sum(1, 2, 3, 4, 5);
```

---

## 2. ref struct Interfaces

C# 13 อนุญาตให้ `ref struct` implement interfaces ได้

```csharp
// C# 12 และก่อน: ref struct ไม่สามารถ implement interface ได้
// C# 13: ref struct สามารถ implement interfaces ได้ (ด้วยข้อจำกัด)

public interface IShape
{
    double Area { get; }
    double Perimeter { get; }
}

// ref struct implement interface
public ref struct Rectangle : IShape
{
    public double Width;
    public double Height;

    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }

    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
}

// Generic methods ที่รองรับทั้ง ref struct และ class
public static void PrintShapeInfo<T>(T shape) where T : IShape, allows ref struct
{
    Console.WriteLine($"Area: {shape.Area:F2}");
    Console.WriteLine($"Perimeter: {shape.Perimeter:F2}");
}

// ใช้งาน
var rect = new Rectangle(5.0, 3.0);
PrintShapeInfo(rect);  // ทำงานได้โดยไม่ boxing

// ref struct ยังคงมีข้อจำกัดเดิม: ไม่สามารถ box เป็น interface ได้
// IShape shape = rect;  // Error! ยังคงทำไม่ได้
// object obj = rect;    // Error! ยังคงทำไม่ได้

// ประโยชน์กับ Span<T>
public interface ISliceable<T> where T : allows ref struct
{
    T Slice(int start, int length);
}

// Span<T> สามารถ implement ใน generic context ได้
public static void ProcessBuffer<TBuffer>(TBuffer buffer)
    where TBuffer : ISliceable<TBuffer>, allows ref struct
{
    var slice = buffer.Slice(0, 10);
    // process slice
}
```

---

## 3. Overload Resolution Priority

ใช้ `[OverloadResolutionPriority]` attribute เพื่อควบคุมลำดับความสำคัญของ overloads

```csharp
using System.Runtime.CompilerServices;

public class TextProcessor
{
    // Priority ต่ำกว่า (default = 0)
    public void Process(IEnumerable<char> chars)
    {
        Console.WriteLine("Processing IEnumerable<char>");
    }

    // Priority สูงกว่า - compiler จะเลือก overload นี้ก่อนเมื่อ ambiguous
    [OverloadResolutionPriority(1)]
    public void Process(ReadOnlySpan<char> span)
    {
        Console.WriteLine("Processing ReadOnlySpan<char> (preferred)");
    }

    [OverloadResolutionPriority(2)]
    public void Process(string text)
    {
        Console.WriteLine("Processing string (highest priority)");
    }
}

var processor = new TextProcessor();
processor.Process("hello");        // → Processing string (highest priority)
processor.Process("hello".AsSpan()); // → Processing ReadOnlySpan<char>

// เหมาะสำหรับ library authors ที่ต้องการ guide compiler
public static class StringExtensions
{
    [OverloadResolutionPriority(1)]
    public static bool Contains(this string text, ReadOnlySpan<char> value)
    {
        return text.AsSpan().Contains(value, StringComparison.Ordinal);
    }

    public static bool Contains(this string text, string value)
    {
        return text.Contains(value);
    }
}
```

---

## 4. New Lock Type

C# 13 แนะนำ `System.Threading.Lock` type ใหม่ที่ดีกว่า `lock` statement แบบเดิม

```csharp
// C# 12 และก่อน: ใช้ object เป็น lock target
private readonly object _syncObj = new();

public void OldWay()
{
    lock (_syncObj)
    {
        // critical section
    }
}

// C# 13: System.Threading.Lock - type ที่ออกแบบมาสำหรับ locking โดยเฉพาะ
private readonly System.Threading.Lock _lock = new();

public void NewWay()
{
    lock (_lock)  // คอมไพเลอร์รู้ว่าเป็น Lock type → optimize
    {
        // critical section
    }
}

// หรือใช้ Enter/Exit แบบ explicit
public void ExplicitLock()
{
    using (_lock.EnterScope())
    {
        // critical section - auto-release เมื่อ scope จบ
    }
}

// ประโยชน์ของ Lock type
// 1. ป้องกัน accidentally ส่ง CancellationToken ไปแทน lock object
// 2. Better diagnostics และ debugging
// 3. อาจ optimize โดย runtime

// ตัวอย่างใน thread-safe repository
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _store = new();
    private readonly Lock _lock = new();

    public bool TryGet(TKey key, out TValue? value)
    {
        lock (_lock)
        {
            return _store.TryGetValue(key, out value);
        }
    }

    public void Set(TKey key, TValue value)
    {
        lock (_lock)
        {
            _store[key] = value;
        }
    }

    public bool TryRemove(TKey key)
    {
        lock (_lock)
        {
            return _store.Remove(key);
        }
    }
}
```

---

## 5. Primary Constructors (Advanced)

Primary constructors ที่ใช้ใน C# 12 มีความสามารถเพิ่มเติมใน C# 13

```csharp
// Primary constructor parameters เป็น mutable ใน C# 13
public class OrderService(IOrderRepository repository, ILogger<OrderService> logger)
{
    // สามารถ reassign parameter ได้ใน body (แต่ไม่แนะนำ)
    public async Task<Order?> GetOrderAsync(Guid id, CancellationToken ct = default)
    {
        logger.LogInformation("Getting order {Id}", id);
        return await repository.GetByIdAsync(id, ct);
    }
}

// Primary constructor กับ base class
public class AuditedService(
    IUnitOfWork uow,
    ICurrentUserService currentUser,
    IDateTimeService dateTime) 
    : BaseService(uow)
{
    protected string? CurrentUserId => currentUser.UserId;
    protected DateTime Now => dateTime.UtcNow;
}

// Primary constructor กับ field initialization
public class ProductService(
    IProductRepository repo,
    ICacheService cache,
    ILogger<ProductService> logger)
{
    // Mix primary constructor params กับ field initialization
    private static readonly TimeSpan DefaultCacheTtl = TimeSpan.FromMinutes(5);
    private int _callCount = 0;

    public async Task<Product?> GetAsync(Guid id)
    {
        Interlocked.Increment(ref _callCount);
        
        var cacheKey = $"product:{id}";
        var cached = await cache.GetAsync<Product>(cacheKey);
        if (cached != null) return cached;
        
        var product = await repo.GetByIdAsync(id);
        if (product != null)
            await cache.SetAsync(cacheKey, product, DefaultCacheTtl);
        
        return product;
    }
    
    public int CallCount => _callCount;
}

// Primary constructor validation
public class EmailAddress(string value)
{
    public string Value { get; } = IsValid(value) ? value 
        : throw new ArgumentException($"Invalid email: {value}");

    private static bool IsValid(string email) =>
        !string.IsNullOrWhiteSpace(email) && email.Contains('@');

    public override string ToString() => Value;
    public static implicit operator string(EmailAddress email) => email.Value;
}

// struct กับ primary constructor
public readonly struct Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount >= 0 ? amount 
        : throw new ArgumentException("Amount cannot be negative");
    public string Currency { get; } = currency.ToUpperInvariant();

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount:N2} {Currency}";
}
```

---

## 6. Collection Expressions (Advanced)

```csharp
// Collection expressions ใน C# 12 ขยายความสามารถใน C# 13

// Spread operator (..) ในสถานการณ์ต่างๆ
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second];  // [1, 2, 3, 4, 5, 6]

// ใช้ใน complex scenarios
List<string> names = ["Alice", "Bob"];
List<string> moreNames = ["Charlie", ..names, "Dave"];

// Collection expressions กับ interfaces
IEnumerable<int> numbers = [1, 2, 3, 4, 5];
IReadOnlyList<string> labels = ["A", "B", "C"];
IReadOnlyCollection<double> values = [1.1, 2.2, 3.3];

// ใช้กับ custom types ที่มี CollectionBuilder attribute
[CollectionBuilder(typeof(ImmutableStack), nameof(ImmutableStack.Create))]
public class ImmutableStack<T> : IEnumerable<T>
{
    // implementation
    public IEnumerator<T> GetEnumerator() => throw new NotImplementedException();
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();
}

// Dictionary expressions (C# 13)
Dictionary<string, int> scores = new()
{
    ["Alice"] = 100,
    ["Bob"] = 95
};

// Inline dictionary init ด้วย collection expression syntax
var config = new Dictionary<string, string>
{
    { "host", "localhost" },
    { "port", "5432" }
};

// ประยุกต์ใช้กับ API responses
public record ApiResponse<T>
{
    public T? Data { get; init; }
    public List<string> Errors { get; init; } = [];
    public Dictionary<string, object> Metadata { get; init; } = [];

    public static ApiResponse<T> Success(T data) => new() 
    { 
        Data = data,
        Metadata = new() { ["timestamp"] = DateTime.UtcNow, ["version"] = "1.0" }
    };
    
    public static ApiResponse<T> Failure(params string[] errors) => new()
    {
        Errors = [..errors]
    };
}

// Pattern matching กับ collection expressions
public string ClassifyList(IList<int> items) => items switch
{
    [] => "empty",
    [var single] => $"single: {single}",
    [var first, var second] => $"pair: {first}, {second}",
    [var head, ..var tail] when tail.Count > 10 => $"large list starting with {head}",
    [var head, ..] => $"list starting with {head}"
};
```

---

## 7. Interceptors (Preview Feature)

Interceptors ช่วยให้ compile-time code replacement สำหรับ Source Generators

```csharp
// ต้องเปิดใช้งานใน .csproj:
// <PropertyGroup>
//   <Features>$(Features);InterceptorsPreview</Features>
// </PropertyGroup>

// Interceptors attribute
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
    public sealed class InterceptsLocationAttribute : Attribute
    {
        public InterceptsLocationAttribute(string filePath, int line, int column) { }
    }
}

// Original code (จะถูก intercept)
// var result = calculator.Add(1, 2);

// Interceptor (generated by Source Generator)
public static class GeneratedInterceptors
{
    [System.Runtime.CompilerServices.InterceptsLocation(
        "Program.cs", line: 10, column: 28)]
    public static int InterceptAdd(this Calculator calc, int a, int b)
    {
        Console.WriteLine($"Intercepted! Adding {a} + {b}");
        var result = a + b;
        Console.WriteLine($"Result: {result}");
        return result;
    }
}

// Use case: EF Core ใช้ Interceptors สำหรับ compile-time query validation
// Use case: Logging injection โดยไม่ต้องแก้ code
// Use case: Dependency injection validation at compile time
```

---

## 8. Other C# 13 Improvements

```csharp
// Implicit index access ใน object initializers
public class Matrix
{
    public int[,] Data { get; set; } = new int[3, 3];
}

// ใช้ ^index ใน initializer
var m = new Matrix
{
    Data = 
    {
        [0, 0] = 1, [0, 1] = 2, [0, 2] = 3,
        [1, 0] = 4, [1, 1] = 5, [1, 2] = 6,
        [2, 0] = 7, [2, 1] = 8, [2, 2] = 9
    }
};

// Escape character \e (ESC character = 0x1B)
Console.Write("\e[31m");  // Set red color in terminal
Console.Write("Red text");
Console.Write("\e[0m");   // Reset

// Method group natural type improvements
var list = new List<int> { 3, 1, 4, 1, 5 };

// C# 13: ดีขึ้นสำหรับ overload resolution ของ method groups
list.Sort(Comparer<int>.Default.Compare);  // ใช้งานได้ดีขึ้น

// Partial properties และ partial events
public partial class MyViewModel
{
    public partial string Name { get; set; }  // Declaration
    public partial event EventHandler? NameChanged;  // Declaration
}

public partial class MyViewModel
{
    private string _name = string.Empty;
    
    public partial string Name  // Implementation
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                NameChanged?.Invoke(this, EventArgs.Empty);
            }
        }
    }

    public partial event EventHandler? NameChanged  // Implementation
    {
        add => _nameChangedHandlers += value;
        remove => _nameChangedHandlers -= value;
    }
    
    private EventHandler? _nameChangedHandlers;
}
```

---

## 9. Benchmark: C# 13 vs Earlier

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class CSharp13Benchmarks
{
    private static readonly int[] Data = Enumerable.Range(0, 1000).ToArray();

    // params IEnumerable<T> vs params T[]
    [Benchmark(Baseline = true)]
    public int SumArray() => SumWithArray(Data);
    
    [Benchmark]
    public int SumSpan() => SumWithSpan(Data);

    private static int SumWithArray(params int[] values)
    {
        int sum = 0;
        foreach (var v in values) sum += v;
        return sum;
    }

    // No allocation for params ReadOnlySpan<T> with inline literals
    private static int SumWithSpan(params ReadOnlySpan<int> values)
    {
        int sum = 0;
        foreach (var v in values) sum += v;
        return sum;
    }

    // Lock benchmark
    private readonly object _oldLock = new();
    private readonly Lock _newLock = new();
    private int _counter = 0;

    [Benchmark]
    public void OldLock()
    {
        lock (_oldLock) { _counter++; }
    }

    [Benchmark]
    public void NewLock()
    {
        lock (_newLock) { _counter++; }
    }
}

BenchmarkRunner.Run<CSharp13Benchmarks>();
```

---

## Exercises / Project Tasks

### Exercise 1: params Span<T>
เขียน utility library ที่ใช้ `params ReadOnlySpan<T>` แทน `params T[]`:
- `Statistics.Mean(params ReadOnlySpan<double>)`
- `Statistics.Max(params ReadOnlySpan<int>)`
- วัด performance ด้วย BenchmarkDotNet

### Exercise 2: ref struct + Interface
สร้าง `ref struct Matrix2x2 : IMultipliable<Matrix2x2>`:
- ใช้ `allows ref struct` ใน generic constraint
- Implement matrix multiplication
- ทดสอบว่าไม่มี boxing

### Exercise 3: Partial Properties
สร้าง ViewModel ด้วย partial properties:
- `UserViewModel` พร้อม validation ใน setter
- Raise `PropertyChanged` event อัตโนมัติ
- สามารถใช้กับ WinForms/WPF binding

### Exercise 4: Collection Expression Pattern
แก้ code เดิมที่ใช้ array/list initialization ให้ใช้ collection expressions:
- ลด noise
- ใช้ spread operator ที่เหมาะสม

---

## สรุป

| Feature | ประโยชน์ |
|---------|---------|
| params collections | รองรับ Span<T> ลด allocations |
| ref struct interfaces | Generic algorithms ไม่ boxing |
| OverloadResolutionPriority | Library authors ควบคุม overload selection |
| Lock type | Type-safe locking |
| Primary constructors | Less boilerplate |
| Collection expressions | Concise initialization |
| Interceptors | Compile-time code injection |
| Partial properties | Source Generator friendly |

---

## Part ถัดไป

**Part 097: Source Generators** - สร้าง code generators ด้วย Roslyn

---

*Part 096/100 | Phase 7/7: ระดับโลก | หลักสูตร C# และ ASP.NET Core*
