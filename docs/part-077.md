# Part 77: Design Patterns: Creational

## เนื้อหาใน Part นี้
- Singleton Pattern
- Factory Method
- Abstract Factory
- Builder Pattern
- Prototype
- โปรแกรมตัวอย่าง: Database connection factory

---

## Design Patterns คืออะไร?

**Design Patterns** คือ "สูตร" การแก้ปัญหาที่พบบ่อยในการออกแบบซอฟต์แวร์ แบ่งเป็น 3 กลุ่ม:

1. **Creational** - การสร้าง objects
2. **Structural** - การจัดโครงสร้าง objects
3. **Behavioral** - การสื่อสารระหว่าง objects

### Creational Patterns

```
Creational Patterns:
├── Singleton     - มีแค่ instance เดียวในระบบ
├── Factory Method - ให้ subclass ตัดสินใจว่าจะสร้าง object ชนิดไหน
├── Abstract Factory - สร้างกลุ่มของ related objects
├── Builder       - สร้าง object ที่ซับซ้อน step by step
└── Prototype     - สร้าง object ใหม่โดย clone จาก object เดิม
```

---

## 1. Singleton Pattern

Singleton ทำให้ class มีแค่ instance เดียว และให้ global access point

### การใช้งานใน .NET (Thread-safe)

```csharp
// ❌ แบบเดิม (ไม่ thread-safe)
public class OldSingleton
{
    private static OldSingleton? _instance;
    
    private OldSingleton() { }
    
    public static OldSingleton Instance
    {
        get
        {
            if (_instance == null)
                _instance = new OldSingleton(); // ไม่ thread-safe!
            return _instance;
        }
    }
}

// ✅ แบบ Lazy<T> (thread-safe, lazy initialization)
public class DatabaseConfig
{
    private static readonly Lazy<DatabaseConfig> _lazy =
        new Lazy<DatabaseConfig>(() => new DatabaseConfig());
    
    private DatabaseConfig()
    {
        // โหลด config จาก environment
        ConnectionString = Environment.GetEnvironmentVariable("DB_CONNECTION")
            ?? "Server=localhost;Database=mydb;";
        CommandTimeout = 30;
        MaxPoolSize = 100;
    }
    
    public static DatabaseConfig Instance => _lazy.Value;
    
    public string ConnectionString { get; private set; }
    public int CommandTimeout { get; private set; }
    public int MaxPoolSize { get; private set; }
}

// ✅ ใน ASP.NET Core - ใช้ DI แทน
public class ApplicationCache
{
    private readonly Dictionary<string, object> _store = new();
    private readonly ReaderWriterLockSlim _lock = new();
    
    public T? Get<T>(string key)
    {
        _lock.EnterReadLock();
        try
        {
            return _store.TryGetValue(key, out var value) && value is T typed ? typed : default;
        }
        finally { _lock.ExitReadLock(); }
    }
    
    public void Set(string key, object value)
    {
        _lock.EnterWriteLock();
        try { _store[key] = value; }
        finally { _lock.ExitWriteLock(); }
    }
    
    public void Remove(string key)
    {
        _lock.EnterWriteLock();
        try { _store.Remove(key); }
        finally { _lock.ExitWriteLock(); }
    }
}

// ลงทะเบียนเป็น Singleton ใน DI
// builder.Services.AddSingleton<ApplicationCache>();
```

---

## 2. Factory Method Pattern

Factory Method ให้ subclass ตัดสินใจว่าจะสร้าง object ชนิดไหน

