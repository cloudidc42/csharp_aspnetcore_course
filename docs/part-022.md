# Part 022: Collections - Queue, Stack, HashSet, LinkedList

## เนื้อหาใน Part นี้
- Queue\<T\>: FIFO, Enqueue, Dequeue, Peek
- Stack\<T\>: LIFO, Push, Pop, Peek
- HashSet\<T\>: Unique elements, Set operations
- SortedList\<TKey, TValue\> และ SortedDictionary
- LinkedList\<T\>: Doubly-linked list
- เปรียบเทียบ Collection แต่ละประเภท
- โปรแกรมตัวอย่าง: Task Scheduler ด้วย Queue

---

## 1. Queue\<T\> - คิว (FIFO: First In, First Out)

Queue จัดการข้อมูลแบบ "เข้าก่อนออกก่อน" เหมือนคิวรอบริการ

```csharp
using System.Collections.Generic;

Queue<string> queue = new Queue<string>();

// Enqueue: เพิ่มท้ายคิว
queue.Enqueue("Task A");
queue.Enqueue("Task B");
queue.Enqueue("Task C");

Console.WriteLine($"Count: {queue.Count}"); // 3

// Peek: ดูข้างหน้าโดยไม่เอาออก
string first = queue.Peek();
Console.WriteLine($"Next: {first}"); // Task A
Console.WriteLine($"Count: {queue.Count}"); // ยังคง 3

// Dequeue: เอาออกจากหัวคิว
string task = queue.Dequeue();
Console.WriteLine($"Processing: {task}"); // Task A
Console.WriteLine($"Count: {queue.Count}"); // 2

// TryDequeue: ปลอดภัยกว่า - ไม่ throw ถ้าว่าง
while (queue.TryDequeue(out string? item))
{
    Console.WriteLine($"Processing: {item}");
}

// TryPeek
if (queue.TryPeek(out string? next))
    Console.WriteLine($"Next item: {next}");
else
    Console.WriteLine("Queue is empty");
```

### 1.1 Queue สำหรับ Breadth-First Search

```csharp
// BFS ด้วย Queue
void BFS(Dictionary<int, List<int>> graph, int start)
{
    Queue<int> queue = new Queue<int>();
    HashSet<int> visited = new HashSet<int>();

    queue.Enqueue(start);
    visited.Add(start);

    while (queue.Count > 0)
    {
        int node = queue.Dequeue();
        Console.Write($"{node} ");

        if (graph.TryGetValue(node, out List<int>? neighbors))
        {
            foreach (int neighbor in neighbors)
            {
                if (visited.Add(neighbor)) // Add คืน false ถ้ามีอยู่แล้ว
                    queue.Enqueue(neighbor);
            }
        }
    }
    Console.WriteLine();
}

var graph = new Dictionary<int, List<int>>
{
    [1] = new() { 2, 3 },
    [2] = new() { 4, 5 },
    [3] = new() { 6 },
    [4] = new(),
    [5] = new(),
    [6] = new()
};

BFS(graph, 1); // 1 2 3 4 5 6
```

### 1.2 Priority Queue (C# 6+ / .NET 6+)

```csharp
// PriorityQueue<TElement, TPriority>
PriorityQueue<string, int> pq = new PriorityQueue<string, int>();

// Enqueue ด้วย priority (ค่าน้อย = priority สูง)
pq.Enqueue("Low priority task", 10);
pq.Enqueue("High priority task", 1);
pq.Enqueue("Medium priority task", 5);
pq.Enqueue("Critical task", 0);

while (pq.TryDequeue(out string? task, out int priority))
{
    Console.WriteLine($"[Priority {priority}] {task}");
}
// [Priority 0] Critical task
// [Priority 1] High priority task
// [Priority 5] Medium priority task
// [Priority 10] Low priority task
```

---

## 2. Stack\<T\> - สแตก (LIFO: Last In, First Out)

Stack จัดการข้อมูลแบบ "เข้าหลังออกก่อน" เหมือนกองจาน

