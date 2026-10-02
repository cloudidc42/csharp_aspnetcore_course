# Part 012: Classes และ Objects เบื้องต้น

## เนื้อหาใน Part นี้
- Class declaration และโครงสร้าง
- Fields vs Properties
- Instance methods
- Static members
- Object creation ด้วย new keyword
- Value types vs Reference types
- โปรแกรมตัวอย่าง: BankAccount class

---

## 1. Class คืออะไร?

Class คือ **blueprint** (แบบแปลน) สำหรับสร้าง object โดย class กำหนด:
- **Fields/Properties** - ข้อมูลที่ object เก็บ (state)
- **Methods** - สิ่งที่ object ทำได้ (behavior)
- **Constructors** - วิธีการสร้าง object

Object คือ **instance** (ตัวอย่าง) ที่สร้างจาก class

```csharp
// Class declaration ง่ายๆ
public class Person
{
    // Fields (ข้อมูลภายใน)
    private string _name;
    private int _age;

    // Properties (ช่องทางเข้าถึงข้อมูล)
    public string Name
    {
        get { return _name; }
        set { _name = value; }
    }

    public int Age
    {
        get { return _age; }
        set { _age = value >= 0 ? value : 0; }
    }

    // Method
    public void Greet()
    {
        Console.WriteLine($"สวัสดี ฉันชื่อ {_name} อายุ {_age} ปี");
    }
}

// สร้าง object จาก class
Person person1 = new Person();
person1.Name = "สมชาย";
person1.Age = 30;
person1.Greet();

// สร้าง object แบบ var
var person2 = new Person();
person2.Name = "สมหญิง";
person2.Age = 25;
```

---

## 2. Class Declaration รูปแบบต่างๆ

```csharp
// Access modifiers
public class PublicClass { }    // เข้าถึงได้จากทุกที่
internal class InternalClass { } // เข้าถึงได้ภายใน assembly เดียวกัน
// private class ใช้ได้เฉพาะ nested class

// Partial class - แบ่งไฟล์ได้หลายไฟล์
public partial class BigClass
{
    public void Method1() { }
}

public partial class BigClass
{
    public void Method2() { }
}

// Sealed class - ไม่สามารถ inherit ได้
public sealed class SingletonClass { }

// Static class - ทุกสมาชิกต้องเป็น static
public static class MathHelper
{
    public static int Add(int a, int b) => a + b;
    public static int Multiply(int a, int b) => a * b;
}

// Record class (C# 9+) - สำหรับ immutable data
public record Point(int X, int Y);
public record Person2(string Name, int Age);
```

---

## 3. Fields

Fields คือตัวแปรที่ประกาศภายใน class

```csharp
public class Car
{
    // Instance fields
    private string _make;           // private - เข้าถึงได้เฉพาะภายใน class
    private string _model;
    private int _year;
    private double _mileage;

    // Public field (ไม่แนะนำ - ควรใช้ Property แทน)
    public string Color;

    // Readonly field - กำหนดค่าได้แค่ใน constructor
    private readonly string _vin;  // Vehicle Identification Number

    // Constant field - ค่าคงที่ที่รู้ตอน compile
    public const int MaxSpeed = 200;

    // Static field - ใช้ร่วมกันทุก instance
    private static int _totalCarsCreated = 0;

    // Field initializer
    private List<string> _maintenanceHistory = new List<string>();

    public Car(string make, string model, int year, string vin)
    {
        _make = make;
        _model = model;
        _year = year;
        _vin = vin;        // กำหนดค่า readonly ได้ใน constructor
        _totalCarsCreated++;
    }

    // Property เพื่อเข้าถึง static field
    public static int TotalCarsCreated => _totalCarsCreated;

    public override string ToString()
    {
        return $"{_year} {_make} {_model} (VIN: {_vin})";
    }
}

// การใช้งาน
var car1 = new Car("Toyota", "Camry", 2024, "1ABC123456");
var car2 = new Car("Honda", "Civic", 2023, "2DEF789012");

Console.WriteLine(car1);
Console.WriteLine($"รถทั้งหมด: {Car.TotalCarsCreated} คัน");

// Constants
Console.WriteLine($"ความเร็วสูงสุด: {Car.MaxSpeed} km/h");
```

---

## 4. Properties

Properties คือ "smart fields" ที่มี getter และ setter

