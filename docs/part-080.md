# Part 80: SOLID Principles

## เนื้อหาใน Part นี้
- Single Responsibility Principle (SRP)
- Open/Closed Principle (OCP)
- Liskov Substitution Principle (LSP)
- Interface Segregation Principle (ISP)
- Dependency Inversion Principle (DIP)
- ตัวอย่างที่ผิดและถูก
- โปรแกรมตัวอย่าง: Refactoring violations

---

## SOLID คืออะไร?

**SOLID** เป็นชื่อย่อของหลักการออกแบบซอฟต์แวร์ 5 ข้อที่เสนอโดย Robert C. Martin (Uncle Bob) ช่วยให้โค้ด:
- อ่านเข้าใจง่าย
- บำรุงรักษาได้ง่าย
- ทดสอบได้ง่าย
- ขยายได้โดยไม่แตกของเดิม

```
S - Single Responsibility Principle
O - Open/Closed Principle
L - Liskov Substitution Principle
I - Interface Segregation Principle
D - Dependency Inversion Principle
```

---

## 1. Single Responsibility Principle (SRP)

> "Class ควรมีเหตุผลในการเปลี่ยนแปลงเพียงอย่างเดียว"

Class ควรมีหน้าที่เดียว เมื่อต้องการเปลี่ยนหน้าที่ใดๆ ควรแก้ไข class ที่รับผิดชอบนั้นเท่านั้น

### ตัวอย่างที่ผิด (Violation)

```csharp
// ❌ ผิด: UserManager รับผิดชอบหลายอย่างเกินไป
public class UserManager
{
    private readonly SqlConnection _connection;
    private readonly SmtpClient _smtpClient;
    
    // หน้าที่ 1: จัดการ user (ถูก)
    public User GetUser(int id)
    {
        // Database query - ปนกันกับ business logic
        using var cmd = new SqlCommand($"SELECT * FROM Users WHERE Id = {id}", _connection);
        using var reader = cmd.ExecuteReader();
        return new User { Id = reader.GetInt32(0), Name = reader.GetString(1) };
    }
    
    // หน้าที่ 2: ส่ง email (ผิด! ควรอยู่ใน EmailService)
    public void SendWelcomeEmail(User user)
    {
        var mail = new MailMessage("noreply@shop.com", user.Email);
        mail.Subject = "ยินดีต้อนรับ";
        mail.Body = $"สวัสดีคุณ {user.Name}";
        _smtpClient.Send(mail);
    }
    
    // หน้าที่ 3: Validate (ผิด! ควรอยู่ใน Validator)
    public bool ValidateUser(User user)
    {
        if (string.IsNullOrEmpty(user.Name)) return false;
        if (string.IsNullOrEmpty(user.Email)) return false;
        if (!user.Email.Contains('@')) return false;
        return true;
    }
    
    // หน้าที่ 4: Log (ผิด! ควรอยู่ใน Logger)
    public void LogUserAction(User user, string action)
    {
        File.AppendAllText("user.log",
            $"{DateTime.Now}: User {user.Id} - {action}\n");
    }
    
    // หน้าที่ 5: สร้าง report (ผิด! ควรอยู่ใน ReportService)
    public string GenerateUserReport(List<User> users)
    {
        var sb = new StringBuilder();
        sb.AppendLine("User Report");
        foreach (var u in users)
            sb.AppendLine($"{u.Id}: {u.Name} ({u.Email})");
        return sb.ToString();
    }
}
```

### ตัวอย่างที่ถูก (Fixed)

