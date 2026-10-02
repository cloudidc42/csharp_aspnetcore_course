# Part 021: Collections - List\<T\> และ Dictionary\<TKey, TValue\>

## เนื้อหาใน Part นี้
- ทำความรู้จักกับ Generic Collections
- List\<T\>: การใช้งาน Add, Remove, Insert, Count, Contains, Find, Sort
- Dictionary\<TKey, TValue\>: การใช้งาน Add, Remove, ContainsKey, TryGetValue, Iteration
- เปรียบเทียบ List vs Array
- รูปแบบการใช้ Dictionary ที่นิยม
- โปรแกรมตัวอย่าง: Contact Book ด้วย Dictionary

---

## 1. Generic Collections คืออะไร?

Generic Collections คือโครงสร้างข้อมูลที่รองรับ Type ที่กำหนดเอง (Type Parameter) ทำให้ปลอดภัยในระดับ Compile-time และไม่ต้องทำ Boxing/Unboxing

```csharp
// ใน namespace System.Collections.Generic
using System.Collections.Generic;

// Non-generic (เก่า - ไม่แนะนำ)
ArrayList list = new ArrayList();
list.Add(1);
list.Add("hello"); // ผิดพลาดได้ที่ runtime

// Generic (ใหม่ - แนะนำ)
List<int> numbers = new List<int>();
numbers.Add(1);
// numbers.Add("hello"); // Error ที่ compile-time!
```

**ข้อดีของ Generic Collections:**
- Type Safety: ตรวจสอบ Type ตั้งแต่ Compile-time
- Performance: ไม่มี Boxing/Unboxing สำหรับ Value Types
- IntelliSense: IDE ช่วยได้ดีกว่า
- Code ชัดเจน: รู้ทันทีว่าเก็บข้อมูลชนิดใด

---

## 2. List\<T\>

`List<T>` คือ Dynamic Array ที่ขยายขนาดได้อัตโนมัติ เป็น Collection ที่ใช้บ่อยที่สุด

### 2.1 การสร้าง List

```csharp
// วิธีที่ 1: Empty list
List<string> names = new List<string>();

// วิธีที่ 2: กำหนด initial capacity (ประสิทธิภาพดีขึ้นเมื่อรู้จำนวนล่วงหน้า)
List<int> scores = new List<int>(100);

// วิธีที่ 3: สร้างจาก collection อื่น
int[] arr = { 1, 2, 3, 4, 5 };
List<int> fromArray = new List<int>(arr);

// วิธีที่ 4: Collection Initializer (C# syntax)
List<string> fruits = new List<string> { "Apple", "Banana", "Cherry" };

// วิธีที่ 5: Target-typed new expression (C# 9+)
List<double> prices = new() { 19.99, 29.99, 9.99 };
```

### 2.2 การเพิ่มข้อมูล: Add, AddRange, Insert

```csharp
List<string> cities = new List<string>();

// Add: เพิ่มที่ท้าย
cities.Add("Bangkok");
cities.Add("Chiang Mai");
cities.Add("Phuket");

// AddRange: เพิ่มหลายรายการพร้อมกัน
string[] moreCities = { "Pattaya", "Hua Hin", "Krabi" };
cities.AddRange(moreCities);

// Insert: แทรกที่ตำแหน่งที่กำหนด
cities.Insert(0, "Nonthaburi"); // แทรกที่ index 0
cities.Insert(2, "Samut Prakan"); // แทรกที่ index 2

Console.WriteLine("Cities:");
foreach (string city in cities)
{
    Console.WriteLine($"  - {city}");
}
// Output:
// Cities:
//   - Nonthaburi
//   - Bangkok
//   - Samut Prakan
//   - Chiang Mai
//   - ...
```

### 2.3 การลบข้อมูล: Remove, RemoveAt, RemoveRange, Clear

