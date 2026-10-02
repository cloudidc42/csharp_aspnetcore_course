# Part 099: Interview Preparation & Career Guide

## เนื้อหาใน Part นี้
- Technical interview topics สำหรับ C#/.NET
- คำถามที่พบบ่อยพร้อมคำตอบ
- LeetCode patterns ใน C#
- System Design สำหรับ .NET developers
- Portfolio tips
- Career roadmap

---

## 1. C# Core Interview Questions

### Heap vs Stack
```csharp
// Stack: value types, method frames, เร็ว, auto-cleanup
int x = 10;  // stack
struct Point { public int X, Y; }
var p = new Point { X = 1, Y = 2 };  // stack

// Heap: reference types, GC จัดการ
class Person { public string Name = ""; }
var person = new Person();  // reference on stack, object on heap

// Boxing: value type → reference type (ช้า! สร้าง allocation)
int num = 42;
object boxed = num;   // boxing - allocates on heap
int unboxed = (int)boxed;  // unboxing

// คำถาม: ทำไม boxing ถึงช้า?
// คำตอบ: ต้อง allocate heap memory, copy value, 
//         และ GC ต้องจัดการ object ที่ boxed ด้วย
```

### Value Types vs Reference Types
```csharp
// Value Type: struct, enum, built-in types (int, double, bool)
// - copied when assigned
// - no null (ยกเว้น Nullable<T>)
// - cannot inherit (ยกเว้น interface)

struct Temperature
{
    public double Celsius { get; init; }
    public double Fahrenheit => Celsius * 9 / 5 + 32;
}

Temperature t1 = new Temperature { Celsius = 25 };
Temperature t2 = t1;  // copy
t2 = t2 with { Celsius = 30 };  // t1 ยังเป็น 25

// Reference Type: class, interface, delegate, record class
// - share reference when assigned
// - can be null
// - supports inheritance

class Box { public int Width; }
var b1 = new Box { Width = 10 };
var b2 = b1;  // same reference!
b2.Width = 20;
Console.WriteLine(b1.Width);  // 20 !!
```

### async/await ทำงานอย่างไร
```csharp
// async/await เป็น syntactic sugar เหนือ Task<T>
// ไม่ได้สร้าง thread ใหม่! ใช้ thread pool อย่างมีประสิทธิภาพ

public async Task<string> FetchDataAsync(string url)
{
    // Thread pool thread รัน code จนถึง await
    var client = new HttpClient();
    
    // await: หยุดรอ I/O โดยไม่บล็อก thread
    // thread pool thread ถูก release ให้งานอื่นใช้
    var response = await client.GetAsync(url);  
    
    // เมื่อ I/O เสร็จ: ได้รับ thread กลับมาทำงานต่อ
    return await response.Content.ReadAsStringAsync();
}

// State Machine ที่ compiler สร้างให้:
// - IAsyncStateMachine
// - MoveNext() จัดการ state transitions
// - SynchronizationContext / TaskScheduler กำหนด thread resumption

// ConfigureAwait(false): ไม่ capture context
// ใช้ใน library code เพื่อประสิทธิภาพ
public async Task<string> LibraryMethod()
{
    var data = await GetDataAsync().ConfigureAwait(false);
    return data.ToString();
}

// Deadlock scenario
// ❌ อย่าทำ:
public string GetDataSync()
{
    return GetDataAsync().Result;  // deadlock ใน ASP.NET classic!
}

// ✅ ทำแบบนี้:
public async Task<string> GetDataAsync2()
{
    return await GetDataAsync();
}
```

### LINQ Deferred Execution
```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Deferred: query ไม่รันจนกว่าจะ enumerate
var query = numbers.Where(n => n > 2).Select(n => n * 2);

numbers.Add(6);  // เพิ่มหลัง define query

foreach (var n in query)  // รันตอนนี้ - รวม 6 ด้วย!
    Console.WriteLine(n);  // 6, 8, 10, 12

// Immediate: รันทันที
var result = numbers.Where(n => n > 2).ToList();  // รันทันที

// LINQ ต่างๆ:
// Deferred: Where, Select, OrderBy, GroupBy, Join, Take, Skip
// Immediate: ToList, ToArray, Count, First, Single, Any, All, Sum, Max
```

