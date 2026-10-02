# Part 027: File I/O

## เนื้อหาใน Part นี้
- File class: ReadAllText, WriteAllText, ReadAllLines
- FileStream และ Buffered I/O
- StreamReader และ StreamWriter
- Directory operations
- Path class
- FileInfo, DirectoryInfo
- JSON Serialization ด้วย System.Text.Json
- CSV Processing
- โปรแกรมตัวอย่าง: Student Records in File

---

## 1. File Class - การดำเนินการพื้นฐาน

```csharp
using System;
using System.IO;

// ===== การเขียนไฟล์ =====
string content = "Hello, File I/O!\nLine 2\nLine 3";
string path = "example.txt";

// WriteAllText: เขียนทับ (หรือสร้างใหม่)
File.WriteAllText(path, content);

// AppendAllText: เพิ่มท้ายไฟล์
File.AppendAllText(path, "\nLine 4 (appended)");

// WriteAllLines: เขียนหลายบรรทัด (แต่ละ string เป็น 1 บรรทัด)
string[] lines = { "Apple", "Banana", "Cherry" };
File.WriteAllLines("fruits.txt", lines);

// ===== การอ่านไฟล์ =====
// ReadAllText: อ่านทั้งไฟล์เป็น string เดียว
string text = File.ReadAllText(path);
Console.WriteLine(text);

// ReadAllLines: อ่านเป็น string[]
string[] readLines = File.ReadAllLines(path);
foreach (string line in readLines)
    Console.WriteLine($"  > {line}");

// ReadAllBytes: อ่านเป็น byte[]
byte[] bytes = File.ReadAllBytes(path);
Console.WriteLine($"File size: {bytes.Length} bytes");

// ReadLines: Lazy - ดีกว่าสำหรับไฟล์ขนาดใหญ่
foreach (string line in File.ReadLines("large_file.txt"))
{
    if (line.StartsWith("ERROR"))
        Console.WriteLine($"Found error: {line}");
    // อ่านทีละบรรทัด ไม่โหลดทั้งไฟล์
}

// ===== File Operations =====
// Exists
bool exists = File.Exists(path);
Console.WriteLine($"File exists: {exists}");

// Copy
File.Copy(path, "example_backup.txt", overwrite: true);

// Move/Rename
File.Move("example_backup.txt", "example_renamed.txt");

// Delete
if (File.Exists("example_renamed.txt"))
    File.Delete("example_renamed.txt");

// GetAttributes
FileAttributes attrs = File.GetAttributes(path);
bool isReadOnly = (attrs & FileAttributes.ReadOnly) != 0;

// GetCreationTime, GetLastWriteTime
DateTime created = File.GetCreationTime(path);
DateTime modified = File.GetLastWriteTime(path);
Console.WriteLine($"Created: {created}, Modified: {modified}");

// Encoding
using System.Text;
File.WriteAllText("utf8.txt", "สวัสดีครับ", Encoding.UTF8);
string thai = File.ReadAllText("utf8.txt", Encoding.UTF8);
```

---

## 2. FileStream - Low-level I/O