```csharp
public class Temperature
{
    // Auto-implemented property
    public string Unit { get; set; } = "Celsius";

    // Property with backing field และ validation
    private double _celsius;
    public double Celsius
    {
        get { return _celsius; }
        set
        {
            if (value < -273.15)
                throw new ArgumentException("อุณหภูมิต่ำกว่า Absolute Zero ไม่ได้");
            _celsius = value;
        }
    }

    // Computed/Derived property - ไม่มี backing field
    public double Fahrenheit
    {
        get { return (_celsius * 9.0 / 5.0) + 32; }
        set { _celsius = (value - 32) * 5.0 / 9.0; }
    }

    // Readonly property (getter เท่านั้น)
    public double Kelvin => _celsius + 273.15;

    // Expression-bodied property
    public string Description => $"{_celsius}°C / {Fahrenheit}°F / {Kelvin}K";
}

// การใช้งาน
var temp = new Temperature();
temp.Celsius = 100;
Console.WriteLine($"เดือดที่: {temp.Description}");

temp.Fahrenheit = 32;
Console.WriteLine($"แข็งตัวที่: {temp.Description}");
```

---

## 5. Instance Methods

```csharp
public class Calculator
{
    private List<double> _history = new List<double>();
    private double _currentValue;

    public double CurrentValue => _currentValue;

    // Instance methods
    public double Add(double value)
    {
        _history.Add(_currentValue);
        _currentValue += value;
        return _currentValue;
    }

    public double Subtract(double value)
    {
        _history.Add(_currentValue);
        _currentValue -= value;
        return _currentValue;
    }

    public double Multiply(double value)
    {
        _history.Add(_currentValue);
        _currentValue *= value;
        return _currentValue;
    }

    public double Divide(double value)
    {
        if (value == 0)
            throw new DivideByZeroException("ไม่สามารถหารด้วย 0 ได้");

        _history.Add(_currentValue);
        _currentValue /= value;
        return _currentValue;
    }

    public void Reset()
    {
        _history.Add(_currentValue);
        _currentValue = 0;
    }

    public void Undo()
    {
        if (_history.Count > 0)
        {
            _currentValue = _history[^1]; // last element
            _history.RemoveAt(_history.Count - 1);
        }
    }

    public void ShowHistory()
    {
        Console.WriteLine("ประวัติการคำนวณ:");
        foreach (double val in _history)
        {
            Console.WriteLine($"  {val}");
        }
        Console.WriteLine($"  ปัจจุบัน: {_currentValue}");
    }

    // Method Chaining - return this เพื่อ chain ได้
    public Calculator SetValue(double value)
    {
        _currentValue = value;
        return this;
    }

    public Calculator AddChain(double value)
    {
        _currentValue += value;
        return this;
    }

    public Calculator MultiplyChain(double value)
    {
        _currentValue *= value;
        return this;
    }
}

// การใช้งาน
var calc = new Calculator();
calc.Add(10);
calc.Multiply(5);
calc.Subtract(20);
calc.ShowHistory();

// Method chaining
var result = new Calculator()
    .SetValue(10)
    .AddChain(5)
    .MultiplyChain(2)
    .CurrentValue;
Console.WriteLine($"ผลลัพธ์: {result}"); // 30
```

---

## 6. Static Members

Static members เป็นของ class ไม่ใช่ของ instance

