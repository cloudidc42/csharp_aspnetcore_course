# Part 034: Attributes

## เนื้อหาใน Part นี้
- Attribute คืออะไร
- Built-in Attributes: [Obsolete], [Serializable], [DllImport]
- Validation Attributes: [Required], [Range], [RegularExpression]
- สร้าง Custom Attribute
- อ่าน Attribute ด้วย Reflection
- โปรแกรมตัวอย่าง: Validation Framework

---

## 1. Attribute คืออะไร

Attribute คือ metadata ที่เพิ่มเข้าไปใน code elements (class, method, property, parameter, etc.) ในรูป `[AttributeName]`

```csharp
using System;

// Attribute บน class
[Serializable]
[Obsolete("ใช้ NewClass แทน")]
public class OldClass
{
    // Attribute บน property
    [Required]
    public string Name { get; set; } = string.Empty;
    
    // Attribute บน method
    [Obsolete("ใช้ BetterMethod แทน")]
    public void OldMethod() { }
    
    // Attribute บน parameter
    public void DoSomething([Required] string input) { }
    
    // Attribute หลายตัว
    [HttpGet]
    [Route("api/data")]
    [Authorize]
    public void ApiMethod() { }
}

// Attribute พร้อม parameters
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true)]
public class MyCustomAttribute : Attribute
{
    public string Description { get; }
    public int Priority { get; set; }
    
    public MyCustomAttribute(string description)
    {
        Description = description;
    }
}

// ใช้ Custom Attribute
[MyCustomAttribute("หลัก", Priority = 1)]
[MyCustomAttribute("รอง", Priority = 2)]
public class MyService
{
    [MyCustomAttribute("method description")]
    public void MyMethod() { }
}
```

### โครงสร้างของ Attribute

```csharp
using System;

// Attribute class สืบทอดจาก System.Attribute
[AttributeUsage(
    AttributeTargets.All,    // สามารถใช้กับ element ประเภทใดได้บ้าง
    AllowMultiple = false,   // ใช้ซ้ำได้หรือไม่
    Inherited = true         // subclass รับ attribute ได้หรือไม่
)]
public class DocumentationAttribute : Attribute
{
    // Positional parameter (required)
    public string Author { get; }
    
    // Named parameter (optional)
    public string? Version { get; set; }
    public string? Description { get; set; }
    
    public DocumentationAttribute(string author)
    {
        Author = author;
    }
}

// ใช้งาน
[Documentation("สมชาย", Version = "1.0", Description = "Class หลัก")]
public class MainClass
{
    [Documentation("สมหญิง")]
    public void SomeMethod() { }
}
```

---

## 2. Built-in Attributes

### [Obsolete]

```csharp
using System;

public class ObsoleteExamples
{
    // แสดง warning
    [Obsolete("ใช้ NewMethod แทน")]
    public static int OldMethod(int x) => x * 2;
    
    // ทำให้ compile error
    [Obsolete("ห้ามใช้ต่อแล้ว", error: true)]
    public static int DangerousMethod() => 0;
    
    // บอก URL เพิ่มเติม
    [Obsolete("ใช้ BetterClass (https://docs.example.com/better-class) แทน")]
    public static string ConvertData(string data) => data;
    
    // ตัวใหม่
    public static int NewMethod(int x) => x * 2;
    
    static void Main()
    {
        // Warning: 'OldMethod' is obsolete: 'ใช้ NewMethod แทน'
        int result = OldMethod(5);
        
        // ✅ ไม่มี warning
        int result2 = NewMethod(5);
        
        Console.WriteLine($"{result}, {result2}");
    }
}
```

### [Serializable] และ JSON Attributes

