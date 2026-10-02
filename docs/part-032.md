# Part 032: Task Parallel Library (TPL)

## เนื้อหาใน Part นี้
- Task.Run
- Parallel.For และ Parallel.ForEach
- PLINQ (AsParallel)
- Thread Safety
- lock keyword
- Interlocked
- SemaphoreSlim
- โปรแกรมตัวอย่าง: Parallel file processing

---

## 1. Task.Run

`Task.Run` ใช้สำหรับ offload งาน CPU-intensive ไปยัง thread pool

```csharp
using System;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

class TaskRunExample
{
    static async Task Main()
    {
        // CPU-intensive work บน thread pool
        var sw = Stopwatch.StartNew();
        
        // ❌ แบบนี้บล็อค UI/main thread
        // int result = HeavyComputation(1000000);
        
        // ✅ offload ไป thread pool
        int result = await Task.Run(() => HeavyComputation(1000000));
        Console.WriteLine($"ผลลัพธ์: {result}, ใช้เวลา: {sw.ElapsedMilliseconds}ms");
        
        // Task.Run กับ async lambda
        var result2 = await Task.Run(async () =>
        {
            await Task.Delay(100); // จำลอง I/O
            return HeavyComputation(500000);
        });
        Console.WriteLine($"Result2: {result2}");
        
        // Task.Run แบบ parallel
        var tasks = new Task<long>[]
        {
            Task.Run(() => SumRange(1, 250000)),
            Task.Run(() => SumRange(250001, 500000)),
            Task.Run(() => SumRange(500001, 750000)),
            Task.Run(() => SumRange(750001, 1000000))
        };
        
        var results = await Task.WhenAll(tasks);
        long total = results.Sum();
        Console.WriteLine($"ผลรวม 1-1,000,000 = {total}");
    }
    
    static int HeavyComputation(int n)
    {
        int sum = 0;
        for (int i = 0; i < n; i++)
        {
            sum += i;
        }
        return sum;
    }
    
    static long SumRange(long start, long end)
    {
        long sum = 0;
        for (long i = start; i <= end; i++)
        {
            sum += i;
        }
        return sum;
    }
}
```

### เมื่อไหร่ควรใช้ Task.Run

```csharp
using System;
using System.Threading.Tasks;

class WhenToUseTaskRun
{
    // ✅ ใช้ Task.Run สำหรับ CPU-bound work
    public async Task<int> ComputeHashAsync(byte[] data)
    {
        return await Task.Run(() =>
        {
            // การคำนวณที่ใช้ CPU สูง
            int hash = 0;
            foreach (byte b in data)
            {
                hash = hash * 31 + b;
            }
            return hash;
        });
    }
    
    // ❌ ไม่ต้องใช้ Task.Run สำหรับ I/O-bound work
    public async Task<string> FetchDataAsync(string url)
    {
        using var client = new System.Net.Http.HttpClient();
        // HttpClient.GetStringAsync ใช้ async I/O แล้ว
        return await client.GetStringAsync(url);
    }
    
    // ❌ ไม่ต้องใช้ Task.Run แค่เพื่อทำให้เป็น async
    public async Task<int> BadAsyncMethodAsync(int n)
    {
        // นี่คือ "fake async" ที่เปลือง overhead
        return await Task.Run(() => n * 2);
    }
    
    // ✅ เปิดรับค่าแล้วคืนโดยตรง
    public Task<int> GoodAsyncMethodAsync(int n)
    {
        return Task.FromResult(n * 2);
    }
}
```

---

## 2. Parallel.For และ Parallel.ForEach

### Parallel.For

