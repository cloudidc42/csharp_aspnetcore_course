# Part 002: ตัวแปรและชนิดข้อมูลพื้นฐาน

## เนื้อหาใน Part นี้
- ตัวแปรคืออะไร
- ชนิดข้อมูลพื้นฐาน (Value Types)
- ชนิดข้อมูลอ้างอิง (Reference Types)
- การประกาศและกำหนดค่าตัวแปร
- ตัวแปรแบบ `var` (Type Inference)
- ค่าคงที่ (Constants)
- Nullable Types
- การแปลงชนิดข้อมูล (Type Conversion)

---

## 1. ตัวแปรคืออะไร?

**ตัวแปร (Variable)** คือพื้นที่ในหน่วยความจำที่ใช้เก็บข้อมูล โดยเราตั้งชื่อให้กับพื้นที่นั้นเพื่อใช้อ้างอิงในภายหลัง

```
หน่วยความจำ RAM:
┌────────────────────────────────────────┐
│  ที่อยู่: 0x1234   ชื่อ: age  ค่า: 25  │
│  ที่อยู่: 0x1238   ชื่อ: name ค่า: "สมชาย" │
│  ที่อยู่: 0x1240   ชื่อ: price ค่า: 99.99 │
└────────────────────────────────────────┘
```

**กฎการตั้งชื่อตัวแปร:**
- ขึ้นต้นด้วยตัวอักษรหรือ `_`
- ประกอบด้วยตัวอักษร ตัวเลข `_`
- ห้ามใช้ Reserved Keywords เช่น `int`, `class`, `if`
- Case-sensitive (`age` ≠ `Age` ≠ `AGE`)
- ใช้ camelCase convention เช่น `firstName`, `totalPrice`

```csharp
// ✅ ชื่อที่ถูกต้อง
int age;
string firstName;
double totalPrice;
bool isActive;
int _privateField;

// ❌ ชื่อที่ผิด
int 1age;        // ขึ้นต้นด้วยตัวเลข
string first name; // มีช่องว่าง
bool int;        // ใช้ keyword
```

---

## 2. ชนิดข้อมูลพื้นฐาน (Primitive Types)

### ตัวเลขจำนวนเต็ม (Integer Types)

```csharp
// sbyte: -128 ถึง 127 (1 byte)
sbyte temperature = -10;

// byte: 0 ถึง 255 (1 byte)
byte r = 255;
byte g = 128;
byte b = 0;

// short: -32,768 ถึง 32,767 (2 bytes)
short year = 2024;

// ushort: 0 ถึง 65,535 (2 bytes)
ushort port = 8080;

// int: -2,147,483,648 ถึง 2,147,483,647 (4 bytes) ← ใช้บ่อยที่สุด
int population = 70_000_000; // ใช้ _ แทน comma ได้
int bankBalance = -500;

// uint: 0 ถึง 4,294,967,295 (4 bytes)
uint fileSize = 1_048_576; // 1 MB

// long: -9,223,372,036,854,775,808 ถึง ... (8 bytes)
long nationalDebt = 9_000_000_000_000L; // ต่อท้ายด้วย L

// ulong: 0 ถึง 18,446,744,073,709,551,615 (8 bytes)
ulong bigNumber = 18_446_744_073_709_551_615UL;
```

**เมื่อไหร่ควรใช้อะไร:**
```
byte   → สี RGB, ข้อมูลไบนารี
short  → ปี, อุณหภูมิ
int    → ✅ ใช้ทั่วไป (default choice)
long   → ID จำนวนมาก, timestamp, file size ใหญ่
```

### ตัวเลขทศนิยม (Floating-Point Types)

```csharp
// float: ~7 ตัวเลข (4 bytes) - ความแม่นยำต่ำ
float price = 99.99f;         // ต้องต่อท้ายด้วย f หรือ F
float pi = 3.14f;

// double: ~15-17 ตัวเลข (8 bytes) ← ใช้บ่อยที่สุด
double latitude = 13.756331;
double pi2 = 3.14159265358979;
double salary = 85_000.50;     // ไม่ต้องต่อท้าย d (default)

// decimal: ~28-29 ตัวเลข (16 bytes) - ความแม่นยำสูงสุด
decimal money = 1_000_000.99m; // ต้องต่อท้ายด้วย m หรือ M
decimal taxRate = 0.07m;
decimal productPrice = 299.00m;
```

