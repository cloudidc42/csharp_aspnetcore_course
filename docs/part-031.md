# Part 031: Async/Await เบื้องต้น

## เนื้อหาใน Part นี้
- Synchronous vs Asynchronous Programming
- Task และ Task\<T\>
- async/await keywords
- await หลาย tasks พร้อมกัน (Task.WhenAll, Task.WhenAny)
- CancellationToken
- async void ปัญหา
- ConfigureAwait
- โปรแกรมตัวอย่าง: Async HTTP requests

---

## 1. Synchronous vs Asynchronous Programming

### Synchronous (แบบเดิม)

ในการเขียนโปรแกรมแบบ synchronous โค้ดจะทำงานตามลำดับ บรรทัดต่อบรรทัด เมื่อมีการเรียกใช้ฟังก์ชันที่ใช้เวลานาน (เช่น อ่านไฟล์, เรียก API) thread จะถูกบล็อครอจนกว่าจะเสร็จ

```csharp
// Synchronous - thread ถูกบล็อคระหว่างรอ
using System;
using System.Net.Http;
using System.Threading;

class SynchronousExample
{
    static void Main()
    {
        Console.WriteLine($"เริ่มต้น - Thread: {Thread.CurrentThread.ManagedThreadId}");
        
        // Thread ถูกบล็อค รอนาน
        string result = GetDataSync("https://api.example.com/data");
        
        Console.WriteLine($"ได้ข้อมูลแล้ว: {result}");
        Console.WriteLine($"เสร็จสิ้น - Thread: {Thread.CurrentThread.ManagedThreadId}");
    }
    
    static string GetDataSync(string url)
    {
        // จำลองการรอ 3 วินาที
        Thread.Sleep(3000);
        return "ข้อมูลจาก API";
    }
}
```

ปัญหาของ synchronous: 
- **UI Applications**: หน้าต่างแอปค้างไม่ตอบสนอง
- **Web Applications**: thread pool หมด รองรับ request น้อยลง
- **ประสิทธิภาพต่ำ**: CPU ว่างแต่ thread ถูกบล็อค

### Asynchronous (แบบใหม่)

```csharp
// Asynchronous - thread ไม่ถูกบล็อค
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;

class AsynchronousExample
{
    static async Task Main()
    {
        Console.WriteLine($"เริ่มต้น - Thread: {Thread.CurrentThread.ManagedThreadId}");
        
        // Thread ไม่ถูกบล็อค สามารถทำงานอื่นได้
        string result = await GetDataAsync("https://api.example.com/data");
        
        Console.WriteLine($"ได้ข้อมูลแล้ว: {result}");
        Console.WriteLine($"เสร็จสิ้น - Thread: {Thread.CurrentThread.ManagedThreadId}");
    }
    
    static async Task<string> GetDataAsync(string url)
    {
        // จำลองการรอ async
        await Task.Delay(3000);
        return "ข้อมูลจาก API";
    }
}
```

---

## 2. Task และ Task\<T\>

`Task` คือ class ที่แทนการดำเนินงานแบบ asynchronous ใน .NET

### Task (ไม่มีค่า return)

```csharp
using System;
using System.Threading.Tasks;

class TaskExample
{
    static async Task Main()
    {
        // สร้าง Task ที่ทำงานแล้วรอ
        Task task = DoWorkAsync();
        Console.WriteLine("Task ถูกสร้างแล้ว กำลังทำงาน...");
        await task;
        Console.WriteLine("Task เสร็จสิ้น");
        
        // Task.CompletedTask - Task ที่เสร็จแล้วทันที
        Task completed = Task.CompletedTask;
        await completed; // ไม่รอ
        
        // Task.FromResult - สร้าง Task ที่มีค่าแล้ว
        Task<int> immediateResult = Task.FromResult(42);
        int value = await immediateResult;
        Console.WriteLine($"ค่า: {value}");
        
        // Task.FromException - สร้าง Task ที่มี exception
        Task<string> failedTask = Task.FromException<string>(
            new InvalidOperationException("เกิดข้อผิดพลาด")
        );
        
        try
        {
            await failedTask;
        }
        catch (InvalidOperationException ex)
        {
            Console.WriteLine($"จับ Exception: {ex.Message}");
        }
    }
    
    static async Task DoWorkAsync()
    {
        await Task.Delay(1000);
        Console.WriteLine("งานเสร็จแล้ว!");
    }
}
```

