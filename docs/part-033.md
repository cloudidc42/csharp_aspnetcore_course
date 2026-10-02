# Part 033: Extension Methods

## เนื้อหาใน Part นี้
- Extension Methods คืออะไร
- สร้าง Extension Method
- Extension Methods กับ LINQ
- Extension Methods สำหรับ string, int, IEnumerable
- Best Practices
- โปรแกรมตัวอย่าง: String/Number Utility Extensions

---

## 1. Extension Methods คืออะไร

Extension Methods คือ methods ที่ "เพิ่มเข้าไป" ใน type ที่มีอยู่แล้วโดยไม่ต้องแก้ไข source code ของ type นั้น

### โครงสร้าง Extension Method

```csharp
// Extension method ต้องอยู่ใน static class
public static class StringExtensions
{
    // Parameter แรกต้องมี 'this' keyword
    public static bool IsNullOrEmpty(this string? value)
    {
        return string.IsNullOrEmpty(value);
    }
    
    // Extension method ที่รับ parameters เพิ่มเติม
    public static string Repeat(this string value, int times)
    {
        return string.Concat(Enumerable.Repeat(value, times));
    }
}

// ใช้งาน
class Program
{
    static void Main()
    {
        string? name = null;
        
        // เรียกแบบ extension method
        bool isEmpty = name.IsNullOrEmpty(); // true
        
        // เรียกแบบ static method (เหมือนกัน)
        bool isEmpty2 = StringExtensions.IsNullOrEmpty(name); // true
        
        string hello = "Hello";
        string repeated = hello.Repeat(3); // "HelloHelloHello"
        Console.WriteLine(repeated);
    }
}
```

### ทำไมต้องใช้ Extension Methods

```csharp
// ❌ แบบเดิม - verbose, ไม่ readable
bool result1 = string.IsNullOrEmpty(someText);
string result2 = string.Format("{0:N2}", someNumber);
int result3 = Math.Abs(someInt);

// ✅ ด้วย extension methods - readable, fluent
bool result4 = someText.IsNullOrEmpty();
string result5 = someNumber.ToFormattedString("N2");
int result6 = someInt.Abs();

// Method chaining ที่ readable
string processed = "  Hello World  "
    .Trim()
    .ToLower()
    .Replace(" ", "-")
    .Truncate(10);
```

---

## 2. สร้าง Extension Method

### Extension Methods สำหรับ string

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;

public static class StringExtensions
{
    // ตรวจสอบ string
    public static bool IsNullOrEmpty(this string? value) =>
        string.IsNullOrEmpty(value);
    
    public static bool IsNullOrWhiteSpace(this string? value) =>
        string.IsNullOrWhiteSpace(value);
    
    public static bool HasValue(this string? value) =>
        !string.IsNullOrWhiteSpace(value);
    
    // ตัดสั้น
    public static string Truncate(this string value, int maxLength, string ellipsis = "...")
    {
        if (string.IsNullOrEmpty(value)) return value ?? string.Empty;
        if (value.Length <= maxLength) return value;
        return value[..(maxLength - ellipsis.Length)] + ellipsis;
    }
    
    // แปลง case
    public static string ToCamelCase(this string value)
    {
        if (string.IsNullOrEmpty(value)) return value ?? string.Empty;
        return char.ToLower(value[0]) + value[1..];
    }
    
    public static string ToPascalCase(this string value)
    {
        if (string.IsNullOrEmpty(value)) return value ?? string.Empty;
        return char.ToUpper(value[0]) + value[1..];
    }
    
    public static string ToSnakeCase(this string value)
    {
        if (string.IsNullOrEmpty(value)) return value ?? string.Empty;
        
        var result = new StringBuilder();
        for (int i = 0; i < value.Length; i++)
        {
            if (char.IsUpper(value[i]) && i > 0)
                result.Append('_');
            result.Append(char.ToLower(value[i]));
        }
        return result.ToString();
    }
    
    public static string ToKebabCase(this string value)
    {
        if (string.IsNullOrEmpty(value)) return value ?? string.Empty;
        return Regex.Replace(value, @"([A-Z])", "-$1")
                    .TrimStart('-')
                    .ToLower();
    }
    
    // แปลงเป็น types อื่น
    public static int ToInt(this string value, int defaultValue = 0) =>
        int.TryParse(value, out int result) ? result : defaultValue;
    
    public static double ToDouble(this string value, double defaultValue = 0) =>
        double.TryParse(value, out double result) ? result : defaultValue;
    
    public static bool ToBool(this string value, bool defaultValue = false) =>
        bool.TryParse(value, out bool result) ? result : defaultValue;
    
    public static T? ToEnum<T>(this string value) where T : struct, Enum =>
        Enum.TryParse<T>(value, ignoreCase: true, out T result) ? result : null;
    