```csharp
// ✅ ถูก: แยกความรับผิดชอบออกจากกัน

// หน้าที่เดียว: จัดการ User data
public class UserRepository
{
    private readonly AppDbContext _context;
    
    public UserRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<User?> GetByIdAsync(int id)
        => await _context.Users.FindAsync(id);
    
    public async Task<User?> GetByEmailAsync(string email)
        => await _context.Users.FirstOrDefaultAsync(u => u.Email == email);
    
    public async Task AddAsync(User user)
    {
        _context.Users.Add(user);
        await _context.SaveChangesAsync();
    }
}

// หน้าที่เดียว: Validate User
public class UserValidator
{
    public (bool IsValid, List<string> Errors) Validate(CreateUserRequest request)
    {
        var errors = new List<string>();
        
        if (string.IsNullOrWhiteSpace(request.Name))
            errors.Add("ชื่อต้องไม่ว่าง");
        else if (request.Name.Length < 2 || request.Name.Length > 100)
            errors.Add("ชื่อต้องมีความยาว 2-100 ตัวอักษร");
        
        if (string.IsNullOrWhiteSpace(request.Email))
            errors.Add("Email ต้องไม่ว่าง");
        else if (!IsValidEmail(request.Email))
            errors.Add("รูปแบบ Email ไม่ถูกต้อง");
        
        if (string.IsNullOrWhiteSpace(request.Password))
            errors.Add("รหัสผ่านต้องไม่ว่าง");
        else if (request.Password.Length < 8)
            errors.Add("รหัสผ่านต้องมีความยาวอย่างน้อย 8 ตัวอักษร");
        
        return (!errors.Any(), errors);
    }
    
    private static bool IsValidEmail(string email)
    {
        try
        {
            var addr = new System.Net.Mail.MailAddress(email);
            return addr.Address == email;
        }
        catch { return false; }
    }
}

// หน้าที่เดียว: ส่ง Email
public class EmailService
{
    private readonly IEmailProvider _provider;
    
    public EmailService(IEmailProvider provider)
    {
        _provider = provider;
    }
    
    public async Task SendWelcomeEmailAsync(User user)
        => await _provider.SendAsync(
            user.Email,
            "ยินดีต้อนรับ",
            $"สวัสดีคุณ {user.Name}, ขอบคุณที่สมัครสมาชิก!");
    
    public async Task SendPasswordResetEmailAsync(User user, string token)
        => await _provider.SendAsync(
            user.Email,
            "รีเซ็ตรหัสผ่าน",
            $"คลิกที่นี่เพื่อรีเซ็ตรหัสผ่าน: https://app.com/reset?token={token}");
}

// หน้าที่เดียว: สร้าง Report
public class UserReportService
{
    private readonly UserRepository _repository;
    
    public UserReportService(UserRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<string> GenerateCsvReportAsync()
    {
        var users = await _repository.GetAllAsync();
        var sb = new StringBuilder("Id,Name,Email,RegisteredAt\n");
        foreach (var u in users)
            sb.AppendLine($"{u.Id},{u.Name},{u.Email},{u.RegisteredAt:yyyy-MM-dd}");
        return sb.ToString();
    }
}

// UserService - orchestrate ทุก service
public class UserService
{
    private readonly UserRepository _repository;
    private readonly UserValidator _validator;
    private readonly EmailService _emailService;
    
    public UserService(
        UserRepository repository,
        UserValidator validator,
        EmailService emailService)
    {
        _repository = repository;
        _validator = validator;
        _emailService = emailService;
    }
    
    public async Task<RegisterResult> RegisterAsync(CreateUserRequest request)
    {
        var (isValid, errors) = _validator.Validate(request);
        if (!isValid) return new RegisterResult(false, errors);
        
        var user = User.Create(request.Name, request.Email, request.Password);
        await _repository.AddAsync(user);
        await _emailService.SendWelcomeEmailAsync(user);
        
        return new RegisterResult(true, Array.Empty<string>(), user.Id);
    }
}
```

---

## 2. Open/Closed Principle (OCP)

> "Software entities ควรเปิดสำหรับการขยาย แต่ปิดสำหรับการแก้ไข"

เมื่อต้องการเพิ่ม feature ใหม่ ควรเพิ่ม code ใหม่ ไม่ใช่แก้ code เดิม

### ตัวอย่างที่ผิด

```csharp
// ❌ ผิด: เพิ่ม payment method ต้องแก้ method นี้ทุกครั้ง
public class PaymentProcessor
{
    public decimal ProcessPayment(decimal amount, string paymentType)
    {
        if (paymentType == "CreditCard")
        {
            // ค่าธรรมเนียม 1.5%
            return amount * 1.015m;
        }
        else if (paymentType == "PayPal")
        {
            // ค่าธรรมเนียม 2%
            return amount * 1.02m;
        }
        else if (paymentType == "Bitcoin")
        {
            // ค่าธรรมเนียม 0.5%
            return amount * 1.005m;
        }
        // ถ้าเพิ่ม payment method ใหม่ ต้องแก้ method นี้!
        throw new NotSupportedException($"ไม่รองรับ: {paymentType}");
    }
}
```

### ตัวอย่างที่ถูก