```csharp
// Abstract Product
public abstract class Logger
{
    public abstract void Log(string message, LogLevel level);
    
    public void LogInfo(string message) => Log(message, LogLevel.Information);
    public void LogError(string message) => Log(message, LogLevel.Error);
    public void LogWarning(string message) => Log(message, LogLevel.Warning);
}

// Concrete Products
public class ConsoleLogger : Logger
{
    private readonly string _prefix;
    
    public ConsoleLogger(string prefix = "")
    {
        _prefix = prefix;
    }
    
    public override void Log(string message, LogLevel level)
    {
        var color = level switch
        {
            LogLevel.Error => ConsoleColor.Red,
            LogLevel.Warning => ConsoleColor.Yellow,
            LogLevel.Information => ConsoleColor.Green,
            _ => ConsoleColor.White
        };
        
        Console.ForegroundColor = color;
        Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] [{level}] {_prefix}{message}");
        Console.ResetColor();
    }
}

public class FileLogger : Logger
{
    private readonly string _filePath;
    
    public FileLogger(string filePath)
    {
        _filePath = filePath;
        Directory.CreateDirectory(Path.GetDirectoryName(filePath) ?? ".");
    }
    
    public override void Log(string message, LogLevel level)
    {
        var entry = $"{DateTime.Now:yyyy-MM-dd HH:mm:ss} [{level}] {message}";
        File.AppendAllText(_filePath, entry + Environment.NewLine);
    }
}

public class CloudLogger : Logger
{
    private readonly string _endpoint;
    private readonly List<string> _buffer = new();
    
    public CloudLogger(string endpoint)
    {
        _endpoint = endpoint;
    }
    
    public override void Log(string message, LogLevel level)
    {
        _buffer.Add($"[{level}] {message}");
        // ส่งไป cloud endpoint (simplified)
        Console.WriteLine($"[CLOUD:{_endpoint}] {message}");
    }
}

// Abstract Creator
public abstract class LoggerFactory
{
    public abstract Logger CreateLogger();
    
    // Template method - ใช้ Factory Method ภายใน
    public Logger GetLogger()
    {
        var logger = CreateLogger();
        ConfigureLogger(logger);
        return logger;
    }
    
    protected virtual void ConfigureLogger(Logger logger) { }
}

// Concrete Creators
public class ConsoleLoggerFactory : LoggerFactory
{
    private readonly string _prefix;
    
    public ConsoleLoggerFactory(string prefix = "")
    {
        _prefix = prefix;
    }
    
    public override Logger CreateLogger() => new ConsoleLogger(_prefix);
}

public class FileLoggerFactory : LoggerFactory
{
    private readonly string _logPath;
    
    public FileLoggerFactory(string logPath)
    {
        _logPath = logPath;
    }
    
    public override Logger CreateLogger() => new FileLogger(_logPath);
}

public class CloudLoggerFactory : LoggerFactory
{
    private readonly string _endpoint;
    
    public CloudLoggerFactory(string endpoint)
    {
        _endpoint = endpoint;
    }
    
    public override Logger CreateLogger() => new CloudLogger(_endpoint);
}

// Static Factory Method (อีกรูปแบบ)
public static class LoggerProvider
{
    public static Logger Create(string type, IConfiguration config)
    {
        return type.ToLower() switch
        {
            "console" => new ConsoleLogger(config["Logger:Prefix"] ?? ""),
            "file" => new FileLogger(config["Logger:Path"] ?? "logs/app.log"),
            "cloud" => new CloudLogger(config["Logger:Endpoint"] ?? "https://logs.example.com"),
            _ => throw new ArgumentException($"Logger type ไม่รองรับ: {type}")
        };
    }
}
```

---

## 3. Abstract Factory Pattern

Abstract Factory สร้างกลุ่มของ related objects โดยไม่ระบุ concrete class

