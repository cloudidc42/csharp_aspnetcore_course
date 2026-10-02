# Part 019: Enums และ Structs

## เนื้อหาใน Part นี้
- Enum declaration และ usage
- Enum methods: ToString, Parse, GetValues
- Flags enum
- struct vs class
- struct immutability
- readonly struct
- โปรแกรมตัวอย่าง: Color struct, DayOfWeek enum

---

## 1. Enum คืออะไร?

Enum (Enumeration) คือชุดของ named constants ที่เกี่ยวข้องกัน

```csharp
// Basic enum declaration
public enum DayOfWeek
{
    Sunday = 0,    // กำหนดค่าเอง (ปกติเริ่มต้นที่ 0)
    Monday = 1,
    Tuesday = 2,
    Wednesday = 3,
    Thursday = 4,
    Friday = 5,
    Saturday = 6
}

// Enum โดยไม่กำหนดค่า - เริ่มต้นที่ 0 และเพิ่มทีละ 1
public enum Season
{
    Spring,  // 0
    Summer,  // 1
    Autumn,  // 2
    Winter   // 3
}

// Enum กับ underlying type ต่างๆ
public enum ByteEnum : byte { A = 1, B = 2, C = 3 }
public enum ShortEnum : short { X = 100, Y = 200 }
public enum LongEnum : long { Big = 1_000_000_000L, Bigger = 2_000_000_000L }

// การใช้งาน
DayOfWeek today = DayOfWeek.Friday;
Season currentSeason = Season.Winter;

Console.WriteLine(today);               // Friday
Console.WriteLine((int)today);          // 5
Console.WriteLine(currentSeason);       // Winter
Console.WriteLine((int)currentSeason);  // 3

// การเปรียบเทียบ
if (today == DayOfWeek.Friday)
{
    Console.WriteLine("TGIF! วันศุกร์แล้ว!");
}

// Switch expression
string workStatus = today switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "หยุดสุดสัปดาห์",
    DayOfWeek.Monday => "เริ่มสัปดาห์ใหม่",
    DayOfWeek.Friday => "ใกล้หยุดแล้ว!",
    _ => "วันทำงาน"
};
Console.WriteLine(workStatus);
```

---

## 2. Enum Methods

```csharp
public enum Priority
{
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4,
    Urgent = 5
}

// ToString
Priority p = Priority.High;
Console.WriteLine(p.ToString()); // "High"
Console.WriteLine(p);            // "High" (implicit ToString)

// Enum.Parse - string → enum
Priority parsed = Enum.Parse<Priority>("Medium");
Console.WriteLine(parsed); // Medium

// TryParse - ปลอดภัยกว่า
if (Enum.TryParse<Priority>("Critical", out Priority result))
{
    Console.WriteLine($"Parse สำเร็จ: {result}");
}

if (!Enum.TryParse<Priority>("Invalid", out Priority invalid))
{
    Console.WriteLine("Parse ล้มเหลว (ค่าไม่ถูกต้อง)");
}

// Case-insensitive parsing
bool success = Enum.TryParse<Priority>("high", ignoreCase: true, out Priority caseResult);
Console.WriteLine($"Case-insensitive: {caseResult}"); // High

// GetValues - ดึงค่าทั้งหมด
Console.WriteLine("\nค่าทั้งหมดของ Priority:");
foreach (Priority val in Enum.GetValues<Priority>())
{
    Console.WriteLine($"  {val} = {(int)val}");
}

// GetNames - ดึงชื่อทั้งหมด
string[] names = Enum.GetNames<Priority>();
Console.WriteLine($"\nชื่อทั้งหมด: {string.Join(", ", names)}");

// IsDefined - ตรวจสอบว่าค่านั้นถูกต้อง
Console.WriteLine(Enum.IsDefined<Priority>(Priority.High));  // True
Console.WriteLine(Enum.IsDefined<Priority>((Priority)99));   // False

// Cast from int
int rawValue = 3;
Priority fromInt = (Priority)rawValue;
Console.WriteLine($"จากเลข {rawValue}: {fromInt}"); // High

// หลุมพรางของ enum - cast ได้แม้ไม่มีค่านั้น
Priority invalid2 = (Priority)99;
Console.WriteLine(invalid2); // 99 - ไม่ crash แต่ไม่ใช่ชื่อที่กำหนด!

// ป้องกันด้วย IsDefined
if (Enum.IsDefined(typeof(Priority), rawValue))
{
    Priority safe = (Priority)rawValue;
    Console.WriteLine($"ค่าปลอดภัย: {safe}");
}

// เรียงลำดับ enum
var priorities = new[] { Priority.Critical, Priority.Low, Priority.High, Priority.Medium };
Array.Sort(priorities);
Console.WriteLine($"เรียงลำดับ: {string.Join(", ", priorities)}");
```

