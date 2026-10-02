# Part 003: ตัวดำเนินการ (Operators)

## เนื้อหาใน Part นี้
- Arithmetic Operators (คณิตศาสตร์)
- Comparison Operators (เปรียบเทียบ)
- Logical Operators (ตรรกะ)
- Assignment Operators (กำหนดค่า)
- Bitwise Operators (บิต)
- Ternary Operator
- Null-related Operators
- Operator Precedence

---

## 1. Arithmetic Operators (ตัวดำเนินการคณิตศาสตร์)

```csharp
int a = 20;
int b = 6;

// บวก
int sum = a + b;          // 26
Console.WriteLine($"{a} + {b} = {sum}");

// ลบ
int diff = a - b;         // 14
Console.WriteLine($"{a} - {b} = {diff}");

// คูณ
int product = a * b;      // 120
Console.WriteLine($"{a} * {b} = {product}");

// หาร (จำนวนเต็ม - ตัดเศษทิ้ง)
int quotient = a / b;     // 3 (ไม่ใช่ 3.33!)
Console.WriteLine($"{a} / {b} = {quotient}");

// หาร (ทศนิยม - ต้องใช้ double/float)
double exactDiv = (double)a / b;  // 3.3333...
Console.WriteLine($"{a} / {b} = {exactDiv:F4}");

// เศษหาร (Modulo)
int remainder = a % b;    // 2 (20 = 6×3 + 2)
Console.WriteLine($"{a} % {b} = {remainder}");

// ยกกำลัง (ใช้ Math.Pow)
double power = Math.Pow(2, 10);  // 1024
Console.WriteLine($"2^10 = {power}");
```

### เพิ่ม/ลดค่าทีละ 1

```csharp
int x = 5;

// Increment
x++;           // x = 6 (post-increment)
++x;           // x = 7 (pre-increment)
x = x + 1;    // x = 8 (เหมือนกัน)

// Decrement
x--;           // x = 7 (post-decrement)
--x;           // x = 6 (pre-decrement)

// ความแตกต่างระหว่าง x++ และ ++x
int a = 5;
int b = a++;   // b = 5, a = 6 (ใช้ค่า a ก่อน แล้วค่อยเพิ่ม)

int c = 5;
int d = ++c;   // d = 6, c = 6 (เพิ่มก่อน แล้วค่อยใช้ค่า)

Console.WriteLine($"a = {a}, b = {b}"); // a = 6, b = 5
Console.WriteLine($"c = {c}, d = {d}"); // c = 6, d = 6
```

### ตัวอย่างการใช้งานจริง

```csharp
// คำนวณดอกเบี้ย
double principal = 100000.0;  // เงินต้น
double rate = 0.05;           // ดอกเบี้ย 5% ต่อปี
int years = 3;

// ดอกเบี้ยทบต้น: A = P(1 + r)^n
double amount = principal * Math.Pow(1 + rate, years);
double interest = amount - principal;

Console.WriteLine($"เงินต้น: {principal:N0} บาท");
Console.WriteLine($"ดอกเบี้ย: {rate * 100}% ต่อปี");
Console.WriteLine($"จำนวนปี: {years} ปี");
Console.WriteLine($"ดอกเบี้ยรวม: {interest:N2} บาท");
Console.WriteLine($"เงินรวม: {amount:N2} บาท");

// คำนวณระยะทาง
double speed = 120.0;  // กม./ชม.
double time = 2.5;     // ชั่วโมง
double distance = speed * time;

Console.WriteLine($"\nความเร็ว: {speed} กม./ชม.");
Console.WriteLine($"เวลา: {time} ชม.");
Console.WriteLine($"ระยะทาง: {distance} กม.");
```

---

## 2. Comparison Operators (ตัวดำเนินการเปรียบเทียบ)

