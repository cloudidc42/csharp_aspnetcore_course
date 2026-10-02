# Part 001: การติดตั้ง .NET และเริ่มต้นโปรแกรมแรก

## เนื้อหาใน Part นี้
- .NET คืออะไร และทำไมต้องเรียน C#
- ติดตั้ง .NET SDK
- ติดตั้ง IDE (Visual Studio / VS Code)
- สร้างโปรแกรม "Hello, World!" แรก
- ทำความเข้าใจโครงสร้างโปรเจค
- การ Build และ Run โปรแกรม

---

## 1. .NET คืออะไร?

**.NET** (ออกเสียงว่า "ดอทเน็ต") คือ platform สำหรับพัฒนาซอฟต์แวร์ที่สร้างโดย Microsoft ซึ่งรองรับการพัฒนาแอพพลิเคชันหลากหลายประเภท:

```
┌─────────────────────────────────────────────────┐
│                    .NET Platform                 │
├──────────────┬──────────────┬────────────────────┤
│  Web Apps    │  Mobile Apps │  Desktop Apps      │
│  (ASP.NET)   │  (MAUI)      │  (WinForms/WPF)    │
├──────────────┴──────────────┴────────────────────┤
│              Cloud / Microservices                │
├─────────────────────────────────────────────────┤
│              Game Development (Unity)             │
├─────────────────────────────────────────────────┤
│           IoT / Embedded Systems                  │
└─────────────────────────────────────────────────┘
```

### ทำไมต้องเรียน C#?

| เหตุผล | รายละเอียด |
|--------|-----------|
| **อาชีพการงาน** | ตลาดงาน C# ในไทยและต่างประเทศมีความต้องการสูง เงินเดือนดี |
| **Cross-platform** | เขียนครั้งเดียว รันได้บน Windows, Linux, macOS |
| **Performance** | เร็วและประหยัด memory กว่าหลายภาษา |
| **Community** | ชุมชนใหญ่ มี documentation ดีเยี่ยม |
| **Modern Language** | มีฟีเจอร์ทันสมัย อัปเดตทุกปี |
| **Enterprise** | ใช้ในองค์กรใหญ่ระดับโลก เช่น Microsoft, Stack Overflow |

### .NET vs .NET Framework vs .NET Core

```
Timeline:
2002  ───── .NET Framework 1.0 (Windows only)
             │
2016  ───── .NET Core 1.0 (Cross-platform)
             │
2020  ───── .NET 5 (รวม .NET Framework + .NET Core)
             │
2021  ───── .NET 6 (LTS)
             │
2022  ───── .NET 7
             │
2023  ───── .NET 8 (LTS) ← แนะนำให้ใช้ใน production
             │
2024  ───── .NET 9 ← เวอร์ชันล่าสุด (ใช้ใน course นี้)
```

**คำแนะนำ:** ใช้ **.NET 8 หรือ .NET 9** สำหรับโปรเจคใหม่ทุกอัน

---

## 2. การติดตั้ง .NET SDK

### Windows

**วิธีที่ 1: ดาวน์โหลดโดยตรง**
1. ไปที่ https://dot.net/download
2. เลือก **.NET 9 SDK** (หรือ .NET 8 LTS)
3. เลือก **Windows x64**
4. ดาวน์โหลดและรัน installer

**วิธีที่ 2: ใช้ winget (แนะนำ)**
```powershell
# ติดตั้ง .NET 9 SDK
winget install Microsoft.DotNet.SDK.9

# หรือ .NET 8 LTS
winget install Microsoft.DotNet.SDK.8
```

**วิธีที่ 3: ใช้ Chocolatey**
```powershell
choco install dotnet-sdk
```

### macOS

**วิธีที่ 1: ดาวน์โหลดโดยตรง**
1. ไปที่ https://dot.net/download
2. เลือก **macOS** (x64 หรือ Arm64 สำหรับ Apple Silicon)
3. ดาวน์โหลดและติดตั้ง

**วิธีที่ 2: ใช้ Homebrew (แนะนำ)**
```bash
# ติดตั้ง .NET 9
brew install --cask dotnet

# หรือระบุเวอร์ชัน
brew install dotnet@9
```

### Linux (Ubuntu/Debian)

```bash
# เพิ่ม Microsoft package repository
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# ติดตั้ง .NET SDK
sudo apt-get update
sudo apt-get install -y dotnet-sdk-9.0
```

