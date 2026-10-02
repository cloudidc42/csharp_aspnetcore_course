# Part 004: การรับและแสดงผลข้อมูล (Input/Output)

## เนื้อหาใน Part นี้
- Console Output (Console.Write/WriteLine)
- String Formatting รูปแบบต่างๆ
- Console Input (Console.ReadLine/ReadKey)
- การตรวจสอบข้อมูล Input
- การแสดงผลแบบต่างๆ (ตาราง, สี)
- Environment Variables
- Debug Output

---

## 1. Console Output

### Console.Write vs Console.WriteLine

```csharp
// WriteLine: แสดงผลและขึ้นบรรทัดใหม่
Console.WriteLine("สวัสดีครับ");
Console.WriteLine("ยินดีต้อนรับ");

// Write: แสดงผลโดยไม่ขึ้นบรรทัดใหม่
Console.Write("A");
Console.Write("B");
Console.Write("C");
Console.WriteLine(); // ขึ้นบรรทัดใหม่

// ผลลัพธ์:
// สวัสดีครับ
// ยินดีต้อนรับ
// ABC
```

### รูปแบบการแสดงผล (Format Strings)

```csharp
string name = "สมชาย";
int age = 25;
double salary = 35750.50;
DateTime date = new DateTime(2024, 12, 25);

// 1. String Concatenation (ไม่แนะนำ - อ่านยาก)
Console.WriteLine("ชื่อ: " + name + " อายุ: " + age);

// 2. Composite Formatting: {index[,width][:format]}
Console.WriteLine("ชื่อ: {0}, อายุ: {1} ปี", name, age);
Console.WriteLine("เงินเดือน: {0:N2} บาท", salary);

// 3. String Interpolation (แนะนำ - อ่านง่าย)
Console.WriteLine($"ชื่อ: {name}, อายุ: {age} ปี");
Console.WriteLine($"เงินเดือน: {salary:N2} บาท");
Console.WriteLine($"วันที่: {date:dd/MM/yyyy}");

// 4. Verbatim String (@)
string path = @"C:\Users\Desktop\file.txt";
Console.WriteLine(path);

// 5. Raw String Literals (C# 11+)
string sql = """
    SELECT *
    FROM Users
    WHERE Age > 18
    """;
Console.WriteLine(sql);
```

### Format Specifiers (ตัวระบุรูปแบบ)

```csharp
double number = 1234567.8912;
int integer = 42;

// ตัวเลข
Console.WriteLine($"{number:N0}");    // 1,234,568         (N = Number, 0 ทศนิยม)
Console.WriteLine($"{number:N2}");    // 1,234,567.89       (N2 = 2 ทศนิยม)
Console.WriteLine($"{number:F4}");    // 1234567.8912       (F = Fixed-point)
Console.WriteLine($"{number:E2}");    // 1.23E+006           (E = Exponential)
Console.WriteLine($"{number:G}");     // 1234567.8912        (G = General)
Console.WriteLine($"{integer:D8}");   // 00000042            (D = Decimal, pad zeros)

// สกุลเงิน
Console.WriteLine($"{number:C}");     // ฿1,234,567.89 หรือ $1,234,567.89 (ตาม locale)
Console.WriteLine($"{number:C2}");    // ฿1,234,567.89

// เปอร์เซ็นต์
double percent = 0.0875;
Console.WriteLine($"{percent:P}");    // 8.75 %
Console.WriteLine($"{percent:P0}");   // 9 %
Console.WriteLine($"{percent:P2}");   // 8.75 %

// เลขฐาน 16
Console.WriteLine($"{255:X}");        // FF (Hexadecimal)
Console.WriteLine($"{255:X4}");       // 00FF (pad zeros)
Console.WriteLine($"{255:x4}");       // 00ff (lowercase)

// วันที่และเวลา
DateTime now = DateTime.Now;
Console.WriteLine($"{now:d}");         // 15/01/2024
Console.WriteLine($"{now:D}");         // 15 January 2024
Console.WriteLine($"{now:t}");         // 14:30
Console.WriteLine($"{now:T}");         // 14:30:00
Console.WriteLine($"{now:f}");         // 15 January 2024 14:30
Console.WriteLine($"{now:F}");         // 15 January 2024 14:30:00
Console.WriteLine($"{now:g}");         // 15/01/2024 14:30
Console.WriteLine($"{now:G}");         // 15/01/2024 14:30:00
Console.WriteLine($"{now:R}");         // Mon, 15 Jan 2024 14:30:00 GMT
Console.WriteLine($"{now:s}");         // 2024-01-15T14:30:00 (ISO 8601)
Console.WriteLine($"{now:yyyy-MM-dd HH:mm:ss}"); // Custom format
Console.WriteLine($"{now:dd/MM/yyyy HH:mm}");    // 15/01/2024 14:30
```