```csharp
using System;
using System.Runtime.Serialization;
using System.Text.Json;
using System.Text.Json.Serialization;

// Binary serialization (legacy)
[Serializable]
public class LegacyData
{
    public string Name { get; set; } = string.Empty;
    
    [NonSerialized]
    private string _password = string.Empty; // ไม่ serialize
}

// JSON serialization
public class UserDto
{
    [JsonPropertyName("user_id")]
    public int UserId { get; set; }
    
    [JsonPropertyName("full_name")]
    public string FullName { get; set; } = string.Empty;
    
    [JsonIgnore]
    public string Password { get; set; } = string.Empty; // ไม่ serialize
    
    [JsonPropertyOrder(1)]
    public string Email { get; set; } = string.Empty;
    
    [JsonConverter(typeof(JsonStringEnumConverter))]
    public UserRole Role { get; set; }
    
    [JsonInclude]
    public DateTime CreatedAt { get; init; } = DateTime.Now;
}

public enum UserRole { Admin, User, Guest }

class SerializationDemo
{
    static void Main()
    {
        var user = new UserDto
        {
            UserId = 1,
            FullName = "สมชาย ใจดี",
            Email = "somchai@example.com",
            Password = "secret123",
            Role = UserRole.Admin
        };
        
        var options = new JsonSerializerOptions { WriteIndented = true };
        string json = JsonSerializer.Serialize(user, options);
        Console.WriteLine(json);
        // Password จะไม่ปรากฏใน JSON
        
        var deserialized = JsonSerializer.Deserialize<UserDto>(json);
        Console.WriteLine($"User: {deserialized?.FullName}");
    }
}
```

### [DllImport] - Platform Invoke

```csharp
using System;
using System.Runtime.InteropServices;

public class NativeMethods
{
    // Windows API
    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern bool Beep(uint frequency, uint duration);
    
    [DllImport("user32.dll", CharSet = CharSet.Auto)]
    public static extern int MessageBox(
        IntPtr hWnd,
        string text,
        string caption,
        uint type
    );
    
    // Linux/Mac
    [DllImport("libc", EntryPoint = "printf")]
    public static extern int PrintF(string format);
    
    // กำหนด calling convention
    [DllImport("mylibrary.dll",
        CallingConvention = CallingConvention.Cdecl,
        CharSet = CharSet.Unicode,
        ExactSpelling = true)]
    public static extern int ProcessString([MarshalAs(UnmanagedType.LPWStr)] string input);
}

// LibraryImport (.NET 7+) - Source generated
public static partial class NativeMethodsModern
{
    [LibraryImport("kernel32.dll")]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static partial bool Beep(uint frequency, uint duration);
}
```

### [Conditional]

```csharp
using System;
using System.Diagnostics;

public class ConditionalExample
{
    // Method จะถูกเรียกเฉพาะเมื่อ compile symbol ถูกกำหนด
    [Conditional("DEBUG")]
    public static void DebugLog(string message)
    {
        Console.WriteLine($"[DEBUG] {message}");
    }
    
    [Conditional("TRACE")]
    public static void TraceLog(string message)
    {
        Trace.WriteLine($"[TRACE] {message}");
    }
    
    static void Main()
    {
        DebugLog("นี่จะแสดงเฉพาะ Debug build"); // ไม่แสดงใน Release
        TraceLog("Trace message");
        Console.WriteLine("Hello World");
    }
}
```

### [CallerMemberName] และ Caller Info Attributes

