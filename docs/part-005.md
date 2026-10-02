# Part 005: คำสั่งเงื่อนไข if-else

## เนื้อหาใน Part นี้
- if statement พื้นฐาน
- if-else และ if-else if-else
- Nested if
- Pattern Matching กับ if
- Guard Clauses (Early Return)
- Best Practices

---

## 1. if Statement พื้นฐาน

```csharp
// รูปแบบ: if (เงื่อนไข) { โค้ด }
int score = 75;

if (score >= 60)
{
    Console.WriteLine("ผ่าน!");
}

// ถ้ามีบรรทัดเดียว (ไม่แนะนำ - อาจทำให้สับสน)
if (score >= 60)
    Console.WriteLine("ผ่าน!");

// ✅ แนะนำ: ใส่ {} เสมอ ป้องกัน bug
if (score >= 60)
{
    Console.WriteLine("ผ่าน!");
    // เพิ่ม code ตรงนี้ได้ปลอดภัย
}
```

---

## 2. if-else

```csharp
int temperature = 28;

if (temperature > 30)
{
    Console.WriteLine("อากาศร้อน");
}
else
{
    Console.WriteLine("อากาศไม่ร้อน");
}

// ตัวอย่างจริง: ตรวจสอบ login
bool isLoggedIn = true;
string username = "สมชาย";

if (isLoggedIn)
{
    Console.WriteLine($"ยินดีต้อนรับ {username}!");
}
else
{
    Console.WriteLine("กรุณาเข้าสู่ระบบ");
}
```

---

## 3. if-else if-else (หลายเงื่อนไข)

```csharp
int score = 75;
string grade;

if (score >= 90)
{
    grade = "A";
}
else if (score >= 80)
{
    grade = "B";
}
else if (score >= 70)
{
    grade = "C";
}
else if (score >= 60)
{
    grade = "D";
}
else
{
    grade = "F";
}

Console.WriteLine($"คะแนน {score} = เกรด {grade}");

// ตัวอย่าง: ค่าธรรมเนียมตาม tier
decimal balance = 75000m;
decimal fee;

if (balance >= 100000m)
{
    fee = 0m; // ไม่มีค่าธรรมเนียม
}
else if (balance >= 50000m)
{
    fee = 50m;
}
else if (balance >= 10000m)
{
    fee = 100m;
}
else
{
    fee = 200m;
}

Console.WriteLine($"ยอดเงิน: {balance:N0} บาท → ค่าธรรมเนียม: {fee} บาท");
```

---

## 4. Nested if (if ซ้อนกัน)

```csharp
int age = 20;
bool hasID = true;
bool hasTicket = true;

// ✅ วิธีที่ 1: Nested if
if (age >= 18)
{
    if (hasID)
    {
        if (hasTicket)
        {
            Console.WriteLine("เข้าได้ 🎉");
        }
        else
        {
            Console.WriteLine("ไม่มีตั๋ว");
        }
    }
    else
    {
        Console.WriteLine("ต้องแสดง ID");
    }
}
else
{
    Console.WriteLine("อายุไม่ถึง");
}

// ✅ วิธีที่ 2: รวมเงื่อนไข (อ่านง่ายกว่า)
if (age >= 18 && hasID && hasTicket)
{
    Console.WriteLine("เข้าได้ 🎉");
}
else if (age < 18)
{
    Console.WriteLine("อายุไม่ถึง");
}
else if (!hasID)
{
    Console.WriteLine("ต้องแสดง ID");
}
else
{
    Console.WriteLine("ไม่มีตั๋ว");
}
```

---

## 5. Pattern Matching กับ if (C# 7+)

```csharp
object obj = "Hello, World!";

// is keyword + type pattern
if (obj is string str)
{
    Console.WriteLine($"เป็น string: {str.ToUpper()}");
}
else if (obj is int num)
{
    Console.WriteLine($"เป็น int: {num * 2}");
}
else if (obj is null)
{
    Console.WriteLine("เป็น null");
}

// Pattern กับ condition (C# 9+)
int value = 45;
if (value is > 0 and < 100)
{
    Console.WriteLine("อยู่ระหว่าง 0-100");
}

if (value is not (< 0 or > 100))
{
    Console.WriteLine("อยู่ระหว่าง 0-100 (วิธีอื่น)");
}

// Property Pattern (C# 8+)
var person = new { Name = "สมชาย", Age = 25, IsEmployed = true };
if (person is { Age: > 18, IsEmployed: true })
{
    Console.WriteLine($"{person.Name} มีงานทำและบรรลุนิติภาวะ");
}
```

---

## 6. Guard Clauses (Early Return) ✅

**Guard Clauses** คือ pattern การตรวจสอบเงื่อนไขที่ไม่ถูกต้องก่อน แล้ว return หรือ throw เลย ทำให้โค้ดอ่านง่ายขึ้น

