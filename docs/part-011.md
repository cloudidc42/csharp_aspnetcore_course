# Part 011: String และการจัดการข้อความ

## เนื้อหาใน Part นี้
- String methods พื้นฐาน: Length, ToUpper/Lower, Trim, Split, Replace, Contains, StartsWith, EndsWith
- Substring, IndexOf, LastIndexOf
- String.Format และ String Interpolation ขั้นสูง
- StringBuilder สำหรับ performance
- Regular Expressions เบื้องต้น
- String Comparison
- โปรแกรมตัวอย่าง: Text Processor

---

## 1. String คืออะไร?

String ใน C# คือลำดับของตัวอักษร (sequence of characters) ที่เป็น **immutable** หมายความว่าเมื่อสร้างแล้วไม่สามารถเปลี่ยนแปลงได้ ทุกครั้งที่ "แก้ไข" string จะสร้าง object ใหม่เสมอ

```csharp
// การสร้าง string
string name = "สวัสดี C#";
string empty = "";
string nullString = null;
string fromChars = new string('A', 5); // "AAAAA"

// String เป็น immutable
string original = "Hello";
string modified = original.ToUpper(); // สร้าง string ใหม่
Console.WriteLine(original);  // ยังคงเป็น "Hello"
Console.WriteLine(modified);  // "HELLO"
```

---

## 2. String Properties พื้นฐาน

### Length
```csharp
string text = "Hello, World!";
int length = text.Length;
Console.WriteLine($"ความยาว: {length}"); // 13

// ตรวจสอบ string ว่างเปล่า
string empty = "";
Console.WriteLine(empty.Length == 0);         // True
Console.WriteLine(string.IsNullOrEmpty(empty)); // True

string spaces = "   ";
Console.WriteLine(string.IsNullOrWhiteSpace(spaces)); // True

// การเข้าถึงแต่ละตัวอักษร
string word = "C#";
char firstChar = word[0]; // 'C'
char secondChar = word[1]; // '#'
Console.WriteLine($"ตัวแรก: {firstChar}, ตัวที่สอง: {secondChar}");
```

---

## 3. String Methods: การแปลงตัวพิมพ์

```csharp
string text = "Hello, World!";

// ToUpper - แปลงเป็นตัวพิมพ์ใหญ่
string upper = text.ToUpper();
Console.WriteLine(upper); // "HELLO, WORLD!"

// ToLower - แปลงเป็นตัวพิมพ์เล็ก
string lower = text.ToLower();
Console.WriteLine(lower); // "hello, world!"

// ToUpperInvariant / ToLowerInvariant - ใช้กับ culture-independent
string invariantUpper = text.ToUpperInvariant();

// ตัวอย่างใช้งานจริง: ตรวจสอบ username
string username = "Admin";
string inputUsername = "admin";

if (username.ToLower() == inputUsername.ToLower())
{
    Console.WriteLine("Username ตรงกัน (case-insensitive)");
}

// หรือใช้ String.Compare
bool isMatch = string.Compare(username, inputUsername, 
    StringComparison.OrdinalIgnoreCase) == 0;
Console.WriteLine($"ตรงกัน: {isMatch}"); // True
```

---

## 4. String Methods: Trim, TrimStart, TrimEnd

```csharp
string textWithSpaces = "   Hello, World!   ";

// Trim - ลบช่องว่างทั้งสองด้าน
string trimmed = textWithSpaces.Trim();
Console.WriteLine($"'{trimmed}'"); // 'Hello, World!'

// TrimStart - ลบช่องว่างด้านซ้าย
string trimStart = textWithSpaces.TrimStart();
Console.WriteLine($"'{trimStart}'"); // 'Hello, World!   '

// TrimEnd - ลบช่องว่างด้านขวา
string trimEnd = textWithSpaces.TrimEnd();
Console.WriteLine($"'{trimEnd}'"); // '   Hello, World!'

// Trim กับตัวอักษรที่กำหนด
string csv = ",,,Hello,World,,,";
string trimmedCommas = csv.Trim(',');
Console.WriteLine(trimmedCommas); // "Hello,World"

// ตัวอย่างใช้งาน: ทำความสะอาด input จาก user
string userInput = "  john.doe@email.com  ";
string cleanEmail = userInput.Trim().ToLower();
Console.WriteLine(cleanEmail); // "john.doe@email.com"
```

---

## 5. String Methods: Contains, StartsWith, EndsWith