```csharp
// FileStream: ควบคุม access mode ได้ละเอียดกว่า
string filePath = "data.bin";

// เขียนด้วย FileStream
using (var fs = new FileStream(filePath, FileMode.Create, FileAccess.Write))
{
    byte[] data = { 72, 101, 108, 108, 111 }; // "Hello" in ASCII
    fs.Write(data, 0, data.Length);
    fs.WriteByte(33); // '!'
}

// อ่านด้วย FileStream
using (var fs = new FileStream(filePath, FileMode.Open, FileAccess.Read))
{
    byte[] buffer = new byte[fs.Length];
    int bytesRead = fs.Read(buffer, 0, buffer.Length);
    string result = System.Text.Encoding.ASCII.GetString(buffer, 0, bytesRead);
    Console.WriteLine(result); // Hello!
}

// FileMode options:
// Create: สร้างใหม่ (ทับถ้ามีอยู่)
// CreateNew: สร้างใหม่ (throw ถ้ามีอยู่)
// Open: เปิดไฟล์ที่มีอยู่ (throw ถ้าไม่มี)
// OpenOrCreate: เปิดถ้ามี, สร้างใหม่ถ้าไม่มี
// Append: เพิ่มท้าย (สร้างใหม่ถ้าไม่มี)
// Truncate: เปิดและล้างเนื้อหา

// FileShare สำหรับ multi-process access
using var fs1 = new FileStream(filePath, FileMode.Open,
    FileAccess.Read, FileShare.Read); // หลายๆ process อ่านพร้อมกันได้

// Async I/O
async Task WriteAsync(string path, string data)
{
    byte[] bytes = System.Text.Encoding.UTF8.GetBytes(data);
    using var fs = new FileStream(path, FileMode.Create,
        FileAccess.Write, FileShare.None,
        bufferSize: 4096,
        useAsync: true);
    await fs.WriteAsync(bytes, 0, bytes.Length);
}

// BufferedStream: เพิ่ม buffer layer
using var baseStream = new FileStream(filePath, FileMode.Open);
using var buffered = new BufferedStream(baseStream, 65536); // 64KB buffer
// ดีกว่าเมื่อมีการอ่าน/เขียนขนาดเล็กหลายครั้ง
```

---

## 3. StreamReader และ StreamWriter

```csharp
// ===== StreamWriter =====
string textFile = "text_output.txt";

using (var writer = new StreamWriter(textFile, append: false,
    encoding: System.Text.Encoding.UTF8))
{
    writer.WriteLine("First line");
    writer.WriteLine("Second line");
    writer.Write("No newline at end");
    writer.Flush(); // บังคับ flush buffer
}

// Auto-flush
using (var writer = new StreamWriter(textFile) { AutoFlush = true })
{
    writer.WriteLine("Auto-flushed line");
}

// ===== StreamReader =====
using (var reader = new StreamReader(textFile))
{
    // ReadLine: อ่านทีละบรรทัด
    string? line;
    while ((line = reader.ReadLine()) != null)
        Console.WriteLine(line);

    // หรือ ReadToEnd: อ่านทั้งหมด
    reader.BaseStream.Seek(0, SeekOrigin.Begin);
    string all = reader.ReadToEnd();
}

// ===== ตัวอย่าง: Log File Writer =====
public class LogWriter : IDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed = false;

    public LogWriter(string path)
    {
        _writer = new StreamWriter(path, append: true, System.Text.Encoding.UTF8)
        {
            AutoFlush = true
        };
    }

    public void Log(string level, string message)
    {
        _writer.WriteLine($"[{DateTime.Now:yyyy-MM-dd HH:mm:ss.fff}] [{level,-5}] {message}");
    }

    public void Info(string message) => Log("INFO", message);
    public void Warn(string message) => Log("WARN", message);
    public void Error(string message) => Log("ERROR", message);

    public void Dispose()
    {
        if (!_disposed)
        {
            _writer.Dispose();
            _disposed = true;
        }
    }
}

// ใช้งาน
using var log = new LogWriter("app.log");
log.Info("Application started");
log.Warn("Low memory warning");
log.Error("Connection failed");
```

---

## 4. Directory Operations

```csharp
// ===== Directory Class =====
string dirPath = "TestDirectory";

// Create
Directory.CreateDirectory(dirPath); // สร้าง recursive ได้
Directory.CreateDirectory(@"a\b\c\d"); // สร้างทุก level

// Exists
bool dirExists = Directory.Exists(dirPath);

// GetFiles: รายชื่อไฟล์
string[] allFiles = Directory.GetFiles(dirPath);
string[] txtFiles = Directory.GetFiles(dirPath, "*.txt");
string[] txtRecursive = Directory.GetFiles(dirPath, "*.txt",
    SearchOption.AllDirectories);

// GetDirectories: รายชื่อ subdirectories
string[] subdirs = Directory.GetDirectories(dirPath);

// EnumerateFiles: Lazy (ดีกว่าสำหรับหลาย files)
foreach (string file in Directory.EnumerateFiles(dirPath, "*.cs",
    SearchOption.AllDirectories))
{
    Console.WriteLine(file);
}

// Move/Delete
Directory.Move("source", "destination");
Directory.Delete(dirPath, recursive: true); // ลบทั้ง directory

// GetCurrentDirectory, SetCurrentDirectory
string current = Directory.GetCurrentDirectory();
Console.WriteLine($"Current: {current}");

// Special Folders
string desktop = Environment.GetFolderPath(Environment.SpecialFolder.Desktop);
string documents = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
string appData = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
Console.WriteLine($"AppData: {appData}");
```