### Alignment และ Padding

```csharp
// {value,width} - จัดตำแหน่ง
// บวก = จัดขวา (right-align), ลบ = จัดซ้าย (left-align)

string[] names = { "สมชาย", "สมหญิง ใจดี", "วรา" };
int[] ages = { 25, 30, 28 };

Console.WriteLine($"{"ชื่อ",-15} {"อายุ",5}");
Console.WriteLine(new string('-', 20));
for (int i = 0; i < names.Length; i++)
{
    Console.WriteLine($"{names[i],-15} {ages[i],5}");
}

// ผลลัพธ์:
// ชื่อ               อายุ
// --------------------
// สมชาย              25
// สมหญิง ใจดี        30
// วรา                28

// PadLeft และ PadRight
string num = "42";
Console.WriteLine(num.PadLeft(8));          // "      42"
Console.WriteLine(num.PadLeft(8, '0'));     // "00000042"
Console.WriteLine(num.PadRight(8));         // "42      "
Console.WriteLine(num.PadRight(8, '-'));    // "42------"
```

---

## 2. Console Input

### Console.ReadLine - อ่านบรรทัด

```csharp
// อ่าน string จาก user
Console.Write("ใส่ชื่อ: ");
string name = Console.ReadLine(); // รอ user กด Enter

// ⚠️ ReadLine() คืนค่า string? (อาจเป็น null ใน .NET 8+)
// ใช้ ?? เพื่อป้องกัน null
string name2 = Console.ReadLine() ?? "";
Console.WriteLine($"สวัสดี {name2}!");

// แปลงเป็นชนิดอื่น
Console.Write("ใส่อายุ: ");
string ageInput = Console.ReadLine() ?? "0";
int age = int.Parse(ageInput);  // อันตราย! ถ้า input ไม่ใช่ตัวเลข
```

### Console.ReadKey - อ่านตัวอักษรเดียว

```csharp
// อ่านตัวอักษรเดียวโดยไม่ต้องกด Enter
Console.Write("กด Y เพื่อยืนยัน หรือ N เพื่อยกเลิก: ");
ConsoleKeyInfo key = Console.ReadKey();
Console.WriteLine(); // ขึ้นบรรทัดใหม่

if (key.Key == ConsoleKey.Y)
{
    Console.WriteLine("ยืนยันแล้ว!");
}
else if (key.Key == ConsoleKey.N)
{
    Console.WriteLine("ยกเลิกแล้ว");
}

// อ่านโดยไม่แสดงตัวอักษร (เช่น password)
Console.Write("ใส่รหัสผ่าน: ");
ConsoleKeyInfo passKey = Console.ReadKey(intercept: true); // ไม่แสดงบนหน้าจอ
```

### Console.Read - อ่าน 1 character

```csharp
Console.Write("กดปุ่มใดก็ได้: ");
int charCode = Console.Read();  // ได้ int (ASCII code)
char c = (char)charCode;
Console.WriteLine($"\nคุณกด: {c} (ASCII: {charCode})");
```

---

## 3. การตรวจสอบ Input (Validation)

### TryParse Pattern

