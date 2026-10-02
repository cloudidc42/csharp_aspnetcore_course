# Part 040: IDisposable และ Resource Management

## เนื้อหาใน Part นี้
- IDisposable interface
- using statement
- using declaration (C# 8+)
- Dispose pattern
- Finalizers
- SafeHandle
- โปรแกรมตัวอย่าง: Database connection manager

---

## 1. IDisposable Interface

IDisposable ใช้สำหรับ cleanup unmanaged resources และ managed resources ที่ต้องการ release โดย explicit

```csharp
using System;
using System.IO;

// IDisposable ง่ายๆ
public class SimpleResource : IDisposable
{
    private bool _disposed = false;
    
    public void DoWork()
    {
        ThrowIfDisposed();
        Console.WriteLine("ทำงาน...");
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            Console.WriteLine("Disposing SimpleResource");
            _disposed = true;
        }
    }
    
    protected void ThrowIfDisposed()
    {
        if (_disposed)
            throw new ObjectDisposedException(GetType().Name);
    }
}

// ทำไมต้องใช้ IDisposable
class WhyDisposable
{
    static void WithoutDispose()
    {
        // ❌ FileStream ไม่ถูก close
        var fs = new FileStream("test.txt", FileMode.CreateNew);
        fs.Write(new byte[] { 1, 2, 3 });
        // ลืม close! file handle ยังค้างอยู่
    }
    
    static void WithDispose()
    {
        // ✅ ปิด file ถูกต้อง
        var fs = new FileStream("test.txt", FileMode.CreateNew);
        try
        {
            fs.Write(new byte[] { 1, 2, 3 });
        }
        finally
        {
            fs.Dispose(); // ปิด file เสมอ
        }
    }
    
    static void WithUsing()
    {
        // ✅ ดีที่สุด - using statement
        using (var fs = new FileStream("test.txt", FileMode.CreateNew))
        {
            fs.Write(new byte[] { 1, 2, 3 });
        } // Dispose() ถูกเรียกอัตโนมัติ
    }
}
```

---

## 2. using Statement

```csharp
using System;
using System.IO;
using System.Net.Http;
using System.Threading.Tasks;

class UsingStatementExamples
{
    static void BasicUsing()
    {
        // using statement - Dispose เรียกอัตโนมัติเมื่อออก block
        using (var stream = new MemoryStream())
        {
            stream.WriteByte(42);
            Console.WriteLine($"Position: {stream.Position}");
        } // Dispose() ถูกเรียกที่นี่
        
        // Multiple using
        using (var reader = new StreamReader("file.txt"))
        using (var writer = new StreamWriter("output.txt"))
        {
            string? line;
            while ((line = reader.ReadLine()) != null)
            {
                writer.WriteLine(line.ToUpper());
            }
        }
        
        // using กับ exception
        try
        {
            using var conn = new System.Data.SqlClient.SqlConnection("bad connection");
            conn.Open(); // จะ throw exception
        }
        catch (Exception ex)
        {
            // conn.Dispose() ถูกเรียกก่อน catch นี้!
            Console.WriteLine($"Error: {ex.Message}");
        }
    }
    
    static async Task AsyncUsing()
    {
        // await using สำหรับ IAsyncDisposable
        await using var httpClient = new HttpClient();
        var response = await httpClient.GetAsync("https://api.example.com/data");
        Console.WriteLine($"Status: {response.StatusCode}");
    }
    
    static void NullCheckedUsing()
    {
        // using กับ nullable - ปลอดภัย
        IDisposable? resource = GetOptionalResource();
        using (resource)
        {
            resource?.DoWork(); // ปลอดภัยกับ null
        }
    }
    
    static IDisposable? GetOptionalResource() => null;
}

// Extension: using statement
static class DisposableExtensions
{
    // Helper สำหรับ using ด้วย action
    public static T Using<T>(this T disposable, Action<T> action) where T : IDisposable
    {
        using (disposable)
        {
            action(disposable);
        }
        return disposable;
    }
}
```

---

## 3. using Declaration (C# 8+)

```csharp
using System;
using System.IO;
using System.Text;

class UsingDeclarationExamples
{
    static string OldStyle(string filePath)
    {
        // ❌ แบบเก่า - nested blocks
        using (var fs = new FileStream(filePath, FileMode.Open))
        {
            using (var reader = new StreamReader(fs, Encoding.UTF8))
            {
                return reader.ReadToEnd();
            }
        }
    }
    
    static string NewStyle(string filePath)
    {
        // ✅ using declaration (C# 8+) - flat, ไม่ nested
        using var fs = new FileStream(filePath, FileMode.Open);
        using var reader = new StreamReader(fs, Encoding.UTF8);
        return reader.ReadToEnd();
        // Dispose เรียกตอนออก method scope (ย้อนกลับ: reader ก่อน, แล้ว fs)
    }
    
    static void MultipleResources()
    {
        using var conn = OpenConnection();
        using var cmd = conn.CreateCommand();
        using var reader = cmd.ExecuteReader();
        
        while (reader.Read())
        {
            Console.WriteLine(reader[0]);
        }
    } // reader, cmd, conn ถูก Dispose ตามลำดับ
    
    static void ConditionalDispose(bool useDatabase)
    {
        // using declaration ใน if block
        if (useDatabase)
        {
            using var conn = OpenConnection();
            conn.Execute("SELECT 1");
        } // conn ถูก Dispose ที่นี่
        
        Console.WriteLine("ยังคงทำงานต่อ");
    }
    
    // Dispose ในกรณีต่างๆ
    static void DisposeScopeDemo()
    {
        Console.WriteLine("เริ่ม method");
        
        using var r1 = new TrackableResource("R1");
        
        {
            using var r2 = new TrackableResource("R2");
            Console.WriteLine("ใน inner block");
        } // R2 ถูก Dispose ที่นี่
        
        using var r3 = new TrackableResource("R3");
        Console.WriteLine("จะออก method");
        
        // R3 แล้ว R1 ถูก Dispose เมื่อออก method
    }
    
    static dynamic OpenConnection() => throw new NotImplementedException();
}

class TrackableResource : IDisposable
{
    private readonly string _name;
    public TrackableResource(string name)
    {
        _name = name;
        Console.WriteLine($"  Created: {_name}");
    }
    public void Dispose() => Console.WriteLine($"  Disposed: {_name}");
}
```

---

## 4. Dispose Pattern (Complete)

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;
using System.Threading;

// Complete Dispose Pattern
public class ManagedAndUnmanagedResource : IDisposable
{
    // Managed resources
    private Stream? _managedStream;
    private Timer? _managedTimer;
    
    // Unmanaged resource (handle)
    private IntPtr _unmanagedHandle = IntPtr.Zero;
    
    // Track disposal
    private bool _disposed = false;
    
    public ManagedAndUnmanagedResource()
    {
        _managedStream = new MemoryStream();
        _managedTimer = new Timer(OnTimer, null, 1000, 1000);
        _unmanagedHandle = AllocateUnmanagedMemory(1024);
        Console.WriteLine("Resource created");
    }
    
    // Public Dispose - เรียกโดย user
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this); // ไม่ต้องเรียก finalizer
    }
    
    // Protected virtual Dispose - สำหรับ subclass
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                // ✅ Free managed resources
                _managedStream?.Dispose();
                _managedStream = null;
                
                _managedTimer?.Dispose();
                _managedTimer = null;
            }
            
            // ✅ Free unmanaged resources เสมอ (ไม่ว่า disposing จะเป็น true/false)
            if (_unmanagedHandle != IntPtr.Zero)
            {
                FreeUnmanagedMemory(_unmanagedHandle);
                _unmanagedHandle = IntPtr.Zero;
            }
            
            _disposed = true;
        }
    }
    
    // Finalizer - เรียกโดย GC เมื่อ user ลืม Dispose
    ~ManagedAndUnmanagedResource()
    {
        Console.WriteLine("WARNING: Finalizer called! User forgot to Dispose!");
        Dispose(disposing: false);
    }
    
    // Methods ต้องตรวจสอบ disposed
    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        Console.WriteLine("ทำงานอยู่...");
    }
    
    private void OnTimer(object? state)
    {
        if (!_disposed)
            Console.WriteLine("Timer tick");
    }
    
    // Simulate unmanaged operations
    private static IntPtr AllocateUnmanagedMemory(int size)
    {
        Console.WriteLine($"Allocated {size} bytes of unmanaged memory");
        return Marshal.AllocHGlobal(size);
    }
    
    private static void FreeUnmanagedMemory(IntPtr handle)
    {
        Console.WriteLine("Freed unmanaged memory");
        Marshal.FreeHGlobal(handle);
    }
}