---

## 5. Path Class

```csharp
// Path: จัดการ file paths แบบ cross-platform
string fullPath = @"C:\Users\Alice\Documents\report.pdf";

// ดึงส่วนต่างๆ ของ path
Console.WriteLine(Path.GetDirectoryName(fullPath));  // C:\Users\Alice\Documents
Console.WriteLine(Path.GetFileName(fullPath));        // report.pdf
Console.WriteLine(Path.GetFileNameWithoutExtension(fullPath)); // report
Console.WriteLine(Path.GetExtension(fullPath));       // .pdf
Console.WriteLine(Path.GetPathRoot(fullPath));        // C:\

// รวม path (cross-platform)
string path1 = Path.Combine("Users", "Alice", "Documents");
string path2 = Path.Combine(@"C:\base", "subdir", "file.txt");
// ใช้ Path.Combine เสมอ แทน string concatenation

// Absolute vs Relative
bool isAbsolute = Path.IsPathRooted(fullPath); // true
bool isRelative = Path.IsPathRooted("relative/path"); // false

// GetFullPath: แปลง relative เป็น absolute
string relative = "data/file.txt";
string absolute = Path.GetFullPath(relative);
Console.WriteLine(absolute);

// Temporary paths
string tempFile = Path.GetTempFileName(); // สร้างไฟล์ temp จริง
string tempPath = Path.GetTempPath();      // ดึง temp directory

// Random filename
string randomName = Path.GetRandomFileName(); // ชื่อสุ่ม
Console.WriteLine(randomName); // e.g., "tmpHi3jk.tmp"

// Path separators (cross-platform)
Console.WriteLine(Path.DirectorySeparatorChar);    // \ on Windows, / on Linux
Console.WriteLine(Path.AltDirectorySeparatorChar); // / on both
Console.WriteLine(Path.PathSeparator);             // ; on Windows, : on Linux

// ChangeExtension
string newPath = Path.ChangeExtension(fullPath, ".txt");
// C:\Users\Alice\Documents\report.txt

// HasExtension
bool hasExt = Path.HasExtension("file.txt"); // true

// Invalid characters
char[] invalidChars = Path.GetInvalidFileNameChars();
char[] invalidPathChars = Path.GetInvalidPathChars();
```

---

## 6. FileInfo และ DirectoryInfo