    // Repeat
    public static string Repeat(this string value, int times) =>
        string.Concat(Enumerable.Repeat(value, times));
    
    // Reverse
    public static string Reverse(this string value)
    {
        if (string.IsNullOrEmpty(value)) return value ?? string.Empty;
        return new string(value.Reverse().ToArray());
    }
    
    // Count occurrences
    public static int CountOccurrences(this string value, string pattern)
    {
        if (string.IsNullOrEmpty(value) || string.IsNullOrEmpty(pattern)) return 0;
        
        int count = 0;
        int index = 0;
        while ((index = value.IndexOf(pattern, index, StringComparison.OrdinalIgnoreCase)) != -1)
        {
            count++;
            index += pattern.Length;
        }
        return count;
    }
    
    // Email validation
    public static bool IsValidEmail(this string value)
    {
        if (string.IsNullOrWhiteSpace(value)) return false;
        return Regex.IsMatch(value, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
    }
    
    // URL validation
    public static bool IsValidUrl(this string value)
    {
        return Uri.TryCreate(value, UriKind.Absolute, out Uri? uri) &&
               (uri.Scheme == Uri.UriSchemeHttp || uri.Scheme == Uri.UriSchemeHttps);
    }
    
    // Remove special characters
    public static string RemoveSpecialCharacters(this string value)
    {
        return Regex.Replace(value, @"[^a-zA-Z0-9\s]", string.Empty);
    }
    
    // Mask sensitive data
    public static string MaskEmail(this string email)
    {
        if (!email.IsValidEmail()) return email;
        
        var parts = email.Split('@');
        string username = parts[0];
        string domain = parts[1];
        
        string masked = username.Length <= 2
            ? username[..1] + "***"
            : username[..2] + new string('*', username.Length - 2);
        
        return $"{masked}@{domain}";
    }
    
    // String formatting
    public static string Format(this string template, params object[] args) =>
        string.Format(template, args);
    
    public static string FormatWith(this string template, object data)
    {
        var properties = data.GetType().GetProperties();
        string result = template;
        
        foreach (var prop in properties)
        {
            result = result.Replace($"{{{prop.Name}}}", prop.GetValue(data)?.ToString() ?? "");
        }
        
        return result;
    }
    
    // Split and trim
    public static string[] SplitAndTrim(this string value, char separator) =>
        value.Split(separator)
             .Select(s => s.Trim())
             .Where(s => !string.IsNullOrEmpty(s))
             .ToArray();
    
    // Contains any
    public static bool ContainsAny(this string value, IEnumerable<string> values) =>
        values.Any(v => value.Contains(v, StringComparison.OrdinalIgnoreCase));
    
    // Title case
    public static string ToTitleCase(this string value) =>
        CultureInfo.CurrentCulture.TextInfo.ToTitleCase(value.ToLower());
}
```

### Extension Methods สำหรับ int และ numeric types

```csharp
using System;

public static class NumberExtensions
{
    // int extensions
    public static bool IsEven(this int value) => value % 2 == 0;
    public static bool IsOdd(this int value) => value % 2 != 0;
    public static bool IsPositive(this int value) => value > 0;
    public static bool IsNegative(this int value) => value < 0;
    public static bool IsZero(this int value) => value == 0;
    public static bool IsBetween(this int value, int min, int max) => value >= min && value <= max;
    public static bool IsPrime(this int value)
    {
        if (value < 2) return false;
        if (value == 2) return true;
        if (value % 2 == 0) return false;
        for (int i = 3; i <= Math.Sqrt(value); i += 2)
            if (value % i == 0) return false;
        return true;
    }
    
    public static int Abs(this int value) => Math.Abs(value);
    public static int Clamp(this int value, int min, int max) => Math.Clamp(value, min, max);
    public static int Factorial(this int value)
    {
        if (value < 0) throw new ArgumentException("ค่าต้องไม่ติดลบ");
        if (value == 0 || value == 1) return 1;
        int result = 1;
        for (int i = 2; i <= value; i++)
            result *= i;
        return result;
    }
    
    // Range operations
    public static IEnumerable<int> To(this int start, int end)
    {
        int step = start <= end ? 1 : -1;
        for (int i = start; i != end + step; i += step)
            yield return i;
    }
    
    public static void Times(this int count, Action action)
    {
        for (int i = 0; i < count; i++)
            action();
    }
    
    public static void Times(this int count, Action<int> action)
    {
        for (int i = 0; i < count; i++)
            action(i);
    }
    
    // Conversion
    public static TimeSpan Seconds(this int value) => TimeSpan.FromSeconds(value);
    public static TimeSpan Minutes(this int value) => TimeSpan.FromMinutes(value);
    public static TimeSpan Hours(this int value) => TimeSpan.FromHours(value);
    public static TimeSpan Days(this int value) => TimeSpan.FromDays(value);
    