---

## 3. Flags Enum

Flags enum ใช้สำหรับ bit flag operations - ทำให้สามารถเก็บหลายค่าในตัวแปรเดียว

```csharp
// Flags enum - ต้องกำหนดค่าเป็น powers of 2
[Flags]
public enum FilePermission
{
    None = 0,
    Read = 1,        // 001
    Write = 2,       // 010
    Execute = 4,     // 100
    Delete = 8,      // 1000

    // Combinations
    ReadWrite = Read | Write,           // 011 = 3
    ReadExecute = Read | Execute,       // 101 = 5
    All = Read | Write | Execute | Delete // 1111 = 15
}

// การใช้งาน Flags
FilePermission userPerm = FilePermission.Read | FilePermission.Write;
Console.WriteLine(userPerm);          // Read, Write
Console.WriteLine((int)userPerm);    // 3

// ตรวจสอบว่ามี flag หนึ่งหรือไม่
bool canRead = (userPerm & FilePermission.Read) != 0;
bool canExecute = (userPerm & FilePermission.Execute) != 0;
Console.WriteLine($"Read: {canRead}, Execute: {canExecute}"); // True, False

// HasFlag method (สะดวกกว่า)
Console.WriteLine(userPerm.HasFlag(FilePermission.Read));    // True
Console.WriteLine(userPerm.HasFlag(FilePermission.Execute)); // False

// เพิ่ม flag
userPerm |= FilePermission.Execute;
Console.WriteLine($"หลังเพิ่ม Execute: {userPerm}"); // Read, Write, Execute

// ลบ flag
userPerm &= ~FilePermission.Write;
Console.WriteLine($"หลังลบ Write: {userPerm}"); // Read, Execute

// Toggle flag
userPerm ^= FilePermission.Delete;
Console.WriteLine($"หลัง toggle Delete: {userPerm}"); // Read, Execute, Delete

// ตัวอย่างที่ใช้งานจริง
[Flags]
public enum UserRole
{
    None = 0,
    Read = 1 << 0,      // 1
    Write = 1 << 1,     // 2
    Delete = 1 << 2,    // 4
    Admin = 1 << 3,     // 8
    Superuser = 1 << 4, // 16

    Editor = Read | Write,          // 3
    Moderator = Read | Write | Delete, // 7
    FullAccess = ~0                 // ทุก bit
}

UserRole adminRole = UserRole.Admin | UserRole.Read | UserRole.Write;

bool hasReadAccess = adminRole.HasFlag(UserRole.Read);
bool hasDeleteAccess = adminRole.HasFlag(UserRole.Delete);

Console.WriteLine($"Admin role: {adminRole}");
Console.WriteLine($"Can read: {hasReadAccess}");
Console.WriteLine($"Can delete: {hasDeleteAccess}");

// Helper method สำหรับ Flags
static string DescribePermissions(FilePermission perm)
{
    if (perm == FilePermission.None) return "ไม่มีสิทธิ์";

    var list = new List<string>();
    if (perm.HasFlag(FilePermission.Read)) list.Add("อ่าน");
    if (perm.HasFlag(FilePermission.Write)) list.Add("เขียน");
    if (perm.HasFlag(FilePermission.Execute)) list.Add("รัน");
    if (perm.HasFlag(FilePermission.Delete)) list.Add("ลบ");
    return string.Join(", ", list);
}

var perm = FilePermission.Read | FilePermission.Execute;
Console.WriteLine($"สิทธิ์: {DescribePermissions(perm)}");
```