// Subclass ของ Disposable
public class ExtendedResource : ManagedAndUnmanagedResource
{
    private Stream? _additionalStream;
    
    public ExtendedResource()
    {
        _additionalStream = new MemoryStream();
    }
    
    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _additionalStream?.Dispose();
            _additionalStream = null;
        }
        
        base.Dispose(disposing); // ต้องเรียก base!
    }
}
```

---

## 5. Finalizers

```csharp
using System;
using System.Runtime.InteropServices;

class FinalizerExamples
{
    // Finalizer ทำงานอย่างไร
    class ResourceWithFinalizer
    {
        private readonly string _name;
        
        public ResourceWithFinalizer(string name)
        {
            _name = name;
            Console.WriteLine($"  Created: {_name}");
        }
        
        ~ResourceWithFinalizer()
        {
            // Finalizer thread ของ GC เรียกนี่
            // ห้าม access managed objects ที่อาจถูก collect แล้ว!
            Console.WriteLine($"  Finalized: {_name}");
        }
    }
    
    static void DemonstrateFinalization()
    {
        {
            var r = new ResourceWithFinalizer("test");
            // r ออก scope แต่ยังไม่ถูก collect ทันที
        }
        
        Console.WriteLine("Object out of scope, not collected yet");
        
        GC.Collect();           // request GC
        GC.WaitForPendingFinalizers(); // รอ finalizers เสร็จ
        Console.WriteLine("After GC.Collect()");
    }
    