```csharp
int x = 10;
int y = 20;

bool isEqual = x == y;         // false (เท่ากัน)
bool isNotEqual = x != y;      // true (ไม่เท่ากัน)
bool isGreater = x > y;        // false (มากกว่า)
bool isLess = x < y;           // true (น้อยกว่า)
bool isGreaterOrEqual = x >= y; // false (มากกว่าหรือเท่ากัน)
bool isLessOrEqual = x <= y;   // true (น้อยกว่าหรือเท่ากัน)

Console.WriteLine($"x = {x}, y = {y}");
Console.WriteLine($"x == y: {isEqual}");
Console.WriteLine($"x != y: {isNotEqual}");
Console.WriteLine($"x > y:  {isGreater}");
Console.WriteLine($"x < y:  {isLess}");
Console.WriteLine($"x >= y: {isGreaterOrEqual}");
Console.WriteLine($"x <= y: {isLessOrEqual}");
```

### เปรียบเทียบ String

```csharp
string s1 = "Hello";
string s2 = "hello";
string s3 = "Hello";

// == เปรียบเทียบค่า (case-sensitive)
Console.WriteLine(s1 == s2);  // false
Console.WriteLine(s1 == s3);  // true

// Compare methods
bool same = s1.Equals(s2);                           // false (case-sensitive)
bool sameIgnoreCase = s1.Equals(s2, StringComparison.OrdinalIgnoreCase); // true

// CompareTo: ใช้เรียงลำดับ
int compare = s1.CompareTo(s2);  // ค่าลบ = s1 น้อยกว่า, 0 = เท่ากัน, ค่าบวก = s1 มากกว่า
Console.WriteLine(compare); // ค่าลบ (H < h ใน ASCII)

// String.Compare
int result = string.Compare(s1, s2, ignoreCase: true);
Console.WriteLine(result); // 0 (เท่ากัน ถ้าไม่สนตัวพิมพ์)
```

---

## 3. Logical Operators (ตัวดำเนินการตรรกะ)

```csharp
bool a = true;
bool b = false;

// AND (&&): ทั้งคู่ต้องเป็น true
bool andResult = a && b;    // false
Console.WriteLine($"true && false = {andResult}");

// OR (||): อย่างน้อยหนึ่งเป็น true
bool orResult = a || b;     // true
Console.WriteLine($"true || false = {orResult}");

// NOT (!): กลับค่า
bool notA = !a;             // false
bool notB = !b;             // true
Console.WriteLine($"!true = {notA}");
Console.WriteLine($"!false = {notB}");

// XOR (^): ต่างกันจึง true
bool xorResult = a ^ b;     // true
bool xorSame = a ^ a;       // false (เหมือนกัน → false)
Console.WriteLine($"true ^ false = {xorResult}");
Console.WriteLine($"true ^ true = {xorSame}");
```

### ตาราง Truth Table

```
AND (&&):
true  && true  = true
true  && false = false
false && true  = false
false && false = false

OR (||):
true  || true  = true
true  || false = true
false || true  = true
false || false = false

NOT (!):
!true  = false
!false = true

XOR (^):
true  ^ true  = false
true  ^ false = true
false ^ true  = true
false ^ false = false
```

### Short-circuit Evaluation

```csharp
int x = 10;
int y = 0;

// && Short-circuit: ถ้า left เป็น false จะไม่ evaluate right
// ป้องกัน DivisionByZero!
if (y != 0 && x / y > 2)
{
    Console.WriteLine("ผ่านเงื่อนไข");
}
else
{
    Console.WriteLine("ไม่ผ่าน (ปลอดภัย!)"); // ได้ผลนี้
}

// || Short-circuit: ถ้า left เป็น true จะไม่ evaluate right
string name = null;
if (name == null || name.Length == 0)
{
    Console.WriteLine("ชื่อว่าง"); // ปลอดภัยจาก NullReferenceException
}

// ⚠️ & และ | คือ Non-short-circuit (evaluate ทั้งสอง)
if (y != 0 & x / y > 2)  // ⚠️ จะ throw DivisionByZeroException!
{
    // ...
}
```

### การรวมเงื่อนไข