---

## 4. Enum Best Practices

```csharp
// ✅ ตั้งชื่อแบบ PascalCase
public enum OrderStatus
{
    Pending,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

// ✅ มีค่า None สำหรับ Flags
[Flags]
public enum Features
{
    None = 0,
    DarkMode = 1,
    Notifications = 2,
    Analytics = 4,
    Export = 8
}

// ✅ Extension methods สำหรับ enum
public static class OrderStatusExtensions
{
    public static bool IsTerminal(this OrderStatus status)
        => status == OrderStatus.Delivered ||
           status == OrderStatus.Cancelled ||
           status == OrderStatus.Refunded;

    public static bool CanBeCancelled(this OrderStatus status)
        => status == OrderStatus.Pending || status == OrderStatus.Processing;

    public static string GetThaiName(this OrderStatus status)
        => status switch
        {
            OrderStatus.Pending => "รอดำเนินการ",
            OrderStatus.Processing => "กำลังดำเนินการ",
            OrderStatus.Shipped => "จัดส่งแล้ว",
            OrderStatus.Delivered => "ส่งถึงแล้ว",
            OrderStatus.Cancelled => "ยกเลิก",
            OrderStatus.Refunded => "คืนเงินแล้ว",
            _ => status.ToString()
        };

    public static string GetIcon(this OrderStatus status)
        => status switch
        {
            OrderStatus.Pending => "⏳",
            OrderStatus.Processing => "⚙️",
            OrderStatus.Shipped => "🚚",
            OrderStatus.Delivered => "✅",
            OrderStatus.Cancelled => "❌",
            OrderStatus.Refunded => "💰",
            _ => "❓"
        };
}

// การใช้งาน extension methods
var status = OrderStatus.Shipped;
Console.WriteLine($"{status.GetIcon()} {status.GetThaiName()}");
Console.WriteLine($"สถานะสิ้นสุด: {status.IsTerminal()}");
Console.WriteLine($"สามารถยกเลิกได้: {status.CanBeCancelled()}");
```

---

## 5. Struct คืออะไร?

Struct คือ value type ที่คล้าย class แต่มีลักษณะสำคัญที่แตกต่าง

```csharp
// Struct declaration
public struct Point
{
    public double X;
    public double Y;

    public Point(double x, double y)
    {
        X = x;
        Y = y;
    }

    public double DistanceTo(Point other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }

    public override string ToString() => $"({X}, {Y})";
}

// Struct เป็น value type
Point p1 = new Point(3, 4);
Point p2 = p1; // copy value (ไม่ใช่ reference)
p2.X = 99;

Console.WriteLine(p1); // (3, 4) - ไม่เปลี่ยน!
Console.WriteLine(p2); // (99, 4)

// ต่างจาก class ที่ copy reference
public class PointClass
{
    public double X, Y;
    public PointClass(double x, double y) { X = x; Y = y; }
}

var c1 = new PointClass(3, 4);
var c2 = c1; // copy reference - ชี้ไปที่เดียวกัน
c2.X = 99;

Console.WriteLine($"c1.X = {c1.X}"); // 99 - เปลี่ยนด้วย!
Console.WriteLine($"c2.X = {c2.X}"); // 99
```

---

## 6. Struct vs Class

| Feature | Struct | Class |
|---------|--------|-------|
| Type | Value type | Reference type |
| Memory | Stack (mostly) | Heap |
| Copy | คัดลอกค่า | คัดลอก reference |
| Null | ไม่เป็น null (เว้นแต่ Nullable) | เป็น null ได้ |
| Inheritance | ไม่รองรับ inherit | รองรับ |
| Default constructor | มีอยู่แล้ว | มีถ้าไม่กำหนด |
| IDisposable | ได้ | ได้ |
| Interface | implement ได้ | implement ได้ |
| Performance | ดีสำหรับข้อมูลเล็ก | ดีสำหรับข้อมูลใหญ่ |