---

## 2. OOP & Design Pattern Questions

### SOLID Principles
```csharp
// S - Single Responsibility Principle
// ❌ ไม่ดี: class ทำหลายอย่าง
class UserManager
{
    public User GetUser(int id) { /* ... */ return null!; }
    public void SaveUser(User user) { /* ... */ }
    public void SendWelcomeEmail(User user) { /* ... */ }  // ควรแยก!
    public string GenerateReport() { /* ... */ return ""; }  // ควรแยก!
}

// ✅ ดี: แยก responsibilities
class UserRepository { public User? Get(int id) { return null; } }
class EmailService { public void SendWelcome(User user) { } }
class UserReportService { public string Generate() { return ""; } }

// O - Open/Closed Principle
// Open for extension, Closed for modification
public abstract class ShippingCalculator
{
    public abstract decimal Calculate(Order order);
}

public class StandardShipping : ShippingCalculator
{
    public override decimal Calculate(Order order) => 50m;
}

public class ExpressShipping : ShippingCalculator
{
    public override decimal Calculate(Order order) => 150m;
}
// เพิ่ม shipping method ใหม่โดยไม่แก้ existing code

// D - Dependency Inversion Principle
// ❌ ไม่ดี: depend on concrete
class OrderService
{
    private readonly SqlOrderRepo _repo = new();  // concrete dependency!
}

// ✅ ดี: depend on abstraction
class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

### Repository vs Service Layer
```csharp
// Repository Pattern: abstract data access
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Product>> GetAllAsync(CancellationToken ct = default);
    Task SaveAsync(Product product, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}

// Service Layer: business logic (uses repositories)
public class ProductService
{
    private readonly IProductRepository _repo;
    private readonly ICacheService _cache;

    public ProductService(IProductRepository repo, ICacheService cache)
    {
        _repo = repo;
        _cache = cache;
    }

    public async Task<ProductDto?> GetProductAsync(Guid id)
    {
        // Business logic: check cache first
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetAsync<ProductDto>(cacheKey);
        if (cached != null) return cached;

        var product = await _repo.GetByIdAsync(id);
        if (product == null) return null;

        var dto = new ProductDto(product.Id, product.Name, product.Price);
        await _cache.SetAsync(cacheKey, dto, TimeSpan.FromMinutes(5));
        return dto;
    }
}
```

---

## 3. ASP.NET Core Questions

### Middleware Pipeline
```csharp
// Middleware รันตามลำดับ (pipeline)
var app = builder.Build();

// ลำดับสำคัญมาก!
app.UseExceptionHandler("/error");  // 1. ต้องมาก่อน
app.UseHsts();                       // 2. Security headers
app.UseHttpsRedirection();           // 3. HTTPS redirect
app.UseStaticFiles();                // 4. Serve static files
app.UseRouting();                    // 5. Route matching
app.UseCors();                       // 6. CORS (after routing)
app.UseAuthentication();             // 7. Who are you?
app.UseAuthorization();              // 8. What can you do?
app.UseRateLimiter();                // 9. Rate limiting
// app.MapControllers();             // 10. Endpoint execution

// Custom middleware
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    
    public RequestTimingMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext ctx)
    {
        var sw = Stopwatch.StartNew();
        
        await _next(ctx);  // "next" ใน pipeline
        
        sw.Stop();
        ctx.Response.Headers["X-Response-Time"] = $"{sw.ElapsedMilliseconds}ms";
    }
}
```

### DI Lifetimes
```csharp
// Singleton: หนึ่ง instance ตลอด app lifetime
// - ใช้กับ stateless services, caching, config
services.AddSingleton<IMemoryCache, MemoryCache>();

// Scoped: หนึ่ง instance ต่อ HTTP request
// - ใช้กับ DbContext, Unit of Work, current user
services.AddScoped<AppDbContext>();
services.AddScoped<ICurrentUserService, CurrentUserService>();