    // ❌ Finalizer ปัญหา: ช้ากว่า Dispose
    class SlowFinalizerProblem
    {
        ~SlowFinalizerProblem()
        {
            // object จะอยู่ใน finalization queue
            // GC ต้อง collect 2 รอบ (1 ตรวจสอบ, 2 finalize)
            // สิ้นเปลือง memory และ CPU
        }
    }
    
    // ✅ GC.SuppressFinalize หลัง Dispose
    class EfficientResource : IDisposable
    {
        ~EfficientResource() => Dispose(false);
        
        public void Dispose()
        {
            Dispose(true);
            GC.SuppressFinalize(this); // ไม่ต้องรอ finalizer
        }
        
        protected virtual void Dispose(bool disposing) { }
    }
}
```

---

## 6. SafeHandle

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;

// SafeHandle ใช้แทนการจัดการ IntPtr โดยตรง
public class CustomSafeHandle : SafeHandle
{
    public CustomSafeHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    
    public override bool IsInvalid => handle == IntPtr.Zero;
    
    protected override bool ReleaseHandle()
    {
        // Release the handle
        Console.WriteLine($"Releasing handle: {handle}");
        // CloseHandle(handle); // Windows API call
        return true;
    }
}

// SafeFileHandle - built-in
class SafeHandleExample
{
    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Auto)]
    static extern Microsoft.Win32.SafeHandles.SafeFileHandle CreateFile(
        string fileName,
        uint desiredAccess,
        uint shareMode,
        IntPtr securityAttributes,
        uint creationDisposition,
        uint flagsAndAttributes,
        IntPtr templateFile
    );
    
    static void SafeHandleDemo()
    {
        // SafeHandle ป้องกัน:
        // 1. Double-free
        // 2. Use-after-free  
        // 3. Handle leak
        
        using var handle = new CustomSafeHandle();
        
        if (!handle.IsInvalid)
        {
            Console.WriteLine("Handle is valid");
        }
    }
    
    // ใช้กับ FileStream
    static void FileStreamWithSafeHandle()
    {
        var handle = File.OpenHandle(
            "test.txt",
            FileMode.OpenOrCreate,
            FileAccess.ReadWrite,
            FileShare.None,
            FileOptions.Asynchronous
        );
        
        using (handle) // SafeFileHandle implements IDisposable
        {
            Console.WriteLine($"File handle: {handle.IsInvalid}");
        }
    }
}
```