```csharp
// เมื่อไหรใช้ Struct:
// 1. ข้อมูลเล็ก (< 16 bytes)
// 2. Immutable data
// 3. Short lifetime / เป็น temporary

// ตัวอย่าง struct ที่ดี
public struct Coordinate
{
    public double Latitude;
    public double Longitude;

    public Coordinate(double lat, double lon)
    {
        Latitude = lat;
        Longitude = lon;
    }
}

public struct DateOnly2  // เหมือน System.DateOnly
{
    public int Year, Month, Day;
}

public struct Temperature2
{
    public double Celsius;
}

// Boxing และ Unboxing กับ struct
Point pt = new Point(1, 2);
object boxed = pt;     // Boxing - struct → object (heap allocation!)
Point unboxed = (Point)boxed; // Unboxing

// หลีกเลี่ยง boxing เมื่อเป็นไปได้
// ใช้ generic type constraints แทน
void ProcessGeneric<T>(T value) where T : struct
{
    // ไม่มี boxing
}

void ProcessObject(object value)
{
    // มี boxing ถ้าส่ง struct เข้ามา
}
```

---

## 7. Struct Immutability

```csharp
// Struct ที่ immutable (แนะนำ)
public readonly struct ImmutablePoint
{
    public double X { get; }
    public double Y { get; }

    public ImmutablePoint(double x, double y)
    {
        X = x;
        Y = y;
    }

    // Methods คืน struct ใหม่แทนการแก้ไข
    public ImmutablePoint Translate(double dx, double dy)
        => new ImmutablePoint(X + dx, Y + dy);

    public ImmutablePoint Scale(double factor)
        => new ImmutablePoint(X * factor, Y * factor);

    public double DistanceTo(ImmutablePoint other)
    {
        double dx = X - other.X;
        double dy = Y - other.Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }

    public static ImmutablePoint Origin => new(0, 0);

    public override string ToString() => $"({X:F2}, {Y:F2})";

    public static implicit operator (double, double)(ImmutablePoint p) => (p.X, p.Y);
    public static implicit operator ImmutablePoint((double x, double y) t) => new(t.x, t.y);
}

// การใช้งาน
var p = new ImmutablePoint(3, 4);
var moved = p.Translate(1, 2);
var scaled = p.Scale(2);

Console.WriteLine($"Original: {p}");  // (3.00, 4.00)
Console.WriteLine($"Moved: {moved}"); // (4.00, 6.00)
Console.WriteLine($"Scaled: {scaled}"); // (6.00, 8.00)
Console.WriteLine($"Distance to origin: {p.DistanceTo(ImmutablePoint.Origin):F2}"); // 5.00

// Deconstruct
(double x, double y) = p;
Console.WriteLine($"x={x}, y={y}");
```

---

## 8. readonly struct