    // Byte size formatting
    public static string ToFileSizeString(this long bytes)
    {
        string[] suffixes = { "B", "KB", "MB", "GB", "TB", "PB" };
        double size = bytes;
        int suffixIndex = 0;
        
        while (size >= 1024 && suffixIndex < suffixes.Length - 1)
        {
            size /= 1024;
            suffixIndex++;
        }
        
        return $"{size:F2} {suffixes[suffixIndex]}";
    }
    
    // double extensions
    public static double Abs(this double value) => Math.Abs(value);
    public static double Round(this double value, int digits = 0) => Math.Round(value, digits);
    public static double Floor(this double value) => Math.Floor(value);
    public static double Ceiling(this double value) => Math.Ceiling(value);
    public static bool IsNaN(this double value) => double.IsNaN(value);
    public static bool IsInfinity(this double value) => double.IsInfinity(value);
    public static string ToPercentage(this double value, int decimals = 1) =>
        $"{value * 100:F{decimals}}%";
    public static string ToCurrency(this decimal value, string symbol = "฿") =>
        $"{symbol}{value:N2}";
}
```

---

## 3. Extension Methods กับ LINQ

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

public static class EnumerableExtensions
{
    // DistinctBy (สำหรับ .NET ก่อน 6)
    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source,
        Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
        {
            if (seen.Add(keySelector(item)))
                yield return item;
        }
    }
    
    // Chunk - แบ่งเป็น batches
    public static IEnumerable<IEnumerable<T>> Chunk<T>(
        this IEnumerable<T> source, int size)
    {
        var list = source.ToList();
        for (int i = 0; i < list.Count; i += size)
        {
            yield return list.Skip(i).Take(size);
        }
    }
    
    // Flatten nested collections
    public static IEnumerable<T> Flatten<T>(this IEnumerable<IEnumerable<T>> source) =>
        source.SelectMany(x => x);
    
    // ForEach (Action version)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source)
            action(item);
    }
    
    public static void ForEach<T>(this IEnumerable<T> source, Action<T, int> action)
    {
        int index = 0;
        foreach (var item in source)
        {
            action(item, index++);
        }
    }
    
    // Async ForEach
    public static async Task ForEachAsync<T>(
        this IEnumerable<T> source,
        Func<T, Task> action)
    {
        foreach (var item in source)
            await action(item);
    }
    
    // None - ตรงข้าม Any
    public static bool None<T>(this IEnumerable<T> source) => !source.Any();
    public static bool None<T>(this IEnumerable<T> source, Func<T, bool> predicate) =>
        !source.Any(predicate);
    
    // IsEmpty
    public static bool IsEmpty<T>(this IEnumerable<T> source) => !source.Any();
    public static bool IsNullOrEmpty<T>(this IEnumerable<T>? source) =>
        source == null || !source.Any();
    
    // Safe operations
    public static T? FirstOrNull<T>(this IEnumerable<T> source) where T : class =>
        source.FirstOrDefault();
    
    public static IEnumerable<T> OrEmpty<T>(this IEnumerable<T>? source) =>
        source ?? Enumerable.Empty<T>();
    
    // Paginate
    public static IEnumerable<T> Page<T>(this IEnumerable<T> source, int page, int pageSize) =>
        source.Skip((page - 1) * pageSize).Take(pageSize);
    
    // Min/Max by
    public static T? MinBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> selector)
        where TKey : IComparable<TKey>
    {
        return source.OrderBy(selector).FirstOrDefault();
    }
    
    public static T? MaxBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> selector)
        where TKey : IComparable<TKey>
    {
        return source.OrderByDescending(selector).FirstOrDefault();
    }
    
    // Partition
    public static (IEnumerable<T> Matching, IEnumerable<T> NotMatching) Partition<T>(
        this IEnumerable<T> source,
        Func<T, bool> predicate)
    {
        var list = source.ToList();
        return (list.Where(predicate), list.Where(x => !predicate(x)));
    }
    
    // Zip with index
    public static IEnumerable<(T item, int index)> WithIndex<T>(this IEnumerable<T> source) =>
        source.Select((item, index) => (item, index));
    
    // Random
    public static T Random<T>(this IEnumerable<T> source)
    {
        var list = source.ToList();
        if (!list.Any()) throw new InvalidOperationException("Collection is empty");
        return list[new Random().Next(list.Count)];
    }
    
    public static IEnumerable<T> Shuffle<T>(this IEnumerable<T> source)
    {
        var list = source.ToList();
        var random = new Random();
        
        for (int i = list.Count - 1; i > 0; i--)
        {
            int j = random.Next(i + 1);
            (list[i], list[j]) = (list[j], list[i]);
        }
        
        return list;
    }
    
    // Convert to Dictionary safely
    public static Dictionary<TKey, TValue> ToDictionarySafe<T, TKey, TValue>(
        this IEnumerable<T> source,
        Func<T, TKey> keySelector,
        Func<T, TValue> valueSelector) where TKey : notnull
    {
        var dict = new Dictionary<TKey, TValue>();
        foreach (var item in source)
        {
            var key = keySelector(item);
            if (!dict.ContainsKey(key))
                dict[key] = valueSelector(item);
        }
        return dict;
    }
    
    // Statistical methods
    public static double Median(this IEnumerable<double> source)
    {
        var sorted = source.OrderBy(x => x).ToList();
        if (!sorted.Any()) throw new InvalidOperationException("Collection is empty");
        
        int mid = sorted.Count / 2;
        return sorted.Count % 2 == 0
            ? (sorted[mid - 1] + sorted[mid]) / 2.0
            : sorted[mid];
    }
    
    public static double StandardDeviation(this IEnumerable<double> source)
    {
        var list = source.ToList();
        if (list.Count < 2) return 0;
        
        double avg = list.Average();
        double variance = list.Average(x => Math.Pow(x - avg, 2));
        return Math.Sqrt(variance);
    }
}
```

---

## 4. Extension Methods สำหรับ types ต่างๆ

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Threading.Tasks;

// DateTime extensions
public static class DateTimeExtensions
{
    public static bool IsWeekend(this DateTime date) =>
        date.DayOfWeek == DayOfWeek.Saturday || date.DayOfWeek == DayOfWeek.Sunday;
    