```csharp
public class MathUtils
{
    // Static fields
    private static int _calculationCount = 0;
    public static readonly double PI = Math.PI;
    public const double E = Math.E;

    // Static property
    public static int CalculationCount => _calculationCount;

    // Static methods
    public static double CircleArea(double radius)
    {
        _calculationCount++;
        return PI * radius * radius;
    }

    public static double CirclePerimeter(double radius)
    {
        _calculationCount++;
        return 2 * PI * radius;
    }

    public static double Power(double base_, double exponent)
    {
        _calculationCount++;
        return Math.Pow(base_, exponent);
    }

    public static (double min, double max) MinMax(params double[] values)
    {
        if (values.Length == 0)
            throw new ArgumentException("ต้องมีค่าอย่างน้อย 1 ค่า");
        return (values.Min(), values.Max());
    }

    // Static constructor - เรียกครั้งเดียวก่อน first use
    static MathUtils()
    {
        Console.WriteLine("MathUtils initialized");
    }
}

// Static class ตัวอย่าง - Extension Methods
public static class StringExtensions
{
    public static string Repeat(this string text, int times)
    {
        var sb = new System.Text.StringBuilder();
        for (int i = 0; i < times; i++)
            sb.Append(text);
        return sb.ToString();
    }

    public static bool IsNumeric(this string text)
    {
        return double.TryParse(text, out _);
    }

    public static string TruncateAt(this string text, int maxLength, string suffix = "...")
    {
        if (text.Length <= maxLength) return text;
        return text[..(maxLength - suffix.Length)] + suffix;
    }
}

// การใช้งาน Static members
Console.WriteLine($"พื้นที่วงกลม r=5: {MathUtils.CircleArea(5):F2}");
Console.WriteLine($"เส้นรอบวง r=5: {MathUtils.CirclePerimeter(5):F2}");
Console.WriteLine($"2^10 = {MathUtils.Power(2, 10)}");

var (min, max) = MathUtils.MinMax(3, 1, 4, 1, 5, 9, 2, 6);
Console.WriteLine($"Min: {min}, Max: {max}");
Console.WriteLine($"คำนวณทั้งหมด: {MathUtils.CalculationCount} ครั้ง");

// Extension methods
string text = "Hello World";
Console.WriteLine(text.Repeat(3));         // HelloWorldHelloWorldHelloWorld
Console.WriteLine("123".IsNumeric());      // True
Console.WriteLine("abc".IsNumeric());      // False
Console.WriteLine("This is a very long text".TruncateAt(15)); // "This is a ve..."
```

---

## 7. Object Creation - new keyword

```csharp
public class Point
{
    public double X { get; set; }
    public double Y { get; set; }

    // Constructors
    public Point() { }
    public Point(double x, double y) { X = x; Y = y; }

    public double DistanceTo(Point other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }

    public override string ToString() => $"({X}, {Y})";
}

// วิธีต่างๆ ในการสร้าง object
// 1. Default constructor
var p1 = new Point();
p1.X = 3;
p1.Y = 4;

// 2. Parameterized constructor
var p2 = new Point(1, 1);

// 3. Object initializer
var p3 = new Point { X = 5, Y = 12 };

// 4. Target-typed new (C# 9+)
Point p4 = new(6, 8);

// 5. Collection initializer
var points = new List<Point>
{
    new Point(0, 0),
    new Point(1, 1),
    new Point { X = 2, Y = 2 },
    new(3, 3)  // Target-typed new
};

foreach (var point in points)
{
    Console.WriteLine($"{point}: ห่างจากจุดกำเนิด {point.DistanceTo(new Point(0, 0)):F2}");
}

// Array of objects
Point[] triangle =
[
    new(0, 0),
    new(3, 0),
    new(0, 4)
];

double perimeter = 0;
for (int i = 0; i < triangle.Length; i++)
{
    int next = (i + 1) % triangle.Length;
    perimeter += triangle[i].DistanceTo(triangle[next]);
}
Console.WriteLine($"เส้นรอบรูปสามเหลี่ยม: {perimeter:F2}");
```

---

## 8. Value Types vs Reference Types

ความแตกต่างสำคัญที่ต้องเข้าใจ

```csharp
// VALUE TYPES: int, double, bool, char, struct, enum
// เก็บค่าโดยตรงใน stack
// การ copy จะ copy ค่า

int a = 10;
int b = a;  // copy ค่า
b = 20;
Console.WriteLine($"a = {a}, b = {b}"); // a = 10, b = 20 (ไม่เปลี่ยน a)

// REFERENCE TYPES: class, string, array, delegate
// เก็บ reference (pointer) ไปยัง heap
// การ copy จะ copy reference (ชี้ไปที่เดียวกัน)

int[] arr1 = { 1, 2, 3 };
int[] arr2 = arr1;  // copy reference
arr2[0] = 999;
Console.WriteLine($"arr1[0] = {arr1[0]}"); // 999 (เปลี่ยนด้วย!)

// เพื่อ copy array จริงๆ ต้องใช้:
int[] arr3 = arr1.ToArray();  // หรือ arr1[..]
arr3[0] = 111;
Console.WriteLine($"arr1[0] = {arr1[0]}"); // ยังคงเป็น 999

// สาธิตด้วย class
public class Box
{
    public int Width { get; set; }
    public int Height { get; set; }
}

var box1 = new Box { Width = 10, Height = 20 };
var box2 = box1;  // copy reference - ชี้ไปที่เดียวกัน
box2.Width = 99;

Console.WriteLine($"box1.Width = {box1.Width}"); // 99 (เปลี่ยนด้วย!)

// null สำหรับ reference types
Box? nullBox = null;  // Nullable reference type
if (nullBox == null)
{
    Console.WriteLine("nullBox เป็น null");
}

// null check ต่างๆ
var box3 = nullBox ?? new Box { Width = 1, Height = 1 }; // null-coalescing
nullBox?.ToString(); // null-conditional - ไม่ throw ถ้า null

// Passing to methods
void ModifyValue(int x)
{
    x = 999; // ไม่เปลี่ยน original
}

void ModifyReference(Box box)
{
    box.Width = 999; // เปลี่ยน original!
}

void ModifyReferenceVariable(ref Box box)
{
    box = new Box { Width = 111 }; // เปลี่ยน variable ด้วย!
}

int num = 5;
ModifyValue(num);
Console.WriteLine(num); // 5 - ไม่เปลี่ยน

var myBox = new Box { Width = 10 };
ModifyReference(myBox);
Console.WriteLine(myBox.Width); // 999 - เปลี่ยน!
```

