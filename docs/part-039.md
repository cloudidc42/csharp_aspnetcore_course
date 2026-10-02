# Part 039: Span\<T\> และ Memory\<T\>

## เนื้อหาใน Part นี้
- Stack-based memory
- Span\<T\> operations
- ReadOnlySpan\<string\> vs string
- Memory\<T\> สำหรับ async
- ArrayPool\<T\>
- Performance comparison
- โปรแกรมตัวอย่าง: High-performance string parsing

---

## 1. Stack-based Memory

### ทำไม Span\<T\> ถึงสำคัญ

```csharp
using System;
using System.Diagnostics;
using System.Text;

class MemoryOverviewExample
{
    static void Main()
    {
        // ปัญหาเดิม: String operations สร้าง allocations มาก
        string data = "2024-01-15,สมชาย,25,Bangkok";
        
        // แต่ละ Substring() allocates new string object
        string[] parts = data.Split(',');        // 1 array + 4 strings
        string date = data.Substring(0, 10);     // 1 new string
        
        // ด้วย Span<T>: ไม่ allocate เลย!
        ReadOnlySpan<char> span = data.AsSpan();
        ReadOnlySpan<char> dateSpan = span[..10]; // ไม่ allocate!
        
        Console.WriteLine($"Date from string: '{date}'");
        Console.WriteLine($"Date from span: '{dateSpan}'"); // ToString() ทำแค่ตอน print
        
        // stackalloc - allocate บน stack แทน heap
        Span<int> numbers = stackalloc int[10];
        for (int i = 0; i < numbers.Length; i++)
            numbers[i] = i * i;
        
        foreach (int n in numbers)
            Console.Write($"{n} ");
        Console.WriteLine();
        
        // Span ห่อ array
        int[] array = { 1, 2, 3, 4, 5 };
        Span<int> arraySpan = array.AsSpan();
        arraySpan[2] = 99; // แก้ไข original array!
        Console.WriteLine(string.Join(",", array)); // 1,2,99,4,5
        
        // Span ห่อ array portion
        Span<int> slice = array.AsSpan(1, 3); // elements index 1-3
        Console.WriteLine($"Slice: [{string.Join(",", slice.ToArray())}]");
    }
}
```

---

## 2. Span\<T\> Operations

```csharp
using System;
using System.Runtime.InteropServices;

class SpanOperationsExample
{
    static void Main()
    {
        // สร้าง Span
        byte[] bytes = new byte[10];
        Span<byte> span1 = bytes;              // จาก array
        Span<byte> span2 = bytes.AsSpan();     // explicit conversion
        Span<byte> span3 = new Span<byte>(bytes, 2, 5); // portion
        Span<byte> span4 = stackalloc byte[10]; // stack allocation
        
        // Slice
        Span<int> numbers = stackalloc int[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
        Span<int> first5 = numbers[..5];         // [1,2,3,4,5]
        Span<int> last5 = numbers[5..];          // [6,7,8,9,10]
        Span<int> middle = numbers[2..7];        // [3,4,5,6,7]
        
        Console.WriteLine($"First5: [{string.Join(",", first5.ToArray())}]");
        Console.WriteLine($"Last5: [{string.Join(",", last5.ToArray())}]");
        Console.WriteLine($"Middle: [{string.Join(",", middle.ToArray())}]");
        
        // Fill
        numbers.Fill(0);
        Console.WriteLine($"Filled: [{string.Join(",", numbers.ToArray())}]");
        
        // Copy
        Span<int> source = stackalloc int[] { 1, 2, 3, 4, 5 };
        Span<int> dest = stackalloc int[5];
        source.CopyTo(dest);
        Console.WriteLine($"Copied: [{string.Join(",", dest.ToArray())}]");
        
        // Clear
        Span<int> toClear = stackalloc int[] { 9, 8, 7, 6, 5 };
        toClear.Clear();
        Console.WriteLine($"Cleared: [{string.Join(",", toClear.ToArray())}]");
        
        // Reverse
        Span<int> toReverse = new int[] { 1, 2, 3, 4, 5 };
        toReverse.Reverse();
        Console.WriteLine($"Reversed: [{string.Join(",", toReverse.ToArray())}]");
        
        // MemoryMarshal - reinterpret memory
        Span<int> ints = stackalloc int[] { 1, 2, 3 };
        Span<byte> reinterpreted = MemoryMarshal.AsBytes(ints);
        Console.WriteLine($"3 ints as bytes: {reinterpreted.Length} bytes");
        
        // Span ของ struct
        Span<DateTime> dates = stackalloc DateTime[3];
        dates[0] = DateTime.Today;
        dates[1] = DateTime.Today.AddDays(1);
        dates[2] = DateTime.Today.AddDays(2);
        
        foreach (var date in dates)
            Console.WriteLine($"  {date:yyyy-MM-dd}");
    }
}
```