---

## 7. IAsyncDisposable

```csharp
using System;
using System.IO;
using System.Threading.Tasks;

// IAsyncDisposable สำหรับ async cleanup
public class AsyncDatabaseConnection : IAsyncDisposable, IDisposable
{
    private bool _disposed = false;
    private Stream? _networkStream;
    
    public AsyncDatabaseConnection()
    {
        _networkStream = new MemoryStream();
        Console.WriteLine("Connection opened");
    }
    
    // Async cleanup
    public async ValueTask DisposeAsync()
    {
        await DisposeAsyncCore();
        
        // Free managed resources
        Dispose(disposing: false);
        GC.SuppressFinalize(this);
    }
    
    protected virtual async ValueTask DisposeAsyncCore()
    {
        if (_networkStream != null)
        {
            // Flush pending data อย่าง async
            await _networkStream.FlushAsync();
            await _networkStream.DisposeAsync();
            _networkStream = null;
            Console.WriteLine("Connection closed (async)");
        }
    }
    
    // Sync Dispose fallback
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                _networkStream?.Dispose();
                _networkStream = null;
                Console.WriteLine("Connection closed (sync)");
            }
            _disposed = true;
        }
    }
    
    public async Task ExecuteAsync(string query)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        await Task.Delay(100); // จำลอง
        Console.WriteLine($"Executed: {query}");
    }
}

class AsyncDisposableExample
{
    static async Task Main()
    {
        // await using สำหรับ IAsyncDisposable
        await using var conn = new AsyncDatabaseConnection();
        await conn.ExecuteAsync("SELECT 1");
        
        // await using ใน .NET 6+ สามารถใช้กับ IEnumerable ได้ด้วย
        await foreach (var item in GetItemsAsync())
        {
            Console.WriteLine($"Item: {item}");
        }
    }
    
    static async IAsyncEnumerable<int> GetItemsAsync()
    {
        for (int i = 0; i < 5; i++)
        {
            await Task.Delay(10);
            yield return i;
        }
    }
}
```

---

## 8. โปรแกรมตัวอย่าง: Database Connection Manager

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

// Simulated database connection
public class DbConnection : IDisposable
{
    private static int _idCounter = 0;
    public int Id { get; } = Interlocked.Increment(ref _idCounter);
    public string ConnectionString { get; }
    public bool IsOpen { get; private set; }
    private bool _disposed = false;
    
    public DbConnection(string connectionString)
    {
        ConnectionString = connectionString;
        Console.WriteLine($"  [Conn-{Id}] สร้างใหม่");
    }
    
    public void Open()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        Thread.Sleep(50); // จำลอง connection time
        IsOpen = true;
        Console.WriteLine($"  [Conn-{Id}] เปิดแล้ว");
    }
    
    public async Task OpenAsync()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        await Task.Delay(50);
        IsOpen = true;
        Console.WriteLine($"  [Conn-{Id}] เปิดแล้ว (async)");
    }
    
    public DbCommand CreateCommand(string sql)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return new DbCommand(this, sql);
    }
    
    public void Close()
    {
        if (IsOpen)
        {
            IsOpen = false;
            Console.WriteLine($"  [Conn-{Id}] ปิดแล้ว");
        }
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            Close();
            _disposed = true;
            Console.WriteLine($"  [Conn-{Id}] Disposed");
        }
    }
    
    public override string ToString() => $"DbConnection-{Id}";
}