### Linux (Fedora/RHEL)
```bash
sudo dnf install dotnet-sdk-9.0
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบเวอร์ชัน .NET
dotnet --version

# ดู SDK ที่ติดตั้งทั้งหมด
dotnet --list-sdks

# ดู runtime ที่ติดตั้งทั้งหมด
dotnet --list-runtimes

# ดูข้อมูลทั้งหมด
dotnet --info
```

**ผลลัพธ์ที่ควรได้:**
```
9.0.100
```

---

## 3. การติดตั้ง IDE

### Visual Studio 2022 (Windows - แนะนำสำหรับมือใหม่)

1. ดาวน์โหลดจาก https://visualstudio.microsoft.com/
2. เลือก **Community Edition** (ฟรี)
3. ระหว่างติดตั้ง เลือก workloads:
   - ✅ **ASP.NET and web development**
   - ✅ **.NET desktop development**
   - ✅ **Azure development** (ถ้าต้องการ)

### Visual Studio Code (ทุก Platform - แนะนำ)

1. ดาวน์โหลดจาก https://code.visualstudio.com/
2. ติดตั้ง Extensions:

```
Extensions ที่จำเป็น:
- C# Dev Kit (by Microsoft) ← สำคัญมาก
- .NET Install Tool
- GitLens (optional แต่มีประโยชน์)
```

**วิธีติดตั้ง Extension ใน VS Code:**
1. กด `Ctrl+Shift+X` (Windows/Linux) หรือ `Cmd+Shift+X` (macOS)
2. ค้นหา "C# Dev Kit"
3. คลิก Install

### JetBrains Rider (ทุก Platform - มีประสิทธิภาพสูงสุด)

- ดาวน์โหลดจาก https://www.jetbrains.com/rider/
- มีค่าใช้จ่าย (มี free trial 30 วัน)
- แนะนำสำหรับ developer มืออาชีพ

---

## 4. สร้างโปรแกรม "Hello, World!" แรก

### วิธีที่ 1: ผ่าน Terminal/Command Prompt

```bash
# 1. สร้าง folder ใหม่
mkdir HelloWorld
cd HelloWorld

# 2. สร้างโปรเจค Console Application
dotnet new console

# 3. ดูไฟล์ที่สร้างขึ้น
ls -la   # Linux/macOS
dir      # Windows
```

**ผลลัพธ์:**
```
HelloWorld/
├── HelloWorld.csproj    ← Project file
├── Program.cs           ← โค้ดหลัก
└── obj/                 ← Build artifacts (สร้างอัตโนมัติ)
```

### ดูโค้ดใน Program.cs

```bash
cat Program.cs   # Linux/macOS
type Program.cs  # Windows
```

**เนื้อหาของ Program.cs:**
```csharp
// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, World!");
```

### รันโปรแกรม

```bash
dotnet run
```

**ผลลัพธ์:**
```
Hello, World!
```

🎉 **ยินดีด้วย!** คุณเพิ่งรันโปรแกรม C# แรกของคุณแล้ว!

---

## 5. ทำความเข้าใจโครงสร้างโปรเจค

### ไฟล์ HelloWorld.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>        <!-- ประเภทของ output: Exe = Console App -->
    <TargetFramework>net9.0</TargetFramework>  <!-- .NET version ที่ใช้ -->
    <RootNamespace>HelloWorld</RootNamespace>  <!-- Namespace หลัก -->
    <Nullable>enable</Nullable>          <!-- เปิดใช้ Nullable Reference Types -->
    <ImplicitUsings>enable</ImplicitUsings>  <!-- Auto-import common namespaces -->
  </PropertyGroup>

</Project>
```

### ไฟล์ Program.cs (แบบเต็ม)

เวอร์ชันที่เห็นข้างบนคือ "Top-level statements" ซึ่งเป็น syntax ใหม่ใน C# 9+

**รูปแบบเดิม (C# 8 และก่อนหน้า):**
```csharp
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