```csharp
// ❌ แบบไม่ดี: โค้ดลึกมาก (Arrow Code)
void ProcessOrder_Bad(Order? order, User? user)
{
    if (order != null)
    {
        if (user != null)
        {
            if (user.IsActive)
            {
                if (order.Amount > 0)
                {
                    // logic จริงๆ อยู่ลึกมาก
                    Console.WriteLine("ประมวลผลสำเร็จ");
                }
                else
                {
                    Console.WriteLine("จำนวนเงินต้องมากกว่า 0");
                }
            }
            else
            {
                Console.WriteLine("User ไม่ active");
            }
        }
        else
        {
            Console.WriteLine("ไม่พบ User");
        }
    }
    else
    {
        Console.WriteLine("ไม่พบ Order");
    }
}

// ✅ แบบดี: Guard Clauses
void ProcessOrder(Order? order, User? user)
{
    if (order == null)
    {
        Console.WriteLine("ไม่พบ Order");
        return;
    }
    
    if (user == null)
    {
        Console.WriteLine("ไม่พบ User");
        return;
    }
    
    if (!user.IsActive)
    {
        Console.WriteLine("User ไม่ active");
        return;
    }
    
    if (order.Amount <= 0)
    {
        Console.WriteLine("จำนวนเงินต้องมากกว่า 0");
        return;
    }
    
    // logic จริงๆ อยู่ตรงนี้ ไม่ต้องลึกเยอะ
    Console.WriteLine("ประมวลผลสำเร็จ");
}

class Order { public decimal Amount { get; set; } }
class User { public bool IsActive { get; set; } }
```

---

## 7. โปรแกรมตัวอย่าง: ระบบ Login

```csharp
// โปรแกรมจำลองระบบ Login

const string VALID_USERNAME = "admin";
const string VALID_PASSWORD = "password123";
const int MAX_ATTEMPTS = 3;

int attempts = 0;
bool isLoggedIn = false;

Console.Clear();
Console.WriteLine("╔══════════════════════════════╗");
Console.WriteLine("║     ระบบเข้าสู่ระบบ          ║");
Console.WriteLine("╚══════════════════════════════╝\n");

while (attempts < MAX_ATTEMPTS && !isLoggedIn)
{
    Console.Write("Username: ");
    string username = Console.ReadLine() ?? "";
    
    Console.Write("Password: ");
    // จำลองการซ่อน password (ใน console จริงต้องใช้ ReadKey)
    string password = Console.ReadLine() ?? "";
    
    attempts++;
    
    if (string.IsNullOrWhiteSpace(username) || string.IsNullOrWhiteSpace(password))
    {
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine("⚠️  กรุณากรอก Username และ Password\n");
        Console.ResetColor();
        continue;
    }
    
    if (username == VALID_USERNAME && password == VALID_PASSWORD)
    {
        isLoggedIn = true;
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine("\n✅ เข้าสู่ระบบสำเร็จ!");
        Console.WriteLine($"ยินดีต้อนรับ, {username}!");
        Console.ResetColor();
    }
    else
    {
        int remaining = MAX_ATTEMPTS - attempts;
        Console.ForegroundColor = ConsoleColor.Red;
        
        if (remaining > 0)
        {
            Console.WriteLine($"\n❌ Username หรือ Password ไม่ถูกต้อง");
            Console.WriteLine($"เหลืออีก {remaining} ครั้ง\n");
        }
        else
        {
            Console.WriteLine("\n🔒 บัญชีถูกล็อก! กรุณาติดต่อผู้ดูแลระบบ");
        }
        
        Console.ResetColor();
    }
}

if (isLoggedIn)
{
    Console.WriteLine("\n--- เมนูหลัก ---");
    Console.WriteLine("1. ดูข้อมูล");
    Console.WriteLine("2. แก้ไขข้อมูล");
    Console.WriteLine("3. ออกจากระบบ");
}
```

---

## 8. โปรแกรมตัวอย่าง: ระบบภาษี