```csharp
// ✅ ถูก: เพิ่ม payment method ใหม่โดยสร้าง class ใหม่

// Interface ที่ "ปิด" สำหรับการแก้ไข
public interface IPaymentMethod
{
    string Name { get; }
    decimal CalculateFee(decimal amount);
    Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details);
}

// Concrete implementations - เปิดสำหรับการขยาย
public class CreditCardPayment : IPaymentMethod
{
    public string Name => "Credit Card";
    public decimal CalculateFee(decimal amount) => amount * 0.015m;
    
    public async Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details)
    {
        Console.WriteLine($"[Credit Card] ชำระ {amount + CalculateFee(amount):N2} บาท");
        return new PaymentResult(true, $"CC-{Guid.NewGuid():N}"[..12]);
    }
}

public class PayPalPayment : IPaymentMethod
{
    public string Name => "PayPal";
    public decimal CalculateFee(decimal amount) => amount * 0.02m;
    
    public async Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details)
    {
        Console.WriteLine($"[PayPal] ชำระ {amount + CalculateFee(amount):N2} บาท");
        return new PaymentResult(true, $"PP-{Guid.NewGuid():N}"[..12]);
    }
}

// เพิ่ม payment method ใหม่ โดยไม่แก้ code เดิม!
public class PromptPayPayment : IPaymentMethod
{
    public string Name => "PromptPay";
    public decimal CalculateFee(decimal amount) => 0; // ไม่มีค่าธรรมเนียม
    
    public async Task<PaymentResult> ProcessAsync(decimal amount, PaymentDetails details)
    {
        Console.WriteLine($"[PromptPay] ชำระ {amount:N2} บาท ผ่าน QR Code");
        return new PaymentResult(true, $"QR-{Guid.NewGuid():N}"[..12]);
    }
}

// Processor ที่ไม่ต้องแก้เมื่อเพิ่ม payment method
public class PaymentProcessor
{
    private readonly Dictionary<string, IPaymentMethod> _methods = new();
    
    public void Register(IPaymentMethod method)
        => _methods[method.Name.ToLower()] = method;
    
    public async Task<PaymentResult> ProcessAsync(
        string methodName, decimal amount, PaymentDetails details)
    {
        if (!_methods.TryGetValue(methodName.ToLower(), out var method))
            throw new NotSupportedException($"ไม่รองรับ: {methodName}");
        
        var fee = method.CalculateFee(amount);
        Console.WriteLine($"ค่าธรรมเนียม: {fee:N2} บาท");
        
        return await method.ProcessAsync(amount, details);
    }
}
```

---

## 3. Liskov Substitution Principle (LSP)

> "Object ของ subclass ควรสามารถแทนที่ object ของ parent class ได้โดยไม่ทำให้ program ผิดพลาด"

### ตัวอย่างที่ผิด

```csharp
// ❌ ผิด: Classic "Square/Rectangle" problem
public class Rectangle
{
    public virtual double Width { get; set; }
    public virtual double Height { get; set; }
    
    public double Area() => Width * Height;
}

public class Square : Rectangle
{
    // ❌ Square แก้ Width ต้องแก้ Height ด้วย ทำให้ LSP ถูกละเมิด
    public override double Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; }
    }
    
    public override double Height
    {
        get => base.Height;
        set { base.Width = value; base.Height = value; }
    }
}

// Code นี้ใช้ Rectangle แต่พอส่ง Square เข้ามา พฤติกรรมผิดคาด!
void ResizeToFitScreen(Rectangle rect, double maxWidth, double maxHeight)
{
    rect.Width = Math.Min(rect.Width, maxWidth);
    rect.Height = Math.Min(rect.Height, maxHeight);  // ถ้าเป็น Square Height จะถูก override Width อีกครั้ง!
    Console.WriteLine($"Area: {rect.Area()}");
}
```

### ตัวอย่างที่ถูก

