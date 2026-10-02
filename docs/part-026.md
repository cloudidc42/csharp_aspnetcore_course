# Part 026: Exception Handling

## เนื้อหาใน Part นี้
- try-catch-finally พื้นฐาน
- Exception Hierarchy ใน .NET
- throw vs throw ex
- Custom Exceptions
- Multiple catch blocks
- Exception Filters (when)
- AggregateException
- Global Exception Handler
- โปรแกรมตัวอย่าง: File Processing with Error Handling

---

## 1. try-catch-finally พื้นฐาน

```csharp
// ===== รูปแบบพื้นฐาน =====
try
{
    // code ที่อาจเกิด exception
    int[] arr = { 1, 2, 3 };
    Console.WriteLine(arr[10]); // throws IndexOutOfRangeException
}
catch (IndexOutOfRangeException ex)
{
    // จัดการ exception เฉพาะ type
    Console.WriteLine($"Array index error: {ex.Message}");
}
catch (Exception ex)
{
    // จัดการ exception อื่นๆ ทั้งหมด
    Console.WriteLine($"General error: {ex.Message}");
}
finally
{
    // Execute เสมอ ไม่ว่า exception จะเกิดหรือไม่
    Console.WriteLine("Finally block always runs");
}

// ===== finally สำหรับ cleanup =====
FileStream? stream = null;
try
{
    stream = File.OpenRead("data.txt");
    // process file...
}
catch (FileNotFoundException)
{
    Console.WriteLine("File not found");
}
finally
{
    stream?.Dispose(); // ปิด stream เสมอ
}

// ===== ใช้ using แทน finally (แนะนำ) =====
try
{
    using var fileStream = File.OpenRead("data.txt");
    // process file - stream ถูก dispose อัตโนมัติ
}
catch (FileNotFoundException ex)
{
    Console.WriteLine($"File not found: {ex.FileName}");
}

// ===== Exception Properties =====
try
{
    throw new InvalidOperationException("Test error");
}
catch (Exception ex)
{
    Console.WriteLine($"Type: {ex.GetType().Name}");
    Console.WriteLine($"Message: {ex.Message}");
    Console.WriteLine($"Source: {ex.Source}");
    Console.WriteLine($"StackTrace: {ex.StackTrace}");
    Console.WriteLine($"InnerException: {ex.InnerException?.Message}");
    Console.WriteLine($"HelpLink: {ex.HelpLink}");
}
```

---

## 2. Exception Hierarchy ใน .NET

```
System.Object
└── System.Exception
    ├── System.SystemException
    │   ├── System.ArgumentException
    │   │   ├── System.ArgumentNullException
    │   │   └── System.ArgumentOutOfRangeException
    │   ├── System.InvalidOperationException
    │   │   └── System.ObjectDisposedException
    │   ├── System.NullReferenceException
    │   ├── System.IndexOutOfRangeException
    │   ├── System.ArithmeticException
    │   │   ├── System.DivideByZeroException
    │   │   └── System.OverflowException
    │   ├── System.FormatException
    │   ├── System.OutOfMemoryException
    │   ├── System.StackOverflowException
    │   ├── System.NotImplementedException
    │   ├── System.NotSupportedException
    │   ├── System.IO.IOException
    │   │   ├── System.IO.FileNotFoundException
    │   │   ├── System.IO.DirectoryNotFoundException
    │   │   └── System.IO.EndOfStreamException
    │   └── System.Collections.Generic.KeyNotFoundException
    └── System.ApplicationException (เก่า - ไม่แนะนำ)
        └── (Custom exceptions)
```