```csharp
// FileInfo: OOP approach สำหรับ file operations
var fileInfo = new FileInfo("example.txt");

Console.WriteLine($"Name: {fileInfo.Name}");
Console.WriteLine($"FullName: {fileInfo.FullName}");
Console.WriteLine($"Extension: {fileInfo.Extension}");
Console.WriteLine($"Directory: {fileInfo.DirectoryName}");
Console.WriteLine($"Exists: {fileInfo.Exists}");
Console.WriteLine($"Length: {fileInfo.Length} bytes");
Console.WriteLine($"Created: {fileInfo.CreationTime}");
Console.WriteLine($"Modified: {fileInfo.LastWriteTime}");
Console.WriteLine($"IsReadOnly: {fileInfo.IsReadOnly}");

// Operations บน FileInfo
fileInfo.CopyTo("backup.txt", overwrite: true);
fileInfo.MoveTo("renamed.txt");
fileInfo.Delete();

// Open file จาก FileInfo
using (StreamWriter writer = fileInfo.CreateText())
    writer.WriteLine("Created via FileInfo");

using (StreamReader reader = fileInfo.OpenText())
    Console.WriteLine(reader.ReadToEnd());

// Refresh: อัพเดต cached info
fileInfo.Refresh();

// DirectoryInfo
var dirInfo = new DirectoryInfo("TestDir");
dirInfo.Create();

Console.WriteLine($"Dir Name: {dirInfo.Name}");
Console.WriteLine($"Parent: {dirInfo.Parent?.Name}");
Console.WriteLine($"Root: {dirInfo.Root}");

// รายชื่อไฟล์ใน directory
FileInfo[] files = dirInfo.GetFiles("*.txt");
DirectoryInfo[] subdirs = dirInfo.GetDirectories();

// EnumerateFiles (lazy)
foreach (FileInfo fi in dirInfo.EnumerateFiles("*.txt", SearchOption.AllDirectories))
{
    Console.WriteLine($"{fi.Name}: {fi.Length} bytes, {fi.LastWriteTime:d}");
}

// Sort files by size
var sortedBySize = dirInfo.GetFiles()
    .OrderByDescending(f => f.Length)
    .Take(10);

// รวม DirectoryInfo
DirectoryInfo subDir = dirInfo.CreateSubdirectory("SubFolder");
dirInfo.Delete(recursive: true);
```

---

## 7. JSON Serialization ด้วย System.Text.Json

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// ===== Models =====
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public string Email { get; set; } = "";

    [JsonIgnore]
    public string Password { get; set; } = ""; // ไม่ serialize

    [JsonPropertyName("created_at")]
    public DateTime CreatedAt { get; set; }

    [JsonConverter(typeof(JsonStringEnumConverter))]
    public UserRole Role { get; set; }
}

public enum UserRole { User, Admin, Moderator }

// ===== Serialization (Object -> JSON) =====
var person = new Person
{
    Id = 1,
    Name = "Alice",
    Age = 25,
    Email = "alice@example.com",
    Password = "secret123",
    CreatedAt = DateTime.Now,
    Role = UserRole.Admin
};

// Default
string json = JsonSerializer.Serialize(person);
Console.WriteLine(json);

// กับ options
var options = new JsonSerializerOptions
{
    WriteIndented = true,                // pretty print
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase, // camelCase
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
    Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping // Thai text
};

string prettyJson = JsonSerializer.Serialize(person, options);
Console.WriteLine(prettyJson);

// ===== Deserialization (JSON -> Object) =====
string jsonInput = """
{
    "id": 2,
    "name": "Bob Smith",
    "age": 30,
    "email": "bob@test.com",
    "created_at": "2024-01-15T10:30:00",
    "role": "User"
}
""";

// Deserialize
var deserializedPerson = JsonSerializer.Deserialize<Person>(jsonInput, options);
Console.WriteLine($"Name: {deserializedPerson?.Name}");

// Nullable (C# 8+)
Person? person2 = JsonSerializer.Deserialize<Person>(jsonInput, options);

// ===== Collections =====
var people = new List<Person> { person, deserializedPerson! };
string peopleJson = JsonSerializer.Serialize(people, options);

List<Person>? loaded = JsonSerializer.Deserialize<List<Person>>(peopleJson, options);

// ===== File I/O กับ JSON =====
// เขียน JSON file
async Task SaveToJsonAsync<T>(string path, T data)
{
    var opts = new JsonSerializerOptions { WriteIndented = true };
    await using var stream = File.Create(path);
    await JsonSerializer.SerializeAsync(stream, data, opts);
}

// อ่าน JSON file
async Task<T?> LoadFromJsonAsync<T>(string path)
{
    if (!File.Exists(path)) return default;
    await using var stream = File.OpenRead(path);
    return await JsonSerializer.DeserializeAsync<T>(stream);
}

// ===== JsonDocument สำหรับ dynamic JSON =====
string dynamicJson = """{"name": "Alice", "scores": [95, 87, 92]}""";

using JsonDocument doc = JsonDocument.Parse(dynamicJson);
JsonElement root = doc.RootElement;

