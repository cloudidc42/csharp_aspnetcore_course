# Part 007: วนซ้ำด้วย for loop

## เนื้อหาใน Part นี้
- for loop พื้นฐาน
- การเพิ่ม/ลดค่า step
- Nested for loops
- break และ continue
- for loop ขั้นสูง
- โปรแกรมตัวอย่าง

---

## 1. for loop พื้นฐาน

```csharp
// รูปแบบ: for (init; condition; step)
// init: กำหนดค่าเริ่มต้น (ทำครั้งเดียว)
// condition: เงื่อนไขวนซ้ำ (ตรวจก่อนทุก iteration)
// step: อัปเดตค่าหลังทุก iteration

// นับ 1-5
for (int i = 1; i <= 5; i++)
{
    Console.WriteLine($"i = {i}");
}

// นับถอยหลัง
for (int i = 10; i >= 1; i--)
{
    Console.Write($"{i} ");
}
Console.WriteLine("🚀 ปล่อย!");

// ผลลัพธ์:
// 10 9 8 7 6 5 4 3 2 1 🚀 ปล่อย!

// นับ 0-9 (วิธีทั่วไปในโปรแกรมมิ่ง)
for (int i = 0; i < 10; i++)
{
    Console.Write($"{i} ");
}
Console.WriteLine();
```

---

## 2. การเพิ่ม/ลดค่า Step

```csharp
// เพิ่มทีละ 2 (เลขคู่)
Console.Write("เลขคู่: ");
for (int i = 0; i <= 20; i += 2)
{
    Console.Write($"{i} ");
}
Console.WriteLine();

// เพิ่มทีละ 5
Console.Write("ทีละ 5: ");
for (int i = 0; i <= 100; i += 5)
{
    Console.Write($"{i} ");
}
Console.WriteLine();

// ลดทีละ 3
Console.Write("ลดทีละ 3: ");
for (int i = 30; i >= 0; i -= 3)
{
    Console.Write($"{i} ");
}
Console.WriteLine();

// คูณ (Geometric progression)
Console.Write("ยกกำลัง 2: ");
for (int i = 1; i <= 1024; i *= 2)
{
    Console.Write($"{i} ");
}
Console.WriteLine();
```

---

## 3. Nested for loops (วนซ้ำซ้อนกัน)

```csharp
// สูตรคูณ 1-5
Console.WriteLine("ตารางสูตรคูณ (1-5):");
Console.Write("  ");
for (int i = 1; i <= 5; i++) Console.Write($"{i,4}");
Console.WriteLine();
Console.WriteLine(new string('-', 22));

for (int i = 1; i <= 5; i++)
{
    Console.Write($"{i} |");
    for (int j = 1; j <= 5; j++)
    {
        Console.Write($"{i * j,4}");
    }
    Console.WriteLine();
}

// วาดรูปด้วย *
// สี่เหลี่ยม
int size = 5;
Console.WriteLine("\nสี่เหลี่ยม:");
for (int row = 0; row < size; row++)
{
    for (int col = 0; col < size; col++)
    {
        Console.Write("* ");
    }
    Console.WriteLine();
}

// สามเหลี่ยม (ขั้นบันได)
Console.WriteLine("\nสามเหลี่ยม:");
for (int row = 1; row <= size; row++)
{
    for (int col = 0; col < row; col++)
    {
        Console.Write("* ");
    }
    Console.WriteLine();
}

// สามเหลี่ยมกลับหัว
Console.WriteLine("\nสามเหลี่ยมกลับหัว:");
for (int row = size; row >= 1; row--)
{
    for (int col = 0; col < row; col++)
    {
        Console.Write("* ");
    }
    Console.WriteLine();
}

// ปิรามิด
Console.WriteLine("\nปิรามิด:");
for (int row = 1; row <= size; row++)
{
    // Spaces
    for (int sp = 0; sp < size - row; sp++) Console.Write("  ");
    // Stars
    for (int col = 0; col < 2 * row - 1; col++) Console.Write("* ");
    Console.WriteLine();
}
```

---

## 4. break และ continue

### break: หยุด loop ทันที

```csharp
// หาจำนวนแรกที่หารด้วย 7 ลงตัว ในช่วง 20-50
Console.Write("หารด้วย 7 ลงตัว: ");
for (int i = 20; i <= 50; i++)
{
    if (i % 7 == 0)
    {
        Console.WriteLine(i);
        break; // หยุดเมื่อเจอแล้ว
    }
}

// ค้นหาใน array
int[] prices = { 150, 300, 75, 500, 200, 125 };
int target = 200;
int foundIndex = -1;

for (int i = 0; i < prices.Length; i++)
{
    if (prices[i] == target)
    {
        foundIndex = i;
        break;
    }
}

if (foundIndex >= 0)
    Console.WriteLine($"พบราคา {target} ที่ index {foundIndex}");
else
    Console.WriteLine($"ไม่พบราคา {target}");
```