```csharp
// Abstract Products
public interface IButton
{
    void Render();
    void OnClick(Action action);
}

public interface ITextInput
{
    void Render();
    string GetValue();
    void SetPlaceholder(string placeholder);
}

public interface IDialog
{
    void Show(string title, string message);
    void Close();
}

// Abstract Factory
public interface IUIFactory
{
    IButton CreateButton(string text);
    ITextInput CreateTextInput();
    IDialog CreateDialog();
}

// Concrete Products - Windows Theme
public class WindowsButton : IButton
{
    private readonly string _text;
    private Action? _clickAction;
    
    public WindowsButton(string text) { _text = text; }
    
    public void Render() 
        => Console.WriteLine($"[Windows Button] {_text}");
    public void OnClick(Action action) => _clickAction = action;
}

public class WindowsTextInput : ITextInput
{
    private string _placeholder = "";
    
    public void Render() 
        => Console.WriteLine($"[Windows TextInput] placeholder: {_placeholder}");
    public string GetValue() => "windows_value";
    public void SetPlaceholder(string placeholder) => _placeholder = placeholder;
}

public class WindowsDialog : IDialog
{
    public void Show(string title, string message)
        => Console.WriteLine($"[Windows Dialog] {title}: {message}");
    public void Close() 
        => Console.WriteLine("[Windows Dialog] Closed");
}

// Concrete Products - Mac Theme
public class MacButton : IButton
{
    private readonly string _text;
    
    public MacButton(string text) { _text = text; }
    
    public void Render() 
        => Console.WriteLine($"🍎 [Mac Button] {_text}");
    public void OnClick(Action action) { }
}

public class MacTextInput : ITextInput
{
    private string _placeholder = "";
    
    public void Render() 
        => Console.WriteLine($"🍎 [Mac TextInput] _{_placeholder}_");
    public string GetValue() => "mac_value";
    public void SetPlaceholder(string placeholder) => _placeholder = placeholder;
}

public class MacDialog : IDialog
{
    public void Show(string title, string message)
        => Console.WriteLine($"🍎 [Mac Dialog] {title}: {message}");
    public void Close() 
        => Console.WriteLine("🍎 [Mac Dialog] Closed");
}

// Concrete Factories
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton(string text) => new WindowsButton(text);
    public ITextInput CreateTextInput() => new WindowsTextInput();
    public IDialog CreateDialog() => new WindowsDialog();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton(string text) => new MacButton(text);
    public ITextInput CreateTextInput() => new MacTextInput();
    public IDialog CreateDialog() => new MacDialog();
}

// Client ที่ใช้ Abstract Factory
public class LoginForm
{
    private readonly IButton _loginButton;
    private readonly IButton _cancelButton;
    private readonly ITextInput _emailInput;
    private readonly ITextInput _passwordInput;
    private readonly IDialog _errorDialog;
    
    public LoginForm(IUIFactory uiFactory)
    {
        _loginButton = uiFactory.CreateButton("เข้าสู่ระบบ");
        _cancelButton = uiFactory.CreateButton("ยกเลิก");
        _emailInput = uiFactory.CreateTextInput();
        _passwordInput = uiFactory.CreateTextInput();
        _errorDialog = uiFactory.CreateDialog();
        
        _emailInput.SetPlaceholder("อีเมล");
        _passwordInput.SetPlaceholder("รหัสผ่าน");
    }
    
    public void Render()
    {
        Console.WriteLine("--- Login Form ---");
        _emailInput.Render();
        _passwordInput.Render();
        _loginButton.Render();
        _cancelButton.Render();
    }
    
    public void ShowError(string message)
    {
        _errorDialog.Show("เกิดข้อผิดพลาด", message);
    }
}
```

---

## 4. Builder Pattern

Builder สร้าง object ที่ซับซ้อนทีละขั้นตอน