    public static bool IsWeekday(this DateTime date) => !date.IsWeekend();
    
    public static bool IsToday(this DateTime date) =>
        date.Date == DateTime.Today;
    
    public static bool IsPast(this DateTime date) => date < DateTime.Now;
    public static bool IsFuture(this DateTime date) => date > DateTime.Now;
    
    public static string ToRelativeString(this DateTime date)
    {
        var timeSpan = DateTime.Now - date;
        
        if (timeSpan.TotalSeconds < 60) return "เมื่อกี้";
        if (timeSpan.TotalMinutes < 60) return $"{(int)timeSpan.TotalMinutes} นาทีที่แล้ว";
        if (timeSpan.TotalHours < 24) return $"{(int)timeSpan.TotalHours} ชั่วโมงที่แล้ว";
        if (timeSpan.TotalDays < 7) return $"{(int)timeSpan.TotalDays} วันที่แล้ว";
        if (timeSpan.TotalDays < 30) return $"{(int)(timeSpan.TotalDays / 7)} สัปดาห์ที่แล้ว";
        if (timeSpan.TotalDays < 365) return $"{(int)(timeSpan.TotalDays / 30)} เดือนที่แล้ว";
        return $"{(int)(timeSpan.TotalDays / 365)} ปีที่แล้ว";
    }
    
    public static DateTime StartOfDay(this DateTime date) =>
        date.Date;
    
    public static DateTime EndOfDay(this DateTime date) =>
        date.Date.AddDays(1).AddTicks(-1);
    
    public static DateTime StartOfMonth(this DateTime date) =>
        new DateTime(date.Year, date.Month, 1);
    
    public static DateTime EndOfMonth(this DateTime date) =>
        date.StartOfMonth().AddMonths(1).AddDays(-1).EndOfDay();
    
    public static IEnumerable<DateTime> EachDay(this (DateTime start, DateTime end) range)
    {
        for (var date = range.start.Date; date <= range.end.Date; date = date.AddDays(1))
            yield return date;
    }
}

// Dictionary extensions
public static class DictionaryExtensions
{
    public static TValue GetOrDefault<TKey, TValue>(
        this Dictionary<TKey, TValue> dict,
        TKey key,
        TValue defaultValue = default!) where TKey : notnull =>
        dict.TryGetValue(key, out TValue? value) ? value : defaultValue;
    
    public static TValue GetOrAdd<TKey, TValue>(
        this Dictionary<TKey, TValue> dict,
        TKey key,
        Func<TKey, TValue> factory) where TKey : notnull
    {
        if (!dict.TryGetValue(key, out TValue? value))
        {
            value = factory(key);
            dict[key] = value;
        }
        return value;
    }
    
    public static void AddRange<TKey, TValue>(
        this Dictionary<TKey, TValue> dict,
        IEnumerable<KeyValuePair<TKey, TValue>> items) where TKey : notnull
    {
        foreach (var item in items)
            dict[item.Key] = item.Value;
    }
    
