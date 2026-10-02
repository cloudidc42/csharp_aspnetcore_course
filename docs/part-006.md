# Part 006: คำสั่ง switch-case และ switch Expression

## เนื้อหาใน Part นี้
- switch-case แบบดั้งเดิม
- switch Expression (C# 8+)
- Pattern Matching ใน switch
- When clause (guards)
- เปรียบเทียบ switch vs if-else
- Best Practices

---

## 1. switch-case แบบดั้งเดิม

```csharp
int dayNumber = 3;
string dayName;

switch (dayNumber)
{
    case 1:
        dayName = "วันจันทร์";
        break;
    case 2:
        dayName = "วันอังคาร";
        break;
    case 3:
        dayName = "วันพุธ";
        break;
    case 4:
        dayName = "วันพฤหัสบดี";
        break;
    case 5:
        dayName = "วันศุกร์";
        break;
    case 6:
        dayName = "วันเสาร์";
        break;
    case 7:
        dayName = "วันอาทิตย์";
        break;
    default:
        dayName = "ไม่รู้จัก";
        break;
}

Console.WriteLine($"วันที่ {dayNumber} = {dayName}");
```

### Fall-through: หลาย case ใช้โค้ดเดียวกัน

```csharp
int month = 4;
int days;

switch (month)
{
    case 1:
    case 3:
    case 5:
    case 7:
    case 8:
    case 10:
    case 12:
        days = 31;
        break;
    case 4:
    case 6:
    case 9:
    case 11:
        days = 30;
        break;
    case 2:
        days = 28; // ไม่คิดปีอธิกสุรทิน
        break;
    default:
        days = -1;
        break;
}

Console.WriteLine($"เดือน {month} มี {days} วัน");

// switch กับ string
string command = "quit";

switch (command.ToLower())
{
    case "start":
    case "begin":
        Console.WriteLine("เริ่มต้น...");
        break;
    case "stop":
    case "end":
    case "quit":
    case "exit":
        Console.WriteLine("หยุด...");
        break;
    default:
        Console.WriteLine($"คำสั่ง '{command}' ไม่รู้จัก");
        break;
}
```

---

## 2. switch Expression (C# 8+) ✅ แนะนำ

```csharp
// รูปแบบ: variable switch { pattern => value, ... }

int dayNumber = 3;
string dayName = dayNumber switch
{
    1 => "วันจันทร์",
    2 => "วันอังคาร",
    3 => "วันพุธ",
    4 => "วันพฤหัสบดี",
    5 => "วันศุกร์",
    6 => "วันเสาร์",
    7 => "วันอาทิตย์",
    _ => "ไม่รู้จัก"  // default case
};

Console.WriteLine($"วันที่ {dayNumber} = {dayName}");

// switch Expression กับหลาย case
int month = 6;
int daysInMonth = month switch
{
    1 or 3 or 5 or 7 or 8 or 10 or 12 => 31,
    4 or 6 or 9 or 11 => 30,
    2 => 28,
    _ => throw new ArgumentOutOfRangeException(nameof(month))
};

Console.WriteLine($"เดือน {month} มี {daysInMonth} วัน");

// ใช้ใน method
static string GetSeason(int month) => month switch
{
    3 or 4 or 5 => "ฤดูร้อน",
    6 or 7 or 8 or 9 or 10 => "ฤดูฝน",
    11 or 12 or 1 or 2 => "ฤดูหนาว",
    _ => "ไม่รู้จัก"
};

Console.WriteLine(GetSeason(7));  // ฤดูฝน
```

---

## 3. Pattern Matching ใน switch

### Type Pattern

```csharp
object[] items = { 42, "hello", 3.14, true, null };

foreach (object item in items)
{
    string description = item switch
    {
        int i => $"จำนวนเต็ม: {i}",
        string s => $"ข้อความ: {s} (ยาว {s.Length})",
        double d => $"ทศนิยม: {d:F2}",
        bool b => $"Boolean: {b}",
        null => "เป็น null",
        _ => $"ชนิดอื่น: {item.GetType().Name}"
    };
    
    Console.WriteLine(description);
}
```

### Relational Pattern (C# 9+)

```csharp
int score = 75;
string grade = score switch
{
    >= 90 => "A",
    >= 80 => "B",
    >= 70 => "C",
    >= 60 => "D",
    < 60 => "F",
    _ => "N/A"
};

Console.WriteLine($"คะแนน {score} = {grade}");

// ตัวอย่างจริง: ค่าไฟฟ้า
int units = 350;
decimal electricBill = units switch
{
    <= 15 => units * 2.3484m,
    <= 25 => 15 * 2.3484m + (units - 15) * 2.9882m,
    <= 35 => 15 * 2.3484m + 10 * 2.9882m + (units - 25) * 3.2405m,
    <= 100 => 15 * 2.3484m + 10 * 2.9882m + 10 * 3.2405m + (units - 35) * 3.6237m,
    _ => 15 * 2.3484m + 10 * 2.9882m + 10 * 3.2405m + 65 * 3.6237m + (units - 100) * 3.7171m
};

Console.WriteLine($"หน่วยที่ใช้: {units} หน่วย");
Console.WriteLine($"ค่าไฟ: {electricBill:N2} บาท");
```

### Property Pattern (C# 8+)

```csharp
record Point(int X, int Y);

Point p = new(3, 4);
string quadrant = p switch
{
    { X: > 0, Y: > 0 } => "Quadrant I (+,+)",
    { X: < 0, Y: > 0 } => "Quadrant II (-,+)",
    { X: < 0, Y: < 0 } => "Quadrant III (-,-)",
    { X: > 0, Y: < 0 } => "Quadrant IV (+,-)",
    { X: 0, Y: 0 }     => "Origin (0,0)",
    { X: 0 }           => "Y-axis",
    { Y: 0 }           => "X-axis",
    _                  => "Unknown"
};

Console.WriteLine($"Point ({p.X}, {p.Y}) อยู่ใน {quadrant}");

// Property Pattern กับ class
class Order
{
    public string Status { get; set; } = "";
    public decimal Amount { get; set; }
    public bool IsPaid { get; set; }
}

var order = new Order { Status = "Pending", Amount = 1500m, IsPaid = false };

string action = order switch
{
    { Status: "Cancelled" } => "Order ถูกยกเลิก",
    { IsPaid: false, Amount: > 1000m } => "ต้องชำระเงินก่อน (จำนวนมาก)",
    { IsPaid: false } => "ต้องชำระเงิน",
    { Status: "Pending", IsPaid: true } => "รอยืนยัน",
    { Status: "Processing" } => "กำลังดำเนินการ",
    { Status: "Shipped" } => "จัดส่งแล้ว",
    _ => "ไม่ทราบสถานะ"
};

Console.WriteLine($"Order: {action}");
```

### Tuple Pattern

```csharp
// switch กับหลาย values พร้อมกัน
string GetTrafficLight(string state, int hour) => (state, hour) switch
{
    ("red", _) => "🔴 หยุด",
    ("yellow", _) => "🟡 ระวัง",
    ("green", < 22) => "🟢 ไปได้",
    ("green", >= 22) => "🟢 ไปได้ (กลางคืน ระวังด้วย)",
    _ => "❓ ไม่ทราบ"
};

Console.WriteLine(GetTrafficLight("green", 20));  // ✅
Console.WriteLine(GetTrafficLight("red", 8));

// Rock Paper Scissors
string PlayGame(string player1, string player2) => (player1, player2) switch
{
    ("rock", "scissors") or ("scissors", "paper") or ("paper", "rock") => "Player 1 ชนะ!",
    ("scissors", "rock") or ("paper", "scissors") or ("rock", "paper") => "Player 2 ชนะ!",
    _ => "เสมอ!"
};

Console.WriteLine(PlayGame("rock", "scissors"));  // Player 1 ชนะ!
Console.WriteLine(PlayGame("paper", "paper"));    // เสมอ!
```

---

## 4. when clause (Guards)

```csharp
// when: เพิ่มเงื่อนไขเพิ่มเติม
object value = 42;

switch (value)
{
    case int n when n < 0:
        Console.WriteLine($"{n} เป็นจำนวนลบ");
        break;
    case int n when n == 0:
        Console.WriteLine("เป็น 0");
        break;
    case int n when n > 0 && n <= 100:
        Console.WriteLine($"{n} อยู่ระหว่าง 1-100");
        break;
    case int n:
        Console.WriteLine($"{n} มากกว่า 100");
        break;
    case string s when s.Length > 10:
        Console.WriteLine($"string ยาว: {s[..10]}...");
        break;
    case string s:
        Console.WriteLine($"string: {s}");
        break;
}

// switch Expression กับ when (ใช้ and/or แทน)
int number = 42;
string description = number switch
{
    < 0 => "ลบ",
    0 => "ศูนย์",
    > 0 and <= 10 => "เล็ก (1-10)",
    > 10 and <= 100 => "กลาง (11-100)",
    _ => "ใหญ่ (>100)"
};
```

---

## 5. switch กับ Enum

```csharp
enum Season { Spring, Summer, Autumn, Winter }
enum DayOfWeek { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday }

Season season = Season.Summer;

// switch statement
switch (season)
{
    case Season.Spring:
        Console.WriteLine("🌸 ฤดูใบไม้ผลิ");
        break;
    case Season.Summer:
        Console.WriteLine("☀️ ฤดูร้อน");
        break;
    case Season.Autumn:
        Console.WriteLine("🍂 ฤดูใบไม้ร่วง");
        break;
    case Season.Winter:
        Console.WriteLine("❄️ ฤดูหนาว");
        break;
}

// switch Expression (สะอาดกว่า)
string seasonEmoji = season switch
{
    Season.Spring => "🌸",
    Season.Summer => "☀️",
    Season.Autumn => "🍂",
    Season.Winter => "❄️",
    _ => "🌍"
};

Console.WriteLine($"ฤดู: {seasonEmoji} {season}");

// ตัวอย่างจริง: เงินเดือน overtime
DayOfWeek workDay = DayOfWeek.Saturday;
double baseOvertimeRate = 500; // บาทต่อชั่วโมง

double overtimeRate = workDay switch
{
    DayOfWeek.Monday or DayOfWeek.Tuesday or DayOfWeek.Wednesday 
        or DayOfWeek.Thursday or DayOfWeek.Friday => baseOvertimeRate * 1.5,
    DayOfWeek.Saturday => baseOvertimeRate * 2.0,
    DayOfWeek.Sunday => baseOvertimeRate * 3.0,
    _ => baseOvertimeRate
};

Console.WriteLine($"ค่า OT วัน {workDay}: {overtimeRate:N0} บาท/ชม.");
```

---

## 6. โปรแกรมตัวอย่างสมบูรณ์: เครื่องคิดเลข

```csharp
// เครื่องคิดเลขพื้นฐาน

Console.Clear();
Console.WriteLine("╔════════════════════════╗");
Console.WriteLine("║    เครื่องคิดเลข        ║");
Console.WriteLine("╚════════════════════════╝");

while (true)
{
    Console.Write("\nใส่ตัวเลขที่ 1: ");
    if (!double.TryParse(Console.ReadLine(), out double num1))
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine("ข้อมูลไม่ถูกต้อง");
        Console.ResetColor();
        continue;
    }
    
    Console.Write("เลือก operator (+, -, *, /, %, ^, q=ออก): ");
    string op = Console.ReadLine()?.Trim() ?? "";
    
    if (op == "q" || op == "quit")
    {
        Console.WriteLine("ขอบคุณที่ใช้บริการ!");
        break;
    }
    
    Console.Write("ใส่ตัวเลขที่ 2: ");
    if (!double.TryParse(Console.ReadLine(), out double num2))
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine("ข้อมูลไม่ถูกต้อง");
        Console.ResetColor();
        continue;
    }
    
    // คำนวณ
    double result;
    bool isValid = true;
    
    switch (op)
    {
        case "+":
            result = num1 + num2;
            break;
        case "-":
            result = num1 - num2;
            break;
        case "*":
        case "x":
            result = num1 * num2;
            break;
        case "/":
            if (num2 == 0)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("❌ ไม่สามารถหารด้วย 0 ได้!");
                Console.ResetColor();
                isValid = false;
                result = 0;
            }
            else
            {
                result = num1 / num2;
            }
            break;
        case "%":
            if (num2 == 0)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("❌ ไม่สามารถหารด้วย 0 ได้!");
                Console.ResetColor();
                isValid = false;
                result = 0;
            }
            else
            {
                result = num1 % num2;
            }
            break;
        case "^":
        case "**":
            result = Math.Pow(num1, num2);
            break;
        default:
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine($"❓ ไม่รู้จัก operator '{op}'");
            Console.ResetColor();
            isValid = false;
            result = 0;
            break;
    }
    
    if (isValid)
    {
        Console.ForegroundColor = ConsoleColor.Green;
        string operatorSymbol = op == "**" ? "^" : op;
        Console.WriteLine($"\n✅ {num1} {operatorSymbol} {num2} = {result:G}");
        Console.ResetColor();
    }
}
```

---

## 7. เปรียบเทียบ switch vs if-else

```csharp
// เมื่อไหร่ใช้ switch:
// ✅ เปรียบเทียบค่าเดียวกับหลายค่า
// ✅ ใช้ Enum
// ✅ Pattern matching
// ✅ Code ดูสะอาดกว่า

// เมื่อไหร่ใช้ if-else:
// ✅ เงื่อนไขซับซ้อน (หลาย variables)
// ✅ Range เช่น >= 80 && < 90
// ✅ Boolean expressions
// ✅ Short-circuit evaluation สำคัญ

// ❌ if-else ยาวๆ เปรียบเทียบค่าเดียวกัน
if (status == "Active") {}
else if (status == "Inactive") {}
else if (status == "Pending") {}
else if (status == "Cancelled") {}

// ✅ ควรใช้ switch
string message = status switch
{
    "Active" => "ใช้งานได้",
    "Inactive" => "ไม่ใช้งาน",
    "Pending" => "รอการยืนยัน",
    "Cancelled" => "ยกเลิกแล้ว",
    _ => "ไม่ทราบสถานะ"
};

string status = "Active";
```

---

## 8. Exercises

### Exercise 1: เครื่องแปลงหน่วย
```csharp
// สร้างโปรแกรมแปลงหน่วย
// เมนู: 1=km→miles, 2=kg→lbs, 3=°C→°F, 4=ออก
// ใช้ switch Expression
```

### Exercise 2: เกม สุ่มเลข
```csharp
// เกมทายตัวเลข 1-10
// บอก: สูงเกิน / ต่ำเกิน / ถูกต้อง
// ใช้ switch กับ comparison results
// นับจำนวนครั้งที่ทาย
```

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ switch-case แบบดั้งเดิม
- ✅ switch Expression (C# 8+) - รูปแบบใหม่ที่แนะนำ
- ✅ Pattern Matching: Type, Relational, Property, Tuple
- ✅ when clause
- ✅ switch กับ Enum
- ✅ เมื่อไหร่ควรใช้ switch vs if-else

## Part ถัดไป
**[Part 007: วนซ้ำด้วย for loop →](part-007.md)**

---

*Part 006/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