**เมื่อไหร่ควรใช้อะไร:**
```
float   → ❌ หลีกเลี่ยง (ความแม่นยำต่ำ)
double  → ✅ การคำนวณทั่วไป, วิทยาศาสตร์
decimal → ✅✅ เงิน, การเงิน, ราคา (ห้ามใช้ double กับเงิน!)
```

**ตัวอย่างปัญหา Float Precision:**
```csharp
double a = 0.1 + 0.2;
Console.WriteLine(a); // ได้ 0.30000000000000004 !!

decimal b = 0.1m + 0.2m;
Console.WriteLine(b); // ได้ 0.3 ✅
```

### ตัวอักษรและข้อความ

```csharp
// char: ตัวอักษรเดี่ยว (2 bytes - Unicode)
char letter = 'A';
char digit = '5';
char thaiChar = 'ก';
char space = ' ';
char newline = '\n';  // Escape characters

// string: ข้อความ (Reference Type)
string name = "สมชาย";
string empty = "";
string nullString = null;     // string สามารถเป็น null ได้
string multiLine = @"บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3";
```

**Escape Characters ที่ใช้บ่อย:**
```csharp
char tab = '\t';        // Tab
char newLine = '\n';    // New Line
char carriage = '\r';   // Carriage Return
char backslash = '\\';  // Backslash
char singleQuote = '\''; // Single Quote
char doubleQuote = '\"'; // Double Quote
char nullChar = '\0';   // Null Character

// ตัวอย่างการใช้งาน
Console.WriteLine("Name:\tสมชาย");      // Name:   สมชาย
Console.WriteLine("Line1\nLine2");       // บรรทัดใหม่
Console.WriteLine("C:\\Users\\Desktop"); // C:\Users\Desktop
```

### Boolean

```csharp
// bool: true หรือ false เท่านั้น
bool isLoggedIn = true;
bool hasPermission = false;
bool isAdult = true;
bool isValid = false;

// ใช้กับ expressions
int age = 20;
bool canVote = age >= 18;    // true
bool isChild = age < 12;     // false
bool isTeen = age >= 13 && age <= 19; // false
```

---

## 3. ตาราง Built-in Types

```
C# Type    .NET Type         Size      Range
─────────────────────────────────────────────────────────────
bool       System.Boolean    1 byte    true/false
byte       System.Byte       1 byte    0 to 255
sbyte      System.SByte      1 byte    -128 to 127
char       System.Char       2 bytes   U+0000 to U+FFFF
short      System.Int16      2 bytes   -32,768 to 32,767
ushort     System.UInt16     2 bytes   0 to 65,535
int        System.Int32      4 bytes   -2.1B to 2.1B
uint       System.UInt32     4 bytes   0 to 4.3B
long       System.Int64      8 bytes   -9.2Q to 9.2Q
ulong      System.UInt64     8 bytes   0 to 18.4Q
float      System.Single     4 bytes   ±1.5×10⁻⁴⁵ to ±3.4×10³⁸
double     System.Double     8 bytes   ±5.0×10⁻³²⁴ to ±1.7×10³⁰⁸
decimal    System.Decimal   16 bytes   ±1.0×10⁻²⁸ to ±7.9×10²⁸
string     System.String    varies    sequence of chars
object     System.Object    varies    any type
```

---

## 4. การประกาศตัวแปร

```csharp
// รูปแบบ: [ชนิดข้อมูล] [ชื่อตัวแปร] = [ค่าเริ่มต้น];

// ประกาศพร้อมกำหนดค่า
int age = 25;
string name = "สมชาย";
double height = 175.5;
bool isStudent = true;

// ประกาศก่อน กำหนดค่าทีหลัง
int score;
score = 95; // กำหนดค่าภายหลัง

// ประกาศหลายตัวพร้อมกัน (ไม่แนะนำ เพราะอ่านยาก)
int x, y, z;
int a = 1, b = 2, c = 3;

// ค่า default ของตัวแปรถ้าไม่กำหนด (สำหรับ class fields)
int defaultInt;    // 0
double defaultDbl; // 0.0
bool defaultBool;  // false
string defaultStr; // null
char defaultChar;  // '\0'
```