```csharp
string text = "The quick brown fox jumps over the lazy dog";

// Contains - ตรวจสอบว่ามีข้อความที่ต้องการหรือไม่
bool hasQuick = text.Contains("quick");
Console.WriteLine($"มี 'quick': {hasQuick}"); // True

bool hasCat = text.Contains("cat");
Console.WriteLine($"มี 'cat': {hasCat}"); // False

// Contains แบบ case-insensitive (C# 10+)
bool hasQuickIgnoreCase = text.Contains("QUICK", StringComparison.OrdinalIgnoreCase);
Console.WriteLine($"มี 'QUICK' (ignore case): {hasQuickIgnoreCase}"); // True

// StartsWith - ตรวจสอบจุดเริ่มต้น
bool startsWithThe = text.StartsWith("The");
Console.WriteLine($"เริ่มด้วย 'The': {startsWithThe}"); // True

// EndsWith - ตรวจสอบจุดสิ้นสุด
bool endsWithDog = text.EndsWith("dog");
Console.WriteLine($"สิ้นสุดด้วย 'dog': {endsWithDog}"); // True

// ตัวอย่างใช้งาน: ตรวจสอบ file extension
string filename = "document.pdf";
if (filename.EndsWith(".pdf", StringComparison.OrdinalIgnoreCase))
{
    Console.WriteLine("ไฟล์ PDF");
}
else if (filename.EndsWith(".docx", StringComparison.OrdinalIgnoreCase))
{
    Console.WriteLine("ไฟล์ Word");
}

// ตัวอย่างใช้งาน: ตรวจสอบ URL
string url = "https://www.example.com/page";
if (url.StartsWith("https://"))
{
    Console.WriteLine("URL ปลอดภัย (HTTPS)");
}
```

---

## 6. String Methods: Replace

```csharp
string text = "Hello, World! Hello, C#!";

// Replace - แทนที่ข้อความ
string replaced = text.Replace("Hello", "Goodbye");
Console.WriteLine(replaced); // "Goodbye, World! Goodbye, C#!"

// Replace ตัวอักษร
string textWithDash = "hello-world-foo";
string textWithUnderscore = textWithDash.Replace('-', '_');
Console.WriteLine(textWithUnderscore); // "hello_world_foo"

// Replace หลายขั้นตอน
string dirtyText = "  Hello,   World!  ";
string cleaned = dirtyText
    .Trim()
    .Replace("  ", " ");  // ลดช่องว่างหลายตัวเป็นตัวเดียว
Console.WriteLine(cleaned);

// ตัวอย่างใช้งาน: sanitize input
string userInput = "<script>alert('xss')</script>";
string safe = userInput
    .Replace("<", "&lt;")
    .Replace(">", "&gt;")
    .Replace("\"", "&quot;");
Console.WriteLine(safe);

// Remove - ลบตัวอักษร
string text2 = "Hello, World!";
string withoutComma = text2.Remove(5, 1); // ลบจากตำแหน่ง 5 จำนวน 1 ตัว
Console.WriteLine(withoutComma); // "Hello World!"
```

---

## 7. String Methods: Split

```csharp
// Split - แบ่ง string
string csv = "Apple,Banana,Cherry,Date";
string[] fruits = csv.Split(',');

foreach (string fruit in fruits)
{
    Console.WriteLine(fruit);
}
// Apple
// Banana
// Cherry
// Date

// Split หลายตัว delimiter
string text = "Hello World\tFoo\nBar";
string[] words = text.Split(new char[] { ' ', '\t', '\n' });
foreach (string word in words)
{
    Console.WriteLine($"'{word}'");
}

// Split พร้อม options
string data = "one,,two,,,three";
string[] parts = data.Split(',', StringSplitOptions.RemoveEmptyEntries);
Console.WriteLine(parts.Length); // 3 (ไม่นับ empty entries)

// Split ด้วย string
string sentence = "the-end-the-beginning-the-middle";
string[] sections = sentence.Split("the-");
foreach (string section in sections)
{
    Console.WriteLine($"'{section}'");
}

// Split กับจำนวนที่กำหนด
string limited = "a,b,c,d,e";
string[] maxParts = limited.Split(',', 3); // แบ่งสูงสุด 3 ส่วน
// ["a", "b", "c,d,e"]
foreach (string part in maxParts)
{
    Console.WriteLine(part);
}

// Join - รวม array กลับเป็น string
string[] words2 = ["Hello", "World", "C#"];
string joined = string.Join(", ", words2);
Console.WriteLine(joined); // "Hello, World, C#"

string joinedWithDash = string.Join("-", words2);
Console.WriteLine(joinedWithDash); // "Hello-World-C#"
```