---

## 9. this Keyword

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }

    // this ใช้อ้างถึง current instance
    public Person(string name, int age)
    {
        this.Name = name;  // this.Name = ชัดเจนว่าเป็น property ของ class
        this.Age = age;
    }

    // this() - เรียก constructor อื่น
    public Person(string name) : this(name, 0) { }

    // Return this เพื่อ method chaining
    public Person SetName(string name)
    {
        this.Name = name;
        return this;
    }

    public Person SetAge(int age)
    {
        this.Age = age;
        return this;
    }

    public void PrintInfo()
    {
        Console.WriteLine($"Name: {Name}, Age: {Age}");
    }
}

// Method chaining
new Person("สมชาย")
    .SetAge(30)
    .PrintInfo();
```

---

## 10. Object Equality และ ToString

```csharp
public class Student
{
    public int Id { get; init; }
    public string Name { get; set; }
    public double GPA { get; set; }

    public Student(int id, string name, double gpa)
    {
        Id = id;
        Name = name;
        GPA = gpa;
    }

    // Override ToString
    public override string ToString()
    {
        return $"Student[{Id}]: {Name} (GPA: {GPA:F2})";
    }

    // Override Equals
    public override bool Equals(object? obj)
    {
        if (obj is Student other)
            return Id == other.Id; // เท่ากันถ้า Id เดียวกัน
        return false;
    }

    // Override GetHashCode - ต้อง override คู่กับ Equals
    public override int GetHashCode()
    {
        return Id.GetHashCode();
    }
}

// การใช้งาน
var s1 = new Student(1, "สมชาย", 3.5);
var s2 = new Student(1, "สมชาย", 3.5); // คนละ object แต่ Id เดียวกัน
var s3 = new Student(2, "สมหญิง", 3.8);

Console.WriteLine(s1.ToString());
Console.WriteLine(s1);              // เรียก ToString อัตโนมัติ

Console.WriteLine(s1 == s2);       // False (reference equality)
Console.WriteLine(s1.Equals(s2));  // True (Id เดียวกัน)

// ใช้ใน Dictionary/HashSet
var studentSet = new HashSet<Student> { s1, s2, s3 };
Console.WriteLine($"นักเรียนที่ unique: {studentSet.Count}"); // 2

// Record สำหรับ value equality อัตโนมัติ
public record StudentRecord(int Id, string Name, double GPA);

var r1 = new StudentRecord(1, "สมชาย", 3.5);
var r2 = new StudentRecord(1, "สมชาย", 3.5);
Console.WriteLine(r1 == r2); // True! (value equality)
Console.WriteLine(r1.Equals(r2)); // True!
```

---

## 11. โปรแกรมตัวอย่าง: BankAccount Class

```csharp
using System;
using System.Collections.Generic;

namespace BankAccountExample
{
    /// <summary>
    /// ประเภทธุรกรรม
    /// </summary>
    public enum TransactionType
    {
        Deposit,    // ฝาก
        Withdrawal, // ถอน
        Transfer,   // โอน
        Fee,        // ค่าธรรมเนียม
        Interest    // ดอกเบี้ย
    }

