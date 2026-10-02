# Part 023: Generics

## เนื้อหาใน Part นี้
- Generic คืออะไร และทำไมต้องใช้
- Generic Class และ Generic Method
- Type Constraints (where T : ...)
- Generic Interface
- Covariance และ Contravariance
- Generic Utility Classes ตัวอย่าง
- โปรแกรมตัวอย่าง: Generic Stack Implementation

---

## 1. Generic คืออะไร?

Generic ให้เราเขียน code ที่ทำงานได้กับหลาย Type โดยไม่ต้อง duplicate code และยังคง Type Safety

```csharp
// ปัญหาก่อนใช้ Generics
public class ObjectBox
{
    private object _value;
    public ObjectBox(object value) { _value = value; }
    public object GetValue() => _value;
}

// ต้อง cast และอาจเกิด InvalidCastException
ObjectBox box = new ObjectBox(42);
int num = (int)box.GetValue(); // ต้อง cast
string str = (string)box.GetValue(); // Runtime Error!

// ===== แก้ปัญหาด้วย Generic =====
public class Box<T>
{
    private T _value;
    public Box(T value) { _value = value; }
    public T GetValue() => _value;
}

Box<int> intBox = new Box<int>(42);
int safeNum = intBox.GetValue(); // ไม่ต้อง cast, type-safe

Box<string> strBox = new Box<string>("Hello");
string safeStr = strBox.GetValue(); // ปลอดภัย

// Generic vs Non-generic Performance
// Non-generic (object): ต้องทำ Boxing สำหรับ value types
// Generic: ไม่มี Boxing, เร็วกว่า
```

---

## 2. Generic Class

```csharp
// Generic class พื้นฐาน
public class Pair<T1, T2>
{
    public T1 First { get; }
    public T2 Second { get; }

    public Pair(T1 first, T2 second)
    {
        First = first;
        Second = second;
    }

    public void Deconstruct(out T1 first, out T2 second)
    {
        first = First;
        second = Second;
    }

    public override string ToString() => $"({First}, {Second})";

    // Swap คืน Pair ใหม่
    public Pair<T2, T1> Swap() => new Pair<T2, T1>(Second, First);
}

// ใช้งาน
var pair1 = new Pair<string, int>("Age", 25);
Console.WriteLine(pair1); // (Age, 25)

var (key, value) = pair1; // Deconstruct
Console.WriteLine($"{key}: {value}"); // Age: 25

var swapped = pair1.Swap();
Console.WriteLine(swapped); // (25, Age)

var pair2 = new Pair<double, bool>(3.14, true);
Console.WriteLine(pair2); // (3.14, True)
```

### 2.1 Generic Repository Pattern

```csharp
// Generic Repository - pattern ที่ใช้บ่อยมาก
public interface IRepository<T, TId>
{
    T? GetById(TId id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    bool Delete(TId id);
    int Count { get; }
}

// Generic In-memory Repository
public class InMemoryRepository<T, TId> : IRepository<T, TId> where TId : notnull
{
    protected readonly Dictionary<TId, T> _store = new();
    private readonly Func<T, TId> _idSelector;

    public InMemoryRepository(Func<T, TId> idSelector)
    {
        _idSelector = idSelector;
    }

    public T? GetById(TId id)
    {
        _store.TryGetValue(id, out T? entity);
        return entity;
    }

    public IEnumerable<T> GetAll() => _store.Values;

    public void Add(T entity)
    {
        var id = _idSelector(entity);
        if (!_store.TryAdd(id, entity))
            throw new InvalidOperationException($"Entity with id '{id}' already exists");
    }

    public void Update(T entity)
    {
        var id = _idSelector(entity);
        if (!_store.ContainsKey(id))
            throw new KeyNotFoundException($"Entity with id '{id}' not found");
        _store[id] = entity;
    }

    public bool Delete(TId id) => _store.Remove(id);

    public int Count => _store.Count;
}

// Model
public record Product(int Id, string Name, decimal Price);

// Specific repository
public class ProductRepository : InMemoryRepository<Product, int>
{
    public ProductRepository() : base(p => p.Id) { }

    public IEnumerable<Product> GetByPriceRange(decimal min, decimal max) =>
        _store.Values.Where(p => p.Price >= min && p.Price <= max);

    public IEnumerable<Product> SearchByName(string term) =>
        _store.Values.Where(p => p.Name.Contains(term, StringComparison.OrdinalIgnoreCase));
}

// ใช้งาน
var repo = new ProductRepository();
repo.Add(new Product(1, "Laptop", 25000));
repo.Add(new Product(2, "Mouse", 500));
repo.Add(new Product(3, "Keyboard", 1500));

var affordable = repo.GetByPriceRange(400, 2000);
foreach (var p in affordable)
    Console.WriteLine($"{p.Name}: {p.Price:C}");
```