---

## 8. Substring, IndexOf, LastIndexOf

```csharp
string text = "Hello, World! Hello, C#!";

// IndexOf - หาตำแหน่งแรกที่พบ
int firstIndex = text.IndexOf("Hello");
Console.WriteLine($"ตำแหน่งแรกของ 'Hello': {firstIndex}"); // 0

int commaIndex = text.IndexOf(',');
Console.WriteLine($"ตำแหน่งของ ',': {commaIndex}"); // 5

// ไม่พบจะคืนค่า -1
int notFound = text.IndexOf("xyz");
Console.WriteLine($"ไม่พบ: {notFound}"); // -1

// IndexOf เริ่มจากตำแหน่งที่กำหนด
int secondHello = text.IndexOf("Hello", 1); // เริ่มหาจากตำแหน่ง 1
Console.WriteLine($"'Hello' ตัวที่สอง: {secondHello}"); // 14

// LastIndexOf - หาตำแหน่งสุดท้ายที่พบ
int lastHello = text.LastIndexOf("Hello");
Console.WriteLine($"ตำแหน่งสุดท้ายของ 'Hello': {lastHello}"); // 14

int lastExclamation = text.LastIndexOf('!');
Console.WriteLine($"ตำแหน่ง '!' สุดท้าย: {lastExclamation}"); // 23

// Substring - ดึงส่วนของ string
string sub1 = text.Substring(7);      // จากตำแหน่ง 7 ถึงจบ
Console.WriteLine(sub1); // "World! Hello, C#!"

string sub2 = text.Substring(7, 5);   // จากตำแหน่ง 7 จำนวน 5 ตัว
Console.WriteLine(sub2); // "World"

// ตัวอย่างใช้งาน: ดึง extension จาก filename
string filename = "document.backup.pdf";
int lastDot = filename.LastIndexOf('.');
if (lastDot >= 0)
{
    string extension = filename.Substring(lastDot + 1);
    string nameWithoutExt = filename.Substring(0, lastDot);
    Console.WriteLine($"ชื่อไฟล์: {nameWithoutExt}");    // "document.backup"
    Console.WriteLine($"Extension: {extension}");          // "pdf"
}

// Range syntax (C# 8+) - วิธีที่ทันสมัยกว่า
string modernSub = text[7..12]; // "World"
Console.WriteLine(modernSub);

string last4 = text[^4..]; // 4 ตัวสุดท้าย
Console.WriteLine(last4);  // "C#!"
```

---

## 9. String.Format และ String Interpolation ขั้นสูง

```csharp
// String.Format พื้นฐาน
string formatted = string.Format("ชื่อ: {0}, อายุ: {1}", "สมชาย", 30);
Console.WriteLine(formatted);

// String Interpolation (แนะนำให้ใช้)
string name = "สมชาย";
int age = 30;
decimal salary = 50000.75m;

string interpolated = $"ชื่อ: {name}, อายุ: {age}, เงินเดือน: {salary}";
Console.WriteLine(interpolated);

// การจัดรูปแบบตัวเลข
Console.WriteLine($"จำนวนเต็ม: {age:D5}");          // 00030
Console.WriteLine($"ทศนิยม: {salary:F2}");            // 50000.75
Console.WriteLine($"เปอร์เซ็นต์: {0.75:P0}");        // 75%
Console.WriteLine($"สกุลเงิน: {salary:C}");           // ฿50,000.75 (ขึ้นกับ locale)
Console.WriteLine($"Scientific: {1234567.89:E2}");    // 1.23E+006
Console.WriteLine($"Hex: {255:X}");                   // FF
Console.WriteLine($"Hex lowercase: {255:x4}");        // 00ff

// การ Align ข้อความ
Console.WriteLine($"{'ชื่อ',10}{'อายุ',5}");          // จัดชิดขวา
Console.WriteLine($"{'สมชาย',-10}{'30',-5}");         // จัดชิดซ้าย

// Raw String Literals (C# 11+)
string json = """
    {
        "name": "สมชาย",
        "age": 30
    }
    """;
Console.WriteLine(json);

// Interpolated Raw String
string personName = "สมชาย";
int personAge = 30;
string jsonInterpolated = $"""
    {{
        "name": "{personName}",
        "age": {personAge}
    }}
    """;
Console.WriteLine(jsonInterpolated);

// วันที่
DateTime now = DateTime.Now;
Console.WriteLine($"วันที่: {now:dd/MM/yyyy}");
Console.WriteLine($"เวลา: {now:HH:mm:ss}");
Console.WriteLine($"วันเวลา: {now:dd/MM/yyyy HH:mm:ss}");
Console.WriteLine($"Long date: {now:D}");
```

