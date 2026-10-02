# Part 010: Methods และ Functions

## เนื้อหาใน Part นี้
- การสร้าง Method
- Parameters และ Return Values
- Method Overloading
- Optional Parameters และ Named Arguments
- ref, out, in parameters
- Local Functions
- Expression-bodied Members

---

## 1. Method พื้นฐาน

```csharp
// รูปแบบ: [access] [static] [return type] MethodName([parameters])

// ไม่มี return value (void)
void SayHello()
{
    Console.WriteLine("สวัสดี!");
}

// มี return value
int Add(int a, int b)
{
    return a + b;
}

// เรียกใช้
SayHello();                    // สวัสดี!
int result = Add(3, 4);        // 7
Console.WriteLine(result);
```

### Return Types ต่างๆ

```csharp
// คืนค่าหลายชนิด
int GetAge() => 25;
string GetName() => "สมชาย";
bool IsAdult(int age) => age >= 18;
double[] GetTemperatures() => new[] { 28.5, 31.0, 29.8 };

// คืน Tuple (หลายค่า)
(string Name, int Age) GetPerson()
{
    return ("สมชาย", 25);
}

var person = GetPerson();
Console.WriteLine($"{person.Name}, {person.Age}");

// Destructuring
var (name, age) = GetPerson();
Console.WriteLine($"{name}, {age}");
```

---

## 2. Parameters

### Value Parameters (ค่าผ่าน)

```csharp
void Double(int x)
{
    x = x * 2;  // แก้ไข local copy ไม่กระทบ caller
}

int num = 5;
Double(num);
Console.WriteLine(num); // 5 (ไม่เปลี่ยน!)
```

### ref Parameters (ส่งอ้างอิง)

```csharp
void DoubleRef(ref int x)
{
    x = x * 2;  // แก้ไข original
}

int num = 5;
DoubleRef(ref num);
Console.WriteLine(num); // 10 (เปลี่ยนแล้ว!)

// Swap สองค่า
void Swap(ref int a, ref int b)
{
    int temp = a;
    a = b;
    b = temp;
}

int x = 10, y = 20;
Swap(ref x, ref y);
Console.WriteLine($"x={x}, y={y}"); // x=20, y=10
```

### out Parameters (ส่งค่าออก)

```csharp
bool TryParseAge(string input, out int age)
{
    if (int.TryParse(input, out age) && age >= 0 && age <= 150)
        return true;
    
    age = -1; // ต้องกำหนดค่าก่อน return
    return false;
}

if (TryParseAge("25", out int parsedAge))
{
    Console.WriteLine($"อายุ: {parsedAge}");
}
else
{
    Console.WriteLine("ข้อมูลไม่ถูกต้อง");
}

// out variable declaration (C# 7+) - ประกาศ inline
if (int.TryParse("42", out int value))
{
    Console.WriteLine(value);
}
// ถ้าไม่สนใจค่า ใช้ _
if (int.TryParse("abc", out _))
    Console.WriteLine("เป็นตัวเลข");
```

### in Parameters (readonly reference)

```csharp
// in: ส่ง by reference แต่ห้ามแก้ไข (performance สำหรับ structs ใหญ่)
void PrintPoint(in System.Drawing.Point p)
{
    Console.WriteLine($"({p.X}, {p.Y})");
    // p.X = 10; // ❌ Error! ห้ามแก้ไข
}
```

---

## 3. Method Overloading

```csharp
// Method เดียวกัน parameter ต่างกัน
int Add(int a, int b) => a + b;
double Add(double a, double b) => a + b;
int Add(int a, int b, int c) => a + b + c;
string Add(string a, string b) => a + b;

Console.WriteLine(Add(1, 2));         // 3 (int version)
Console.WriteLine(Add(1.5, 2.5));     // 4.0 (double version)
Console.WriteLine(Add(1, 2, 3));      // 6 (3-param version)
Console.WriteLine(Add("Hello", " World")); // Hello World
```

---

## 4. Optional Parameters