---

## 3. ReadOnlySpan\<char\> vs string

```csharp
using System;
using System.Diagnostics;

class SpanVsStringExample
{
    // ❌ แบบเดิม: สร้าง string objects มาก
    static string OldExtractPart(string input, int start, int length)
    {
        return input.Substring(start, length); // allocates!
    }
    
    // ✅ ด้วย Span: ไม่ allocate
    static ReadOnlySpan<char> NewExtractPart(ReadOnlySpan<char> input, int start, int length)
    {
        return input.Slice(start, length); // no allocation!
    }
    
    // String parsing แบบเดิม
    static (int year, int month, int day) ParseDateOld(string date)
    {
        return (
            int.Parse(date.Substring(0, 4)),
            int.Parse(date.Substring(5, 2)),
            int.Parse(date.Substring(8, 2))
        );
    }
    
    // String parsing ด้วย Span
    static (int year, int month, int day) ParseDateNew(ReadOnlySpan<char> date)
    {
        return (
            int.Parse(date[..4]),         // ไม่สร้าง string ใหม่!
            int.Parse(date.Slice(5, 2)),
            int.Parse(date.Slice(8, 2))
        );
    }
    
    // Split ด้วย Span
    static void ParseCSVLineWithSpan(ReadOnlySpan<char> line)
    {
        int start = 0;
        int fieldIndex = 0;
        
        for (int i = 0; i <= line.Length; i++)
        {
            if (i == line.Length || line[i] == ',')
            {
                ReadOnlySpan<char> field = line.Slice(start, i - start);
                Console.WriteLine($"  Field {fieldIndex}: '{field}'");
                start = i + 1;
                fieldIndex++;
            }
        }
    }
    
    static void Main()
    {
        // ParseDate comparison
        string dateStr = "2024-03-15";
        var (y1, m1, d1) = ParseDateOld(dateStr);
        var (y2, m2, d2) = ParseDateNew(dateStr.AsSpan());
        
        Console.WriteLine($"Old: {y1}-{m1:D2}-{d1:D2}");
        Console.WriteLine($"New: {y2}-{m2:D2}-{d2:D2}");
        
        // CSV parsing
        Console.WriteLine("\nCSV parsing:");
        ParseCSVLineWithSpan("สมชาย,25,Bangkok,Thailand".AsSpan());
        
        // Span operations บน string
        ReadOnlySpan<char> text = "Hello, World! How are you?".AsSpan();
        
        // Contains - ไม่ allocate
        bool hasWorld = text.Contains("World".AsSpan(), StringComparison.OrdinalIgnoreCase);
        Console.WriteLine($"\nContains 'World': {hasWorld}");
        
        // StartsWith/EndsWith
        Console.WriteLine($"StartsWith 'Hello': {text.StartsWith("Hello".AsSpan())}");
        Console.WriteLine($"EndsWith '?': {text.EndsWith("?".AsSpan())}");
        
        // IndexOf
        int idx = text.IndexOf(',');
        Console.WriteLine($"Index of ',': {idx}");
        
        // Trim
        ReadOnlySpan<char> padded = "  trimmed  ".AsSpan();
        Console.WriteLine($"Trimmed: '{padded.Trim()}'");
        
        // Performance demo
        Console.WriteLine("\n--- Performance ---");
        const int iterations = 1_000_000;
        string longDate = "2024-03-15T10:30:00";
        
        var sw = Stopwatch.StartNew();
        for (int i = 0; i < iterations; i++)
        {
            var _ = ParseDateOld(longDate);
        }
        Console.WriteLine($"Substring (old): {sw.ElapsedMilliseconds}ms");
        
        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            var _ = ParseDateNew(longDate.AsSpan());
        }
        Console.WriteLine($"Span (new): {sw.ElapsedMilliseconds}ms");
    }
}
```