---

## 10. StringBuilder - สำหรับ Performance

เมื่อต้องต่อ string หลายครั้ง ควรใช้ `StringBuilder` แทน `+` operator เพราะ string เป็น immutable การใช้ `+` จะสร้าง object ใหม่ทุกครั้ง

```csharp
using System.Text;

// ปัญหาของการใช้ string concatenation ธรรมดา
// ไม่ดี - สร้าง string ใหม่ทุกครั้ง
string result = "";
for (int i = 0; i < 10000; i++)
{
    result += i.ToString(); // สร้าง object ใหม่ทุก loop!
}

// ดี - ใช้ StringBuilder
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++)
{
    sb.Append(i.ToString()); // ไม่สร้าง object ใหม่
}
string efficientResult = sb.ToString();

// StringBuilder Methods
var builder = new StringBuilder("Hello");

// Append
builder.Append(", ");
builder.Append("World");
builder.AppendLine("!"); // เพิ่ม newline ด้วย
builder.Append("C#");

// AppendFormat
builder.AppendFormat(" v{0}.{1}", 13, 0);

// Insert
builder.Insert(0, ">>> ");

// Remove
builder.Remove(0, 4); // ลบ ">>> "

// Replace
builder.Replace("World", "Universe");

// ดึงผลลัพธ์
Console.WriteLine(builder.ToString());
Console.WriteLine($"ความยาว: {builder.Length}");

// ตัวอย่างใช้งาน: สร้าง HTML
var html = new StringBuilder();
string[] items = ["รายการ 1", "รายการ 2", "รายการ 3"];

html.AppendLine("<ul>");
foreach (string item in items)
{
    html.AppendLine($"  <li>{item}</li>");
}
html.AppendLine("</ul>");

Console.WriteLine(html.ToString());

// Benchmark เปรียบเทียบ
using System.Diagnostics;

int iterations = 10000;

// วิธีที่ 1: String concatenation
var sw1 = Stopwatch.StartNew();
string s = "";
for (int i = 0; i < iterations; i++)
{
    s += "x";
}
sw1.Stop();

// วิธีที่ 2: StringBuilder
var sw2 = Stopwatch.StartNew();
var sbBench = new StringBuilder();
for (int i = 0; i < iterations; i++)
{
    sbBench.Append("x");
}
string result2 = sbBench.ToString();
sw2.Stop();

Console.WriteLine($"String concat: {sw1.ElapsedMilliseconds}ms");
Console.WriteLine($"StringBuilder: {sw2.ElapsedMilliseconds}ms");
```

---

## 11. Regular Expressions (Regex) เบื้องต้น