### การแสดงค่าตัวแปร

```csharp
string firstName = "สมชาย";
string lastName = "ใจดี";
int age = 25;
double salary = 35000.50;
bool isEmployed = true;

// วิธีที่ 1: String Concatenation
Console.WriteLine("ชื่อ: " + firstName + " " + lastName);

// วิธีที่ 2: String Format
Console.WriteLine("อายุ: {0} ปี, เงินเดือน: {1:N2} บาท", age, salary);

// วิธีที่ 3: String Interpolation (แนะนำ) ← ใช้ $ หน้า string
Console.WriteLine($"ชื่อ: {firstName} {lastName}");
Console.WriteLine($"อายุ: {age} ปี");
Console.WriteLine($"เงินเดือน: {salary:N2} บาท");     // N2 = เลขทศนิยม 2 ตำแหน่ง พร้อม comma
Console.WriteLine($"ทำงาน: {isEmployed}");

// วิธีที่ 4: Raw String Literals (C# 11+)
string json = """
    {
        "name": "สมชาย",
        "age": 25
    }
    """;
Console.WriteLine(json);
```

---

## 5. var - Type Inference

**`var`** คือ keyword ที่ให้ compiler คิดชนิดข้อมูลให้อัตโนมัติ:

```csharp
// ทั้งคู่เหมือนกันทุกอย่าง
int x = 10;
var y = 10;    // compiler รู้ว่า y เป็น int

// ตัวอย่างการใช้ var
var name = "สมชาย";           // string
var age = 25;                 // int
var price = 99.99;            // double
var isValid = true;           // bool
var list = new List<string>(); // List<string>

// ❌ var ต้องกำหนดค่าพร้อมกัน
var x; // Error! ไม่รู้ชนิดข้อมูล

// ✅ เมื่อไหร่ควรใช้ var?
// 1. ชนิดข้อมูลชัดเจนจากค่าที่กำหนด
var count = 0;
var message = "Hello";

// 2. ชื่อชนิดข้อมูลยาวมาก
var employee = new Dictionary<string, List<int>>();

// 3. Anonymous Types
var person = new { Name = "สมชาย", Age = 25 };
```

---

## 6. ค่าคงที่ (Constants)

### const - Compile-time Constant

```csharp
// const: ค่าที่ไม่เปลี่ยนแปลงตลอดโปรแกรม
const double PI = 3.14159265358979;
const int MAX_SIZE = 100;
const string APP_NAME = "MyApplication";
const int DAYS_IN_WEEK = 7;

// ❌ ไม่สามารถเปลี่ยนค่าได้
PI = 3.0; // Error!

// ตัวอย่างการใช้งาน
double radius = 5.0;
double area = PI * radius * radius;
Console.WriteLine($"พื้นที่วงกลม: {area:F2}");

// const กับ string
const string CONNECTION_STRING = "Server=localhost;Database=mydb";
Console.WriteLine($"เชื่อมต่อ: {CONNECTION_STRING}");
```

### readonly - Runtime Constant

```csharp
class Config
{
    // readonly: กำหนดค่าได้ครั้งเดียวใน constructor
    readonly DateTime startDate;
    readonly string applicationName;
    
    public Config()
    {
        startDate = DateTime.Now;   // กำหนดใน constructor ได้
        applicationName = "MyApp";
    }
    
    public void ShowInfo()
    {
        Console.WriteLine($"App: {applicationName}");
        Console.WriteLine($"Started: {startDate}");
    }
}
```

### const vs readonly

| | `const` | `readonly` |
|--|---------|------------|
| กำหนดค่าได้เมื่อ | ประกาศ | ประกาศหรือ constructor |
| ชนิดที่ใช้ได้ | Primitive, string | ทุกชนิด |
| การคำนวณ | Compile-time | Runtime |
| ตัวอย่างการใช้ | PI, MAX_SIZE | DateTime, Config objects |

---

## 7. Nullable Types

ในภาษา C# ตัวแปรประเภท Value Type (int, double, bool ฯลฯ) ปกติไม่สามารถเป็น null ได้:

```csharp
int age = null;    // ❌ Error!
double price = null; // ❌ Error!
```

แต่บางครั้งเราต้องการให้ตัวแปรเป็น null ได้ (เช่น ยังไม่ได้กรอกข้อมูล):

```csharp
// เพิ่ม ? หลังชนิดข้อมูล
int? age = null;           // ✅ Nullable int
double? price = null;      // ✅ Nullable double
bool? isVerified = null;   // ✅ Nullable bool
DateTime? birthDate = null; // ✅ Nullable DateTime

// กำหนดค่าได้ปกติ
age = 25;
price = 99.99;
isVerified = true;

// ตรวจสอบว่าเป็น null
if (age == null)
{
    Console.WriteLine("ยังไม่ได้กรอกอายุ");
}

// HasValue property
if (age.HasValue)
{
    Console.WriteLine($"อายุ: {age.Value} ปี");
}

// GetValueOrDefault
int ageValue = age.GetValueOrDefault();    // ถ้า null จะได้ 0
int ageValue2 = age.GetValueOrDefault(18); // ถ้า null จะได้ 18

// Null-coalescing operator (??)
int displayAge = age ?? 0;  // ถ้า age เป็น null ให้ใช้ 0
Console.WriteLine($"อายุที่แสดง: {displayAge}");

// Null-coalescing assignment (??=)
age ??= 25;  // ถ้า age เป็น null ให้กำหนดค่าเป็น 25
```

### Nullable Reference Types (C# 8+)

```csharp
// เปิดใช้งานใน .csproj:
// <Nullable>enable</Nullable>

// ปกติ string สามารถเป็น null ได้
string name = null;  // ⚠️ Warning ใน .NET 8+

// string? แสดงว่าตั้งใจให้เป็น null ได้
string? optionalName = null;  // ✅

// ต้องตรวจสอบ null ก่อนใช้
if (optionalName != null)
{
    Console.WriteLine(optionalName.Length);
}

// หรือใช้ null-conditional operator (?.)
Console.WriteLine(optionalName?.Length);  // จะแสดง null ถ้า optionalName เป็น null
Console.WriteLine(optionalName?.ToUpper()); // ไม่ crash ถ้า null

// Null-forgiving operator (!)
// บอก compiler ว่าเรามั่นใจว่าไม่เป็น null
Console.WriteLine(optionalName!.Length); // อันตราย! ถ้า null จะ crash
```

---

## 8. การแปลงชนิดข้อมูล (Type Conversion)

### Implicit Conversion (อัตโนมัติ)

```csharp
// จาก "เล็ก" ไป "ใหญ่" อัตโนมัติ ไม่มี data loss
int i = 42;
long l = i;          // int → long ✅
float f = i;         // int → float ✅
double d = i;        // int → double ✅
double d2 = 3.14f;  // float → double ✅

byte b = 100;
int i2 = b;          // byte → int ✅
long l2 = b;         // byte → long ✅
```

### Explicit Conversion (Cast)

```csharp
// จาก "ใหญ่" ไป "เล็ก" ต้องบอกชัดเจน อาจมี data loss
double pi = 3.14159;
int truncated = (int)pi;     // ตัดทศนิยมทิ้ง → 3
Console.WriteLine(truncated); // 3

long bigNum = 5_000_000_000L;
int smallNum = (int)bigNum;  // อาจ overflow!
Console.WriteLine(smallNum); // ผลลัพธ์ไม่ถูกต้อง!

// Checked context (ตรวจสอบ overflow)
try
{
    int safe = checked((int)bigNum); // จะ throw OverflowException
}
catch (OverflowException)
{
    Console.WriteLine("เกินขอบเขต!");
}

float fVal = 9.78f;
int iVal = (int)fVal; // 9 (ตัดทศนิยม ไม่ปัดเศษ)
```

### Convert Class