**รูปแบบใหม่ (C# 9+ Top-level statements):**
```csharp
Console.WriteLine("Hello, World!");
```

ในหลักสูตรนี้เราจะใช้ **รูปแบบใหม่** เป็นหลัก แต่ต้องเข้าใจรูปแบบเดิมด้วย เพราะจะเจอในโค้ดเก่า

---

## 6. การเขียนโปรแกรมเพิ่มเติม

### แก้ไข Program.cs

เปิดไฟล์ `Program.cs` และแก้ไขเป็น:

```csharp
// โปรแกรมแรกของฉัน
Console.WriteLine("Hello, World!");
Console.WriteLine("สวัสดีชาวโลก!");
Console.WriteLine("ยินดีต้อนรับสู่หลักสูตร C#");
Console.WriteLine("วันนี้คือวันแรกของการเป็น Developer!");

// แสดงผลโดยไม่ขึ้นบรรทัดใหม่
Console.Write("ชื่อฉันคือ ");
Console.Write("Developer ");
Console.WriteLine("คนใหม่");

// แสดงบรรทัดว่าง
Console.WriteLine();

// แสดงข้อความซ้ำ 5 ครั้ง
for (int i = 1; i <= 5; i++)
{
    Console.WriteLine($"บรรทัดที่ {i}: Hello!");
}
```

**ผลลัพธ์:**
```
Hello, World!
สวัสดีชาวโลก!
ยินดีต้อนรับสู่หลักสูตร C#
วันนี้คือวันแรกของการเป็น Developer!
ชื่อฉันคือ Developer คนใหม่

บรรทัดที่ 1: Hello!
บรรทัดที่ 2: Hello!
บรรทัดที่ 3: Hello!
บรรทัดที่ 4: Hello!
บรรทัดที่ 5: Hello!
```

---

## 7. คำสั่ง dotnet CLI ที่ควรรู้

```bash
# สร้างโปรเจคใหม่
dotnet new console -n MyApp    # Console App
dotnet new webapi -n MyApi     # Web API
dotnet new mvc -n MyWeb        # MVC Web App
dotnet new classlib -n MyLib   # Class Library

# Build โปรเจค
dotnet build

# Run โปรเจค
dotnet run

# Run พร้อม watch (auto-reload เมื่อโค้ดเปลี่ยน)
dotnet watch run

# ทดสอบ
dotnet test

# Publish (สำหรับ deploy)
dotnet publish -c Release -o ./publish

# เพิ่ม NuGet package
dotnet add package Newtonsoft.Json

# ดู packages ที่ติดตั้ง
dotnet list package

# ลบ package
dotnet remove package Newtonsoft.Json
```

### ชนิดของโปรเจคที่สร้างได้

```bash
# ดูประเภทโปรเจคทั้งหมด
dotnet new list
```

**โปรเจคที่ใช้บ่อย:**

| คำสั่ง | ประเภท | ใช้สำหรับ |
|--------|--------|----------|
| `dotnet new console` | Console App | เรียนรู้ C#, CLI tools |
| `dotnet new webapi` | Web API | REST API |
| `dotnet new mvc` | MVC | Web Application |
| `dotnet new blazorwasm` | Blazor WASM | Single Page App |
| `dotnet new worker` | Worker Service | Background services |
| `dotnet new classlib` | Class Library | Shared code |
| `dotnet new xunit` | Unit Test | Testing |

---

## 8. โครงสร้าง Solution (สำหรับโปรเจคใหญ่)

```bash
# สร้าง Solution
dotnet new sln -n MyCompany

# เพิ่มโปรเจคลงใน Solution
dotnet sln add HelloWorld/HelloWorld.csproj
dotnet sln add MyLibrary/MyLibrary.csproj

# ดู Solution structure
dotnet sln list
```

**โครงสร้างทั่วไปของโปรเจคจริง:**
```
MyCompany/
├── MyCompany.sln
├── src/
│   ├── MyCompany.Core/           # Business logic
│   ├── MyCompany.Infrastructure/ # Database, External services
│   ├── MyCompany.Api/            # Web API
│   └── MyCompany.Web/            # Frontend
├── tests/
│   ├── MyCompany.Core.Tests/
│   └── MyCompany.Api.Tests/
└── docs/
    └── README.md
```

---

## 9. การใช้ VS Code กับ C#

### เปิดโปรเจคใน VS Code

```bash
# เปิด VS Code ใน folder ปัจจุบัน
code .
```

### Keyboard Shortcuts ที่ควรรู้

| Shortcut (Windows/Linux) | Shortcut (macOS) | การทำงาน |
|--------------------------|-----------------|---------|
| `Ctrl+Shift+P` | `Cmd+Shift+P` | Command Palette |
| `F5` | `F5` | Start Debugging |
| `Ctrl+F5` | `Ctrl+F5` | Run without Debugging |
| `Ctrl+`` ` | `Ctrl+`` ` | เปิด Terminal |
| `Ctrl+S` | `Cmd+S` | Save file |
| `Ctrl+Z` | `Cmd+Z` | Undo |
| `F12` | `F12` | Go to Definition |
| `Ctrl+.` | `Cmd+.` | Quick Fix |

### การ Debug โปรแกรม

1. คลิกที่ขอบซ้ายของ editor เพื่อตั้ง **Breakpoint** (จะเห็นจุดแดง)
2. กด `F5` เพื่อ Debug
3. โปรแกรมจะหยุดที่ Breakpoint
4. ใช้ `F10` เพื่อ Step Over, `F11` เพื่อ Step Into
5. ดู variables ใน Debug panel ทางซ้าย

---

## 10. โปรแกรมฝึกหัด

### Exercise 1: Hello Personal
สร้างโปรแกรมที่แสดงข้อมูลส่วนตัว:

```csharp
// แก้ไขให้เป็นข้อมูลของคุณเอง
Console.WriteLine("=================================");
Console.WriteLine("       ข้อมูลส่วนตัว           ");
Console.WriteLine("=================================");
Console.WriteLine("ชื่อ: สมชาย ใจดี");
Console.WriteLine("อายุ: 25 ปี");
Console.WriteLine("อาชีพ: นักพัฒนาซอฟต์แวร์ (กำลังเรียน)");
Console.WriteLine("ภาษาที่เรียน: C# และ ASP.NET Core");
Console.WriteLine("เป้าหมาย: เป็น Senior Developer ภายใน 2 ปี");
Console.WriteLine("=================================");
```

### Exercise 2: ASCII Art
```csharp
Console.WriteLine("   /\\_/\\  ");
Console.WriteLine("  ( o.o ) ");
Console.WriteLine("   > ^ <  ");
Console.WriteLine("  |     | ");
Console.WriteLine("  Hello C#!");
```

### Exercise 3: สูตรคูณ
```csharp
int table = 5; // เปลี่ยนเป็นตารางที่ต้องการ

Console.WriteLine($"สูตรคูณแม่ {table}");
Console.WriteLine("================");
for (int i = 1; i <= 12; i++)
{
    Console.WriteLine($"{table} x {i} = {table * i}");
}
```

**ผลลัพธ์ Exercise 3:**
```
สูตรคูณแม่ 5
================
5 x 1 = 5
5 x 2 = 10
5 x 3 = 15
5 x 4 = 20
5 x 5 = 25
5 x 6 = 30
5 x 7 = 35
5 x 8 = 40
5 x 9 = 45
5 x 10 = 50
5 x 11 = 55
5 x 12 = 60
```

---

## 11. ข้อผิดพลาดที่พบบ่อยและวิธีแก้

### Error 1: dotnet ไม่เจอ Command
```
'dotnet' is not recognized as an internal or external command
```
**แก้ไข:** ติดตั้ง .NET SDK และ Restart terminal

### Error 2: Build Error
```
error CS1002: ; expected
```
**แก้ไข:** ลืมใส่ `;` ท้าย statement

### Error 3: File Not Found
```
The project file could not be found.
```
**แก้ไข:** ต้องอยู่ใน folder เดียวกับ `.csproj` file

### Error 4: Port Already in Use
```
Failed to bind to address https://localhost:5001
```
**แก้ไข:** มีโปรแกรมอื่นใช้ port อยู่ หรือเปลี่ยน port ใน settings

---

## 12. สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ .NET คืออะไรและมีประโยชน์อย่างไร
- ✅ วิธีติดตั้ง .NET SDK บนทุก Platform
- ✅ วิธีติดตั้งและใช้งาน IDE
- ✅ สร้างโปรแกรม Hello World แรก
- ✅ โครงสร้างโปรเจค .NET
- ✅ คำสั่ง `dotnet` CLI พื้นฐาน

## Part ถัดไป
**[Part 002: ตัวแปรและชนิดข้อมูลพื้นฐาน →](part-002.md)**

---

## แหล่งเรียนรู้เพิ่มเติม

- [Microsoft .NET Documentation](https://docs.microsoft.com/dotnet/)
- [C# Documentation](https://docs.microsoft.com/csharp/)
- [.NET Foundation](https://dotnetfoundation.org/)
- [C# Corner](https://www.c-sharpcorner.com/)

---

*Part 001/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