```csharp
using System.Text.RegularExpressions;

// Pattern พื้นฐาน
// .  - ตัวอักษรใดก็ได้ 1 ตัว
// *  - 0 หรือมากกว่า
// +  - 1 หรือมากกว่า
// ?  - 0 หรือ 1
// \d - ตัวเลข [0-9]
// \w - ตัวอักษรหรือตัวเลข [a-zA-Z0-9_]
// \s - whitespace
// ^  - จุดเริ่มต้น
// $  - จุดสิ้นสุด

// ตรวจสอบ Email
string emailPattern = @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$";
string email1 = "user@example.com";
string email2 = "invalid-email";

bool isValidEmail1 = Regex.IsMatch(email1, emailPattern);
bool isValidEmail2 = Regex.IsMatch(email2, emailPattern);
Console.WriteLine($"{email1}: {isValidEmail1}"); // True
Console.WriteLine($"{email2}: {isValidEmail2}"); // False

// ตรวจสอบเบอร์โทรศัพท์ไทย
string phonePattern = @"^(0[689]\d{8}|0[2-9]\d{7})$";
string phone1 = "0812345678"; // มือถือ
string phone2 = "025551234";  // บ้าน

Console.WriteLine($"เบอร์ {phone1}: {Regex.IsMatch(phone1, phonePattern)}");
Console.WriteLine($"เบอร์ {phone2}: {Regex.IsMatch(phone2, phonePattern)}");

// Match - ค้นหาตรงกัน
string text = "วันที่ 01/01/2567 และ 31/12/2566";
string datePattern = @"\d{2}/\d{2}/\d{4}";

Match match = Regex.Match(text, datePattern);
if (match.Success)
{
    Console.WriteLine($"พบวันที่: {match.Value}");
}

// Matches - ค้นหาทั้งหมด
MatchCollection matches = Regex.Matches(text, datePattern);
Console.WriteLine($"พบวันที่ {matches.Count} รายการ:");
foreach (Match m in matches)
{
    Console.WriteLine($"  - {m.Value} (ตำแหน่ง {m.Index})");
}

// Groups - จับกลุ่ม
string dateText = "2567-01-15";
string groupPattern = @"(\d{4})-(\d{2})-(\d{2})";
Match dateMatch = Regex.Match(dateText, groupPattern);
if (dateMatch.Success)
{
    Console.WriteLine($"ปี: {dateMatch.Groups[1].Value}");
    Console.WriteLine($"เดือน: {dateMatch.Groups[2].Value}");
    Console.WriteLine($"วัน: {dateMatch.Groups[3].Value}");
}

// Named Groups
string namedPattern = @"(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})";
Match namedMatch = Regex.Match(dateText, namedPattern);
if (namedMatch.Success)
{
    Console.WriteLine($"ปี: {namedMatch.Groups["year"].Value}");
    Console.WriteLine($"เดือน: {namedMatch.Groups["month"].Value}");
    Console.WriteLine($"วัน: {namedMatch.Groups["day"].Value}");
}

// Replace ด้วย Regex
string dirtyText = "Hello   World    Foo";
string cleanText = Regex.Replace(dirtyText, @"\s+", " ");
Console.WriteLine(cleanText); // "Hello World Foo"

// Source Generators สำหรับ performance (C# 10+)
[GeneratedRegex(@"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$")]
static partial Regex EmailRegex();
```

---

## 12. String Comparison

```csharp
string str1 = "Hello";
string str2 = "hello";
string str3 = "Hello";

// == operator
Console.WriteLine(str1 == str2); // False (case-sensitive)
Console.WriteLine(str1 == str3); // True

// Equals method
Console.WriteLine(str1.Equals(str2)); // False
Console.WriteLine(str1.Equals(str2, StringComparison.OrdinalIgnoreCase)); // True

// String.Compare
int result1 = string.Compare(str1, str2);
// result > 0: str1 > str2
// result < 0: str1 < str2  
// result == 0: เท่ากัน

Console.WriteLine(result1); // ผลลัพธ์ขึ้นกับ ASCII

// StringComparison Options
// Ordinal - เปรียบเทียบตาม byte value (เร็วที่สุด, ไม่ขึ้นกับ culture)
// OrdinalIgnoreCase - เหมือน Ordinal แต่ ignore case
// CurrentCulture - ใช้ culture ปัจจุบัน
// CurrentCultureIgnoreCase
// InvariantCulture - ใช้ invariant culture (ดีสำหรับข้อมูลที่ไม่ขึ้นกับ locale)
// InvariantCultureIgnoreCase

// แนะนำให้ใช้ Ordinal สำหรับ internal string operations
string a = "file.txt";
string b = "File.txt";

bool ordinalEqual = string.Equals(a, b, StringComparison.Ordinal);
bool ordinalIgnoreCase = string.Equals(a, b, StringComparison.OrdinalIgnoreCase);

Console.WriteLine($"Ordinal: {ordinalEqual}");           // False
Console.WriteLine($"OrdinalIgnoreCase: {ordinalIgnoreCase}"); // True

// CompareTo - เรียงลำดับ
string[] names = ["Charlie", "alice", "Bob", "david"];
Array.Sort(names, StringComparer.OrdinalIgnoreCase);
Console.WriteLine(string.Join(", ", names)); // alice, Bob, Charlie, david

// String interning
string s1 = "hello";
string s2 = "hello";
string s3 = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });

Console.WriteLine(object.ReferenceEquals(s1, s2)); // True (interned)
Console.WriteLine(object.ReferenceEquals(s1, s3)); // False

string interned = string.Intern(s3);
Console.WriteLine(object.ReferenceEquals(s1, interned)); // True
```