public class DbCommand : IDisposable
{
    private readonly DbConnection _connection;
    public string Sql { get; }
    private bool _disposed = false;
    
    public DbCommand(DbConnection connection, string sql)
    {
        _connection = connection;
        Sql = sql;
    }
    
    public async Task<List<Dictionary<string, object>>> ExecuteAsync()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        
        await Task.Delay(20); // จำลอง query
        Console.WriteLine($"    Executed: {Sql}");
        
        return new List<Dictionary<string, object>>
        {
            new() { ["id"] = 1, ["name"] = "สมชาย" },
            new() { ["id"] = 2, ["name"] = "สมหญิง" }
        };
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _disposed = true;
        }
    }
}

// Connection Pool
public class ConnectionPool : IDisposable
{
    private readonly string _connectionString;
    private readonly int _maxConnections;
    private readonly ConcurrentBag<DbConnection> _available = new();
    private readonly SemaphoreSlim _semaphore;
    private int _totalCreated = 0;
    private bool _disposed = false;
    
    public ConnectionPool(string connectionString, int maxConnections = 5)
    {
        _connectionString = connectionString;
        _maxConnections = maxConnections;
        _semaphore = new SemaphoreSlim(maxConnections, maxConnections);
        
        Console.WriteLine($"Connection Pool created (max: {maxConnections})");
    }
    
    public async Task<PooledConnection> GetConnectionAsync(CancellationToken ct = default)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        
        // รอจน slot ว่าง
        await _semaphore.WaitAsync(ct);
        
        DbConnection conn;
        
        if (_available.TryTake(out conn!))
        {
            Console.WriteLine($"  Reusing {conn}");
        }
        else
        {
            int id = Interlocked.Increment(ref _totalCreated);
            conn = new DbConnection(_connectionString);
            await conn.OpenAsync();
        }
        
        return new PooledConnection(conn, this);
    }
    
    internal void ReturnConnection(DbConnection connection)
    {
        if (_disposed)
        {
            connection.Dispose();
            return;
        }
        
        Console.WriteLine($"  Returning {connection} to pool");
        _available.Add(connection);
        _semaphore.Release();
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _disposed = true;
            
            // Dispose ทุก connections ใน pool
            while (_available.TryTake(out var conn))
            {
                conn.Dispose();
            }
            
            _semaphore.Dispose();
            Console.WriteLine($"Connection Pool disposed ({_totalCreated} connections were created)");
        }
    }
}

// Pooled connection - คืน pool เมื่อ dispose
public class PooledConnection : IAsyncDisposable, IDisposable
{
    private readonly DbConnection _connection;
    private readonly ConnectionPool _pool;
    private bool _disposed = false;
    
    public PooledConnection(DbConnection connection, ConnectionPool pool)
    {
        _connection = connection;
        _pool = pool;
    }
    
    public DbCommand CreateCommand(string sql)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return _connection.CreateCommand(sql);
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            _disposed = true;
            _pool.ReturnConnection(_connection);
        }
    }
    
    public ValueTask DisposeAsync()
    {
        Dispose();
        return ValueTask.CompletedTask;
    }
}

// Unit of Work pattern ด้วย IDisposable
public class UnitOfWork : IDisposable
{
    private PooledConnection? _connection;
    private readonly List<Func<Task>> _pendingOperations = new();
    private bool _committed = false;
    private bool _disposed = false;
    
    public static async Task<UnitOfWork> CreateAsync(ConnectionPool pool)
    {
        var uow = new UnitOfWork();
        uow._connection = await pool.GetConnectionAsync();
        return uow;
    }
    
