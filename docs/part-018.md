# Part 018: Abstract Classes

## เนื้อหาใน Part นี้
- abstract keyword และการใช้งาน
- Abstract methods และ properties
- Template Method pattern
- Abstract class vs Interface เปรียบเทียบเชิงลึก
- โปรแกรมตัวอย่าง: Report Generator

---

## 1. Abstract Class คืออะไร?

Abstract class คือ class ที่:
- **ไม่สามารถ instantiate** (สร้าง object โดยตรง) ได้
- สามารถมี **abstract members** ที่ derived classes ต้องไป implement
- สามารถมี **concrete members** (implementation จริงๆ) ได้

```csharp
// Abstract class
public abstract class Animal
{
    // Concrete properties
    public string Name { get; set; }
    public int Age { get; set; }

    // Concrete constructor
    protected Animal(string name, int age)
    {
        Name = name;
        Age = age;
    }

    // Abstract method - ไม่มี body
    public abstract void MakeSound();
    public abstract string GetAnimalType();

    // Abstract property
    public abstract double Weight { get; }

    // Concrete method
    public void Sleep()
    {
        Console.WriteLine($"{Name} กำลังนอนหลับ...");
    }

    // Virtual method - มี default แต่ override ได้
    public virtual void Eat(string food)
    {
        Console.WriteLine($"{Name} กินอาหาร: {food}");
    }

    // Non-virtual - ไม่สามารถ override ได้
    public void Breathe()
    {
        Console.WriteLine($"{Name} หายใจ");
    }
}

// var animal = new Animal("test", 1); // Error! Cannot instantiate abstract class

// Derived class ต้อง implement abstract members ทั้งหมด
public class Lion : Animal
{
    private double _weight;

    public Lion(string name, int age, double weight)
        : base(name, age)
    {
        _weight = weight;
    }

    // ต้อง implement abstract members
    public override void MakeSound() => Console.WriteLine($"{Name}: ก้อง!!! 🦁");
    public override string GetAnimalType() => "สัตว์เลี้ยงลูกด้วยนม";
    public override double Weight => _weight;

    // Override virtual (optional)
    public override void Eat(string food)
    {
        base.Eat(food);
        Console.WriteLine($"{Name} กัดเนื้อด้วยฟันคม 🦷");
    }
}

var lion = new Lion("Simba", 5, 180);
lion.MakeSound();
lion.Eat("wildebeest");
lion.Sleep();
lion.Breathe();
Console.WriteLine($"ประเภท: {lion.GetAnimalType()}, น้ำหนัก: {lion.Weight}kg");
```

---

## 2. Abstract Properties และ Abstract Indexers

```csharp
public abstract class DataStore
{
    // Abstract properties
    public abstract string StoreName { get; }
    public abstract int ItemCount { get; }
    public abstract bool IsReadOnly { get; }

    // Abstract indexer
    public abstract string this[string key] { get; set; }

    // Abstract methods
    public abstract bool ContainsKey(string key);
    public abstract bool Remove(string key);
    public abstract void Clear();

    // Concrete method ที่ใช้ abstract members
    public virtual string GetOrDefault(string key, string defaultValue = "")
    {
        return ContainsKey(key) ? this[key] : defaultValue;
    }

    public virtual void Print()
    {
        Console.WriteLine($"=== {StoreName} ({ItemCount} items) ===");
    }
}

// Concrete implementation ด้วย Dictionary
public class DictionaryStore : DataStore
{
    private readonly Dictionary<string, string> _data = new();
    private readonly string _name;

    public DictionaryStore(string name) { _name = name; }

    public override string StoreName => _name;
    public override int ItemCount => _data.Count;
    public override bool IsReadOnly => false;

    public override string this[string key]
    {
        get => _data.TryGetValue(key, out var val) ? val : "";
        set => _data[key] = value;
    }

    public override bool ContainsKey(string key) => _data.ContainsKey(key);
    public override bool Remove(string key) => _data.Remove(key);
    public override void Clear() => _data.Clear();
}

// Concrete implementation แบบ read-only
public class ConfigStore : DataStore
{
    private readonly IReadOnlyDictionary<string, string> _config;

    public ConfigStore(Dictionary<string, string> config)
    {
        _config = config;
    }

    public override string StoreName => "Config";
    public override int ItemCount => _config.Count;
    public override bool IsReadOnly => true;

    public override string this[string key]
    {
        get => _config.TryGetValue(key, out var val) ? val : "";
        set => throw new InvalidOperationException("Config store is read-only");
    }

    public override bool ContainsKey(string key) => _config.ContainsKey(key);
    public override bool Remove(string key) => throw new InvalidOperationException("Read-only");
    public override void Clear() => throw new InvalidOperationException("Read-only");
}

// การใช้งาน
DataStore store1 = new DictionaryStore("UserPrefs");
store1["theme"] = "dark";
store1["language"] = "th";
store1["fontSize"] = "14";
store1.Print();

Console.WriteLine(store1.GetOrDefault("theme"));
Console.WriteLine(store1.GetOrDefault("missing", "default value"));

DataStore config = new ConfigStore(new Dictionary<string, string>
{
    ["server"] = "localhost",
    ["port"] = "8080"
});
config.Print();
```