```csharp
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Threading;
using System.Threading.Tasks;

class ParallelForExample
{
    static void Main()
    {
        int itemCount = 10;
        
        // Sequential
        var sw = Stopwatch.StartNew();
        for (int i = 0; i < itemCount; i++)
        {
            ProcessItem(i);
        }
        Console.WriteLine($"Sequential: {sw.ElapsedMilliseconds}ms");
        
        // Parallel.For
        sw.Restart();
        Parallel.For(0, itemCount, i =>
        {
            ProcessItem(i);
        });
        Console.WriteLine($"Parallel.For: {sw.ElapsedMilliseconds}ms");
        
        // Parallel.For กับ options
        var options = new ParallelOptions
        {
            MaxDegreeOfParallelism = 4 // จำกัด threads
        };
        
        sw.Restart();
        Parallel.For(0, itemCount, options, i =>
        {
            ProcessItem(i);
        });
        Console.WriteLine($"Parallel.For (max 4): {sw.ElapsedMilliseconds}ms");
        
        // Parallel.For กับ local state (thread-local accumulation)
        long totalSum = 0;
        Parallel.For<long>(
            0, 1000000,
            () => 0L,                          // Initialize local state
            (i, state, localSum) => localSum + i, // Loop body
            localSum => Interlocked.Add(ref totalSum, localSum) // Finalize
        );
        Console.WriteLine($"Sum 0-999999 = {totalSum}");
        
        // Break และ Stop
        Parallel.For(0, 100, (i, state) =>
        {
            if (i == 50)
            {
                state.Break(); // หยุดหลังจาก index 50
                return;
            }
            if (state.ShouldExitCurrentIteration)
                return;
                
            Console.Write($"{i} ");
        });
    }
    
    static void ProcessItem(int id)
    {
        Thread.Sleep(100); // จำลองงาน 100ms
        Console.WriteLine($"  ประมวลผล item {id} บน thread {Thread.CurrentThread.ManagedThreadId}");
    }
}
```

### Parallel.ForEach

```csharp
using System;
using System.Collections.Generic;
using System.Collections.Concurrent;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

class ParallelForEachExample
{
    static void Main()
    {
        var items = Enumerable.Range(1, 20).ToList();
        
        // Basic Parallel.ForEach
        Parallel.ForEach(items, item =>
        {
            ProcessItem(item);
        });
        
        // Thread-safe collection
        var results = new ConcurrentBag<int>();
        Parallel.ForEach(items, item =>
        {
            int result = item * item;
            results.Add(result); // Thread-safe add
        });
        Console.WriteLine($"ได้ {results.Count} results");
        
        // Parallel.ForEach กับ Partitioner (สำหรับงานเล็กๆ จำนวนมาก)
        var smallItems = Enumerable.Range(1, 1000000).ToList();
        var partitioner = Partitioner.Create(smallItems, loadBalance: true);
        
        long sum = 0;
        Parallel.ForEach(partitioner,
            () => 0L,              // Thread-local init
            (item, state, local) => local + item,  // Body
            local => Interlocked.Add(ref sum, local) // Combine
        );
        Console.WriteLine($"Sum = {sum}");
        
        // Parallel.ForEachAsync (.NET 6+)
        ParallelForEachAsyncExample().Wait();
    }
    
    static async Task ParallelForEachAsyncExample()
    {
        var urls = new[] { "url1", "url2", "url3", "url4", "url5" };
        
        await Parallel.ForEachAsync(
            urls,
            new ParallelOptions { MaxDegreeOfParallelism = 3 },
            async (url, ct) =>
            {
                await Task.Delay(100, ct); // Async I/O
                Console.WriteLine($"ดาวน์โหลด {url} เสร็จแล้ว");
            }
        );
    }
    
    static void ProcessItem(int id)
    {
        Thread.Sleep(50);
        Console.WriteLine($"  Item {id} บน thread {Thread.CurrentThread.ManagedThreadId}");
    }
}
```

---

## 3. PLINQ (AsParallel)

PLINQ (Parallel LINQ) ทำให้ LINQ queries ทำงานแบบ parallel

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;