```csharp
// วิธีที่ดีที่สุดในการรับ input จาก user
static int ReadInt(string prompt, int min = int.MinValue, int max = int.MaxValue)
{
    while (true)
    {
        Console.Write(prompt);
        string input = Console.ReadLine() ?? "";
        
        if (int.TryParse(input, out int value))
        {
            if (value >= min && value <= max)
                return value;
            Console.WriteLine($"กรุณาใส่ตัวเลขระหว่าง {min} ถึง {max}");
        }
        else
        {
            Console.WriteLine("กรุณาใส่ตัวเลขจำนวนเต็มเท่านั้น");
        }
    }
}

static double ReadDouble(string prompt)
{
    while (true)
    {
        Console.Write(prompt);
        string input = Console.ReadLine() ?? "";
        
        if (double.TryParse(input, out double value))
            return value;
        
        Console.WriteLine("กรุณาใส่ตัวเลขทศนิยมเท่านั้น");
    }
}

static string ReadString(string prompt, bool allowEmpty = false)
{
    while (true)
    {
        Console.Write(prompt);
        string input = Console.ReadLine() ?? "";
        
        if (allowEmpty || !string.IsNullOrWhiteSpace(input))
            return input;
        
        Console.WriteLine("กรุณาใส่ข้อมูล");
    }
}

// การใช้งาน
int age = ReadInt("ใส่อายุ (1-120): ", min: 1, max: 120);
double price = ReadDouble("ใส่ราคา: ");
string name = ReadString("ใส่ชื่อ: ");
```

### โปรแกรมตัวอย่าง: รับข้อมูลนักเรียน

```csharp
// โปรแกรมรับข้อมูลและแสดงผล
Console.Clear();
Console.WriteLine("╔═══════════════════════════════════╗");
Console.WriteLine("║   ระบบบันทึกข้อมูลนักเรียน       ║");
Console.WriteLine("╚═══════════════════════════════════╝\n");

// รับข้อมูล
Console.Write("ชื่อ: ");
string name = Console.ReadLine() ?? "ไม่ระบุ";

Console.Write("นามสกุล: ");
string lastName = Console.ReadLine() ?? "ไม่ระบุ";

int age;
while (true)
{
    Console.Write("อายุ (1-100): ");
    if (int.TryParse(Console.ReadLine(), out age) && age >= 1 && age <= 100)
        break;
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine("❌ กรุณาใส่อายุที่ถูกต้อง");
    Console.ResetColor();
}

double gpa;
while (true)
{
    Console.Write("เกรดเฉลี่ย (0.00-4.00): ");
    if (double.TryParse(Console.ReadLine(), out gpa) && gpa >= 0 && gpa <= 4)
        break;
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine("❌ กรุณาใส่เกรดเฉลี่ยที่ถูกต้อง");
    Console.ResetColor();
}

Console.Write("ภาควิชา: ");
string department = Console.ReadLine() ?? "ไม่ระบุ";

// แสดงผล
Console.WriteLine("\n");
Console.ForegroundColor = ConsoleColor.Green;
Console.WriteLine("✅ บันทึกข้อมูลสำเร็จ!");
Console.ResetColor();

Console.WriteLine("\n╔═══════════════════════════════════╗");
Console.WriteLine("║          ข้อมูลนักเรียน           ║");
Console.WriteLine("╠═══════════════════════════════════╣");
Console.WriteLine($"║ ชื่อ-นามสกุล: {name + " " + lastName,-20}║");
Console.WriteLine($"║ อายุ        : {age,-20}║");
Console.WriteLine($"║ เกรดเฉลี่ย  : {gpa,-20:F2}║");
Console.WriteLine($"║ ภาควิชา     : {department,-20}║");
Console.WriteLine("╚═══════════════════════════════════╝");
```

---

## 4. Console Styling

### สีใน Console

```csharp
// เปลี่ยนสีข้อความ
Console.ForegroundColor = ConsoleColor.Red;
Console.WriteLine("ข้อความสีแดง");

Console.ForegroundColor = ConsoleColor.Green;
Console.WriteLine("ข้อความสีเขียว");

Console.ForegroundColor = ConsoleColor.Yellow;
Console.WriteLine("ข้อความสีเหลือง");

// เปลี่ยนสีพื้นหลัง
Console.BackgroundColor = ConsoleColor.Blue;
Console.ForegroundColor = ConsoleColor.White;
Console.WriteLine("ข้อความขาวพื้นน้ำเงิน");

// Reset สีกลับปกติ
Console.ResetColor();
Console.WriteLine("กลับสีปกติ");

// สีทั้งหมดที่ใช้ได้
ConsoleColor[] colors = (ConsoleColor[])Enum.GetValues(typeof(ConsoleColor));
foreach (var color in colors)
{
    Console.ForegroundColor = color;
    Console.WriteLine($"▉ สี {color}");
}
Console.ResetColor();
```