```csharp
// Product
public class DatabaseConnection
{
    public string Host { get; internal set; } = "localhost";
    public int Port { get; internal set; } = 5432;
    public string Database { get; internal set; } = "";
    public string Username { get; internal set; } = "";
    public string Password { get; internal set; } = "";
    public int MaxPoolSize { get; internal set; } = 10;
    public int MinPoolSize { get; internal set; } = 1;
    public int ConnectionTimeout { get; internal set; } = 30;
    public int CommandTimeout { get; internal set; } = 60;
    public bool EnableSsl { get; internal set; } = false;
    public bool EnableRetry { get; internal set; } = false;
    public int MaxRetries { get; internal set; } = 3;
    public string ConnectionString { get; internal set; } = "";
    
    public override string ToString()
        => $"Host={Host};Port={Port};Database={Database};Username={Username}";
}

// Builder Interface
public interface IDatabaseConnectionBuilder
{
    IDatabaseConnectionBuilder WithHost(string host);
    IDatabaseConnectionBuilder WithPort(int port);
    IDatabaseConnectionBuilder WithDatabase(string database);
    IDatabaseConnectionBuilder WithCredentials(string username, string password);
    IDatabaseConnectionBuilder WithPooling(int minSize, int maxSize);
    IDatabaseConnectionBuilder WithTimeouts(int connectionTimeout, int commandTimeout);
    IDatabaseConnectionBuilder WithSsl(bool enable = true);
    IDatabaseConnectionBuilder WithRetry(int maxRetries = 3);
    DatabaseConnection Build();
}

// Concrete Builder
public class PostgreSqlConnectionBuilder : IDatabaseConnectionBuilder
{
    private readonly DatabaseConnection _connection = new();
    
    public IDatabaseConnectionBuilder WithHost(string host)
    {
        _connection.Host = host;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithPort(int port)
    {
        if (port < 1 || port > 65535)
            throw new ArgumentException("Port ต้องอยู่ระหว่าง 1-65535");
        _connection.Port = port;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithDatabase(string database)
    {
        if (string.IsNullOrWhiteSpace(database))
            throw new ArgumentException("Database name ต้องไม่ว่าง");
        _connection.Database = database;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithCredentials(string username, string password)
    {
        _connection.Username = username;
        _connection.Password = password;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithPooling(int minSize, int maxSize)
    {
        if (minSize < 0 || maxSize < minSize)
            throw new ArgumentException("Pool size ไม่ถูกต้อง");
        _connection.MinPoolSize = minSize;
        _connection.MaxPoolSize = maxSize;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithTimeouts(int connectionTimeout, int commandTimeout)
    {
        _connection.ConnectionTimeout = connectionTimeout;
        _connection.CommandTimeout = commandTimeout;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithSsl(bool enable = true)
    {
        _connection.EnableSsl = enable;
        return this;
    }
    
    public IDatabaseConnectionBuilder WithRetry(int maxRetries = 3)
    {
        _connection.EnableRetry = true;
        _connection.MaxRetries = maxRetries;
        return this;
    }
    
    public DatabaseConnection Build()
    {
        // Validate ก่อน build
        if (string.IsNullOrWhiteSpace(_connection.Database))
            throw new InvalidOperationException("ต้องระบุ Database name");
        if (string.IsNullOrWhiteSpace(_connection.Username))
            throw new InvalidOperationException("ต้องระบุ Username");
            
        // สร้าง connection string
        var parts = new List<string>
        {
            $"Host={_connection.Host}",
            $"Port={_connection.Port}",
            $"Database={_connection.Database}",
            $"Username={_connection.Username}",
            $"Password={_connection.Password}",
            $"Minimum Pool Size={_connection.MinPoolSize}",
            $"Maximum Pool Size={_connection.MaxPoolSize}",
            $"Timeout={_connection.ConnectionTimeout}",
            $"Command Timeout={_connection.CommandTimeout}"
        };
        
        if (_connection.EnableSsl)
            parts.Add("SSL Mode=Require");
            
        _connection.ConnectionString = string.Join(";", parts);
        
        return _connection;
    }
}

// Director (Optional) - รู้จักขั้นตอนการสร้าง
public class DatabaseConnectionDirector
{
    private readonly IDatabaseConnectionBuilder _builder;
    
    public DatabaseConnectionDirector(IDatabaseConnectionBuilder builder)
    {
        _builder = builder;
    }
    
    public DatabaseConnection BuildDevelopmentConnection()
    {
        return _builder
            .WithHost("localhost")
            .WithPort(5432)
            .WithDatabase("myapp_dev")
            .WithCredentials("dev_user", "dev_password")
            .WithPooling(1, 5)
            .Build();
    }
    
    public DatabaseConnection BuildProductionConnection(
        string host, string db, string user, string pass)
    {
        return _builder
            .WithHost(host)
            .WithPort(5432)
            .WithDatabase(db)
            .WithCredentials(user, pass)
            .WithPooling(5, 50)
            .WithTimeouts(10, 30)
            .WithSsl()
            .WithRetry(5)
            .Build();
    }
}
```

---

## 5. Prototype Pattern