class PLINQExample
{
    static void Main()
    {
        var numbers = Enumerable.Range(1, 10000000).ToList();
        var sw = Stopwatch.StartNew();
        
        // Sequential LINQ
        var seqResult = numbers
            .Where(n => IsPrime(n))
            .Take(100)
            .ToList();
        Console.WriteLine($"Sequential: {sw.ElapsedMilliseconds}ms, found {seqResult.Count}");
        
        sw.Restart();
        
        // Parallel LINQ
        var parResult = numbers
            .AsParallel()
            .Where(n => IsPrime(n))
            .Take(100)
            .ToList();
        Console.WriteLine($"Parallel: {sw.ElapsedMilliseconds}ms, found {parResult.Count}");
        
        // AsOrdered - รักษาลำดับ (ช้ากว่า)
        var orderedResult = numbers
            .AsParallel()
            .AsOrdered()
            .Where(n => n % 7 == 0)
            .Take(20)
            .ToList();
        Console.WriteLine($"Ordered result: {string.Join(", ", orderedResult)}");
        
        // WithDegreeOfParallelism - จำกัด thread
        var limitedResult = numbers
            .AsParallel()
            .WithDegreeOfParallelism(2)
            .Where(n => n % 1000 == 0)
            .Select(n => n * 2)
            .ToList();
        
        // ForAll - execute action แบบ parallel
        numbers
            .AsParallel()
            .Where(n => n % 1000000 == 0)
            .ForAll(n => Console.WriteLine($"  พบตัวหาร 1,000,000: {n}"));
        
        // Aggregate แบบ parallel
        long sum = numbers
            .AsParallel()
            .Aggregate(
                () => 0L,           // เริ่มต้น local accumulator
                (local, n) => local + n,    // รวม item เข้า local
                (total, local) => total + local, // รวม local ทั้งหมด
                total => total      // ผลลัพธ์สุดท้าย
            );
        Console.WriteLine($"Sum: {sum}");
        
        // WithCancellation
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(2));
        try
        {
            var cancelResult = numbers
                .AsParallel()
                .WithCancellation(cts.Token)
                .Where(n => IsPrime(n))
                .ToList();
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("PLINQ ถูกยกเลิก");
        }
        catch (AggregateException ae) when (ae.InnerExceptions.Any(e => e is OperationCanceledException))
        {
            Console.WriteLine("PLINQ ถูกยกเลิก (AggregateException)");
        }
    }
    
    static bool IsPrime(int n)
    {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        
        for (int i = 3; i <= Math.Sqrt(n); i += 2)
        {
            if (n % i == 0) return false;
        }
        return true;
    }
}
```

---

## 4. Thread Safety

Thread safety หมายความว่าโค้ดทำงานถูกต้องเมื่อมีหลาย threads ทำงานพร้อมกัน

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

class ThreadSafetyExample
{
    // ❌ ไม่ thread-safe - Race condition
    static int _counter = 0;
    
    static async Task DemonstrateRaceCondition()
    {
        _counter = 0;
        var tasks = Enumerable.Range(0, 1000).Select(_ => Task.Run(() =>
        {
            _counter++; // ไม่ atomic! race condition!
        }));
        
        await Task.WhenAll(tasks);
        Console.WriteLine($"Expected: 1000, Actual: {_counter}"); // มักได้ค่าน้อยกว่า 1000
    }
    
    // ✅ Thread-safe ด้วย lock
    static readonly object _lockObj = new object();
    static int _safeCounter = 0;
    
    static async Task DemonstrateLock()
    {
        _safeCounter = 0;
        var tasks = Enumerable.Range(0, 1000).Select(_ => Task.Run(() =>
        {
            lock (_lockObj)
            {
                _safeCounter++; // thread-safe
            }
        }));
        
        await Task.WhenAll(tasks);
        Console.WriteLine($"Expected: 1000, Actual: {_safeCounter}"); // ได้ 1000 เสมอ
    }
    
    static async Task Main()
    {
        Console.WriteLine("Race Condition Demo:");
        await DemonstrateRaceCondition();
        
        Console.WriteLine("\nlock Demo:");
        await DemonstrateLock();
    }
}
```