```csharp
Stack<string> stack = new Stack<string>();

// Push: วางบนสุด
stack.Push("First");
stack.Push("Second");
stack.Push("Third");

Console.WriteLine($"Count: {stack.Count}"); // 3

// Peek: ดูบนสุดโดยไม่เอาออก
string top = stack.Peek();
Console.WriteLine($"Top: {top}"); // Third

// Pop: เอาออกจากบนสุด
string item = stack.Pop();
Console.WriteLine($"Popped: {item}"); // Third
Console.WriteLine($"Count: {stack.Count}"); // 2

// TryPop และ TryPeek
while (stack.TryPop(out string? val))
    Console.WriteLine($"Popped: {val}");

// Contains: ตรวจสอบ (O(n))
Stack<int> nums = new Stack<int>(new[] { 1, 2, 3, 4, 5 });
bool has3 = nums.Contains(3); // true

// ToArray: แปลงเป็น array (บนสุดก่อน)
int[] arr = nums.ToArray(); // [5, 4, 3, 2, 1]
```

### 2.1 Stack สำหรับ Undo/Redo

```csharp
public class TextEditor
{
    private string _text = "";
    private Stack<string> _undoStack = new Stack<string>();
    private Stack<string> _redoStack = new Stack<string>();

    public string Text => _text;

    public void Type(string text)
    {
        _undoStack.Push(_text); // บันทึก state เดิม
        _redoStack.Clear();     // clear redo history
        _text += text;
        Console.WriteLine($"Typed: '{text}' -> Text: '{_text}'");
    }

    public void Undo()
    {
        if (_undoStack.Count == 0)
        {
            Console.WriteLine("Nothing to undo");
            return;
        }
        _redoStack.Push(_text);
        _text = _undoStack.Pop();
        Console.WriteLine($"Undo -> Text: '{_text}'");
    }

    public void Redo()
    {
        if (_redoStack.Count == 0)
        {
            Console.WriteLine("Nothing to redo");
            return;
        }
        _undoStack.Push(_text);
        _text = _redoStack.Pop();
        Console.WriteLine($"Redo -> Text: '{_text}'");
    }
}

// ทดสอบ
var editor = new TextEditor();
editor.Type("Hello");
editor.Type(" World");
editor.Type("!");
editor.Undo();    // ลบ "!"
editor.Undo();    // ลบ " World"
editor.Redo();    // คืน " World"
```

### 2.2 Stack สำหรับ Expression Evaluation

```csharp
// ตรวจสอบ Bracket Matching
bool IsBalanced(string expression)
{
    Stack<char> stack = new Stack<char>();
    Dictionary<char, char> pairs = new() { [')'] = '(', [']'] = '[', ['}'] = '{' };

    foreach (char ch in expression)
    {
        if ("([{".Contains(ch))
            stack.Push(ch);
        else if (")]}"  .Contains(ch))
        {
            if (stack.Count == 0 || stack.Pop() != pairs[ch])
                return false;
        }
    }
    return stack.Count == 0;
}

Console.WriteLine(IsBalanced("({[]})")); // true
Console.WriteLine(IsBalanced("({[})]")); // false
Console.WriteLine(IsBalanced("((()))")); // true
```

---

## 3. HashSet\<T\> - เซตไม่ซ้ำ

HashSet เก็บ element ที่ไม่ซ้ำกัน การค้นหา/เพิ่ม/ลบเร็วมาก O(1)

```csharp
HashSet<string> set = new HashSet<string>();

// Add: คืน true ถ้าเพิ่มได้, false ถ้าซ้ำ
bool a1 = set.Add("Apple");  // true
bool a2 = set.Add("Banana"); // true
bool a3 = set.Add("Apple");  // false (ซ้ำ)

Console.WriteLine($"Count: {set.Count}"); // 2

// Remove: ลบ
set.Remove("Banana");

// Contains: ตรวจสอบ (เร็วมาก O(1))
bool has = set.Contains("Apple"); // true

// ตรวจสอบซ้ำด้วย HashSet (เร็วกว่า List มาก)
HashSet<int> visited = new HashSet<int>();
int[] data = { 1, 2, 3, 2, 1, 4, 5, 4 };
foreach (int n in data)
{
    if (visited.Add(n)) // เพิ่มและตรวจสอบพร้อมกัน
        Console.Write($"{n} "); // 1 2 3 4 5
}
```