---

## 13. String Methods อื่นๆ ที่มีประโยชน์

```csharp
// PadLeft / PadRight - เติมช่องว่าง
string number = "42";
string padded = number.PadLeft(5);       // "   42"
string paddedZero = number.PadLeft(5, '0'); // "00042"
string paddedRight = number.PadRight(5, '*'); // "42***"

Console.WriteLine($"'{padded}'");
Console.WriteLine($"'{paddedZero}'");
Console.WriteLine($"'{paddedRight}'");

// Concat
string part1 = "Hello";
string part2 = ", ";
string part3 = "World!";
string combined = string.Concat(part1, part2, part3);
Console.WriteLine(combined);

// Concat array
string[] parts = ["Hello", ", ", "World", "!"];
string concatAll = string.Concat(parts);
Console.WriteLine(concatAll);

// String.Copy (deprecated ใน .NET 5+)
// ใช้ new string() แทน

// ToCharArray
string text = "Hello";
char[] chars = text.ToCharArray();
chars[0] = 'J';
string newText = new string(chars);
Console.WriteLine(newText); // "Jello"

// string.IsNullOrEmpty vs string.IsNullOrWhiteSpace
string? s = null;
Console.WriteLine(string.IsNullOrEmpty(s));       // True
Console.WriteLine(string.IsNullOrWhiteSpace(s));  // True

s = "";
Console.WriteLine(string.IsNullOrEmpty(s));       // True
Console.WriteLine(string.IsNullOrWhiteSpace(s));  // True

s = "   ";
Console.WriteLine(string.IsNullOrEmpty(s));       // False
Console.WriteLine(string.IsNullOrWhiteSpace(s));  // True

// Count occurrences
string sentence = "the cat sat on the mat";
int count = sentence.Split("the").Length - 1;
Console.WriteLine($"พบ 'the' {count} ครั้ง"); // 2

// หรือใช้ Regex
int countRegex = Regex.Matches(sentence, @"\bthe\b").Count;
Console.WriteLine($"พบ 'the' (whole word) {countRegex} ครั้ง");
```

---

## 14. โปรแกรมตัวอย่าง: Text Processor