```csharp
List<int> numbers = new List<int> { 10, 20, 30, 40, 50, 30, 60 };

// Remove: ลบ element แรกที่พบ (คืน true/false)
bool removed = numbers.Remove(30); // ลบ 30 ตัวแรก
Console.WriteLine($"Removed: {removed}"); // true
Console.WriteLine(string.Join(", ", numbers)); // 10, 20, 40, 50, 30, 60

// RemoveAt: ลบตามตำแหน่ง index
numbers.RemoveAt(0); // ลบ index 0 (ค่า 10)
Console.WriteLine(string.Join(", ", numbers)); // 20, 40, 50, 30, 60

// RemoveRange: ลบหลายรายการ
numbers.RemoveRange(1, 2); // ลบ 2 รายการ เริ่มจาก index 1
Console.WriteLine(string.Join(", ", numbers)); // 20, 30, 60

// RemoveAll: ลบทุกรายการที่ตรงกับเงื่อนไข
numbers.RemoveAll(n => n > 25); // ลบทุกค่าที่มากกว่า 25
Console.WriteLine(string.Join(", ", numbers)); // 20

// Clear: ลบทั้งหมด
numbers.Clear();
Console.WriteLine($"Count after clear: {numbers.Count}"); // 0
```

### 2.4 Count, Capacity, Contains

```csharp
List<string> items = new List<string>(10); // Capacity เริ่มต้น 10

items.Add("A");
items.Add("B");
items.Add("C");

Console.WriteLine($"Count: {items.Count}");       // 3 (จำนวนจริง)
Console.WriteLine($"Capacity: {items.Capacity}"); // 10 (พื้นที่จองไว้)

// Contains: ตรวจสอบว่ามีอยู่หรือไม่
bool hasA = items.Contains("A");   // true
bool hasZ = items.Contains("Z");   // false

// IndexOf: หา index
int idx = items.IndexOf("B");      // 1
int notFound = items.IndexOf("X"); // -1

// LastIndexOf: หา index จากท้าย
List<int> nums = new List<int> { 1, 2, 3, 2, 1 };
int lastTwo = nums.LastIndexOf(2); // 3
```

### 2.5 Find, FindAll, FindIndex

```csharp
List<int> scores = new List<int> { 85, 92, 67, 78, 95, 88, 71 };

// Find: หา element แรกที่ตรงเงื่อนไข (หรือ default ถ้าไม่พบ)
int firstHigh = scores.Find(s => s >= 90); // 92

// FindLast: หา element สุดท้ายที่ตรงเงื่อนไข
int lastHigh = scores.FindLast(s => s >= 90); // 95

// FindAll: หาทุก element ที่ตรงเงื่อนไข
List<int> highScores = scores.FindAll(s => s >= 85);
Console.WriteLine("High scores: " + string.Join(", ", highScores)); // 85, 92, 95, 88

// FindIndex: หา index ของ element แรกที่ตรงเงื่อนไข
int idx = scores.FindIndex(s => s >= 90); // 1

// Exists: ตรวจสอบว่ามี element ที่ตรงเงื่อนไขหรือไม่
bool hasPerfect = scores.Exists(s => s == 100); // false
bool hasGood = scores.Exists(s => s >= 85);     // true
```

### 2.6 Sort, Reverse, BinarySearch

```csharp
List<int> numbers = new List<int> { 5, 2, 8, 1, 9, 3, 7, 4, 6 };

// Sort: เรียงลำดับ (ascending โดยค่าเริ่มต้น)
numbers.Sort();
Console.WriteLine(string.Join(", ", numbers)); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// Sort ด้วย Comparison (descending)
numbers.Sort((a, b) => b.CompareTo(a));
Console.WriteLine(string.Join(", ", numbers)); // 9, 8, 7, 6, 5, 4, 3, 2, 1

// Reverse: กลับลำดับ
numbers.Reverse();
Console.WriteLine(string.Join(", ", numbers)); // 1, 2, 3, 4, 5, 6, 7, 8, 9

// BinarySearch: ค้นหาแบบ Binary (list ต้องเรียงแล้ว)
int pos = numbers.BinarySearch(5); // 4 (index ของ 5)

// Sort กับ Custom Object
List<string> names = new List<string> { "Charlie", "Alice", "Bob" };
names.Sort(StringComparer.OrdinalIgnoreCase);
Console.WriteLine(string.Join(", ", names)); // Alice, Bob, Charlie

// Sort กับ Object โดยใช้ Comparison
List<(string Name, int Age)> people = new()
{
    ("Charlie", 30),
    ("Alice", 25),
    ("Bob", 35)
};
people.Sort((a, b) => a.Age.CompareTo(b.Age));
foreach (var p in people)
    Console.WriteLine($"{p.Name}: {p.Age}");
// Alice: 25
// Charlie: 30
// Bob: 35
```