```csharp
// Default values
void Greet(string name, string greeting = "สวัสดี", bool exclaim = true)
{
    string msg = $"{greeting} {name}";
    Console.WriteLine(exclaim ? msg + "!" : msg);
}

Greet("สมชาย");                   // สวัสดี สมชาย!
Greet("สมชาย", "Hello");          // Hello สมชาย!
Greet("สมชาย", exclaim: false);   // สวัสดี สมชาย (Named argument)

// Named Arguments
void CreateUser(string name, int age, string role = "User", bool isActive = true)
{
    Console.WriteLine($"สร้าง: {name}, {age}, {role}, {isActive}");
}

// Named arguments ทำให้ชัดเจน
CreateUser("สมชาย", 25);
CreateUser("สมหญิง", 30, role: "Admin");
CreateUser(name: "วรา", age: 28, isActive: false);
```

---

## 5. Params Array

```csharp
// รับ parameters จำนวนไม่จำกัด
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int n in numbers)
        total += n;
    return total;
}

Console.WriteLine(Sum(1, 2));           // 3
Console.WriteLine(Sum(1, 2, 3));        // 6
Console.WriteLine(Sum(1, 2, 3, 4, 5)); // 15

int[] arr = { 10, 20, 30 };
Console.WriteLine(Sum(arr));            // 60 (ส่ง array ได้)

// ตัวอย่างจริง: Log
void Log(string level, string format, params object[] args)
{
    string message = string.Format(format, args);
    Console.ForegroundColor = level switch
    {
        "ERROR" => ConsoleColor.Red,
        "WARN" => ConsoleColor.Yellow,
        "INFO" => ConsoleColor.Cyan,
        _ => ConsoleColor.White
    };
    Console.WriteLine($"[{level}] {message}");
    Console.ResetColor();
}

Log("INFO", "เริ่ม server ที่ port {0}", 8080);
Log("WARN", "Memory usage {0}%", 85);
Log("ERROR", "ไม่สามารถเชื่อมต่อ {0}:{1}", "localhost", 5432);
```

---

## 6. Expression-bodied Members

```csharp
// แบบปกติ
int Square(int x)
{
    return x * x;
}

// Expression-bodied (=>)
int Square2(int x) => x * x;
double PI => Math.PI;
string Greet(string name) => $"สวัสดี {name}!";

// ใช้กับ void
void Print(string msg) => Console.WriteLine(msg);

// ใช้กับ property
class Circle
{
    public double Radius { get; set; }
    public double Area => Math.PI * Radius * Radius;
    public double Circumference => 2 * Math.PI * Radius;
    public string Description => $"วงกลม radius={Radius:F2}, area={Area:F2}";
}
```

---

## 7. Local Functions (C# 7+)

```csharp
// ฟังก์ชันภายในฟังก์ชัน
double CalculateTax(double income)
{
    // Local function
    double GetBracketTax(double amount, double rate) => amount * rate;
    
    if (income <= 150000) return 0;
    if (income <= 300000) return GetBracketTax(income - 150000, 0.05);
    // ...
    return GetBracketTax(income - 300000, 0.10) + 7500;
}

Console.WriteLine(CalculateTax(250000)); // 5000

// Local function ใน recursive
int Factorial(int n)
{
    if (n < 0) throw new ArgumentException("n ต้องไม่ติดลบ");
    
    // Local recursive function
    int Calc(int x) => x <= 1 ? 1 : x * Calc(x - 1);
    
    return Calc(n);
}

Console.WriteLine(Factorial(5));  // 120
Console.WriteLine(Factorial(10)); // 3628800
```

---

## 8. Recursive Methods

```csharp
// Recursion: เรียกตัวเอง
long Fibonacci(int n)
{
    if (n <= 1) return n;
    return Fibonacci(n - 1) + Fibonacci(n - 2);
}

// ⚠️ version นี้ช้ามากสำหรับ n ใหญ่!
for (int i = 0; i <= 10; i++)
    Console.Write($"{Fibonacci(i)} ");
Console.WriteLine();

// Fibonacci ที่เร็วกว่า (Memoization)
Dictionary<int, long> memo = new();
long FibFast(int n)
{
    if (n <= 1) return n;
    if (memo.ContainsKey(n)) return memo[n];
    
    memo[n] = FibFast(n - 1) + FibFast(n - 2);
    return memo[n];
}

// เร็วกว่ามาก!
Console.WriteLine(FibFast(50)); // 12586269025

// Factorial
ulong FactorialLoop(int n)
{
    ulong result = 1;
    for (int i = 2; i <= n; i++)
        result *= (ulong)i;
    return result;
}

Console.WriteLine(FactorialLoop(20)); // 2432902008176640000
```