### Task\<T\> (มีค่า return)

```csharp
using System;
using System.Threading.Tasks;

class TaskOfTExample
{
    static async Task Main()
    {
        // Task<string> - return string
        string text = await FetchTextAsync();
        Console.WriteLine($"Text: {text}");
        
        // Task<int> - return int
        int number = await CalculateAsync(10, 20);
        Console.WriteLine($"ผลลัพธ์: {number}");
        
        // Task<List<string>> - return List
        var items = await GetItemsAsync();
        foreach (var item in items)
        {
            Console.WriteLine($"  - {item}");
        }
    }
    
    static async Task<string> FetchTextAsync()
    {
        await Task.Delay(500);
        return "Hello, Async World!";
    }
    
    static async Task<int> CalculateAsync(int a, int b)
    {
        await Task.Delay(100);
        return a + b;
    }
    
    static async Task<List<string>> GetItemsAsync()
    {
        await Task.Delay(200);
        return new List<string> { "item1", "item2", "item3" };
    }
}
```

### Task Status

```csharp
using System;
using System.Threading.Tasks;

class TaskStatusExample
{
    static async Task Main()
    {
        // สร้าง Task ที่ยังไม่เริ่ม
        var task = new Task(async () => await Task.Delay(2000));
        Console.WriteLine($"Status: {task.Status}"); // Created
        
        task.Start();
        Console.WriteLine($"Status: {task.Status}"); // Running/WaitingForActivation
        
        await task;
        Console.WriteLine($"Status: {task.Status}"); // RanToCompletion
        
        // Task ที่ถูก cancel
        var cts = new CancellationTokenSource();
        var cancelTask = Task.Run(async () =>
        {
            await Task.Delay(5000, cts.Token);
        }, cts.Token);
        
        cts.Cancel();
        
        try
        {
            await cancelTask;
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine($"Canceled Status: {cancelTask.Status}"); // Canceled
        }
        
        // Task ที่เกิด exception
        var faultedTask = Task.Run(() => throw new Exception("เกิดข้อผิดพลาด!"));
        
        try
        {
            await faultedTask;
        }
        catch
        {
            Console.WriteLine($"Faulted Status: {faultedTask.Status}"); // Faulted
        }
    }
}
```

---

## 3. async/await Keywords

### หลักการทำงาน

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class AsyncAwaitMechanics
{
    static async Task Main()
    {
        Console.WriteLine($"[Main] เริ่มต้น Thread: {Thread.CurrentThread.ManagedThreadId}");
        
        // await จะ:
        // 1. หยุดการทำงานของ method นี้ชั่วคราว
        // 2. return control กลับไปยัง caller
        // 3. เมื่อ Task เสร็จ ทำงานต่อจากจุดที่หยุด
        await ProcessAsync();
        
        Console.WriteLine($"[Main] เสร็จสิ้น Thread: {Thread.CurrentThread.ManagedThreadId}");
    }
    
    static async Task ProcessAsync()
    {
        Console.WriteLine($"[Process] ก่อน await Thread: {Thread.CurrentThread.ManagedThreadId}");
        
        await Task.Delay(1000); // ปล่อย thread ไปทำงานอื่น
        
        // อาจเป็น thread อื่น (ขึ้นกับ SynchronizationContext)
        Console.WriteLine($"[Process] หลัง await Thread: {Thread.CurrentThread.ManagedThreadId}");
    }
}
```

### การเขียน async method ที่ถูกต้อง

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

class AsyncMethodPatterns
{
    // Pattern 1: async Task - สำหรับ void methods
    static async Task SaveDataAsync(string data)
    {
        await Task.Delay(100); // จำลองการบันทึก
        Console.WriteLine($"บันทึกข้อมูล: {data}");
    }
    
    // Pattern 2: async Task<T> - สำหรับ methods ที่ return ค่า
    static async Task<string> FetchDataAsync(string url)
    {
        await Task.Delay(200); // จำลองการดึงข้อมูล
        return $"ข้อมูลจาก {url}";
    }
    
    // Pattern 3: async IAsyncEnumerable<T> - สำหรับ streaming data
    static async IAsyncEnumerable<int> GenerateNumbersAsync(int count)
    {
        for (int i = 0; i < count; i++)
        {
            await Task.Delay(100);
            yield return i;
        }
    }
    
    // Pattern 4: ValueTask<T> - เมื่อผลลัพธ์มักพร้อมทันที (performance optimization)
    static async ValueTask<int> GetCachedValueAsync(bool useCache)
    {
        if (useCache)
        {
            return 42; // ไม่ต้องสร้าง Task object
        }
        
        await Task.Delay(100);
        return 100;
    }
    
    static async Task Main()
    {
        await SaveDataAsync("test data");
        
        string data = await FetchDataAsync("https://api.example.com");
        Console.WriteLine(data);
        
        await foreach (int num in GenerateNumbersAsync(5))
        {
            Console.WriteLine($"ได้ค่า: {num}");
        }
        
        int cached = await GetCachedValueAsync(true);
        int fetched = await GetCachedValueAsync(false);
        Console.WriteLine($"Cached: {cached}, Fetched: {fetched}");
    }
}
```