---

## 3. Template Method Pattern

Template Method Pattern ใช้ abstract class กำหนด "โครงสร้าง" ของ algorithm โดยให้ subclasses กำหนดรายละเอียด

```csharp
/// <summary>
/// Template Method Pattern: Game
/// </summary>
public abstract class Game
{
    // Template method - กำหนดโครงสร้างการเล่น
    public void Play()
    {
        Console.WriteLine($"\n=== เริ่มเกม: {GameName} ===");
        Initialize();
        Console.WriteLine("เกมเริ่ม!");

        while (!IsGameOver())
        {
            PrintCurrentState();
            TakeTurn();
        }

        PrintResult();
        Cleanup();
        Console.WriteLine("=== จบเกม ===");
    }

    // Abstract methods ที่ subclasses ต้อง implement
    protected abstract string GameName { get; }
    protected abstract void Initialize();
    protected abstract bool IsGameOver();
    protected abstract void TakeTurn();
    protected abstract void PrintResult();

    // Hook methods - optional override
    protected virtual void PrintCurrentState() { }
    protected virtual void Cleanup() { }
}

/// <summary>
/// เกมทายเลข
/// </summary>
public class NumberGuessingGame : Game
{
    private int _target;
    private int _guess;
    private int _attempts;
    private const int MaxAttempts = 7;

    protected override string GameName => "ทายเลข 1-100";

    protected override void Initialize()
    {
        _target = Random.Shared.Next(1, 101);
        _attempts = 0;
        Console.WriteLine($"คิดเลขแล้ว! ทายเลข 1-100 ({MaxAttempts} ครั้ง)");
    }

    protected override bool IsGameOver()
        => _guess == _target || _attempts >= MaxAttempts;

    protected override void TakeTurn()
    {
        _attempts++;
        Console.Write($"ครั้งที่ {_attempts}: ");
        // จำลอง input
        _guess = Random.Shared.Next(1, 101);
        Console.Write($"ทาย {_guess} ");

        if (_guess < _target) Console.WriteLine("→ สูงกว่านี้");
        else if (_guess > _target) Console.WriteLine("→ ต่ำกว่านี้");
        else Console.WriteLine("→ ถูกต้อง! 🎉");
    }

    protected override void PrintResult()
    {
        if (_guess == _target)
            Console.WriteLine($"ยินดีด้วย! ทายถูกใน {_attempts} ครั้ง");
        else
            Console.WriteLine($"หมดโอกาส! เลขที่คิดคือ {_target}");
    }
}

/// <summary>
/// เกม FizzBuzz
/// </summary>
public class FizzBuzzGame : Game
{
    private int _currentNumber;
    private int _maxNumber;
    private int _score;

    protected override string GameName => "FizzBuzz";

    protected override void Initialize()
    {
        _currentNumber = 1;
        _maxNumber = 20;
        _score = 0;
        Console.WriteLine($"นับ 1 ถึง {_maxNumber}");
    }

    protected override bool IsGameOver() => _currentNumber > _maxNumber;

    protected override void TakeTurn()
    {
        string result = (_currentNumber % 15 == 0) ? "FizzBuzz" :
                       (_currentNumber % 3 == 0) ? "Fizz" :
                       (_currentNumber % 5 == 0) ? "Buzz" :
                       _currentNumber.ToString();
        Console.Write($"{result} ");
        if (_currentNumber % 3 == 0 || _currentNumber % 5 == 0) _score++;
        _currentNumber++;
    }

    protected override void PrintResult()
    {
        Console.WriteLine($"\nFizz/Buzz เกิดขึ้น {_score} ครั้ง");
    }
}

// การใช้งาน
var guessing = new NumberGuessingGame();
guessing.Play();

var fizzBuzz = new FizzBuzzGame();
fizzBuzz.Play();
```