---

## 3. Generic Method

```csharp
public static class GenericUtils
{
    // Generic swap
    public static void Swap<T>(ref T a, ref T b)
    {
        T temp = a;
        a = b;
        b = temp;
    }

    // Generic max
    public static T Max<T>(T a, T b) where T : IComparable<T>
    {
        return a.CompareTo(b) >= 0 ? a : b;
    }

    // Generic min
    public static T Min<T>(T a, T b) where T : IComparable<T>
    {
        return a.CompareTo(b) <= 0 ? a : b;
    }

    // Generic clamp
    public static T Clamp<T>(T value, T min, T max) where T : IComparable<T>
    {
        if (value.CompareTo(min) < 0) return min;
        if (value.CompareTo(max) > 0) return max;
        return value;
    }

    // Generic filter
    public static List<T> Filter<T>(IEnumerable<T> source, Func<T, bool> predicate)
    {
        var result = new List<T>();
        foreach (T item in source)
        {
            if (predicate(item))
                result.Add(item);
        }
        return result;
    }

    // Generic transform (Map)
    public static List<TResult> Map<TSource, TResult>(
        IEnumerable<TSource> source,
        Func<TSource, TResult> transform)
    {
        var result = new List<TResult>();
        foreach (TSource item in source)
            result.Add(transform(item));
        return result;
    }

    // Generic reduce (Fold)
    public static TAccumulate Reduce<TSource, TAccumulate>(
        IEnumerable<TSource> source,
        TAccumulate seed,
        Func<TAccumulate, TSource, TAccumulate> func)
    {
        TAccumulate result = seed;
        foreach (TSource item in source)
            result = func(result, item);
        return result;
    }

    // Generic null check
    public static T RequireNotNull<T>(T? value, string paramName = "value") where T : class
    {
        return value ?? throw new ArgumentNullException(paramName);
    }
}

// ใช้งาน
int x = 5, y = 10;
GenericUtils.Swap(ref x, ref y);
Console.WriteLine($"x={x}, y={y}"); // x=10, y=5

Console.WriteLine(GenericUtils.Max(3, 7));          // 7
Console.WriteLine(GenericUtils.Max("apple", "zebra")); // zebra
Console.WriteLine(GenericUtils.Clamp(15, 0, 10));   // 10
Console.WriteLine(GenericUtils.Clamp(-5, 0, 10));   // 0

var numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };
var evens = GenericUtils.Filter(numbers, n => n % 2 == 0);
var doubled = GenericUtils.Map(numbers, n => n * 2);
var sum = GenericUtils.Reduce(numbers, 0, (acc, n) => acc + n);

Console.WriteLine(string.Join(", ", evens));   // 2, 4, 6, 8
Console.WriteLine(string.Join(", ", doubled)); // 2, 4, 6, 8, 10, 12, 14, 16
Console.WriteLine(sum);                         // 36
```

---

## 4. Type Constraints (where T : ...)

Constraints กำหนดข้อจำกัดว่า T ต้องเป็นอะไร