string name = root.GetProperty("name").GetString()!;
var scores = root.GetProperty("scores").EnumerateArray()
    .Select(e => e.GetInt32())
    .ToList();

Console.WriteLine($"{name}: {string.Join(", ", scores)}");
```

---

## 8. CSV Processing

```csharp
// ===== Simple CSV Reader =====
public static class CsvHelper
{
    // อ่าน CSV ธรรมดา
    public static IEnumerable<string[]> ReadCsv(string path, bool skipHeader = true)
    {
        using var reader = new StreamReader(path, System.Text.Encoding.UTF8);

        if (skipHeader) reader.ReadLine(); // skip header

        string? line;
        while ((line = reader.ReadLine()) != null)
        {
            if (!string.IsNullOrWhiteSpace(line))
                yield return ParseCsvLine(line);
        }
    }

    // Parse บรรทัด CSV (รองรับ quoted fields)
    public static string[] ParseCsvLine(string line)
    {
        var fields = new List<string>();
        bool inQuotes = false;
        var current = new System.Text.StringBuilder();

        for (int i = 0; i < line.Length; i++)
        {
            char c = line[i];

            if (c == '"')
            {
                if (inQuotes && i + 1 < line.Length && line[i + 1] == '"')
                {
                    current.Append('"'); // escaped quote
                    i++;
                }
                else
                {
                    inQuotes = !inQuotes;
                }
            }
            else if (c == ',' && !inQuotes)
            {
                fields.Add(current.ToString().Trim());
                current.Clear();
            }
            else
            {
                current.Append(c);
            }
        }

        fields.Add(current.ToString().Trim());
        return fields.ToArray();
    }

    // เขียน CSV
    public static void WriteCsv<T>(string path, IEnumerable<T> data,
        Func<T, string[]> rowSelector, string[]? headers = null)
    {
        using var writer = new StreamWriter(path, false, System.Text.Encoding.UTF8);

        if (headers != null)
            writer.WriteLine(string.Join(",", headers.Select(h => EscapeField(h))));

        foreach (T item in data)
        {
            var fields = rowSelector(item);
            writer.WriteLine(string.Join(",", fields.Select(f => EscapeField(f))));
        }
    }

    private static string EscapeField(string field)
    {
        if (field.Contains(',') || field.Contains('"') || field.Contains('\n'))
            return $"\"{field.Replace("\"", "\"\"")}\"";
        return field;
    }
}

// ===== ตัวอย่างการใช้งาน CSV =====
public record Employee(
    int Id,
    string Name,
    string Department,
    decimal Salary,
    DateTime HireDate
);

// อ่าน CSV
public static List<Employee> ReadEmployees(string path)
{
    var employees = new List<Employee>();

    foreach (string[] fields in CsvHelper.ReadCsv(path, skipHeader: true))
    {
        if (fields.Length < 5) continue;

        try
        {
            employees.Add(new Employee(
                int.Parse(fields[0]),
                fields[1],
                fields[2],
                decimal.Parse(fields[3]),
                DateTime.Parse(fields[4])
            ));
        }
        catch (FormatException ex)
        {
            Console.Error.WriteLine($"Parse error: {ex.Message}");
        }
    }

    return employees;
}

