# Part 008: วนซ้ำด้วย while และ do-while

## เนื้อหาใน Part นี้
- while loop
- do-while loop
- เปรียบเทียบ for vs while vs do-while
- โปรแกรมตัวอย่าง

---

## 1. while loop

```csharp
// รูปแบบ: while (เงื่อนไข) { โค้ด }
// ตรวจสอบเงื่อนไขก่อน ถ้าเป็น false จะไม่ทำเลย

int count = 1;
while (count <= 5)
{
    Console.WriteLine($"count = {count}");
    count++;
}

// ⚠️ ระวัง Infinite Loop!
// int x = 1;
// while (x > 0)  // เงื่อนไขเป็น true ตลอด
// {
//     x++;  // x เพิ่มขึ้นเรื่อยๆ
// }

// while กับเงื่อนไขซับซ้อน
int num = 1024;
int steps = 0;
Console.Write($"หาร {num} ด้วย 2 ซ้ำๆ: ");
while (num > 1)
{
    num /= 2;
    steps++;
    Console.Write($"{num} ");
}
Console.WriteLine($"\nทำ {steps} ครั้ง");
```

### while สำหรับ User Input

```csharp
// รับ input จนกว่าจะถูกต้อง
int age = -1;
while (age < 0 || age > 150)
{
    Console.Write("ใส่อายุ (0-150): ");
    if (!int.TryParse(Console.ReadLine(), out age))
        age = -1;
    
    if (age < 0 || age > 150)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine("อายุไม่ถูกต้อง");
        Console.ResetColor();
    }
}
Console.WriteLine($"อายุของคุณคือ {age} ปี");
```

---

## 2. do-while loop

```csharp
// รูปแบบ: do { โค้ด } while (เงื่อนไข);
// ทำโค้ดก่อน แล้วค่อยตรวจสอบเงื่อนไข
// ทำอย่างน้อย 1 ครั้งเสมอ!

int n = 1;
do
{
    Console.WriteLine($"n = {n}");
    n++;
} while (n <= 5);

// ตัวอย่างที่ดีของ do-while: Menu
string choice;
do
{
    Console.WriteLine("\n=== เมนู ===");
    Console.WriteLine("1. ตัวเลือกที่ 1");
    Console.WriteLine("2. ตัวเลือกที่ 2");
    Console.WriteLine("0. ออก");
    Console.Write("เลือก: ");
    choice = Console.ReadLine() ?? "";
    
    switch (choice)
    {
        case "1":
            Console.WriteLine("คุณเลือก 1");
            break;
        case "2":
            Console.WriteLine("คุณเลือก 2");
            break;
        case "0":
            Console.WriteLine("ลาก่อน!");
            break;
        default:
            Console.WriteLine("เลือกไม่ถูกต้อง");
            break;
    }
} while (choice != "0");
```

---

## 3. เปรียบเทียบ for vs while vs do-while

```
╔══════════════╦═══════════════════════════════════════════╗
║  loop ชนิด  ║  ใช้เมื่อ                                ║
╠══════════════╬═══════════════════════════════════════════╣
║  for         ║ รู้จำนวนรอบที่แน่นอน เช่น 1-10          ║
║  while       ║ ไม่รู้จำนวนรอบ ตรวจเงื่อนไขก่อน         ║
║  do-while    ║ ต้องทำอย่างน้อย 1 ครั้ง เช่น เมนู       ║
╚══════════════╩═══════════════════════════════════════════╝
```

```csharp
// ทั้ง 3 แบบ เหมือนกัน
// for
for (int i = 0; i < 5; i++)
    Console.Write($"{i} ");

// while
int i2 = 0;
while (i2 < 5)
{
    Console.Write($"{i2} ");
    i2++;
}

// do-while
int i3 = 0;
do
{
    Console.Write($"{i3} ");
    i3++;
} while (i3 < 5);
```

---

## 4. โปรแกรมตัวอย่าง: คิดเงินสด

```csharp
// ระบบคิดเงินสด: แบงก์ใหญ่ก่อน
decimal change = 0;
Console.Write("เงินทอน (บาท): ");
if (decimal.TryParse(Console.ReadLine(), out decimal amount) && amount >= 0)
{
    change = amount;
}

decimal[] denominations = { 1000, 500, 100, 50, 20, 10, 5, 2, 1, 0.50m, 0.25m };
string[] names = { "1000", "500", "100", "50", "20", "10", "5", "2", "1", "50สต", "25สต" };

Console.WriteLine($"\nเงินทอน {change:N2} บาท:");
Console.WriteLine(new string('-', 30));

for (int i = 0; i < denominations.Length; i++)
{
    if (change <= 0) break;
    
    int count = (int)(change / denominations[i]);
    if (count > 0)
    {
        Console.WriteLine($"  แบงก์/เหรียญ {names[i],6}: {count,3} ใบ/เหรียญ");
        change -= count * denominations[i];
        change = Math.Round(change, 2); // แก้ floating point error
    }
}
```

