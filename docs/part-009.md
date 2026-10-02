# Part 009: foreach loop และ Arrays

## เนื้อหาใน Part นี้
- Arrays พื้นฐาน (1D, 2D, Jagged)
- foreach loop
- Array methods และ properties
- การจัดเรียงและค้นหา
- Span\<T\> เบื้องต้น

---

## 1. Arrays พื้นฐาน

```csharp
// ประกาศ array
int[] numbers;                        // ประกาศเฉยๆ
int[] scores = new int[5];            // 5 elements, default = 0
int[] primes = { 2, 3, 5, 7, 11 };   // กำหนดค่าพร้อมกัน
string[] names = new string[] { "สมชาย", "สมหญิง", "วรา" };
var fruits = new[] { "แอปเปิ้ล", "กล้วย", "ส้ม" }; // type inference

// Access elements (0-indexed!)
Console.WriteLine(primes[0]); // 2 (แรก)
Console.WriteLine(primes[4]); // 11 (สุดท้าย)
Console.WriteLine(primes[^1]); // 11 (Index from end, C# 8+)
Console.WriteLine(primes[^2]); // 7

// แก้ไขค่า
scores[0] = 85;
scores[1] = 92;
scores[2] = 78;

// Array Length
Console.WriteLine($"จำนวน: {primes.Length}"); // 5

// ❌ IndexOutOfRangeException
// Console.WriteLine(primes[5]); // Error!
```

### Array กับ Loop

```csharp
int[] data = { 10, 25, 8, 45, 32, 17, 56, 3 };

// for loop
Console.Write("for: ");
for (int i = 0; i < data.Length; i++)
{
    Console.Write($"{data[i]} ");
}
Console.WriteLine();

// foreach loop (อ่านง่ายกว่า แต่ไม่มี index)
Console.Write("foreach: ");
foreach (int num in data)
{
    Console.Write($"{num} ");
}
Console.WriteLine();

// คำนวณ
int sum = 0;
int max = data[0];
int min = data[0];

foreach (int n in data)
{
    sum += n;
    if (n > max) max = n;
    if (n < min) min = n;
}

Console.WriteLine($"Sum: {sum}, Average: {(double)sum/data.Length:F2}");
Console.WriteLine($"Max: {max}, Min: {min}");
```

---

## 2. 2D Arrays

```csharp
// สร้าง 2D array (matrix)
int[,] matrix = {
    { 1, 2, 3 },
    { 4, 5, 6 },
    { 7, 8, 9 }
};

// Access
Console.WriteLine(matrix[0, 0]); // 1 (row 0, col 0)
Console.WriteLine(matrix[1, 2]); // 6 (row 1, col 2)
Console.WriteLine(matrix[2, 2]); // 9

// Dimensions
int rows = matrix.GetLength(0); // 3
int cols = matrix.GetLength(1); // 3

// แสดงผล
for (int r = 0; r < rows; r++)
{
    for (int c = 0; c < cols; c++)
    {
        Console.Write($"{matrix[r, c],3}");
    }
    Console.WriteLine();
}

// ตัวอย่าง: ตารางแผนที่
char[,] map = {
    { '#', '#', '#', '#', '#' },
    { '#', '.', '.', '.', '#' },
    { '#', '.', 'P', '.', '#' },
    { '#', '.', '.', '.', '#' },
    { '#', '#', '#', '#', '#' }
};

// แสดงแผนที่
for (int r = 0; r < map.GetLength(0); r++)
{
    for (int c = 0; c < map.GetLength(1); c++)
    {
        Console.Write(map[r, c] == '#' ? "██" : map[r, c] == 'P' ? "😀" : "  ");
    }
    Console.WriteLine();
}
```

---

## 3. Jagged Arrays (Array of Arrays)