### 2.7 การแปลง: ToArray, ConvertAll, ForEach

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };

// ToArray: แปลงเป็น Array
int[] arr = numbers.ToArray();

// ConvertAll: แปลง type ของแต่ละ element
List<string> strNums = numbers.ConvertAll(n => n.ToString());
List<double> doubled = numbers.ConvertAll(n => n * 2.0);

// ForEach: ทำงานกับแต่ละ element
numbers.ForEach(n => Console.Write($"{n} ")); // 1 2 3 4 5

// GetRange: ดึง sublist
List<int> sub = numbers.GetRange(1, 3); // index 1, จำนวน 3
Console.WriteLine(string.Join(", ", sub)); // 2, 3, 4
```

---

## 3. Dictionary\<TKey, TValue\>

`Dictionary<TKey, TValue>` เก็บคู่ Key-Value โดย Key ต้อง Unique และการค้นหาด้วย Key มีความเร็ว O(1) โดยเฉลี่ย

### 3.1 การสร้าง Dictionary

```csharp
// วิธีที่ 1: Empty dictionary
Dictionary<string, int> ages = new Dictionary<string, int>();

// วิธีที่ 2: Collection Initializer
Dictionary<string, string> capitals = new Dictionary<string, string>
{
    { "Thailand", "Bangkok" },
    { "Japan", "Tokyo" },
    { "UK", "London" }
};

// วิธีที่ 3: Index Initializer (C# 6+)
Dictionary<string, int> scores = new Dictionary<string, int>
{
    ["Alice"] = 95,
    ["Bob"] = 87,
    ["Charlie"] = 92
};

// วิธีที่ 4: Target-typed new (C# 9+)
Dictionary<int, string> idToName = new()
{
    [1] = "Alice",
    [2] = "Bob"
};
```

### 3.2 การเพิ่มและแก้ไข

```csharp
Dictionary<string, int> inventory = new Dictionary<string, int>();

// Add: เพิ่ม key-value (Exception ถ้า key ซ้ำ)
inventory.Add("Apple", 100);
inventory.Add("Banana", 50);
inventory.Add("Cherry", 75);

// Indexer: เพิ่มหรือแก้ไข (ไม่ throw Exception)
inventory["Apple"] = 120;     // แก้ไข Apple
inventory["Durian"] = 30;     // เพิ่มใหม่

// TryAdd: เพิ่มเฉพาะกรณีที่ไม่มี key อยู่แล้ว
bool added = inventory.TryAdd("Apple", 999); // false (มีอยู่แล้ว)
bool ok = inventory.TryAdd("Elderberry", 25); // true

Console.WriteLine($"Apple count: {inventory["Apple"]}"); // 120

// การเพิ่มค่าสะสม (pattern ที่ใช้บ่อย)
Dictionary<string, int> wordCount = new Dictionary<string, int>();
string[] words = { "the", "quick", "brown", "fox", "the", "quick", "the" };

foreach (string word in words)
{
    // วิธีที่ 1: ตรวจสอบก่อน
    if (wordCount.ContainsKey(word))
        wordCount[word]++;
    else
        wordCount[word] = 1;
}

// วิธีที่ 2: TryGetValue (ดีกว่า)
Dictionary<string, int> wordCount2 = new Dictionary<string, int>();
foreach (string word in words)
{
    wordCount2.TryGetValue(word, out int count);
    wordCount2[word] = count + 1;
}