### 3.1 Set Operations (Union, Intersect, Except)

```csharp
HashSet<int> setA = new HashSet<int> { 1, 2, 3, 4, 5 };
HashSet<int> setB = new HashSet<int> { 3, 4, 5, 6, 7 };

// Union: รวม (A ∪ B) - แก้ไข setA
HashSet<int> union = new HashSet<int>(setA);
union.UnionWith(setB);
Console.WriteLine("Union: " + string.Join(", ", union.OrderBy(x => x)));
// 1, 2, 3, 4, 5, 6, 7

// Intersect: ตัดกัน (A ∩ B) - เฉพาะที่มีในทั้งสอง
HashSet<int> intersect = new HashSet<int>(setA);
intersect.IntersectWith(setB);
Console.WriteLine("Intersect: " + string.Join(", ", intersect.OrderBy(x => x)));
// 3, 4, 5

// Except: ผลต่าง (A - B) - มีใน A แต่ไม่มีใน B
HashSet<int> except = new HashSet<int>(setA);
except.ExceptWith(setB);
Console.WriteLine("Except: " + string.Join(", ", except.OrderBy(x => x)));
// 1, 2

// SymmetricExcept: มีใน A หรือ B แต่ไม่ใช่ทั้งสอง (A △ B)
HashSet<int> symExcept = new HashSet<int>(setA);
symExcept.SymmetricExceptWith(setB);
Console.WriteLine("SymmetricExcept: " + string.Join(", ", symExcept.OrderBy(x => x)));
// 1, 2, 6, 7

// ตรวจสอบความสัมพันธ์
bool isSubset = new HashSet<int> { 3, 4 }.IsSubsetOf(setA);   // true
bool isSuperset = setA.IsSupersetOf(new HashSet<int> { 1, 2 }); // true
bool overlaps = setA.Overlaps(setB); // true (มีตัดกัน)
bool setEquals = setA.SetEquals(new HashSet<int> { 5, 4, 3, 2, 1 }); // true
```

### 3.2 HashSet ตัด Duplicate

```csharp
// ลบ duplicate จาก List
List<int> listWithDups = new List<int> { 1, 2, 3, 2, 4, 1, 5, 3 };
List<int> unique = new HashSet<int>(listWithDups).ToList();
Console.WriteLine(string.Join(", ", unique)); // 1, 2, 3, 4, 5

// หา duplicate
int[] arr = { 1, 2, 3, 2, 4, 1 };
HashSet<int> seen = new HashSet<int>();
List<int> duplicates = arr.Where(x => !seen.Add(x)).ToList();
Console.WriteLine("Duplicates: " + string.Join(", ", duplicates)); // 2, 1

// ตรวจสอบทุก element ต่างกัน
bool allUnique = arr.Length == new HashSet<int>(arr).Count; // false
```

---

## 4. SortedList\<TKey, TValue\>

SortedList เก็บข้อมูล Key-Value โดยเรียง Key อัตโนมัติ

```csharp
SortedList<string, int> sortedList = new SortedList<string, int>();

sortedList.Add("Charlie", 30);
sortedList.Add("Alice", 25);
sortedList.Add("Bob", 28);

// Key จะเรียงอัตโนมัติ
foreach (var (name, age) in sortedList)
    Console.WriteLine($"{name}: {age}");
// Alice: 25
// Bob: 28
// Charlie: 30

// เข้าถึงด้วย Index
Console.WriteLine($"First key: {sortedList.Keys[0]}");   // Alice
Console.WriteLine($"First value: {sortedList.Values[0]}"); // 25

// IndexOfKey, IndexOfValue
int idx = sortedList.IndexOfKey("Bob"); // 1

// SortedList vs SortedDictionary:
// SortedList: ใช้ Array ภายใน - memory น้อยกว่า, random access เร็ว, insert/delete ช้า
// SortedDictionary: ใช้ BST ภายใน - insert/delete เร็วกว่า, memory มากกว่า
```