```csharp
// Constraint ชนิดต่างๆ
public class ConstraintExamples
{
    // where T : class - T ต้องเป็น Reference Type
    public static T? GetFirst<T>(List<T> list) where T : class
    {
        return list.Count > 0 ? list[0] : null;
    }

    // where T : struct - T ต้องเป็น Value Type
    public static T DefaultValue<T>() where T : struct
    {
        return default(T);
    }

    // where T : new() - T ต้องมี parameterless constructor
    public static T CreateNew<T>() where T : new()
    {
        return new T();
    }

    // where T : BaseClass - T ต้องสืบทอดจาก BaseClass
    public static void ProcessEntity<T>(T entity) where T : Entity
    {
        Console.WriteLine($"Processing entity id: {entity.Id}");
    }

    // where T : IInterface - T ต้อง implement interface
    public static void Print<T>(T item) where T : IFormattable
    {
        Console.WriteLine(item.ToString("G", null));
    }

    // หลาย constraints พร้อมกัน
    public static T CreateAndInit<T>() where T : Entity, new()
    {
        T instance = new T();
        instance.CreatedAt = DateTime.Now;
        return instance;
    }

    // where T : notnull (C# 8+) - T ไม่ใช่ null
    public static bool IsDefault<T>(T value) where T : notnull
    {
        return EqualityComparer<T>.Default.Equals(value, default!);
    }
}

// Base class สำหรับ constraint
public abstract class Entity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; }
}

// where T : IComparable<T> - ใช้บ่อยมาก
public class SortedCollection<T> where T : IComparable<T>
{
    private List<T> _items = new();

    public void Add(T item)
    {
        _items.Add(item);
        _items.Sort();
    }

    public T Min() => _items.First();
    public T Max() => _items.Last();
    public IEnumerable<T> GetAll() => _items;
}

// Generic Method with Multiple Type Parameters and Constraints
public static TResult Transform<TSource, TResult>(
    TSource source,
    Func<TSource, TResult> converter)
    where TSource : class
    where TResult : class, new()
{
    ArgumentNullException.ThrowIfNull(source);
    return converter(source);
}
```

### 4.1 Constraint ใช้งานจริง

```csharp
// Generic Calculator
public class Calculator<T> where T : INumber<T>
{
    public T Add(T a, T b) => a + b;
    public T Subtract(T a, T b) => a - b;
    public T Multiply(T a, T b) => a * b;
    public T Divide(T a, T b) => a / b;

    public T Sum(IEnumerable<T> values) =>
        values.Aggregate(T.Zero, (acc, v) => acc + v);

    public T Average(IEnumerable<T> values)
    {
        var list = values.ToList();
        if (list.Count == 0) throw new InvalidOperationException("Empty collection");
        return Sum(list) / T.CreateChecked(list.Count);
    }
}

// ใช้งาน (.NET 7+ with INumber<T>)
var intCalc = new Calculator<int>();
Console.WriteLine(intCalc.Add(3, 4));          // 7
Console.WriteLine(intCalc.Sum(new[] { 1, 2, 3, 4, 5 })); // 15

var doubleCalc = new Calculator<double>();
Console.WriteLine(doubleCalc.Average(new[] { 1.5, 2.5, 3.5 })); // 2.5

// Generic Comparer Factory
public static IComparer<T> CreateComparer<T, TKey>(
    Func<T, TKey> keySelector,
    bool descending = false)
    where TKey : IComparable<TKey>
{
    return Comparer<T>.Create((a, b) =>
    {
        int result = keySelector(a).CompareTo(keySelector(b));
        return descending ? -result : result;
    });
}

var people = new List<(string Name, int Age)>
{
    ("Charlie", 30), ("Alice", 25), ("Bob", 35)
};

var byAge = CreateComparer<(string, int), int>(p => p.Age);
var byAgeDesc = CreateComparer<(string, int), int>(p => p.Age, descending: true);

people.Sort(byAge);
people.ForEach(p => Console.WriteLine($"{p.Name}: {p.Age}"));
```

---

## 5. Generic Interface