---

## 5. lock keyword

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

class LockExample
{
    private readonly object _lock = new object();
    private List<string> _sharedList = new List<string>();
    private int _count = 0;
    
    // Thread-safe method ด้วย lock
    public void AddItem(string item)
    {
        lock (_lock)
        {
            _sharedList.Add(item);
            _count++;
        }
    }
    
    // Thread-safe read
    public List<string> GetItems()
    {
        lock (_lock)
        {
            return new List<string>(_sharedList); // Return copy
        }
    }
    
    // Double-check locking pattern (for lazy initialization)
    private volatile object? _expensiveObject;
    private readonly object _initLock = new object();
    
    public object GetExpensiveObject()
    {
        if (_expensiveObject == null)
        {
            lock (_initLock)
            {
                if (_expensiveObject == null) // double-check
                {
                    _expensiveObject = CreateExpensiveObject();
                }
            }
        }
        return _expensiveObject;
    }
    
    // ReaderWriterLockSlim - อนุญาตหลาย readers แต่ writer เดียว
    private readonly ReaderWriterLockSlim _rwLock = new ReaderWriterLockSlim();
    private Dictionary<string, string> _cache = new Dictionary<string, string>();
    
    public string? ReadFromCache(string key)
    {
        _rwLock.EnterReadLock();
        try
        {
            return _cache.TryGetValue(key, out string? value) ? value : null;
        }
        finally
        {
            _rwLock.ExitReadLock();
        }
    }
    
    public void WriteToCache(string key, string value)
    {
        _rwLock.EnterWriteLock();
        try
        {
            _cache[key] = value;
        }
        finally
        {
            _rwLock.ExitWriteLock();
        }
    }
    
    private object CreateExpensiveObject() => new object();
    
    static async Task Main()
    {
        var example = new LockExample();
        
        // Test thread-safe list
        var tasks = Enumerable.Range(0, 100).Select(i =>
            Task.Run(() => example.AddItem($"item_{i}"))
        );
        
        await Task.WhenAll(tasks);
        var items = example.GetItems();
        Console.WriteLine($"Total items: {items.Count} (expected 100)");
        
        // Test cache with reader/writer lock
        var writeTasks = Enumerable.Range(0, 10).Select(i =>
            Task.Run(() => example.WriteToCache($"key{i}", $"value{i}"))
        );
        
        var readTasks = Enumerable.Range(0, 50).Select(i =>
            Task.Run(() =>
            {
                string? val = example.ReadFromCache($"key{i % 10}");
                return val;
            })
        );
        
        await Task.WhenAll(writeTasks.Concat(readTasks.Cast<Task>()));
        Console.WriteLine("Cache operations completed successfully");
    }
}
```

---

## 6. Interlocked

`Interlocked` ให้ atomic operations สำหรับ simple numeric operations โดยไม่ต้องใช้ lock

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class InterlockedExample
{
    private static int _counter = 0;
    private static long _totalBytes = 0;
    
    static async Task Main()
    {
        // Interlocked.Increment - atomic ++
        var tasks = Enumerable.Range(0, 10000).Select(_ => Task.Run(() =>
        {
            Interlocked.Increment(ref _counter);
        }));
        
        await Task.WhenAll(tasks);
        Console.WriteLine($"Counter: {_counter}"); // เสมอ 10000
        
        // Interlocked operations ต่างๆ
        int initial = 0;
        
        // Increment (++)
        int newValue = Interlocked.Increment(ref initial);
        Console.WriteLine($"After Increment: {initial}"); // 1
        
        // Decrement (--)
        Interlocked.Decrement(ref initial);
        Console.WriteLine($"After Decrement: {initial}"); // 0
        
        // Add
        Interlocked.Add(ref initial, 10);
        Console.WriteLine($"After Add 10: {initial}"); // 10
        
        // Exchange (set และ return old value)
        int old = Interlocked.Exchange(ref initial, 100);
        Console.WriteLine($"Exchange: old={old}, new={initial}"); // old=10, new=100
        
        // CompareExchange (CAS - Compare And Swap)
        // เปลี่ยนค่าก็ต่อเมื่อค่าปัจจุบัน == expected
        int expected = 100;
        int desired = 200;
        int result = Interlocked.CompareExchange(ref initial, desired, expected);
        Console.WriteLine($"CAS: result={result}, initial={initial}"); // result=100 (old), initial=200
        
        // ตัวอย่าง Lock-free counter
        var counter = new LockFreeCounter();
        
        var countTasks = Enumerable.Range(0, 10000).Select(_ =>
            Task.Run(() => counter.Increment())
        );
        await Task.WhenAll(countTasks);
        Console.WriteLine($"Lock-free count: {counter.Value}"); // 10000
    }
}

// Lock-free counter ด้วย Interlocked
public class LockFreeCounter
{
    private int _value = 0;
    
    public int Value => _value;
    
    public void Increment() => Interlocked.Increment(ref _value);
    
    public void Decrement() => Interlocked.Decrement(ref _value);
    
    public void Add(int amount) => Interlocked.Add(ref _value, amount);
    
    // Thread-safe lazy init pattern
    private object? _resource;
    
    public object GetOrCreateResource()
    {
        if (_resource != null) return _resource;
        
        var newResource = new object();
        return Interlocked.CompareExchange(ref _resource, newResource, null) ?? newResource;
    }
}
```