---

## 5. SortedDictionary\<TKey, TValue\>

```csharp
SortedDictionary<int, string> sortedDict = new SortedDictionary<int, string>
{
    [3] = "Three",
    [1] = "One",
    [4] = "Four",
    [2] = "Two"
};

// Key เรียง ascending อัตโนมัติ
foreach (var (key, value) in sortedDict)
    Console.WriteLine($"{key}: {value}");
// 1: One
// 2: Two
// 3: Three
// 4: Four

// หา min/max
(int firstKey, string firstVal) = (sortedDict.Keys.Min(), sortedDict[sortedDict.Keys.Min()]);
Console.WriteLine($"Min key: {firstKey} = {firstVal}"); // 1 = One

// SortedDictionary ด้วย Custom Comparer
SortedDictionary<string, int> caseInsensitive =
    new SortedDictionary<string, int>(StringComparer.OrdinalIgnoreCase)
    {
        ["Banana"] = 2,
        ["apple"] = 1,
        ["cherry"] = 3
    };

foreach (var (k, v) in caseInsensitive)
    Console.WriteLine($"{k}: {v}");
// apple: 1
// Banana: 2
// cherry: 3
```

---

## 6. LinkedList\<T\>

LinkedList เป็น Doubly-Linked List เหมาะสำหรับการ insert/remove ที่กลางลิสต์บ่อยๆ

```csharp
LinkedList<string> linkedList = new LinkedList<string>();

// AddFirst: เพิ่มหัว
LinkedListNode<string> nodeA = linkedList.AddFirst("A");
// AddLast: เพิ่มท้าย
LinkedListNode<string> nodeC = linkedList.AddLast("C");
// AddBefore: เพิ่มก่อน node
LinkedListNode<string> nodeB = linkedList.AddBefore(nodeC, "B");
// AddAfter: เพิ่มหลัง node
linkedList.AddAfter(nodeC, "D");

// วนซ้ำ
foreach (string item in linkedList)
    Console.Write($"{item} "); // A B C D

// Navigation
LinkedListNode<string>? current = linkedList.First;
while (current != null)
{
    Console.Write($"{current.Value} ");
    current = current.Next;
}

// วนจากท้าย
current = linkedList.Last;
while (current != null)
{
    Console.Write($"{current.Value} ");
    current = current.Previous;
}
// D C B A

// Remove
linkedList.Remove(nodeB);     // ลบ node B
linkedList.RemoveFirst();     // ลบหัว (A)
linkedList.RemoveLast();      // ลบท้าย (D)
Console.WriteLine(linkedList.First?.Value); // C

// Find
LinkedListNode<string>? found = linkedList.Find("C");
Console.WriteLine(found?.Value); // C

// LinkedList ดีกว่า List เมื่อ:
// - Insert/Delete กลาง list บ่อย
// - ไม่ต้องการ random access (List[i])
// - ทำ Deque (double-ended queue)
```

### 6.1 LinkedList เป็น LRU Cache