```csharp
int age = 25;
bool hasLicense = true;
bool hasCar = true;
double salary = 35000;

// ตรวจสอบเงื่อนไขหลายอย่าง
bool canDrive = age >= 18 && hasLicense;
bool canLoan = age >= 20 && salary >= 30000 && (hasLicense || hasCar);
bool isQualified = (age >= 18 && age <= 60) && (salary >= 25000 || hasCar);

Console.WriteLine($"ขับรถได้: {canDrive}");
Console.WriteLine($"กู้ได้: {canLoan}");
Console.WriteLine($"มีคุณสมบัติ: {isQualified}");
```

---

## 4. Assignment Operators (ตัวดำเนินการกำหนดค่า)

```csharp
int x = 10;

// Basic assignment
x = 20;              // x = 20

// Compound assignment
x += 5;              // x = x + 5 = 25
x -= 3;              // x = x - 3 = 22
x *= 2;              // x = x * 2 = 44
x /= 4;              // x = x / 4 = 11
x %= 3;              // x = x % 3 = 2

// Power assignment (ไม่มีใน C# ต้องใช้ Math.Pow)
// x **= 3; ❌ ไม่มี

// Null-coalescing assignment (C# 8+)
string name = null;
name ??= "ไม่ระบุ";  // name = name ?? "ไม่ระบุ"
Console.WriteLine(name); // ไม่ระบุ

string name2 = "สมชาย";
name2 ??= "ไม่ระบุ";
Console.WriteLine(name2); // สมชาย (ไม่เปลี่ยน เพราะไม่เป็น null)
```

---

## 5. Bitwise Operators (ตัวดำเนินการบิต)

```csharp
// ใช้กับ integer types
int a = 0b_1010_1010;  // 170 ในเลขฐาน 2
int b = 0b_1100_1100;  // 204 ในเลขฐาน 2

// AND (&): ทั้งคู่เป็น 1 จึงได้ 1
int andResult = a & b;   // 0b_1000_1000 = 136
Console.WriteLine($"AND: {andResult} = {Convert.ToString(andResult, 2).PadLeft(8, '0')}");

// OR (|): อย่างน้อยหนึ่งเป็น 1
int orResult = a | b;    // 0b_1110_1110 = 238
Console.WriteLine($"OR:  {orResult} = {Convert.ToString(orResult, 2).PadLeft(8, '0')}");

// XOR (^): ต่างกันจึงได้ 1
int xorResult = a ^ b;   // 0b_0110_0110 = 102
Console.WriteLine($"XOR: {xorResult} = {Convert.ToString(xorResult, 2).PadLeft(8, '0')}");

// NOT (~): กลับทุกบิต
int notResult = ~a;      // -171 (Two's complement)
Console.WriteLine($"NOT: {notResult}");

// Left Shift (<<): เลื่อนบิตไปซ้าย = คูณ 2^n
int shifted = 5 << 2;   // 5 * 4 = 20
Console.WriteLine($"5 << 2 = {shifted}");

// Right Shift (>>): เลื่อนบิตไปขวา = หาร 2^n
int rShifted = 20 >> 2; // 20 / 4 = 5
Console.WriteLine($"20 >> 2 = {rShifted}");
```

### การใช้งาน Bitwise ในชีวิตจริง

```csharp
// Flags Enum - ใช้บ่อยมากใน .NET
[Flags]
enum Permissions
{
    None = 0,
    Read = 1,    // 001
    Write = 2,   // 010
    Execute = 4, // 100
    All = Read | Write | Execute  // 111
}

Permissions userPerm = Permissions.Read | Permissions.Write;
Console.WriteLine($"สิทธิ์: {userPerm}");

// ตรวจสอบสิทธิ์
bool canRead = (userPerm & Permissions.Read) != 0;
bool canExecute = (userPerm & Permissions.Execute) != 0;
Console.WriteLine($"อ่านได้: {canRead}");       // true
Console.WriteLine($"รันได้: {canExecute}");      // false

// เพิ่มสิทธิ์
userPerm |= Permissions.Execute;
canExecute = (userPerm & Permissions.Execute) != 0;
Console.WriteLine($"รันได้หลังเพิ่ม: {canExecute}"); // true

// ลบสิทธิ์
userPerm &= ~Permissions.Write;
bool canWrite = (userPerm & Permissions.Write) != 0;
Console.WriteLine($"เขียนได้หลังลบ: {canWrite}"); // false
```