```csharp
// Convert: แปลงระหว่างชนิดต่างๆ พร้อม error handling
string ageStr = "25";
int age = Convert.ToInt32(ageStr);      // string → int
double price = Convert.ToDouble("99.99"); // string → double
bool flag = Convert.ToBoolean("true");    // string → bool
string str = Convert.ToString(42);        // int → string

// แปลงจาก/เป็น DateTime
string dateStr = "2024-01-15";
DateTime date = Convert.ToDateTime(dateStr);

// ❌ ถ้าแปลงไม่ได้ จะ throw FormatException
try
{
    int x = Convert.ToInt32("abc"); // Error!
}
catch (FormatException)
{
    Console.WriteLine("แปลงไม่ได้!");
}
```

### Parse และ TryParse

```csharp
// Parse: แปลง string เป็นชนิดอื่น
int x = int.Parse("42");           // ✅
double d = double.Parse("3.14");   // ✅
bool b = bool.Parse("true");       // ✅

// ❌ ถ้าแปลงไม่ได้ จะ throw Exception
int bad = int.Parse("hello");      // FormatException!

// TryParse: ปลอดภัยกว่า ไม่ throw Exception
string input = "123";
if (int.TryParse(input, out int result))
{
    Console.WriteLine($"แปลงสำเร็จ: {result}");
}
else
{
    Console.WriteLine("แปลงไม่ได้");
}

// TryParse กับค่าทศนิยม
string priceStr = "99.99";
if (double.TryParse(priceStr, out double price))
{
    Console.WriteLine($"ราคา: {price}");
}

// TryParse กับ DateTime
string dateStr = "2024-01-15";
if (DateTime.TryParse(dateStr, out DateTime date))
{
    Console.WriteLine($"วันที่: {date:dd/MM/yyyy}");
}
```

### ToString

```csharp
int num = 1234567;
double price = 12345.678;
DateTime now = DateTime.Now;

// รูปแบบตัวเลข
Console.WriteLine(num.ToString());           // "1234567"
Console.WriteLine(num.ToString("N0"));       // "1,234,567"
Console.WriteLine(num.ToString("D8"));       // "01234567" (8 หลัก)
Console.WriteLine(price.ToString("F2"));     // "12345.68" (2 ทศนิยม)
Console.WriteLine(price.ToString("C2"));     // "$12,345.68" (currency)
Console.WriteLine(price.ToString("N2"));     // "12,345.68" (number with comma)
Console.WriteLine(price.ToString("P0"));     // "1,234,568%" (percent)
Console.WriteLine(price.ToString("E2"));     // "1.23E+004" (scientific)

// รูปแบบวันที่
Console.WriteLine(now.ToString("dd/MM/yyyy"));        // "15/01/2024"
Console.WriteLine(now.ToString("dd MMMM yyyy"));      // "15 January 2024"
Console.WriteLine(now.ToString("HH:mm:ss"));          // "14:30:00"
Console.WriteLine(now.ToString("dd/MM/yyyy HH:mm")); // "15/01/2024 14:30"
```

---

## 9. ขอบเขตของตัวแปร (Scope)

```csharp
// 1. ตัวแปรระดับ Block
{
    int x = 10;
    Console.WriteLine(x); // ✅ อยู่ใน block เดียวกัน
}
// Console.WriteLine(x); // ❌ Error! x ไม่อยู่ใน scope แล้ว

// 2. ตัวแปรระดับ Method
void MyMethod()
{
    int localVar = 5; // ใช้ได้แค่ใน method นี้
}

// 3. ตัวแปรใน Loop
for (int i = 0; i < 5; i++)
{
    Console.WriteLine(i); // ✅
}
// Console.WriteLine(i); // ❌ i ไม่อยู่ใน scope แล้ว

// 4. Nested Scope
int outer = 1;
{
    int inner = 2;
    Console.WriteLine(outer); // ✅ เข้าถึง outer ได้
    Console.WriteLine(inner); // ✅
}
// Console.WriteLine(inner); // ❌ inner ไม่อยู่ใน scope แล้ว
Console.WriteLine(outer);    // ✅
```

---

## 10. โปรแกรมตัวอย่างสมบูรณ์