```csharp
// LRU Cache ด้วย LinkedList + Dictionary
public class LRUCache<TKey, TValue> where TKey : notnull
{
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _cache;
    private readonly LinkedList<(TKey Key, TValue Value)> _list;

    public LRUCache(int capacity)
    {
        _capacity = capacity;
        _cache = new Dictionary<TKey, LinkedListNode<(TKey, TValue)>>(capacity);
        _list = new LinkedList<(TKey, TValue)>();
    }

    public TValue? Get(TKey key)
    {
        if (_cache.TryGetValue(key, out var node))
        {
            // Move to front (recently used)
            _list.Remove(node);
            _list.AddFirst(node);
            return node.Value.Value;
        }
        return default;
    }

    public void Put(TKey key, TValue value)
    {
        if (_cache.TryGetValue(key, out var existing))
        {
            _list.Remove(existing);
            _cache.Remove(key);
        }
        else if (_cache.Count >= _capacity)
        {
            // Remove LRU (last item)
            var lru = _list.Last!;
            _cache.Remove(lru.Value.Key);
            _list.RemoveLast();
        }

        var node = _list.AddFirst((key, value));
        _cache[key] = node;
    }

    public int Count => _cache.Count;
}

// ทดสอบ LRU Cache
var cache = new LRUCache<int, string>(3);
cache.Put(1, "One");
cache.Put(2, "Two");
cache.Put(3, "Three");
Console.WriteLine(cache.Get(1));  // "One" - 1 is now most recent
cache.Put(4, "Four");             // evicts 2 (least recently used)
Console.WriteLine(cache.Get(2));  // null (evicted)
Console.WriteLine(cache.Get(3));  // "Three"
```

---

## 7. เปรียบเทียบ Collections

| Collection | Access | Insert | Delete | Duplicate | Order |
|---|---|---|---|---|---|
| `List<T>` | O(1) index | O(1) end, O(n) mid | O(n) | Yes | Insertion |
| `Queue<T>` | O(1) front | O(1) | O(1) | Yes | FIFO |
| `Stack<T>` | O(1) top | O(1) | O(1) | Yes | LIFO |
| `HashSet<T>` | O(1) | O(1) | O(1) | No | None |
| `SortedSet<T>` | O(log n) | O(log n) | O(log n) | No | Sorted |
| `Dictionary<K,V>` | O(1) | O(1) | O(1) | Key unique | None |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | O(log n) | Key unique | Sorted |
| `SortedList<K,V>` | O(log n) | O(n) | O(n) | Key unique | Sorted |
| `LinkedList<T>` | O(n) | O(1) | O(1) | Yes | Insertion |

```csharp
// เลือก Collection ที่เหมาะสม
// 1. ต้องการ unique elements เร็ว -> HashSet<T>
// 2. ต้องการ sorted unique -> SortedSet<T>
// 3. ต้องการ FIFO -> Queue<T>
// 4. ต้องการ LIFO -> Stack<T>
// 5. ต้องการ key-value, เร็ว -> Dictionary<K,V>
// 6. ต้องการ key-value, เรียง -> SortedDictionary<K,V>
// 7. ต้องการ list ทั่วไป -> List<T>
// 8. ต้องการ insert กลาง list บ่อย -> LinkedList<T>
```

---

## 8. โปรแกรมตัวอย่าง: Task Scheduler ด้วย Queue