// วิธีที่ 3: GetValueOrDefault (C# 8+)
Dictionary<string, int> wordCount3 = new Dictionary<string, int>();
foreach (string word in words)
{
    wordCount3[word] = wordCount3.GetValueOrDefault(word) + 1;
}

foreach (var kvp in wordCount3)
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
```

### 3.3 การลบ

```csharp
Dictionary<string, int> data = new Dictionary<string, int>
{
    ["A"] = 1, ["B"] = 2, ["C"] = 3
};

// Remove: ลบด้วย key
bool removed = data.Remove("B"); // true

// Remove with out value (C# 7+)
if (data.Remove("A", out int removedValue))
    Console.WriteLine($"Removed value: {removedValue}"); // 1

// Clear: ลบทั้งหมด
data.Clear();
Console.WriteLine($"Count: {data.Count}"); // 0
```

### 3.4 ContainsKey, ContainsValue, TryGetValue

```csharp
Dictionary<string, string> config = new Dictionary<string, string>
{
    ["host"] = "localhost",
    ["port"] = "5432",
    ["database"] = "mydb"
};

// ContainsKey: ตรวจสอบ key
bool hasHost = config.ContainsKey("host");     // true
bool hasUser = config.ContainsKey("username"); // false

// ContainsValue: ตรวจสอบ value (ช้ากว่า - O(n))
bool hasLocalhost = config.ContainsValue("localhost"); // true

// TryGetValue: ดีที่สุดสำหรับการอ่านค่า (ไม่ throw Exception)
if (config.TryGetValue("port", out string portValue))
{
    Console.WriteLine($"Port: {portValue}"); // 5432
}

// การเข้าถึงด้วย Indexer (throw KeyNotFoundException ถ้าไม่มี key)
try
{
    string value = config["nonexistent"]; // throws!
}
catch (KeyNotFoundException ex)
{
    Console.WriteLine("Key not found!");
}

// GetValueOrDefault: ค่า default ถ้าไม่มี key
string host = config.GetValueOrDefault("host", "unknown"); // localhost
string user = config.GetValueOrDefault("user", "admin");   // admin (default)
```

### 3.5 การวนซ้ำ (Iteration)

```csharp
Dictionary<string, int> scores = new Dictionary<string, int>
{
    ["Alice"] = 95,
    ["Bob"] = 87,
    ["Charlie"] = 92
};

// วนซ้ำ KeyValuePair
foreach (KeyValuePair<string, int> kvp in scores)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}

// Pattern Matching Deconstruction (C# 7+)
foreach (var (name, score) in scores)
{
    Console.WriteLine($"{name} scored {score}");
}

// วนซ้ำเฉพาะ Keys
foreach (string name in scores.Keys)
{
    Console.WriteLine(name);
}

// วนซ้ำเฉพาะ Values
foreach (int score in scores.Values)
{
    Console.WriteLine(score);
}

// Keys และ Values เป็น ICollection
ICollection<string> keyCollection = scores.Keys;
ICollection<int> valueCollection = scores.Values;

// เรียง Keys
var sortedKeys = scores.Keys.OrderBy(k => k).ToList();
foreach (string k in sortedKeys)
    Console.WriteLine($"{k}: {scores[k]}");
```

---

## 4. List vs Array เปรียบเทียบ

```csharp
// ===== Array =====
// ขนาดคงที่ หลังจากสร้างแล้วเปลี่ยนไม่ได้
int[] arr = new int[5];
arr[0] = 10;
arr[4] = 50;
// arr[5] = 60; // IndexOutOfRangeException!

// Array ดีสำหรับ:
// - ขนาดคงที่ที่รู้แน่นอน
// - ใช้หน่วยความจำน้อยกว่า
// - เข้าถึงด้วย index เร็ว
// - Mathematical operations (Matrix, vectors)

// ===== List<T> =====
// ขนาดยืดหยุ่น
List<int> list = new List<int>();
list.Add(10);
list.Add(20);
list.Add(30);
list.Remove(20);