Prototype สร้าง object ใหม่โดย clone จาก object เดิม

```csharp
// Prototype interface
public interface IPrototype<T>
{
    T Clone();
}

// Concrete Prototype
public class EmailTemplate : IPrototype<EmailTemplate>
{
    public string Subject { get; set; } = "";
    public string Body { get; set; } = "";
    public string FromAddress { get; set; } = "";
    public List<string> ToAddresses { get; set; } = new();
    public Dictionary<string, string> Variables { get; set; } = new();
    
    public EmailTemplate Clone()
    {
        return new EmailTemplate
        {
            Subject = this.Subject,
            Body = this.Body,
            FromAddress = this.FromAddress,
            ToAddresses = new List<string>(this.ToAddresses),
            Variables = new Dictionary<string, string>(this.Variables)
        };
    }
    
    public string Render()
    {
        var result = Body;
        foreach (var (key, value) in Variables)
            result = result.Replace($"{{{{{key}}}}}", value);
        return result;
    }
}

// ใช้งาน Prototype
public class EmailTemplateRegistry
{
    private readonly Dictionary<string, EmailTemplate> _templates = new();
    
    public void Register(string name, EmailTemplate template)
    {
        _templates[name] = template;
    }
    
    public EmailTemplate GetTemplate(string name)
    {
        if (!_templates.TryGetValue(name, out var template))
            throw new KeyNotFoundException($"ไม่พบ template: {name}");
            
        return template.Clone(); // Clone เพื่อไม่แก้ไข original
    }
}
```

---

## โปรแกรมตัวอย่าง: Database Connection Factory