```csharp
// Jagged array: แต่ละแถวมีขนาดต่างกันได้
int[][] jagged = new int[3][];
jagged[0] = new int[] { 1, 2 };
jagged[1] = new int[] { 3, 4, 5, 6 };
jagged[2] = new int[] { 7 };

// หรือ
int[][] triangle = {
    new[] { 1 },
    new[] { 1, 2 },
    new[] { 1, 2, 3 },
    new[] { 1, 2, 3, 4 },
    new[] { 1, 2, 3, 4, 5 }
};

// แสดงสามเหลี่ยม Pascal
foreach (int[] row in triangle)
{
    foreach (int val in row)
    {
        Console.Write($"{val} ");
    }
    Console.WriteLine();
}
```

---

## 4. Array Methods

```csharp
int[] numbers = { 5, 2, 8, 1, 9, 3, 7, 4, 6 };
string[] names = { "Charlie", "Alice", "Bob", "David" };

// Sort
Array.Sort(numbers);
Console.WriteLine("Sorted: " + string.Join(", ", numbers));
// 1, 2, 3, 4, 5, 6, 7, 8, 9

Array.Sort(names);
Console.WriteLine("Names: " + string.Join(", ", names));
// Alice, Bob, Charlie, David

// Reverse
Array.Reverse(numbers);
Console.WriteLine("Reversed: " + string.Join(", ", numbers));
// 9, 8, 7, 6, 5, 4, 3, 2, 1

// Search (ต้อง Sort ก่อน!)
Array.Sort(numbers);
int index = Array.BinarySearch(numbers, 5);
Console.WriteLine($"ตำแหน่งของ 5: {index}"); // 4

// IndexOf (ไม่ต้อง Sort)
int[] arr = { 10, 20, 30, 40, 50 };
int idx = Array.IndexOf(arr, 30);
Console.WriteLine($"ตำแหน่งของ 30: {idx}"); // 2

// Fill
int[] zeros = new int[5];
Array.Fill(zeros, 99);
Console.WriteLine("Fill: " + string.Join(", ", zeros));
// 99, 99, 99, 99, 99

// Copy
int[] source = { 1, 2, 3, 4, 5 };
int[] dest = new int[5];
Array.Copy(source, dest, 5);
// หรือ
int[] copy = (int[])source.Clone();

// Clear (กำหนดเป็น 0/null)
Array.Clear(numbers, 0, numbers.Length);
Console.WriteLine("After Clear: " + string.Join(", ", numbers));
// 0, 0, 0, 0, 0, ...

// Exists
bool hasEven = Array.Exists(source, n => n % 2 == 0);
Console.WriteLine($"มีเลขคู่: {hasEven}"); // true

// Find
int firstEven = Array.Find(source, n => n % 2 == 0);
Console.WriteLine($"เลขคู่แรก: {firstEven}"); // 2

int[] allEvens = Array.FindAll(source, n => n % 2 == 0);
Console.WriteLine("เลขคู่ทั้งหมด: " + string.Join(", ", allEvens));
// 2, 4
```

---

## 5. Array Slicing (C# 8+)

```csharp
int[] arr = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// Range operator ..
int[] first3 = arr[0..3];    // { 0, 1, 2 }
int[] last3 = arr[^3..];     // { 7, 8, 9 }
int[] middle = arr[2..7];    // { 2, 3, 4, 5, 6 }
int[] copy = arr[..];        // ทั้งหมด

Console.WriteLine("First 3: " + string.Join(", ", first3));
Console.WriteLine("Last 3: " + string.Join(", ", last3));
Console.WriteLine("Middle: " + string.Join(", ", middle));

// Index type
Index last = ^1;
Console.WriteLine($"Element ก่อนสุดท้าย: {arr[^2]}"); // 8
```

---

## 6. string[] → string

```csharp
string[] words = { "Hello", "World", "C#", "Programming" };

// Join
string joined = string.Join(" ", words);
Console.WriteLine(joined); // Hello World C# Programming

string csv = string.Join(", ", words);
Console.WriteLine(csv); // Hello, World, C#, Programming

// Concat
string concat = string.Concat(words);
Console.WriteLine(concat); // HelloWorldC#Programming

// Split กลับไป
string sentence = "สวัสดี ชาวโลก โปรแกรมมิ่ง";
string[] parts = sentence.Split(' ');
foreach (string part in parts)
{
    Console.WriteLine(part);
}

// Split หลายตัวคั่น
string data = "apple,banana;orange|grape";
string[] fruits = data.Split(new char[] { ',', ';', '|' });
```