```csharp
// ตัวอย่าง exceptions ที่พบบ่อย
try
{
    // ArgumentNullException
    string? s = null;
    ArgumentNullException.ThrowIfNull(s, nameof(s));
}
catch (ArgumentNullException ex)
{
    Console.WriteLine($"Null: {ex.ParamName}");
}

try
{
    // ArgumentOutOfRangeException
    int age = -1;
    ArgumentOutOfRangeException.ThrowIfNegative(age, nameof(age));
}
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine($"Out of range: {ex.ParamName} = {ex.ActualValue}");
}

try
{
    // FormatException
    int n = int.Parse("not a number");
}
catch (FormatException ex)
{
    Console.WriteLine($"Format error: {ex.Message}");
}

try
{
    // DivideByZeroException
    int result = 10 / 0;
}
catch (DivideByZeroException)
{
    Console.WriteLine("Cannot divide by zero");
}

try
{
    // InvalidCastException
    object obj = "hello";
    int n = (int)obj;
}
catch (InvalidCastException ex)
{
    Console.WriteLine($"Cast error: {ex.Message}");
}
```

---

## 3. throw vs throw ex

```csharp
// ===== throw ex - ไม่แนะนำ =====
// รีเซ็ต stack trace! ทำให้หาต้นตอยาก
void BadRethrow()
{
    try
    {
        throw new InvalidOperationException("Original error");
    }
    catch (Exception ex)
    {
        // ❌ throw ex: ล้าง stack trace - ไม่รู้ว่า error เกิดที่ไหนจริงๆ
        throw ex;
    }
}

// ===== throw - แนะนำ =====
// รักษา stack trace เดิม
void GoodRethrow()
{
    try
    {
        throw new InvalidOperationException("Original error");
    }
    catch (Exception ex)
    {
        // Log ก่อน
        Console.WriteLine($"Logging: {ex.Message}");
        // ✅ throw: คง stack trace เดิมไว้
        throw;
    }
}

// ===== Wrapping Exception =====
// เพิ่ม context โดยใช้ InnerException
void ProcessFile(string path)
{
    try
    {
        var content = File.ReadAllText(path);
        // process...
    }
    catch (IOException ex)
    {
        // ✅ wrap ด้วย inner exception เพื่อเพิ่ม context
        throw new InvalidOperationException(
            $"Failed to process file: {path}", ex);
    }
}

// ===== ExceptionDispatchInfo (Advanced) =====
// Capture และ rethrow exception โดยรักษา stack trace
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? capturedEx = null;

try
{
    throw new InvalidOperationException("Captured error");
}
catch (Exception ex)
{
    capturedEx = ExceptionDispatchInfo.Capture(ex);
}

// rethrow ในภายหลัง (ยังคง stack trace เดิม)
capturedEx?.Throw();
```

---

## 4. Custom Exceptions