```csharp
// ✅ ถูก: ใช้ abstraction แทน inheritance
public abstract class Shape
{
    public abstract double Area();
    public abstract double Perimeter();
    public abstract string Name { get; }
}

public class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }
    
    public override string Name => "Rectangle";
    
    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }
    
    public override double Area() => Width * Height;
    public override double Perimeter() => 2 * (Width + Height);
}

public class Square : Shape
{
    public double Side { get; }
    
    public override string Name => "Square";
    
    public Square(double side)
    {
        Side = side;
    }
    
    public override double Area() => Side * Side;
    public override double Perimeter() => 4 * Side;
}

public class Circle : Shape
{
    public double Radius { get; }
    
    public override string Name => "Circle";
    
    public Circle(double radius)
    {
        Radius = radius;
    }
    
    public override double Area() => Math.PI * Radius * Radius;
    public override double Perimeter() => 2 * Math.PI * Radius;
}

// ✅ ทุก Shape สามารถแทนกันได้
void PrintShapeInfo(Shape shape)
{
    Console.WriteLine($"{shape.Name}: Area={shape.Area():N2}, Perimeter={shape.Perimeter():N2}");
}

// LSP ในบริบท Repository
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
}

public interface IWriteRepository<T> : IReadRepository<T>
{
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

// ReadOnlyRepository ไม่ throw NotImplementedException
public class ReadOnlyProductRepository : IReadRepository<Product>
{
    private readonly AppDbContext _context;
    
    public ReadOnlyProductRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<Product?> GetByIdAsync(Guid id)
        => await _context.Products.AsNoTracking().FirstOrDefaultAsync(p => p.Id == id);
    
    public async Task<IEnumerable<Product>> GetAllAsync()
        => await _context.Products.AsNoTracking().ToListAsync();
}
```

---

## 4. Interface Segregation Principle (ISP)

> "Client ไม่ควรถูกบังคับให้ depend on interface ที่ไม่ได้ใช้"

แยก interface ใหญ่ๆ ออกเป็น interface เล็กๆ ที่เฉพาะเจาะจง

### ตัวอย่างที่ผิด

```csharp
// ❌ ผิด: Interface ใหญ่เกินไป บาง implement ต้องใช้ NotImplementedException
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
    void GetPaid();
    void TakeVacation();
    void AttendMeeting();
    void WriteReport();
    void ManageTeam();    // ไม่ใช่ทุก worker ที่ manage team
    void HireEmployee();  // ไม่ใช่ทุก worker ที่ hire
}

public class Engineer : IWorker
{
    public void Work() => Console.WriteLine("เขียนโค้ด");
    public void Eat() => Console.WriteLine("กินข้าว");
    public void Sleep() => Console.WriteLine("นอนหลับ");
    public void GetPaid() => Console.WriteLine("รับเงินเดือน");
    public void TakeVacation() => Console.WriteLine("ลาพักร้อน");
    public void AttendMeeting() => Console.WriteLine("ประชุม");
    public void WriteReport() => Console.WriteLine("เขียน report");
    
    // ❌ Engineer ไม่ได้ manage team แต่ต้อง implement!
    public void ManageTeam() => throw new NotImplementedException();
    public void HireEmployee() => throw new NotImplementedException();
}
```

### ตัวอย่างที่ถูก

```csharp
// ✅ ถูก: แยก interfaces ออกตาม responsibility

public interface IWorkable
{
    void Work();
}

public interface IEatable
{
    void Eat();
}

public interface ISleepable
{
    void Sleep();
}

public interface IPayable
{
    void GetPaid();
    decimal GetSalary();
}

public interface IVacationable
{
    void TakeVacation(int days);
    int GetRemainingVacationDays();
}

public interface IReportWriter
{
    void WriteReport(string content);
    List<string> GetReports();
}

public interface ITeamManager
{
    void ManageTeam();
    void AddTeamMember(Employee employee);
    List<Employee> GetTeamMembers();
}

public interface IHirer
{
    void HireEmployee(JobApplication application);
    List<JobApplication> GetApplications();
}

// Engineer implement เฉพาะที่ใช้จริง
public class Engineer : IWorkable, IEatable, ISleepable, IPayable, IVacationable, IReportWriter
{
    public void Work() => Console.WriteLine("เขียนโค้ด");
    public void Eat() => Console.WriteLine("กินข้าว");
    public void Sleep() => Console.WriteLine("นอนหลับ");
    public void GetPaid() => Console.WriteLine("รับเงินเดือน");
    public decimal GetSalary() => 80000m;
    public void TakeVacation(int days) => Console.WriteLine($"ลาพักร้อน {days} วัน");
    public int GetRemainingVacationDays() => 10;
    public void WriteReport(string content) => Console.WriteLine($"เขียน report: {content}");
    public List<string> GetReports() => new List<string>();
}

// Manager implement ITeamManager และ IHirer เพิ่ม
public class Manager : Engineer, ITeamManager, IHirer
{
    private readonly List<Employee> _team = new();
    private readonly List<JobApplication> _applications = new();
    
    public void ManageTeam() => Console.WriteLine("จัดการทีม");
    public void AddTeamMember(Employee emp) => _team.Add(emp);
    public List<Employee> GetTeamMembers() => _team;
    public void HireEmployee(JobApplication app) => _applications.Add(app);
    public List<JobApplication> GetApplications() => _applications;
}
```