```csharp
// readonly struct - ทุก field/property เป็น readonly
public readonly struct Color
{
    public byte R { get; }
    public byte G { get; }
    public byte B { get; }
    public byte A { get; }

    public Color(byte r, byte g, byte b, byte a = 255)
    {
        R = r; G = g; B = b; A = a;
    }

    // Static factory methods
    public static Color FromHex(string hex)
    {
        hex = hex.TrimStart('#');
        return new Color(
            Convert.ToByte(hex[..2], 16),
            Convert.ToByte(hex[2..4], 16),
            Convert.ToByte(hex[4..6], 16));
    }

    public static Color FromRgb(int r, int g, int b)
        => new Color((byte)r, (byte)g, (byte)b);

    // Predefined colors
    public static readonly Color Red = new(255, 0, 0);
    public static readonly Color Green = new(0, 255, 0);
    public static readonly Color Blue = new(0, 0, 255);
    public static readonly Color White = new(255, 255, 255);
    public static readonly Color Black = new(0, 0, 0);
    public static readonly Color Transparent = new(0, 0, 0, 0);

    // Computed properties
    public double Luminance => 0.2126 * R / 255.0 + 0.7152 * G / 255.0 + 0.0722 * B / 255.0;
    public bool IsLight => Luminance > 0.5;
    public Color Inverted => new((byte)(255 - R), (byte)(255 - G), (byte)(255 - B), A);

    // Operations
    public Color Blend(Color other, double ratio = 0.5)
    {
        ratio = Math.Clamp(ratio, 0, 1);
        return new Color(
            (byte)(R * (1 - ratio) + other.R * ratio),
            (byte)(G * (1 - ratio) + other.G * ratio),
            (byte)(B * (1 - ratio) + other.B * ratio));
    }

    public Color WithAlpha(byte alpha) => new(R, G, B, alpha);

    // Conversions
    public string ToHex() => $"#{R:X2}{G:X2}{B:X2}";
    public string ToHexWithAlpha() => $"#{A:X2}{R:X2}{G:X2}{B:X2}";
    public string ToCss() => A == 255 ? $"rgb({R}, {G}, {B})" : $"rgba({R}, {G}, {B}, {A/255.0:F2})";

    // Operators
    public static Color operator +(Color a, Color b)
        => new((byte)Math.Min(a.R + b.R, 255),
               (byte)Math.Min(a.G + b.G, 255),
               (byte)Math.Min(a.B + b.B, 255));

    public static bool operator ==(Color a, Color b)
        => a.R == b.R && a.G == b.G && a.B == b.B && a.A == b.A;

    public static bool operator !=(Color a, Color b) => !(a == b);

    public override bool Equals(object? obj) => obj is Color c && this == c;
    public override int GetHashCode() => HashCode.Combine(R, G, B, A);
    public override string ToString() => $"Color({R}, {G}, {B}, {A}) = {ToHex()}";
}

// การใช้งาน
var red = Color.Red;
var blue = Color.Blue;
var purple = red.Blend(blue, 0.5);
var halfTransparent = red.WithAlpha(128);

Console.WriteLine($"Red: {red.ToHex()} CSS: {red.ToCss()}");
Console.WriteLine($"Blue: {blue.ToHex()} CSS: {blue.ToCss()}");
Console.WriteLine($"Purple blend: {purple.ToHex()}");
Console.WriteLine($"Half transparent: {halfTransparent.ToCss()}");
Console.WriteLine($"Luminance: {red.Luminance:F3}, Is Light: {red.IsLight}");
Console.WriteLine($"Inverted: {red.Inverted.ToHex()}");

var fromHex = Color.FromHex("#FF5733");
Console.WriteLine($"From Hex: {fromHex}");
```

---

## 9. Record Struct (C# 10+)

```csharp
// Record struct - value semantics + immutable + auto ToString/Equals
public readonly record struct Point3D(double X, double Y, double Z)
{
    // Properties จาก primary constructor เป็น init-only

    // เพิ่ม method
    public double DistanceTo(Point3D other)
    {
        double dx = X - other.X, dy = Y - other.Y, dz = Z - other.Z;
        return Math.Sqrt(dx*dx + dy*dy + dz*dz);
    }

    public double Magnitude => Math.Sqrt(X*X + Y*Y + Z*Z);

    public Point3D Normalize()
    {
        double mag = Magnitude;
        if (mag == 0) return this;
        return new Point3D(X/mag, Y/mag, Z/mag);
    }

    // Operators
    public static Point3D operator +(Point3D a, Point3D b)
        => new(a.X + b.X, a.Y + b.Y, a.Z + b.Z);

    public static Point3D operator *(Point3D p, double scale)
        => new(p.X * scale, p.Y * scale, p.Z * scale);

    // Static methods
    public static Point3D Origin => new(0, 0, 0);
    public static double Dot(Point3D a, Point3D b)
        => a.X*b.X + a.Y*b.Y + a.Z*b.Z;
}

// การใช้งาน
var origin = Point3D.Origin;
var p = new Point3D(1, 2, 3);
var q = new Point3D(4, 5, 6);

Console.WriteLine($"p = {p}");          // Point3D { X = 1, Y = 2, Z = 3 }
Console.WriteLine($"p + q = {p + q}");
Console.WriteLine($"p * 2 = {p * 2}");
Console.WriteLine($"|p| = {p.Magnitude:F2}");
Console.WriteLine($"p normalized = {p.Normalize()}");
Console.WriteLine($"p · q = {Point3D.Dot(p, q)}");

// Value equality
var p2 = new Point3D(1, 2, 3);
Console.WriteLine($"p == p2: {p == p2}"); // True

// with expression
var p3 = p with { Z = 10 }; // copy แล้วเปลี่ยน Z
Console.WriteLine($"p3 = {p3}");
```