### Exception Handling ใน async

```csharp
using System;
using System.Threading.Tasks;

class AsyncExceptionHandling
{
    static async Task Main()
    {
        // Pattern 1: try-catch ปกติ
        try
        {
            await OperationThatThrowsAsync();
        }
        catch (InvalidOperationException ex)
        {
            Console.WriteLine($"จับ exception: {ex.Message}");
        }
        
        // Pattern 2: AggregateException เมื่อ await หลาย Tasks
        try
        {
            await Task.WhenAll(
                ThrowAfterDelayAsync(100, "error 1"),
                ThrowAfterDelayAsync(200, "error 2")
            );
        }
        catch (Exception ex)
        {
            // จะได้แค่ exception แรก
            Console.WriteLine($"First exception: {ex.Message}");
        }
        
        // Pattern 3: ดึง exceptions ทั้งหมด
        var task1 = ThrowAfterDelayAsync(100, "error 1");
        var task2 = ThrowAfterDelayAsync(200, "error 2");
        
        var combinedTask = Task.WhenAll(task1, task2);
        try
        {
            await combinedTask;
        }
        catch
        {
            if (combinedTask.Exception != null)
            {
                foreach (var ex in combinedTask.Exception.InnerExceptions)
                {
                    Console.WriteLine($"Exception: {ex.Message}");
                }
            }
        }
    }
    
    static async Task OperationThatThrowsAsync()
    {
        await Task.Delay(100);
        throw new InvalidOperationException("การดำเนินการล้มเหลว");
    }
    
    static async Task ThrowAfterDelayAsync(int delay, string message)
    {
        await Task.Delay(delay);
        throw new Exception(message);
    }
}
```

---

## 4. await หลาย Tasks พร้อมกัน

### Task.WhenAll - รอทุก Task เสร็จ

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading.Tasks;

class TaskWhenAllExample
{
    static async Task Main()
    {
        var sw = Stopwatch.StartNew();
        
        // Sequential - ช้า
        Console.WriteLine("Sequential:");
        await FetchUserAsync(1);
        await FetchUserAsync(2);
        await FetchUserAsync(3);
        Console.WriteLine($"ใช้เวลา: {sw.ElapsedMilliseconds}ms"); // ~3000ms
        
        sw.Restart();
        
        // Parallel with WhenAll - เร็ว
        Console.WriteLine("\nParallel with WhenAll:");
        string[] users = await Task.WhenAll(
            FetchUserAsync(1),
            FetchUserAsync(2),
            FetchUserAsync(3)
        );
        Console.WriteLine($"ใช้เวลา: {sw.ElapsedMilliseconds}ms"); // ~1000ms
        
        foreach (var user in users)
        {
            Console.WriteLine($"  User: {user}");
        }
        
        // WhenAll กับ List of tasks
        var userIds = new List<int> { 1, 2, 3, 4, 5 };
        var tasks = userIds.Select(id => FetchUserAsync(id));
        var allUsers = await Task.WhenAll(tasks);
        
        Console.WriteLine($"\nได้ {allUsers.Length} users");
    }
    