---

## 5. Dependency Inversion Principle (DIP)

> "High-level modules ไม่ควร depend on Low-level modules ทั้งคู่ควร depend on Abstractions"
> "Abstractions ไม่ควร depend on Details. Details ควร depend on Abstractions"

### ตัวอย่างที่ผิด

```csharp
// ❌ ผิด: High-level depend on Low-level directly
public class OrderService  // High-level
{
    // ❌ Depend on concrete classes โดยตรง
    private readonly MySqlOrderRepository _repository;  // Low-level
    private readonly SmtpEmailService _emailService;    // Low-level
    private readonly ConsoleLogger _logger;              // Low-level
    
    public OrderService()
    {
        // ❌ สร้าง instances เองใน constructor
        _repository = new MySqlOrderRepository("Server=...;");
        _emailService = new SmtpEmailService("smtp.gmail.com", 587);
        _logger = new ConsoleLogger();
    }
    
    public void PlaceOrder(Order order)
    {
        _logger.Log($"Placing order for {order.CustomerId}");
        _repository.Save(order);
        _emailService.Send(order.CustomerEmail, "Order confirmed");
    }
}
```

### ตัวอย่างที่ถูก

```csharp
// ✅ ถูก: Depend on Abstractions

// Abstractions (interfaces)
public interface IOrderRepository
{
    Task SaveAsync(Order order, CancellationToken ct = default);
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
}

public interface IEmailNotifier
{
    Task NotifyAsync(string recipient, string subject, string body, CancellationToken ct = default);
}

public interface ILogger
{
    void Log(string message, LogLevel level = LogLevel.Information);
    void LogError(string message, Exception? exception = null);
}

// High-level: depend on abstractions
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailNotifier _notifier;
    private readonly ILogger _logger;
    
    // ✅ Constructor injection
    public OrderService(
        IOrderRepository repository,
        IEmailNotifier notifier,
        ILogger logger)
    {
        _repository = repository;
        _notifier = notifier;
        _logger = logger;
    }
    
    public async Task<PlaceOrderResult> PlaceOrderAsync(
        CreateOrderRequest request, CancellationToken ct = default)
    {
        _logger.Log($"กำลังสร้างคำสั่งซื้อสำหรับ {request.CustomerId}");
        
        var order = Order.Create(request.CustomerId, request.Items);
        await _repository.SaveAsync(order, ct);
        
        await _notifier.NotifyAsync(
            request.CustomerEmail,
            "ยืนยันการสั่งซื้อ",
            $"คำสั่งซื้อ #{order.Id} ได้รับการยืนยัน",
            ct);
        
        _logger.Log($"สร้างคำสั่งซื้อสำเร็จ: {order.Id}");
        return new PlaceOrderResult(order.Id, order.TotalAmount);
    }
}

// Low-level: Details depend on Abstractions
public class SqlServerOrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    
    public SqlServerOrderRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task SaveAsync(Order order, CancellationToken ct)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync(ct);
    }
    
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _context.Orders.FindAsync(new object[] { id }, ct);
}

public class SmtpEmailNotifier : IEmailNotifier
{
    private readonly SmtpSettings _settings;
    
    public SmtpEmailNotifier(SmtpSettings settings)
    {
        _settings = settings;
    }
    
    public async Task NotifyAsync(
        string recipient, string subject, string body, CancellationToken ct)
    {
        Console.WriteLine($"[SMTP] ส่ง email ถึง {recipient}: {subject}");
        await Task.Delay(50, ct);
    }
}

// ง่ายต่อการ swap implementation
public class InMemoryOrderRepository : IOrderRepository
{
    private readonly Dictionary<Guid, Order> _orders = new();
    
    public Task SaveAsync(Order order, CancellationToken ct)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }
    
    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => Task.FromResult(_orders.GetValueOrDefault(id));
}
```

---

## โปรแกรมตัวอย่าง: Refactoring SOLID Violations