---

## 4. Memory\<T\> สำหรับ async

```csharp
using System;
using System.IO;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

class MemoryTExample
{
    // Memory<T> ใช้กับ async (Span ใช้ไม่ได้กับ async!)
    
    // ❌ ไม่ได้ - Span ไม่ support async
    // static async Task BadMethodAsync(Span<byte> buffer) { ... }
    
    // ✅ ถูกต้อง - ใช้ Memory<T>
    static async Task ReadDataAsync(Memory<byte> buffer, CancellationToken ct = default)
    {
        // จำลอง async read
        await Task.Delay(10, ct);
        
        // Fill buffer with data
        for (int i = 0; i < buffer.Length; i++)
            buffer.Span[i] = (byte)(i % 256);
    }
    
    // Read file ด้วย Memory<T>
    static async Task<int> ReadFileAsync(
        string path,
        Memory<byte> buffer,
        CancellationToken ct = default)
    {
        using var fs = new FileStream(path, FileMode.Open, FileAccess.Read,
            FileShare.Read, bufferSize: 4096, useAsync: true);
        
        return await fs.ReadAsync(buffer, ct);
    }
    
    // Network operations
    static async Task<string> ProcessNetworkDataAsync(
        Stream stream,
        int bufferSize = 4096,
        CancellationToken ct = default)
    {
        using var memOwner = System.Buffers.MemoryPool<byte>.Shared.Rent(bufferSize);
        Memory<byte> buffer = memOwner.Memory[..bufferSize];
        
        var result = new StringBuilder();
        int bytesRead;
        
        while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
        {
            // Process ข้อมูลจาก buffer
            var text = Encoding.UTF8.GetString(buffer.Span[..bytesRead]);
            result.Append(text);
        }
        
        return result.ToString();
    }
    
    static async Task Main()
    {
        // สร้าง buffer
        byte[] rawBuffer = new byte[1024];
        Memory<byte> buffer = rawBuffer;
        
        // Slice Memory
        Memory<byte> firstHalf = buffer[..512];
        Memory<byte> secondHalf = buffer[512..];
        
        // ส่งไปยัง async method
        await ReadDataAsync(firstHalf);
        Console.WriteLine($"First byte: {firstHalf.Span[0]}");
        Console.WriteLine($"Buffer size: {buffer.Length}");
        
        // Memory conversion
        ReadOnlyMemory<byte> readOnly = buffer;
        Span<byte> span = buffer.Span; // ใช้ใน sync context
        
        // MemoryMarshal
        Memory<int> intMemory = System.Runtime.InteropServices.MemoryMarshal
            .Cast<byte, int>(buffer);
        Console.WriteLine($"Int memory length: {intMemory.Length}");
    }
}
```

---

## 5. ArrayPool\<T\>