    /// <summary>
    /// บันทึกธุรกรรม
    /// </summary>
    public class Transaction
    {
        public int Id { get; init; }
        public TransactionType Type { get; init; }
        public decimal Amount { get; init; }
        public decimal BalanceAfter { get; init; }
        public DateTime Timestamp { get; init; }
        public string Description { get; init; }

        public Transaction(int id, TransactionType type, decimal amount,
            decimal balanceAfter, string description)
        {
            Id = id;
            Type = type;
            Amount = amount;
            BalanceAfter = balanceAfter;
            Timestamp = DateTime.Now;
            Description = description;
        }

        public override string ToString()
        {
            string sign = Type == TransactionType.Deposit ||
                          Type == TransactionType.Interest ? "+" : "-";
            return $"{Timestamp:dd/MM/yyyy HH:mm} | {Type,-12} | " +
                   $"{sign}{Amount,10:N2} | คงเหลือ: {BalanceAfter,12:N2} | {Description}";
        }
    }

    /// <summary>
    /// บัญชีธนาคาร
    /// </summary>
    public class BankAccount
    {
        // Private fields
        private decimal _balance;
        private readonly List<Transaction> _transactions;
        private static int _nextTransactionId = 1;
        private static int _totalAccounts = 0;

        // Constants
        public const decimal MinimumBalance = 500m;
        public const decimal OverdraftFee = 200m;
        public const decimal TransferFee = 25m;
        public const double AnnualInterestRate = 0.015; // 1.5%

        // Properties
        public string AccountNumber { get; init; }
        public string OwnerName { get; set; }
        public decimal Balance => _balance;
        public bool IsActive { get; private set; }
        public DateTime OpenedDate { get; init; }
        public static int TotalAccounts => _totalAccounts;

        // Constructor
        public BankAccount(string accountNumber, string ownerName, decimal initialDeposit = 0)
        {
            if (string.IsNullOrWhiteSpace(accountNumber))
                throw new ArgumentException("หมายเลขบัญชีต้องไม่ว่าง");
            if (string.IsNullOrWhiteSpace(ownerName))
                throw new ArgumentException("ชื่อเจ้าของบัญชีต้องไม่ว่าง");
            if (initialDeposit < 0)
                throw new ArgumentException("เงินฝากเริ่มต้นต้องไม่ติดลบ");

            AccountNumber = accountNumber;
            OwnerName = ownerName;
            _balance = 0;
            _transactions = new List<Transaction>();
            IsActive = true;
            OpenedDate = DateTime.Now;
            _totalAccounts++;

            if (initialDeposit > 0)
                Deposit(initialDeposit, "เงินฝากเปิดบัญชี");
        }

        // Methods
        /// <summary>ฝากเงิน</summary>
        public decimal Deposit(decimal amount, string description = "ฝากเงิน")
        {
            ValidateAccount();
            if (amount <= 0)
                throw new ArgumentException("จำนวนเงินที่ฝากต้องมากกว่า 0");

            _balance += amount;
            RecordTransaction(TransactionType.Deposit, amount, description);

            Console.WriteLine($"ฝากเงิน {amount:N2} บาท สำเร็จ คงเหลือ: {_balance:N2} บาท");
            return _balance;
        }

        /// <summary>ถอนเงิน</summary>
        public decimal Withdraw(decimal amount, string description = "ถอนเงิน")
        {
            ValidateAccount();
            if (amount <= 0)
                throw new ArgumentException("จำนวนเงินที่ถอนต้องมากกว่า 0");
            if (_balance - amount < MinimumBalance)
                throw new InvalidOperationException(
                    $"ยอดคงเหลือต้องไม่ต่ำกว่า {MinimumBalance:N2} บาท");

            _balance -= amount;
            RecordTransaction(TransactionType.Withdrawal, amount, description);

            Console.WriteLine($"ถอนเงิน {amount:N2} บาท สำเร็จ คงเหลือ: {_balance:N2} บาท");
            return _balance;
        }