    public static Dictionary<TValue, TKey> Invert<TKey, TValue>(
        this Dictionary<TKey, TValue> dict) where TKey : notnull where TValue : notnull =>
        dict.ToDictionary(kv => kv.Value, kv => kv.Key);
}

// Object extensions
public static class ObjectExtensions
{
    // Deep clone via JSON
    public static T? DeepClone<T>(this T obj)
    {
        var json = JsonSerializer.Serialize(obj);
        return JsonSerializer.Deserialize<T>(json);
    }
    
    // Null check
    public static bool IsNull<T>(this T? obj) where T : class => obj is null;
    public static bool IsNotNull<T>(this T? obj) where T : class => obj is not null;
    
    // Apply (Tap pattern)
    public static T Apply<T>(this T obj, Action<T> action)
    {
        action(obj);
        return obj;
    }
    
    // Transform
    public static TResult Transform<T, TResult>(this T obj, Func<T, TResult> transform) =>
        transform(obj);
    
    // To JSON
    public static string ToJson(this object obj, bool indented = false) =>
        JsonSerializer.Serialize(obj, new JsonSerializerOptions { WriteIndented = indented });
}

// Task extensions
public static class TaskExtensions
{
    // Timeout
    public static async Task<T> WithTimeout<T>(this Task<T> task, TimeSpan timeout)
    {
        using var cts = new CancellationTokenSource(timeout);
        var timeoutTask = Task.Delay(timeout, cts.Token);
        
        if (await Task.WhenAny(task, timeoutTask) == timeoutTask)
            throw new TimeoutException($"Task timed out after {timeout.TotalSeconds:F1}s");
        
        cts.Cancel();
        return await task;
    }
    
    // Ignore result (fire and forget)
    public static void FireAndForget(this Task task, Action<Exception>? onError = null)
    {
        task.ContinueWith(t =>
        {
            if (t.IsFaulted && t.Exception != null)
                onError?.Invoke(t.Exception.InnerException ?? t.Exception);
        }, TaskContinuationOptions.OnlyOnFaulted);
    }
}
```

---

## 5. Best Practices

```csharp
using System;
using System.Collections.Generic;

// ✅ Best Practice 1: ใส่ extensions ใน namespace ที่เหมาะสม
namespace MyApp.Extensions.String
{
    public static class StringExtensions { }
}

namespace MyApp.Extensions.Collections
{
    public static class CollectionExtensions { }
}

// ✅ Best Practice 2: ตรวจสอบ null สำหรับ value types
public static class GoodNullHandling
{
    // ✅ ถูกต้อง - รับ null ได้
    public static bool IsNullOrEmpty(this string? value) =>
        string.IsNullOrEmpty(value);
    
    // ✅ ใส่ ArgumentNullException สำหรับ required parameters
    public static string[] Split(this string value, string separator)
    {
        ArgumentNullException.ThrowIfNull(value);
        ArgumentNullException.ThrowIfNull(separator);
        return value.Split(separator);
    }
}

// ✅ Best Practice 3: ไม่ใช้ extension methods แทน member methods
public class MyClass
{
    // ✅ ถ้าเป็นของ class นี้ ใส่เป็น member
    public bool IsValid() => true;
}

// ❌ ไม่ควรทำ
public static class BadExtensions
{
    // ❌ ไม่ควรใช้ extension สำหรับ functionality ที่ควรเป็น member
    public static bool IsValid(this MyClass obj) => true;
}

// ✅ Best Practice 4: Extension methods ที่เหมาะสม
public static class AppropriateExtensions
{
    // ✅ เพิ่ม utility ให้ third-party types
    public static bool HasItems<T>(this List<T> list) => list.Count > 0;
    
    // ✅ เพิ่ม domain-specific methods
    public static bool IsAdult(this DateTime birthDate) =>
        (DateTime.Today - birthDate).TotalDays >= 18 * 365.25;
    
    // ✅ Fluent API
    public static StringBuilder AppendIf(
        this StringBuilder sb,
        bool condition,
        string value)
    {
        if (condition) sb.Append(value);
        return sb;
    }
}

// ✅ Best Practice 5: ตั้งชื่อให้ชัดเจน
public static class NamingConventions
{
    // ✅ ชื่อบอกว่าทำอะไร
    public static bool ContainsOnlyDigits(this string value) =>
        value.All(char.IsDigit);
    
    public static string RemoveHtmlTags(this string html) =>
        System.Text.RegularExpressions.Regex.Replace(html, @"<[^>]*>", string.Empty);
    
    // ✅ Consistent naming กับ BCL
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) where T : class =>
        source.Where(x => x is not null)!;
}

// ❌ Bad Practices
public static class BadPractices
{
    // ❌ ชื่อไม่ชัดเจน
    public static string Process(this string value) => value.Trim();
    
    // ❌ Side effects ที่ไม่คาดคิด
    public static string LogAndReturn(this string value)
    {
        Console.WriteLine(value); // Side effect ที่ไม่ดี
        return value;
    }
    