    static async Task<string> FetchUserAsync(int id)
    {
        await Task.Delay(1000); // จำลองการดึงข้อมูล
        return $"User_{id}";
    }
}
```

### Task.WhenAny - รอ Task แรกที่เสร็จ

```csharp
using System;
using System.Diagnostics;
using System.Threading.Tasks;

class TaskWhenAnyExample
{
    static async Task Main()
    {
        // Pattern 1: รับผลลัพธ์จาก task ที่เร็วที่สุด
        var task1 = SlowFetchAsync("Server A", 3000);
        var task2 = SlowFetchAsync("Server B", 1000);
        var task3 = SlowFetchAsync("Server C", 2000);
        
        Task<string> firstTask = await Task.WhenAny(task1, task2, task3);
        string firstResult = await firstTask;
        Console.WriteLine($"ได้ผลแรก: {firstResult}");
        
        // Pattern 2: Timeout pattern
        var longOperation = LongRunningOperationAsync();
        var timeoutTask = Task.Delay(2000); // timeout 2 วินาที
        
        if (await Task.WhenAny(longOperation, timeoutTask) == timeoutTask)
        {
            Console.WriteLine("Timeout! การดำเนินการใช้เวลานานเกินไป");
        }
        else
        {
            string result = await longOperation;
            Console.WriteLine($"สำเร็จ: {result}");
        }
        
        // Pattern 3: Process tasks as they complete
        var tasks = new[]
        {
            ProcessItemAsync("item1", 3000),
            ProcessItemAsync("item2", 1000),
            ProcessItemAsync("item3", 2000)
        };
        
        var remaining = new List<Task<string>>(tasks);
        while (remaining.Count > 0)
        {
            Task<string> completed = await Task.WhenAny(remaining);
            remaining.Remove(completed);
            Console.WriteLine($"เสร็จ: {await completed}");
        }
    }
    
    static async Task<string> SlowFetchAsync(string server, int delay)
    {
        await Task.Delay(delay);
        return $"ข้อมูลจาก {server}";
    }
    
    static async Task<string> LongRunningOperationAsync()
    {
        await Task.Delay(5000);
        return "ผลลัพธ์จากการดำเนินการนาน";
    }
    
    static async Task<string> ProcessItemAsync(string item, int delay)
    {
        await Task.Delay(delay);
        return $"{item} ประมวลผลแล้ว";
    }
}
```

---

## 5. CancellationToken

CancellationToken ใช้เพื่อยกเลิกการทำงาน async ที่ใช้เวลานาน

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class CancellationTokenExample
{
    static async Task Main()
    {
        // Pattern 1: Basic cancellation
        using var cts = new CancellationTokenSource();
        
        // ยกเลิกหลัง 2 วินาที
        cts.CancelAfter(2000);
        
        try
        {
            await LongOperationAsync(cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("การดำเนินการถูกยกเลิก");
        }
        
        // Pattern 2: Manual cancellation
        using var cts2 = new CancellationTokenSource();
        
        var task = ProcessDataAsync(cts2.Token);
        
        Console.WriteLine("กด Enter เพื่อยกเลิก...");
        // จำลองการกด Enter
        await Task.Delay(500);
        cts2.Cancel();
        
        try
        {
            await task;
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("ผู้ใช้ยกเลิกการดำเนินการ");
        }
        
        // Pattern 3: Linked cancellation tokens
        using var userCts = new CancellationTokenSource();
        using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
        using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
            userCts.Token, timeoutCts.Token
        );
        
        try
        {
            await DoWorkWithLinkedTokenAsync(linkedCts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine($"ยกเลิกเนื่องจาก: " +
                $"user={userCts.IsCancellationRequested}, " +
                $"timeout={timeoutCts.IsCancellationRequested}");
        }
    }
    
    static async Task LongOperationAsync(CancellationToken cancellationToken = default)
    {
        for (int i = 0; i < 10; i++)
        {
            // ตรวจสอบการยกเลิกก่อนทำงาน
            cancellationToken.ThrowIfCancellationRequested();
            
            Console.WriteLine($"ขั้นตอนที่ {i + 1}/10");
            await Task.Delay(500, cancellationToken);
        }
        
        Console.WriteLine("งานเสร็จสมบูรณ์");
    }
    
    static async Task ProcessDataAsync(CancellationToken cancellationToken)
    {
        int processed = 0;
        try
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                await Task.Delay(200, cancellationToken);
                processed++;
                Console.WriteLine($"ประมวลผลรายการที่ {processed}");
            }
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine($"ถูกยกเลิก ประมวลผลไป {processed} รายการ");
            throw;
        }
    }
    
    static async Task DoWorkWithLinkedTokenAsync(CancellationToken cancellationToken)
    {
        for (int i = 0; i < 100; i++)
        {
            cancellationToken.ThrowIfCancellationRequested();
            await Task.Delay(200, cancellationToken);
            Console.WriteLine($"ทำงาน {i + 1}");
        }
    }
}
```