```csharp
using System;
using System.Collections.Generic;
using System.Threading;

namespace TaskScheduler
{
    public enum TaskPriority { Low = 3, Normal = 2, High = 1, Critical = 0 }
    public enum TaskStatus { Pending, Running, Completed, Failed }

    public class ScheduledTask
    {
        public int Id { get; init; }
        public string Name { get; init; } = "";
        public string Description { get; init; } = "";
        public TaskPriority Priority { get; init; }
        public TaskStatus Status { get; set; }
        public DateTime CreatedAt { get; init; }
        public DateTime? StartedAt { get; set; }
        public DateTime? CompletedAt { get; set; }
        public int DurationMs { get; init; } // simulated duration

        public override string ToString() =>
            $"[{Id}] {Name} ({Priority}) - {Status}";
    }

    public class TaskScheduler
    {
        private readonly PriorityQueue<ScheduledTask, int> _pendingQueue;
        private readonly Queue<ScheduledTask> _completedQueue;
        private readonly Stack<ScheduledTask> _undoStack;
        private readonly HashSet<int> _processedIds;
        private int _nextId = 1;
        private bool _isRunning = false;

        public TaskScheduler()
        {
            _pendingQueue = new PriorityQueue<ScheduledTask, int>();
            _completedQueue = new Queue<ScheduledTask>();
            _undoStack = new Stack<ScheduledTask>();
            _processedIds = new HashSet<int>();
        }

        public ScheduledTask AddTask(
            string name,
            string description,
            TaskPriority priority = TaskPriority.Normal,
            int durationMs = 500)
        {
            var task = new ScheduledTask
            {
                Id = _nextId++,
                Name = name,
                Description = description,
                Priority = priority,
                Status = TaskStatus.Pending,
                CreatedAt = DateTime.Now,
                DurationMs = durationMs
            };

            _pendingQueue.Enqueue(task, (int)priority);
            Console.WriteLine($"Added: {task}");
            return task;
        }

        public void RemoveLastAdded()
        {
            // ใช้ Stack ทำ undo
            if (_undoStack.TryPop(out ScheduledTask? last))
                Console.WriteLine($"Removed task: {last.Name}");
            else
                Console.WriteLine("No tasks to undo");
        }

        public void ProcessAll()
        {
            Console.WriteLine("\n=== Starting Task Processor ===");
            _isRunning = true;

            while (_pendingQueue.TryDequeue(out ScheduledTask? task, out int priority))
            {
                if (_processedIds.Contains(task.Id))
                {
                    Console.WriteLine($"Skip already processed: {task.Name}");
                    continue;
                }

                // Simulate processing
                task.Status = TaskStatus.Running;
                task.StartedAt = DateTime.Now;
                Console.WriteLine($"\nRunning: {task}");
                Console.WriteLine($"  Description: {task.Description}");

                // Simulate work (in real app, this would be actual work)
                Thread.Sleep(Math.Min(task.DurationMs, 100)); // cap at 100ms for demo

                // Randomly fail some tasks for demo
                bool success = task.Id % 5 != 0; // every 5th task fails
                task.Status = success ? TaskStatus.Completed : TaskStatus.Failed;
                task.CompletedAt = DateTime.Now;

                _processedIds.Add(task.Id);
                _completedQueue.Enqueue(task);
                _undoStack.Push(task);

                Console.WriteLine($"  Result: {task.Status} " +
                    $"(Duration: {(task.CompletedAt - task.StartedAt)?.TotalMilliseconds:F0}ms)");
            }

            _isRunning = false;
            Console.WriteLine("\n=== All tasks processed ===");
        }

        public void ShowSummary()
        {
            Console.WriteLine("\n=== Task Summary ===");

            List<ScheduledTask> completed = new List<ScheduledTask>(_completedQueue);
            int total = completed.Count;
            int success = completed.Count(t => t.Status == TaskStatus.Completed);
            int failed = completed.Count(t => t.Status == TaskStatus.Failed);

            Console.WriteLine($"Total processed: {total}");
            Console.WriteLine($"Successful: {success}");
            Console.WriteLine($"Failed: {failed}");
            Console.WriteLine($"Pending: {_pendingQueue.Count}");

            // Group by priority
            var byPriority = completed
                .GroupBy(t => t.Priority)
                .OrderBy(g => (int)g.Key);

            Console.WriteLine("\nBreakdown by priority:");
            foreach (var group in byPriority)
            {
                int cnt = group.Count();
                int ok = group.Count(t => t.Status == TaskStatus.Completed);
                Console.WriteLine($"  {group.Key}: {cnt} tasks, {ok} succeeded");
            }

            // Completed task history (from Queue)
            Console.WriteLine("\nRecent completions (last 3):");
            foreach (var task in completed.TakeLast(3))
                Console.WriteLine($"  {task}");
        }

        public void ShowRecentProcessed(int count = 3)
        {
            Console.WriteLine($"\n=== Last {count} processed (Stack) ===");
            Stack<ScheduledTask> tempStack = new Stack<ScheduledTask>(_undoStack);
            int shown = 0;
            while (tempStack.TryPop(out ScheduledTask? task) && shown < count)
            {
                Console.WriteLine($"  {task}");
                shown++;
            }
        }
    }

    class Program
    {
        static void Main()
        {
            var scheduler = new TaskScheduler();

            // เพิ่ม tasks ด้วย priority ต่างกัน
            scheduler.AddTask("Send email notification", "Notify users", TaskPriority.Low, 200);
            scheduler.AddTask("Process payment", "Handle payment", TaskPriority.Critical, 300);
            scheduler.AddTask("Update database", "Sync records", TaskPriority.High, 400);
            scheduler.AddTask("Generate report", "Daily report", TaskPriority.Normal, 250);
            scheduler.AddTask("Backup data", "Nightly backup", TaskPriority.Low, 600);
            scheduler.AddTask("Send alert", "Critical alert", TaskPriority.Critical, 100);
            scheduler.AddTask("Cache refresh", "Update cache", TaskPriority.High, 150);
            scheduler.AddTask("Cleanup logs", "Remove old logs", TaskPriority.Low, 200);
            scheduler.AddTask("User sync", "Sync user data", TaskPriority.Normal, 300);
            scheduler.AddTask("Health check", "System health", TaskPriority.High, 50);

            // ประมวลผล (Critical ก่อน, Low สุดท้าย)
            scheduler.ProcessAll();

            // แสดงสรุป
            scheduler.ShowSummary();
            scheduler.ShowRecentProcessed(5);
        }
    }
}
```