```csharp
// ===== สร้าง Custom Exception =====
// Best practice: สืบทอดจาก Exception หรือ ApplicationException

// Custom Exception พื้นฐาน
public class ValidationException : Exception
{
    public ValidationException()
        : base("Validation failed") { }

    public ValidationException(string message)
        : base(message) { }

    public ValidationException(string message, Exception innerException)
        : base(message, innerException) { }
}

// Custom Exception พร้อม Properties
public class BusinessRuleException : Exception
{
    public string RuleCode { get; }
    public string EntityName { get; }
    public object? EntityId { get; }

    public BusinessRuleException(
        string ruleCode,
        string entityName,
        object? entityId,
        string message)
        : base(message)
    {
        RuleCode = ruleCode;
        EntityName = entityName;
        EntityId = entityId;
    }

    public BusinessRuleException(
        string ruleCode,
        string entityName,
        object? entityId,
        string message,
        Exception innerException)
        : base(message, innerException)
    {
        RuleCode = ruleCode;
        EntityName = entityName;
        EntityId = entityId;
    }

    public override string ToString() =>
        $"[{RuleCode}] {EntityName}#{EntityId}: {Message}";
}

// Domain-Specific Exceptions
public class InsufficientStockException : Exception
{
    public string ProductName { get; }
    public int Available { get; }
    public int Requested { get; }

    public InsufficientStockException(string productName, int available, int requested)
        : base($"Insufficient stock for '{productName}': " +
               $"requested {requested}, available {available}")
    {
        ProductName = productName;
        Available = available;
        Requested = requested;
    }
}

public class ProductNotFoundException : Exception
{
    public int ProductId { get; }

    public ProductNotFoundException(int productId)
        : base($"Product with ID {productId} was not found")
    {
        ProductId = productId;
    }
}

public class PaymentException : Exception
{
    public string TransactionId { get; }
    public decimal Amount { get; }
    public string Reason { get; }

    public PaymentException(string transactionId, decimal amount, string reason)
        : base($"Payment failed for transaction {transactionId}: {reason}")
    {
        TransactionId = transactionId;
        Amount = amount;
        Reason = reason;
    }
}

// ===== ใช้งาน Custom Exceptions =====
public class OrderService
{
    private Dictionary<int, (string Name, int Stock, decimal Price)> _products = new()
    {
        [1] = ("Laptop", 5, 35000m),
        [2] = ("Mouse", 2, 890m),
        [3] = ("Keyboard", 0, 1500m), // out of stock
    };

    public void PlaceOrder(int productId, int quantity)
    {
        if (!_products.TryGetValue(productId, out var product))
            throw new ProductNotFoundException(productId);

        if (quantity <= 0)
            throw new ArgumentOutOfRangeException(nameof(quantity),
                "Quantity must be positive");

        if (product.Stock < quantity)
            throw new InsufficientStockException(product.Name, product.Stock, quantity);

        // Process order...
        Console.WriteLine($"Order placed: {quantity}x {product.Name}");
    }

    public void ProcessPayment(string transactionId, decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount));

        // Simulate payment failure
        if (amount > 100000)
            throw new PaymentException(transactionId, amount,
                "Amount exceeds daily limit");

        Console.WriteLine($"Payment processed: {amount:C}");
    }
}

// Test
var service = new OrderService();
try
{
    service.PlaceOrder(3, 1); // out of stock
}
catch (InsufficientStockException ex)
{
    Console.WriteLine($"Stock error: {ex.Message}");
    Console.WriteLine($"  Product: {ex.ProductName}");
    Console.WriteLine($"  Available: {ex.Available}, Requested: {ex.Requested}");
}
catch (ProductNotFoundException ex)
{
    Console.WriteLine($"Not found: Product #{ex.ProductId}");
}
```

---

## 5. Multiple catch blocks

```csharp
// Multiple catch - ลำดับสำคัญ! Specific ก่อน General
void ProcessInput(string input)
{
    try
    {
        if (string.IsNullOrEmpty(input))
            throw new ArgumentNullException(nameof(input));

        int value = int.Parse(input);

        if (value == 0)
            throw new DivideByZeroException();

        int result = 100 / value;
        Console.WriteLine($"Result: {result}");
    }
    catch (ArgumentNullException ex)
    {
        // จัดการ specific ก่อน
        Console.WriteLine($"Input is null/empty: {ex.ParamName}");
    }
    catch (FormatException)
    {
        Console.WriteLine($"'{input}' is not a valid number");
    }
    catch (DivideByZeroException)
    {
        Console.WriteLine("Cannot divide by zero");
    }
    catch (OverflowException)
    {
        Console.WriteLine("Number is too large");
    }
    catch (Exception ex) when (ex is IOException or UnauthorizedAccessException)
    {
        // Multiple types ใน single catch
        Console.WriteLine($"IO/Access error: {ex.Message}");
    }
    catch (Exception ex)
    {
        // Catch-all - ควรมีไว้
        Console.WriteLine($"Unexpected error: {ex.GetType().Name}: {ex.Message}");
    }
}

ProcessInput(null);      // ArgumentNullException
ProcessInput("abc");     // FormatException
ProcessInput("0");       // DivideByZeroException
ProcessInput("5");       // Result: 20

// ===== catch หลาย types ด้วย pattern matching (C# 9+) =====
try
{
    // ...
}
catch (Exception ex) when (ex is ArgumentException or InvalidOperationException)
{
    Console.WriteLine($"Validation or operation error: {ex.Message}");
}
```

---

## 6. Exception Filters (when)