---

## 9. โปรแกรมตัวอย่าง: Math Library

```csharp
// สร้าง Math utility methods
static class MathHelper
{
    // ค่าสัมบูรณ์
    public static double Abs(double x) => x < 0 ? -x : x;
    
    // ค่าสูงสุด/ต่ำสุด
    public static int Max(int a, int b) => a > b ? a : b;
    public static int Min(int a, int b) => a < b ? a : b;
    public static int Clamp(int value, int min, int max) 
        => value < min ? min : value > max ? max : value;
    
    // หาร.ร.ม.
    public static int GCD(int a, int b)
    {
        while (b != 0)
        {
            int temp = b;
            b = a % b;
            a = temp;
        }
        return a;
    }
    
    // ค.ร.น.
    public static int LCM(int a, int b) => a / GCD(a, b) * b;
    
    // ตรวจจำนวนเฉพาะ
    public static bool IsPrime(int n)
    {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;
        
        for (int i = 3; i <= Math.Sqrt(n); i += 2)
            if (n % i == 0) return false;
        
        return true;
    }
    
    // ปัดเศษ
    public static double RoundTo(double value, int decimals)
        => Math.Round(value, decimals, MidpointRounding.AwayFromZero);
    
    // แปลงหน่วย
    public static double CelsiusToFahrenheit(double celsius) => celsius * 9 / 5 + 32;
    public static double FahrenheitToCelsius(double fahrenheit) => (fahrenheit - 32) * 5 / 9;
    public static double KgToLbs(double kg) => kg * 2.20462;
    public static double LbsToKg(double lbs) => lbs / 2.20462;
    
    // สถิติ
    public static double Mean(double[] values)
    {
        if (values.Length == 0) return 0;
        double sum = 0;
        foreach (double v in values) sum += v;
        return sum / values.Length;
    }
    
    public static double Median(double[] values)
    {
        if (values.Length == 0) return 0;
        double[] sorted = (double[])values.Clone();
        Array.Sort(sorted);
        
        int mid = sorted.Length / 2;
        return sorted.Length % 2 == 0 
            ? (sorted[mid - 1] + sorted[mid]) / 2 
            : sorted[mid];
    }
    
    public static double StandardDeviation(double[] values)
    {
        double mean = Mean(values);
        double sum = 0;
        foreach (double v in values)
            sum += Math.Pow(v - mean, 2);
        return Math.Sqrt(sum / values.Length);
    }
}

// การใช้งาน
Console.WriteLine($"GCD(12,8) = {MathHelper.GCD(12, 8)}");    // 4
Console.WriteLine($"LCM(4,6) = {MathHelper.LCM(4, 6)}");      // 12
Console.WriteLine($"IsPrime(17) = {MathHelper.IsPrime(17)}");  // true
Console.WriteLine($"IsPrime(15) = {MathHelper.IsPrime(15)}");  // false

double[] data = { 85, 92, 78, 95, 88 };
Console.WriteLine($"Mean: {MathHelper.Mean(data):F2}");
Console.WriteLine($"Median: {MathHelper.Median(data):F2}");
Console.WriteLine($"StdDev: {MathHelper.StandardDeviation(data):F2}");
```

---

## 10. Exercises

### Exercise 1: String Helper
```csharp
// สร้าง method เหล่านี้:
// 1. IsPalindrome(string s) → bool
//    "racecar" = true, "hello" = false
// 2. CountWords(string s) → int
// 3. Capitalize(string s) → string
//    "hello world" → "Hello World"
// 4. Reverse(string s) → string
```

### Exercise 2: Number Helper
```csharp
// สร้าง:
// 1. IsEven/IsOdd(int n)
// 2. ToRoman(int n) → string (1-3999)
//    1="I", 4="IV", 5="V", 9="IX", 10="X"...
// 3. IsArmstrong(int n) → bool
//    153 = 1³+5³+3³ = true
```

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ Method declaration, parameters, return values
- ✅ ref, out, in parameters
- ✅ Method Overloading
- ✅ Optional Parameters และ Named Arguments
- ✅ params array
- ✅ Expression-bodied Members
- ✅ Local Functions
- ✅ Recursion

## Part ถัดไป
**[Part 011: String และการจัดการข้อความ →](part-011.md)**

---

*Part 010/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