### ตัวช่วย Helper Class

```csharp
static class ConsoleHelper
{
    public static void WriteSuccess(string message)
    {
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine($"✅ {message}");
        Console.ResetColor();
    }
    
    public static void WriteError(string message)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine($"❌ {message}");
        Console.ResetColor();
    }
    
    public static void WriteWarning(string message)
    {
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine($"⚠️  {message}");
        Console.ResetColor();
    }
    
    public static void WriteInfo(string message)
    {
        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine($"ℹ️  {message}");
        Console.ResetColor();
    }
    
    public static void WriteHeader(string title)
    {
        string border = new string('═', title.Length + 4);
        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine($"╔{border}╗");
        Console.WriteLine($"║  {title}  ║");
        Console.WriteLine($"╚{border}╝");
        Console.ResetColor();
    }
    
    public static void WriteTable(string[] headers, string[][] rows)
    {
        // คำนวณความกว้างคอลัมน์
        int[] widths = new int[headers.Length];
        for (int i = 0; i < headers.Length; i++)
        {
            widths[i] = headers[i].Length;
            foreach (var row in rows)
            {
                if (i < row.Length)
                    widths[i] = Math.Max(widths[i], row[i].Length);
            }
        }
        
        // สร้าง separator
        string separator = "+" + string.Join("+", Array.ConvertAll(widths, w => new string('-', w + 2))) + "+";
        
        // Header
        Console.WriteLine(separator);
        Console.Write("|");
        for (int i = 0; i < headers.Length; i++)
        {
            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.Write($" {headers[i].PadRight(widths[i])} |");
            Console.ResetColor();
        }
        Console.WriteLine();
        Console.WriteLine(separator);
        
        // Rows
        foreach (var row in rows)
        {
            Console.Write("|");
            for (int i = 0; i < headers.Length; i++)
            {
                string cell = i < row.Length ? row[i] : "";
                Console.Write($" {cell.PadRight(widths[i])} |");
            }
            Console.WriteLine();
        }
        Console.WriteLine(separator);
    }
    
    public static void Pause(string message = "กด Enter เพื่อดำเนินการต่อ...")
    {
        Console.ForegroundColor = ConsoleColor.DarkGray;
        Console.Write($"\n{message}");
        Console.ResetColor();
        Console.ReadLine();
    }
}

// การใช้งาน
ConsoleHelper.WriteHeader("ระบบจัดการนักเรียน");
ConsoleHelper.WriteSuccess("บันทึกข้อมูลสำเร็จ");
ConsoleHelper.WriteError("ไม่พบข้อมูล");
ConsoleHelper.WriteWarning("กรุณาตรวจสอบข้อมูล");
ConsoleHelper.WriteInfo("กำลังประมวลผล...");

// แสดงตาราง
string[] headers = { "ลำดับ", "ชื่อ", "อายุ", "เกรด" };
string[][] rows = {
    new[] { "1", "สมชาย", "20", "3.50" },
    new[] { "2", "สมหญิง", "21", "3.75" },
    new[] { "3", "วรา", "19", "3.25" },
};
ConsoleHelper.WriteTable(headers, rows);
```

---

## 5. Console Window Control

```csharp
// ล้างหน้าจอ
Console.Clear();

// ขนาดหน้าต่าง
Console.WindowWidth = 100;
Console.WindowHeight = 40;

// ตำแหน่ง cursor
Console.SetCursorPosition(10, 5); // column, row
Console.WriteLine("ข้อความที่ตำแหน่ง (10, 5)");

// ซ่อน/แสดง cursor
Console.CursorVisible = false;  // ซ่อน
Console.CursorVisible = true;   // แสดง

// ชื่อ Title bar
Console.Title = "My Application v1.0";

// Beep
Console.Beep();          // เสียง default
Console.Beep(440, 500);  // frequency Hz, duration ms
```