---

## 4. Abstract Class สำหรับ Data Access Layer

```csharp
// Generic Repository Pattern ด้วย Abstract Class
public abstract class Repository<TEntity, TKey>
    where TEntity : class
{
    protected readonly List<TEntity> _entities = new();
    private static int _queryCount = 0;

    // Abstract methods
    protected abstract TKey GetEntityKey(TEntity entity);
    protected abstract bool KeyEquals(TKey key1, TKey key2);

    // Template methods - กำหนด workflow
    public TEntity? GetById(TKey id)
    {
        _queryCount++;
        BeforeQuery("GetById");
        var result = _entities.FirstOrDefault(e => KeyEquals(GetEntityKey(e), id));
        AfterQuery("GetById", result != null);
        return result;
    }

    public IEnumerable<TEntity> GetAll()
    {
        _queryCount++;
        BeforeQuery("GetAll");
        var result = _entities.ToList();
        AfterQuery("GetAll", true);
        return result;
    }

    public void Add(TEntity entity)
    {
        ValidateEntity(entity);
        BeforeAdd(entity);
        _entities.Add(entity);
        AfterAdd(entity);
    }

    public bool Update(TKey id, TEntity updatedEntity)
    {
        ValidateEntity(updatedEntity);
        int index = _entities.FindIndex(e => KeyEquals(GetEntityKey(e), id));
        if (index < 0) return false;

        var oldEntity = _entities[index];
        BeforeUpdate(oldEntity, updatedEntity);
        _entities[index] = updatedEntity;
        AfterUpdate(oldEntity, updatedEntity);
        return true;
    }

    public bool Delete(TKey id)
    {
        var entity = GetById(id);
        if (entity == null) return false;

        BeforeDelete(entity);
        _entities.Remove(entity);
        AfterDelete(entity);
        return true;
    }

    public int Count => _entities.Count;
    public static int TotalQueries => _queryCount;

    // Abstract validation
    protected abstract void ValidateEntity(TEntity entity);

    // Hook methods (optional)
    protected virtual void BeforeQuery(string operation) { }
    protected virtual void AfterQuery(string operation, bool found) { }
    protected virtual void BeforeAdd(TEntity entity) { }
    protected virtual void AfterAdd(TEntity entity)
        => Console.WriteLine($"Added {typeof(TEntity).Name}");
    protected virtual void BeforeUpdate(TEntity old, TEntity updated) { }
    protected virtual void AfterUpdate(TEntity old, TEntity updated)
        => Console.WriteLine($"Updated {typeof(TEntity).Name}");
    protected virtual void BeforeDelete(TEntity entity) { }
    protected virtual void AfterDelete(TEntity entity)
        => Console.WriteLine($"Deleted {typeof(TEntity).Name}");
}

// Concrete implementation
public record Product(int Id, string Name, decimal Price, int Stock);

public class ProductRepository : Repository<Product, int>
{
    protected override int GetEntityKey(Product entity) => entity.Id;

    protected override bool KeyEquals(int key1, int key2) => key1 == key2;

    protected override void ValidateEntity(Product entity)
    {
        if (string.IsNullOrWhiteSpace(entity.Name))
            throw new ArgumentException("Product name is required");
        if (entity.Price < 0)
            throw new ArgumentException("Price cannot be negative");
    }

    protected override void BeforeAdd(Product entity)
    {
        if (_entities.Any(p => p.Id == entity.Id))
            throw new InvalidOperationException($"Product ID {entity.Id} already exists");
    }

    // เพิ่ม methods เฉพาะ Product
    public IEnumerable<Product> GetByPriceRange(decimal min, decimal max)
        => _entities.Where(p => p.Price >= min && p.Price <= max);

    public IEnumerable<Product> GetLowStock(int threshold = 5)
        => _entities.Where(p => p.Stock <= threshold);
}

// การใช้งาน
var repo = new ProductRepository();
repo.Add(new Product(1, "Laptop", 35000, 10));
repo.Add(new Product(2, "Mouse", 500, 50));
repo.Add(new Product(3, "Keyboard", 800, 30));
repo.Add(new Product(4, "Monitor", 8000, 3));

var product = repo.GetById(1);
Console.WriteLine($"Found: {product?.Name}");

var cheapProducts = repo.GetByPriceRange(0, 1000);
Console.WriteLine("\nสินค้าราคาต่ำกว่า 1000:");
foreach (var p in cheapProducts)
    Console.WriteLine($"  {p.Name}: {p.Price:N0}฿");
```