// List ดีสำหรับ:
// - ไม่รู้จำนวนล่วงหน้า
// - ต้องเพิ่ม/ลบบ่อย
// - ต้องการ methods เพิ่มเติม (Find, Sort, ForEach)
// - ส่วนใหญ่ควรใช้ List<T>

// Performance Comparison
Console.WriteLine("=== Performance Comparison ===");
int size = 1_000_000;

// Array access - O(1), fast
int[] bigArr = new int[size];
var sw = System.Diagnostics.Stopwatch.StartNew();
for (int i = 0; i < size; i++) bigArr[i] = i;
sw.Stop();
Console.WriteLine($"Array fill: {sw.ElapsedMilliseconds}ms");

// List access - O(1), slightly slower due to overhead
List<int> bigList = new List<int>(size);
sw.Restart();
for (int i = 0; i < size; i++) bigList.Add(i);
sw.Stop();
Console.WriteLine($"List fill: {sw.ElapsedMilliseconds}ms");

// Conversion
int[] fromList = bigList.ToArray();
List<int> fromArr = new List<int>(bigArr);
```

---

## 5. Dictionary Patterns ที่นิยม

### 5.1 Grouping/Counting Pattern

```csharp
string[] fruits = { "Apple", "Banana", "Apple", "Cherry", "Banana", "Apple" };

// Count occurrences
Dictionary<string, int> counts = new Dictionary<string, int>();
foreach (string fruit in fruits)
{
    counts[fruit] = counts.GetValueOrDefault(fruit) + 1;
}

foreach (var (fruit, count) in counts.OrderByDescending(x => x.Value))
    Console.WriteLine($"{fruit}: {count}");
// Apple: 3
// Banana: 2
// Cherry: 1
```

### 5.2 Lookup/Cache Pattern

```csharp
// แทนที่ switch/if-else ด้วย Dictionary
Dictionary<string, string> errorMessages = new Dictionary<string, string>
{
    ["404"] = "Page Not Found",
    ["403"] = "Forbidden",
    ["500"] = "Internal Server Error",
    ["200"] = "OK"
};

string GetMessage(string code) =>
    errorMessages.GetValueOrDefault(code, "Unknown Error");

Console.WriteLine(GetMessage("404")); // Page Not Found
Console.WriteLine(GetMessage("999")); // Unknown Error
```

### 5.3 Inverse Dictionary Pattern

```csharp
Dictionary<string, int> nameToId = new Dictionary<string, int>
{
    ["Alice"] = 1,
    ["Bob"] = 2,
    ["Charlie"] = 3
};

// สร้าง inverse lookup
Dictionary<int, string> idToName = nameToId
    .ToDictionary(kvp => kvp.Value, kvp => kvp.Key);

Console.WriteLine(idToName[2]); // Bob
```

### 5.4 Dictionary of Lists Pattern

```csharp
// จัดกลุ่มนักเรียนตามห้องเรียน
Dictionary<string, List<string>> classrooms = new Dictionary<string, List<string>>();

void AddStudent(string room, string student)
{
    if (!classrooms.TryGetValue(room, out List<string>? students))
    {
        students = new List<string>();
        classrooms[room] = students;
    }
    students.Add(student);
}

AddStudent("A", "Alice");
AddStudent("A", "Alex");
AddStudent("B", "Bob");
AddStudent("B", "Betty");
AddStudent("B", "Brian");