```csharp
using System;
using System.Runtime.CompilerServices;

public class CallerInfoExample
{
    // ดูข้อมูลของ caller
    public static void Log(
        string message,
        [CallerMemberName] string memberName = "",
        [CallerFilePath] string filePath = "",
        [CallerLineNumber] int lineNumber = 0)
    {
        Console.WriteLine($"[{memberName}:{lineNumber}] {message}");
        Console.WriteLine($"  File: {System.IO.Path.GetFileName(filePath)}");
    }
    
    // INotifyPropertyChanged pattern
    public class Person : System.ComponentModel.INotifyPropertyChanged
    {
        public event System.ComponentModel.PropertyChangedEventHandler? PropertyChanged;
        
        private string _name = string.Empty;
        public string Name
        {
            get => _name;
            set
            {
                _name = value;
                OnPropertyChanged(); // ส่งชื่อ property อัตโนมัติ
            }
        }
        
        protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
        {
            PropertyChanged?.Invoke(this, new System.ComponentModel.PropertyChangedEventArgs(propertyName));
        }
    }
    
    static void Main()
    {
        Log("เริ่มทำงาน"); // Output: [Main:...] เริ่มทำงาน
        
        var person = new Person();
        person.PropertyChanged += (s, e) =>
            Console.WriteLine($"Property '{e.PropertyName}' เปลี่ยนแปลง");
        
        person.Name = "สมชาย"; // แสดง: Property 'Name' เปลี่ยนแปลง
    }
}
```

---

## 3. Validation Attributes

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Text.RegularExpressions;