        /// <summary>โอนเงินไปบัญชีอื่น</summary>
        public void Transfer(BankAccount targetAccount, decimal amount,
            string description = "โอนเงิน")
        {
            ValidateAccount();
            targetAccount.ValidateAccount();

            if (amount <= 0)
                throw new ArgumentException("จำนวนเงินที่โอนต้องมากกว่า 0");

            decimal totalAmount = amount + TransferFee;
            if (_balance - totalAmount < MinimumBalance)
                throw new InvalidOperationException(
                    $"ยอดเงินไม่เพียงพอ (ต้องการ {totalAmount:N2} + คงเหลือขั้นต่ำ {MinimumBalance:N2})");

            // ถอนจากบัญชีต้นทาง
            _balance -= amount + TransferFee;
            RecordTransaction(TransactionType.Transfer, amount,
                $"โอนไป {targetAccount.AccountNumber} - {description}");
            RecordTransaction(TransactionType.Fee, TransferFee,
                "ค่าธรรมเนียมการโอน");

            // ฝากเข้าบัญชีปลายทาง
            targetAccount._balance += amount;
            targetAccount.RecordTransaction(TransactionType.Transfer, amount,
                $"รับโอนจาก {AccountNumber} - {description}");

            Console.WriteLine($"โอนเงิน {amount:N2} บาท ไปยัง {targetAccount.AccountNumber} สำเร็จ");
            Console.WriteLine($"  ค่าธรรมเนียม: {TransferFee:N2} บาท");
            Console.WriteLine($"  ยอดคงเหลือของคุณ: {_balance:N2} บาท");
        }

        /// <summary>คำนวณและเพิ่มดอกเบี้ย (รายปี)</summary>
        public decimal AddAnnualInterest()
        {
            ValidateAccount();
            decimal interest = _balance * (decimal)AnnualInterestRate;
            _balance += interest;
            RecordTransaction(TransactionType.Interest, interest,
                $"ดอกเบี้ยประจำปี ({AnnualInterestRate:P1})");

            Console.WriteLine($"เพิ่มดอกเบี้ย {interest:N2} บาท คงเหลือ: {_balance:N2} บาท");
            return interest;
        }

        /// <summary>ปิดบัญชี</summary>
        public decimal CloseAccount()
        {
            ValidateAccount();
            IsActive = false;
            decimal finalBalance = _balance;
            _balance = 0;
            Console.WriteLine($"ปิดบัญชี {AccountNumber} - คืนเงิน {finalBalance:N2} บาท");
            return finalBalance;
        }

        /// <summary>แสดงประวัติธุรกรรม</summary>
        public void PrintStatement(int lastNTransactions = 0)
        {
            Console.WriteLine($"\n=== Statement: {AccountNumber} - {OwnerName} ===");
            Console.WriteLine($"วันที่เปิดบัญชี: {OpenedDate:dd/MM/yyyy}");
            Console.WriteLine($"สถานะ: {(IsActive ? "Active" : "Closed")}");
            Console.WriteLine($"ยอดคงเหลือ: {_balance:N2} บาท");
            Console.WriteLine(new string('-', 80));

            var transToShow = lastNTransactions > 0
                ? _transactions.TakeLast(lastNTransactions)
                : _transactions;

            foreach (var transaction in transToShow)
            {
                Console.WriteLine(transaction);
            }
            Console.WriteLine(new string('=', 80));
        }

        /// <summary>สรุปบัญชี</summary>
        public AccountSummary GetSummary()
        {
            decimal totalDeposits = _transactions
                .Where(t => t.Type == TransactionType.Deposit || t.Type == TransactionType.Interest)
                .Sum(t => t.Amount);

            decimal totalWithdrawals = _transactions
                .Where(t => t.Type == TransactionType.Withdrawal ||
                            t.Type == TransactionType.Transfer ||
                            t.Type == TransactionType.Fee)
                .Sum(t => t.Amount);

            return new AccountSummary
            {
                AccountNumber = AccountNumber,
                OwnerName = OwnerName,
                Balance = _balance,
                TotalDeposits = totalDeposits,
                TotalWithdrawals = totalWithdrawals,
                TransactionCount = _transactions.Count,
                IsActive = IsActive
            };
        }

        // Private helper methods
        private void ValidateAccount()
        {
            if (!IsActive)
                throw new InvalidOperationException($"บัญชี {AccountNumber} ปิดแล้ว");
        }

        private void RecordTransaction(TransactionType type, decimal amount, string description)
        {
            var transaction = new Transaction(
                _nextTransactionId++,
                type,
                amount,
                _balance,
                description);
            _transactions.Add(transaction);
        }

        public override string ToString()
        {
            return $"Account[{AccountNumber}] {OwnerName}: {_balance:N2} บาท";
        }
    }