---

## Exercises

### Exercise 1: Browser History
สร้าง Browser History ด้วย Stack:
- `Navigate(url)`: เปิดหน้าใหม่
- `Back()`: ย้อนกลับ
- `Forward()`: ไปหน้าถัดไป
- แสดง current page

```csharp
public class BrowserHistory
{
    private Stack<string> _backStack = new Stack<string>();
    private Stack<string> _forwardStack = new Stack<string>();
    private string _current = "about:blank";

    public void Navigate(string url) { /* TODO */ }
    public void Back() { /* TODO */ }
    public void Forward() { /* TODO */ }
    public string Current => _current;
}
```

### Exercise 2: Unique Word Set
อ่านไฟล์ text และ:
- เก็บคำทั้งหมดใน HashSet
- เปรียบเทียบคำใน 2 ไฟล์ว่ามีคำใดร่วมกัน (Intersect)
- หาคำที่มีในไฟล์แรกแต่ไม่มีในไฟล์ที่สอง (Except)

### Exercise 3: Sorted Student Records
ใช้ SortedDictionary เก็บข้อมูลนักเรียน (Student ID -> Name):
- เรียง ID อัตโนมัติ
- เพิ่ม/ลบนักเรียน
- แสดงรายชื่อตามลำดับ ID

---

## สรุป

- ✅ `Queue<T>` คือ FIFO, ใช้ Enqueue/Dequeue, เหมาะสำหรับ task processing, BFS
- ✅ `PriorityQueue<T, P>` เหมือน Queue แต่เรียงตาม priority
- ✅ `Stack<T>` คือ LIFO, ใช้ Push/Pop, เหมาะสำหรับ Undo, DFS, bracket matching
- ✅ `HashSet<T>` เก็บ unique values, เร็วมาก O(1), รองรับ Set operations
- ✅ `SortedSet<T>` เหมือน HashSet แต่เรียงลำดับ
- ✅ `SortedList<K,V>` ใช้ Array, memory น้อย, random access เร็ว
- ✅ `SortedDictionary<K,V>` ใช้ BST, insert/delete เร็วกว่า SortedList
- ✅ `LinkedList<T>` เป็น Doubly-linked list, insert/delete กลางลิสต์เร็ว O(1)
- ✅ เลือก Collection ตามลักษณะการใช้งาน: FIFO, LIFO, Unique, Sorted, Random access

## Part ถัดไป
**Part 023** จะพูดถึง Generics การสร้าง Generic class, method, interface, constraints และ Covariance/Contravariance

---
*Part 022/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