```csharp
// when clause: กรอง exception ด้วยเงื่อนไขเพิ่มเติม
// ข้อดี: ถ้า filter false, ยังคง stack trace เดิม (ไม่เหมือน catch + throw)

int errorCode = 404;

try
{
    throw new HttpRequestException($"HTTP Error {errorCode}");
}
catch (HttpRequestException ex) when (ex.Message.Contains("404"))
{
    Console.WriteLine("Page not found - showing 404 page");
}
catch (HttpRequestException ex) when (ex.Message.Contains("500"))
{
    Console.WriteLine("Server error - retrying...");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"Other HTTP error: {ex.Message}");
}

// ===== Exception Filter สำหรับ Logging =====
// Clever trick: ใช้ filter สำหรับ side effects โดยไม่ catch
static bool LogException(Exception ex)
{
    Console.WriteLine($"[LOG] {DateTime.Now}: {ex.GetType().Name}: {ex.Message}");
    return false; // return false: ไม่ catch exception, แค่ log
}

try
{
    throw new InvalidOperationException("Test for logging");
}
catch (Exception ex) when (LogException(ex))
{
    // จะไม่เข้าที่นี่เพราะ LogException คืน false
    Console.WriteLine("This won't execute");
}
// Exception ยังคง propagate ต่อ

// ===== Transient Error Retry =====
public static async Task<T> WithRetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    Func<Exception, bool>? shouldRetry = null)
{
    shouldRetry ??= ex => ex is TimeoutException or HttpRequestException;
    int attempt = 0;

    while (true)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (shouldRetry(ex) && attempt < maxRetries)
        {
            attempt++;
            Console.WriteLine($"Attempt {attempt} failed: {ex.Message}. Retrying...");
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)));
        }
    }
}
```

---

## 7. AggregateException

AggregateException ใช้กับ parallel operations และ async code

```csharp
using System.Threading.Tasks;

// AggregateException ใน Task.WaitAll / Task.WhenAll
try
{
    var tasks = new[]
    {
        Task.Run(() => throw new InvalidOperationException("Error in task 1")),
        Task.Run(() => { }),
        Task.Run(() => throw new ArgumentException("Error in task 3")),
    };

    Task.WaitAll(tasks); // รวม exceptions จากหลาย tasks
}
catch (AggregateException ae)
{
    // Flatten รวม nested AggregateExceptions
    foreach (Exception inner in ae.Flatten().InnerExceptions)
    {
        Console.WriteLine($"Task error: {inner.GetType().Name}: {inner.Message}");
    }
}

// Handle แบบ selective
try
{
    Task.WaitAll(
        Task.Run(() => throw new InvalidOperationException("Op error")),
        Task.Run(() => throw new ArgumentException("Arg error"))
    );
}
catch (AggregateException ae)
{
    ae.Handle(ex =>
    {
        if (ex is InvalidOperationException)
        {
            Console.WriteLine($"Handled: {ex.Message}");
            return true; // handled
        }
        return false; // not handled - rethrow
    });
}

// ===== async/await ===== (C# จัดการ unwrap ให้อัตโนมัติ)
async Task ProcessAsync()
{
    var tasks = new[]
    {
        Task.Run<int>(() => { throw new InvalidOperationException("Async error"); }),
        Task.Run<int>(() => 42),
    };

    try
    {
        // Task.WhenAll ใช้ await ได้
        int[] results = await Task.WhenAll(tasks);
    }
    catch (InvalidOperationException ex)
    {
        // C# unwrap เป็น first exception โดยอัตโนมัติ
        Console.WriteLine($"Caught: {ex.Message}");
    }
}
```

---

## 8. Global Exception Handler