### continue: ข้ามไปยัง iteration ถัดไป

```csharp
// แสดงเลขคี่เท่านั้น
Console.Write("เลขคี่ 1-20: ");
for (int i = 1; i <= 20; i++)
{
    if (i % 2 == 0)
        continue; // ข้ามเลขคู่
    
    Console.Write($"{i} ");
}
Console.WriteLine();

// กรองข้อมูล
string[] names = { "สมชาย", "", "สมหญิง", null, "วรา", "  " };
Console.WriteLine("ชื่อที่ถูกต้อง:");
for (int i = 0; i < names.Length; i++)
{
    if (string.IsNullOrWhiteSpace(names[i]))
        continue;
    
    Console.WriteLine($"  {names[i]}");
}
```

### break ใน Nested Loop

```csharp
// break หยุดแค่ loop ใกล้สุด
bool found = false;
int findRow = -1, findCol = -1;

int[,] grid = {
    { 1, 2, 3 },
    { 4, 5, 6 },
    { 7, 8, 9 }
};

int target = 5;

for (int row = 0; row < 3; row++)
{
    for (int col = 0; col < 3; col++)
    {
        if (grid[row, col] == target)
        {
            found = true;
            findRow = row;
            findCol = col;
            break; // หยุดแค่ inner loop
        }
    }
    
    if (found) break; // หยุด outer loop
}

if (found)
    Console.WriteLine($"พบ {target} ที่ [{findRow},{findCol}]");
```

### goto (ไม่แนะนำแต่ควรรู้)

```csharp
// goto ใน C# สำหรับออกจาก nested loop
// (แนะนำใช้ flag boolean แทน)
bool found2 = false;

outerLoop:
for (int i = 0; i < 5; i++)
{
    for (int j = 0; j < 5; j++)
    {
        if (i * j == 6)
        {
            Console.WriteLine($"พบ: {i} * {j} = 6");
            found2 = true;
            goto outerLoop; // ข้ามออกไป label
        }
    }
}
```

---

## 5. for loop ขั้นสูง

### Infinite Loop

```csharp
// Infinite loop (ออกด้วย break)
int count = 0;
for (;;) // เหมือน while(true)
{
    count++;
    if (count >= 5) break;
    Console.WriteLine($"Loop {count}");
}
```

### Multiple Variables ใน for

```csharp
// ตัวแปรหลายตัวใน for
for (int i = 0, j = 10; i < j; i++, j--)
{
    Console.WriteLine($"i={i}, j={j}");
}

// ผลลัพธ์:
// i=0, j=10
// i=1, j=9
// i=2, j=8
// i=3, j=7
// i=4, j=6
```

### Loop กับ Array

```csharp
int[] scores = { 85, 92, 78, 95, 88, 72, 90 };
int total = 0;
int max = scores[0];
int min = scores[0];

for (int i = 0; i < scores.Length; i++)
{
    total += scores[i];
    if (scores[i] > max) max = scores[i];
    if (scores[i] < min) min = scores[i];
}

double average = (double)total / scores.Length;

Console.WriteLine($"จำนวน: {scores.Length}");
Console.WriteLine($"รวม: {total}");
Console.WriteLine($"เฉลี่ย: {average:F2}");
Console.WriteLine($"สูงสุด: {max}");
Console.WriteLine($"ต่ำสุด: {min}");
```

---

## 6. โปรแกรมตัวอย่าง: ตัวเลขฟีโบนัชชี

```csharp
// Fibonacci sequence: 0, 1, 1, 2, 3, 5, 8, 13, 21, ...
Console.Write("จำนวน Fibonacci ที่ต้องการ: ");
int n = int.Parse(Console.ReadLine() ?? "10");

if (n <= 0)
{
    Console.WriteLine("ต้องมากกว่า 0");
}
else if (n == 1)
{
    Console.WriteLine("0");
}
else
{
    long prev = 0, curr = 1;
    Console.Write($"0 1 ");
    
    for (int i = 2; i < n; i++)
    {
        long next = prev + curr;
        Console.Write($"{next} ");
        prev = curr;
        curr = next;
    }
    Console.WriteLine();
}
```

---

## 7. โปรแกรมตัวอย่าง: เกม guess number