---

## 10. โปรแกรมตัวอย่าง: Color Struct และ DayOfWeek Enum

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace ColorAndDayOfWeekExample
{
    /// <summary>
    /// วันในสัปดาห์พร้อมฟีเจอร์เพิ่มเติม
    /// </summary>
    public enum ThaiDayOfWeek
    {
        Sunday = 0,
        Monday = 1,
        Tuesday = 2,
        Wednesday = 3,
        Thursday = 4,
        Friday = 5,
        Saturday = 6
    }

    /// <summary>
    /// Extension methods สำหรับ ThaiDayOfWeek
    /// </summary>
    public static class ThaiDayOfWeekExtensions
    {
        private static readonly Dictionary<ThaiDayOfWeek, string> ThaiNames = new()
        {
            [ThaiDayOfWeek.Sunday] = "วันอาทิตย์",
            [ThaiDayOfWeek.Monday] = "วันจันทร์",
            [ThaiDayOfWeek.Tuesday] = "วันอังคาร",
            [ThaiDayOfWeek.Wednesday] = "วันพุธ",
            [ThaiDayOfWeek.Thursday] = "วันพฤหัสบดี",
            [ThaiDayOfWeek.Friday] = "วันศุกร์",
            [ThaiDayOfWeek.Saturday] = "วันเสาร์"
        };

        private static readonly Dictionary<ThaiDayOfWeek, Color> DayColors = new()
        {
            [ThaiDayOfWeek.Sunday] = new Color(255, 0, 0),      // แดง
            [ThaiDayOfWeek.Monday] = new Color(255, 255, 0),    // เหลือง
            [ThaiDayOfWeek.Tuesday] = new Color(255, 105, 180), // ชมพู
            [ThaiDayOfWeek.Wednesday] = new Color(0, 128, 0),   // เขียว
            [ThaiDayOfWeek.Thursday] = new Color(255, 165, 0),  // ส้ม
            [ThaiDayOfWeek.Friday] = new Color(0, 0, 255),      // น้ำเงิน
            [ThaiDayOfWeek.Saturday] = new Color(128, 0, 128)   // ม่วง
        };

        public static string GetThaiName(this ThaiDayOfWeek day)
            => ThaiNames[day];

        public static Color GetDayColor(this ThaiDayOfWeek day)
            => DayColors[day];

        public static bool IsWeekend(this ThaiDayOfWeek day)
            => day == ThaiDayOfWeek.Saturday || day == ThaiDayOfWeek.Sunday;

        public static bool IsWorkday(this ThaiDayOfWeek day)
            => !day.IsWeekend();

        public static ThaiDayOfWeek NextDay(this ThaiDayOfWeek day)
            => (ThaiDayOfWeek)(((int)day + 1) % 7);

        public static ThaiDayOfWeek PreviousDay(this ThaiDayOfWeek day)
            => (ThaiDayOfWeek)(((int)day + 6) % 7);

        public static int DaysUntil(this ThaiDayOfWeek from, ThaiDayOfWeek target)
        {
            int diff = (int)target - (int)from;
            return diff <= 0 ? diff + 7 : diff;
        }
    }

    /// <summary>
    /// Palette - คอลเลกชันของสี
    /// </summary>
    public readonly struct Palette
    {
        private readonly Color[] _colors;
        public string Name { get; }
        public int Count => _colors?.Length ?? 0;

        public Palette(string name, params Color[] colors)
        {
            Name = name;
            _colors = colors ?? Array.Empty<Color>();
        }

        public Color this[int index] => _colors[index];

        public Color GetInterpolated(double t)
        {
            if (_colors.Length == 0) return Color.Black;
            if (_colors.Length == 1) return _colors[0];

            t = Math.Clamp(t, 0, 1);
            double scaledT = t * (_colors.Length - 1);
            int lower = (int)scaledT;
            int upper = Math.Min(lower + 1, _colors.Length - 1);
            double ratio = scaledT - lower;

            return _colors[lower].Blend(_colors[upper], ratio);
        }

        public static Palette Sunset => new Palette("Sunset",
            Color.FromRgb(255, 94, 77),
            Color.FromRgb(255, 154, 0),
            Color.FromRgb(255, 206, 0),
            Color.FromRgb(255, 255, 150));

        public static Palette Ocean => new Palette("Ocean",
            Color.FromRgb(0, 20, 80),
            Color.FromRgb(0, 80, 160),
            Color.FromRgb(0, 150, 220),
            Color.FromRgb(100, 210, 255));

        public static Palette Forest => new Palette("Forest",
            Color.FromRgb(10, 60, 20),
            Color.FromRgb(30, 120, 50),
            Color.FromRgb(60, 180, 80),
            Color.FromRgb(120, 220, 120));
    }

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("=== ระบบสีและวันในสัปดาห์ ===\n");

            // สาธิต ThaiDayOfWeek
            Console.WriteLine("=== วันในสัปดาห์ไทย ===");
            foreach (ThaiDayOfWeek day in Enum.GetValues<ThaiDayOfWeek>())
            {
                Color dayColor = day.GetDayColor();
                string workStatus = day.IsWeekend() ? "หยุด" : "ทำงาน";
                Console.WriteLine($"{day.GetThaiName(),16} | สี: {dayColor.ToHex()} | {workStatus}");
            }

            // วันถัดไปและก่อนหน้า
            Console.WriteLine("\n=== Navigation ===");
            var friday = ThaiDayOfWeek.Friday;
            Console.WriteLine($"{friday.GetThaiName()} → วันถัดไป: {friday.NextDay().GetThaiName()}");
            Console.WriteLine($"{friday.GetThaiName()} → วันก่อน: {friday.PreviousDay().GetThaiName()}");

            // นับวันถึงวันหยุด
            var today = ThaiDayOfWeek.Wednesday;
            int daysToFriday = today.DaysUntil(ThaiDayOfWeek.Friday);
            int daysToSunday = today.DaysUntil(ThaiDayOfWeek.Sunday);
            Console.WriteLine($"\nวันนี้: {today.GetThaiName()}");
            Console.WriteLine($"อีก {daysToFriday} วันถึงวันศุกร์");
            Console.WriteLine($"อีก {daysToSunday} วันถึงวันอาทิตย์");

            // สาธิต Color struct
            Console.WriteLine("\n=== Color Struct ===");
            var primary = new[] { Color.Red, Color.Green, Color.Blue };
            foreach (var color in primary)
            {
                Console.WriteLine($"  {color.ToHex()} | CSS: {color.ToCss()} | " +
                    $"Luminance: {color.Luminance:F3} | Light: {color.IsLight}");
            }

            // Color blending
            Console.WriteLine("\n=== Color Blending ===");
            var blended = Color.Red.Blend(Color.Blue, 0.5);
            Console.WriteLine($"Red + Blue (50/50): {blended.ToHex()}");

            var gradient = new Color[5];
            for (int i = 0; i < 5; i++)
            {
                double t = i / 4.0;
                gradient[i] = Color.Red.Blend(Color.Blue, t);
                Console.WriteLine($"  t={t:F2}: {gradient[i].ToHex()}");
            }

            // Palette
            Console.WriteLine("\n=== Palettes ===");
            var palettes = new[] { Palette.Sunset, Palette.Ocean, Palette.Forest };
            foreach (var palette in palettes)
            {
                Console.Write($"{palette.Name,8}: ");
                for (int i = 0; i < 10; i++)
                {
                    double t = i / 9.0;
                    Console.Write($"{palette.GetInterpolated(t).ToHex()} ");
                }
                Console.WriteLine();
            }

            // Day color palette
            Console.WriteLine("\n=== วันสีสัปดาห์ ===");
            foreach (ThaiDayOfWeek day in Enum.GetValues<ThaiDayOfWeek>())
            {
                Color c = day.GetDayColor();
                string bar = new string('█', 20);
                Console.WriteLine($"{day.GetThaiName(),16}: {c.ToHex()} {bar}");
            }

            // Struct value semantics
            Console.WriteLine("\n=== Value Semantics ===");
            var c1 = Color.Red;
            var c2 = c1;           // copy by value
            c2 = c2.WithAlpha(128); // สร้าง Color ใหม่

            Console.WriteLine($"c1 (original): {c1}");
            Console.WriteLine($"c2 (modified): {c2}");
            Console.WriteLine($"c1 == c2: {c1 == c2}");

            // Equality
            var red1 = new Color(255, 0, 0);
            var red2 = new Color(255, 0, 0);
            Console.WriteLine($"red1 == red2: {red1 == red2}"); // True (value equality)
        }
    }
}
```

---

## Exercises

### Exercise 1: Chess Board
```csharp
// TODO: สร้าง enum ChessFile (A-H) และ ChessRank (1-8)
// สร้าง readonly struct ChessSquare ที่มี:
// - File (ChessFile), Rank (ChessRank)
// - IsLightSquare (computed)
// - ToString() เช่น "e4", "a1"
// - TryParse(string notation, out ChessSquare square)
// - DistanceTo(ChessSquare other) - Chebyshev distance