    public class AccountSummary
    {
        public string AccountNumber { get; set; } = "";
        public string OwnerName { get; set; } = "";
        public decimal Balance { get; set; }
        public decimal TotalDeposits { get; set; }
        public decimal TotalWithdrawals { get; set; }
        public int TransactionCount { get; set; }
        public bool IsActive { get; set; }

        public override string ToString()
        {
            return $"""
                === บัญชีสรุป ===
                หมายเลขบัญชี : {AccountNumber}
                เจ้าของบัญชี  : {OwnerName}
                ยอดคงเหลือ   : {Balance:N2} บาท
                รวมฝาก       : {TotalDeposits:N2} บาท
                รวมถอน       : {TotalWithdrawals:N2} บาท
                ธุรกรรมทั้งหมด: {TransactionCount} รายการ
                สถานะ        : {(IsActive ? "เปิดอยู่" : "ปิดแล้ว")}
                """;
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("=== ระบบธนาคาร ===\n");

            // สร้างบัญชี
            var account1 = new BankAccount("1234567890", "สมชาย ใจดี", 10000);
            var account2 = new BankAccount("0987654321", "สมหญิง ใจงาม", 5000);

            Console.WriteLine($"\nบัญชีทั้งหมด: {BankAccount.TotalAccounts} บัญชี\n");

            // ธุรกรรมต่างๆ
            account1.Deposit(5000, "เงินเดือน");
            account1.Withdraw(2000, "ค่าใช้จ่าย");
            account1.Transfer(account2, 3000, "ค่าอาหาร");
            account1.AddAnnualInterest();

            // แสดง statement
            account1.PrintStatement(5);
            account2.PrintStatement();

            // สรุปบัญชี
            Console.WriteLine("\n" + account1.GetSummary());

            // ทดสอบ error handling
            Console.WriteLine("\n=== ทดสอบ Error Handling ===");
            try
            {
                account1.Withdraw(100000); // เกินยอด
            }
            catch (InvalidOperationException ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }

            try
            {
                account1.Deposit(-100); // ติดลบ
            }
            catch (ArgumentException ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }
        }
    }
}
```

---

## Exercises

### Exercise 1: Library Book
```csharp
// TODO: สร้าง class Book ที่มี:
// - Fields: _isCheckedOut (private), _checkOutDate (private nullable)
// - Properties: ISBN, Title, Author, PublicationYear (readonly), PageCount
// - Methods: CheckOut(string borrowerName), Return(), GetAvailability()
// - Static: TotalBooksInLibrary counter
// - ToString() แสดงข้อมูลหนังสือ

public class Book
{
    // TODO: Implement
}
```

### Exercise 2: Shopping Cart
```csharp
// TODO: สร้าง class ShoppingCart ที่มี:
// - CartItem class (ProductName, Price, Quantity)
// - Methods: AddItem, RemoveItem, UpdateQuantity, GetTotal, GetItemCount
// - Property: Items (read-only collection)
// - ApplyDiscount(decimal percentage) - ลดราคา
// - ToString() แสดงรายการทั้งหมด

public class ShoppingCart
{
    // TODO: Implement
}
```

### Exercise 3: Student Registry
```csharp
// TODO: สร้าง static class StudentRegistry ที่:
// - เก็บ list ของ students
// - Methods: Register(Student), Remove(int id), FindById(int id)
// - FindByName(string name) - ค้นหาบางส่วน
// - GetTopStudents(int n) - นักเรียน GPA สูงสุด n คน
// - GetStatistics() - min/max/avg GPA

public static class StudentRegistry
{
    // TODO: Implement
}
```

---

## สรุป

✅ Class คือ blueprint สำหรับสร้าง object  
✅ Fields เก็บข้อมูลภายใน, Properties เป็น public interface สำหรับเข้าถึงข้อมูล  
✅ Instance methods ทำงานกับข้อมูลของ object นั้นๆ  
✅ Static members เป็นของ class ไม่ใช่ instance ใช้ร่วมกันทุก object  
✅ new keyword สร้าง object ใหม่บน heap  
✅ Reference types copy reference, Value types copy value  
✅ Override ToString(), Equals(), GetHashCode() เพื่อ custom behavior  

## Part ถัดไป
**Part 013: Properties และ Fields** - เรียนรู้ auto-implemented properties, getters/setters, read-only, init-only, computed properties และ validation

---
*Part 012/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