```csharp
// โปรแกรมคำนวณค่า BMI
using System;

// ข้อมูลนักเรียน
string name = "สมชาย ใจดี";
int age = 20;
double weight = 65.5;   // กิโลกรัม
double height = 1.72;   // เมตร

// คำนวณ BMI
double bmi = weight / (height * height);

// แสดงผล
Console.WriteLine("=============================");
Console.WriteLine("   ผลการคำนวณค่า BMI        ");
Console.WriteLine("=============================");
Console.WriteLine($"ชื่อ: {name}");
Console.WriteLine($"อายุ: {age} ปี");
Console.WriteLine($"น้ำหนัก: {weight} กก.");
Console.WriteLine($"ส่วนสูง: {height * 100:F0} ซม.");
Console.WriteLine($"BMI: {bmi:F2}");
Console.WriteLine();

// ประเมินผล
string category;
if (bmi < 18.5)
    category = "น้ำหนักต่ำกว่าเกณฑ์";
else if (bmi < 25.0)
    category = "น้ำหนักปกติ";
else if (bmi < 30.0)
    category = "น้ำหนักเกิน";
else
    category = "อ้วน";

Console.WriteLine($"ผลการประเมิน: {category}");
Console.WriteLine("=============================");

// แปลงหน่วย
double heightInCm = height * 100;
double heightInFeet = height * 3.28084;
double weightInLbs = weight * 2.20462;

Console.WriteLine("\nหน่วยอื่นๆ:");
Console.WriteLine($"ส่วนสูง: {heightInCm:F1} ซม. = {heightInFeet:F2} ฟุต");
Console.WriteLine($"น้ำหนัก: {weight} กก. = {weightInLbs:F1} ปอนด์");
```

**ผลลัพธ์:**
```
=============================
   ผลการคำนวณค่า BMI        
=============================
ชื่อ: สมชาย ใจดี
อายุ: 20 ปี
น้ำหนัก: 65.5 กก.
ส่วนสูง: 172 ซม.
BMI: 22.14
     
ผลการประเมิน: น้ำหนักปกติ
=============================

หน่วยอื่นๆ:
ส่วนสูง: 172.0 ซม. = 5.64 ฟุต
น้ำหนัก: 65.5 กก. = 144.4 ปอนด์
```

---

## 11. Exercises

### Exercise 1: ข้อมูลรถยนต์
สร้างโปรแกรมเก็บข้อมูลรถยนต์และคำนวณค่าใช้จ่ายเชื้อเพลิง:

```csharp
// กำหนดข้อมูลรถ
string brand = "Toyota";
string model = "Camry";
int year = 2023;
double fuelCapacity = 60.0; // ลิตร
double avgConsumption = 12.0; // กม./ลิตร
double fuelPrice = 40.5; // บาท/ลิตร

// TODO: คำนวณ
// 1. ระยะทางที่วิ่งได้เต็มถัง
// 2. ค่าน้ำมันเต็มถัง
// 3. ค่าน้ำมันต่อ 100 กม.

// แสดงผลให้ครบถ้วน
```

### Exercise 2: ตรวจสอบ Type

```csharp
// ใส่ค่าต่างๆ แล้วตรวจสอบชนิดข้อมูลด้วย GetType()
var values = new object[] { 42, 3.14, "hello", true, 'A', 100L, 9.99m };

foreach (var val in values)
{
    Console.WriteLine($"ค่า: {val,-10} ชนิด: {val.GetType().Name}");
}
```

### Exercise 3: แปลงหน่วยวัด

```csharp
// สร้างโปรแกรมแปลงหน่วย
double celsius = 100.0;

// TODO: แปลงเป็น
// 1. Fahrenheit: (C × 9/5) + 32
// 2. Kelvin: C + 273.15

// แสดงผล
```

---

## 12. สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ ตัวแปรและกฎการตั้งชื่อ
- ✅ ชนิดข้อมูลทั้งหมดใน C#
- ✅ เมื่อไหร่ควรใช้ชนิดไหน (int vs long, double vs decimal)
- ✅ `var` สำหรับ Type Inference
- ✅ `const` และ `readonly`
- ✅ Nullable Types (`int?`, `string?`)
- ✅ การแปลงชนิดข้อมูล (Implicit, Cast, Convert, Parse)
- ✅ Scope ของตัวแปร

## Part ถัดไป
**[Part 003: ตัวดำเนินการ (Operators) →](part-003.md)**

---

*Part 002/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