```csharp
// ===== Complete Database Factory Example =====
public enum DatabaseType { SqlServer, PostgreSql, MySql, SQLite, InMemory }

// Abstract connection
public abstract class DatabaseConnection
{
    public abstract string ConnectionString { get; }
    public abstract DatabaseType Type { get; }
    
    public abstract void Open();
    public abstract void Close();
    public abstract IDbCommand CreateCommand(string sql);
    public abstract bool TestConnection();
}

// Concrete connections
public class SqlServerConnection : DatabaseConnection
{
    public override string ConnectionString { get; }
    public override DatabaseType Type => DatabaseType.SqlServer;
    
    public SqlServerConnection(string connectionString)
    {
        ConnectionString = connectionString;
    }
    
    public override void Open()
        => Console.WriteLine($"[SQL Server] เปิดการเชื่อมต่อ: {ConnectionString[..Math.Min(50, ConnectionString.Length)]}...");
    public override void Close()
        => Console.WriteLine("[SQL Server] ปิดการเชื่อมต่อ");
    public override IDbCommand CreateCommand(string sql)
        => new FakeDbCommand(sql, "SQL Server");
    public override bool TestConnection()
    {
        Console.WriteLine("[SQL Server] ทดสอบการเชื่อมต่อ...");
        return true;
    }
}

public class PostgreSqlConnection : DatabaseConnection
{
    public override string ConnectionString { get; }
    public override DatabaseType Type => DatabaseType.PostgreSql;
    
    public PostgreSqlConnection(string connectionString)
    {
        ConnectionString = connectionString;
    }
    
    public override void Open()
        => Console.WriteLine($"[PostgreSQL] เปิดการเชื่อมต่อ");
    public override void Close()
        => Console.WriteLine("[PostgreSQL] ปิดการเชื่อมต่อ");
    public override IDbCommand CreateCommand(string sql)
        => new FakeDbCommand(sql, "PostgreSQL");
    public override bool TestConnection()
    {
        Console.WriteLine("[PostgreSQL] ทดสอบการเชื่อมต่อ...");
        return true;
    }
}

public class InMemoryConnection : DatabaseConnection
{
    public override string ConnectionString => "InMemory://test";
    public override DatabaseType Type => DatabaseType.InMemory;
    
    public override void Open() => Console.WriteLine("[InMemory] เปิดการเชื่อมต่อ InMemory");
    public override void Close() => Console.WriteLine("[InMemory] ปิดการเชื่อมต่อ InMemory");
    public override IDbCommand CreateCommand(string sql) => new FakeDbCommand(sql, "InMemory");
    public override bool TestConnection() => true;
}

public class FakeDbCommand : IDbCommand
{
    private readonly string _provider;
    public string CommandText { get; set; }
    // ... implement IDbCommand members
    
    public FakeDbCommand(string sql, string provider)
    {
        CommandText = sql;
        _provider = provider;
    }
    
    public void ExecuteNonQuery()
        => Console.WriteLine($"[{_provider}] Execute: {CommandText}");
}

// Factory
public interface IDatabaseConnectionFactory
{
    DatabaseConnection CreateConnection();
    DatabaseConnection CreateConnection(string connectionString);
}

public class SqlServerConnectionFactory : IDatabaseConnectionFactory
{
    private readonly string _defaultConnectionString;
    
    public SqlServerConnectionFactory(string defaultConnectionString)
    {
        _defaultConnectionString = defaultConnectionString;
    }
    
    public DatabaseConnection CreateConnection()
        => new SqlServerConnection(_defaultConnectionString);
    public DatabaseConnection CreateConnection(string connectionString)
        => new SqlServerConnection(connectionString);
}

public class PostgreSqlConnectionFactory : IDatabaseConnectionFactory
{
    private readonly string _defaultConnectionString;
    
    public PostgreSqlConnectionFactory(string connectionString)
    {
        _defaultConnectionString = connectionString;
    }
    
    public DatabaseConnection CreateConnection()
        => new PostgreSqlConnection(_defaultConnectionString);
    public DatabaseConnection CreateConnection(string connectionString)
        => new PostgreSqlConnection(connectionString);
}

// Abstract Factory - สร้าง family ของ database objects
public interface IDatabaseFactory
{
    IDatabaseConnectionFactory CreateConnectionFactory();
    IQueryBuilder CreateQueryBuilder();
    ISchemaManager CreateSchemaManager();
}

public class SqlServerDatabaseFactory : IDatabaseFactory
{
    private readonly string _connectionString;
    
    public SqlServerDatabaseFactory(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public IDatabaseConnectionFactory CreateConnectionFactory()
        => new SqlServerConnectionFactory(_connectionString);
    public IQueryBuilder CreateQueryBuilder() => new SqlServerQueryBuilder();
    public ISchemaManager CreateSchemaManager() => new SqlServerSchemaManager();
}

// Builder สำหรับ connection string
public class ConnectionStringBuilder
{
    private readonly Dictionary<string, string> _params = new();
    
    public ConnectionStringBuilder Server(string server) 
        => Set("Server", server);
    public ConnectionStringBuilder Database(string db) 
        => Set("Database", db);
    public ConnectionStringBuilder UserId(string user) 
        => Set("User Id", user);
    public ConnectionStringBuilder Password(string pass) 
        => Set("Password", pass);
    public ConnectionStringBuilder IntegratedSecurity(bool enabled = true)
        => Set("Integrated Security", enabled ? "true" : "false");
    public ConnectionStringBuilder MultipleActiveResultSets(bool enabled = true)
        => Set("MultipleActiveResultSets", enabled ? "true" : "false");
    public ConnectionStringBuilder TrustServerCertificate(bool enabled = true)
        => Set("TrustServerCertificate", enabled ? "Yes" : "No");
    public ConnectionStringBuilder ConnectTimeout(int seconds)
        => Set("Connect Timeout", seconds.ToString());
        
    private ConnectionStringBuilder Set(string key, string value)
    {
        _params[key] = value;
        return this;
    }
    
    public string Build() => string.Join(";", _params.Select(p => $"{p.Key}={p.Value}"));
}

// ===== Main Program =====
Console.WriteLine("=== Database Connection Factory Demo ===\n");

// 1. Factory Method
Console.WriteLine("--- Factory Method Demo ---");
var sqlConnStr = new ConnectionStringBuilder()
    .Server("localhost")
    .Database("MyApp")
    .UserId("sa")
    .Password("YourPassword!")
    .MultipleActiveResultSets()
    .TrustServerCertificate()
    .Build();

Console.WriteLine($"SQL Server Connection String:\n{sqlConnStr}\n");

var sqlFactory = new SqlServerConnectionFactory(sqlConnStr);
var conn1 = sqlFactory.CreateConnection();
conn1.Open();
conn1.TestConnection();
conn1.Close();

// 2. Builder
Console.WriteLine("\n--- Builder Demo ---");
var pgBuilder = new PostgreSqlConnectionBuilder();
var pgDirector = new DatabaseConnectionDirector(pgBuilder);

var devConn = pgDirector.BuildDevelopmentConnection();
Console.WriteLine($"Dev Connection: {devConn}");

var prodConn = pgDirector.BuildProductionConnection(
    "prod-db.company.com", "production", "app_user", "SecurePass!");
Console.WriteLine($"Prod Connection String: {prodConn.ConnectionString[..50]}...");

// 3. Singleton
Console.WriteLine("\n--- Singleton Demo ---");
var config1 = DatabaseConfig.Instance;
var config2 = DatabaseConfig.Instance;
Console.WriteLine($"Same instance: {ReferenceEquals(config1, config2)}");
Console.WriteLine($"Command Timeout: {config1.CommandTimeout}s");

// 4. Prototype
Console.WriteLine("\n--- Prototype Demo ---");
var registry = new EmailTemplateRegistry();

var orderTemplate = new EmailTemplate
{
    Subject = "ยืนยันการสั่งซื้อ #{{OrderNumber}}",
    Body = "สวัสดีคุณ {{CustomerName}},\nคำสั่งซื้อหมายเลข {{OrderNumber}} ได้รับการยืนยันแล้ว\nยอดรวม: {{Total}} บาท",
    FromAddress = "orders@shop.com"
};
registry.Register("order_confirmation", orderTemplate);

var email1 = registry.GetTemplate("order_confirmation");
email1.Variables["OrderNumber"] = "ORD-001";
email1.Variables["CustomerName"] = "สมชาย";
email1.Variables["Total"] = "45,000";
email1.ToAddresses.Add("somchai@example.com");

var email2 = registry.GetTemplate("order_confirmation"); // clone ใหม่
email2.Variables["OrderNumber"] = "ORD-002";
email2.Variables["CustomerName"] = "สมหญิง";
email2.Variables["Total"] = "9,000";

Console.WriteLine($"Email 1:\n{email1.Render()}\n");
Console.WriteLine($"Email 2:\n{email2.Render()}");
```

