# Part 017: Interfaces

## เนื้อหาใน Part นี้
- Interface declaration และโครงสร้าง
- Implementing interfaces
- Multiple interface implementation
- Default interface methods (C# 8+)
- Interface vs Abstract class
- Common .NET interfaces: IComparable, IEnumerable, IDisposable
- โปรแกรมตัวอย่าง: Payment system

---

## 1. Interface คืออะไร?

Interface คือ "สัญญา" (contract) ที่กำหนดว่า class ต้องมี members อะไรบ้าง โดยไม่บอกวิธีการ implement

**Interface บอกว่า "ต้องทำอะไร" ไม่ใช่ "ทำอย่างไร"**

```csharp
// Interface declaration
public interface IShape
{
    // Properties (abstract โดยปริยาย)
    double Area { get; }
    double Perimeter { get; }
    string Color { get; set; }

    // Methods (abstract โดยปริยาย)
    void Draw();
    double Scale(double factor);

    // Static interface member (C# 11+)
    static string GetCategory() => "2D Shape";
}

// Implement interface
public class Circle : IShape
{
    private double _radius;

    public Circle(double radius, string color = "Black")
    {
        _radius = radius;
        Color = color;
    }

    // ต้อง implement ทุก member ของ interface
    public double Area => Math.PI * _radius * _radius;
    public double Perimeter => 2 * Math.PI * _radius;
    public string Color { get; set; }

    public void Draw()
    {
        Console.WriteLine($"⭕ Circle(r={_radius}, color={Color}, area={Area:F2})");
    }

    public double Scale(double factor)
    {
        _radius *= factor;
        return Area;
    }
}

// ใช้งานผ่าน interface type
IShape shape = new Circle(5, "Red");
shape.Draw();
Console.WriteLine($"Area: {shape.Area:F2}");
Console.WriteLine($"Perimeter: {shape.Perimeter:F2}");
```

---

## 2. Interface vs Class

```csharp
// Interface ไม่สามารถ:
// - มี constructor
// - มี instance fields (แต่ static fields ได้ใน C# 8+)
// - มี access modifier ใน members (ทุกอย่าง public โดยปริยาย)

// Interface สามารถ:
// - มี properties, methods, events, indexers
// - มี default implementations (C# 8+)
// - สืบทอดจาก interfaces อื่น
// - มี static members (C# 8+)

public interface IAnimal
{
    // Properties
    string Name { get; }
    int Age { get; }
    bool IsAlive { get; }

    // Methods
    void MakeSound();
    void Eat(string food);

    // Default method (C# 8+)
    void Sleep()
    {
        Console.WriteLine($"{Name} กำลังนอนหลับ...");
    }

    // Default property implementation (C# 8+)
    bool IsAdult => Age >= 2;

    // Static method (C# 8+)
    static string GetKingdom() => "Animalia";
}

// ตัวอย่าง: interface สืบทอดจาก interface
public interface IFlyable
{
    double MaxAltitudeMeters { get; }
    void Fly();
    void Land();
}

public interface ISwimmable
{
    double MaxDepthMeters { get; }
    void Swim();
    void Surface();
}

// Interface สืบทอดหลาย interfaces
public interface IWaterbird : IAnimal, IFlyable, ISwimmable
{
    // เพิ่ม member เฉพาะ waterbird
    void Dive();
}
```

---

## 3. Implementing Multiple Interfaces

```csharp
// Class สามารถ implement หลาย interfaces
public class Duck : IAnimal, IFlyable, ISwimmable
{
    public string Name { get; }
    public int Age { get; }
    public bool IsAlive { get; private set; } = true;

    // IFlyable
    public double MaxAltitudeMeters => 100;
    // ISwimmable
    public double MaxDepthMeters => 0.5;

    public Duck(string name, int age)
    {
        Name = name;
        Age = age;
    }

    // IAnimal methods
    public void MakeSound() => Console.WriteLine($"{Name}: กวั้กๆ!");
    public void Eat(string food) => Console.WriteLine($"{Name} กินอาหาร: {food}");

    // IFlyable methods
    public void Fly() => Console.WriteLine($"{Name} บิน (สูงสุด {MaxAltitudeMeters}m)");
    public void Land() => Console.WriteLine($"{Name} ลงจอด");

    // ISwimmable methods
    public void Swim() => Console.WriteLine($"{Name} ว่ายน้ำ (ลึกสุด {MaxDepthMeters}m)");
    public void Surface() => Console.WriteLine($"{Name} ขึ้นผิวน้ำ");
}

public class Submarine : ISwimmable
{
    public string Name { get; }
    public double MaxDepthMeters => 500;

    public Submarine(string name) { Name = name; }

    public void Swim() => Console.WriteLine($"{Name} ดำน้ำ...");
    public void Surface() => Console.WriteLine($"{Name} โผล่ขึ้นผิวน้ำ");
}

// Polymorphism ผ่าน interfaces
var duck = new Duck("Donald", 3);
var sub = new Submarine("USS Enterprise");

// ใช้ interface type
ISwimmable[] swimmers = [duck, sub];
foreach (ISwimmable swimmer in swimmers)
{
    swimmer.Swim();
    // เข้าถึงได้เฉพาะ ISwimmable members
}

// ตรวจสอบ capabilities
void TryAll(object obj)
{
    if (obj is IAnimal a) a.MakeSound();
    if (obj is IFlyable f) f.Fly();
    if (obj is ISwimmable s) s.Swim();
}

TryAll(duck);       // ทำได้ทั้งหมด
TryAll(sub);        // ว่ายน้ำได้อย่างเดียว
```

---

## 4. Explicit Interface Implementation

เมื่อมีสมาชิกที่ชื่อเดียวกันจากหลาย interfaces

```csharp
public interface ILogger
{
    void Log(string message);
    string Name { get; }
}

public interface IConsoleOutput
{
    void Log(string message); // ชื่อเดียวกับ ILogger.Log!
    string Name { get; }      // ชื่อเดียวกับ ILogger.Name!
}

public class MultiLogger : ILogger, IConsoleOutput
{
    // Explicit implementation - ระบุ interface อย่างชัดเจน
    void ILogger.Log(string message)
    {
        Console.WriteLine($"[FILE LOG] {message}");
    }

    void IConsoleOutput.Log(string message)
    {
        Console.WriteLine($"[CONSOLE] {message}");
    }

    // Explicit property
    string ILogger.Name => "FileLogger";
    string IConsoleOutput.Name => "ConsoleLogger";

    // Regular method ที่ใช้ทั้งสอง
    public void LogAll(string message)
    {
        ((ILogger)this).Log(message);
        ((IConsoleOutput)this).Log(message);
    }
}

var logger = new MultiLogger();
logger.LogAll("Test message");

// เข้าถึง explicit implementation ผ่าน interface type
ILogger fileLogger = logger;
fileLogger.Log("File-only message");

IConsoleOutput consoleLogger = logger;
consoleLogger.Log("Console-only message");
```

---

## 5. Default Interface Methods (C# 8+)

```csharp
public interface INotification
{
    string Title { get; }
    string Message { get; }
    DateTime SentAt { get; }

    // Default implementation - ไม่จำเป็นต้อง implement ใน class
    void Send()
    {
        Console.WriteLine($"[{SentAt:HH:mm}] {Title}: {Message}");
    }

    // Default property
    string Priority => "Normal";

    // Default method ที่เรียก abstract method
    void SendWithRetry(int maxRetries = 3)
    {
        for (int i = 1; i <= maxRetries; i++)
        {
            try
            {
                Send();
                Console.WriteLine($"ส่งสำเร็จ (ครั้งที่ {i})");
                return;
            }
            catch (Exception ex)
            {
                Console.WriteLine($"ครั้งที่ {i} ล้มเหลว: {ex.Message}");
            }
        }
        Console.WriteLine("ส่งไม่สำเร็จหลังจากลองหลายครั้ง");
    }

    // Static method (C# 8+)
    static INotification CreateSimple(string title, string message)
    {
        return new SimpleNotification(title, message);
    }
}

// Class ที่ implement interface โดยไม่ต้อง implement default method
public class EmailNotification : INotification
{
    public string Title { get; }
    public string Message { get; }
    public DateTime SentAt { get; }
    public string RecipientEmail { get; }

    public EmailNotification(string title, string message, string email)
    {
        Title = title;
        Message = message;
        SentAt = DateTime.Now;
        RecipientEmail = email;
    }

    // Override default Send()
    public void Send()
    {
        Console.WriteLine($"📧 Email → {RecipientEmail}");
        Console.WriteLine($"   Subject: {Title}");
        Console.WriteLine($"   Body: {Message}");
    }

    // Priority override
    public string Priority => "High";
}

// Class ที่ใช้ default implementation
public class SimpleNotification : INotification
{
    public string Title { get; }
    public string Message { get; }
    public DateTime SentAt { get; } = DateTime.Now;

    public SimpleNotification(string title, string message)
    {
        Title = title;
        Message = message;
    }
    // ใช้ default Send() และ Priority
}

// การใช้งาน
var email = new EmailNotification("สวัสดี", "ยินดีต้อนรับ!", "user@example.com");
var simple = INotification.CreateSimple("แจ้งเตือน", "มีข้อความใหม่");

INotification[] notifications = [email, simple];
foreach (var n in notifications)
{
    n.Send();
    Console.WriteLine($"Priority: {n.Priority}\n");
}
```

---

## 6. Interface vs Abstract Class

| Feature | Interface | Abstract Class |
|---------|-----------|----------------|
| Multiple inheritance | ✅ หลาย interfaces | ❌ class เดียว |
| Constructors | ❌ ไม่มี | ✅ มีได้ |
| Instance fields | ❌ ไม่มี (static ได้) | ✅ มีได้ |
| Access modifiers | public ทั้งหมด | ปรับได้ |
| Default implementation | ✅ C# 8+ | ✅ ตลอดมา |
| ใช้เมื่อ | Define capability | Define base behavior |

```csharp
// เมื่อไหรใช้ Interface
public interface ISortable
{
    int CompareTo(object other);
}

public interface ISerializable
{
    string Serialize();
    void Deserialize(string data);
}

// เมื่อไหรใช้ Abstract Class
public abstract class DatabaseRepository<T>
{
    protected string _connectionString;

    protected DatabaseRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    // Template Method - กำหนดโครงสร้าง
    public T? FindById(int id)
    {
        OpenConnection();
        var result = ExecuteQuery(id);
        CloseConnection();
        return result;
    }

    protected abstract T? ExecuteQuery(int id);

    private void OpenConnection() => Console.WriteLine("Opening connection...");
    private void CloseConnection() => Console.WriteLine("Closing connection...");
}

// ใช้ทั้งสอง - Class extend Abstract + implement Interface
public class ProductRepository
    : DatabaseRepository<Product>,
      ISortable,
      ISerializable
{
    public ProductRepository(string connStr) : base(connStr) { }

    protected override Product? ExecuteQuery(int id)
    {
        // Database logic
        return null;
    }

    public int CompareTo(object other) => 0;
    public string Serialize() => "{}";
    public void Deserialize(string data) { }
}

// Simple record สำหรับตัวอย่าง
public record Product(int Id, string Name, decimal Price);
```

---

## 7. Common .NET Interfaces

### IComparable และ IComparable\<T\>

```csharp
// IComparable<T> - สำหรับการเรียงลำดับ
public class Temperature : IComparable<Temperature>
{
    public double Celsius { get; }

    public Temperature(double celsius) { Celsius = celsius; }

    // CompareTo: คืน < 0 ถ้าน้อยกว่า, 0 ถ้าเท่ากัน, > 0 ถ้ามากกว่า
    public int CompareTo(Temperature? other)
    {
        if (other == null) return 1;
        return Celsius.CompareTo(other.Celsius);
    }

    public override string ToString() => $"{Celsius}°C";
}

var temps = new List<Temperature>
{
    new Temperature(100), new Temperature(0), new Temperature(37), new Temperature(-10)
};

temps.Sort(); // ใช้ IComparable<Temperature>
Console.WriteLine(string.Join(", ", temps));
// -10°C, 0°C, 37°C, 100°C

// IComparer<T> - external comparator
public class TemperatureDescComparer : IComparer<Temperature>
{
    public int Compare(Temperature? x, Temperature? y)
        => y?.CompareTo(x) ?? 0; // Reverse order
}

temps.Sort(new TemperatureDescComparer());
Console.WriteLine(string.Join(", ", temps));
// 100°C, 37°C, 0°C, -10°C
```

### IEnumerable\<T\>

```csharp
// IEnumerable<T> - สำหรับ iteration
public class NumberRange : IEnumerable<int>
{
    private readonly int _start;
    private readonly int _end;
    private readonly int _step;

    public NumberRange(int start, int end, int step = 1)
    {
        _start = start;
        _end = end;
        _step = step;
    }

    public IEnumerator<int> GetEnumerator()
    {
        for (int i = _start; i <= _end; i += _step)
            yield return i;
    }

    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
        => GetEnumerator();
}

// ใช้งาน
var evens = new NumberRange(2, 20, 2);
foreach (int n in evens)
    Console.Write($"{n} ");
// 2 4 6 8 10 12 14 16 18 20

// ใช้กับ LINQ
var sumOfEvens = evens.Sum();
var bigEvens = evens.Where(n => n > 10).ToList();
Console.WriteLine($"\nผลรวม: {sumOfEvens}");
Console.WriteLine($"เลขคู่มากกว่า 10: {string.Join(", ", bigEvens)}");
```

### IDisposable

```csharp
// IDisposable - สำหรับ resource management
public class DatabaseConnection : IDisposable
{
    private bool _disposed = false;
    private readonly string _connectionString;

    public bool IsOpen { get; private set; }

    public DatabaseConnection(string connectionString)
    {
        _connectionString = connectionString;
        Open();
    }

    private void Open()
    {
        Console.WriteLine($"เปิด connection: {_connectionString}");
        IsOpen = true;
    }

    public void ExecuteQuery(string sql)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        if (!IsOpen) throw new InvalidOperationException("Connection ปิดอยู่");
        Console.WriteLine($"Execute: {sql}");
    }

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (!_disposed)
        {
            if (disposing)
            {
                Console.WriteLine("ปิด database connection");
                IsOpen = false;
            }
            _disposed = true;
        }
    }

    ~DatabaseConnection() => Dispose(false);
}

// ใช้งาน using
using (var conn = new DatabaseConnection("Server=localhost;DB=mydb"))
{
    conn.ExecuteQuery("SELECT * FROM users");
    conn.ExecuteQuery("SELECT * FROM products");
}
// Connection ปิดอัตโนมัติ

// C# 8+ using declaration
using var conn2 = new DatabaseConnection("Server=db2;DB=testdb");
conn2.ExecuteQuery("SELECT 1");
// ปิดเมื่อออกจาก scope
```

### IEquatable\<T\>

```csharp
// IEquatable<T> - สำหรับ value equality
public class Money : IEquatable<Money>, IComparable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency = "THB")
    {
        Amount = Math.Round(amount, 2);
        Currency = currency.ToUpper();
    }

    public bool Equals(Money? other)
    {
        if (other == null) return false;
        return Amount == other.Amount && Currency == other.Currency;
    }

    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);

    public int CompareTo(Money? other)
    {
        if (other == null) return 1;
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot compare different currencies");
        return Amount.CompareTo(other.Amount);
    }

    // Operators
    public static bool operator ==(Money? a, Money? b)
        => a?.Equals(b) ?? b is null;
    public static bool operator !=(Money? a, Money? b) => !(a == b);
    public static bool operator <(Money a, Money b) => a.CompareTo(b) < 0;
    public static bool operator >(Money a, Money b) => a.CompareTo(b) > 0;

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public override string ToString() => $"{Amount:N2} {Currency}";
}

var m1 = new Money(100.50m, "THB");
var m2 = new Money(100.50m, "THB");
var m3 = new Money(200m, "THB");

Console.WriteLine(m1 == m2);   // True
Console.WriteLine(m1 == m3);   // False
Console.WriteLine(m1 + m3);    // 300.50 THB
Console.WriteLine(m1 < m3);    // True

var prices = new List<Money>
{
    new Money(500m), new Money(100m), new Money(300m)
};
prices.Sort();
Console.WriteLine(string.Join(", ", prices)); // 100.00 THB, 300.00 THB, 500.00 THB
```

---

## 8. โปรแกรมตัวอย่าง: Payment System

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace PaymentSystemExample
{
    /// <summary>
    /// ผลลัพธ์การชำระเงิน
    /// </summary>
    public class PaymentResult
    {
        public bool IsSuccessful { get; init; }
        public decimal AmountPaid { get; init; }
        public decimal FeesCharged { get; init; }
        public string TransactionId { get; init; }
        public string Message { get; init; }
        public DateTime ProcessedAt { get; init; }

        public PaymentResult(bool success, decimal amount, decimal fees,
            string transactionId, string message)
        {
            IsSuccessful = success;
            AmountPaid = amount;
            FeesCharged = fees;
            TransactionId = transactionId;
            Message = message;
            ProcessedAt = DateTime.Now;
        }

        public override string ToString()
        {
            string status = IsSuccessful ? "✅ สำเร็จ" : "❌ ล้มเหลว";
            return $"{status} | TxID: {TransactionId} | {AmountPaid:N2}฿ | Fee: {FeesCharged:N2}฿ | {Message}";
        }
    }

    /// <summary>
    /// Interface หลักสำหรับ Payment
    /// </summary>
    public interface IPaymentMethod
    {
        string MethodName { get; }
        string ProviderName { get; }
        bool IsAvailable { get; }

        PaymentResult ProcessPayment(decimal amount, string description);
        bool Refund(string transactionId, decimal amount);

        // Default method
        void PrintInfo()
        {
            Console.WriteLine($"💳 {MethodName} by {ProviderName} - " +
                $"{(IsAvailable ? "พร้อมใช้งาน" : "ไม่พร้อมใช้งาน")}");
        }
    }

    /// <summary>
    /// Interface สำหรับ payment ที่มี balance
    /// </summary>
    public interface IBalancePayment : IPaymentMethod
    {
        decimal Balance { get; }
        void TopUp(decimal amount);
        bool HasSufficientFunds(decimal amount);
    }

    /// <summary>
    /// Interface สำหรับ payment ที่มี limit
    /// </summary>
    public interface ILimitedPayment : IPaymentMethod
    {
        decimal CreditLimit { get; }
        decimal UsedCredit { get; }
        decimal AvailableCredit => CreditLimit - UsedCredit;
    }

    /// <summary>
    /// Interface สำหรับ recurring payment
    /// </summary>
    public interface IRecurringPayment : IPaymentMethod
    {
        string SetupRecurring(decimal amount, string description, int intervalDays);
        void CancelRecurring(string recurringId);
    }

    /// <summary>
    /// Cash Payment
    /// </summary>
    public class CashPayment : IPaymentMethod
    {
        public string MethodName => "เงินสด";
        public string ProviderName => "Cash";
        public bool IsAvailable => true;

        private decimal _cashHeld;

        public CashPayment(decimal cashAmount)
        {
            _cashHeld = cashAmount;
        }

        public PaymentResult ProcessPayment(decimal amount, string description)
        {
            if (_cashHeld < amount)
            {
                return new PaymentResult(false, 0, 0,
                    GenerateTransactionId(),
                    $"เงินสดไม่พอ (มี {_cashHeld:N2}฿ ต้องการ {amount:N2}฿)");
            }

            _cashHeld -= amount;
            decimal change = _cashHeld > 0 ? 0 : 0;

            return new PaymentResult(true, amount, 0,
                GenerateTransactionId(),
                $"ชำระเงินสด {amount:N2}฿ สำเร็จ");
        }

        public bool Refund(string transactionId, decimal amount)
        {
            _cashHeld += amount;
            Console.WriteLine($"คืนเงินสด {amount:N2}฿");
            return true;
        }

        private static string GenerateTransactionId()
            => $"CASH-{DateTime.Now:yyyyMMddHHmmss}-{Random.Shared.Next(1000, 9999)}";
    }

    /// <summary>
    /// Credit Card Payment
    /// </summary>
    public class CreditCardPayment : IPaymentMethod, ILimitedPayment, IRecurringPayment
    {
        private readonly string _cardNumber;
        private readonly string _cardHolderName;
        private decimal _usedCredit;
        private readonly List<string> _recurringIds = new();
        private readonly Dictionary<string, (decimal amount, string desc, int interval)> _recurringSetups = new();

        public string MethodName => "บัตรเครดิต";
        public string ProviderName => "VISA/Mastercard";
        public bool IsAvailable => AvailableCredit > 0;

        public decimal CreditLimit { get; }
        public decimal UsedCredit => _usedCredit;
        public decimal AvailableCredit => CreditLimit - _usedCredit;

        public const decimal FeeRate = 0.015m; // 1.5%
        public const decimal MinFee = 10m;

        public CreditCardPayment(string cardNumber, string cardHolder,
            decimal creditLimit)
        {
            _cardNumber = cardNumber;
            _cardHolderName = cardHolder;
            CreditLimit = creditLimit;
        }

        public PaymentResult ProcessPayment(decimal amount, string description)
        {
            decimal fee = Math.Max(amount * FeeRate, MinFee);
            decimal total = amount + fee;

            if (total > AvailableCredit)
            {
                return new PaymentResult(false, 0, 0,
                    GenerateTransactionId(),
                    $"วงเงินไม่พอ (ใช้ได้ {AvailableCredit:N2}฿ ต้องการ {total:N2}฿)");
            }

            _usedCredit += total;
            return new PaymentResult(true, amount, fee,
                GenerateTransactionId(),
                $"ชำระบัตรเครดิต {amount:N2}฿ + ค่าธรรมเนียม {fee:N2}฿");
        }

        public bool Refund(string transactionId, decimal amount)
        {
            _usedCredit = Math.Max(0, _usedCredit - amount);
            Console.WriteLine($"คืนเงินบัตรเครดิต {amount:N2}฿ (TxID: {transactionId})");
            return true;
        }

        public string SetupRecurring(decimal amount, string description, int intervalDays)
        {
            string id = $"REC-{Guid.NewGuid():N}".Substring(0, 12);
            _recurringSetups[id] = (amount, description, intervalDays);
            Console.WriteLine($"ตั้งค่าจ่ายซ้ำ: {description} {amount:N2}฿ ทุก {intervalDays} วัน (ID: {id})");
            return id;
        }

        public void CancelRecurring(string recurringId)
        {
            if (_recurringSetups.Remove(recurringId))
                Console.WriteLine($"ยกเลิกการจ่ายซ้ำ: {recurringId}");
        }

        public void PrintInfo()
        {
            string masked = $"****-****-****-{_cardNumber[^4..]}";
            Console.WriteLine($"💳 {MethodName}: {masked} | {_cardHolderName}");
            Console.WriteLine($"   วงเงิน: {CreditLimit:N0}฿ | ใช้ไป: {_usedCredit:N0}฿ | คงเหลือ: {AvailableCredit:N0}฿");
        }

        private static string GenerateTransactionId()
            => $"CC-{DateTime.Now:yyyyMMddHHmmss}-{Random.Shared.Next(1000, 9999)}";
    }

    /// <summary>
    /// Digital Wallet (PromptPay / TrueMoney / etc.)
    /// </summary>
    public class DigitalWallet : IPaymentMethod, IBalancePayment
    {
        private decimal _balance;
        private readonly List<(DateTime date, decimal amount, string desc)> _transactions = new();

        public string MethodName => "กระเป๋าเงินดิจิตัล";
        public string ProviderName { get; }
        public bool IsAvailable => _balance > 0;
        public decimal Balance => _balance;

        public const decimal TransferFee = 5m; // ค่าธรรมเนียมโอน

        public DigitalWallet(string providerName, decimal initialBalance = 0)
        {
            ProviderName = providerName;
            _balance = initialBalance;
        }

        public void TopUp(decimal amount)
        {
            if (amount <= 0)
                throw new ArgumentException("จำนวนต้องมากกว่า 0");
            _balance += amount;
            _transactions.Add((DateTime.Now, amount, "เติมเงิน"));
            Console.WriteLine($"เติมเงิน {ProviderName}: +{amount:N2}฿ (คงเหลือ {_balance:N2}฿)");
        }

        public bool HasSufficientFunds(decimal amount)
            => _balance >= amount + TransferFee;

        public PaymentResult ProcessPayment(decimal amount, string description)
        {
            decimal total = amount + TransferFee;

            if (!HasSufficientFunds(amount))
            {
                return new PaymentResult(false, 0, 0,
                    GenerateTransactionId(),
                    $"ยอดเงินไม่พอ (มี {_balance:N2}฿ ต้องการ {total:N2}฿)");
            }

            _balance -= total;
            _transactions.Add((DateTime.Now, -total, description));

            return new PaymentResult(true, amount, TransferFee,
                GenerateTransactionId(),
                $"ชำระ {ProviderName} {amount:N2}฿ + ค่าโอน {TransferFee:N2}฿");
        }

        public bool Refund(string transactionId, decimal amount)
        {
            _balance += amount;
            _transactions.Add((DateTime.Now, amount, $"คืนเงิน TxID:{transactionId}"));
            Console.WriteLine($"คืนเงิน {ProviderName}: +{amount:N2}฿ (คงเหลือ {_balance:N2}฿)");
            return true;
        }

        public void PrintStatement()
        {
            Console.WriteLine($"\n=== Statement: {ProviderName} ===");
            foreach (var (date, amount, desc) in _transactions.TakeLast(10))
            {
                string sign = amount > 0 ? "+" : "";
                Console.WriteLine($"  {date:dd/MM HH:mm} | {sign}{amount:N2}฿ | {desc}");
            }
            Console.WriteLine($"  คงเหลือ: {_balance:N2}฿");
        }

        private static string GenerateTransactionId()
            => $"EW-{DateTime.Now:yyyyMMddHHmmss}-{Random.Shared.Next(1000, 9999)}";
    }

    /// <summary>
    /// Payment Gateway - จัดการ payment methods ทั้งหมด
    /// </summary>
    public class PaymentGateway
    {
        private readonly List<IPaymentMethod> _methods = new();
        private readonly List<PaymentResult> _history = new();

        public void RegisterMethod(IPaymentMethod method)
        {
            _methods.Add(method);
            Console.WriteLine($"ลงทะเบียน: {method.MethodName} ({method.ProviderName})");
        }

        public PaymentResult Pay(string methodName, decimal amount, string description)
        {
            var method = _methods
                .FirstOrDefault(m => m.MethodName == methodName && m.IsAvailable);

            if (method == null)
            {
                return new PaymentResult(false, 0, 0,
                    "N/A", $"ไม่พบวิธีการชำระเงิน: {methodName}");
            }

            Console.WriteLine($"\nกำลังชำระ {amount:N2}฿ ผ่าน {method.MethodName}...");
            var result = method.ProcessPayment(amount, description);
            _history.Add(result);

            Console.WriteLine(result);
            return result;
        }

        public void PrintPaymentMethods()
        {
            Console.WriteLine("\n=== วิธีการชำระเงินที่รองรับ ===");
            foreach (var method in _methods)
            {
                method.PrintInfo();
            }
        }

        public void PrintHistory()
        {
            Console.WriteLine("\n=== ประวัติการชำระเงิน ===");
            decimal totalPaid = _history.Where(h => h.IsSuccessful).Sum(h => h.AmountPaid);
            decimal totalFees = _history.Where(h => h.IsSuccessful).Sum(h => h.FeesCharged);

            foreach (var result in _history)
                Console.WriteLine($"  {result}");

            Console.WriteLine($"\nรวมชำระ: {totalPaid:N2}฿ | รวมค่าธรรมเนียม: {totalFees:N2}฿");
        }

        // ค้นหา payment method ที่ถูกที่สุดสำหรับจำนวนเงินที่กำหนด
        public IPaymentMethod? FindCheapestMethod(decimal amount)
        {
            return _methods
                .Where(m => m.IsAvailable)
                .MinBy(m => GetEstimatedFee(m, amount));
        }

        private decimal GetEstimatedFee(IPaymentMethod method, decimal amount)
        {
            return method switch
            {
                CreditCardPayment cc => Math.Max(amount * CreditCardPayment.FeeRate,
                    CreditCardPayment.MinFee),
                DigitalWallet => DigitalWallet.TransferFee,
                CashPayment => 0,
                _ => decimal.MaxValue
            };
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("=== ระบบชำระเงิน ===\n");

            var gateway = new PaymentGateway();

            // ลงทะเบียนวิธีการชำระเงิน
            var cash = new CashPayment(2000m);
            var creditCard = new CreditCardPayment("1234567890123456",
                "สมชาย ใจดี", 50000m);
            var trueMoney = new DigitalWallet("TrueMoney", 1500m);
            var promptPay = new DigitalWallet("PromptPay");

            gateway.RegisterMethod(cash);
            gateway.RegisterMethod(creditCard);
            gateway.RegisterMethod(trueMoney);
            gateway.RegisterMethod(promptPay);

            gateway.PrintPaymentMethods();

            // ชำระเงิน
            Console.WriteLine("\n=== ทดสอบการชำระเงิน ===");

            gateway.Pay("เงินสด", 500, "ซื้ออาหาร");
            gateway.Pay("บัตรเครดิต", 15000, "ซื้อ Laptop");
            gateway.Pay("กระเป๋าเงินดิจิตัล", 300, "ค่า Netflix");
            gateway.Pay("กระเป๋าเงินดิจิตัล", 5000, "ยอดเกิน"); // ล้มเหลว

            // เติมเงิน
            Console.WriteLine("\n=== เติมเงิน ===");
            promptPay.TopUp(1000m);
            gateway.Pay("กระเป๋าเงินดิจิตัล", 300, "ค่า Spotify");

            // ประวัติ
            gateway.PrintHistory();

            // ค้นหาวิธีถูกสุด
            var cheapest = gateway.FindCheapestMethod(500m);
            Console.WriteLine($"\nวิธีชำระราคาถูกสุดสำหรับ 500฿: {cheapest?.MethodName}");

            // Interface polymorphism
            Console.WriteLine("\n=== Interface Polymorphism ===");
            var balanceMethods = new List<IBalancePayment> { trueMoney, promptPay };
            foreach (var method in balanceMethods)
            {
                Console.WriteLine($"{method.ProviderName}: {method.Balance:N2}฿");
            }
        }
    }
}
```

---

## Exercises

### Exercise 1: Pluggable Logger
```csharp
// TODO: สร้าง ILogger interface และ implementations:
// interface ILogger: Log(string message, LogLevel level)
// - ConsoleLogger: สีตาม level (Info=white, Warn=yellow, Error=red)
// - FileLogger: เขียนไฟล์พร้อม timestamp
// - DatabaseLogger: เก็บใน List<LogEntry>
// - CompositeLogger: เรียกหลาย loggers พร้อมกัน

public enum LogLevel { Debug, Info, Warning, Error, Critical }

public interface ILogger
{
    // TODO: Implement
}
```

### Exercise 2: Export Formats
```csharp
// TODO: สร้าง IExporter interface:
// string Export(IEnumerable<object> data, string[] columns)
// Implementations:
// - CsvExporter: CSV format
// - JsonExporter: JSON array
// - HtmlExporter: HTML table
// - ExcelLikeExporter: tab-separated

public interface IExporter
{
    // TODO: Implement
}
```

### Exercise 3: Sorting Algorithms
```csharp
// TODO: สร้าง ISortAlgorithm<T> interface:
// T[] Sort(T[] data) - คืน sorted array
// string AlgorithmName { get; }
// int ComparisonCount { get; } - นับจำนวนครั้งที่เปรียบเทียบ
// Implementations: BubbleSort, SelectionSort, InsertionSort
// SortBenchmark class ที่ทดสอบทุก algorithm

public interface ISortAlgorithm<T> where T : IComparable<T>
{
    // TODO: Implement
}
```

---

## สรุป

✅ Interface กำหนด "สัญญา" ว่าต้องมี members อะไร  
✅ Class implement ได้หลาย interfaces (แก้ปัญหา multiple inheritance)  
✅ Default interface methods (C# 8+) ให้ default implementation ได้  
✅ Explicit implementation ใช้เมื่อมีชื่อ method ซ้ำกัน  
✅ IComparable ใช้สำหรับ sorting, IEnumerable ใช้สำหรับ iteration  
✅ IDisposable ใช้สำหรับ resource management ด้วย `using`  
✅ Interface ดีกว่า abstract class เมื่อต้องการ multiple behaviors  

## Part ถัดไป
**Part 018: Abstract Classes** - เรียนรู้ abstract keyword, Template Method pattern และการเปรียบเทียบกับ interface

---
*Part 017/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