```csharp
using System;
using System.Buffers;
using System.Diagnostics;
using System.Text;
using System.Threading.Tasks;

class ArrayPoolExample
{
    // ปัญหาเดิม: allocate array ทุกครั้ง = GC pressure
    static byte[] BadAllocate(int size)
    {
        return new byte[size]; // ทุก call สร้าง array ใหม่
    }
    
    // ArrayPool: reuse arrays จาก pool
    static void GoodProcess(int size)
    {
        byte[] buffer = ArrayPool<byte>.Shared.Rent(size); // อาจได้ array ที่ใหญ่กว่า
        try
        {
            // ใช้ buffer
            ProcessBuffer(buffer.AsSpan(0, size));
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer); // คืน pool
        }
    }
    
    // Pattern ดีกว่า: ด้วย IMemoryOwner
    static async Task BetterProcessAsync(int size)
    {
        using var owner = MemoryPool<byte>.Shared.Rent(size);
        Memory<byte> buffer = owner.Memory[..size];
        
        await ProcessBufferAsync(buffer);
        // using block คืน memory อัตโนมัติ
    }
    
    static void ProcessBuffer(Span<byte> data)
    {
        for (int i = 0; i < data.Length; i++)
            data[i] = (byte)(i & 0xFF);
    }
    
    static async Task ProcessBufferAsync(Memory<byte> data)
    {
        await Task.Delay(1);
        for (int i = 0; i < data.Length; i++)
            data.Span[i] = (byte)(i & 0xFF);
    }
    
    // Custom pool
    class RecyclableBuffer : IDisposable
    {
        private readonly byte[] _buffer;
        private readonly ArrayPool<byte> _pool;
        private bool _disposed;
        
        public RecyclableBuffer(int size, ArrayPool<byte>? pool = null)
        {
            _pool = pool ?? ArrayPool<byte>.Shared;
            _buffer = _pool.Rent(size);
            Size = size;
        }
        
        public int Size { get; }
        public Span<byte> Span => _buffer.AsSpan(0, Size);
        public Memory<byte> Memory => _buffer.AsMemory(0, Size);
        
        public void Dispose()
        {
            if (!_disposed)
            {
                _pool.Return(_buffer, clearArray: true);
                _disposed = true;
            }
        }
    }
    
    static void Main()
    {
        const int iterations = 1_000_000;
        const int bufferSize = 4096;
        
        // แบบเดิม: allocate ทุกครั้ง
        var sw = Stopwatch.StartNew();
        for (int i = 0; i < iterations; i++)
        {
            var buf = new byte[bufferSize];
            ProcessBuffer(buf);
        }
        Console.WriteLine($"New array each time: {sw.ElapsedMilliseconds}ms");
        
        // ArrayPool
        sw.Restart();
        for (int i = 0; i < iterations; i++)
        {
            GoodProcess(bufferSize);
        }
        Console.WriteLine($"ArrayPool: {sw.ElapsedMilliseconds}ms");
        
        // RecyclableBuffer
        using (var buffer = new RecyclableBuffer(bufferSize))
        {
            ProcessBuffer(buffer.Span);
            Console.WriteLine($"\nBuffer first byte: {buffer.Span[0]}");
        }
        
        // ArrayPool สำหรับ string building
        Console.WriteLine("\n--- StringBuilder with ArrayPool ---");
        var result = BuildStringWithPool("Hello World! This is a test.");
        Console.WriteLine($"Result: {result}");
    }
    
    static string BuildStringWithPool(string input)
    {
        var pool = ArrayPool<char>.Shared;
        char[] buffer = pool.Rent(input.Length * 2);
        
        try
        {
            int written = 0;
            foreach (char c in input)
            {
                buffer[written++] = char.ToUpper(c);
                if (c == ' ')
                    buffer[written++] = '_';
            }
            
            return new string(buffer, 0, written);
        }
        finally
        {
            pool.Return(buffer);
        }
    }
}
```

---

## 6. Performance Comparison