    public void AddOperation(Func<Task> operation)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _pendingOperations.Add(operation);
    }
    
    public async Task CommitAsync()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        
        Console.WriteLine($"  Committing {_pendingOperations.Count} operations...");
        foreach (var op in _pendingOperations)
        {
            await op();
        }
        
        _committed = true;
        Console.WriteLine("  Committed successfully");
    }
    
    public void Rollback()
    {
        if (!_committed)
        {
            Console.WriteLine("  Rolled back");
            _pendingOperations.Clear();
        }
    }
    
    public void Dispose()
    {
        if (!_disposed)
        {
            if (!_committed)
                Rollback();
            
            _connection?.Dispose();
            _disposed = true;
        }
    }
}

// Repository pattern ด้วย IDisposable
public class UserRepository : IDisposable
{
    private readonly ConnectionPool _pool;
    private bool _disposed = false;
    
    public UserRepository(ConnectionPool pool)
    {
        _pool = pool;
    }
    
    public async Task<List<Dictionary<string, object>>> GetAllAsync(CancellationToken ct = default)
    {
        await using var conn = await _pool.GetConnectionAsync(ct);
        using var cmd = conn.CreateCommand("SELECT * FROM users");
        return await cmd.ExecuteAsync();
    }
    
    public async Task<Dictionary<string, object>?> GetByIdAsync(int id, CancellationToken ct = default)
    {
        await using var conn = await _pool.GetConnectionAsync(ct);
        using var cmd = conn.CreateCommand($"SELECT * FROM users WHERE id = {id}");
        var results = await cmd.ExecuteAsync();
        return results.FirstOrDefault();
    }
    
    public async Task CreateAsync(Dictionary<string, object> user, CancellationToken ct = default)
    {
        await using var conn = await _pool.GetConnectionAsync(ct);
        using var cmd = conn.CreateCommand($"INSERT INTO users VALUES (...)");
        await cmd.ExecuteAsync();
    }
    
    public void Dispose()
    {
        // ไม่ dispose pool เพราะ pool ถูกส่งมาจากภายนอก (dependency injection)
        _disposed = true;
    }
}

// Main Program
class Program
{
    static async Task Main()
    {
        Console.WriteLine("===== Database Connection Manager Demo =====\n");
        
        // Pool configuration
        using var pool = new ConnectionPool("Server=localhost;Database=MyDB", maxConnections: 3);
        
        Console.WriteLine("\n--- Simple Connection Usage ---");
        
        // Basic usage
        await using (var conn = await pool.GetConnectionAsync())
        {
            using var cmd = conn.CreateCommand("SELECT * FROM products");
            var results = await cmd.ExecuteAsync();
            Console.WriteLine($"Got {results.Count} products");
        }
        
        Console.WriteLine("\n--- Parallel Connections ---");
        
        // Multiple parallel operations
        var tasks = Enumerable.Range(1, 5).Select(async i =>
        {
            await using var conn = await pool.GetConnectionAsync();
            using var cmd = conn.CreateCommand($"SELECT * FROM table WHERE id = {i}");
            await cmd.ExecuteAsync();
        });
        
        await Task.WhenAll(tasks);
        
        Console.WriteLine("\n--- Unit of Work Pattern ---");
        
        // Unit of Work
        var uow = await UnitOfWork.CreateAsync(pool);
        using (uow)
        {
            uow.AddOperation(async () =>
            {
                var conn2 = await pool.GetConnectionAsync();
                await using (conn2)
                using (var cmd = conn2.CreateCommand("INSERT INTO orders ..."))
                    await cmd.ExecuteAsync();
            });
            
            uow.AddOperation(async () =>
            {
                var conn2 = await pool.GetConnectionAsync();
                await using (conn2)
                using (var cmd = conn2.CreateCommand("UPDATE inventory ..."))
                    await cmd.ExecuteAsync();
            });
            
            await uow.CommitAsync();
        }
        
        Console.WriteLine("\n--- Repository Pattern ---");
        
        using var repo = new UserRepository(pool);
        var users = await repo.GetAllAsync();
        Console.WriteLine($"Found {users.Count} users");
        
        var user = await repo.GetByIdAsync(1);
        Console.WriteLine($"User 1: {(user != null ? user["name"] : "not found")}");
        
        Console.WriteLine("\n--- Cleanup ---");
    }
}
```

---

## Exercises

### Exercise 1: Resource Tracker
```csharp
// TODO: สร้าง resource tracker ที่ detect memory leaks