```csharp
// ===== Console Application =====
// จัดการ unhandled exceptions ระดับ global

AppDomain.CurrentDomain.UnhandledException += (sender, e) =>
{
    var ex = (Exception)e.ExceptionObject;
    Console.Error.WriteLine($"FATAL: Unhandled exception: {ex.Message}");
    Console.Error.WriteLine(ex.StackTrace);
    // Log to file, send alert, etc.
    Environment.Exit(1);
};

TaskScheduler.UnobservedTaskException += (sender, e) =>
{
    Console.Error.WriteLine($"Unobserved task exception: {e.Exception.Message}");
    e.SetObserved(); // ป้องกัน crash
};

// ===== ASP.NET Core Global Handler =====
// ใน Program.cs:

/*
var app = builder.Build();

// Development: Detailed error page
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    // Production: Custom error page
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// Middleware สำหรับ API
app.UseExceptionHandler(handler =>
{
    handler.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        context.Response.ContentType = "application/json";

        (int statusCode, string message) = exception switch
        {
            NotFoundException => (404, exception.Message),
            ValidationException => (400, exception.Message),
            UnauthorizedAccessException => (401, "Unauthorized"),
            _ => (500, "An error occurred")
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new { error = message });
    });
});
*/

// ===== Middleware-based Error Handler =====
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;

    public ErrorHandlingMiddleware(RequestDelegate next, ILogger<ErrorHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";

        var (statusCode, message) = exception switch
        {
            ArgumentNullException => (400, "Required parameter is missing"),
            ArgumentException => (400, exception.Message),
            KeyNotFoundException => (404, "Resource not found"),
            UnauthorizedAccessException => (403, "Access denied"),
            NotImplementedException => (501, "Feature not implemented"),
            _ => (500, "Internal server error")
        };

        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new
        {
            status = statusCode,
            error = message,
            timestamp = DateTime.UtcNow
        });
    }
}
```

---

## 9. โปรแกรมตัวอย่าง: File Processing with Error Handling

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text.Json;

namespace FileProcessing
{
    // Custom exceptions สำหรับ file processing
    public class FileProcessingException : Exception
    {
        public string FilePath { get; }
        public int LineNumber { get; }

        public FileProcessingException(string filePath, int lineNumber, string message)
            : base($"Error in '{filePath}' at line {lineNumber}: {message}")
        {
            FilePath = filePath;
            LineNumber = lineNumber;
        }

        public FileProcessingException(string filePath, int lineNumber, string message,
            Exception innerException)
            : base($"Error in '{filePath}' at line {lineNumber}: {message}", innerException)
        {
            FilePath = filePath;
            LineNumber = lineNumber;
        }
    }

    public class DataValidationException : Exception
    {
        public string FieldName { get; }
        public object? InvalidValue { get; }

        public DataValidationException(string fieldName, object? value, string message)
            : base($"Validation failed for '{fieldName}' = '{value}': {message}")
        {
            FieldName = fieldName;
            InvalidValue = value;
        }
    }

    public record StudentRecord(
        int Id,
        string Name,
        string Email,
        int Age,
        double GPA
    );

    public class ProcessingResult
    {
        public List<StudentRecord> Successful { get; } = new();
        public List<(int Line, string Error)> Errors { get; } = new();
        public int TotalLines { get; set; }

        public double SuccessRate =>
            TotalLines > 0 ? (double)Successful.Count / TotalLines * 100 : 0;
    }

    public class StudentFileProcessor
    {
        private readonly string _logPath;

        public StudentFileProcessor(string logPath = "processing.log")
        {
            _logPath = logPath;
        }

        public ProcessingResult ProcessFile(string filePath)
        {
            var result = new ProcessingResult();

            // Validate file exists
            if (!File.Exists(filePath))
                throw new FileNotFoundException($"File not found: {filePath}", filePath);

            // Check extension
            if (Path.GetExtension(filePath).ToLower() != ".csv")
                throw new NotSupportedException($"Only CSV files are supported. Got: {Path.GetExtension(filePath)}");

            try
            {
                using var reader = new StreamReader(filePath);
                using var logWriter = new StreamWriter(_logPath, append: true);

                logWriter.WriteLine($"\n=== Processing: {filePath} ({DateTime.Now}) ===");

                string? line = reader.ReadLine(); // Skip header
                int lineNum = 1;

                while ((line = reader.ReadLine()) != null)
                {
                    lineNum++;
                    result.TotalLines++;

                    try
                    {
                        var student = ParseLine(line, lineNum);
                        ValidateStudent(student);
                        result.Successful.Add(student);
                        logWriter.WriteLine($"[OK] Line {lineNum}: {student.Name}");
                    }
                    catch (FileProcessingException ex)
                    {
                        result.Errors.Add((lineNum, ex.Message));
                        logWriter.WriteLine($"[ERROR] {ex.Message}");
                        // ไม่ rethrow - continue processing next line
                    }
                    catch (DataValidationException ex)
                    {
                        result.Errors.Add((lineNum, ex.Message));
                        logWriter.WriteLine($"[VALIDATION] Line {lineNum}: {ex.Message}");
                    }
                }

                logWriter.WriteLine($"Summary: {result.Successful.Count}/{result.TotalLines} OK");
            }
            catch (UnauthorizedAccessException ex)
            {
                throw new InvalidOperationException(
                    $"Cannot read file '{filePath}': Permission denied", ex);
            }
            catch (IOException ex)
            {
                throw new InvalidOperationException(
                    $"IO error while reading '{filePath}'", ex);
            }

            return result;
        }