```csharp
// คำนวณภาษีเงินได้บุคคลธรรมดา (ไทย 2024)

Console.WriteLine("=== คำนวณภาษีเงินได้ ===");
Console.Write("รายได้ต่อปี (บาท): ");
if (!decimal.TryParse(Console.ReadLine(), out decimal income) || income < 0)
{
    Console.WriteLine("ข้อมูลไม่ถูกต้อง");
    return;
}

// คำนวณเงินได้สุทธิ (หักค่าใช้จ่าย 50% แต่ไม่เกิน 100,000)
decimal expenseDeduction = Math.Min(income * 0.5m, 100000m);
decimal personalDeduction = 60000m;  // ค่าลดหย่อนส่วนตัว
decimal netIncome = income - expenseDeduction - personalDeduction;

decimal tax = 0m;
string bracket = "";

if (netIncome <= 0)
{
    tax = 0m;
    bracket = "ไม่ต้องเสียภาษี";
}
else if (netIncome <= 150000m)
{
    tax = 0m;  // ได้รับยกเว้น
    bracket = "ยกเว้น (≤ 150,000)";
}
else if (netIncome <= 300000m)
{
    tax = (netIncome - 150000m) * 0.05m;
    bracket = "5% (150,001 - 300,000)";
}
else if (netIncome <= 500000m)
{
    tax = 7500m + (netIncome - 300000m) * 0.10m;
    bracket = "10% (300,001 - 500,000)";
}
else if (netIncome <= 750000m)
{
    tax = 27500m + (netIncome - 500000m) * 0.15m;
    bracket = "15% (500,001 - 750,000)";
}
else if (netIncome <= 1000000m)
{
    tax = 65000m + (netIncome - 750000m) * 0.20m;
    bracket = "20% (750,001 - 1,000,000)";
}
else if (netIncome <= 2000000m)
{
    tax = 115000m + (netIncome - 1000000m) * 0.25m;
    bracket = "25% (1,000,001 - 2,000,000)";
}
else if (netIncome <= 5000000m)
{
    tax = 365000m + (netIncome - 2000000m) * 0.30m;
    bracket = "30% (2,000,001 - 5,000,000)";
}
else
{
    tax = 1265000m + (netIncome - 5000000m) * 0.35m;
    bracket = "35% (> 5,000,000)";
}

decimal effectiveRate = income > 0 ? tax / income * 100 : 0;

Console.WriteLine("\n=== ผลการคำนวณ ===");
Console.WriteLine($"รายได้รวม       : {income:N2} บาท");
Console.WriteLine($"หักค่าใช้จ่าย   : {expenseDeduction:N2} บาท");
Console.WriteLine($"หักลดหย่อนส่วนตัว: {personalDeduction:N2} บาท");
Console.WriteLine($"เงินได้สุทธิ    : {netIncome:N2} บาท");
Console.WriteLine($"อัตราภาษี       : {bracket}");
Console.WriteLine($"ภาษีที่ต้องชำระ : {tax:N2} บาท");
Console.WriteLine($"อัตราภาษีจริง   : {effectiveRate:F2}%");
```

---

## 9. Best Practices

```csharp
// ✅ 1. ใช้ Guard Clauses แทน Deep Nesting
string ProcessName(string? name)
{
    if (name == null) return "Unknown";
    if (name.Length == 0) return "Empty";
    if (name.Length > 50) return name[..50]; // Trim
    return name.Trim();
}

// ✅ 2. เปรียบเทียบ Null อยู่ทางซ้าย
string result = null;
if (result == null) { } // ✅
if (null == result) { } // ✅ แต่อ่านยาก

// ✅ 3. หลีกเลี่ยงการเปรียบเทียบ bool กับ true/false
bool isValid = true;
if (isValid) { }      // ✅
if (isValid == true)  // ❌ ไม่จำเป็น
if (isValid == false) // ❌ ใช้ !isValid แทน
if (!isValid) { }     // ✅

// ✅ 4. ใช้ switch แทน if-else if ยาวๆ (ดูใน Part 006)
// ✅ 5. ตั้งชื่อ boolean ให้ชัดเจน
bool isUserActive = true;       // ✅
bool userStatus = true;          // ❌ ไม่ชัดว่า status คืออะไร
bool canUserAccessPage = true;   // ✅
```

---

## 10. Exercises

### Exercise 1: ตรวจสอบปีอธิกสุรทิน
```csharp
// ปีอธิกสุรทิน: หารด้วย 4 ลงตัว แต่ถ้าหารด้วย 100 ลงตัว
// ต้องหารด้วย 400 ลงตัวด้วย
Console.Write("ใส่ปี: ");
int year = int.Parse(Console.ReadLine() ?? "2024");

// TODO: ตรวจสอบว่าเป็นปีอธิกสุรทินหรือไม่
// ตัวอย่าง: 2024 = ✅, 1900 = ❌, 2000 = ✅
```

### Exercise 2: เครื่องวัด BMI
```csharp
Console.Write("น้ำหนัก (กก.): ");
double weight = double.Parse(Console.ReadLine() ?? "0");
Console.Write("ส่วนสูง (ซม.): ");
double height = double.Parse(Console.ReadLine() ?? "0");

// TODO: คำนวณ BMI และแสดงผล
// < 18.5 = น้ำหนักต่ำ
// 18.5-24.9 = ปกติ
// 25.0-29.9 = น้ำหนักเกิน
// >= 30 = อ้วน
// พร้อมแสดงสีตามผล
```

### Exercise 3: ระบบ ATM
```csharp
decimal balance = 10000m;

// TODO: สร้างระบบ ATM ง่ายๆ ที่:
// 1. แสดงยอดเงิน
// 2. ถอนเงิน (ตรวจสอบยอดพอ, ขั้นต่ำ 100, ทวีคูณของ 100)
// 3. ฝากเงิน (ขั้นต่ำ 100)
// 4. แสดง error ที่เหมาะสม
```

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ if, if-else, if-else if-else
- ✅ Nested if
- ✅ Pattern Matching กับ is keyword
- ✅ Guard Clauses pattern
- ✅ Best Practices สำหรับ conditionals

## Part ถัดไป
**[Part 006: คำสั่ง switch-case →](part-006.md)**

---

*Part 005/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