```csharp
// ===== ก่อน Refactor: โค้ดที่ละเมิด SOLID =====
/*
public class ReportGenerator
{
    // ❌ SRP: ทำหลายอย่าง
    // ❌ OCP: เพิ่ม format ต้องแก้ if-else
    // ❌ DIP: depend on concrete classes
    
    public string GenerateReport(List<SalesData> data, string format)
    {
        // Fetch data (ควรอยู่ใน repository)
        var conn = new SqlConnection("Server=...");
        var cmd = new SqlCommand("SELECT * FROM Sales", conn);
        
        // Format (ควร open for extension)
        if (format == "CSV")
        {
            var sb = new StringBuilder("Date,Product,Amount\n");
            foreach (var d in data)
                sb.AppendLine($"{d.Date},{d.Product},{d.Amount}");
            return sb.ToString();
        }
        else if (format == "HTML")
        {
            // HTML format
        }
        // เพิ่ม JSON, PDF ต้องแก้ method นี้!
        
        // Send email (ควรอยู่ใน EmailService)
        var smtp = new SmtpClient("smtp.server.com");
        smtp.Send(new MailMessage("from@shop.com", "manager@shop.com", 
                                   "Report", "See attached"));
        
        throw new NotSupportedException();
    }
}
*/

// ===== หลัง Refactor: โค้ดที่ถูก SOLID =====

// Domain Model
public record SalesData(DateTime Date, string Product, decimal Amount, string Category);
public record SalesReport(string Title, DateTime GeneratedAt, List<SalesData> Data);

// SRP + DIP: Interface สำหรับ data access
public interface ISalesRepository
{
    Task<List<SalesData>> GetSalesAsync(DateTime from, DateTime to, CancellationToken ct = default);
    Task<decimal> GetTotalSalesAsync(DateTime from, DateTime to, CancellationToken ct = default);
}

// OCP + SRP: Format interface
public interface IReportFormatter
{
    string FormatName { get; }
    string Format(SalesReport report);
}

// DIP + SRP: Email interface
public interface IReportEmailer
{
    Task SendReportAsync(string recipient, SalesReport report, string formattedContent, CancellationToken ct = default);
}

// Concrete Implementations (Low-level - depend on nothing)
public class InMemorySalesRepository : ISalesRepository
{
    private readonly List<SalesData> _data;
    
    public InMemorySalesRepository(List<SalesData>? seedData = null)
    {
        _data = seedData ?? GenerateSampleData();
    }
    
    public Task<List<SalesData>> GetSalesAsync(DateTime from, DateTime to, CancellationToken ct)
    {
        var result = _data
            .Where(d => d.Date >= from && d.Date <= to)
            .ToList();
        return Task.FromResult(result);
    }
    
    public Task<decimal> GetTotalSalesAsync(DateTime from, DateTime to, CancellationToken ct)
    {
        var total = _data
            .Where(d => d.Date >= from && d.Date <= to)
            .Sum(d => d.Amount);
        return Task.FromResult(total);
    }
    
    private static List<SalesData> GenerateSampleData()
    {
        var random = new Random(42);
        var products = new[] { "iPhone 15", "MacBook Pro", "AirPods Pro", "iPad" };
        var categories = new[] { "Phone", "Laptop", "Audio", "Tablet" };
        
        return Enumerable.Range(1, 30)
            .Select(i => new SalesData(
                DateTime.Today.AddDays(-i),
                products[random.Next(products.Length)],
                random.Next(1000, 50000),
                categories[random.Next(categories.Length)]))
            .ToList();
    }
}

// OCP: เพิ่ม formatter ใหม่โดยไม่แก้ code เดิม
public class CsvReportFormatter : IReportFormatter
{
    public string FormatName => "CSV";
    
    public string Format(SalesReport report)
    {
        var sb = new StringBuilder();
        sb.AppendLine($"# {report.Title}");
        sb.AppendLine($"# Generated: {report.GeneratedAt:yyyy-MM-dd HH:mm:ss}");
        sb.AppendLine("Date,Product,Category,Amount");
        
        foreach (var d in report.Data)
            sb.AppendLine($"{d.Date:yyyy-MM-dd},{EscapeCsv(d.Product)},{d.Category},{d.Amount:N2}");
        
        sb.AppendLine($",,Total,{report.Data.Sum(d => d.Amount):N2}");
        return sb.ToString();
    }
    
    private static string EscapeCsv(string value) 
        => value.Contains(',') ? $"\"{value}\"" : value;
}

public class HtmlReportFormatter : IReportFormatter
{
    public string FormatName => "HTML";
    
    public string Format(SalesReport report)
    {
        var total = report.Data.Sum(d => d.Amount);
        var sb = new StringBuilder();
        
        sb.AppendLine($"<!DOCTYPE html><html><head><title>{report.Title}</title>");
        sb.AppendLine("<style>table{{border-collapse:collapse;width:100%}} " +
                      "th,td{{border:1px solid #ddd;padding:8px;}} " +
                      "th{{background-color:#4CAF50;color:white;}}</style></head><body>");
        sb.AppendLine($"<h1>{report.Title}</h1>");
        sb.AppendLine($"<p>Generated: {report.GeneratedAt:yyyy-MM-dd HH:mm:ss}</p>");
        sb.AppendLine("<table><tr><th>วันที่</th><th>สินค้า</th><th>หมวดหมู่</th><th>ยอด</th></tr>");
        
        foreach (var d in report.Data)
            sb.AppendLine($"<tr><td>{d.Date:yyyy-MM-dd}</td><td>{d.Product}</td>" +
                         $"<td>{d.Category}</td><td>{d.Amount:N2}</td></tr>");
        
        sb.AppendLine($"<tr><td colspan='3'><b>Total</b></td><td><b>{total:N2}</b></td></tr>");
        sb.AppendLine("</table></body></html>");
        
        return sb.ToString();
    }
}

// เพิ่ม formatter ใหม่: JSON - ไม่ต้องแก้ code เดิม!
public class JsonReportFormatter : IReportFormatter
{
    public string FormatName => "JSON";
    
    public string Format(SalesReport report)
    {
        var obj = new
        {
            title = report.Title,
            generatedAt = report.GeneratedAt,
            totalAmount = report.Data.Sum(d => d.Amount),
            data = report.Data.Select(d => new
            {
                date = d.Date.ToString("yyyy-MM-dd"),
                product = d.Product,
                category = d.Category,
                amount = d.Amount
            })
        };
        
        return JsonSerializer.Serialize(obj, new JsonSerializerOptions { WriteIndented = true });
    }
}

// SRP + DIP: High-level service
public class SalesReportService
{
    private readonly ISalesRepository _repository;
    private readonly Dictionary<string, IReportFormatter> _formatters;
    private readonly IReportEmailer _emailer;
    
    public SalesReportService(
        ISalesRepository repository,
        IEnumerable<IReportFormatter> formatters,
        IReportEmailer emailer)
    {
        _repository = repository;
        _emailer = emailer;
        _formatters = formatters.ToDictionary(f => f.FormatName.ToUpper());
    }
    
    public async Task<string> GenerateAsync(
        DateTime from, DateTime to, string format, CancellationToken ct = default)
    {
        format = format.ToUpper();
        if (!_formatters.ContainsKey(format))
            throw new ArgumentException($"ไม่รองรับ format: {format}. รองรับ: {string.Join(", ", _formatters.Keys)}");
        
        var data = await _repository.GetSalesAsync(from, to, ct);
        var report = new SalesReport($"Sales Report {from:MM/dd} - {to:MM/dd/yyyy}", DateTime.UtcNow, data);
        
        return _formatters[format].Format(report);
    }
    
    public async Task GenerateAndEmailAsync(
        DateTime from, DateTime to, string format, 
        string recipient, CancellationToken ct = default)
    {
        var data = await _repository.GetSalesAsync(from, to, ct);
        var report = new SalesReport($"Sales Report", DateTime.UtcNow, data);
        
        format = format.ToUpper();
        if (!_formatters.TryGetValue(format, out var formatter))
            throw new ArgumentException($"ไม่รองรับ format: {format}");
        
        var content = formatter.Format(report);
        await _emailer.SendReportAsync(recipient, report, content, ct);
    }
}

// Simple emailer
public class ConsoleReportEmailer : IReportEmailer
{
    public Task SendReportAsync(
        string recipient, SalesReport report, string content, CancellationToken ct)
    {
        Console.WriteLine($"\n[Email] ส่ง report ไปยัง {recipient}");
        Console.WriteLine($"  Subject: {report.Title}");
        Console.WriteLine($"  Content preview: {content[..Math.Min(100, content.Length)]}...");
        return Task.CompletedTask;
    }
}

// ===== DI Container Setup =====
var services = new ServiceCollection();

// Register dependencies (DIP in action)
services.AddScoped<ISalesRepository, InMemorySalesRepository>();
services.AddScoped<IReportEmailer, ConsoleReportEmailer>();

// OCP: ลงทะเบียน formatters ทั้งหมด
services.AddScoped<IReportFormatter, CsvReportFormatter>();
services.AddScoped<IReportFormatter, HtmlReportFormatter>();
services.AddScoped<IReportFormatter, JsonReportFormatter>();

services.AddScoped<SalesReportService>();

var sp = services.BuildServiceProvider();
var reportService = sp.GetRequiredService<SalesReportService>();

// ===== Main Demo =====
Console.WriteLine("=== SOLID Principles Demo: Sales Report System ===\n");

var from = DateTime.Today.AddDays(-7);
var to = DateTime.Today;

// Generate CSV
Console.WriteLine("--- CSV Report ---");
var csv = await reportService.GenerateAsync(from, to, "CSV");
Console.WriteLine(csv[..Math.Min(300, csv.Length)]);

// Generate JSON
Console.WriteLine("\n--- JSON Report ---");
var json = await reportService.GenerateAsync(from, to, "JSON");
Console.WriteLine(json[..Math.Min(300, json.Length)]);

// Generate HTML
Console.WriteLine("\n--- HTML Report (Preview) ---");
var html = await reportService.GenerateAsync(from, to, "HTML");
Console.WriteLine(html[..Math.Min(200, html.Length)] + "...");

// Email
Console.WriteLine("\n--- Email Report ---");
await reportService.GenerateAndEmailAsync(from, to, "CSV", "manager@company.com");

Console.WriteLine("\n=== SOLID Principles ที่ใช้ในตัวอย่างนี้ ===");
Console.WriteLine("S - SalesReportService, CsvFormatter, HtmlFormatter มีหน้าที่เดียว");
Console.WriteLine("O - เพิ่ม formatter ใหม่โดยสร้าง class ใหม่ ไม่แก้ code เดิม");
Console.WriteLine("L - ทุก IReportFormatter สามารถแทนกันได้");
Console.WriteLine("I - ISalesRepository และ IReportFormatter แยก interface ชัดเจน");
Console.WriteLine("D - SalesReportService depend on interfaces ไม่ใช่ concrete classes");
```