public class ResourceTracker
{
    private static readonly ConcurrentDictionary<string, (WeakReference Ref, string StackTrace)>
        _tracked = new();
    
    // Register resource เมื่อสร้าง
    public static void Track(IDisposable resource)
    {
        throw new NotImplementedException();
    }
    
    // Untrack เมื่อ Dispose
    public static void Untrack(IDisposable resource)
    {
        throw new NotImplementedException();
    }
    
    // Report resources ที่ยังไม่ถูก Dispose
    public static IEnumerable<string> GetLeaks()
    {
        throw new NotImplementedException();
    }
}

// ใช้ด้วย base class
public abstract class TrackedDisposable : IDisposable
{
    protected TrackedDisposable()
    {
        ResourceTracker.Track(this);
    }
    
    public virtual void Dispose()
    {
        ResourceTracker.Untrack(this);
        GC.SuppressFinalize(this);
    }
}
```

### Exercise 2: Lease Pattern
```csharp
// TODO: สร้าง leasing system สำหรับ shared resources

public class SharedResource
{
    public string Name { get; init; } = string.Empty;
    // ... resource data
}

public class ResourceLease : IDisposable
{
    // เมื่อ Dispose, คืน resource กลับ pool
    public SharedResource Resource { get; }
    public TimeSpan Duration { get; }
    public bool IsExpired => DateTime.Now > ExpiresAt;
    public DateTime ExpiresAt { get; }
    
    // TODO: implement
    public void Dispose() { throw new NotImplementedException(); }
}

public class ResourceManager
{
    // Lease resource โดยให้ duration
    public ResourceLease? TryAcquire(string resourceName, TimeSpan leaseDuration)
    {
        throw new NotImplementedException();
    }
    
    // Return expired leases
    public void CleanupExpiredLeases()
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: Transaction Scope
```csharp
// TODO: สร้าง transaction scope คล้าย System.Transactions

public class TransactionScope : IDisposable
{
    private bool _completed = false;
    private readonly List<Action> _rollbackActions = new();
    
    public void AddRollbackAction(Action action)
    {
        _rollbackActions.Add(action);
    }
    
    public void Complete()
    {
        _completed = true;
    }
    
    public void Dispose()
    {
        // ถ้าไม่ Complete -> Rollback
        // ถ้า Complete -> Commit
        throw new NotImplementedException();
    }
    
    // ใช้งาน:
    // using var scope = new TransactionScope();
    // scope.AddRollbackAction(() => UndoOperation1());
    // DoOperation1();
    // scope.AddRollbackAction(() => UndoOperation2());
    // DoOperation2();
    // scope.Complete(); // commit
}
```

---

## สรุป

✅ **IDisposable** ใช้สำหรับ cleanup resources เช่น file handles, network connections, database connections

✅ **using statement** เรียก Dispose อัตโนมัติแม้เกิด exception

✅ **using declaration** (C# 8+) ลด nesting ทำให้โค้ด flat กว่า

✅ **Dispose pattern** แยก managed และ unmanaged resources

✅ **Finalizer** เป็น safety net สำหรับกรณีลืม Dispose แต่ช้ากว่า

✅ **GC.SuppressFinalize()** หลัง Dispose ช่วย performance

✅ **SafeHandle** ปลอดภัยกว่า IntPtr สำหรับ unmanaged handles

✅ **IAsyncDisposable + await using** สำหรับ async cleanup

✅ **Connection Pool** pattern ลด overhead จากการสร้าง connections ใหม่

---

## Part ถัดไป

➡️ **Part 041**: LINQ ขั้นสูง - Custom operators, Expression trees, Query providers

---

*Part 040/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