```csharp
using System;
using System.Buffers;
using System.Diagnostics;
using System.Text;

class PerformanceComparison
{
    const int Iterations = 500_000;
    
    static void Main()
    {
        string csvData = GenerateCsvData(100); // 100 rows
        
        Console.WriteLine("=== Parsing Performance ===");
        Console.WriteLine($"CSV size: {csvData.Length:N0} chars\n");
        
        // Test 1: String.Split
        MeasureGC("String.Split", () =>
        {
            for (int i = 0; i < Iterations; i++)
                ParseWithStringSplit(csvData);
        });
        
        // Test 2: Span
        MeasureGC("Span Parsing", () =>
        {
            for (int i = 0; i < Iterations; i++)
                ParseWithSpan(csvData.AsSpan());
        });
        
        Console.WriteLine("\n=== Allocation Comparison ===");
        
        // Memory allocation test
        MeasureGC("Substring (allocation)", () =>
        {
            for (int i = 0; i < Iterations; i++)
                AllocateWithSubstring("Hello, World! Test String");
        });
        
        MeasureGC("Span (no allocation)", () =>
        {
            for (int i = 0; i < Iterations; i++)
                NoAllocateWithSpan("Hello, World! Test String".AsSpan());
        });
        
        Console.WriteLine("\n=== Buffer Operations ===");
        
        MeasureGC("new byte[] each time", () =>
        {
            for (int i = 0; i < 100_000; i++)
            {
                var buf = new byte[4096];
                buf[0] = 1;
            }
        });
        
        MeasureGC("ArrayPool<byte>", () =>
        {
            for (int i = 0; i < 100_000; i++)
            {
                var buf = ArrayPool<byte>.Shared.Rent(4096);
                try { buf[0] = 1; }
                finally { ArrayPool<byte>.Shared.Return(buf); }
            }
        });
    }
    
    static void MeasureGC(string name, Action action)
    {
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        
        long gen0Before = GC.CollectionCount(0);
        long gen1Before = GC.CollectionCount(1);
        long allocBefore = GC.GetTotalAllocatedBytes();
        
        var sw = Stopwatch.StartNew();
        action();
        sw.Stop();
        
        long gen0After = GC.CollectionCount(0);
        long allocAfter = GC.GetTotalAllocatedBytes();
        long allocated = allocAfter - allocBefore;
        
        Console.WriteLine($"{name,-30}: {sw.ElapsedMilliseconds,5}ms, " +
            $"GC0={gen0After - gen0Before,3}, " +
            $"Alloc={allocated / 1024,6}KB");
    }
    
    static int ParseWithStringSplit(string csv)
    {
        int count = 0;
        var rows = csv.Split('\n');
        foreach (var row in rows)
        {
            var fields = row.Split(',');
            count += fields.Length;
        }
        return count;
    }
    
    static int ParseWithSpan(ReadOnlySpan<char> csv)
    {
        int count = 0;
        int lineStart = 0;
        
        for (int i = 0; i <= csv.Length; i++)
        {
            if (i == csv.Length || csv[i] == '\n')
            {
                var line = csv[lineStart..i];
                int fieldStart = 0;
                for (int j = 0; j <= line.Length; j++)
                {
                    if (j == line.Length || line[j] == ',')
                    {
                        count++;
                        fieldStart = j + 1;
                    }
                }
                lineStart = i + 1;
            }
        }
        return count;
    }
    
    static string AllocateWithSubstring(string text)
    {
        return text.Substring(0, 5).ToUpper() + text.Substring(7, 5).ToLower();
    }
    
    static bool NoAllocateWithSpan(ReadOnlySpan<char> text)
    {
        var first5 = text[..5];
        var next5 = text.Slice(7, 5);
        return first5.Length > 0 && next5.Length > 0; // ไม่ allocate
    }
    
    static string GenerateCsvData(int rows)
    {
        var sb = new StringBuilder();
        var random = new Random(42);
        
        for (int i = 0; i < rows; i++)
        {
            sb.AppendLine($"{i},Name{i},{random.Next(18, 65)},City{i % 10}");
        }
        return sb.ToString();
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: High-performance String Parsing

```csharp
using System;
using System.Buffers;
using System.Collections.Generic;
using System.Diagnostics;
using System.Text;