---

## 7. โปรแกรมตัวอย่าง: ระบบคะแนนนักเรียน

```csharp
// ระบบจัดการคะแนน
const int NUM_STUDENTS = 5;
const int NUM_SUBJECTS = 4;

string[] studentNames = { "สมชาย", "สมหญิง", "วรา", "สมปอง", "วิไล" };
string[] subjects = { "คณิตฯ", "วิทย์", "ไทย", "อังกฤษ" };
int[,] scores = {
    { 85, 92, 78, 88 },
    { 90, 85, 95, 82 },
    { 72, 68, 80, 75 },
    { 88, 91, 85, 89 },
    { 76, 80, 88, 84 }
};

// Header
Console.Write($"{"ชื่อ",-12}");
foreach (string subj in subjects)
    Console.Write($"{subj,8}");
Console.Write($"{"รวม",8}{"เฉลี่ย",10}{"เกรด",6}");
Console.WriteLine();
Console.WriteLine(new string('-', 70));

// แสดงผลแต่ละคน
for (int s = 0; s < NUM_STUDENTS; s++)
{
    Console.Write($"{studentNames[s],-12}");
    int total = 0;
    
    for (int sub = 0; sub < NUM_SUBJECTS; sub++)
    {
        Console.Write($"{scores[s, sub],8}");
        total += scores[s, sub];
    }
    
    double avg = (double)total / NUM_SUBJECTS;
    string grade = avg >= 80 ? "A" : avg >= 70 ? "B" : avg >= 60 ? "C" : "F";
    
    Console.ForegroundColor = grade == "A" ? ConsoleColor.Green 
                            : grade == "F" ? ConsoleColor.Red 
                            : ConsoleColor.White;
    Console.WriteLine($"{total,8}{avg,10:F2}{grade,6}");
    Console.ResetColor();
}

Console.WriteLine(new string('-', 70));

// สถิติแต่ละวิชา
Console.WriteLine("\nสถิติแต่ละวิชา:");
Console.Write($"{"วิชา",-12}{"เฉลี่ย",10}{"สูงสุด",10}{"ต่ำสุด",10}");
Console.WriteLine();

for (int sub = 0; sub < NUM_SUBJECTS; sub++)
{
    int max = scores[0, sub], min = scores[0, sub], total = 0;
    
    for (int s = 0; s < NUM_STUDENTS; s++)
    {
        total += scores[s, sub];
        if (scores[s, sub] > max) max = scores[s, sub];
        if (scores[s, sub] < min) min = scores[s, sub];
    }
    
    Console.WriteLine($"{subjects[sub],-12}{(double)total/NUM_STUDENTS,10:F2}{max,10}{min,10}");
}
```

---

## 8. Exercises

### Exercise 1: Bubble Sort
```csharp
// Implement bubble sort เอง
int[] arr = { 64, 34, 25, 12, 22, 11, 90 };
// Sort ascending without using Array.Sort
// แสดงผลทุก pass
```

### Exercise 2: Matrix Multiplication
```csharp
// คูณ matrix 2x2 กับ 2x2
// A = [[1,2],[3,4]], B = [[5,6],[7,8]]
// C = A × B
```

---

## สรุป

ใน Part นี้คุณได้เรียนรู้:
- ✅ Arrays 1D, 2D, Jagged
- ✅ foreach loop
- ✅ Array.Sort, BinarySearch, IndexOf, Fill, Copy
- ✅ Array Slicing กับ Range operator
- ✅ การจัดการ string arrays

## Part ถัดไป
**[Part 010: Methods และ Functions →](part-010.md)**

---

*Part 009/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