// เขียน CSV
public static void WriteEmployees(string path, IEnumerable<Employee> employees)
{
    CsvHelper.WriteCsv(
        path,
        employees,
        e => new[]
        {
            e.Id.ToString(),
            e.Name,
            e.Department,
            e.Salary.ToString("F2"),
            e.HireDate.ToString("yyyy-MM-dd")
        },
        headers: new[] { "Id", "Name", "Department", "Salary", "HireDate" }
    );
}
```

---

## 9. โปรแกรมตัวอย่าง: Student Records in File

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace StudentRecords
{
    public class Student
    {
        public int Id { get; set; }
        public string Name { get; set; } = "";
        public string Email { get; set; } = "";
        public int Age { get; set; }
        public string Major { get; set; } = "";
        public List<Grade> Grades { get; set; } = new();
        public DateTime EnrollDate { get; set; }

        [JsonIgnore]
        public double GPA => Grades.Any()
            ? Grades.Average(g => g.Score)
            : 0.0;

        public override string ToString() =>
            $"[{Id}] {Name} ({Major}) - GPA: {GPA:F2}";
    }

    public record Grade(string Subject, double Score, string Semester);

    public class StudentRepository
    {
        private readonly string _dataPath;
        private readonly string _csvPath;
        private List<Student> _students = new();
        private int _nextId = 1;

        private static readonly JsonSerializerOptions JsonOptions = new()
        {
            WriteIndented = true,
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping
        };

        public StudentRepository(string dataPath = "students.json", string csvPath = "students.csv")
        {
            _dataPath = dataPath;
            _csvPath = csvPath;
        }

        // ===== CRUD Operations =====
        public Student Add(Student student)
        {
            student.Id = _nextId++;
            student.EnrollDate = DateTime.Now;
            _students.Add(student);
            return student;
        }

        public Student? GetById(int id) =>
            _students.FirstOrDefault(s => s.Id == id);

        public List<Student> GetAll() =>
            _students.OrderBy(s => s.Name).ToList();

        public bool Update(Student updated)
        {
            int idx = _students.FindIndex(s => s.Id == updated.Id);
            if (idx < 0) return false;
            _students[idx] = updated;
            return true;
        }

        public bool Delete(int id)
        {
            int removed = _students.RemoveAll(s => s.Id == id);
            return removed > 0;
        }

        public void AddGrade(int studentId, Grade grade)
        {
            var student = GetById(studentId)
                ?? throw new KeyNotFoundException($"Student {studentId} not found");
            student.Grades.Add(grade);
        }

        // ===== JSON Persistence =====
        public async Task SaveToJsonAsync()
        {
            try
            {
                // สร้าง directory ถ้ายังไม่มี
                var dir = Path.GetDirectoryName(_dataPath);
                if (!string.IsNullOrEmpty(dir))
                    Directory.CreateDirectory(dir);

                // Backup ก่อน save
                if (File.Exists(_dataPath))
                    File.Copy(_dataPath, _dataPath + ".bak", overwrite: true);

                await using var stream = File.Create(_dataPath);
                await JsonSerializer.SerializeAsync(stream, _students, JsonOptions);

                Console.WriteLine($"Saved {_students.Count} students to {_dataPath}");
            }
            catch (Exception ex)
            {
                Console.Error.WriteLine($"Save failed: {ex.Message}");
                throw;
            }
        }

        public async Task LoadFromJsonAsync()
        {
            if (!File.Exists(_dataPath))
            {
                Console.WriteLine("No existing data file found");
                return;
            }

            try
            {
                await using var stream = File.OpenRead(_dataPath);
                var loaded = await JsonSerializer.DeserializeAsync<List<Student>>(stream, JsonOptions);

                if (loaded != null)
                {
                    _students = loaded;
                    _nextId = _students.Any() ? _students.Max(s => s.Id) + 1 : 1;
                    Console.WriteLine($"Loaded {_students.Count} students");
                }
            }
            catch (JsonException ex)
            {
                Console.Error.WriteLine($"JSON parse error: {ex.Message}");
                // Try backup
                if (File.Exists(_dataPath + ".bak"))
                {
                    Console.WriteLine("Trying backup file...");
                    File.Copy(_dataPath + ".bak", _dataPath, overwrite: true);
                    await LoadFromJsonAsync();
                }
            }
        }

        // ===== CSV Export/Import =====
        public void ExportToCsv()
        {
            try
            {
                using var writer = new StreamWriter(_csvPath, false,
                    System.Text.Encoding.UTF8);

                // Header
                writer.WriteLine("Id,Name,Email,Age,Major,GPA,EnrollDate,GradeCount");

                foreach (var s in _students.OrderBy(s => s.Id))
                {
                    writer.WriteLine(
                        $"{s.Id}," +
                        $"\"{s.Name}\"," +
                        $"{s.Email}," +
                        $"{s.Age}," +
                        $"\"{s.Major}\"," +
                        $"{s.GPA:F2}," +
                        $"{s.EnrollDate:yyyy-MM-dd}," +
                        $"{s.Grades.Count}"
                    );
                }

                Console.WriteLine($"Exported to CSV: {_csvPath}");
            }
            catch (IOException ex)
            {
                Console.Error.WriteLine($"CSV export failed: {ex.Message}");
            }
        }

        public void ImportFromCsv(string importPath)
        {
            if (!File.Exists(importPath))
                throw new FileNotFoundException($"Import file not found: {importPath}");

            int imported = 0;
            int errors = 0;
            int line = 0;

            foreach (string[] fields in CsvHelper.ReadCsv(importPath, skipHeader: true))
            {
                line++;
                try
                {
                    if (fields.Length < 4) throw new FormatException("Too few fields");

                    var student = new Student
                    {
                        Name = fields[0],
                        Email = fields[1],
                        Age = int.Parse(fields[2]),
                        Major = fields[3],
                    };

                    Add(student);
                    imported++;
                }
                catch (Exception ex)
                {
                    Console.Error.WriteLine($"Import error line {line}: {ex.Message}");
                    errors++;
                }
            }

            Console.WriteLine($"Import complete: {imported} imported, {errors} errors");
        }

        // ===== Reports =====
        public void GenerateReport(string reportPath)
        {
            using var writer = new StreamWriter(reportPath, false, System.Text.Encoding.UTF8);

            writer.WriteLine("STUDENT RECORDS REPORT");
            writer.WriteLine($"Generated: {DateTime.Now:yyyy-MM-dd HH:mm:ss}");
            writer.WriteLine(new string('=', 60));

            writer.WriteLine($"\nTotal Students: {_students.Count}");

            if (_students.Any())
            {
                writer.WriteLine($"Average GPA: {_students.Average(s => s.GPA):F2}");
                writer.WriteLine($"Highest GPA: {_students.Max(s => s.GPA):F2}");
                writer.WriteLine($"Lowest GPA: {_students.Min(s => s.GPA):F2}");

                // By Major
                writer.WriteLine("\nBy Major:");
                foreach (var g in _students.GroupBy(s => s.Major).OrderBy(g => g.Key))
                {
                    writer.WriteLine($"  {g.Key}: {g.Count()} students, " +
                        $"Avg GPA: {g.Average(s => s.GPA):F2}");
                }

                // Top students
                writer.WriteLine("\nTop 5 Students:");
                foreach (var (i, s) in _students.OrderByDescending(s => s.GPA)
                    .Take(5).Select((s, i) => (i + 1, s)))
                {
                    writer.WriteLine($"  {i}. {s.Name} ({s.Major}) - GPA: {s.GPA:F2}");
                }

                // All students
                writer.WriteLine("\nAll Students:");
                foreach (var s in _students.OrderBy(s => s.Name))
                {
                    writer.WriteLine($"\n  {s.Name} (ID: {s.Id})");
                    writer.WriteLine($"    Major: {s.Major}, Age: {s.Age}");
                    writer.WriteLine($"    Email: {s.Email}");
                    writer.WriteLine($"    GPA: {s.GPA:F2}, Grades: {s.Grades.Count}");
                    if (s.Grades.Any())
                    {
                        foreach (var grade in s.Grades)
                            writer.WriteLine($"    - {grade.Subject}: {grade.Score:F1}");
                    }
                }
            }

            Console.WriteLine($"Report saved to: {reportPath}");
        }
    }

    class Program
    {
        static async Task Main()
        {
            Console.WriteLine("=== Student Records System ===\n");

            var repo = new StudentRepository("data/students.json", "data/students.csv");

            // Create data directory
            Directory.CreateDirectory("data");

            // เพิ่มนักเรียน
            var alice = repo.Add(new Student
            {
                Name = "Alice Johnson",
                Email = "alice@university.edu",
                Age = 20,
                Major = "Computer Science"
            });

            var bob = repo.Add(new Student
            {
                Name = "Bob Smith",
                Email = "bob@university.edu",
                Age = 22,
                Major = "Mathematics"
            });

            var charlie = repo.Add(new Student
            {
                Name = "Charlie Brown",
                Email = "charlie@university.edu",
                Age = 21,
                Major = "Computer Science"
            });

            // เพิ่มเกรด
            repo.AddGrade(alice.Id, new Grade("Programming", 92.0, "2024/1"));
            repo.AddGrade(alice.Id, new Grade("Database", 88.5, "2024/1"));
            repo.AddGrade(alice.Id, new Grade("Math", 95.0, "2024/1"));

            repo.AddGrade(bob.Id, new Grade("Calculus", 98.0, "2024/1"));
            repo.AddGrade(bob.Id, new Grade("Statistics", 91.0, "2024/1"));

            repo.AddGrade(charlie.Id, new Grade("Programming", 78.0, "2024/1"));
            repo.AddGrade(charlie.Id, new Grade("Algorithms", 82.0, "2024/1"));

            // แสดงรายชื่อ
            Console.WriteLine("All Students:");
            foreach (var s in repo.GetAll())
                Console.WriteLine($"  {s}");

            // Save JSON
            await repo.SaveToJsonAsync();

            // Export CSV
            repo.ExportToCsv();

            // Generate Report
            repo.GenerateReport("data/report.txt");

            // Test Load
            var repo2 = new StudentRepository("data/students.json");
            await repo2.LoadFromJsonAsync();

            Console.WriteLine($"\nLoaded from file: {repo2.GetAll().Count} students");

            // แสดง report
            Console.WriteLine("\n--- Report Preview ---");
            Console.WriteLine(File.ReadAllText("data/report.txt"));

            // Cleanup
            Directory.Delete("data", recursive: true);
            Console.WriteLine("\nCleanup complete");
        }
    }
}
```