// Data Annotations ใช้ใน ASP.NET Core, EF Core, WPF
public class RegisterRequest
{
    [Required(ErrorMessage = "กรุณากรอกชื่อผู้ใช้")]
    [StringLength(50, MinimumLength = 3, ErrorMessage = "ชื่อผู้ใช้ต้องมี 3-50 ตัวอักษร")]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "ชื่อผู้ใช้ใช้ได้แค่ a-z, A-Z, 0-9, _")]
    public string Username { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณากรอกรหัสผ่าน")]
    [MinLength(8, ErrorMessage = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")]
    [DataType(DataType.Password)]
    public string Password { get; set; } = string.Empty;
    
    [Compare("Password", ErrorMessage = "รหัสผ่านไม่ตรงกัน")]
    public string ConfirmPassword { get; set; } = string.Empty;
    
    [Required(ErrorMessage = "กรุณากรอก Email")]
    [EmailAddress(ErrorMessage = "รูปแบบ Email ไม่ถูกต้อง")]
    [MaxLength(100)]
    public string Email { get; set; } = string.Empty;
    
    [Range(13, 120, ErrorMessage = "อายุต้องอยู่ระหว่าง 13-120 ปี")]
    public int Age { get; set; }
    
    [Url(ErrorMessage = "URL ไม่ถูกต้อง")]
    public string? Website { get; set; }
    
    [Phone(ErrorMessage = "เบอร์โทรศัพท์ไม่ถูกต้อง")]
    public string? PhoneNumber { get; set; }
    
    [CreditCard(ErrorMessage = "หมายเลขบัตรเครดิตไม่ถูกต้อง")]
    public string? CreditCard { get; set; }
    
    [Required]
    public List<string> Tags { get; set; } = new();
}

// ตรวจสอบด้วย Validator
class ValidationDemo
{
    static void ValidateModel(object model)
    {
        var context = new ValidationContext(model);
        var results = new List<ValidationResult>();
        
        bool isValid = Validator.TryValidateObject(model, context, results, validateAllProperties: true);
        
        if (isValid)
        {
            Console.WriteLine("✅ ข้อมูลถูกต้องทั้งหมด");
        }
        else
        {
            Console.WriteLine("❌ พบข้อผิดพลาด:");
            foreach (var result in results)
            {
                Console.WriteLine($"  - {result.ErrorMessage}");
                Console.WriteLine($"    Fields: {string.Join(", ", result.MemberNames)}");
            }
        }
    }
    
    static void Main()
    {
        var request = new RegisterRequest
        {
            Username = "ab",           // ❌ สั้นเกินไป
            Password = "pass",          // ❌ สั้นเกินไป
            ConfirmPassword = "password", // ❌ ไม่ตรงกัน
            Email = "not-an-email",    // ❌ รูปแบบผิด
            Age = 5                    // ❌ ต่ำกว่า 13
        };
        
        ValidateModel(request);
        
        Console.WriteLine("\n--- Valid Request ---");
        var validRequest = new RegisterRequest
        {
            Username = "somchai123",
            Password = "SecurePass123!",
            ConfirmPassword = "SecurePass123!",
            Email = "somchai@example.com",
            Age = 25,
            Tags = new List<string> { "user" }
        };
        
        ValidateModel(validRequest);
    }
}
```

### Custom Validation Attributes

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;

// Custom attribute สำหรับตรวจสอบวันที่ในอนาคต
[AttributeUsage(AttributeTargets.Property)]
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is DateTime date)
        {
            if (date <= DateTime.Now)
            {
                return new ValidationResult(
                    ErrorMessage ?? "วันที่ต้องเป็นอนาคต",
                    new[] { validationContext.MemberName! }
                );
            }
        }
        
        return ValidationResult.Success;
    }
}

// Custom attribute สำหรับ Thai ID
[AttributeUsage(AttributeTargets.Property)]
public class ThaiIdAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is not string id || string.IsNullOrEmpty(id))
            return ValidationResult.Success; // ให้ Required จัดการ
        
        // เลขบัตรประชาชนไทย 13 หลัก
        if (!System.Text.RegularExpressions.Regex.IsMatch(id, @"^\d{13}$"))
            return new ValidationResult("เลขบัตรประชาชนต้องมี 13 หลัก");
        
        // ตรวจสอบ checksum
        int sum = 0;
        for (int i = 0; i < 12; i++)
            sum += int.Parse(id[i].ToString()) * (13 - i);
        
        int checkDigit = (11 - (sum % 11)) % 10;
        if (int.Parse(id[12].ToString()) != checkDigit)
            return new ValidationResult("เลขบัตรประชาชนไม่ถูกต้อง");
        
        return ValidationResult.Success;
    }
}

// Custom attribute ที่ใช้ IValidatableObject
public class BookingRequest : IValidatableObject
{
    [Required]
    public DateTime CheckIn { get; set; }
    
    [Required]
    [FutureDate(ErrorMessage = "วันเช็คเอาท์ต้องเป็นอนาคต")]
    public DateTime CheckOut { get; set; }
    
    [Required]
    [ThaiId]
    public string GuestId { get; set; } = string.Empty;
    
    [Range(1, 10, ErrorMessage = "จำนวนผู้เข้าพักต้อง 1-10 คน")]
    public int GuestCount { get; set; }
    
    // IValidatableObject สำหรับ complex validation
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (CheckOut <= CheckIn)
        {
            yield return new ValidationResult(
                "วันเช็คเอาท์ต้องหลังจากวันเช็คอิน",
                new[] { nameof(CheckOut) }
            );
        }
        
        if ((CheckOut - CheckIn).TotalDays > 30)
        {
            yield return new ValidationResult(
                "ไม่สามารถจองเกิน 30 คืน",
                new[] { nameof(CheckIn), nameof(CheckOut) }
            );
        }
        
        if (CheckIn.DayOfWeek == DayOfWeek.Friday && GuestCount > 5)
        {
            yield return new ValidationResult(
                "วันศุกร์รับได้ไม่เกิน 5 คน",
                new[] { nameof(GuestCount) }
            );
        }
    }
}
```

---

## 4. สร้าง Custom Attribute

```csharp
using System;

// Attribute พื้นฐาน
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public class LogAttribute : Attribute
{
    public string Level { get; set; } = "Info";
    public bool LogParameters { get; set; } = false;
    public bool LogReturn { get; set; } = false;
    
    public LogAttribute() { }
    public LogAttribute(string level) { Level = level; }
}

// Attribute สำหรับ caching
[AttributeUsage(AttributeTargets.Method)]
public class CacheAttribute : Attribute
{
    public int DurationSeconds { get; }
    public string? Key { get; set; }
    
    public CacheAttribute(int durationSeconds = 300)
    {
        DurationSeconds = durationSeconds;
    }
}

// Attribute สำหรับ permissions
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true)]
public class RequirePermissionAttribute : Attribute
{
    public string Permission { get; }
    
    public RequirePermissionAttribute(string permission)
    {
        Permission = permission;
    }
}

// Attribute สำหรับ database mapping
[AttributeUsage(AttributeTargets.Class)]
public class TableAttribute : Attribute
{
    public string TableName { get; }
    public string? Schema { get; set; }
    
    public TableAttribute(string tableName)
    {
        TableName = tableName;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class ColumnAttribute : Attribute
{
    public string? ColumnName { get; set; }
    public bool IsPrimaryKey { get; set; }
    public bool IsNullable { get; set; } = true;
    public int MaxLength { get; set; } = -1;
}

// ใช้งาน Custom Attributes
[Table("users", Schema = "dbo")]
[Log("Debug")]
public class UserService
{
    [Cache(300)]
    [Log("Info", LogReturn = true)]
    [RequirePermission("users.read")]
    public User? GetUser(int id)
    {
        return null; // จำลอง
    }
    
    [RequirePermission("users.write")]
    [RequirePermission("admin")]
    public void UpdateUser(User user)
    {
        // จำลอง
    }
}

[Table("users")]
public class User
{
    [Column(IsPrimaryKey = true, IsNullable = false)]
    public int Id { get; set; }
    
    [Column(ColumnName = "full_name", MaxLength = 100, IsNullable = false)]
    public string Name { get; set; } = string.Empty;
    
    [Column(ColumnName = "email_address", MaxLength = 200)]
    public string Email { get; set; } = string.Empty;
    
    [Column(IsNullable = true)]
    public DateTime? DeletedAt { get; set; }
}
```

---

## 5. อ่าน Attribute ด้วย Reflection

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;

class AttributeReader
{
    static void Main()
    {
        // อ่าน attributes จาก class
        var type = typeof(User);
        
        // TableAttribute
        var tableAttr = type.GetCustomAttribute<TableAttribute>();
        if (tableAttr != null)
        {
            Console.WriteLine($"Table: {tableAttr.Schema}.{tableAttr.TableName}");
        }
        
        // อ่าน attributes จาก properties
        foreach (var prop in type.GetProperties())
        {
            var colAttr = prop.GetCustomAttribute<ColumnAttribute>();
            if (colAttr != null)
            {
                string colName = colAttr.ColumnName ?? prop.Name.ToLower();
                Console.WriteLine($"Property: {prop.Name} -> Column: {colName}" +
                    $" (PK={colAttr.IsPrimaryKey}, Nullable={colAttr.IsNullable})");
            }
        }
        
        // อ่าน attributes จาก methods
        var serviceType = typeof(UserService);
        foreach (var method in serviceType.GetMethods(BindingFlags.Public | BindingFlags.Instance))
        {
            var permissions = method.GetCustomAttributes<RequirePermissionAttribute>();
            var perms = permissions.Select(p => p.Permission).ToList();
            
            if (perms.Any())
            {
                Console.WriteLine($"\nMethod: {method.Name}");
                Console.WriteLine($"  Permissions: {string.Join(", ", perms)}");
            }
        }
    }
}

// Attribute Inspector utility
public class AttributeInspector
{
    public static Dictionary<string, List<string>> GetMethodPermissions(Type type)
    {
        var result = new Dictionary<string, List<string>>();
        
        foreach (var method in type.GetMethods(BindingFlags.Public | BindingFlags.Instance))
        {
            var attrs = method.GetCustomAttributes<RequirePermissionAttribute>().ToList();
            if (attrs.Any())
            {
                result[method.Name] = attrs.Select(a => a.Permission).ToList();
            }
        }
        
        return result;
    }
    
    public static IEnumerable<(PropertyInfo Property, ColumnAttribute Column)> GetColumns(Type type)
    {
        return type.GetProperties()
            .Select(p => (Property: p, Column: p.GetCustomAttribute<ColumnAttribute>()!))
            .Where(x => x.Column != null);
    }
    
    public static bool HasPermission(MethodInfo method, string userPermission)
    {
        var requiredPermissions = method.GetCustomAttributes<RequirePermissionAttribute>();
        return requiredPermissions.Any(attr =>
            attr.Permission.Equals(userPermission, StringComparison.OrdinalIgnoreCase));
    }
}
```

---

## 6. โปรแกรมตัวอย่าง: Validation Framework

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Reflection;
using System.Text.RegularExpressions;

// Custom Attributes สำหรับ Validation Framework
[AttributeUsage(AttributeTargets.Property)]
public abstract class ValidatorAttribute : Attribute
{
    public string ErrorMessage { get; set; } = "ข้อมูลไม่ถูกต้อง";
    public abstract bool IsValid(object? value);
}

[AttributeUsage(AttributeTargets.Property)]
public class NotNullAttribute : ValidatorAttribute
{
    public NotNullAttribute() { ErrorMessage = "ค่าต้องไม่เป็น null"; }
    public override bool IsValid(object? value) => value != null;
}

[AttributeUsage(AttributeTargets.Property)]
public class NotEmptyAttribute : ValidatorAttribute
{
    public NotEmptyAttribute() { ErrorMessage = "ค่าต้องไม่ว่างเปล่า"; }
    public override bool IsValid(object? value) =>
        value switch
        {
            string s => !string.IsNullOrWhiteSpace(s),
            System.Collections.ICollection c => c.Count > 0,
            _ => value != null
        };
}

[AttributeUsage(AttributeTargets.Property)]
public class LengthAttribute : ValidatorAttribute
{
    public int Min { get; }
    public int Max { get; }
    
    public LengthAttribute(int min, int max)
    {
        Min = min;
        Max = max;
        ErrorMessage = $"ความยาวต้องอยู่ระหว่าง {min}-{max} ตัวอักษร";
    }
    
    public override bool IsValid(object? value)
    {
        if (value is string s)
            return s.Length >= Min && s.Length <= Max;
        return true;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class RangeValueAttribute : ValidatorAttribute
{
    public double Min { get; }
    public double Max { get; }
    
    public RangeValueAttribute(double min, double max)
    {
        Min = min;
        Max = max;
        ErrorMessage = $"ค่าต้องอยู่ระหว่าง {min}-{max}";
    }
    
    public override bool IsValid(object? value)
    {
        if (value == null) return true;
        double d = Convert.ToDouble(value);
        return d >= Min && d <= Max;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class PatternAttribute : ValidatorAttribute
{
    private readonly Regex _regex;
    
    public PatternAttribute(string pattern)
    {
        _regex = new Regex(pattern, RegexOptions.Compiled);
        ErrorMessage = $"ค่าไม่ตรงตาม pattern: {pattern}";
    }
    
    public override bool IsValid(object? value)
    {
        if (value is string s)
            return _regex.IsMatch(s);
        return true;
    }
}

[AttributeUsage(AttributeTargets.Property)]
public class EmailValidatorAttribute : ValidatorAttribute
{
    public EmailValidatorAttribute() { ErrorMessage = "รูปแบบ Email ไม่ถูกต้อง"; }
    
    public override bool IsValid(object? value)
    {
        if (value is not string email || string.IsNullOrEmpty(email)) return true;
        return Regex.IsMatch(email, @"^[^@\s]+@[^@\s]+\.[^@\s]+$");
    }
}

// Validation Result
public record ValidationError(string PropertyName, string ErrorMessage);

public record ValidationResult2(bool IsValid, List<ValidationError> Errors)
{
    public static ValidationResult2 Success() => new(true, new List<ValidationError>());
    public static ValidationResult2 Failure(List<ValidationError> errors) => new(false, errors);
}

// Validation Engine
public class ModelValidator
{
    public static ValidationResult2 Validate(object model)
    {
        var errors = new List<ValidationError>();
        var type = model.GetType();
        
        foreach (var property in type.GetProperties())
        {
            var value = property.GetValue(model);
            var validators = property.GetCustomAttributes<ValidatorAttribute>();
            
            foreach (var validator in validators)
            {
                if (!validator.IsValid(value))
                {
                    errors.Add(new ValidationError(
                        property.Name,
                        validator.ErrorMessage
                    ));
                }
            }
        }
        
        return errors.Any()
            ? ValidationResult2.Failure(errors)
            : ValidationResult2.Success();
    }
    
    // Validate แบบ generic
    public static ValidationResult2 Validate<T>(T model) where T : notnull
        => Validate((object)model);
}

// Model ที่ใช้ custom attributes
public class ProductRequest
{
    [NotNull]
    [NotEmpty(ErrorMessage = "กรุณากรอกชื่อสินค้า")]
    [Length(2, 100)]
    public string Name { get; set; } = string.Empty;
    
    [NotNull]
    [LengthAttribute(0, 500)]
    public string Description { get; set; } = string.Empty;
    
    [RangeValue(0.01, 999999.99, ErrorMessage = "ราคาต้องอยู่ระหว่าง 0.01-999,999.99")]
    public decimal Price { get; set; }
    
    [RangeValue(0, 9999, ErrorMessage = "จำนวนสต็อกต้อง 0-9,999")]
    public int Stock { get; set; }
    
    [NotEmpty(ErrorMessage = "กรุณาใส่หมวดหมู่")]
    public string Category { get; set; } = string.Empty;
    
    [EmailValidator]
    public string? ContactEmail { get; set; }
    
    [Pattern(@"^\d{13}$", ErrorMessage = "บาร์โค้ดต้องเป็นตัวเลข 13 หลัก")]
    public string? Barcode { get; set; }
}

// Advanced Validator with rules
public class FluentValidator<T>
{
    private readonly List<(string property, Func<T, bool> rule, string message)> _rules = new();
    
    public FluentValidator<T> AddRule(
        string propertyName,
        Func<T, bool> rule,
        string errorMessage)
    {
        _rules.Add((propertyName, rule, errorMessage));
        return this;
    }
    
    public ValidationResult2 Validate(T model)
    {
        var errors = new List<ValidationError>();
        
        foreach (var (property, rule, message) in _rules)
        {
            if (!rule(model))
            {
                errors.Add(new ValidationError(property, message));
            }
        }
        
        return errors.Any()
            ? ValidationResult2.Failure(errors)
            : ValidationResult2.Success();
    }
}

// Main Program
class Program
{
    static void Main()
    {
        Console.WriteLine("===== Validation Framework Demo =====\n");
        
        // Test 1: Invalid product
        var invalid = new ProductRequest
        {
            Name = "A",           // ❌ สั้นเกินไป
            Price = -10,          // ❌ ติดลบ
            Stock = -1,           // ❌ ติดลบ
            ContactEmail = "bad-email", // ❌ รูปแบบผิด
            Barcode = "123"       // ❌ ไม่ใช่ 13 หลัก
        };
        
        var result = ModelValidator.Validate(invalid);
        PrintResult("สินค้าไม่ถูกต้อง", result);
        
        // Test 2: Valid product
        var valid = new ProductRequest
        {
            Name = "iPhone 16 Pro",
            Description = "สมาร์ทโฟนรุ่นใหม่",
            Price = 49900,
            Stock = 100,
            Category = "Electronics",
            ContactEmail = "store@example.com",
            Barcode = "1234567890123"
        };
        
        var result2 = ModelValidator.Validate(valid);
        PrintResult("สินค้าถูกต้อง", result2);
        
        // Test 3: Fluent Validation
        Console.WriteLine("\n--- Fluent Validator ---");
        
        var validator = new FluentValidator<ProductRequest>()
            .AddRule("Name", p => p.Name.Length >= 2, "ชื่อสั้นเกินไป")
            .AddRule("Price", p => p.Price > 0, "ราคาต้องมากกว่า 0")
            .AddRule("Stock", p => p.Stock >= 0, "สต็อกต้องไม่ติดลบ")
            .AddRule("Category", p => !string.IsNullOrEmpty(p.Category), "กรุณาใส่หมวดหมู่");
        
        var result3 = validator.Validate(invalid);
        PrintResult("Fluent Validation", result3);
    }
    
    static void PrintResult(string title, ValidationResult2 result)
    {
        Console.WriteLine($"[{title}]");
        if (result.IsValid)
        {
            Console.WriteLine("  ✅ ข้อมูลถูกต้องทั้งหมด");
        }
        else
        {
            Console.WriteLine($"  ❌ พบ {result.Errors.Count} ข้อผิดพลาด:");
            foreach (var error in result.Errors)
            {
                Console.WriteLine($"    - {error.PropertyName}: {error.ErrorMessage}");
            }
        }
        Console.WriteLine();
    }
}
```

---

## Exercises

### Exercise 1: Cache Attribute Implementation
```csharp
// TODO: สร้าง simple in-memory cache ที่ใช้ CacheAttribute
// เมื่อเรียก method ที่มี [Cache] ให้:
// 1. ตรวจสอบว่ามีใน cache ไหม (ใช้ method signature + parameters เป็น key)
// 2. ถ้ามี return จาก cache
// 3. ถ้าไม่มี execute method แล้ว save ผลลัพธ์ลง cache
// Note: ใช้ MethodBase.GetCurrentMethod() และ Reflection

public class CacheInterceptor
{
    private static readonly Dictionary<string, (object? Value, DateTime Expiry)> _cache = new();
    
    public static T? Execute<T>(Func<T> method, MethodInfo methodInfo)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 2: ORM-like Query Builder
```csharp
// TODO: สร้าง simple ORM query builder ที่อ่าน TableAttribute และ ColumnAttribute
// เพื่อ generate SQL queries

public class QueryBuilder<T> where T : class, new()
{
    // สร้าง SELECT query จาก attributes
    public string BuildSelectQuery()
    {
        // ตัวอย่าง output:
        // SELECT id, full_name, email_address FROM dbo.users
        throw new NotImplementedException();
    }
    
    // สร้าง INSERT query
    public (string Query, Dictionary<string, object?> Parameters) BuildInsertQuery(T entity)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: Permission Middleware
```csharp
// TODO: สร้าง middleware-like class ที่ตรวจสอบ RequirePermissionAttribute
// ก่อน execute method

public class PermissionChecker
{
    private readonly HashSet<string> _userPermissions;
    
    public PermissionChecker(IEnumerable<string> userPermissions)
    {
        _userPermissions = new HashSet<string>(userPermissions);
    }
    
    public bool CanExecute(MethodInfo method)
    {
        throw new NotImplementedException();
    }
    
    public T Execute<T>(object instance, MethodInfo method, params object[] args)
    {
        // ตรวจสอบ permissions แล้วเรียก method
        throw new NotImplementedException();
    }
}
```

---

## สรุป

✅ **Attribute** คือ metadata ที่เพิ่มให้ code elements ใน C#

✅ **Built-in attributes** เช่น [Obsolete], [Serializable], [DllImport] ใช้บ่อยมาก

✅ **JSON attributes** ([JsonPropertyName], [JsonIgnore]) ใช้ควบคุม serialization

✅ **Data Annotations** ([Required], [Range], [EmailAddress]) ใช้ validation

✅ สร้าง **Custom Attribute** โดย inherit จาก System.Attribute

✅ ใช้ **[AttributeUsage]** กำหนดว่า attribute ใช้ได้กับอะไร

✅ อ่าน attributes ด้วย **Reflection** (GetCustomAttribute\<T\>)

✅ **IValidatableObject** ใช้สำหรับ complex validation ที่ต้องใช้หลาย properties

---

## Part ถัดไป

➡️ **Part 035**: Reflection - Type inspection, Dynamic invocation, Plugin systems และ performance considerations

---

*Part 034/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