---

## 6. Ternary Operator

```csharp
// รูปแบบ: เงื่อนไข ? ค่าถ้าจริง : ค่าถ้าเท็จ
int age = 20;
string status = age >= 18 ? "ผู้ใหญ่" : "เด็ก";
Console.WriteLine(status); // ผู้ใหญ่

// ซ้อนกัน (แต่ไม่แนะนำ อ่านยาก)
int score = 75;
string grade = score >= 80 ? "A" 
             : score >= 70 ? "B" 
             : score >= 60 ? "C" 
             : "F";
Console.WriteLine($"เกรด: {grade}"); // B

// ใช้ใน string interpolation
double temperature = 25.0;
string weather = $"อุณหภูมิ {temperature}°C: {(temperature > 30 ? "ร้อน" : temperature > 20 ? "อุ่น" : "เย็น")}";
Console.WriteLine(weather);

// ใช้กับ method call
int number = -5;
int absolute = number >= 0 ? number : -number;
Console.WriteLine($"|{number}| = {absolute}"); // 5
```

---

## 7. Null-related Operators

```csharp
string name = null;
string displayName = "สมชาย";

// Null-coalescing operator (??)
string result1 = name ?? "ไม่ระบุ";          // "ไม่ระบุ"
string result2 = displayName ?? "ไม่ระบุ";   // "สมชาย"

// Null-conditional operator (?.)
string? upperName = name?.ToUpper();          // null (ไม่ crash)
int? length = name?.Length;                   // null

string realName = "สมชาย";
string? upperReal = realName?.ToUpper();      // "สมชาย" (ToUpper)

// รวมกัน: ?. และ ??
string? nullableName = null;
string display = nullableName?.ToUpper() ?? "ไม่ระบุ";
Console.WriteLine(display); // "ไม่ระบุ"

// Null-conditional กับ indexer
int[]? array = null;
int? firstElement = array?[0];  // null (ไม่ crash)

int[] realArray = { 1, 2, 3 };
int? realFirst = realArray?[0]; // 1

// Null-forgiving operator (!)
string? possiblyNull = GetSomething();
// เมื่อเรามั่นใจว่าไม่เป็น null
string definitelyNotNull = possiblyNull!;
Console.WriteLine(definitelyNotNull.Length);

static string? GetSomething() => "hello";
```

---

## 8. typeof, sizeof, nameof

```csharp
// typeof: ได้ Type object
Type intType = typeof(int);
Type stringType = typeof(string);
Console.WriteLine(intType.Name);    // Int32
Console.WriteLine(stringType.Name); // String

// is: ตรวจสอบชนิด
object obj = "Hello";
bool isString = obj is string;  // true
bool isInt = obj is int;        // false
Console.WriteLine(isString);

// is กับ Pattern Matching
if (obj is string str)
{
    Console.WriteLine($"เป็น string ยาว {str.Length} ตัวอักษร");
}

// sizeof: ขนาด value type (bytes) - ใช้ใน unsafe context ปกติ
// หรือใช้ Marshal.SizeOf
Console.WriteLine(sizeof(int));     // 4
Console.WriteLine(sizeof(double));  // 8
Console.WriteLine(sizeof(char));    // 2
Console.WriteLine(sizeof(bool));    // 1

// nameof: ได้ชื่อตัวแปร/method/property เป็น string
int myVariable = 10;
Console.WriteLine(nameof(myVariable)); // "myVariable"

// ประโยชน์ของ nameof: ป้องกันการพิมพ์ชื่อผิด
void ValidateAge(int age)
{
    if (age < 0)
    {
        throw new ArgumentException("อายุต้องมากกว่า 0", nameof(age));
    }
}
```