```csharp
// Generic Interface
public interface IConverter<TFrom, TTo>
{
    TTo Convert(TFrom source);
    TFrom ConvertBack(TTo target);
}

// Generic Interface ด้วย Constraint
public interface IComparable<T> where T : notnull
{
    int CompareTo(T other);
    bool IsLessThan(T other) => CompareTo(other) < 0;
    bool IsGreaterThan(T other) => CompareTo(other) > 0;
    bool IsEqualTo(T other) => CompareTo(other) == 0;
}

// ตัวอย่าง: Temperature Converter
public record Celsius(double Value);
public record Fahrenheit(double Value);
public record Kelvin(double Value);

public class TemperatureConverter :
    IConverter<Celsius, Fahrenheit>,
    IConverter<Celsius, Kelvin>
{
    public Fahrenheit Convert(Celsius source) =>
        new Fahrenheit(source.Value * 9 / 5 + 32);

    public Celsius ConvertBack(Fahrenheit target) =>
        new Celsius((target.Value - 32) * 5 / 9);

    Kelvin IConverter<Celsius, Kelvin>.Convert(Celsius source) =>
        new Kelvin(source.Value + 273.15);

    Celsius IConverter<Celsius, Kelvin>.ConvertBack(Kelvin target) =>
        new Celsius(target.Value - 273.15);
}

// Generic Pipeline Interface
public interface IPipelineStep<TInput, TOutput>
{
    TOutput Process(TInput input);
}

// Chainable Pipeline
public class Pipeline<T>
{
    private readonly List<Func<T, T>> _steps = new();

    public Pipeline<T> AddStep(Func<T, T> step)
    {
        _steps.Add(step);
        return this; // Fluent API
    }

    public T Execute(T input)
    {
        T result = input;
        foreach (var step in _steps)
            result = step(result);
        return result;
    }
}

// ใช้งาน Pipeline
var pipeline = new Pipeline<string>()
    .AddStep(s => s.Trim())
    .AddStep(s => s.ToLower())
    .AddStep(s => s.Replace(" ", "_"))
    .AddStep(s => $"processed_{s}");

string result = pipeline.Execute("  Hello World  ");
Console.WriteLine(result); // processed_hello_world
```

---

## 6. Covariance และ Contravariance

Covariance/Contravariance เกี่ยวกับการแทนที่ Generic Type ด้วย Type ที่เกี่ยวข้อง

```csharp
// ===== Covariance (out) =====
// Interface ที่ใช้ T เฉพาะ output position
// ทำให้ IReadable<Derived> แทน IReadable<Base> ได้

public interface IReadable<out T>  // out = covariant
{
    T Read();
}

public class StringReader : IReadable<string>
{
    public string Read() => "Hello from StringReader";
}

// Covariant: IReadable<string> แทน IReadable<object> ได้
IReadable<object> reader = new StringReader(); // ✅ ใช้ได้
object value = reader.Read();
Console.WriteLine(value); // Hello from StringReader

// Real example: IEnumerable<T> เป็น covariant
IEnumerable<string> strings = new List<string> { "a", "b", "c" };
IEnumerable<object> objects = strings; // ✅ ใช้ได้เพราะ IEnumerable<out T>

// ===== Contravariance (in) =====
// Interface ที่ใช้ T เฉพาะ input position
// ทำให้ IWritable<Base> แทน IWritable<Derived> ได้

public interface IWritable<in T>  // in = contravariant
{
    void Write(T value);
}

public class ObjectWriter : IWritable<object>
{
    public void Write(object value) => Console.WriteLine($"Writing: {value}");
}

// Contravariant: IWritable<object> แทน IWritable<string> ได้
IWritable<string> writer = new ObjectWriter(); // ✅ ใช้ได้
writer.Write("Hello"); // Writing: Hello

// Real example: Action<T> เป็น contravariant
Action<object> printObject = obj => Console.WriteLine(obj);
Action<string> printString = printObject; // ✅ ใช้ได้

// ===== Invariant (ค่าเริ่มต้น) =====
// ถ้าไม่ใส่ in/out, type parameter เป็น invariant
public interface IReadWrite<T>  // invariant
{
    T Read();
    void Write(T value);
}

// IReadWrite<string> ไม่สามารถแทน IReadWrite<object> ได้
// และ IReadWrite<object> ก็ไม่สามารถแทน IReadWrite<string> ได้
```