```csharp
using System;
using System.Text;
using System.Text.RegularExpressions;
using System.Collections.Generic;
using System.Linq;

namespace TextProcessorExample
{
    /// <summary>
    /// TextProcessor - โปรแกรมสำหรับประมวลผลข้อความ
    /// </summary>
    public class TextProcessor
    {
        private readonly string _text;

        public TextProcessor(string text)
        {
            _text = text ?? throw new ArgumentNullException(nameof(text));
        }

        /// <summary>
        /// นับจำนวนคำในข้อความ
        /// </summary>
        public int CountWords()
        {
            if (string.IsNullOrWhiteSpace(_text))
                return 0;

            return _text.Split(new char[] { ' ', '\t', '\n', '\r' },
                StringSplitOptions.RemoveEmptyEntries).Length;
        }

        /// <summary>
        /// นับจำนวนประโยค
        /// </summary>
        public int CountSentences()
        {
            if (string.IsNullOrWhiteSpace(_text))
                return 0;

            return _text.Split(new char[] { '.', '!', '?' },
                StringSplitOptions.RemoveEmptyEntries).Length;
        }

        /// <summary>
        /// ดึงคำที่ไม่ซ้ำกัน (unique words)
        /// </summary>
        public IEnumerable<string> GetUniqueWords()
        {
            if (string.IsNullOrWhiteSpace(_text))
                return [];

            string cleaned = Regex.Replace(_text.ToLower(), @"[^a-zA-Z0-9฀-๿\s]", "");
            return cleaned.Split(new char[] { ' ', '\t', '\n', '\r' },
                StringSplitOptions.RemoveEmptyEntries)
                .Distinct()
                .OrderBy(w => w);
        }

        /// <summary>
        /// หาคำที่ปรากฏบ่อยที่สุด
        /// </summary>
        public Dictionary<string, int> GetWordFrequency()
        {
            if (string.IsNullOrWhiteSpace(_text))
                return [];

            string cleaned = Regex.Replace(_text.ToLower(), @"[^a-zA-Z0-9฀-๿\s]", "");
            var words = cleaned.Split(new char[] { ' ', '\t', '\n', '\r' },
                StringSplitOptions.RemoveEmptyEntries);

            return words
                .GroupBy(w => w)
                .ToDictionary(g => g.Key, g => g.Count());
        }

        /// <summary>
        /// แปลงเป็น Title Case
        /// </summary>
        public string ToTitleCase()
        {
            if (string.IsNullOrWhiteSpace(_text))
                return _text;

            var words = _text.Split(' ');
            var sb = new StringBuilder();

            foreach (string word in words)
            {
                if (word.Length > 0)
                {
                    sb.Append(char.ToUpper(word[0]));
                    sb.Append(word.Substring(1).ToLower());
                    sb.Append(' ');
                }
            }

            return sb.ToString().TrimEnd();
        }

        /// <summary>
        /// ตรวจสอบว่าข้อความเป็น Palindrome หรือไม่
        /// </summary>
        public bool IsPalindrome()
        {
            if (string.IsNullOrWhiteSpace(_text))
                return false;

            string cleaned = Regex.Replace(_text.ToLower(), @"[^a-z0-9]", "");
            string reversed = new string(cleaned.Reverse().ToArray());
            return cleaned == reversed;
        }

        /// <summary>
        /// แทนที่ข้อความด้วย Regex
        /// </summary>
        public string ReplaceWithRegex(string pattern, string replacement,
            bool ignoreCase = false)
        {
            var options = ignoreCase ? RegexOptions.IgnoreCase : RegexOptions.None;
            return Regex.Replace(_text, pattern, replacement, options);
        }

        /// <summary>
        /// ดึง Email addresses จากข้อความ
        /// </summary>
        public IEnumerable<string> ExtractEmails()
        {
            const string emailPattern = @"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}";
            var matches = Regex.Matches(_text, emailPattern);
            return matches.Select(m => m.Value).Distinct();
        }

        /// <summary>
        /// ดึง URLs จากข้อความ
        /// </summary>
        public IEnumerable<string> ExtractUrls()
        {
            const string urlPattern = @"https?://[^\s]+";
            var matches = Regex.Matches(_text, urlPattern);
            return matches.Select(m => m.Value).Distinct();
        }

        /// <summary>
        /// สรุปสถิติของข้อความ
        /// </summary>
        public TextStatistics GetStatistics()
        {
            return new TextStatistics
            {
                CharacterCount = _text.Length,
                CharacterCountNoSpaces = _text.Replace(" ", "").Length,
                WordCount = CountWords(),
                SentenceCount = CountSentences(),
                UniqueWordCount = GetUniqueWords().Count(),
                AverageWordLength = GetAverageWordLength(),
                MostFrequentWords = GetWordFrequency()
                    .OrderByDescending(kv => kv.Value)
                    .Take(5)
                    .ToDictionary(kv => kv.Key, kv => kv.Value)
            };
        }

        private double GetAverageWordLength()
        {
            var words = _text.Split(new char[] { ' ', '\t', '\n', '\r' },
                StringSplitOptions.RemoveEmptyEntries);
            if (words.Length == 0) return 0;
            return words.Average(w => w.Length);
        }
    }

    public class TextStatistics
    {
        public int CharacterCount { get; set; }
        public int CharacterCountNoSpaces { get; set; }
        public int WordCount { get; set; }
        public int SentenceCount { get; set; }
        public int UniqueWordCount { get; set; }
        public double AverageWordLength { get; set; }
        public Dictionary<string, int> MostFrequentWords { get; set; } = [];

        public override string ToString()
        {
            var sb = new StringBuilder();
            sb.AppendLine("=== สถิติข้อความ ===");
            sb.AppendLine($"จำนวนตัวอักษร: {CharacterCount}");
            sb.AppendLine($"จำนวนตัวอักษร (ไม่นับช่องว่าง): {CharacterCountNoSpaces}");
            sb.AppendLine($"จำนวนคำ: {WordCount}");
            sb.AppendLine($"จำนวนประโยค: {SentenceCount}");
            sb.AppendLine($"คำที่ไม่ซ้ำกัน: {UniqueWordCount}");
            sb.AppendLine($"ความยาวคำเฉลี่ย: {AverageWordLength:F2}");
            sb.AppendLine("คำที่ปรากฏบ่อย:");
            foreach (var (word, count) in MostFrequentWords)
            {
                sb.AppendLine($"  - '{word}': {count} ครั้ง");
            }
            return sb.ToString();
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            string sampleText = """
                The quick brown fox jumps over the lazy dog.
                The dog barked loudly. The fox ran away quickly.
                Contact us at info@example.com or support@company.org.
                Visit our website at https://www.example.com.
                """;

            var processor = new TextProcessor(sampleText);

            // แสดงสถิติ
            var stats = processor.GetStatistics();
            Console.WriteLine(stats);

            // Title Case
            Console.WriteLine("\nTitle Case:");
            Console.WriteLine(processor.ToTitleCase());

            // ดึง Email
            Console.WriteLine("\nEmail ที่พบ:");
            foreach (string email in processor.ExtractEmails())
            {
                Console.WriteLine($"  - {email}");
            }

            // ดึง URL
            Console.WriteLine("\nURL ที่พบ:");
            foreach (string url in processor.ExtractUrls())
            {
                Console.WriteLine($"  - {url}");
            }

            // Palindrome test
            var palindromeTest = new TextProcessor("racecar");
            Console.WriteLine($"\n'racecar' เป็น palindrome: {palindromeTest.IsPalindrome()}");

            var notPalindrome = new TextProcessor("hello");
            Console.WriteLine($"'hello' เป็น palindrome: {notPalindrome.IsPalindrome()}");

            // Replace ด้วย Regex
            string cleaned = processor.ReplaceWithRegex(@"\s+", " ");
            Console.WriteLine($"\nทำความสะอาดช่องว่าง (ย่อ): {cleaned[..50]}...");
        }
    }
}
```