### CancellationToken ใน ASP.NET Core

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Threading;
using System.Threading.Tasks;

[ApiController]
[Route("api/[controller]")]
public class DataController : ControllerBase
{
    private readonly IDataService _dataService;
    
    public DataController(IDataService dataService)
    {
        _dataService = dataService;
    }
    
    // CancellationToken จะถูก inject อัตโนมัติ
    // ถ้า client ยกเลิก request, token จะถูก cancel
    [HttpGet("{id}")]
    public async Task<IActionResult> GetData(int id, CancellationToken cancellationToken)
    {
        try
        {
            var data = await _dataService.GetDataAsync(id, cancellationToken);
            return Ok(data);
        }
        catch (OperationCanceledException)
        {
            // Client ยกเลิก request
            return StatusCode(499, "Client Closed Request");
        }
    }
}
```

---

## 6. async void ปัญหา

**async void ควรหลีกเลี่ยงยกเว้น event handlers!**

```csharp
using System;
using System.Threading.Tasks;

class AsyncVoidProblem
{
    static async Task Main()
    {
        // ปัญหา 1: ไม่สามารถ await ได้
        // AsyncVoidMethod(); // ไม่สามารถ await
        
        // ปัญหา 2: Exception ไม่สามารถ catch ได้
        try
        {
            AsyncVoidMethodWithException();
            await Task.Delay(500); // รอให้ method ทำงาน
        }
        catch (Exception ex)
        {
            // จะไม่ถูก catch!
            Console.WriteLine($"ไม่เคยถูก execute: {ex.Message}");
        }
        
        // ปัญหา 3: ไม่รู้ว่าเสร็จเมื่อไหร่
        AsyncVoidMethod();
        Console.WriteLine("อาจ print ก่อน AsyncVoidMethod เสร็จ");
        
        await Task.Delay(1000);
    }
    
    // ❌ ไม่ควรทำ
    static async void AsyncVoidMethod()
    {
        await Task.Delay(500);
        Console.WriteLine("AsyncVoid เสร็จ (แต่ไม่มีใครรู้)");
    }
    
    // ❌ อันตราย - exception จะทำให้ app crash
    static async void AsyncVoidMethodWithException()
    {
        await Task.Delay(100);
        throw new Exception("Exception จาก async void!");
    }
    
    // ✅ ควรใช้ async Task แทน
    static async Task BetterMethodAsync()
    {
        await Task.Delay(500);
        Console.WriteLine("Better method เสร็จ");
    }
    
    // ✅ async void ใช้ได้แค่กับ event handlers
    static async void Button_Click(object sender, EventArgs e)
    {
        try
        {
            // ต้องมี try-catch ใน async void event handlers
            await DoSomethingAsync();
        }
        catch (Exception ex)
        {
            // Handle error
            Console.WriteLine($"Error: {ex.Message}");
        }
    }
    