        private StudentRecord ParseLine(string line, int lineNum)
        {
            try
            {
                string[] parts = line.Split(',');
                if (parts.Length != 5)
                    throw new FileProcessingException(
                        "input.csv", lineNum,
                        $"Expected 5 fields, got {parts.Length}");

                int id = int.Parse(parts[0].Trim());
                string name = parts[1].Trim();
                string email = parts[2].Trim();
                int age = int.Parse(parts[3].Trim());
                double gpa = double.Parse(parts[4].Trim());

                return new StudentRecord(id, name, email, age, gpa);
            }
            catch (FormatException ex)
            {
                throw new FileProcessingException(
                    "input.csv", lineNum,
                    "Invalid number format", ex);
            }
            catch (FileProcessingException)
            {
                throw; // rethrow as-is
            }
        }

        private void ValidateStudent(StudentRecord student)
        {
            if (string.IsNullOrWhiteSpace(student.Name))
                throw new DataValidationException("Name", student.Name, "Cannot be empty");

            if (student.Age < 15 || student.Age > 100)
                throw new DataValidationException("Age", student.Age,
                    "Must be between 15 and 100");

            if (student.GPA < 0 || student.GPA > 4.0)
                throw new DataValidationException("GPA", student.GPA,
                    "Must be between 0.0 and 4.0");

            if (!student.Email.Contains('@'))
                throw new DataValidationException("Email", student.Email,
                    "Invalid email format");
        }

        public void ExportResults(ProcessingResult result, string outputPath)
        {
            try
            {
                var options = new JsonSerializerOptions { WriteIndented = true };
                string json = JsonSerializer.Serialize(result.Successful, options);
                File.WriteAllText(outputPath, json);
                Console.WriteLine($"Results exported to: {outputPath}");
            }
            catch (Exception ex) when (ex is IOException or UnauthorizedAccessException)
            {
                Console.Error.WriteLine($"Cannot export results: {ex.Message}");
                // ไม่ rethrow - export failure ไม่ควร crash โปรแกรม
            }
        }
    }

    class Program
    {
        static void Main()
        {
            Console.WriteLine("=== File Processing Demo ===\n");

            // สร้าง test CSV file
            string csvPath = "students.csv";
            CreateTestFile(csvPath);

            var processor = new StudentFileProcessor("processing.log");

            try
            {
                var result = processor.ProcessFile(csvPath);

                Console.WriteLine($"Processing Complete!");
                Console.WriteLine($"Success: {result.Successful.Count}/{result.TotalLines}");
                Console.WriteLine($"Success Rate: {result.SuccessRate:F1}%");

                if (result.Errors.Any())
                {
                    Console.WriteLine($"\nErrors ({result.Errors.Count}):");
                    foreach (var (line, error) in result.Errors)
                        Console.WriteLine($"  Line {line}: {error}");
                }

                Console.WriteLine("\nSuccessful Records:");
                foreach (var s in result.Successful.OrderBy(s => s.Name))
                    Console.WriteLine($"  [{s.Id}] {s.Name,-15} Age: {s.Age}, GPA: {s.GPA:F2}");

                // Export
                processor.ExportResults(result, "output.json");

                // Statistics (ด้วย LINQ)
                if (result.Successful.Any())
                {
                    Console.WriteLine("\nStatistics:");
                    Console.WriteLine($"  Average GPA: {result.Successful.Average(s => s.GPA):F2}");
                    Console.WriteLine($"  Highest GPA: {result.Successful.Max(s => s.GPA):F2}");
                    Console.WriteLine($"  Average Age: {result.Successful.Average(s => s.Age):F1}");
                }
            }
            catch (FileNotFoundException ex)
            {
                Console.Error.WriteLine($"File not found: {ex.FileName}");
            }
            catch (NotSupportedException ex)
            {
                Console.Error.WriteLine($"Unsupported format: {ex.Message}");
            }
            catch (InvalidOperationException ex)
            {
                Console.Error.WriteLine($"Processing failed: {ex.Message}");
                if (ex.InnerException != null)
                    Console.Error.WriteLine($"  Caused by: {ex.InnerException.Message}");
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"Unexpected error: {ex.GetType().Name}: {ex.Message}");
                Console.Error.WriteLine(ex.StackTrace);
            }
            finally
            {
                // Cleanup temp files
                TryDeleteFile(csvPath);
                Console.WriteLine("\nCleanup complete");
            }
        }