---

## 5. โปรแกรมตัวอย่าง: ATM จำลอง

```csharp
decimal balance = 15750.00m;
string pin = "1234";
bool authenticated = false;
int pinAttempts = 0;
const int MAX_PIN_ATTEMPTS = 3;

Console.Clear();
Console.WriteLine("╔═══════════════════╗");
Console.WriteLine("║    🏧 ATM          ║");
Console.WriteLine("╚═══════════════════╝");

// ตรวจสอบ PIN
while (pinAttempts < MAX_PIN_ATTEMPTS)
{
    Console.Write("ใส่ PIN (4 หลัก): ");
    string inputPin = Console.ReadLine() ?? "";
    pinAttempts++;
    
    if (inputPin == pin)
    {
        authenticated = true;
        break;
    }
    
    int remaining = MAX_PIN_ATTEMPTS - pinAttempts;
    if (remaining > 0)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"PIN ไม่ถูกต้อง เหลือ {remaining} ครั้ง");
        Console.ResetColor();
    }
}

if (!authenticated)
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine("🔒 บัตรถูกล็อก!");
    Console.ResetColor();
    return;
}

// Main ATM loop
bool running = true;
do
{
    Console.Clear();
    Console.WriteLine("╔═══════════════════════════╗");
    Console.WriteLine("║    🏧 ATM Menu             ║");
    Console.WriteLine("╠═══════════════════════════╣");
    Console.WriteLine($"║ ยอดเงิน: {balance,14:N2} ║");
    Console.WriteLine("╠═══════════════════════════╣");
    Console.WriteLine("║  1. ถอนเงิน               ║");
    Console.WriteLine("║  2. ฝากเงิน               ║");
    Console.WriteLine("║  3. ดูยอดเงิน              ║");
    Console.WriteLine("║  0. ออก                   ║");
    Console.WriteLine("╚═══════════════════════════╝");
    Console.Write("เลือก: ");
    
    string choice = Console.ReadLine() ?? "";
    
    switch (choice)
    {
        case "1": // ถอนเงิน
            Console.Write("จำนวนที่ต้องการถอน: ");
            if (decimal.TryParse(Console.ReadLine(), out decimal withdraw) && withdraw > 0)
            {
                if (withdraw > balance)
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine("❌ ยอดเงินไม่พอ");
                    Console.ResetColor();
                }
                else if (withdraw % 100 != 0)
                {
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    Console.WriteLine("⚠️  ถอนได้เฉพาะทวีคูณของ 100");
                    Console.ResetColor();
                }
                else
                {
                    balance -= withdraw;
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine($"✅ ถอนสำเร็จ {withdraw:N2} บาท");
                    Console.WriteLine($"   ยอดคงเหลือ: {balance:N2} บาท");
                    Console.ResetColor();
                }
            }
            break;
            
        case "2": // ฝากเงิน
            Console.Write("จำนวนที่ต้องการฝาก: ");
            if (decimal.TryParse(Console.ReadLine(), out decimal deposit) && deposit > 0)
            {
                balance += deposit;
                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine($"✅ ฝากสำเร็จ {deposit:N2} บาท");
                Console.WriteLine($"   ยอดคงเหลือ: {balance:N2} บาท");
                Console.ResetColor();
            }
            break;
            
        case "3": // ดูยอดเงิน
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine($"\nยอดเงินคงเหลือ: {balance:N2} บาท");
            Console.ResetColor();
            break;
            
        case "0":
            running = false;
            Console.WriteLine("\nขอบคุณที่ใช้บริการ");
            break;
            
        default:
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine("เมนูไม่ถูกต้อง");
            Console.ResetColor();
            break;
    }
    
    if (running) 
    {
        Console.Write("\nกด Enter เพื่อดำเนินการต่อ...");
        Console.ReadLine();
    }
    
} while (running);
```

---

## 6. Exercises

### Exercise 1: ทายตัวเลข (while version)
```csharp
// ใช้ while loop สร้างเกมทายตัวเลข
// - ทายจนกว่าจะถูก
// - บอกสูง/ต่ำ
// - นับจำนวนครั้ง
```

### Exercise 2: ตรวจสอบรหัสผ่าน
```csharp
// ใช้ do-while รับรหัสผ่านจนกว่าจะผ่านเงื่อนไข:
// - อย่างน้อย 8 ตัวอักษร
// - มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
// - มีตัวเลขอย่างน้อย 1 ตัว
```

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ while loop: ตรวจสอบก่อน → อาจไม่ทำเลย
- ✅ do-while loop: ทำก่อน → ตรวจสอบทีหลัง → ทำอย่างน้อย 1 ครั้ง
- ✅ เมื่อไหร่ควรใช้ loop แต่ละชนิด
- ✅ โปรแกรม Menu-driven และ ATM

## Part ถัดไป
**[Part 009: foreach loop และ Arrays →](part-009.md)**

---

*Part 008/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