    static async Task DoSomethingAsync()
    {
        await Task.Delay(100);
    }
}
```

---

## 7. ConfigureAwait

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

class ConfigureAwaitExample
{
    // ConfigureAwait(true) = default
    // หลัง await ต้องกลับมา SynchronizationContext เดิม (UI thread)
    // ใช้ใน: UI code, code ที่ต้องการ context เดิม

    // ConfigureAwait(false)
    // หลัง await ไม่ต้องกลับ context เดิม
    // ใช้ใน: Library code, ไม่ต้องการ context เดิม (เร็วกว่า)
    
    static async Task Main()
    {
        Console.WriteLine($"Thread ก่อน: {Thread.CurrentThread.ManagedThreadId}");
        
        // ConfigureAwait(true) - default behavior
        await Task.Delay(100).ConfigureAwait(true);
        Console.WriteLine($"Thread หลัง ConfigureAwait(true): {Thread.CurrentThread.ManagedThreadId}");
        
        // ConfigureAwait(false) - ไม่ต้องกลับ context
        await Task.Delay(100).ConfigureAwait(false);
        Console.WriteLine($"Thread หลัง ConfigureAwait(false): {Thread.CurrentThread.ManagedThreadId}");
    }
    
    // Library code - ควรใช้ ConfigureAwait(false)
    public static async Task<string> LibraryMethodAsync()
    {
        var result = await FetchFromDatabaseAsync().ConfigureAwait(false);
        var processed = await ProcessDataAsync(result).ConfigureAwait(false);
        return processed;
    }
    
    // Application code - ปกติไม่ต้องระบุ (default = true)
    public static async Task ApplicationMethodAsync()
    {
        // ใน WPF/WinForms UI thread จะถูกรักษาไว้
        var data = await GetDataAsync();
        // UpdateUI(data); // ปลอดภัยเพราะอยู่ใน UI thread
    }
    
    static async Task<string> FetchFromDatabaseAsync()
    {
        await Task.Delay(100).ConfigureAwait(false);
        return "database data";
    }
    
    static async Task<string> ProcessDataAsync(string data)
    {
        await Task.Delay(50).ConfigureAwait(false);
        return data.ToUpper();
    }
    
    static async Task<string> GetDataAsync()
    {
        await Task.Delay(100);
        return "application data";
    }
}
```

---

## 8. โปรแกรมตัวอย่าง: Async HTTP Requests

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading;
using System.Threading.Tasks;

// Models
public record Post(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("userId")] int UserId,
    [property: JsonPropertyName("title")] string Title,
    [property: JsonPropertyName("body")] string Body
);

public record User(
    [property: JsonPropertyName("id")] int Id,
    [property: JsonPropertyName("name")] string Name,
    [property: JsonPropertyName("email")] string Email
);

// HTTP Service
public class JsonPlaceholderService : IDisposable
{
    private readonly HttpClient _httpClient;
    private readonly string _baseUrl = "https://jsonplaceholder.typicode.com";
    
    public JsonPlaceholderService()
    {
        _httpClient = new HttpClient
        {
            BaseAddress = new Uri(_baseUrl),
            Timeout = TimeSpan.FromSeconds(30)
        };
    }
    
    // GET single post
    public async Task<Post?> GetPostAsync(int id, CancellationToken cancellationToken = default)
    {
        var response = await _httpClient
            .GetAsync($"/posts/{id}", cancellationToken)
            .ConfigureAwait(false);
        
        response.EnsureSuccessStatusCode();
        return await response.Content
            .ReadFromJsonAsync<Post>(cancellationToken: cancellationToken)
            .ConfigureAwait(false);
    }
    
    // GET all posts
    public async Task<List<Post>> GetPostsAsync(CancellationToken cancellationToken = default)
    {
        var posts = await _httpClient
            .GetFromJsonAsync<List<Post>>("/posts", cancellationToken)
            .ConfigureAwait(false);
        
        return posts ?? new List<Post>();
    }
    
    // GET user
    public async Task<User?> GetUserAsync(int id, CancellationToken cancellationToken = default)
    {
        return await _httpClient
            .GetFromJsonAsync<User>($"/users/{id}", cancellationToken)
            .ConfigureAwait(false);
    }
    
    // GET posts by user
    public async Task<List<Post>> GetPostsByUserAsync(int userId, CancellationToken cancellationToken = default)
    {
        var posts = await _httpClient
            .GetFromJsonAsync<List<Post>>($"/posts?userId={userId}", cancellationToken)
            .ConfigureAwait(false);
        
        return posts ?? new List<Post>();
    }
    
    public void Dispose() => _httpClient.Dispose();
}