---

## 7. SemaphoreSlim

`SemaphoreSlim` ใช้จำกัดจำนวน concurrent operations

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

class SemaphoreSlimExample
{
    static async Task Main()
    {
        // Basic usage - จำกัดให้ทำงานพร้อมกันได้ 3 thread
        using var semaphore = new SemaphoreSlim(3, 3);
        
        var tasks = Enumerable.Range(1, 10).Select(async id =>
        {
            await semaphore.WaitAsync(); // รอจน slot ว่าง
            try
            {
                Console.WriteLine($"Task {id} เริ่มทำงาน (available: {semaphore.CurrentCount})");
                await Task.Delay(1000); // จำลองงาน
                Console.WriteLine($"Task {id} เสร็จสิ้น");
            }
            finally
            {
                semaphore.Release(); // คืน slot
            }
        });
        
        await Task.WhenAll(tasks);
        
        // API Rate Limiting
        Console.WriteLine("\n--- Rate Limiting Demo ---");
        var rateLimiter = new ApiRateLimiter(maxConcurrent: 2, maxPerSecond: 5);
        
        var apiTasks = Enumerable.Range(1, 20).Select(async i =>
        {
            await rateLimiter.ExecuteAsync(async () =>
            {
                await Task.Delay(200); // จำลอง API call
                Console.WriteLine($"  API call {i} เสร็จ");
            });
        });
        
        await Task.WhenAll(apiTasks);
        Console.WriteLine("ทุก API calls เสร็จสิ้น");
    }
}

// Rate limiter ด้วย SemaphoreSlim
public class ApiRateLimiter
{
    private readonly SemaphoreSlim _concurrentLimit;
    private readonly SemaphoreSlim _rateLimit;
    private readonly int _maxPerSecond;
    
    public ApiRateLimiter(int maxConcurrent, int maxPerSecond)
    {
        _concurrentLimit = new SemaphoreSlim(maxConcurrent, maxConcurrent);
        _rateLimit = new SemaphoreSlim(maxPerSecond, maxPerSecond);
        _maxPerSecond = maxPerSecond;
        
        // Reset rate limit ทุก 1 วินาที
        Task.Run(async () =>
        {
            while (true)
            {
                await Task.Delay(1000);
                int toRelease = _maxPerSecond - _rateLimit.CurrentCount;
                if (toRelease > 0)
                {
                    _rateLimit.Release(toRelease);
                }
            }
        });
    }
    