// High-performance CSV parser

public readonly ref struct CsvField
{
    private readonly ReadOnlySpan<char> _data;
    
    public CsvField(ReadOnlySpan<char> data) { _data = data; }
    
    public ReadOnlySpan<char> Raw => _data;
    public int Length => _data.Length;
    public bool IsEmpty => _data.IsEmpty;
    
    public string ToString() => _data.ToString();
    public int ToInt() => int.Parse(_data);
    public double ToDouble() => double.Parse(_data);
    public bool ToBool() => bool.Parse(_data);
    public DateTime ToDate() => DateTime.Parse(_data);
    
    public bool Equals(string other) =>
        _data.Equals(other.AsSpan(), StringComparison.Ordinal);
    
    public bool EqualsCaseInsensitive(string other) =>
        _data.Equals(other.AsSpan(), StringComparison.OrdinalIgnoreCase);
}

// Zero-allocation CSV parser
public ref struct CsvRowReader
{
    private ReadOnlySpan<char> _remaining;
    private readonly char _separator;
    
    public CsvRowReader(ReadOnlySpan<char> data, char separator = ',')
    {
        _remaining = data;
        _separator = separator;
    }
    
    public bool TryReadField(out CsvField field)
    {
        if (_remaining.IsEmpty)
        {
            field = default;
            return false;
        }
        
        // Handle quoted fields
        if (_remaining[0] == '"')
        {
            int endQuote = _remaining.Slice(1).IndexOf('"') + 1;
            if (endQuote > 0)
            {
                field = new CsvField(_remaining.Slice(1, endQuote - 1));
                _remaining = _remaining.Slice(endQuote + 1);
                if (!_remaining.IsEmpty && _remaining[0] == _separator)
                    _remaining = _remaining.Slice(1);
                return true;
            }
        }
        
        int sepIndex = _remaining.IndexOf(_separator);
        if (sepIndex == -1)
        {
            field = new CsvField(_remaining.TrimEnd('\r', '\n'));
            _remaining = ReadOnlySpan<char>.Empty;
        }
        else
        {
            field = new CsvField(_remaining[..sepIndex]);
            _remaining = _remaining.Slice(sepIndex + 1);
        }
        
        return true;
    }
    
    public ReadOnlySpan<char> Remaining => _remaining;
}

// HTTP-like header parser
public ref struct HttpHeaderParser
{
    private ReadOnlySpan<char> _data;
    
    public HttpHeaderParser(ReadOnlySpan<char> data) { _data = data; }
    
    public bool TryReadHeader(out ReadOnlySpan<char> name, out ReadOnlySpan<char> value)
    {
        name = default;
        value = default;
        
        if (_data.IsEmpty) return false;
        
        // หา end of line
        int lineEnd = _data.IndexOf('\n');
        ReadOnlySpan<char> line = lineEnd == -1 ? _data : _data[..lineEnd];
        line = line.TrimEnd('\r');
        
        if (line.IsEmpty) return false;
        
        // หา ':'
        int colonIdx = line.IndexOf(':');
        if (colonIdx == -1) return false;
        
        name = line[..colonIdx].Trim();
        value = line.Slice(colonIdx + 1).Trim();
        
        _data = lineEnd == -1 ? ReadOnlySpan<char>.Empty : _data.Slice(lineEnd + 1);
        return true;
    }
}

// Log parser
public readonly struct LogEntry
{
    public DateTime Timestamp { get; init; }
    public string Level { get; init; }
    public string Message { get; init; }
    public string? Source { get; init; }
}