// Transient: new instance ทุกครั้งที่ inject
// - ใช้กับ lightweight, stateless services
services.AddTransient<IEmailFormatter, HtmlEmailFormatter>();

// ⚠️ Captive Dependency: inject Scoped ใน Singleton = BUG
// services.AddSingleton<OrderService>();  // ถ้า OrderService inject Scoped = error!
// แก้โดยใช้ IServiceScopeFactory:
public class BackgroundWorker(IServiceScopeFactory scopeFactory)
{
    public async Task DoWorkAsync()
    {
        using var scope = scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // ใช้ dbContext ได้อย่างปลอดภัย
    }
}
```

---

## 4. LeetCode Patterns ใน C#

### Two Pointers
```csharp
// Two Sum II (sorted array)
public static int[] TwoSum(int[] numbers, int target)
{
    int left = 0, right = numbers.Length - 1;
    
    while (left < right)
    {
        int sum = numbers[left] + numbers[right];
        if (sum == target) return [left + 1, right + 1];
        else if (sum < target) left++;
        else right--;
    }
    
    return [];
}

// Valid Palindrome
public static bool IsPalindrome(string s)
{
    int left = 0, right = s.Length - 1;
    
    while (left < right)
    {
        while (left < right && !char.IsLetterOrDigit(s[left])) left++;
        while (left < right && !char.IsLetterOrDigit(s[right])) right--;
        
        if (char.ToLower(s[left]) != char.ToLower(s[right])) return false;
        left++;
        right--;
    }
    
    return true;
}
```

### Sliding Window
```csharp
// Longest Substring Without Repeating Characters
public static int LengthOfLongestSubstring(string s)
{
    var seen = new Dictionary<char, int>();
    int maxLen = 0, left = 0;
    
    for (int right = 0; right < s.Length; right++)
    {
        char c = s[right];
        if (seen.TryGetValue(c, out var prevIndex) && prevIndex >= left)
            left = prevIndex + 1;
        
        seen[c] = right;
        maxLen = Math.Max(maxLen, right - left + 1);
    }
    
    return maxLen;
}

// Maximum Sum Subarray of Size K
public static int MaxSumSubarray(int[] nums, int k)
{
    if (nums.Length < k) return 0;
    
    int windowSum = nums[..k].Sum();
    int maxSum = windowSum;
    
    for (int i = k; i < nums.Length; i++)
    {
        windowSum += nums[i] - nums[i - k];
        maxSum = Math.Max(maxSum, windowSum);
    }
    
    return maxSum;
}
```

### BFS / DFS ด้วย Queue/Stack
```csharp
// Binary Tree Level Order Traversal (BFS)
public static IList<IList<int>> LevelOrder(TreeNode? root)
{
    var result = new List<IList<int>>();
    if (root == null) return result;
    
    var queue = new Queue<TreeNode>();
    queue.Enqueue(root);
    
    while (queue.Count > 0)
    {
        var level = new List<int>();
        int levelSize = queue.Count;
        
        for (int i = 0; i < levelSize; i++)
        {
            var node = queue.Dequeue();
            level.Add(node.Val);
            if (node.Left != null) queue.Enqueue(node.Left);
            if (node.Right != null) queue.Enqueue(node.Right);
        }
        
        result.Add(level);
    }
    
    return result;
}

// Number of Islands (DFS)
public static int NumIslands(char[][] grid)
{
    int count = 0;
    int rows = grid.Length, cols = grid[0].Length;
    
    void DFS(int r, int c)
    {
        if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1')
            return;
        
        grid[r][c] = '0';  // mark visited
        DFS(r + 1, c);
        DFS(r - 1, c);
        DFS(r, c + 1);
        DFS(r, c - 1);
    }
    
    for (int r = 0; r < rows; r++)
        for (int c = 0; c < cols; c++)
            if (grid[r][c] == '1') { DFS(r, c); count++; }
    
    return count;
}