### 6.1 ตัวอย่างจริง: Covariance ใน Collection

```csharp
// เข้าใจผ่านตัวอย่างจริง
class Animal { public string Name { get; set; } = ""; }
class Dog : Animal { public string Breed { get; set; } = ""; }
class Cat : Animal { }

// IEnumerable<out T> - Covariant
List<Dog> dogs = new() { new Dog { Name = "Rex", Breed = "Lab" } };
IEnumerable<Animal> animals = dogs; // ✅ Dog is-a Animal

// ทำไม List<T> ไม่ covariant
// List<Animal> animalList = dogs; // ❌ Compile Error!
// เพราะถ้าทำได้ จะเพิ่ม Cat ลงใน List ของ Dog ได้ - อันตราย!

// แต่ถ้าต้องการ - ใช้ AsReadOnly()
IReadOnlyList<Dog> readOnlyDogs = dogs.AsReadOnly();
// IReadOnlyList<Animal> readOnlyAnimals = readOnlyDogs; // ยังไม่ได้ (invariant)

// Covariance ผ่าน Func return type
Func<Dog> getDog = () => new Dog { Name = "Buddy" };
Func<Animal> getAnimal = getDog; // ✅ Func<TResult> เป็น covariant

// Contravariance ผ่าน Action parameter
Action<Animal> processAnimal = a => Console.WriteLine($"Processing: {a.Name}");
Action<Dog> processDog = processAnimal; // ✅ Action<T> เป็น contravariant
processDog(new Dog { Name = "Rex" }); // Processing: Rex
```

---

## 7. Generic Utility Classes