---

## Exercises

### Exercise 1: String Formatter
```csharp
// TODO: สร้าง method FormatPhoneNumber ที่รับ string เบอร์โทร
// เช่น "0812345678" -> "081-234-5678"
// เช่น "025551234" -> "02-555-1234"
// ต้องตรวจสอบว่าเบอร์ถูกต้องก่อน

public static string FormatPhoneNumber(string phone)
{
    // TODO: ลบตัวอักษรที่ไม่ใช่ตัวเลขออก
    // TODO: ตรวจสอบความยาว (9-10 หลัก)
    // TODO: จัดรูปแบบตามประเภท (มือถือ/บ้าน)
    throw new NotImplementedException();
}
```

### Exercise 2: Password Validator
```csharp
// TODO: สร้าง method ValidatePassword ที่ตรวจสอบ:
// - ความยาวอย่างน้อย 8 ตัว
// - มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว
// - มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว
// - มีตัวเลขอย่างน้อย 1 ตัว
// - มีอักขระพิเศษอย่างน้อย 1 ตัว (!@#$%^&*)
// คืนค่า List<string> ของข้อผิดพลาด (ถ้าไม่มีข้อผิดพลาด = valid)

public static List<string> ValidatePassword(string password)
{
    var errors = new List<string>();
    // TODO: เพิ่มการตรวจสอบแต่ละเงื่อนไข
    return errors;
}
```

### Exercise 3: CSV Parser
```csharp
// TODO: สร้าง class SimpleCsvParser ที่:
// - ParseLine(string line) -> string[] (รองรับ quoted fields)
// - ParseAll(string csvContent) -> List<string[]>
// - GetColumn(List<string[]> data, int columnIndex) -> string[]
// ตัวอย่าง: ParseLine("John,\"Doe, Jr.\",30") -> ["John", "Doe, Jr.", "30"]

public class SimpleCsvParser
{
    // TODO: Implement methods
}
```

---

## สรุป

✅ String ใน C# เป็น immutable - ทุก operation สร้าง string ใหม่  
✅ String methods พื้นฐาน: Length, ToUpper/Lower, Trim, Split, Replace, Contains  
✅ IndexOf/LastIndexOf ใช้หาตำแหน่ง, Substring ใช้ตัดข้อความ  
✅ String interpolation `$"..."` ใช้งานง่ายและอ่านง่ายกว่า String.Format  
✅ StringBuilder ควรใช้เมื่อต้องต่อ string หลายครั้งในลูป  
✅ Regex ใช้สำหรับ pattern matching และ validation ที่ซับซ้อน  
✅ ใช้ StringComparison.OrdinalIgnoreCase สำหรับเปรียบเทียบแบบ case-insensitive  
✅ Raw string literals `"""..."""` (C# 11+) สะดวกสำหรับ multiline strings  

## Part ถัดไป
**Part 012: Classes และ Objects เบื้องต้น** - เรียนรู้การสร้าง class, fields, properties, methods และ object creation

---
*Part 011/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