---

## 5. Abstract Class vs Interface เปรียบเทียบ

```csharp
// ============ เมื่อไหรใช้ Abstract Class ============

// 1. มี state ที่ต้องการ share
public abstract class Logger
{
    protected string _prefix = "[LOG]"; // state
    private readonly List<string> _history = new(); // state

    protected void WriteToHistory(string message)
    {
        _history.Add(message);
    }

    public IReadOnlyList<string> History => _history;

    public abstract void Write(string message);

    public virtual void WriteError(string message)
    {
        Write($"[ERROR] {message}");
    }
}

// 2. Constructor logic ที่ต้อง share
public abstract class Connection
{
    protected readonly string _connectionString;
    protected bool _isConnected;

    protected Connection(string connectionString) // constructor!
    {
        _connectionString = connectionString ?? throw new ArgumentNullException();
    }

    public abstract void Connect();
    public abstract void Disconnect();
    public abstract void ExecuteCommand(string command);

    // Template method ใช้ abstract methods
    public void ExecuteSafely(string command)
    {
        Connect();
        try { ExecuteCommand(command); }
        finally { Disconnect(); }
    }
}

// 3. Code reuse ที่ meaningful
public abstract class SortAlgorithm<T> where T : IComparable<T>
{
    private int _comparisons = 0;
    private int _swaps = 0;

    public int Comparisons => _comparisons;
    public int Swaps => _swaps;

    public T[] Sort(T[] data)
    {
        var copy = (T[])data.Clone();
        _comparisons = 0;
        _swaps = 0;
        DoSort(copy);
        return copy;
    }

    protected abstract void DoSort(T[] data);

    protected int Compare(T a, T b)
    {
        _comparisons++;
        return a.CompareTo(b);
    }

    protected void Swap(T[] data, int i, int j)
    {
        _swaps++;
        (data[i], data[j]) = (data[j], data[i]);
    }

    public string GetStats()
        => $"Comparisons: {_comparisons}, Swaps: {_swaps}";
}

// Concrete sort implementations
public class BubbleSort<T> : SortAlgorithm<T> where T : IComparable<T>
{
    protected override void DoSort(T[] data)
    {
        int n = data.Length;
        for (int i = 0; i < n - 1; i++)
            for (int j = 0; j < n - i - 1; j++)
                if (Compare(data[j], data[j + 1]) > 0)
                    Swap(data, j, j + 1);
    }
}

public class SelectionSort<T> : SortAlgorithm<T> where T : IComparable<T>
{
    protected override void DoSort(T[] data)
    {
        int n = data.Length;
        for (int i = 0; i < n - 1; i++)
        {
            int minIdx = i;
            for (int j = i + 1; j < n; j++)
                if (Compare(data[j], data[minIdx]) < 0)
                    minIdx = j;
            if (minIdx != i)
                Swap(data, i, minIdx);
        }
    }
}

// ============ เมื่อไหรใช้ Interface ============

// 1. Multiple behaviors
public interface ISerializable2
{
    string Serialize();
    void Deserialize(string data);
}

public interface ICacheable
{
    string CacheKey { get; }
    TimeSpan CacheDuration { get; }
}

public interface IValidatable
{
    IEnumerable<string> Validate();
    bool IsValid => !Validate().Any();
}

// Class implement ได้หลาย interfaces
public class UserProfile : ISerializable2, ICacheable, IValidatable
{
    public int Id { get; set; }
    public string Username { get; set; } = "";
    public string Email { get; set; } = "";

    public string Serialize() => System.Text.Json.JsonSerializer.Serialize(this);
    public void Deserialize(string data)
    {
        var obj = System.Text.Json.JsonSerializer.Deserialize<UserProfile>(data);
        if (obj != null) { Id = obj.Id; Username = obj.Username; Email = obj.Email; }
    }

    public string CacheKey => $"user:{Id}";
    public TimeSpan CacheDuration => TimeSpan.FromMinutes(30);

    public IEnumerable<string> Validate()
    {
        if (string.IsNullOrWhiteSpace(Username)) yield return "Username is required";
        if (string.IsNullOrWhiteSpace(Email)) yield return "Email is required";
        if (!Email.Contains('@')) yield return "Invalid email format";
    }
}

// การใช้งาน sort algorithms
int[] data = [64, 34, 25, 12, 22, 11, 90];

var bubble = new BubbleSort<int>();
int[] sortedBubble = bubble.Sort(data);
Console.WriteLine($"Bubble: {string.Join(", ", sortedBubble)} | {bubble.GetStats()}");

var selection = new SelectionSort<int>();
int[] sortedSel = selection.Sort(data);
Console.WriteLine($"Selection: {string.Join(", ", sortedSel)} | {selection.GetStats()}");
```