---

## 9. Operator Precedence (ลำดับการทำงาน)

```
ลำดับสูงสุด → ต่ำสุด:
1.  ()  []  .  ->  ++  --  (postfix)
2.  !  ~  ++  --  (prefix)  (type)  sizeof
3.  *  /  %
4.  +  -
5.  <<  >>
6.  <  >  <=  >=  is  as
7.  ==  !=
8.  &
9.  ^
10. |
11. &&
12. ||
13. ??
14. ?:
15. =  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=  ??=
```

```csharp
// ตัวอย่าง Precedence
int result1 = 2 + 3 * 4;      // 14 (คูณก่อน)
int result2 = (2 + 3) * 4;    // 20 (วงเล็บก่อน)

bool result3 = true || false && false;  // true (AND ก่อน OR)
// เหมือน: true || (false && false) = true || false = true

bool result4 = (true || false) && false; // false (วงเล็บก่อน)

// ตัวอย่างที่อาจสับสน
int a = 5;
int b = ++a * 2;  // a = 6 ก่อน แล้วค่อย 6 * 2 = 12
Console.WriteLine($"a = {a}, b = {b}"); // a = 6, b = 12

int c = 5;
int d = c++ * 2;  // 5 * 2 = 10 ก่อน แล้วค่อย c = 6
Console.WriteLine($"c = {c}, d = {d}"); // c = 6, d = 10
```

---

## 10. String Operators

```csharp
string first = "สวัสดี";
string last = " ชาวโลก";

// + Concatenation
string hello = first + last;
Console.WriteLine(hello); // สวัสดี ชาวโลก

// += Append
string greeting = "สวัสดี";
greeting += " ครับ";
Console.WriteLine(greeting); // สวัสดี ครับ

// == เปรียบเทียบเนื้อหา
string s1 = "hello";
string s2 = "hello";
Console.WriteLine(s1 == s2);  // true (เปรียบเทียบเนื้อหา ไม่ใช่ reference)

// Repeat (ไม่มี operator ต้องใช้ method)
string line = new string('-', 40);  // "----------------------------------------"
string stars = string.Concat(Enumerable.Repeat("* ", 5)); // "* * * * * "
```

---

## 11. โปรแกรมตัวอย่างสมบูรณ์

```csharp
// โปรแกรมคิดคะแนนนักเรียน

// ข้อมูลคะแนน
string studentName = "สมชาย ใจดี";
int mathScore = 85;
int scienceScore = 72;
int thaiScore = 90;
int englishScore = 78;
int socialScore = 88;

// คำนวณ
int totalScore = mathScore + scienceScore + thaiScore + englishScore + socialScore;
double average = (double)totalScore / 5;  // หารด้วย 5 วิชา

// คำนวณเกรดด้วย ternary
string grade = average >= 80 ? "A" 
             : average >= 70 ? "B" 
             : average >= 60 ? "C" 
             : average >= 50 ? "D" : "F";

// ตรวจสอบผ่าน/ไม่ผ่าน
bool passed = average >= 50 && mathScore >= 40 && thaiScore >= 40;

// สถานะด้วย bitwise flags
[Flags]
enum Achievement {
    None = 0,
    MathExcellent = 1,     // คณิตศาสตร์ยอดเยี่ยม >= 90
    ScienceGood = 2,       // วิทยาศาสตร์ดี >= 70
    PerfectAttendance = 4  // เข้าเรียนครบ
}

var achievements = Achievement.None;
if (mathScore >= 90) achievements |= Achievement.MathExcellent;
if (scienceScore >= 70) achievements |= Achievement.ScienceGood;
achievements |= Achievement.PerfectAttendance; // สมมติว่ามาครบ

// แสดงผล
Console.WriteLine("=" .PadRight(45, '='));
Console.WriteLine($"   ผลการเรียน: {studentName}");
Console.WriteLine("=".PadRight(45, '='));
Console.WriteLine($"  คณิตศาสตร์ : {mathScore,3} คะแนน");
Console.WriteLine($"  วิทยาศาสตร์: {scienceScore,3} คะแนน");
Console.WriteLine($"  ภาษาไทย   : {thaiScore,3} คะแนน");
Console.WriteLine($"  ภาษาอังกฤษ: {englishScore,3} คะแนน");
Console.WriteLine($"  สังคมศึกษา: {socialScore,3} คะแนน");
Console.WriteLine("-".PadRight(45, '-'));
Console.WriteLine($"  รวม        : {totalScore,3} คะแนน");
Console.WriteLine($"  เฉลี่ย     : {average,6:F2} คะแนน");
Console.WriteLine($"  เกรด       : {grade}");
Console.WriteLine($"  ผล         : {(passed ? "✅ ผ่าน" : "❌ ไม่ผ่าน")}");
Console.WriteLine($"  รางวัล     : {achievements}");
Console.WriteLine("=".PadRight(45, '='));

// คำนวณระดับ percentile
double maxPossible = 500.0;
double percentage = totalScore / maxPossible * 100;
Console.WriteLine($"\nคิดเป็น {percentage:F1}% จากคะแนนเต็ม");
```