    public async Task ExecuteAsync(Func<Task> operation, CancellationToken ct = default)
    {
        await _rateLimit.WaitAsync(ct);
        await _concurrentLimit.WaitAsync(ct);
        
        try
        {
            await operation();
        }
        finally
        {
            _concurrentLimit.Release();
        }
    }
}
```

---

## 8. โปรแกรมตัวอย่าง: Parallel File Processing

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

// Models
public record FileProcessingResult(
    string FilePath,
    long FileSize,
    int LineCount,
    int WordCount,
    string Checksum,
    TimeSpan ProcessingTime,
    bool Success,
    string? Error = null
);

public record ProcessingSummary(
    int TotalFiles,
    int SuccessCount,
    int ErrorCount,
    long TotalBytes,
    long TotalWords,
    TimeSpan TotalTime,
    double FilesPerSecond
);

// File Processor
public class ParallelFileProcessor
{
    private readonly int _maxConcurrency;
    private readonly SemaphoreSlim _semaphore;
    
    public ParallelFileProcessor(int maxConcurrency = 4)
    {
        _maxConcurrency = maxConcurrency;
        _semaphore = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    }
    
    // Process ไฟล์ทั้งหมดแบบ parallel
    public async Task<ProcessingSummary> ProcessDirectoryAsync(
        string directoryPath,
        string searchPattern = "*.txt",
        IProgress<(string file, int percent)>? progress = null,
        CancellationToken cancellationToken = default)
    {
        var files = Directory.GetFiles(directoryPath, searchPattern, SearchOption.AllDirectories);
        Console.WriteLine($"พบ {files.Length} ไฟล์");
        
        var results = new ConcurrentBag<FileProcessingResult>();
        int processed = 0;
        
        var sw = Stopwatch.StartNew();
        
        await Parallel.ForEachAsync(
            files,
            new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxConcurrency,
                CancellationToken = cancellationToken
            },
            async (filePath, ct) =>
            {
                var result = await ProcessFileAsync(filePath, ct);
                results.Add(result);
                
                int current = Interlocked.Increment(ref processed);
                int percent = (int)((double)current / files.Length * 100);
                progress?.Report((filePath, percent));
                
                Console.WriteLine($"[{percent:D3}%] ประมวลผล: {Path.GetFileName(filePath)} " +
                    $"({result.LineCount} บรรทัด, {result.WordCount} คำ)");
            }
        );
        
        sw.Stop();
        
        var allResults = results.ToList();
        return new ProcessingSummary(
            TotalFiles: allResults.Count,
            SuccessCount: allResults.Count(r => r.Success),
            ErrorCount: allResults.Count(r => !r.Success),
            TotalBytes: allResults.Where(r => r.Success).Sum(r => r.FileSize),
            TotalWords: allResults.Where(r => r.Success).Sum(r => (long)r.WordCount),
            TotalTime: sw.Elapsed,
            FilesPerSecond: allResults.Count / sw.Elapsed.TotalSeconds
        );
    }
    
    // Process ไฟล์เดียว
    private async Task<FileProcessingResult> ProcessFileAsync(
        string filePath,
        CancellationToken cancellationToken)
    {
        var sw = Stopwatch.StartNew();
        
        try
        {
            var content = await File.ReadAllTextAsync(filePath, cancellationToken);
            
            // คำนวณ metrics แบบ parallel ด้วย Task.WhenAll
            var (lineCount, wordCount, checksum) = await Task.WhenAll(
                Task.Run(() => CountLines(content), cancellationToken),
                Task.Run(() => CountWords(content), cancellationToken),
                Task.Run(() => ComputeChecksum(content), cancellationToken)
            ).ContinueWith(t => (t.Result[0], t.Result[1], (string)t.Result[2]));
            
            return new FileProcessingResult(
                FilePath: filePath,
                FileSize: new FileInfo(filePath).Length,
                LineCount: lineCount,
                WordCount: wordCount,
                Checksum: checksum,
                ProcessingTime: sw.Elapsed,
                Success: true
            );
        }
        catch (Exception ex)
        {
            return new FileProcessingResult(
                FilePath: filePath,
                FileSize: 0,
                LineCount: 0,
                WordCount: 0,
                Checksum: string.Empty,
                ProcessingTime: sw.Elapsed,
                Success: false,
                Error: ex.Message
            );
        }
    }
    
    private static int CountLines(string content)
    {
        if (string.IsNullOrEmpty(content)) return 0;
        int count = 1;
        foreach (char c in content)
        {
            if (c == '\n') count++;
        }
        return count;
    }
    
    private static int CountWords(string content)
    {
        if (string.IsNullOrEmpty(content)) return 0;
        return content.Split(new[] { ' ', '\t', '\n', '\r' },
            StringSplitOptions.RemoveEmptyEntries).Length;
    }
    
    private static string ComputeChecksum(string content)
    {
        using var sha256 = SHA256.Create();
        var bytes = Encoding.UTF8.GetBytes(content);
        var hash = sha256.ComputeHash(bytes);
        return Convert.ToHexString(hash)[..8];
    }
}

// Comparison: Sequential vs Parallel
public class ProcessingComparison
{
    public static async Task RunComparison(string directory)
    {
        var files = Directory.GetFiles(directory, "*.txt");
        Console.WriteLine($"\n=== เปรียบเทียบ Sequential vs Parallel ===");
        Console.WriteLine($"จำนวนไฟล์: {files.Length}");
        
        // Sequential
        var sw = Stopwatch.StartNew();
        long seqWordCount = 0;
        
        foreach (var file in files)
        {
            var content = await File.ReadAllTextAsync(file);
            seqWordCount += content.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
        }
        
        var seqTime = sw.Elapsed;
        Console.WriteLine($"Sequential: {seqTime.TotalMilliseconds:F0}ms, คำทั้งหมด: {seqWordCount}");
        
        // Parallel with PLINQ
        sw.Restart();
        var parWordCount = files
            .AsParallel()
            .WithDegreeOfParallelism(Environment.ProcessorCount)
            .Select(file => File.ReadAllText(file))
            .Sum(content => content.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length);
        
        var parTime = sw.Elapsed;
        Console.WriteLine($"PLINQ: {parTime.TotalMilliseconds:F0}ms, คำทั้งหมด: {parWordCount}");
        Console.WriteLine($"Speedup: {seqTime.TotalMilliseconds / parTime.TotalMilliseconds:F2}x");
    }
}

// Main Program
class Program
{
    static async Task Main()
    {
        // สร้าง test files
        var tempDir = Path.Combine(Path.GetTempPath(), "parallel_test");
        Directory.CreateDirectory(tempDir);
        
        Console.WriteLine("สร้าง test files...");
        await CreateTestFilesAsync(tempDir, 20);
        
        Console.WriteLine("\n=== Parallel File Processing Demo ===");
        
        var processor = new ParallelFileProcessor(maxConcurrency: 4);
        
        var progress = new Progress<(string file, int percent)>(p =>
        {
            // Progress callback
        });
        
        using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(2));
        
        var summary = await processor.ProcessDirectoryAsync(
            tempDir,
            progress: progress,
            cancellationToken: cts.Token
        );
        
        Console.WriteLine("\n=== สรุปผลการประมวลผล ===");
        Console.WriteLine($"ไฟล์ทั้งหมด:  {summary.TotalFiles}");
        Console.WriteLine($"สำเร็จ:       {summary.SuccessCount}");
        Console.WriteLine($"ผิดพลาด:      {summary.ErrorCount}");
        Console.WriteLine($"ขนาดรวม:     {summary.TotalBytes:N0} bytes");
        Console.WriteLine($"คำรวม:       {summary.TotalWords:N0} คำ");
        Console.WriteLine($"ใช้เวลา:     {summary.TotalTime.TotalSeconds:F2} วินาที");
        Console.WriteLine($"อัตรา:       {summary.FilesPerSecond:F1} ไฟล์/วินาที");
        
        // Cleanup
        Directory.Delete(tempDir, recursive: true);
    }
    
    static async Task CreateTestFilesAsync(string directory, int count)
    {
        var tasks = Enumerable.Range(1, count).Select(i =>
            File.WriteAllTextAsync(
                Path.Combine(directory, $"file_{i:D3}.txt"),
                GenerateContent(i * 100)
            )
        );
        await Task.WhenAll(tasks);
        Console.WriteLine($"สร้าง {count} ไฟล์เรียบร้อย");
    }
    
    static string GenerateContent(int wordCount)
    {
        var words = new[] { "the", "quick", "brown", "fox", "jumps", "over",
                           "lazy", "dog", "hello", "world", "async", "parallel",
                           "task", "thread", "programming", "dotnet", "csharp" };
        
        var sb = new StringBuilder();
        var random = new Random();
        
        for (int i = 0; i < wordCount; i++)
        {
            sb.Append(words[random.Next(words.Length)]);
            sb.Append(i % 15 == 14 ? '\n' : ' ');
        }
        
        return sb.ToString();
    }
}
```