public enum ChessFile { A, B, C, D, E, F, G, H }
public enum ChessRank { R1 = 1, R2, R3, R4, R5, R6, R7, R8 }

public readonly struct ChessSquare
{
    // TODO: Implement
}
```

### Exercise 2: Traffic Light
```csharp
// TODO: สร้าง TrafficLightState enum
// สร้าง TrafficLight struct ที่มี:
// - CurrentState property
// - Next() -> TrafficLight (next state)
// - IsGo, IsStop, IsCaution computed properties
// - DurationSeconds computed property (Green=30, Yellow=5, Red=25)
// - [Flags] AllowedVehicles enum

public enum TrafficLightState { Red, Yellow, Green }

[Flags]
public enum AllowedVehicles { None = 0, Cars = 1, Motorcycles = 2, Pedestrians = 4 }

public readonly struct TrafficLight
{
    // TODO: Implement
}
```

### Exercise 3: Unit Measurement
```csharp
// TODO: สร้าง unit measurement system:
// enum LengthUnit { Millimeter, Centimeter, Meter, Kilometer, Inch, Foot, Yard, Mile }
// readonly struct Length ที่:
// - เก็บค่าใน meter (base unit)
// - Constructor: Length(double value, LengthUnit unit)
// - Convert(LengthUnit targetUnit) -> Length
// - Operators: +, -, *, / (scalar), <, >, ==
// - ToString(LengthUnit? unit = null)

public enum LengthUnit { Millimeter, Centimeter, Meter, Kilometer, Inch, Foot, Yard, Mile }

public readonly struct Length
{
    // TODO: Implement
}
```

---

## สรุป

✅ Enum เป็น named constants ที่เกี่ยวข้องกัน - ทำให้โค้ดอ่านง่ายขึ้น  
✅ Flags enum ใช้ bit operations เพื่อเก็บหลายค่าในตัวแปรเดียว  
✅ Enum.GetValues, Parse, TryParse, IsDefined ใช้สำหรับ reflection  
✅ Struct เป็น value type - copy by value, เหมาะกับข้อมูลเล็กและ immutable  
✅ readonly struct บังคับ immutability ทำให้ performance ดีขึ้น  
✅ Record struct (C# 10+) ให้ value equality และ auto-generated methods  
✅ Color, Point, Temperature เป็นตัวอย่างดีของ struct  
✅ ไม่ควรใช้ struct กับข้อมูลที่ใหญ่ (> 16 bytes) หรือมีการแก้ไขบ่อย  

## Part ถัดไป
**Part 020: Namespaces และ Using** - เรียนรู้ namespace declaration, nested namespaces, using directives, aliases และ global using

---
*Part 019/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