foreach (var (room, students) in classrooms)
{
    Console.WriteLine($"Room {room}: {string.Join(", ", students)}");
}
// Room A: Alice, Alex
// Room B: Bob, Betty, Brian
```

---

## 6. โปรแกรมตัวอย่าง: Contact Book

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace ContactBook
{
    // Model
    public record Contact(string Name, string Phone, string Email, string Category);

    public class ContactManager
    {
        private readonly Dictionary<string, Contact> _contacts = new(StringComparer.OrdinalIgnoreCase);

        // เพิ่มผู้ติดต่อ
        public bool AddContact(Contact contact)
        {
            if (string.IsNullOrWhiteSpace(contact.Name))
                throw new ArgumentException("Name cannot be empty");

            return _contacts.TryAdd(contact.Name, contact);
        }

        // แก้ไขผู้ติดต่อ
        public bool UpdateContact(string name, Contact updatedContact)
        {
            if (!_contacts.ContainsKey(name))
                return false;

            _contacts[name] = updatedContact;
            return true;
        }

        // ลบผู้ติดต่อ
        public bool RemoveContact(string name)
        {
            return _contacts.Remove(name);
        }

        // ค้นหาด้วยชื่อ
        public Contact? FindByName(string name)
        {
            _contacts.TryGetValue(name, out Contact? contact);
            return contact;
        }

        // ค้นหาด้วย partial name
        public List<Contact> SearchByName(string searchTerm)
        {
            return _contacts.Values
                .Where(c => c.Name.Contains(searchTerm, StringComparison.OrdinalIgnoreCase))
                .OrderBy(c => c.Name)
                .ToList();
        }

        // ค้นหาด้วย category
        public List<Contact> GetByCategory(string category)
        {
            return _contacts.Values
                .Where(c => c.Category.Equals(category, StringComparison.OrdinalIgnoreCase))
                .OrderBy(c => c.Name)
                .ToList();
        }

        // แสดงทั้งหมด
        public List<Contact> GetAll()
        {
            return _contacts.Values.OrderBy(c => c.Name).ToList();
        }

        // สถิติ
        public Dictionary<string, int> GetCategoryStats()
        {
            return _contacts.Values
                .GroupBy(c => c.Category, StringComparer.OrdinalIgnoreCase)
                .ToDictionary(g => g.Key, g => g.Count());
        }

        public int Count => _contacts.Count;

        // Export as formatted string
        public string ExportAll()
        {
            var lines = _contacts.Values
                .OrderBy(c => c.Name)
                .Select(c => $"{c.Name}|{c.Phone}|{c.Email}|{c.Category}");
            return string.Join(Environment.NewLine, lines);
        }
    }

    class Program
    {
        static void Main()
        {
            var manager = new ContactManager();

            // เพิ่มผู้ติดต่อ
            Console.WriteLine("=== เพิ่มผู้ติดต่อ ===");
            var contacts = new[]
            {
                new Contact("Alice Johnson", "081-234-5678", "alice@email.com", "Family"),
                new Contact("Bob Smith", "089-876-5432", "bob@work.com", "Work"),
                new Contact("Charlie Brown", "090-111-2222", "charlie@email.com", "Friend"),
                new Contact("Diana Prince", "091-333-4444", "diana@work.com", "Work"),
                new Contact("Eve Wilson", "092-555-6666", "eve@email.com", "Friend"),
                new Contact("Frank Miller", "093-777-8888", "frank@email.com", "Family"),
            };

            foreach (var c in contacts)
            {
                bool added = manager.AddContact(c);
                Console.WriteLine($"Added {c.Name}: {added}");
            }

            // ลอง add ซ้ำ
            bool duplicate = manager.AddContact(new Contact("Alice Johnson", "000", "new@email.com", "Work"));
            Console.WriteLine($"Add duplicate Alice: {duplicate}"); // false

            Console.WriteLine($"\nTotal contacts: {manager.Count}");

            // ค้นหา
            Console.WriteLine("\n=== ค้นหา ===");
            var alice = manager.FindByName("alice johnson");
            if (alice != null)
                Console.WriteLine($"Found: {alice.Name} - {alice.Phone} ({alice.Category})");

            Console.WriteLine("\nSearch 'son':");
            var results = manager.SearchByName("son");
            foreach (var c in results)
                Console.WriteLine($"  {c.Name}");

            // กรองตาม category
            Console.WriteLine("\nWork contacts:");
            var workContacts = manager.GetByCategory("Work");
            foreach (var c in workContacts)
                Console.WriteLine($"  {c.Name} - {c.Email}");

            // แก้ไข
            Console.WriteLine("\n=== แก้ไข ===");
            bool updated = manager.UpdateContact("Bob Smith",
                new Contact("Bob Smith", "088-999-0000", "bob.new@work.com", "Work"));
            Console.WriteLine($"Updated Bob: {updated}");

            var bob = manager.FindByName("Bob Smith");
            Console.WriteLine($"New phone: {bob?.Phone}");

            // สถิติ
            Console.WriteLine("\n=== สถิติตาม Category ===");
            var stats = manager.GetCategoryStats();
            foreach (var (category, count) in stats.OrderByDescending(x => x.Value))
                Console.WriteLine($"  {category}: {count} contacts");

            // แสดงทั้งหมด
            Console.WriteLine("\n=== รายชื่อทั้งหมด ===");
            foreach (var c in manager.GetAll())
                Console.WriteLine($"  [{c.Category}] {c.Name} - {c.Phone}");

            // ลบ
            Console.WriteLine("\n=== ลบผู้ติดต่อ ===");
            bool removed = manager.RemoveContact("Charlie Brown");
            Console.WriteLine($"Removed Charlie: {removed}");
            Console.WriteLine($"Total contacts after removal: {manager.Count}");

            // Export
            Console.WriteLine("\n=== Export ===");
            Console.WriteLine(manager.ExportAll());
        }
    }
}
```