// Main Program
class Program
{
    static async Task Main(string[] args)
    {
        using var service = new JsonPlaceholderService();
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
        
        Console.WriteLine("=== Async HTTP Requests Demo ===\n");
        
        try
        {
            // Demo 1: Sequential requests (ช้า)
            await Demo1_SequentialRequests(service, cts.Token);
            
            // Demo 2: Parallel requests (เร็ว)
            await Demo2_ParallelRequests(service, cts.Token);
            
            // Demo 3: WhenAny - รับผลแรกที่เสร็จ
            await Demo3_WhenAny(service, cts.Token);
            
            // Demo 4: Rate limiting
            await Demo4_RateLimiting(service, cts.Token);
            
            // Demo 5: Retry pattern
            await Demo5_RetryPattern(service, cts.Token);
        }
        catch (OperationCanceledException)
        {
            Console.WriteLine("\nการดำเนินการถูกยกเลิก");
        }
        catch (HttpRequestException ex)
        {
            Console.WriteLine($"\nHTTP Error: {ex.Message}");
        }
    }
    
    static async Task Demo1_SequentialRequests(JsonPlaceholderService service, CancellationToken ct)
    {
        Console.WriteLine("--- Demo 1: Sequential Requests ---");
        var sw = Stopwatch.StartNew();
        
        for (int i = 1; i <= 3; i++)
        {
            var post = await service.GetPostAsync(i, ct);
            Console.WriteLine($"Post {i}: {post?.Title?[..Math.Min(40, post.Title.Length)]}...");
        }
        
        Console.WriteLine($"ใช้เวลา: {sw.ElapsedMilliseconds}ms\n");
    }
    
    static async Task Demo2_ParallelRequests(JsonPlaceholderService service, CancellationToken ct)
    {
        Console.WriteLine("--- Demo 2: Parallel Requests ---");
        var sw = Stopwatch.StartNew();
        
        // ดึงข้อมูลพร้อมกัน
        var postTasks = Enumerable.Range(1, 5)
            .Select(i => service.GetPostAsync(i, ct));
        
        var posts = await Task.WhenAll(postTasks);
        
        foreach (var post in posts.Where(p => p != null))
        {
            Console.WriteLine($"Post {post!.Id}: {post.Title[..Math.Min(40, post.Title.Length)]}...");
        }
        
        Console.WriteLine($"ใช้เวลา: {sw.ElapsedMilliseconds}ms\n");
    }
    
    static async Task Demo3_WhenAny(JsonPlaceholderService service, CancellationToken ct)
    {
        Console.WriteLine("--- Demo 3: WhenAny - First Response ---");
        
        // ดึงจากหลาย endpoint แล้วเอาที่เร็วที่สุด
        var tasks = new[]
        {
            service.GetPostAsync(1, ct),
            service.GetPostAsync(2, ct),
            service.GetPostAsync(3, ct)
        };
        
        var firstCompleted = await Task.WhenAny(tasks);
        var firstPost = await firstCompleted;
        Console.WriteLine($"ได้ผลแรก - Post {firstPost?.Id}: {firstPost?.Title}");
        Console.WriteLine();
    }
    
    static async Task Demo4_RateLimiting(JsonPlaceholderService service, CancellationToken ct)
    {
        Console.WriteLine("--- Demo 4: Rate Limiting with SemaphoreSlim ---");
        
        // จำกัดให้ request พร้อมกันได้ไม่เกิน 3 ตัว
        using var semaphore = new SemaphoreSlim(3, 3);
        var sw = Stopwatch.StartNew();
        
        var tasks = Enumerable.Range(1, 10).Select(async id =>
        {
            await semaphore.WaitAsync(ct);
            try
            {
                var post = await service.GetPostAsync(id, ct);
                Console.WriteLine($"  Post {id} ดึงข้อมูลเสร็จ");
                return post;
            }
            finally
            {
                semaphore.Release();
            }
        });
        
        var results = await Task.WhenAll(tasks);
        Console.WriteLine($"ดึงข้อมูลสำเร็จ {results.Length} รายการ ใน {sw.ElapsedMilliseconds}ms\n");
    }
    
    static async Task Demo5_RetryPattern(JsonPlaceholderService service, CancellationToken ct)
    {
        Console.WriteLine("--- Demo 5: Retry Pattern ---");
        
        var post = await RetryAsync(
            () => service.GetPostAsync(1, ct),
            maxRetries: 3,
            delay: TimeSpan.FromSeconds(1)
        );
        
        Console.WriteLine($"ดึงข้อมูลสำเร็จ: Post {post?.Id}");
        Console.WriteLine();
    }
    