public class TreeNode(int val, TreeNode? left = null, TreeNode? right = null)
{
    public int Val = val;
    public TreeNode? Left = left;
    public TreeNode? Right = right;
}
```

### Dynamic Programming
```csharp
// Fibonacci (bottom-up DP)
public static long Fibonacci(int n)
{
    if (n <= 1) return n;
    
    long prev = 0, curr = 1;
    for (int i = 2; i <= n; i++)
    {
        long next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}

// 0/1 Knapsack
public static int Knapsack(int[] weights, int[] values, int capacity)
{
    int n = weights.Length;
    var dp = new int[n + 1, capacity + 1];
    
    for (int i = 1; i <= n; i++)
    {
        for (int w = 0; w <= capacity; w++)
        {
            dp[i, w] = dp[i - 1, w];
            if (weights[i - 1] <= w)
                dp[i, w] = Math.Max(dp[i, w], dp[i - 1, w - weights[i - 1]] + values[i - 1]);
        }
    }
    
    return dp[n, capacity];
}

// Longest Common Subsequence
public static int LCS(string text1, string text2)
{
    int m = text1.Length, n = text2.Length;
    var dp = new int[m + 1, n + 1];
    
    for (int i = 1; i <= m; i++)
        for (int j = 1; j <= n; j++)
            dp[i, j] = text1[i - 1] == text2[j - 1]
                ? dp[i - 1, j - 1] + 1
                : Math.Max(dp[i - 1, j], dp[i, j - 1]);
    
    return dp[m, n];
}
```

---

## 5. System Design Questions

### Design URL Shortener
```
คำถาม: ออกแบบ URL shortener เช่น bit.ly

ขั้นตอนการตอบ:
1. Clarify requirements
   - Short URL format (6 chars base62 = 56 billion unique URLs)
   - Expiry support? Custom aliases?
   - Read-heavy (100:1 read/write ratio)
   - Scale: 100M URLs, 10B redirects/day

2. API Design
   POST /api/shorten   { url, customAlias?, expiresAt? } → { shortCode }
   GET  /{shortCode}   → 301/302 redirect

3. Database Choice
   - PostgreSQL (URLs metadata + analytics)
   - Redis (cache hot URLs - LRU cache)

4. Architecture
   Client → CDN → API Servers → Redis Cache → PostgreSQL
   
   Short Code Generation:
   - Random 6-char base62 string
   - Check collision in DB
   - Or: Auto-increment ID → base62 encode

5. Scale considerations
   - Read heavy → cache aggressively (Redis)
   - DB read replicas for redirects
   - CDN for most popular URLs
   - Horizontal scaling API servers
```

### Design API Rate Limiter
```
ออกแบบ Rate Limiter สำหรับ API

Algorithms:
1. Token Bucket: tokens refill at rate R, max N tokens
   - Allows bursts
   - Smooth average rate

2. Sliding Window Log: track request timestamps
   - Exact count, memory-intensive

3. Sliding Window Counter: combine fixed + sliding windows
   - Good balance

Implementation ด้วย Redis:
- key: "rate:{userId}:{window}"
- value: count
- TTL: window duration

// Pseudocode:
key = "rate:{userId}:{minute}"
count = INCR key
if count == 1: EXPIRE key 60
if count > limit: reject with 429

// .NET Implementation:
// ASP.NET Core Rate Limiting (Part 088 ครอบคลุมแล้ว)
```

---

## 6. Portfolio Tips

### โปรเจค Portfolio ที่แนะนำ
```
1. E-commerce API (Clean Architecture)
   ✅ เราสร้างใน Part 093
   - แสดง: Domain modeling, CQRS, Repository Pattern
   - เพิ่ม: Tests, CI/CD pipeline, Swagger docs

2. Social Platform API
   ✅ เราสร้างใน Part 094  
   - แสดง: Real-time (SignalR), Feed algorithm
   - เพิ่ม: Rate limiting, Auth, Performance benchmarks

3. SaaS Starter
   ✅ เราสร้างใน Part 095
   - แสดง: Multi-tenancy, Feature flags
   - เพิ่ม: Stripe integration, Email templates

4. Microservices Demo
   - แสดง: Service discovery, Message queue, Docker
   - เครื่องมือ: Docker Compose, RabbitMQ, YARP

5. Open Source Contribution
   - Contribute to .NET ecosystem
   - ASP.NET Core, EF Core, MassTransit, etc.
```

### GitHub Profile Best Practices
```
1. Professional README
   - Name, Title, Skills
   - Featured projects with screenshots
   - Contact links

2. Pinned Repositories
   - Best 6 projects
   - Include README per project

3. README per project:
   - ปัญหาที่แก้
   - Tech stack
   - Architecture diagram
   - How to run
   - Live demo link (ถ้ามี)

4. Code Quality Indicators:
   - Tests (code coverage badge)
   - CI badge (GitHub Actions)
   - Clean commit history
   - Consistent code style
```

---

## 7. ตัวอย่างคำถาม Behavioral

```
STAR Method: Situation, Task, Action, Result

"Tell me about a challenging bug you fixed"
Situation: Production system หยุดทำงาน 30 นาที
Task: ต้องหา root cause และ fix ด่วน
Action: 
  - ตรวจ logs พบ memory leak
  - Profile ด้วย dotnet-dump
  - พบ event handler ที่ไม่ unsubscribe
  - Deploy hotfix
Result: System กลับมาทำงาน downtime รวม 45 นาที
        เพิ่ม alerting เพื่อป้องกันในอนาคต

"Describe a time you improved system performance"
Situation: API response time > 3 วินาที
Task: ลด latency ให้ < 500ms
Action:
  - Profile ด้วย MiniProfiler
  - พบ N+1 query problem
  - Fix ด้วย Include() และ projection
  - เพิ่ม Redis cache
  - ลด DB queries จาก 50 เป็น 3
Result: P99 latency ลดจาก 3.2s → 180ms (17x improvement)
```

---

## Exercises / Project Tasks

### Exercise 1: Mock Interview Practice
ฝึกตอบคำถาม (ตั้งเวลา 3 นาทีต่อข้อ):
1. อธิบาย async/await ให้ junior developer ฟัง
2. ต่างกันอย่างไร interface vs abstract class
3. อธิบาย dependency injection
4. เมื่อไหรใช้ struct vs class

### Exercise 2: Solve LeetCode
แก้ปัญหาเหล่านี้ใน C#:
- Two Sum (Easy)
- Longest Substring Without Repeating Characters (Medium)
- Valid Parentheses (Easy)
- Merge Two Sorted Lists (Easy)
- Maximum Depth of Binary Tree (Easy)

### Exercise 3: System Design Practice
ออกแบบระบบ:
- Design a Notification System
- Design Rate Limiter
- Design a Cache System

### Exercise 4: Portfolio Project
เลือกหนึ่งโปรเจคจาก course นี้:
- Deploy ขึ้น Azure/Railway/Render
- เพิ่ม CI/CD pipeline
- เขียน README สมบูรณ์
- Demo video 2-3 นาที

---

## สรุป

### Cheat Sheet: สิ่งที่ต้องรู้สำหรับ .NET Interview

**C# Fundamentals:**
- Value types vs Reference types
- async/await และ Task
- LINQ (deferred vs immediate)
- Generics และ constraints
- Delegates, events, lambda

**OOP & Design:**
- SOLID Principles
- Common Design Patterns (Factory, Repository, Strategy, Observer)
- Clean Architecture layers

**ASP.NET Core:**
- Middleware pipeline
- DI lifetimes
- Authentication/Authorization
- Minimal API vs Controllers

**Database:**
- EF Core: migrations, relationships, querying
- N+1 problem
- Transactions
- Index strategy

**Performance:**
- Profiling tools
- Common bottlenecks
- Caching strategies

**Architecture:**
- Microservices vs Monolith
- Message Queue
- CQRS
- Event-driven architecture

---

## Part ถัดไป

**Part 100: Course Summary and Next Steps** - สรุปหลักสูตรและก้าวต่อไป

---

*Part 099/100 | Phase 7/7: ระดับโลก | หลักสูตร C# และ ASP.NET Core*