**ผลลัพธ์:**
```
=============================================
   ผลการเรียน: สมชาย ใจดี
=============================================
  คณิตศาสตร์ :  85 คะแนน
  วิทยาศาสตร์:  72 คะแนน
  ภาษาไทย   :  90 คะแนน
  ภาษาอังกฤษ:  78 คะแนน
  สังคมศึกษา:  88 คะแนน
---------------------------------------------
  รวม        : 413 คะแนน
  เฉลี่ย     :  82.60 คะแนน
  เกรด       : A
  ผล         : ✅ ผ่าน
  รางวัล     : ScienceGood, PerfectAttendance
=============================================

คิดเป็น 82.6% จากคะแนนเต็ม
```

---

## 12. Exercises

### Exercise 1: เครื่องคิดเลข
```csharp
double num1 = 15.5;
double num2 = 4.0;

// TODO: คำนวณและแสดงผล
// - บวก ลบ คูณ หาร
// - เศษหาร
// - ยกกำลัง
// - ค่าสัมบูรณ์ของ (num1 - num2)
```

### Exercise 2: ตรวจสอบเลขคู่/คี่
```csharp
int number = 17;

// TODO: ใช้ % เพื่อตรวจสอบว่าเป็นเลขคู่หรือเลขคี่
// แสดง: "17 เป็นเลขคี่"
```

### Exercise 3: Bitwise สีผสม
```csharp
// RGB ใช้ byte (0-255)
byte red = 255;
byte green = 128;
byte blue = 0;

// TODO: รวมสีเป็น 32-bit integer (ARGB)
// สูตร: (alpha << 24) | (red << 16) | (green << 8) | blue
// alpha = 255 (ทึบ 100%)
int color = ???;

// TODO: แตกสีออกมา
byte r = (byte)((color >> 16) & 0xFF);
byte g = (byte)((color >> 8) & 0xFF);
byte b = (byte)(color & 0xFF);

Console.WriteLine($"R={r}, G={g}, B={b}");
```

---

## 13. สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ Arithmetic Operators (+, -, *, /, %, ++, --)
- ✅ Comparison Operators (==, !=, >, <, >=, <=)
- ✅ Logical Operators (&&, ||, !, ^) และ Short-circuit
- ✅ Assignment Operators (=, +=, -=, *=, /=, ??=)
- ✅ Bitwise Operators (&, |, ^, ~, <<, >>) และการใช้งานจริง
- ✅ Ternary Operator (?:)
- ✅ Null-related Operators (??, ?., !)
- ✅ Operator Precedence

## Part ถัดไป
**[Part 004: การรับและแสดงผลข้อมูล →](part-004.md)**

---

*Part 003/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