---

## 6. String.Format ขั้นสูง

```csharp
// ใช้เมื่อต้องการ reuse format string
string format = "ชื่อ: {0,-20} อายุ: {1,3} เกรด: {2:F2}";

string[] names = { "สมชาย ใจดี", "สมหญิง", "วรา มีสุข" };
int[] ages = { 20, 21, 19 };
double[] gpas = { 3.50, 3.75, 3.25 };

for (int i = 0; i < names.Length; i++)
{
    Console.WriteLine(string.Format(format, names[i], ages[i], gpas[i]));
}

// FormattableString (สำหรับ i18n)
FormattableString fs = $"Today is {DateTime.Now:D}";
string formatted = FormattableString.Invariant(fs); // ไม่ขึ้นกับ locale
Console.WriteLine(formatted);

// Custom ToString
Console.WriteLine($"{1234:##,##0.00}"); // 1,234.00
Console.WriteLine($"{0.5:0%}");         // 50%
Console.WriteLine($"{12:00}:{34:00}");  // 12:34
```

---

## 7. Environment Variables

```csharp
// อ่าน Environment Variable
string? path = Environment.GetEnvironmentVariable("PATH");
string? home = Environment.GetEnvironmentVariable("HOME");
string? username = Environment.GetEnvironmentVariable("USERNAME");

Console.WriteLine($"OS: {Environment.OSVersion}");
Console.WriteLine($"Machine: {Environment.MachineName}");
Console.WriteLine($"Username: {Environment.UserName}");
Console.WriteLine($".NET Version: {Environment.Version}");
Console.WriteLine($"Processors: {Environment.ProcessorCount}");
Console.WriteLine($"64-bit: {Environment.Is64BitProcess}");
Console.WriteLine($"Working Dir: {Environment.CurrentDirectory}");

// Set Environment Variable (ชั่วคราว - เฉพาะ process นี้)
Environment.SetEnvironmentVariable("MY_APP_ENV", "development");
string? env = Environment.GetEnvironmentVariable("MY_APP_ENV");
Console.WriteLine($"Environment: {env}");

// Command-line arguments
// Program.cs ได้รับ args[] จาก Main method
// dotnet run -- arg1 arg2 arg3
```

---

## 8. Debug Output

```csharp
using System.Diagnostics;

// Trace: แสดงผลใน Debug window (VS) หรือ Console
Trace.WriteLine("Trace message");
Trace.TraceInformation("Information: process started");
Trace.TraceWarning("Warning: low memory");
Trace.TraceError("Error: cannot connect to database");

// Debug: แสดงผลเฉพาะ Debug mode
Debug.WriteLine("Debug message");
Debug.Assert(2 + 2 == 4, "คณิตศาสตร์ผิดพลาด!");  // ถ้า false → แสดง dialog/exception

// Stopwatch: วัดเวลา
var sw = new Stopwatch();
sw.Start();

// ทำงานบางอย่าง
for (int i = 0; i < 1000000; i++) { }

sw.Stop();
Console.WriteLine($"ใช้เวลา: {sw.ElapsedMilliseconds} ms");
Console.WriteLine($"ใช้เวลา: {sw.Elapsed.TotalSeconds:F4} วินาที");

// ใช้งาน multiple measurements
var sw2 = Stopwatch.StartNew();
Thread.Sleep(100);
Console.WriteLine($"Round 1: {sw2.ElapsedMilliseconds}ms");

sw2.Restart();
Thread.Sleep(200);
Console.WriteLine($"Round 2: {sw2.ElapsedMilliseconds}ms");
```

---

## 9. โปรแกรมตัวอย่างสมบูรณ์: เมนูระบบ