    // ❌ Extension บน object (กว้างเกินไป)
    public static string Serialize(this object obj) => obj.ToString() ?? "";
}
```

---

## 6. โปรแกรมตัวอย่าง: String/Number Utility Extensions

```csharp
using System;
using System.Collections.Generic;
using System.Globalization;
using System.Linq;
using System.Text;
using System.Text.RegularExpressions;

// ===== String Utilities =====

public static class StringUtilities
{
    // Slugify - แปลง title เป็น URL-friendly
    public static string Slugify(this string value)
    {
        if (string.IsNullOrWhiteSpace(value)) return string.Empty;
        
        // Normalize unicode
        value = value.Normalize(NormalizationForm.FormD);
        
        var sb = new StringBuilder();
        foreach (char c in value)
        {
            var category = CharUnicodeInfo.GetUnicodeCategory(c);
            if (category != UnicodeCategory.NonSpacingMark)
                sb.Append(c);
        }
        
        value = sb.ToString().Normalize(NormalizationForm.FormC).ToLower();
        value = Regex.Replace(value, @"[^a-z0-9\s-]", "");
        value = Regex.Replace(value, @"[\s-]+", "-");
        return value.Trim('-');
    }
    
    // Word wrap
    public static string WordWrap(this string text, int maxWidth)
    {
        if (string.IsNullOrEmpty(text)) return text ?? string.Empty;
        if (text.Length <= maxWidth) return text;
        
        var words = text.Split(' ');
        var lines = new List<string>();
        var currentLine = new StringBuilder();
        
        foreach (var word in words)
        {
            if (currentLine.Length > 0 && currentLine.Length + word.Length + 1 > maxWidth)
            {
                lines.Add(currentLine.ToString().TrimEnd());
                currentLine.Clear();
            }
            
            currentLine.Append(word).Append(' ');
        }
        
        if (currentLine.Length > 0)
            lines.Add(currentLine.ToString().TrimEnd());
        
        return string.Join(Environment.NewLine, lines);
    }
    
    // Extract numbers
    public static IEnumerable<int> ExtractNumbers(this string text) =>
        Regex.Matches(text, @"\d+")
             .Select(m => int.Parse(m.Value));
    
    // Highlight substring
    public static string Highlight(this string text, string term, string prefix = "[", string suffix = "]")
    {
        if (string.IsNullOrEmpty(term)) return text;
        return Regex.Replace(text, Regex.Escape(term),
            m => $"{prefix}{m.Value}{suffix}", RegexOptions.IgnoreCase);
    }
    
    // Soundex (เปรียบเทียบเสียงคล้าย)
    public static string Soundex(this string value)
    {
        if (string.IsNullOrEmpty(value)) return string.Empty;
        
        value = value.ToUpper();
        char firstLetter = value[0];
        
        var codes = new Dictionary<char, char>
        {
            {'B','2'}, {'F','2'}, {'P','2'}, {'V','2'},
            {'C','3'}, {'G','3'}, {'J','3'}, {'K','3'}, {'Q','3'}, {'S','3'}, {'X','3'}, {'Z','3'},
            {'D','3'}, {'T','3'},
            {'L','4'},
            {'M','5'}, {'N','5'},
            {'R','6'}
        };
        
        var sb = new StringBuilder(firstLetter.ToString());
        char lastCode = codes.GetValueOrDefault(firstLetter, '0');
        
        foreach (char c in value.Skip(1))
        {
            if (codes.TryGetValue(c, out char code) && code != lastCode)
            {
                sb.Append(code);
                lastCode = code;
                if (sb.Length == 4) break;
            }
            else if (!codes.ContainsKey(c))
            {
                lastCode = '0';
            }
        }
        
        return sb.ToString().PadRight(4, '0');
    }
}

// ===== Number Utilities =====

public static class NumberUtilities
{
    // Thai Baht to text
    public static string ToThaiText(this decimal amount)
    {
        if (amount == 0) return "ศูนย์บาทถ้วน";
        
        string[] units = { "", "หนึ่ง", "สอง", "สาม", "สี่", "ห้า", "หก", "เจ็ด", "แปด", "เก้า" };
        string[] places = { "", "สิบ", "ร้อย", "พัน", "หมื่น", "แสน", "ล้าน" };
        
        bool isNegative = amount < 0;
        amount = Math.Abs(amount);
        
        long baht = (long)amount;
        int satang = (int)((amount - baht) * 100);
        
        string result = ConvertToThaiText(baht) + "บาท";
        
        if (satang > 0)
            result += ConvertToThaiText(satang) + "สตางค์";
        else
            result += "ถ้วน";
        
        return (isNegative ? "ลบ" : "") + result;
    }
    