---

## Exercises

1. **Exercise 1**: Refactor `UserAuthService` ที่ละเมิด SRP:
   - แยก PasswordHasher ออกมา
   - แยก TokenGenerator ออกมา
   - แยก LoginAuditLogger ออกมา

2. **Exercise 2**: Apply OCP ให้กับ `TaxCalculator`:
   - ต้องรองรับภาษีของหลายประเทศ
   - เพิ่มประเทศใหม่โดยไม่แก้ existing code

3. **Exercise 3**: แก้ LSP violation ใน `Bird` hierarchy:
   - Bird.Fly() → Eagle (สามารถบินได้) vs Penguin (บินไม่ได้)

4. **Exercise 4**: Apply ISP ให้กับ `IPrinter` interface ที่ใหญ่เกินไป:
   - Print, Scan, Fax, Email, Staple
   - แยกออกเป็น interface ย่อยๆ

5. **Exercise 5**: สร้าง `NotificationService` ที่ follow DIP ทุก dependencies ต้อง inject ผ่าน constructor

---

## สรุป

SOLID Principles เป็นรากฐานของ good software design:

| Principle | ประโยชน์ |
|-----------|----------|
| **SRP** | โค้ดอ่านง่าย แก้ไขได้ง่าย |
| **OCP** | ขยายได้โดยไม่ทำลายของเดิม |
| **LSP** | Polymorphism ทำงานได้ถูกต้อง |
| **ISP** | ลด coupling, implement เฉพาะที่ใช้ |
| **DIP** | ทดสอบได้ง่าย, swap implementations ได้ |

การ follow SOLID จะทำให้โค้ดมีคุณสมบัติที่ดี:
- **Maintainable** - แก้ไขได้ง่าย
- **Testable** - unit test ได้ง่าย
- **Extensible** - ขยายได้โดยไม่ทำลาย
- **Reusable** - ใช้ซ้ำได้ในที่ต่างๆ

---

## Part ถัดไป

ใน Part 81 เราจะเรียนรู้เรื่อง **Event-Driven Architecture** การออกแบบระบบโดยใช้ events เป็นศูนย์กลาง

---

*Part 80/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