public class HighPerfLogParser
{
    // Parse log entries แบบ zero-allocation (เท่าที่ทำได้)
    public static IEnumerable<LogEntry> Parse(string logContent)
    {
        var span = logContent.AsSpan();
        int lineStart = 0;
        
        while (lineStart < span.Length)
        {
            int lineEnd = span.Slice(lineStart).IndexOf('\n');
            ReadOnlySpan<char> line;
            
            if (lineEnd == -1)
            {
                line = span.Slice(lineStart).TrimEnd();
                lineStart = span.Length;
            }
            else
            {
                line = span.Slice(lineStart, lineEnd).TrimEnd('\r');
                lineStart += lineEnd + 1;
            }
            
            if (line.IsEmpty) continue;
            
            if (TryParseLogLine(line, out var entry))
                yield return entry;
        }
    }
    
    // Format: [2024-01-15 10:30:00] [INFO] [Service] Message text
    private static bool TryParseLogLine(ReadOnlySpan<char> line, out LogEntry entry)
    {
        entry = default;
        
        if (line.IsEmpty || line[0] != '[') return false;
        
        // Parse timestamp
        int closeBracket = line.IndexOf(']');
        if (closeBracket == -1) return false;
        
        ReadOnlySpan<char> timestampStr = line.Slice(1, closeBracket - 1);
        if (!DateTime.TryParse(timestampStr, out DateTime timestamp)) return false;
        
        line = line.Slice(closeBracket + 1).TrimStart();
        if (line.IsEmpty || line[0] != '[') return false;
        
        // Parse level
        closeBracket = line.IndexOf(']');
        if (closeBracket == -1) return false;
        
        string level = line.Slice(1, closeBracket - 1).ToString();
        line = line.Slice(closeBracket + 1).TrimStart();
        
        // Parse optional source
        string? source = null;
        if (!line.IsEmpty && line[0] == '[')
        {
            closeBracket = line.IndexOf(']');
            if (closeBracket != -1)
            {
                source = line.Slice(1, closeBracket - 1).ToString();
                line = line.Slice(closeBracket + 1).TrimStart();
            }
        }
        
        entry = new LogEntry
        {
            Timestamp = timestamp,
            Level = level,
            Message = line.ToString(),
            Source = source
        };
        
        return true;
    }
}

// Main Program
class Program
{
    static void Main()
    {
        Console.WriteLine("===== High-Performance String Parsing Demo =====\n");
        
        // CSV Parsing Demo
        Console.WriteLine("--- CSV Parser ---");
        string csvData = """
            1,สมชาย ใจดี,25,Bangkok,IT
            2,"Marie, Curie",42,Paris,Science
            3,John Doe,30,London,Engineering
            """;
        
        var lines = csvData.Split('\n', StringSplitOptions.RemoveEmptyEntries);
        foreach (var line in lines)
        {
            Console.Write($"Row: ");
            var reader = new CsvRowReader(line.AsSpan());
            bool first = true;
            
            while (reader.TryReadField(out var field))
            {
                if (!first) Console.Write(" | ");
                Console.Write(field.Raw.ToString());
                first = false;
            }
            Console.WriteLine();
        }
        
        // HTTP Headers Demo
        Console.WriteLine("\n--- HTTP Headers ---");
        string headers = """
            Content-Type: application/json
            Authorization: Bearer token123
            X-Request-Id: abc-def-123
            Accept-Encoding: gzip, deflate
            """;
        
        var headerParser = new HttpHeaderParser(headers.AsSpan());
        while (headerParser.TryReadHeader(out var name, out var value))
        {
            Console.WriteLine($"  {name}: {value}");
        }
        
        // Log Parser Demo
        Console.WriteLine("\n--- Log Parser ---");
        string logs = """
            [2024-01-15 10:30:00] [INFO] [AuthService] User logged in successfully
            [2024-01-15 10:30:05] [WARN] [Database] Slow query detected (>500ms)
            [2024-01-15 10:30:10] [ERROR] [PaymentService] Payment failed: timeout
            [2024-01-15 10:30:15] [DEBUG] API request received
            """;
        
        var entries = HighPerfLogParser.Parse(logs).ToList();
        foreach (var entry in entries)
        {
            Console.WriteLine($"  [{entry.Timestamp:HH:mm:ss}] {entry.Level,-6} " +
                $"{(entry.Source != null ? $"[{entry.Source}] " : "")}{entry.Message}");
        }
        
        Console.WriteLine($"\nParsed {entries.Count} log entries");
        
        // Performance
        Console.WriteLine("\n--- Performance Test ---");
        string bigCsv = GenerateBigCsv(10000);
        
        var sw = Stopwatch.StartNew();
        int total1 = 0;
        for (int i = 0; i < 100; i++)
        {
            foreach (var line in bigCsv.Split('\n'))
            {
                total1 += line.Split(',').Length;
            }
        }
        Console.WriteLine($"String.Split: {sw.ElapsedMilliseconds}ms (total={total1})");
        
        sw.Restart();
        int total2 = 0;
        for (int i = 0; i < 100; i++)
        {
            var span = bigCsv.AsSpan();
            int lineStart = 0;
            
            for (int j = 0; j <= span.Length; j++)
            {
                if (j == span.Length || span[j] == '\n')
                {
                    var reader = new CsvRowReader(span.Slice(lineStart, j - lineStart));
                    while (reader.TryReadField(out _)) total2++;
                    lineStart = j + 1;
                }
            }
        }
        Console.WriteLine($"Span parser: {sw.ElapsedMilliseconds}ms (total={total2})");
    }
    