```csharp
using System;

class MenuProgram
{
    static void Main()
    {
        Console.Title = "ระบบจัดการข้อมูล";
        
        bool running = true;
        while (running)
        {
            ShowMainMenu();
            
            Console.Write("\nเลือกเมนู (1-5): ");
            string choice = Console.ReadLine() ?? "";
            
            Console.Clear();
            switch (choice)
            {
                case "1":
                    ShowStudentInfo();
                    break;
                case "2":
                    CalculateGrade();
                    break;
                case "3":
                    ShowColorDemo();
                    break;
                case "4":
                    ShowSystemInfo();
                    break;
                case "5":
                    running = false;
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    Console.WriteLine("ขอบคุณที่ใช้บริการ!");
                    Console.ResetColor();
                    break;
                default:
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine("เมนูไม่ถูกต้อง กรุณาลองใหม่");
                    Console.ResetColor();
                    break;
            }
            
            if (running && choice != "")
            {
                Console.Write("\nกด Enter เพื่อกลับหน้าหลัก...");
                Console.ReadLine();
                Console.Clear();
            }
        }
    }
    
    static void ShowMainMenu()
    {
        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine("╔════════════════════════════════════╗");
        Console.WriteLine("║         เมนูหลัก                   ║");
        Console.WriteLine("╠════════════════════════════════════╣");
        Console.ResetColor();
        
        Console.WriteLine("║  1. ดูข้อมูลนักเรียน               ║");
        Console.WriteLine("║  2. คำนวณเกรด                      ║");
        Console.WriteLine("║  3. ทดสอบสี Console                ║");
        Console.WriteLine("║  4. ข้อมูลระบบ                     ║");
        Console.WriteLine("║  5. ออกจากโปรแกรม                  ║");
        
        Console.ForegroundColor = ConsoleColor.Cyan;
        Console.WriteLine("╚════════════════════════════════════╝");
        Console.ResetColor();
    }
    
    static void ShowStudentInfo()
    {
        Console.WriteLine("=== ข้อมูลนักเรียน ===\n");
        
        Console.Write("ชื่อ: ");
        string name = Console.ReadLine() ?? "ไม่ระบุ";
        
        Console.Write("อายุ: ");
        int.TryParse(Console.ReadLine(), out int age);
        
        Console.Write("เกรดเฉลี่ย: ");
        double.TryParse(Console.ReadLine(), out double gpa);
        
        Console.WriteLine("\n--- ข้อมูลที่บันทึก ---");
        Console.WriteLine($"ชื่อ       : {name}");
        Console.WriteLine($"อายุ       : {age} ปี");
        Console.WriteLine($"เกรดเฉลี่ย : {gpa:F2}");
        
        string level = gpa >= 3.5 ? "เกียรตินิยมอันดับ 1"
                     : gpa >= 3.0 ? "เกียรตินิยมอันดับ 2"
                     : gpa >= 2.0 ? "ผ่าน" : "ไม่ผ่านเกณฑ์";
        
        Console.ForegroundColor = gpa >= 3.5 ? ConsoleColor.Gold
                                 : gpa >= 2.0 ? ConsoleColor.Green 
                                 : ConsoleColor.Red;
        Console.WriteLine($"ระดับ      : {level}");
        Console.ResetColor();
    }
    
    static void CalculateGrade()
    {
        Console.WriteLine("=== คำนวณเกรด ===\n");
        Console.WriteLine("ใส่คะแนนแต่ละวิชา (Enter เพื่อเสร็จสิ้น):");
        
        var scores = new System.Collections.Generic.List<double>();
        int subjectNum = 1;
        
        while (true)
        {
            Console.Write($"วิชาที่ {subjectNum} (หรือ Enter เพื่อเสร็จ): ");
            string input = Console.ReadLine() ?? "";
            
            if (string.IsNullOrWhiteSpace(input)) break;
            
            if (double.TryParse(input, out double score) && score >= 0 && score <= 100)
            {
                scores.Add(score);
                subjectNum++;
            }
            else
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("คะแนนต้องอยู่ระหว่าง 0-100");
                Console.ResetColor();
            }
        }
        
        if (scores.Count == 0)
        {
            Console.WriteLine("ไม่มีข้อมูลคะแนน");
            return;
        }
        
        double average = 0;
        double max = scores[0], min = scores[0];
        foreach (double s in scores)
        {
            average += s;
            if (s > max) max = s;
            if (s < min) min = s;
        }
        average /= scores.Count;
        
        Console.WriteLine("\n--- ผลการคำนวณ ---");
        Console.WriteLine($"จำนวนวิชา : {scores.Count}");
        Console.WriteLine($"คะแนนรวม  : {average * scores.Count:F0}");
        Console.WriteLine($"คะแนนเฉลี่ย: {average:F2}");
        Console.WriteLine($"สูงสุด     : {max}");
        Console.WriteLine($"ต่ำสุด     : {min}");
    }
    
    static void ShowColorDemo()
    {
        Console.WriteLine("=== สีใน Console ===\n");
        
        string[] colorNames = { "Black", "DarkBlue", "DarkGreen", "DarkCyan", 
            "DarkRed", "DarkMagenta", "DarkYellow", "Gray", "DarkGray",
            "Blue", "Green", "Cyan", "Red", "Magenta", "Yellow", "White" };
        
        for (int i = 0; i < colorNames.Length; i++)
        {
            if (Enum.TryParse<ConsoleColor>(colorNames[i], out var color))
            {
                Console.ForegroundColor = color;
                Console.WriteLine($"  {i + 1,2}. {colorNames[i],-15} - ข้อความตัวอย่าง");
            }
        }
        Console.ResetColor();
    }
    
    static void ShowSystemInfo()
    {
        Console.WriteLine("=== ข้อมูลระบบ ===\n");
        
        Console.WriteLine($"ระบบปฏิบัติการ  : {Environment.OSVersion}");
        Console.WriteLine($"ชื่อเครื่อง      : {Environment.MachineName}");
        Console.WriteLine($"ผู้ใช้งาน        : {Environment.UserName}");
        Console.WriteLine($"เวอร์ชัน .NET    : {Environment.Version}");
        Console.WriteLine($"จำนวน CPU        : {Environment.ProcessorCount} cores");
        Console.WriteLine($"Architecture     : {(Environment.Is64BitProcess ? "64-bit" : "32-bit")}");
        Console.WriteLine($"Working Directory: {Environment.CurrentDirectory}");
        Console.WriteLine($"Temp Directory   : {Path.GetTempPath()}");
    }
}
```