    private static string ConvertToThaiText(long number)
    {
        if (number == 0) return "ศูนย์";
        
        string[] units = { "", "หนึ่ง", "สอง", "สาม", "สี่", "ห้า", "หก", "เจ็ด", "แปด", "เก้า" };
        string[] places = { "", "สิบ", "ร้อย", "พัน", "หมื่น", "แสน" };
        
        if (number >= 1000000)
        {
            return ConvertToThaiText(number / 1000000) + "ล้าน" +
                   (number % 1000000 > 0 ? ConvertToThaiText(number % 1000000) : "");
        }
        
        var digits = number.ToString();
        var result = new StringBuilder();
        
        for (int i = 0; i < digits.Length; i++)
        {
            int digit = digits[i] - '0';
            int place = digits.Length - i - 1;
            
            if (digit == 0) continue;
            
            if (place == 1 && digit == 2) result.Append("ยี่");
            else if (place == 1 && digit == 1) result.Append("");
            else result.Append(units[digit]);
            
            result.Append(places[place]);
        }
        
        return result.ToString();
    }
    
    // Fibonacci
    public static IEnumerable<long> FibonacciSequence(this int count)
    {
        if (count <= 0) yield break;
        
        long a = 0, b = 1;
        for (int i = 0; i < count; i++)
        {
            yield return a;
            (a, b) = (b, a + b);
        }
    }
    
    // Lerp (Linear interpolation)
    public static double Lerp(this double start, double end, double t) =>
        start + (end - start) * Math.Clamp(t, 0, 1);
    
    // Map range
    public static double MapRange(this double value, double fromMin, double fromMax, double toMin, double toMax)
    {
        double normalized = (value - fromMin) / (fromMax - fromMin);
        return toMin + normalized * (toMax - toMin);
    }
    
    // Round to nearest multiple
    public static double RoundToNearest(this double value, double multiple) =>
        Math.Round(value / multiple) * multiple;
}

// ===== Demo Program =====
class Program
{
    static void Main()
    {
        Console.WriteLine("===== String Extensions Demo =====\n");
        
        // Truncate
        string longText = "สวัสดีครับ นี่คือข้อความยาวๆ ที่ต้องการตัดให้สั้นลง";
        Console.WriteLine($"Original: {longText}");
        Console.WriteLine($"Truncate(20): {longText.Truncate(20)}");
        
        // Case conversion
        string camel = "helloWorldFromCSharp";
        Console.WriteLine($"\nCamelCase: {camel}");
        Console.WriteLine($"PascalCase: {camel.ToPascalCase()}");
        Console.WriteLine($"SnakeCase: {camel.ToSnakeCase()}");
        Console.WriteLine($"KebabCase: {camel.ToKebabCase()}");
        
        // Validation
        string email = "user@example.com";
        Console.WriteLine($"\nEmail '{email}' valid: {email.IsValidEmail()}");
        Console.WriteLine($"Masked: {email.MaskEmail()}");
        
        // Slug
        string title = "สวัสดี Hello World!";
        Console.WriteLine($"\nSlugify: {title.Slugify()}");
        
        // Word wrap
        string text = "The quick brown fox jumps over the lazy dog and then runs away quickly";
        Console.WriteLine($"\nWord wrap (20 chars):");
        Console.WriteLine(text.WordWrap(20));
        
        Console.WriteLine("\n===== Number Extensions Demo =====\n");
        
        // Range
        Console.Write("1.To(5): ");
        foreach (int n in 1.To(5))
            Console.Write($"{n} ");
        Console.WriteLine();
        
        // Times
        Console.Write("3.Times: ");
        3.Times(() => Console.Write("Hi "));
        Console.WriteLine();
        
        // File size
        long bytes = 1_500_000;
        Console.WriteLine($"\n{bytes} bytes = {bytes.ToFileSizeString()}");
        
        // Thai Baht
        decimal amount = 1234.56m;
        Console.WriteLine($"\n{amount} บาท = {amount.ToThaiText()}");
        
        // Fibonacci
        Console.Write("\nFibonacci(10): ");
        foreach (long fib in 10.FibonacciSequence())
            Console.Write($"{fib} ");
        Console.WriteLine();
        
        // Statistics
        var numbers = new double[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
        Console.WriteLine($"\nData: {string.Join(", ", numbers)}");
        Console.WriteLine($"Median: {numbers.Median()}");
        Console.WriteLine($"Std Dev: {numbers.StandardDeviation():F4}");
        
        // Collection extensions
        Console.WriteLine("\n===== Collection Extensions Demo =====\n");
        
        var items = Enumerable.Range(1, 20).ToList();
        
        // Chunk
        Console.WriteLine("Chunks of 5:");
        foreach (var chunk in items.Chunk(5))
        {
            Console.WriteLine($"  [{string.Join(", ", chunk)}]");
        }
        
        // Partition
        var (evens, odds) = items.Partition(n => n % 2 == 0);
        Console.WriteLine($"\nEvens: {string.Join(", ", evens.Take(5))}...");
        Console.WriteLine($"Odds: {string.Join(", ", odds.Take(5))}...");
        
        // WithIndex
        Console.WriteLine("\nWith index:");
        foreach (var (item, index) in items.Take(3).WithIndex())
        {
            Console.WriteLine($"  [{index}] = {item}");
        }
        
        // DateTimeExtensions
        Console.WriteLine("\n===== DateTime Extensions Demo =====\n");
        
        var now = DateTime.Now;
        var past = DateTime.Now.AddDays(-3);
        var birthday = new DateTime(1990, 1, 1);
        
        Console.WriteLine($"วันนี้ใช่ไหม: {now.IsToday()}");
        Console.WriteLine($"วันหยุดไหม: {now.IsWeekend()}");
        Console.WriteLine($"3 วันที่แล้ว: {past.ToRelativeString()}");
        Console.WriteLine($"1990-01-01 เป็นผู้ใหญ่ไหม: {birthday.IsAdult()}");
        Console.WriteLine($"ต้นเดือน: {now.StartOfMonth():dd/MM/yyyy}");
        Console.WriteLine($"สิ้นเดือน: {now.EndOfMonth():dd/MM/yyyy}");
    }
}
```

---

## Exercises

### Exercise 1: HTML Builder Extension
```csharp
// TODO: สร้าง extension methods สำหรับ StringBuilder ที่ช่วยสร้าง HTML
public static class HtmlBuilderExtensions
{
    // ใช้งานแบบ fluent:
    // new StringBuilder()
    //   .OpenTag("div", "class='container'")
    //   .AppendText("Hello World")
    //   .CloseTag("div")
    //   .ToString()
    