    static string GenerateBigCsv(int rows)
    {
        var sb = new StringBuilder();
        for (int i = 0; i < rows; i++)
            sb.AppendLine($"{i},Name{i},{18 + (i % 50)},City{i % 20},Dept{i % 10}");
        return sb.ToString();
    }
}
```

---

## Exercises

### Exercise 1: Network Protocol Parser
```csharp
// TODO: Parse binary protocol data ด้วย Span<byte>
// Protocol: [1 byte version][2 bytes length][N bytes payload]

public ref struct ProtocolReader
{
    public bool TryRead(ReadOnlySpan<byte> data, out byte version, out ReadOnlySpan<byte> payload)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 2: Number Parsing Optimization
```csharp
// TODO: สร้าง number parser ที่ไม่ allocate
// รองรับ: integer, decimal, negative numbers, scientific notation

public static class FastNumberParser
{
    public static bool TryParseInt(ReadOnlySpan<char> text, out int result)
    {
        throw new NotImplementedException();
    }
    
    public static bool TryParseDouble(ReadOnlySpan<char> text, out double result)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: Token Splitter
```csharp
// TODO: สร้าง tokenizer ที่ใช้ Span โดยไม่ allocate
// แยก: identifiers, operators, numbers, strings

public ref struct Tokenizer
{
    public bool TryNext(out ReadOnlySpan<char> token, out TokenType type)
    {
        throw new NotImplementedException();
    }
}

public enum TokenType { Identifier, Number, String, Operator, Punctuation, Whitespace }
```

---

## สรุป

✅ **Span\<T\>** เป็น stack-only type ที่ไม่ allocate heap memory

✅ **ReadOnlySpan\<char\>** สำหรับ string operations แบบ zero-copy

✅ **Memory\<T\>** คือ Span ที่ใช้กับ async methods ได้

✅ **stackalloc** allocate memory บน stack สำหรับ small arrays

✅ **ArrayPool\<T\>** ลด GC pressure โดย reuse arrays

✅ **MemoryPool\<T\>** เหมาะกับ async operations พร้อม IDisposable

✅ Performance improvement สำคัญสำหรับ: CSV parsing, HTTP headers, log parsing, network protocols

✅ ข้อจำกัด: Span ใช้ได้เฉพาะ synchronous code, ไม่ใช้เป็น field, ไม่ใช้ใน async

---

## Part ถัดไป

➡️ **Part 040**: IDisposable และ Resource Management - Dispose pattern, using declarations, SafeHandle

---

*Part 039/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
