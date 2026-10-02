# Part 030: Nullable Types ขั้นสูง

## เนื้อหาใน Part นี้
- Nullable\<T\> vs T? สำหรับ Value Types
- Null-conditional operators (?. และ ?[])
- Null-coalescing operators (?? และ ??=)
- Nullable Reference Types (C# 8+)
- Nullability Annotations
- Pattern Matching กับ null
- แนวทางการเขียน Null-safe code
- โปรแกรมตัวอย่าง: Safe User Profile Handling

---

## 1. Nullable\<T\> vs T?

Value Types ปกติ (int, double, bool, struct) ไม่สามารถเป็น null ได้ แต่ Nullable\<T\> แก้ปัญหานี้

```csharp
// ===== Value Types ปกติ - ไม่สามารถเป็น null =====
int notNullable = 42;
// int cannotBeNull = null; // Error!

// ===== Nullable<T> =====
Nullable<int> nullableInt = null;
Nullable<double> nullableDouble = 3.14;
Nullable<bool> nullableBool = true;

// ย่อด้วย ? syntax (เหมือนกันทุกประการ)
int? shortInt = null;
double? shortDouble = 3.14;
bool? shortBool = true;
DateTime? shortDate = null;

// ===== Properties ของ Nullable<T> =====
int? value = 42;
Console.WriteLine(value.HasValue); // true
Console.WriteLine(value.Value);    // 42

int? nullValue = null;
Console.WriteLine(nullValue.HasValue); // false
// Console.WriteLine(nullValue.Value); // throws InvalidOperationException!

// ===== GetValueOrDefault =====
int defaultResult = nullValue.GetValueOrDefault();    // 0 (default(int))
int customDefault = nullValue.GetValueOrDefault(99);  // 99

// ===== Boxing Nullable =====
int? boxedNullable = 42;
object obj = boxedNullable; // box ค่า 42 (ไม่ใช่ Nullable<int>!)

int? unboxed = (int?)obj; // unbox

// null nullable boxing
int? nullBox = null;
object nullObj = nullBox; // null object (ไม่ใช่ Nullable<int> ที่ null)
Console.WriteLine(nullObj == null); // true

// ===== Nullable กับ Arithmetic =====
int? a = 10;
int? b = null;
int? c = 5;

int? sum = a + b;        // null (propagates)
int? product = a * c;    // 50
int? division = a / b;   // null

// Comparison
bool? equal = a == b;    // false (not null)
bool? greater = a > c;   // true
bool? lessNull = a < b;  // null? No: null comparisons return false

Console.WriteLine(a > b); // false (null ไม่ > ค่าอะไร)
Console.WriteLine(b > a); // false
Console.WriteLine(b == null); // true

// ===== Nullable Structs =====
public struct Point
{
    public int X { get; set; }
    public int Y { get; set; }
    public Point(int x, int y) { X = x; Y = y; }
}

Point? point = null;
Point? point2 = new Point(3, 4);

Console.WriteLine(point.HasValue);         // false
Console.WriteLine(point2?.X);              // 3
Console.WriteLine(point?.X ?? -1);         // -1 (default)
```

---

## 2. Null-conditional Operators (?. และ ?[])

```csharp
// ===== ?. (Null-conditional member access) =====
// ถ้า object เป็น null - คืน null แทนที่จะ throw NullReferenceException

string? name = null;

// ก่อน: ต้องตรวจสอบก่อน
int length1 = name != null ? name.Length : 0;

// หลัง: ?. อ่านง่ายกว่า
int? length2 = name?.Length; // null

// Chain
public class Address
{
    public string? City { get; set; }
    public string? ZipCode { get; set; }
    public Country? Country { get; set; }
}

public class Country
{
    public string? Name { get; set; }
    public string? Code { get; set; }
}

public class Person
{
    public string? Name { get; set; }
    public Address? Address { get; set; }
}

Person? person = null;
string? city = person?.Address?.City; // null (ไม่ throw)

Person person2 = new Person
{
    Name = "Alice",
    Address = new Address
    {
        City = "Bangkok",
        Country = new Country { Name = "Thailand", Code = "TH" }
    }
};

// Chain หลายระดับ
string? countryCode = person2?.Address?.Country?.Code; // "TH"
string? zipCode = person2?.Address?.ZipCode;           // null (ZipCode ไม่ได้ set)

// ===== ?. กับ Methods =====
string? text = null;
string? upper = text?.ToUpper();        // null
int? len = text?.Length;                // null

List<string>? list = null;
int? count = list?.Count;              // null
list?.Add("item");                     // ไม่ทำอะไรถ้า null

// ?. กับ Events
event Action? OnEvent;
OnEvent?.Invoke(); // thread-safe null check!

// ===== ?[] (Null-conditional indexer) =====
string[]? arr = null;
string? first = arr?[0];              // null (ไม่ throw)
int? arrLength = arr?.Length;        // null

Dictionary<string, int>? dict = null;
// int? val = dict?["key"]; // null - แต่ถ้า key ไม่อยู่ยัง KeyNotFoundException

Dictionary<string, int> dict2 = new() { ["a"] = 1, ["b"] = 2 };
int? value = dict2.TryGetValue("a", out int v) ? v : (int?)null; // safe

// ?[] ใช้บ่อยกับ List
List<string>? names = null;
string? firstName = names?[0]; // null

// ===== ?. กับ Casting =====
object? obj = "Hello";
int? intValue = (obj as string)?.Length; // 5
```

---

## 3. Null-coalescing Operators (?? และ ??=)

```csharp
// ===== ?? (Null-coalescing) =====
// คืนค่าซ้ายถ้าไม่ null, ไม่งั้นคืนค่าขวา

string? name = null;
string result = name ?? "Unknown";     // "Unknown"

name = "Alice";
string result2 = name ?? "Unknown";   // "Alice"

// Chain ??
string? a = null;
string? b = null;
string? c = "Found!";
string d = a ?? b ?? c ?? "Default"; // "Found!"

// ?? กับ method calls
int pageSize = GetPageSizeFromConfig() ?? 10;
string host = Environment.GetEnvironmentVariable("HOST") ?? "localhost";

int? GetPageSizeFromConfig() => null; // simulated

// ===== ??= (Null-coalescing assignment) C# 8+ =====
// กำหนดค่าเฉพาะกรณีที่ตัวแปรเป็น null

string? text = null;
text ??= "Default Value"; // text กำหนดเป็น "Default Value"
Console.WriteLine(text); // "Default Value"

text ??= "Other"; // text ไม่เปลี่ยนเพราะไม่เป็น null
Console.WriteLine(text); // "Default Value"

// ??= ใช้บ่อยสำหรับ Lazy initialization
private List<string>? _items;
public List<string> Items => _items ??= new List<string>();

// ตัวอย่างใช้งาน ??=
Dictionary<string, List<int>> groups = new();
string key = "numbers";

groups.TryGetValue(key, out List<int>? existingList);
existingList ??= new List<int>(); // สร้างใหม่ถ้ายังไม่มี
existingList.Add(42);
groups[key] = existingList;

// Cleaner with ??=
List<int>? cached = null;
cached ??= LoadFromDatabase(); // โหลดเมื่อจำเป็น
List<int> LoadFromDatabase() => new() { 1, 2, 3 };

// ===== ?? กับ Throw Expression (C# 7+) =====
string? input = null;
string safe = input ?? throw new ArgumentNullException(nameof(input));

// ใช้บ่อยใน constructor validation
public class UserService
{
    private readonly string _connectionString;

    public UserService(string? connectionString)
    {
        _connectionString = connectionString
            ?? throw new ArgumentNullException(nameof(connectionString));
    }
}

// หรือใช้ ArgumentNullException.ThrowIfNull (C# 10+)
public class Repository
{
    private readonly string _connStr;
    public Repository(string? connStr)
    {
        ArgumentNullException.ThrowIfNull(connStr);
        _connStr = connStr;
    }
}
```

---

## 4. Nullable Reference Types (C# 8+)

```csharp
// ===== Enable Nullable Reference Types =====
// ใน .csproj: <Nullable>enable</Nullable>
// หรือ ต่อไฟล์: #nullable enable

#nullable enable

// ===== Non-nullable Reference Types =====
string nonNullable = "Hello"; // ต้องไม่เป็น null
// nonNullable = null; // Warning! CS8600

// ===== Nullable Reference Types =====
string? nullable = null; // บอกว่าอาจเป็น null
nullable = "World";
nullable = null;

// ===== ต้องตรวจสอบก่อนใช้ =====
string? maybeNull = GetString();
// Console.WriteLine(maybeNull.Length); // Warning! CS8602

if (maybeNull != null)
    Console.WriteLine(maybeNull.Length); // OK

// หรือใช้ ?.
Console.WriteLine(maybeNull?.Length);

// ===== Null-forgiving operator (!) =====
// บอก compiler ว่าแน่ใจว่าไม่ null (suppress warning)
string? value = GetString();
// ใช้เฉพาะเมื่อแน่ใจ 100%!
Console.WriteLine(value!.Length); // ไม่มี warning แต่อาจ crash ถ้าจริงๆ เป็น null

string? GetString() => null;

// ===== ใน Class Properties =====
#nullable enable

public class Customer
{
    // Non-nullable: ต้อง initialize
    public int Id { get; set; }
    public string Name { get; set; } = ""; // default ที่ไม่ null
    public string Email { get; set; }

    // Nullable: อาจเป็น null
    public string? Phone { get; set; }
    public string? Address { get; set; }
    public DateTime? BirthDate { get; set; }

    // Constructor เพื่อ ensure non-null
    public Customer(int id, string name, string email)
    {
        Id = id;
        Name = name ?? throw new ArgumentNullException(nameof(name));
        Email = email ?? throw new ArgumentNullException(nameof(email));
    }
}

// ===== Nullable Parameters =====
public string FormatName(string firstName, string? lastName = null)
{
    return lastName != null
        ? $"{firstName} {lastName}"
        : firstName;
}

Console.WriteLine(FormatName("Alice", "Smith")); // Alice Smith
Console.WriteLine(FormatName("Bob"));            // Bob

// ===== Return Types =====
public string? FindByEmail(string email)
{
    // Return null ถ้าไม่พบ
    return email == "admin@example.com" ? "Admin" : null;
}

string? found = FindByEmail("unknown@example.com");
// found อาจเป็น null - compiler บังคับให้ตรวจ

if (found is not null)
    Console.WriteLine(found.ToUpper());
```

---

## 5. Nullability Annotations (C# 8+)

```csharp
using System.Diagnostics.CodeAnalysis;

// ===== MaybeNull =====
// บอกว่า output อาจเป็น null (แม้ return type จะไม่ nullable)
[return: MaybeNull]
public T GetOrDefault<T>(Dictionary<string, T> dict, string key)
{
    dict.TryGetValue(key, out T? value);
    return value!;
}

// ===== NotNull =====
// บอกว่า output ไม่เป็น null เมื่อ method succeed
[return: NotNull]
public string GetRequiredString(string? value, string paramName)
{
    return value ?? throw new ArgumentNullException(paramName);
}

// ===== NotNullWhen =====
// Parameter จะ not-null เมื่อ return true/false
public bool TryGetValue<T>(string key, [NotNullWhen(true)] out T? value)
{
    // ...
    value = default;
    return false;
}

// ===== MaybeNullWhen =====
public bool TryParse(string? input, [MaybeNullWhen(false)] out int value)
{
    return int.TryParse(input, out value);
}

// ===== NotNullIfNotNull =====
// Output เป็น null เมื่อ input เป็น null
[return: NotNullIfNotNull("input")]
public string? ToUpperOrNull(string? input)
{
    return input?.ToUpper();
}

// ===== DoesNotReturn =====
// Method ไม่ return (throw หรือ infinite loop)
[DoesNotReturn]
public void ThrowError(string message)
{
    throw new InvalidOperationException(message);
}

// ===== DoesNotReturnIf =====
// Method throw เมื่อ parameter เป็น true/false
public void Assert([DoesNotReturnIf(false)] bool condition, string message)
{
    if (!condition)
        throw new InvalidOperationException(message);
}

// ===== AllowNull =====
// Property setter รับ null แต่ getter return non-null
public class SafeString
{
    private string _value = "";

    [AllowNull]
    public string Value
    {
        get => _value;
        set => _value = value ?? ""; // null -> empty string
    }
}

var s = new SafeString();
s.Value = null;              // OK (AllowNull)
string v = s.Value;          // guaranteed non-null
```

---

## 6. Pattern Matching กับ null

```csharp
// ===== is null / is not null (C# 9+) =====
string? name = null;

if (name is null)
    Console.WriteLine("Name is null");

if (name is not null)
    Console.WriteLine($"Name is: {name}");

// ===== switch Expression กับ null =====
string? status = null;
string result = status switch
{
    null => "Unknown",
    "" => "Empty",
    "active" => "User is active",
    "inactive" => "User is inactive",
    _ => $"Status: {status}"
};

Console.WriteLine(result); // "Unknown"

// ===== Pattern Matching ใน Null-safe way =====
object? obj = GetObject();

string description = obj switch
{
    null => "nothing",
    string s when s.Length == 0 => "empty string",
    string s => $"string: '{s}'",
    int n when n > 0 => $"positive int: {n}",
    int n => $"non-positive int: {n}",
    List<int> list => $"list with {list.Count} items",
    _ => $"unknown: {obj.GetType().Name}"
};

object? GetObject() => "Hello World";

// ===== Nested Null Pattern Matching =====
public class Order
{
    public int Id { get; set; }
    public Customer? Customer { get; set; }
    public List<OrderItem>? Items { get; set; }
}

public record OrderItem(string Name, decimal Price, int Qty);

Order? order = GetOrder();

// Pattern match ซ้อนกัน
string summary = order switch
{
    null => "No order",
    { Customer: null } => "Order without customer",
    { Customer.Name: var name, Items: null or { Count: 0 } } => $"{name}'s empty order",
    { Customer.Name: var name, Items: { Count: var count } } =>
        $"{name}'s order with {count} items",
};

Console.WriteLine(summary);

Order? GetOrder() => new Order
{
    Id = 1,
    Customer = new Customer(1, "Alice", "alice@example.com"),
    Items = new List<OrderItem>
    {
        new("Laptop", 35000, 1),
        new("Mouse", 890, 2)
    }
};

// ===== when clause กับ null =====
void Process(object? input)
{
    if (input is string s and { Length: > 0 })
        Console.WriteLine($"Non-empty string: {s}");
    else if (input is int n and > 0)
        Console.WriteLine($"Positive int: {n}");
    else if (input is null)
        Console.WriteLine("null input");
    else
        Console.WriteLine($"Other: {input}");
}
```

---

## 7. Best Practices สำหรับ Null-safe Code

```csharp
// ===== Null Object Pattern =====
// แทนที่ null ด้วย null object ที่มี default behavior
public interface ILogger
{
    void Log(string message);
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
}

// Null Object - ไม่ทำอะไร แต่ไม่ throw
public class NullLogger : ILogger
{
    public static readonly NullLogger Instance = new();
    private NullLogger() { }
    public void Log(string message) { } // do nothing
}

// ใช้งาน
ILogger logger = GetLogger() ?? NullLogger.Instance;
logger.Log("No null check needed!"); // safe เสมอ

ILogger? GetLogger() => null; // simulated

// ===== Guard Clauses =====
public void ProcessUser(string? name, int? age)
{
    // Guard เร็ว fail fast
    ArgumentNullException.ThrowIfNull(name);
    ArgumentOutOfRangeException.ThrowIfNull(age);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(age.Value);

    // main logic ต่อไปได้โดยแน่ใจว่าค่าถูกต้อง
    Console.WriteLine($"Processing: {name}, age {age.Value}");
}

// ===== Null-safe Method Chaining =====
public class Builder
{
    private string? _name;
    private int? _age;

    public Builder WithName(string? name)
    {
        _name = name;
        return this;
    }

    public Builder WithAge(int? age)
    {
        _age = age;
        return this;
    }

    public Customer? Build()
    {
        if (_name is null) return null;
        return new Customer(1, _name, _age?.ToString() ?? "");
    }
}

// ===== Try Pattern =====
// แทนที่ throw/catch ด้วย bool return + out parameter
public bool TryParseDate(string? input, out DateTime date)
{
    if (string.IsNullOrEmpty(input))
    {
        date = default;
        return false;
    }
    return DateTime.TryParse(input, out date);
}

if (TryParseDate("2024-01-15", out DateTime parsed))
    Console.WriteLine($"Parsed: {parsed:D}");
else
    Console.WriteLine("Invalid date");

// ===== Avoid Null with Records =====
// Records ช่วย enforce non-null
public record UserProfile(
    string Name,       // non-null
    string Email,      // non-null
    string? Phone,     // optional
    string? Bio        // optional
);

// ไม่สามารถสร้าง UserProfile ที่มี Name = null ได้ (ถ้า enable nullable)
// var profile = new UserProfile(null!, "email"); // กลายเป็น runtime issue ถ้า bypass
```

---

## 8. โปรแกรมตัวอย่าง: Safe User Profile Handling

```csharp
#nullable enable

using System;
using System.Collections.Generic;
using System.Diagnostics.CodeAnalysis;
using System.Linq;

namespace SafeUserProfile
{
    // ===== Models =====
    public record Address(
        string Street,
        string City,
        string? State,
        string Country,
        string? ZipCode
    );

    public record SocialLinks(
        string? Facebook,
        string? Twitter,
        string? LinkedIn,
        string? GitHub
    );

    public class UserProfile
    {
        public int Id { get; init; }
        public string Username { get; init; }
        public string Email { get; init; }
        public string? DisplayName { get; set; }
        public string? Bio { get; set; }
        public string? AvatarUrl { get; set; }
        public DateTime? BirthDate { get; set; }
        public Address? Address { get; set; }
        public SocialLinks? Social { get; set; }
        public List<string>? Skills { get; set; }
        public DateTime CreatedAt { get; init; }
        public DateTime? UpdatedAt { get; set; }
        public bool IsActive { get; set; }

        public UserProfile(int id, string username, string email)
        {
            ArgumentNullException.ThrowIfNull(username);
            ArgumentNullException.ThrowIfNull(email);

            Id = id;
            Username = username;
            Email = email;
            CreatedAt = DateTime.UtcNow;
            IsActive = true;
        }

        // Computed properties ที่ null-safe
        public string EffectiveName => DisplayName ?? Username;
        public int? Age => BirthDate.HasValue
            ? (int)((DateTime.Today - BirthDate.Value).TotalDays / 365.25)
            : null;

        public string Location =>
            Address is { City: var city, Country: var country }
                ? $"{city}, {country}"
                : "Location not set";

        public bool HasCompleteProfile =>
            DisplayName is not null &&
            Bio is not null &&
            Address is not null &&
            AvatarUrl is not null;
    }

    // ===== Repository =====
    public class UserRepository
    {
        private readonly Dictionary<int, UserProfile> _users = new();
        private int _nextId = 1;

        public UserProfile Create(string username, string email)
        {
            if (string.IsNullOrWhiteSpace(username))
                throw new ArgumentException("Username cannot be empty", nameof(username));

            if (_users.Values.Any(u => u.Username.Equals(username, StringComparison.OrdinalIgnoreCase)))
                throw new InvalidOperationException($"Username '{username}' already exists");

            var user = new UserProfile(_nextId++, username, email);
            _users[user.Id] = user;
            return user;
        }

        public UserProfile? GetById(int id)
        {
            _users.TryGetValue(id, out UserProfile? user);
            return user;
        }

        public UserProfile? GetByUsername(string? username)
        {
            if (string.IsNullOrEmpty(username)) return null;
            return _users.Values.FirstOrDefault(u =>
                u.Username.Equals(username, StringComparison.OrdinalIgnoreCase));
        }

        public IEnumerable<UserProfile> GetAll() => _users.Values;

        public bool Update(UserProfile updated)
        {
            if (!_users.ContainsKey(updated.Id)) return false;
            _users[updated.Id] = updated;
            return true;
        }

        public bool Delete(int id) => _users.Remove(id);
    }

    // ===== Service Layer - null-safe operations =====
    public class UserProfileService
    {
        private readonly UserRepository _repo;

        public UserProfileService(UserRepository repo)
        {
            _repo = repo ?? throw new ArgumentNullException(nameof(repo));
        }

        // Safe get with default
        public string GetDisplayName(int userId)
        {
            var user = _repo.GetById(userId);
            return user?.EffectiveName ?? "Anonymous";
        }

        // Safe social link access
        public string? GetSocialLink(int userId, string platform)
        {
            var user = _repo.GetById(userId);
            if (user?.Social is null) return null;

            return platform.ToLower() switch
            {
                "facebook" => user.Social.Facebook,
                "twitter" => user.Social.Twitter,
                "linkedin" => user.Social.LinkedIn,
                "github" => user.Social.GitHub,
                _ => null
            };
        }

        // Update profile ด้วย null-safe
        public bool UpdateProfile(int userId, Action<UserProfile> updater)
        {
            var user = _repo.GetById(userId);
            if (user is null) return false;

            updater(user);
            user.UpdatedAt = DateTime.UtcNow;
            return _repo.Update(user);
        }

        // Search users - null-safe
        public List<UserProfile> SearchUsers(
            string? keyword = null,
            string? city = null,
            string? skill = null,
            bool? activeOnly = null)
        {
            IEnumerable<UserProfile> query = _repo.GetAll();

            if (activeOnly == true)
                query = query.Where(u => u.IsActive);

            if (!string.IsNullOrEmpty(keyword))
                query = query.Where(u =>
                    u.Username.Contains(keyword, StringComparison.OrdinalIgnoreCase) ||
                    (u.DisplayName?.Contains(keyword, StringComparison.OrdinalIgnoreCase) ?? false) ||
                    (u.Bio?.Contains(keyword, StringComparison.OrdinalIgnoreCase) ?? false));

            if (!string.IsNullOrEmpty(city))
                query = query.Where(u =>
                    u.Address?.City.Equals(city, StringComparison.OrdinalIgnoreCase) ?? false);

            if (!string.IsNullOrEmpty(skill))
                query = query.Where(u =>
                    u.Skills?.Any(s => s.Equals(skill, StringComparison.OrdinalIgnoreCase)) ?? false);

            return query.OrderBy(u => u.Username).ToList();
        }

        // Profile completeness report
        public (int Complete, int Incomplete, double Rate) GetCompletenessStats()
        {
            var all = _repo.GetAll().ToList();
            int complete = all.Count(u => u.HasCompleteProfile);
            int incomplete = all.Count - complete;
            double rate = all.Count > 0 ? complete * 100.0 / all.Count : 0;
            return (complete, incomplete, rate);
        }

        // สรุปข้อมูล user แบบ null-safe
        public void PrintProfile(int userId)
        {
            var user = _repo.GetById(userId);
            if (user is null)
            {
                Console.WriteLine($"User #{userId} not found");
                return;
            }

            Console.WriteLine($"\n{'─', -40}");
            Console.WriteLine($"User Profile: {user.EffectiveName} (@{user.Username})");
            Console.WriteLine($"{'─', -40}");
            Console.WriteLine($"  ID: {user.Id}");
            Console.WriteLine($"  Email: {user.Email}");
            Console.WriteLine($"  Display Name: {user.DisplayName ?? "(not set)"}");
            Console.WriteLine($"  Age: {user.Age?.ToString() ?? "Unknown"}");
            Console.WriteLine($"  Location: {user.Location}");
            Console.WriteLine($"  Bio: {user.Bio ?? "(no bio)"}");
            Console.WriteLine($"  Avatar: {user.AvatarUrl ?? "(no avatar)"}");
            Console.WriteLine($"  Active: {user.IsActive}");
            Console.WriteLine($"  Joined: {user.CreatedAt:d}");

            if (user.Skills?.Any() == true)
                Console.WriteLine($"  Skills: {string.Join(", ", user.Skills)}");

            if (user.Social is not null)
            {
                var links = new List<string>();
                if (user.Social.GitHub is not null) links.Add($"GitHub: {user.Social.GitHub}");
                if (user.Social.LinkedIn is not null) links.Add($"LinkedIn: {user.Social.LinkedIn}");
                if (user.Social.Twitter is not null) links.Add($"Twitter: {user.Social.Twitter}");
                if (links.Any())
                    Console.WriteLine($"  Social: {string.Join(", ", links)}");
            }

            Console.WriteLine($"  Profile Complete: {user.HasCompleteProfile}");
        }
    }

    class Program
    {
        static void Main()
        {
            Console.WriteLine("=== Safe User Profile System ===\n");

            var repo = new UserRepository();
            var service = new UserProfileService(repo);

            // สร้าง users
            var alice = repo.Create("alice_dev", "alice@example.com");
            var bob = repo.Create("bob_design", "bob@example.com");
            var charlie = repo.Create("charlie_pm", "charlie@example.com");
            var diana = repo.Create("diana_data", "diana@example.com");

            // ตั้งค่า profiles ต่างๆ
            service.UpdateProfile(alice.Id, u =>
            {
                u.DisplayName = "Alice Johnson";
                u.Bio = "Full-stack developer passionate about C# and .NET";
                u.AvatarUrl = "https://example.com/alice.jpg";
                u.BirthDate = new DateTime(1995, 3, 15);
                u.Address = new Address("123 Dev St", "Bangkok", null, "Thailand", "10110");
                u.Social = new SocialLinks(null, "@alice_dev", "alice-johnson", "alice-j");
                u.Skills = new List<string> { "C#", "ASP.NET", "SQL", "Azure" };
            });

            service.UpdateProfile(bob.Id, u =>
            {
                u.DisplayName = "Bob Smith";
                u.AvatarUrl = "https://example.com/bob.jpg";
                // No bio, no address
                u.Skills = new List<string> { "Figma", "CSS", "JavaScript" };
                u.Social = new SocialLinks("bob.smith", "@bobsmith", null, null);
            });

            // ไม่ set profile ของ charlie และ diana (partial)
            service.UpdateProfile(charlie.Id, u =>
            {
                u.Address = new Address("456 PM Ave", "Chiang Mai", null, "Thailand", null);
                u.Skills = new List<string> { "Scrum", "JIRA", "Communication" };
            });

            // ===== Test Null-safe Operations =====
            Console.WriteLine("1. Get display names (with null-safe fallback):");
            for (int i = 1; i <= 5; i++)
            {
                string name = service.GetDisplayName(i);
                Console.WriteLine($"   User #{i}: {name}");
            }

            // ?. และ ?? ใน action
            Console.WriteLine("\n2. Safe property access:");
            var user = repo.GetById(1);
            Console.WriteLine($"   Country: {user?.Address?.Country ?? "Unknown"}");
            Console.WriteLine($"   GitHub: {user?.Social?.GitHub ?? "Not set"}");
            Console.WriteLine($"   Phone: {user?.Bio?[..20] ?? "No bio"}...");

            // Pattern matching
            Console.WriteLine("\n3. Pattern matching with null:");
            foreach (var u in repo.GetAll())
            {
                string ageInfo = u.Age switch
                {
                    null => "Age unknown",
                    < 18 => "Minor",
                    >= 18 and < 30 => "Young adult",
                    >= 30 and < 60 => "Adult",
                    _ => "Senior"
                };
                Console.WriteLine($"   {u.Username}: {ageInfo}");
            }

            // Search (null-safe)
            Console.WriteLine("\n4. Search users (keyword='dev', skill='C#'):");
            var results = service.SearchUsers(keyword: "dev", skill: "C#");
            foreach (var r in results)
                Console.WriteLine($"   Found: {r.EffectiveName} - {r.Location}");

            Console.WriteLine("\n5. Search by city:");
            var bangkokUsers = service.SearchUsers(city: "Bangkok");
            Console.WriteLine($"   Bangkok users: {bangkokUsers.Count}");
            foreach (var u2 in bangkokUsers)
                Console.WriteLine($"   - {u2.EffectiveName}");

            // Profile completeness
            var (complete, incomplete, rate) = service.GetCompletenessStats();
            Console.WriteLine($"\n6. Profile Completeness:");
            Console.WriteLine($"   Complete: {complete}, Incomplete: {incomplete}");
            Console.WriteLine($"   Rate: {rate:F1}%");

            // Print profiles
            Console.WriteLine("\n7. Detailed Profiles:");
            service.PrintProfile(1); // Alice - complete
            service.PrintProfile(2); // Bob - partial
            service.PrintProfile(99); // Not found

            // ??= demo
            Console.WriteLine("\n8. Lazy initialization with ??=:");
            var target = repo.GetById(4)!;
            target.Skills ??= new List<string>();
            target.Skills.Add("Python");
            target.Skills ??= new List<string> { "Java" }; // ไม่เปลี่ยนแล้ว
            Console.WriteLine($"   Diana's skills: {string.Join(", ", target.Skills)}");
        }
    }
}
```

---

## Exercises

### Exercise 1: Safe JSON Parser
สร้าง JSON parser ที่:
- รับ JSON string ที่อาจเป็น null
- ใช้ `?[]` สำหรับ JsonElement
- คืน `Optional<T>` แทน null

### Exercise 2: Nullable Configuration
สร้าง configuration reader ที่:
- อ่านค่า nullable จาก config
- ใช้ ?? สำหรับ default values
- Validate ด้วย pattern matching

### Exercise 3: Database Mapper
สร้าง database row mapper ที่:
- Map nullable DB values ไป nullable C# properties
- ใช้ Nullable annotations
- Handle DBNull อย่างถูกต้อง

---

## สรุป

- ✅ `T?` (Nullable\<T\>) ทำให้ Value Types เป็น null ได้
- ✅ `.HasValue` และ `.Value` สำหรับตรวจสอบ/อ่านค่า
- ✅ `?.` null-conditional: ไม่ throw ถ้าเป็น null, คืน null แทน
- ✅ `?[]` null-conditional indexer สำหรับ array/dictionary
- ✅ `??` null-coalescing: คืน default ถ้าเป็น null
- ✅ `??=` null-coalescing assignment: กำหนดค่าเฉพาะกรณีที่เป็น null
- ✅ Nullable Reference Types (C# 8+): แยกแยะ nullable/non-nullable reference types
- ✅ `!` null-forgiving operator: ระงับ warning (ใช้ด้วยความระมัดระวัง)
- ✅ `is null` / `is not null` pattern matching ที่ชัดเจน
- ✅ Annotations เช่น `[NotNullWhen]`, `[MaybeNull]` ช่วย Flow analysis
- ✅ Null Object Pattern แก้ปัญหาการ check null ซ้ำๆ

## Part ถัดไป
**Part 031** จะพูดถึง Async/Await และ Task-based Asynchronous Pattern (TAP)

---
*Part 030/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