```csharp
// Result<T> - Functional Error Handling
public class Result<T>
{
    private readonly T? _value;
    private readonly string? _error;

    private Result(T value)
    {
        _value = value;
        IsSuccess = true;
    }

    private Result(string error)
    {
        _error = error;
        IsSuccess = false;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;

    public T Value => IsSuccess ? _value! :
        throw new InvalidOperationException($"Cannot access value of failed result: {_error}");

    public string Error => IsFailure ? _error! :
        throw new InvalidOperationException("Cannot access error of successful result");

    public static Result<T> Ok(T value) => new(value);
    public static Result<T> Fail(string error) => new(error);

    public Result<TNew> Map<TNew>(Func<T, TNew> mapper)
    {
        if (IsSuccess)
            return Result<TNew>.Ok(mapper(_value!));
        return Result<TNew>.Fail(_error!);
    }

    public Result<T> OnSuccess(Action<T> action)
    {
        if (IsSuccess) action(_value!);
        return this;
    }

    public Result<T> OnFailure(Action<string> action)
    {
        if (IsFailure) action(_error!);
        return this;
    }

    public TResult Match<TResult>(Func<T, TResult> onSuccess, Func<string, TResult> onFailure)
    {
        return IsSuccess ? onSuccess(_value!) : onFailure(_error!);
    }

    public override string ToString() =>
        IsSuccess ? $"Ok({_value})" : $"Fail({_error})";
}

// ใช้งาน Result<T>
Result<int> ParseInt(string input)
{
    if (int.TryParse(input, out int value))
        return Result<int>.Ok(value);
    return Result<int>.Fail($"'{input}' is not a valid integer");
}

Result<double> Divide(int a, int b)
{
    if (b == 0) return Result<double>.Fail("Division by zero");
    return Result<double>.Ok((double)a / b);
}

// Chain operations
var result = ParseInt("42")
    .Map(n => n * 2)
    .OnSuccess(n => Console.WriteLine($"Success: {n}"))
    .OnFailure(e => Console.WriteLine($"Error: {e}"));

var failed = ParseInt("not a number")
    .Map(n => n * 2);
Console.WriteLine(failed); // Fail('not a number' is not a valid integer)

// Match
string message = result.Match(
    onSuccess: n => $"Result is {n}",
    onFailure: e => $"Failed: {e}"
);
Console.WriteLine(message);

// Optional<T> - Maybe Monad
public class Optional<T>
{
    private readonly T? _value;

    private Optional() { HasValue = false; }
    private Optional(T value) { _value = value; HasValue = true; }

    public bool HasValue { get; }
    public bool IsEmpty => !HasValue;

    public T Value => HasValue ? _value! :
        throw new InvalidOperationException("Optional has no value");

    public static Optional<T> Some(T value) => new(value);
    public static Optional<T> None() => new();

    public T GetValueOrDefault(T defaultValue) =>
        HasValue ? _value! : defaultValue;

    public Optional<TNew> Map<TNew>(Func<T, TNew> mapper) =>
        HasValue ? Optional<TNew>.Some(mapper(_value!)) : Optional<TNew>.None();

    public Optional<T> Filter(Func<T, bool> predicate) =>
        HasValue && predicate(_value!) ? this : None();

    public void IfPresent(Action<T> action)
    {
        if (HasValue) action(_value!);
    }

    public override string ToString() =>
        HasValue ? $"Some({_value})" : "None";
}

// Generic Cache
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, (TValue Value, DateTime ExpiresAt)> _cache = new();
    private readonly TimeSpan _ttl;

    public Cache(TimeSpan ttl) { _ttl = ttl; }

    public void Set(TKey key, TValue value)
    {
        _cache[key] = (value, DateTime.UtcNow.Add(_ttl));
    }

    public Optional<TValue> Get(TKey key)
    {
        if (_cache.TryGetValue(key, out var entry))
        {
            if (DateTime.UtcNow <= entry.ExpiresAt)
                return Optional<TValue>.Some(entry.Value);
            _cache.Remove(key); // expired
        }
        return Optional<TValue>.None();
    }

    public TValue GetOrSet(TKey key, Func<TValue> factory)
    {
        var cached = Get(key);
        if (cached.HasValue) return cached.Value;

        TValue value = factory();
        Set(key, value);
        return value;
    }

    public void Invalidate(TKey key) => _cache.Remove(key);
    public void Clear() => _cache.Clear();
    public int Count => _cache.Count;
}
```

---

## 8. โปรแกรมตัวอย่าง: Generic Stack Implementation