---

## 6. โปรแกรมตัวอย่าง: Report Generator

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace ReportGeneratorExample
{
    /// <summary>
    /// ข้อมูลสำหรับรายงาน
    /// </summary>
    public class ReportData
    {
        public string Title { get; set; } = "";
        public DateTime GeneratedAt { get; set; } = DateTime.Now;
        public string Author { get; set; } = "";
        public List<string[]> Rows { get; set; } = new();
        public string[] Headers { get; set; } = [];
        public Dictionary<string, object> Metadata { get; set; } = new();
    }

    /// <summary>
    /// Abstract base สำหรับ Report Generator
    /// Template Method Pattern
    /// </summary>
    public abstract class ReportGenerator
    {
        protected ReportData Data { get; private set; } = new();
        private bool _isGenerated = false;

        // Template Method - กำหนด workflow
        public string GenerateReport(ReportData data)
        {
            Data = data ?? throw new ArgumentNullException(nameof(data));
            _isGenerated = false;

            Validate();
            PrepareData();

            var sb = new StringBuilder();
            RenderHeader(sb);
            RenderBody(sb);
            RenderFooter(sb);

            _isGenerated = true;
            return PostProcess(sb.ToString());
        }

        // Abstract methods ที่ subclass ต้อง implement
        protected abstract void RenderHeader(StringBuilder sb);
        protected abstract void RenderBody(StringBuilder sb);
        protected abstract void RenderFooter(StringBuilder sb);
        public abstract string FormatName { get; }

        // Hook methods - optional override
        protected virtual void Validate()
        {
            if (string.IsNullOrWhiteSpace(Data.Title))
                throw new ArgumentException("Report title is required");
        }

        protected virtual void PrepareData()
        {
            // Default: ไม่มีการเตรียมข้อมูลพิเศษ
        }

        protected virtual string PostProcess(string content) => content;

        // Shared helper methods
        protected string GetCurrentDatetime()
            => Data.GeneratedAt.ToString("dd/MM/yyyy HH:mm:ss");

        protected string GetColumnValue(string[] row, int index, string defaultValue = "")
            => index < row.Length ? row[index] : defaultValue;
    }

    /// <summary>
    /// Plain Text Report
    /// </summary>
    public class TextReportGenerator : ReportGenerator
    {
        public override string FormatName => "Plain Text";

        protected override void RenderHeader(StringBuilder sb)
        {
            string separator = new string('=', 60);
            sb.AppendLine(separator);
            sb.AppendLine($"  {Data.Title.ToUpper()}");
            sb.AppendLine($"  สร้างโดย: {Data.Author}  วันที่: {GetCurrentDatetime()}");
            sb.AppendLine(separator);
            sb.AppendLine();
        }

        protected override void RenderBody(StringBuilder sb)
        {
            if (Data.Headers.Length > 0)
            {
                // คำนวณความกว้างแต่ละคอลัมน์
                int[] colWidths = CalculateColumnWidths();

                // แสดง headers
                sb.Append("| ");
                for (int i = 0; i < Data.Headers.Length; i++)
                {
                    sb.Append(Data.Headers[i].PadRight(colWidths[i]));
                    sb.Append(" | ");
                }
                sb.AppendLine();

                // เส้นคั่น
                sb.Append("|");
                foreach (int w in colWidths)
                    sb.Append(new string('-', w + 2) + "|");
                sb.AppendLine();

                // แสดงข้อมูล
                foreach (var row in Data.Rows)
                {
                    sb.Append("| ");
                    for (int i = 0; i < Data.Headers.Length; i++)
                    {
                        string val = GetColumnValue(row, i);
                        sb.Append(val.PadRight(colWidths[i]));
                        sb.Append(" | ");
                    }
                    sb.AppendLine();
                }
            }

            sb.AppendLine();
            sb.AppendLine($"รวมทั้งหมด: {Data.Rows.Count} รายการ");
        }

        protected override void RenderFooter(StringBuilder sb)
        {
            sb.AppendLine(new string('-', 60));
            sb.AppendLine("--- สิ้นสุดรายงาน ---");
        }

        private int[] CalculateColumnWidths()
        {
            int[] widths = Data.Headers.Select(h => h.Length).ToArray();
            foreach (var row in Data.Rows)
            {
                for (int i = 0; i < widths.Length && i < row.Length; i++)
                {
                    widths[i] = Math.Max(widths[i], row[i].Length);
                }
            }
            return widths;
        }
    }

    /// <summary>
    /// HTML Report
    /// </summary>
    public class HtmlReportGenerator : ReportGenerator
    {
        private readonly string _cssStyle;
        public override string FormatName => "HTML";

        public HtmlReportGenerator(string? cssStyle = null)
        {
            _cssStyle = cssStyle ?? DefaultStyle;
        }

        private const string DefaultStyle = """
            body { font-family: Arial, sans-serif; margin: 20px; }
            h1 { color: #333; border-bottom: 2px solid #007bff; }
            .meta { color: #666; font-size: 0.9em; margin-bottom: 20px; }
            table { border-collapse: collapse; width: 100%; }
            th { background-color: #007bff; color: white; padding: 8px 12px; text-align: left; }
            td { border: 1px solid #ddd; padding: 8px 12px; }
            tr:nth-child(even) { background-color: #f5f5f5; }
            tr:hover { background-color: #e8f4fd; }
            .footer { margin-top: 20px; color: #666; font-size: 0.8em; }
            """;

        protected override void RenderHeader(StringBuilder sb)
        {
            sb.AppendLine("<!DOCTYPE html>");
            sb.AppendLine("<html lang=\"th\">");
            sb.AppendLine("<head>");
            sb.AppendLine($"  <meta charset=\"UTF-8\">");
            sb.AppendLine($"  <title>{Data.Title}</title>");
            sb.AppendLine($"  <style>{_cssStyle}</style>");
            sb.AppendLine("</head>");
            sb.AppendLine("<body>");
            sb.AppendLine($"  <h1>{Data.Title}</h1>");
            sb.AppendLine($"  <div class=\"meta\">");
            sb.AppendLine($"    สร้างโดย: <strong>{Data.Author}</strong> | ");
            sb.AppendLine($"    วันที่: <strong>{GetCurrentDatetime()}</strong>");
            sb.AppendLine($"  </div>");
        }

        protected override void RenderBody(StringBuilder sb)
        {
            if (Data.Headers.Length == 0 && Data.Rows.Count == 0) return;

            sb.AppendLine("  <table>");

            if (Data.Headers.Length > 0)
            {
                sb.AppendLine("    <thead><tr>");
                foreach (string header in Data.Headers)
                    sb.AppendLine($"      <th>{EscapeHtml(header)}</th>");
                sb.AppendLine("    </tr></thead>");
            }

            sb.AppendLine("    <tbody>");
            foreach (var row in Data.Rows)
            {
                sb.AppendLine("      <tr>");
                for (int i = 0; i < Data.Headers.Length; i++)
                {
                    string val = EscapeHtml(GetColumnValue(row, i));
                    sb.AppendLine($"        <td>{val}</td>");
                }
                sb.AppendLine("      </tr>");
            }
            sb.AppendLine("    </tbody>");
            sb.AppendLine("  </table>");
            sb.AppendLine($"  <p>รวมทั้งหมด: <strong>{Data.Rows.Count}</strong> รายการ</p>");
        }

        protected override void RenderFooter(StringBuilder sb)
        {
            sb.AppendLine($"  <div class=\"footer\">");
            sb.AppendLine($"    รายงานนี้สร้างโดยระบบอัตโนมัติ | {FormatName} Report");
            sb.AppendLine($"  </div>");
            sb.AppendLine("</body></html>");
        }

        private static string EscapeHtml(string text)
            => text.Replace("&", "&amp;").Replace("<", "&lt;").Replace(">", "&gt;")
                   .Replace("\"", "&quot;").Replace("'", "&#39;");
    }

    /// <summary>
    /// CSV Report
    /// </summary>
    public class CsvReportGenerator : ReportGenerator
    {
        private readonly char _delimiter;
        public override string FormatName => "CSV";

        public CsvReportGenerator(char delimiter = ',')
        {
            _delimiter = delimiter;
        }

        protected override void RenderHeader(StringBuilder sb)
        {
            // CSV ไม่มี header section ปกติ
            // เพิ่ม comment ถ้าต้องการ
            sb.AppendLine($"# {Data.Title}");
            sb.AppendLine($"# สร้างเมื่อ: {GetCurrentDatetime()}");
        }

        protected override void RenderBody(StringBuilder sb)
        {
            if (Data.Headers.Length > 0)
            {
                sb.AppendLine(FormatCsvRow(Data.Headers));
            }

            foreach (var row in Data.Rows)
            {
                sb.AppendLine(FormatCsvRow(row));
            }
        }

        protected override void RenderFooter(StringBuilder sb)
        {
            // CSV ไม่มี footer
        }

        private string FormatCsvRow(string[] values)
        {
            return string.Join(_delimiter.ToString(), values.Select(v =>
            {
                // Escape ถ้ามี delimiter, newline, หรือ quotes
                if (v.Contains(_delimiter) || v.Contains('\n') || v.Contains('"'))
                    return $"\"{v.Replace("\"", "\"\"")}\"";
                return v;
            }));
        }
    }

    /// <summary>
    /// Markdown Report
    /// </summary>
    public class MarkdownReportGenerator : ReportGenerator
    {
        public override string FormatName => "Markdown";

        protected override void RenderHeader(StringBuilder sb)
        {
            sb.AppendLine($"# {Data.Title}");
            sb.AppendLine();
            sb.AppendLine($"> **สร้างโดย:** {Data.Author}  ");
            sb.AppendLine($"> **วันที่:** {GetCurrentDatetime()}");
            sb.AppendLine();
        }

        protected override void RenderBody(StringBuilder sb)
        {
            if (Data.Headers.Length > 0)
            {
                sb.AppendLine("| " + string.Join(" | ", Data.Headers) + " |");
                sb.AppendLine("|" + string.Join("|", Data.Headers.Select(_ => "---")) + "|");

                foreach (var row in Data.Rows)
                {
                    var cells = Data.Headers.Select((_, i) => GetColumnValue(row, i));
                    sb.AppendLine("| " + string.Join(" | ", cells) + " |");
                }

                sb.AppendLine();
                sb.AppendLine($"**รวม {Data.Rows.Count} รายการ**");
            }
        }

        protected override void RenderFooter(StringBuilder sb)
        {
            sb.AppendLine();
            sb.AppendLine("---");
            sb.AppendLine($"*รายงานนี้สร้างโดยระบบอัตโนมัติ ({FormatName})*");
        }
    }

    /// <summary>
    /// Report Factory
    /// </summary>
    public static class ReportFactory
    {
        public static ReportGenerator Create(string format)
        {
            return format.ToLower() switch
            {
                "text" or "txt" => new TextReportGenerator(),
                "html" => new HtmlReportGenerator(),
                "csv" => new CsvReportGenerator(),
                "markdown" or "md" => new MarkdownReportGenerator(),
                _ => throw new ArgumentException($"ไม่รองรับ format: {format}")
            };
        }

        public static IEnumerable<string> SupportedFormats
            => new[] { "text", "html", "csv", "markdown" };
    }

    class Program
    {
        static void Main(string[] args)
        {
            // สร้างข้อมูลสำหรับรายงาน
            var data = new ReportData
            {
                Title = "รายงานยอดขายประจำเดือน มกราคม 2568",
                Author = "ระบบ ERP",
                Headers = ["สินค้า", "จำนวน", "ราคา/ชิ้น", "รวม"],
                Rows = new List<string[]>
                {
                    ["Laptop Pro 15\"", "5", "35,000.00", "175,000.00"],
                    ["Mouse Wireless", "20", "500.00", "10,000.00"],
                    ["Keyboard RGB", "15", "800.00", "12,000.00"],
                    ["Monitor 27\"", "8", "8,000.00", "64,000.00"],
                    ["USB Hub", "30", "250.00", "7,500.00"],
                },
                Metadata = new Dictionary<string, object>
                {
                    ["department"] = "IT Sales",
                    ["quarter"] = "Q1",
                    ["currency"] = "THB"
                }
            };

            // สร้างรายงานในหลายรูปแบบ
            Console.WriteLine("สร้างรายงานในรูปแบบต่างๆ:");
            Console.WriteLine();

            foreach (string format in ReportFactory.SupportedFormats)
            {
                var generator = ReportFactory.Create(format);
                string report = generator.GenerateReport(data);

                Console.WriteLine($"=== {generator.FormatName} ({report.Length} chars) ===");
                // แสดงแค่ส่วนแรก
                var lines = report.Split('\n');
                foreach (string line in lines.Take(8))
                    Console.WriteLine(line);
                if (lines.Length > 8)
                    Console.WriteLine($"... (+ {lines.Length - 8} บรรทัด)");
                Console.WriteLine();
            }
        }
    }
}
```

---

## Exercises

### Exercise 1: Data Validator
```csharp
// TODO: สร้าง abstract DataValidator ด้วย Template Method:
// - Validate(T data) -> ValidationResult
// - abstract ValidateRequired(T data)
// - abstract ValidateFormat(T data)
// - abstract ValidateBusinessRules(T data)
// - virtual OnValidationComplete(ValidationResult result)
// Implementations:
// - EmailValidator
// - PhoneValidator
// - PasswordValidator
// - AgeValidator