    public static StringBuilder OpenTag(this StringBuilder sb, string tag, string? attributes = null)
    {
        throw new NotImplementedException();
    }
    
    public static StringBuilder CloseTag(this StringBuilder sb, string tag)
    {
        throw new NotImplementedException();
    }
    
    public static StringBuilder AppendText(this StringBuilder sb, string text)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 2: Validation Extensions
```csharp
// TODO: สร้าง validation extension methods
public static class ValidationExtensions
{
    // IsValidThaiId - ตรวจสอบเลขบัตรประชาชนไทย (13 หลัก)
    public static bool IsValidThaiId(this string id)
    {
        throw new NotImplementedException();
    }
    
    // IsValidPhoneNumber - ตรวจสอบเบอร์โทรไทย
    public static bool IsValidThaiPhone(this string phone)
    {
        throw new NotImplementedException();
    }
    
    // IsValidCreditCard - Luhn algorithm
    public static bool IsValidCreditCard(this string cardNumber)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: LINQ Extensions
```csharp
// TODO: Implement LINQ-style extensions
public static class AdvancedLinqExtensions
{
    // RunningTotal - คำนวณ running sum
    // [1,2,3,4] -> [1,3,6,10]
    public static IEnumerable<TResult> RunningAggregate<T, TResult>(
        this IEnumerable<T> source,
        TResult seed,
        Func<TResult, T, TResult> accumulator)
    {
        throw new NotImplementedException();
    }
    
    // Window - sliding window
    // [1,2,3,4,5] window(3) -> [[1,2,3],[2,3,4],[3,4,5]]
    public static IEnumerable<IEnumerable<T>> Window<T>(
        this IEnumerable<T> source, int size)
    {
        throw new NotImplementedException();
    }
    
    // ZipAll - zip ทุก elements (ไม่ตัด)
    public static IEnumerable<(T1? First, T2? Second)> ZipAll<T1, T2>(
        this IEnumerable<T1> first,
        IEnumerable<T2> second)
    {
        throw new NotImplementedException();
    }
}
```

---

## สรุป

✅ **Extension Methods** เพิ่มความสามารถให้ types ที่มีอยู่โดยไม่ต้องแก้ source code

✅ ต้องอยู่ใน **static class** และ parameter แรกมี **this** keyword

✅ ใช้สำหรับ **utility methods**, **fluent APIs**, และ **domain-specific operations**

✅ ควรตรวจสอบ **null** สำหรับ nullable parameters

✅ ใส่ใน **namespace** ที่เหมาะสมเพื่อควบคุม scope

✅ **หลีกเลี่ยง** extension บน types กว้างๆ เช่น object

✅ Extension methods ที่ดีควร **immutable** (ไม่แก้ไข state ของ source)

✅ ใช้ **method chaining** เพื่อให้โค้ด readable

---

## Part ถัดไป

➡️ **Part 034**: Attributes - Built-in attributes, Custom attributes, Validation attributes และการอ่านด้วย Reflection

---

*Part 033/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