```csharp
using System;
using System.Collections;
using System.Collections.Generic;

namespace GenericStackDemo
{
    // Custom Generic Stack
    public class GenericStack<T> : IEnumerable<T>
    {
        private T[] _items;
        private int _count;
        private const int DefaultCapacity = 4;

        public GenericStack() : this(DefaultCapacity) { }

        public GenericStack(int capacity)
        {
            if (capacity < 0)
                throw new ArgumentOutOfRangeException(nameof(capacity));
            _items = new T[capacity];
        }

        public GenericStack(IEnumerable<T> collection)
        {
            _items = collection.ToArray();
            _count = _items.Length;
        }

        public int Count => _count;
        public bool IsEmpty => _count == 0;

        public void Push(T item)
        {
            if (_count == _items.Length)
                Grow();

            _items[_count++] = item;
        }

        public T Pop()
        {
            if (IsEmpty)
                throw new InvalidOperationException("Stack is empty");

            T item = _items[--_count];
            _items[_count] = default!; // Release reference (for GC)
            return item;
        }

        public T Peek()
        {
            if (IsEmpty)
                throw new InvalidOperationException("Stack is empty");
            return _items[_count - 1];
        }

        public bool TryPop(out T item)
        {
            if (IsEmpty)
            {
                item = default!;
                return false;
            }
            item = Pop();
            return true;
        }

        public bool TryPeek(out T item)
        {
            if (IsEmpty)
            {
                item = default!;
                return false;
            }
            item = Peek();
            return true;
        }

        public bool Contains(T item) =>
            Array.IndexOf(_items, item, 0, _count) >= 0;

        public void Clear()
        {
            Array.Clear(_items, 0, _count);
            _count = 0;
        }

        public T[] ToArray()
        {
            T[] result = new T[_count];
            Array.Copy(_items, result, _count);
            Array.Reverse(result); // Stack order (top first)
            return result;
        }

        private void Grow()
        {
            int newCapacity = _items.Length == 0 ? DefaultCapacity : _items.Length * 2;
            T[] newItems = new T[newCapacity];
            Array.Copy(_items, newItems, _count);
            _items = newItems;
        }

        // Implement IEnumerable<T>
        public IEnumerator<T> GetEnumerator()
        {
            for (int i = _count - 1; i >= 0; i--)
                yield return _items[i];
        }

        IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

        public override string ToString() =>
            $"Stack<{typeof(T).Name}>[{string.Join(", ", this)}]";
    }

    // Generic Stack-based Calculator
    public class ExpressionCalculator
    {
        private readonly GenericStack<double> _operands = new();
        private readonly GenericStack<char> _operators = new();

        private int Precedence(char op) => op switch
        {
            '+' or '-' => 1,
            '*' or '/' => 2,
            '^' => 3,
            _ => 0
        };

        private double ApplyOp(char op, double b, double a) => op switch
        {
            '+' => a + b,
            '-' => a - b,
            '*' => a * b,
            '/' => b == 0 ? throw new DivideByZeroException() : a / b,
            _ => throw new InvalidOperationException($"Unknown operator: {op}")
        };

        public double Evaluate(string expression)
        {
            _operands.Clear();
            _operators.Clear();

            int i = 0;
            while (i < expression.Length)
            {
                char c = expression[i];

                if (char.IsWhiteSpace(c)) { i++; continue; }

                if (char.IsDigit(c) || (c == '-' && i == 0))
                {
                    int start = i;
                    if (c == '-') i++;
                    while (i < expression.Length && (char.IsDigit(expression[i]) || expression[i] == '.'))
                        i++;
                    _operands.Push(double.Parse(expression[start..i]));
                    continue;
                }

                if (c == '(')
                {
                    _operators.Push(c);
                }
                else if (c == ')')
                {
                    while (_operators.Peek() != '(')
                    {
                        _operands.Push(ApplyOp(_operators.Pop(), _operands.Pop(), _operands.Pop()));
                    }
                    _operators.Pop(); // Remove '('
                }
                else if ("+-*/".Contains(c))
                {
                    while (!_operators.IsEmpty && _operators.Peek() != '(' &&
                           Precedence(_operators.Peek()) >= Precedence(c))
                    {
                        _operands.Push(ApplyOp(_operators.Pop(), _operands.Pop(), _operands.Pop()));
                    }
                    _operators.Push(c);
                }

                i++;
            }

            while (!_operators.IsEmpty)
                _operands.Push(ApplyOp(_operators.Pop(), _operands.Pop(), _operands.Pop()));

            return _operands.Pop();
        }
    }

    class Program
    {
        static void Main()
        {
            Console.WriteLine("=== Generic Stack Test ===\n");

            // Basic operations
            var stack = new GenericStack<string>();
            Console.WriteLine($"IsEmpty: {stack.IsEmpty}");

            stack.Push("First");
            stack.Push("Second");
            stack.Push("Third");

            Console.WriteLine($"Count: {stack.Count}");
            Console.WriteLine($"Peek: {stack.Peek()}");
            Console.WriteLine($"Stack: {stack}");

            // TryPop
            while (stack.TryPop(out string? item))
                Console.WriteLine($"Popped: {item}");

            // With numbers
            Console.WriteLine("\n=== Numeric Stack ===");
            var numStack = new GenericStack<int>(new[] { 1, 2, 3, 4, 5 });
            Console.WriteLine(numStack);

            // Contains
            Console.WriteLine($"Contains 3: {numStack.Contains(3)}");
            Console.WriteLine($"Contains 9: {numStack.Contains(9)}");

            // ToArray
            int[] arr = numStack.ToArray();
            Console.WriteLine("ToArray: " + string.Join(", ", arr));

            // Foreach (IEnumerable)
            Console.Write("Foreach: ");
            foreach (int n in numStack)
                Console.Write($"{n} ");
            Console.WriteLine();

            // Expression Calculator
            Console.WriteLine("\n=== Expression Calculator ===");
            var calc = new ExpressionCalculator();

            string[] expressions = {
                "3 + 4",
                "10 - 3 * 2",
                "(10 - 3) * 2",
                "2 + 3 * 4 - 1",
                "100 / 4 + 5 * 3"
            };

            foreach (string expr in expressions)
            {
                double result = calc.Evaluate(expr);
                Console.WriteLine($"{expr} = {result}");
            }

            // Generic Stack with custom objects
            Console.WriteLine("\n=== Stack with Custom Objects ===");
            var personStack = new GenericStack<(string Name, int Age)>();
            personStack.Push(("Alice", 25));
            personStack.Push(("Bob", 30));
            personStack.Push(("Charlie", 35));

            Console.WriteLine(personStack);

            while (personStack.TryPop(out var person))
                Console.WriteLine($"  {person.Name} (age {person.Age})");
        }
    }
}
```