---

## Exercises

### Exercise 1: Inventory System
สร้างระบบ inventory ด้วย `List<Product>` ที่มี:
- เพิ่ม, ลบ, ค้นหาสินค้า
- เรียงตาม ราคา/ชื่อ
- กรองสินค้าที่ stock น้อยกว่า threshold

```csharp
// โครงสร้าง
public record Product(string Name, decimal Price, int Stock);

List<Product> inventory = new();
// TODO: implement Add, Remove, Search, Sort, Filter
```

### Exercise 2: Word Frequency Counter
เขียนโปรแกรมที่:
- รับ text input จาก user
- นับความถี่ของแต่ละคำ
- แสดง top 5 คำที่ใช้บ่อยที่สุด

```csharp
string text = "the quick brown fox jumps over the lazy dog the fox";
// TODO: use Dictionary<string, int> to count words
// TODO: display top 5 most frequent words
```

### Exercise 3: Phone Directory
สร้าง Phone Directory ที่:
- เก็บ `Dictionary<string, List<string>>` (ชื่อ -> รายการเบอร์)
- เพิ่ม/ลบเบอร์โทรของแต่ละคนได้
- ค้นหาชื่อจากเบอร์โทร (Reverse lookup)

---

## สรุป

- ✅ `List<T>` คือ Dynamic Array ที่ใช้บ่อยที่สุด มี methods ครบครัน
- ✅ `Add/AddRange/Insert` สำหรับเพิ่มข้อมูล, `Remove/RemoveAt/RemoveAll` สำหรับลบ
- ✅ `Find/FindAll/FindIndex` สำหรับค้นหาด้วยเงื่อนไข
- ✅ `Sort` ด้วย default หรือ custom Comparison
- ✅ `Dictionary<K,V>` เก็บ Key-Value, Key ต้อง Unique, การค้นหา O(1)
- ✅ `TryGetValue` ดีกว่า Indexer เพราะไม่ throw Exception
- ✅ `GetValueOrDefault` สะดวกสำหรับกรณีไม่มี key
- ✅ วน Dictionary ด้วย `foreach (var (key, value) in dict)`
- ✅ List ยืดหยุ่นกว่า Array แต่ใช้หน่วยความจำมากกว่าเล็กน้อย
- ✅ Dictionary เหมาะสำหรับ Lookup, Grouping, Counting patterns

## Part ถัดไป
**Part 022** จะพูดถึง Collections อื่นๆ ได้แก่ Queue\<T\>, Stack\<T\>, HashSet\<T\>, SortedList, SortedDictionary และ LinkedList\<T\>

---
*Part 021/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