---

## Exercises

### Exercise 1: Parallel Matrix Multiplication
```csharp
// TODO: Implement matrix multiplication แบบ parallel
public static int[,] MultiplyMatricesParallel(int[,] a, int[,] b)
{
    // 1. ตรวจสอบ dimensions
    // 2. ใช้ Parallel.For เพื่อคำนวณแต่ละ row
    // 3. Return ผลลัพธ์
    throw new NotImplementedException();
}

// Test:
// var a = new int[,] { {1,2,3}, {4,5,6} };
// var b = new int[,] { {7,8}, {9,10}, {11,12} };
// var result = MultiplyMatricesParallel(a, b);
// Expected: { {58,64}, {139,154} }
```

### Exercise 2: Concurrent Cache
```csharp
// TODO: สร้าง thread-safe cache ด้วย ConcurrentDictionary
public class ThreadSafeCache<TKey, TValue> where TKey : notnull
{
    // ใช้ ConcurrentDictionary แทน Dictionary + lock
    // Implement: Get, Set, GetOrAdd, Remove, Clear
    // Add expiration support
}
```

### Exercise 3: Parallel Image Processing
```csharp
// TODO: ประมวลผลไฟล์ภาพหลายไฟล์พร้อมกัน
// - Resize images
// - Convert to grayscale  
// - ใช้ PLINQ
// - Report progress
```

---

## สรุป

✅ **Task.Run** ใช้ offload งาน CPU-intensive ไปยัง thread pool

✅ **Parallel.For/ForEach** สำหรับ data parallelism แบบ CPU-bound

✅ **Parallel.ForEachAsync** สำหรับ async operations แบบ parallel (.NET 6+)

✅ **PLINQ** ทำให้ LINQ queries ทำงาน parallel ด้วย .AsParallel()

✅ **lock** ใช้ป้องกัน race conditions บน shared resources

✅ **Interlocked** สำหรับ atomic operations บน simple types (เร็วกว่า lock)

✅ **SemaphoreSlim** จำกัดจำนวน concurrent operations เช่น API rate limiting

✅ ใช้ **ConcurrentDictionary, ConcurrentBag, ConcurrentQueue** แทน non-thread-safe collections

---

## Part ถัดไป

➡️ **Part 033**: Extension Methods - สร้าง extension methods สำหรับ string, int, IEnumerable และ best practices

---

*Part 032/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