---

## Exercises

1. **Exercise 1**: Implement `HttpClientFactory` ที่สร้าง HttpClient ด้วย configuration ต่างกัน:
   - `DefaultHttpClient` - timeout 30s
   - `LongRunningHttpClient` - timeout 5 minutes
   - `RetryHttpClient` - retry 3 ครั้ง

2. **Exercise 2**: สร้าง `ReportBuilder` ด้วย Builder Pattern:
   - WithTitle, WithSubtitle
   - AddSection, AddTable, AddChart
   - WithFooter
   - Build() → Report object

3. **Exercise 3**: Implement `CacheFactory` ด้วย Abstract Factory:
   - `MemoryCacheFactory` - in-memory
   - `RedisCacheFactory` - distributed
   - `HybridCacheFactory` - two-level

4. **Exercise 4**: สร้าง `DocumentPrototype` ที่ deep clone document พร้อม attachments

5. **Exercise 5**: ทดสอบทุก Pattern ด้วย unit tests

---

## สรุป

Creational Patterns ช่วยแก้ปัญหาการสร้าง objects:
- **Singleton**: มีแค่ instance เดียว
- **Factory Method**: ให้ subclass เลือก type ที่สร้าง
- **Abstract Factory**: สร้างกลุ่ม related objects
- **Builder**: สร้าง complex object ทีละขั้น
- **Prototype**: clone object ที่มีอยู่

---

## Part ถัดไป

ใน Part 78 เราจะเรียนรู้ **Design Patterns: Structural** เช่น Adapter, Decorator, Facade, Proxy

---

*Part 77/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