public abstract class DataValidator<T>
{
    // TODO: Implement Template Method
}
```

### Exercise 2: File Processor
```csharp
// TODO: สร้าง abstract FileProcessor ด้วย Template Method:
// Process(string filePath) -> ProcessResult
// Abstract: ReadFile, ParseContent, TransformData, WriteOutput
// Hook: OnProgress(int percent), OnError(Exception ex)
// Concrete implementations:
// - CsvToJsonProcessor
// - JsonToCsvProcessor
// - TextStatisticsProcessor

public abstract class FileProcessor
{
    // TODO: Implement
}
```

### Exercise 3: UI Component
```csharp
// TODO: สร้าง abstract UIComponent ด้วย Template Method:
// Render() -> string (HTML)
// Abstract: RenderContent()
// Virtual: RenderWrapper(), AddAttributes()
// Concrete classes:
// - Button, TextInput, Dropdown, DataTable
// ทุก component ต้องมี: id, class, disabled state, validation

public abstract class UIComponent
{
    // TODO: Implement
}
```

---

## สรุป

✅ Abstract class ไม่สามารถ instantiate โดยตรง - ต้องสร้าง derived class  
✅ Abstract methods บังคับให้ derived classes ต้อง implement  
✅ Template Method Pattern กำหนดโครงสร้าง algorithm ใน abstract class  
✅ Hook methods ใน Template Method เป็น optional overrides  
✅ Abstract class ดีกว่า interface เมื่อ: มี state, มี constructor, ต้องการ code reuse  
✅ Interface ดีกว่า abstract class เมื่อ: ต้องการ multiple behaviors  
✅ ใช้ทั้งสอง: class extend abstract + implement interfaces  

## Part ถัดไป
**Part 019: Enums และ Structs** - เรียนรู้ enum declarations, Flags enum, struct vs class และ readonly struct

---
*Part 018/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