```csharp
// เกมทายตัวเลข
var random = new Random();
int secret = random.Next(1, 101); // 1-100
int attempts = 0;
int maxAttempts = 7;
bool won = false;

Console.Clear();
Console.WriteLine("╔═══════════════════════════════╗");
Console.WriteLine("║    🎮 เกมทายตัวเลข 1-100      ║");
Console.WriteLine($"║       คุณมี {maxAttempts} ครั้ง          ║");
Console.WriteLine("╚═══════════════════════════════╝\n");

for (int attempt = 1; attempt <= maxAttempts; attempt++)
{
    Console.Write($"ครั้งที่ {attempt}/{maxAttempts}: ");
    
    if (!int.TryParse(Console.ReadLine(), out int guess) || guess < 1 || guess > 100)
    {
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine("กรุณาใส่ตัวเลข 1-100");
        Console.ResetColor();
        attempt--; // ไม่นับครั้งนี้
        continue;
    }
    
    attempts++;
    
    if (guess == secret)
    {
        won = true;
        break;
    }
    
    int remaining = maxAttempts - attempt;
    string hint = guess < secret ? "⬆️  สูงกว่า" : "⬇️  ต่ำกว่า";
    string proximity = Math.Abs(guess - secret) switch
    {
        <= 5 => "🔥 ร้อนมาก!",
        <= 15 => "♨️  ร้อน",
        <= 30 => "😐 ห่างพอสมควร",
        _ => "🥶 เย็นมาก"
    };
    
    Console.ForegroundColor = ConsoleColor.Yellow;
    Console.WriteLine($"   {hint} | {proximity} | เหลือ {remaining} ครั้ง");
    Console.ResetColor();
}

Console.WriteLine();
if (won)
{
    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine($"🎉 ยินดีด้วย! คุณทายถูก! ตัวเลขคือ {secret}");
    Console.WriteLine($"   ใช้ {attempts} ครั้ง");
    
    string rating = attempts switch
    {
        1 => "⭐⭐⭐⭐⭐ เก่งมาก!",
        <= 3 => "⭐⭐⭐⭐ ดีมาก!",
        <= 5 => "⭐⭐⭐ ดี",
        _ => "⭐⭐ พยายามอีกนิด"
    };
    Console.WriteLine($"   {rating}");
}
else
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine($"😔 หมดครั้งแล้ว! ตัวเลขที่ถูกคือ {secret}");
}
Console.ResetColor();
```

---

## 8. Performance: for vs foreach

```csharp
using System.Diagnostics;

int[] bigArray = new int[10_000_000];
for (int i = 0; i < bigArray.Length; i++)
    bigArray[i] = i;

// วัดเวลา for loop
var sw = Stopwatch.StartNew();
long sum1 = 0;
for (int i = 0; i < bigArray.Length; i++)
    sum1 += bigArray[i];
sw.Stop();
Console.WriteLine($"for loop: {sw.ElapsedMilliseconds}ms, sum={sum1}");

// วัดเวลา foreach
sw.Restart();
long sum2 = 0;
foreach (int val in bigArray)
    sum2 += val;
sw.Stop();
Console.WriteLine($"foreach:  {sw.ElapsedMilliseconds}ms, sum={sum2}");

// Note: ผลต่างมักน้อยมาก ใช้อะไรก็ได้ที่อ่านง่ายกว่า
```

---

## 9. Exercises

### Exercise 1: จำนวนเฉพาะ
```csharp
// แสดงจำนวนเฉพาะทั้งหมดในช่วง 2-100
// จำนวนเฉพาะ: หารได้ด้วย 1 กับตัวเอง
// ใช้ nested loop ตรวจสอบ
```

### Exercise 2: ปิรามิดตัวเลข
```csharp
// แสดง:
//     1
//    1 2
//   1 2 3
//  1 2 3 4
// 1 2 3 4 5
// ใช้ nested loop + padding
```

### Exercise 3: เกม แบบ FizzBuzz
```csharp
// วนซ้ำ 1-100
// ถ้าหารด้วย 3: "Fizz"
// ถ้าหารด้วย 5: "Buzz"
// ถ้าหารด้วย 15: "FizzBuzz"
// อื่นๆ: ตัวเลข
```

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ for loop พื้นฐาน (init; condition; step)
- ✅ การเพิ่ม/ลดค่าหลายรูปแบบ
- ✅ Nested loops
- ✅ break และ continue
- ✅ Patterns ที่ใช้บ่อย (หาค่าสูงสุด/ต่ำสุด, ค้นหา)

## Part ถัดไป
**[Part 008: วนซ้ำด้วย while และ do-while →](part-008.md)**

---

*Part 007/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