---

## 10. Exercises

### Exercise 1: เครื่องคิดเลขพื้นฐาน
```csharp
// สร้างเครื่องคิดเลขที่:
// 1. รับตัวเลข 2 ตัวจาก user
// 2. รับ operator (+, -, *, /)
// 3. คำนวณและแสดงผล
// 4. ป้องกันการหารด้วย 0
// 5. ถามว่าต้องการคำนวณต่อหรือไม่

// Example:
// ใส่ตัวเลขที่ 1: 10
// ใส่ตัวเลขที่ 2: 3
// เลือก operator (+,-,*,/): *
// 10 * 3 = 30
// คำนวณต่อ? (Y/N): 
```

### Exercise 2: ตาราง Multiplication
```csharp
// สร้างโปรแกรมที่:
// 1. ถามว่าต้องการดูสูตรคูณแม่ไหน
// 2. ถามว่าต้องการดูถึงเลขอะไร (default 12)
// 3. แสดงผลเป็นตาราง มีสี
// ตัวอย่าง:
// ═══════════════════
// สูตรคูณแม่ 7 (ถึง 12)
// ═══════════════════
// 7 × 1  =   7
// 7 × 2  =  14
// ...
```

---

## 11. สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ Console.Write / WriteLine / Read / ReadLine / ReadKey
- ✅ String Formatting: Interpolation, Composite, Format Specifiers
- ✅ Alignment และ Padding
- ✅ TryParse Pattern สำหรับ input validation
- ✅ Console Styling (สี, ตำแหน่ง)
- ✅ Environment Variables
- ✅ Debug/Trace output
- ✅ การสร้าง Menu-driven program

## Part ถัดไป
**[Part 005: คำสั่งเงื่อนไข if-else →](part-005.md)**

---

*Part 004/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