---

## Exercises

### Exercise 1: Generic Pair Collection
สร้าง `BiMap<T1, T2>` ที่เป็น bidirectional dictionary:
- ค้นหาจาก T1 ไป T2 ได้
- ค้นหาจาก T2 ไป T1 ได้
- ต้องมี constraint ที่เหมาะสม

```csharp
public class BiMap<T1, T2> where T1 : notnull where T2 : notnull
{
    // TODO: implement Forward and Reverse lookup
    public void Add(T1 forward, T2 backward) { }
    public T2 GetForward(T1 key) => default!;
    public T1 GetBackward(T2 key) => default!;
}
```

### Exercise 2: Generic Event Bus
สร้าง Event Bus ที่:
- Subscribe/Unsubscribe handler สำหรับ event type
- Publish event ได้
- ใช้ Generics สำหรับ event types

```csharp
public interface IEvent { }
public record UserCreated(string Name, string Email) : IEvent;
public record OrderPlaced(int OrderId, decimal Amount) : IEvent;

public class EventBus
{
    // TODO: Subscribe<T>(Action<T> handler) where T : IEvent
    // TODO: Unsubscribe<T>(Action<T> handler) where T : IEvent
    // TODO: Publish<T>(T @event) where T : IEvent
}
```

### Exercise 3: Generic Tree
สร้าง Generic Tree:
- `TreeNode<T>` มี Value, Children
- Add/Remove children
- DFS/BFS traversal
- Find node ด้วย predicate

---

## สรุป

- ✅ Generic ช่วยเขียน code ที่ใช้ได้กับหลาย Type โดยคง Type Safety
- ✅ `class Box<T>` สร้าง Generic Class ที่ทำงานกับ T ใดก็ได้
- ✅ Generic Methods `static T Max<T>(T a, T b)` ทำงานกับ type ที่กำหนดตอนเรียก
- ✅ Constraints กำหนดข้อจำกัดของ T: `where T : class`, `new()`, `IComparable<T>` เป็นต้น
- ✅ Generic Interface ทำให้ code ยืดหยุ่นและ reusable
- ✅ Covariance (`out T`) ให้แทนที่ด้วย derived type (output position)
- ✅ Contravariance (`in T`) ให้แทนที่ด้วย base type (input position)
- ✅ `Result<T>` และ `Optional<T>` เป็น Generic patterns ที่ดีสำหรับ error handling
- ✅ Generics ทำงานได้ดีกับ performance - ไม่มี Boxing/Unboxing

## Part ถัดไป
**Part 024** จะพูดถึง LINQ พื้นฐาน: Query Syntax vs Method Syntax, Where, Select, OrderBy, GroupBy และการทำ Aggregation

---
*Part 023/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