---

## Exercises

### Exercise 1: Config File Manager
สร้าง ConfigManager ที่:
- อ่าน/เขียน key=value config file
- Support comments (#)
- Cache ค่า config ใน Dictionary
- Watch file changes ด้วย FileSystemWatcher

### Exercise 2: Log Analyzer
สร้าง log analyzer ที่:
- อ่านไฟล์ log ขนาดใหญ่ (lazy loading)
- Filter ตาม level (ERROR, WARN, INFO)
- สรุปจำนวน errors ต่อวัน
- Export summary เป็น CSV

### Exercise 3: Backup System
สร้าง backup system ที่:
- Copy files ระหว่าง directories
- Skip files ที่ไม่เปลี่ยนแปลง (compare LastWriteTime)
- สร้าง backup manifest (JSON)
- Progress reporting

---

## สรุป

- ✅ `File.ReadAllText/WriteAllText` สะดวกสำหรับไฟล์ขนาดเล็ก
- ✅ `File.ReadLines` lazy loading เหมาะสำหรับไฟล์ใหญ่
- ✅ `StreamReader/StreamWriter` ยืดหยุ่นกว่า, รองรับ encoding
- ✅ `FileStream` low-level, ควบคุม mode/access/share ได้
- ✅ ใช้ `using` หรือ `await using` เสมอ เพื่อ dispose resource
- ✅ `Path.Combine` แทน string concatenation - cross-platform
- ✅ `FileInfo/DirectoryInfo` เหมาะสำหรับ OOP approach
- ✅ `JsonSerializer.Serialize/Deserialize` ใช้ System.Text.Json
- ✅ `JsonSerializerOptions` ปรับแต่ง naming policy, formatting
- ✅ `StreamAsync` methods สำหรับ async file operations
- ✅ CSV parsing ต้องรองรับ quoted fields

## Part ถัดไป
**Part 028** จะพูดถึง Delegates และ Events: การประกาศ delegate, Multicast delegates, Func/Action/Predicate, Event pattern และ EventHandler

---
*Part 027/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