        static void CreateTestFile(string path)
        {
            var lines = new[]
            {
                "Id,Name,Email,Age,GPA",                      // header
                "1,Alice Johnson,alice@example.com,20,3.8",   // valid
                "2,Bob Smith,bob@example.com,22,3.2",         // valid
                "3,,invalid-email,25,2.9",                    // invalid name
                "4,Charlie Brown,charlie@test.com,19,4.1",   // GPA out of range
                "5,Diana Prince,diana@test.com,21,3.5",       // valid
                "6,Eve Wilson,eve@test.com,150,3.0",           // age out of range
                "7,Frank Miller,frank@test.com,23,3.7",       // valid
                "8,invalid_data",                              // too few fields
                "9,Grace Lee,grace@test.com,20,2.8",          // valid
            };
            File.WriteAllLines(path, lines);
        }

        static void TryDeleteFile(string path)
        {
            try
            {
                if (File.Exists(path)) File.Delete(path);
            }
            catch (Exception ex) when (ex is IOException or UnauthorizedAccessException)
            {
                // Log แต่ไม่ crash
                Console.Error.WriteLine($"Cannot delete {path}: {ex.Message}");
            }
        }
    }
}
```

---

## Exercises

### Exercise 1: Safe Calculator
สร้าง calculator ที่จัดการ exceptions ทั้งหมด:
- DivideByZeroException
- OverflowException
- FormatException
- Stack overflow (recursive factorial ด้วยจำนวนใหญ่)

### Exercise 2: HTTP Client Retry
สร้าง HTTP client wrapper ที่:
- Retry เมื่อเกิด transient errors
- ใช้ Exception filter
- Circuit breaker pattern

### Exercise 3: Transaction with Rollback
สร้าง transaction system ที่:
- ถ้า exception เกิดระหว่าง transaction ให้ rollback
- Log ทุก exception พร้อม context
- Custom exceptions สำหรับ each failure type

---

## สรุป

- ✅ `try-catch-finally`: catch จัดการ exception, finally cleanup เสมอ
- ✅ จัดลำดับ catch จาก Specific ไป General (ลำดับสำคัญ!)
- ✅ `throw` รักษา stack trace, `throw ex` ลบ - ใช้ `throw` เสมอ
- ✅ Custom exceptions: สืบทอด Exception, เพิ่ม properties ที่มีความหมาย
- ✅ Exception filters (`when`) กรอง exception โดยไม่ catch unwanted
- ✅ `AggregateException.Flatten()` จัดการ multiple exceptions จาก parallel ops
- ✅ Global handler: `AppDomain.UnhandledException` สำหรับ last-resort
- ✅ ใช้ `using` แทน try/finally สำหรับ IDisposable resources
- ✅ ห้าม swallow exceptions (catch แล้วไม่ทำอะไร)
- ✅ Log exception พร้อม context ก่อน rethrow หรือ handle

## Part ถัดไป
**Part 027** จะพูดถึง File I/O: File class, FileStream, StreamReader/Writer, Directory operations, JSON serialization และ CSV processing

---
*Part 026/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