    static async Task<T> RetryAsync<T>(
        Func<Task<T>> operation, 
        int maxRetries, 
        TimeSpan delay,
        CancellationToken ct = default)
    {
        Exception? lastException = null;
        
        for (int attempt = 1; attempt <= maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (!(ex is OperationCanceledException))
            {
                lastException = ex;
                Console.WriteLine($"  พยายามครั้งที่ {attempt}/{maxRetries} ล้มเหลว: {ex.Message}");
                
                if (attempt < maxRetries)
                {
                    await Task.Delay(delay * attempt, ct); // Exponential backoff
                }
            }
        }
        
        throw new Exception($"ล้มเหลวหลังจากพยายาม {maxRetries} ครั้ง", lastException);
    }
}
```

---

## Exercises

### Exercise 1: Async File Processor
สร้างโปรแกรมที่อ่านไฟล์หลายไฟล์พร้อมกัน นับจำนวนคำ และแสดงผลรวม

```csharp
// TODO: ให้ implement ฟังก์ชันต่อไปนี้
public static async Task<Dictionary<string, int>> CountWordsInFilesAsync(
    IEnumerable<string> filePaths,
    CancellationToken cancellationToken = default)
{
    // 1. ใช้ Task.WhenAll เพื่ออ่านไฟล์ทุกไฟล์พร้อมกัน
    // 2. สำหรับแต่ละไฟล์ นับจำนวนคำ
    // 3. Return Dictionary<filename, wordCount>
    throw new NotImplementedException();
}

// Expected behavior:
// - อ่านไฟล์ทุกไฟล์พร้อมกัน (parallel)
// - ถ้า file ไม่มีอยู่ ให้ข้ามและ log warning
// - Support cancellation
```

### Exercise 2: Download Manager
สร้าง download manager ที่ download ไฟล์หลายไฟล์พร้อมกัน พร้อม progress tracking

```csharp
public class DownloadManager
{
    // TODO: Implement download method
    // - Download ไฟล์หลายไฟล์พร้อมกัน
    // - จำกัด concurrent downloads ด้วย SemaphoreSlim
    // - Report progress สำหรับแต่ละไฟล์
    // - Support cancellation
    public async Task DownloadFilesAsync(
        IEnumerable<(string url, string savePath)> files,
        int maxConcurrent = 3,
        IProgress<(string url, int percent)>? progress = null,
        CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: Async Cache
สร้าง simple async cache ที่ cache ผลลัพธ์จาก async operations

```csharp
public class AsyncCache<TKey, TValue> where TKey : notnull
{
    // TODO: Implement
    // - GetOrCreateAsync: ถ้ามีใน cache return เลย ถ้าไม่มีเรียก factory
    // - ป้องกัน thundering herd problem (คำขอเดียวกันหลายอัน)
    // - Support expiration
    // - Thread-safe
    
    public async Task<TValue> GetOrCreateAsync(
        TKey key,
        Func<TKey, CancellationToken, Task<TValue>> factory,
        TimeSpan? expiration = null,
        CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException();
    }
}
```

---

## สรุป

✅ **Async/Await** ช่วยให้เขียน asynchronous code ได้ง่ายขึ้นโดยไม่ต้อง callback หรือ manual thread management

✅ **Task** แทน asynchronous operation, **Task\<T\>** แทน operation ที่ return ค่า

✅ **Task.WhenAll** รอ tasks ทั้งหมดพร้อมกัน, **Task.WhenAny** รอ task แรกที่เสร็จ

✅ **CancellationToken** ใช้ยกเลิกการทำงาน async สำคัญมากสำหรับ web APIs

✅ **async void ควรหลีกเลี่ยง** เพราะ exceptions ไม่สามารถ catch ได้และไม่สามารถ await ได้

✅ **ConfigureAwait(false)** ใช้ใน library code เพื่อ performance

✅ **SemaphoreSlim** ใช้จำกัด concurrent async operations

✅ ใช้ **Retry pattern** สำหรับ transient errors ใน HTTP requests

---

## Part ถัดไป

➡️ **Part 032**: Task Parallel Library (TPL) - Parallel.For, Parallel.ForEach, PLINQ และการจัดการ thread safety

---

*Part 031/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
