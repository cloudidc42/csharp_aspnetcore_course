# Part 78: Design Patterns: Structural

## เนื้อหาใน Part นี้
- Adapter Pattern
- Decorator Pattern
- Facade Pattern
- Proxy Pattern
- Composite Pattern
- โปรแกรมตัวอย่าง: Payment Gateway Adapter

---

## Structural Patterns

Structural Patterns เกี่ยวกับการจัดโครงสร้างของ classes และ objects เพื่อสร้าง structure ใหม่

```
Structural Patterns:
├── Adapter     - แปลง interface ที่ไม่เข้ากันให้ใช้งานร่วมกันได้
├── Decorator   - เพิ่ม behavior โดยไม่แก้ original class
├── Facade      - interface ง่ายๆ สำหรับ subsystem ซับซ้อน
├── Proxy       - ตัวแทนที่ควบคุมการเข้าถึง object จริง
├── Composite   - จัดการ single objects และ groups เหมือนกัน
├── Bridge      - แยก abstraction จาก implementation
└── Flyweight   - share objects เพื่อลดการใช้ memory
```

---

## 1. Adapter Pattern

Adapter แปลง interface ของ class ให้เข้ากับ interface ที่ client ต้องการ

### ปัญหาที่ Adapter แก้

```csharp
// Legacy Payment System (ไม่สามารถแก้ไขได้)
public class LegacyPaymentSystem
{
    public int ProcessPayment(string cardNumber, string expiry, string cvv, float amount)
    {
        Console.WriteLine($"[Legacy] Processing payment: ${amount} for card ending in {cardNumber[^4..]}");
        // คืน transaction code (int)
        return new Random().Next(100000, 999999);
    }
    
    public bool RefundTransaction(int transactionCode, float refundAmount)
    {
        Console.WriteLine($"[Legacy] Refunding ${refundAmount} for transaction {transactionCode}");
        return true;
    }
    
    public float GetAccountBalance(string accountNumber)
    {
        return 50000f;
    }
}

// New Payment Interface ที่ระบบใหม่ต้องการ
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(RefundRequest request);
    Task<BalanceResult> GetBalanceAsync(string accountId);
}

public record PaymentRequest(
    string CardNumber, string ExpiryDate, string Cvv,
    decimal Amount, string Currency, string Description);

public record PaymentResult(bool Success, string TransactionId, string Message);
public record RefundRequest(string TransactionId, decimal Amount, string Reason);
public record RefundResult(bool Success, string RefundId, string Message);
public record BalanceResult(decimal Balance, string Currency);

// Adapter - แปลง LegacyPaymentSystem → IPaymentGateway
public class LegacyPaymentAdapter : IPaymentGateway
{
    private readonly LegacyPaymentSystem _legacySystem;
    
    public LegacyPaymentAdapter(LegacyPaymentSystem legacySystem)
    {
        _legacySystem = legacySystem;
    }
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest request)
    {
        try
        {
            // แปลง decimal → float, string → int/float
            var transactionCode = _legacySystem.ProcessPayment(
                request.CardNumber,
                request.ExpiryDate,
                request.Cvv,
                (float)request.Amount);
            
            return new PaymentResult(
                Success: true,
                TransactionId: transactionCode.ToString(),
                Message: "Payment processed successfully");
        }
        catch (Exception ex)
        {
            return new PaymentResult(false, "", $"Payment failed: {ex.Message}");
        }
    }
    
    public async Task<RefundResult> RefundAsync(RefundRequest request)
    {
        if (!int.TryParse(request.TransactionId, out int txCode))
            return new RefundResult(false, "", "Invalid transaction ID format");
        
        var success = _legacySystem.RefundTransaction(txCode, (float)request.Amount);
        
        return new RefundResult(
            Success: success,
            RefundId: $"REF-{txCode}",
            Message: success ? "Refund successful" : "Refund failed");
    }
    
    public async Task<BalanceResult> GetBalanceAsync(string accountId)
    {
        var balance = _legacySystem.GetAccountBalance(accountId);
        return new BalanceResult((decimal)balance, "THB");
    }
}
```

---

## 2. Decorator Pattern

Decorator เพิ่ม functionality ให้ object โดยห่อมัน โดยไม่ต้องแก้ class เดิม

```csharp
// Component Interface
public interface IEmailSender
{
    Task SendAsync(EmailMessage message, CancellationToken ct = default);
}

public record EmailMessage(
    string To, string Subject, string Body, 
    string From = "noreply@example.com",
    bool IsHtml = false);

// Concrete Component
public class SmtpEmailSender : IEmailSender
{
    private readonly SmtpSettings _settings;
    
    public SmtpEmailSender(SmtpSettings settings)
    {
        _settings = settings;
    }
    
    public async Task SendAsync(EmailMessage message, CancellationToken ct)
    {
        Console.WriteLine($"[SMTP] ส่งอีเมลไปยัง {message.To}: {message.Subject}");
        await Task.Delay(100, ct); // Simulate sending
    }
}

// Base Decorator
public abstract class EmailSenderDecorator : IEmailSender
{
    protected readonly IEmailSender _wrapped;
    
    protected EmailSenderDecorator(IEmailSender emailSender)
    {
        _wrapped = emailSender;
    }
    
    public virtual async Task SendAsync(EmailMessage message, CancellationToken ct)
        => await _wrapped.SendAsync(message, ct);
}

// Concrete Decorators
public class LoggingEmailDecorator : EmailSenderDecorator
{
    private readonly ILogger _logger;
    
    public LoggingEmailDecorator(IEmailSender emailSender, ILogger logger)
        : base(emailSender)
    {
        _logger = logger;
    }
    
    public override async Task SendAsync(EmailMessage message, CancellationToken ct)
    {
        _logger.LogInformation(
            "กำลังส่งอีเมลไปยัง {To} เรื่อง: {Subject}", message.To, message.Subject);
        
        var sw = Stopwatch.StartNew();
        try
        {
            await base.SendAsync(message, ct);
            sw.Stop();
            _logger.LogInformation(
                "ส่งอีเมลสำเร็จ ใช้เวลา {Ms}ms", sw.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ส่งอีเมลล้มเหลว");
            throw;
        }
    }
}

public class RetryEmailDecorator : EmailSenderDecorator
{
    private readonly int _maxRetries;
    private readonly TimeSpan _delay;
    
    public RetryEmailDecorator(IEmailSender emailSender, int maxRetries = 3,
        TimeSpan? delay = null) : base(emailSender)
    {
        _maxRetries = maxRetries;
        _delay = delay ?? TimeSpan.FromSeconds(1);
    }
    
    public override async Task SendAsync(EmailMessage message, CancellationToken ct)
    {
        int attempt = 0;
        while (true)
        {
            try
            {
                await base.SendAsync(message, ct);
                return;
            }
            catch (Exception ex) when (attempt < _maxRetries)
            {
                attempt++;
                Console.WriteLine($"[Retry] ความพยายามที่ {attempt}/{_maxRetries}: {ex.Message}");
                await Task.Delay(_delay, ct);
            }
        }
    }
}

public class ThrottleEmailDecorator : EmailSenderDecorator
{
    private readonly int _maxPerMinute;
    private readonly Queue<DateTime> _sentTimes = new();
    private readonly SemaphoreSlim _semaphore = new(1, 1);
    
    public ThrottleEmailDecorator(IEmailSender emailSender, int maxPerMinute = 60)
        : base(emailSender)
    {
        _maxPerMinute = maxPerMinute;
    }
    
    public override async Task SendAsync(EmailMessage message, CancellationToken ct)
    {
        await _semaphore.WaitAsync(ct);
        try
        {
            var now = DateTime.UtcNow;
            // ลบ timestamps ที่เกิน 1 นาทีที่แล้ว
            while (_sentTimes.Count > 0 && 
                   (now - _sentTimes.Peek()).TotalMinutes >= 1)
                _sentTimes.Dequeue();
            
            if (_sentTimes.Count >= _maxPerMinute)
                throw new RateLimitException($"เกินขีดจำกัด {_maxPerMinute} อีเมล/นาที");
            
            _sentTimes.Enqueue(now);
        }
        finally
        {
            _semaphore.Release();
        }
        
        await base.SendAsync(message, ct);
    }
}

public class SpamCheckEmailDecorator : EmailSenderDecorator
{
    private readonly string[] _blockedDomains = { "spam.com", "trash.net", "blocked.org" };
    
    public SpamCheckEmailDecorator(IEmailSender emailSender) : base(emailSender) { }
    
    public override async Task SendAsync(EmailMessage message, CancellationToken ct)
    {
        var domain = message.To.Split('@').LastOrDefault();
        if (_blockedDomains.Contains(domain))
            throw new BlockedDomainException($"Domain {domain} ถูก block");
        
        await base.SendAsync(message, ct);
    }
}

// Decorator Chain (Fluent Builder)
public class EmailSenderBuilder
{
    private IEmailSender _sender;
    
    public EmailSenderBuilder(SmtpSettings settings)
    {
        _sender = new SmtpEmailSender(settings);
    }
    
    public EmailSenderBuilder WithLogging(ILogger logger)
    {
        _sender = new LoggingEmailDecorator(_sender, logger);
        return this;
    }
    
    public EmailSenderBuilder WithRetry(int maxRetries = 3)
    {
        _sender = new RetryEmailDecorator(_sender, maxRetries);
        return this;
    }
    
    public EmailSenderBuilder WithThrottle(int maxPerMinute = 60)
    {
        _sender = new ThrottleEmailDecorator(_sender, maxPerMinute);
        return this;
    }
    
    public EmailSenderBuilder WithSpamCheck()
    {
        _sender = new SpamCheckEmailDecorator(_sender);
        return this;
    }
    
    public IEmailSender Build() => _sender;
}
```

---

## 3. Facade Pattern

Facade ให้ interface ง่ายๆ สำหรับ subsystem ที่ซับซ้อน

```csharp
// Complex subsystems
public class OrderValidator
{
    public (bool IsValid, List<string> Errors) Validate(OrderRequest request)
    {
        var errors = new List<string>();
        if (string.IsNullOrEmpty(request.CustomerId)) errors.Add("CustomerId ต้องไม่ว่าง");
        if (!request.Items.Any()) errors.Add("ต้องมีสินค้าอย่างน้อย 1 รายการ");
        return (!errors.Any(), errors);
    }
}

public class InventorySystem
{
    public async Task<bool> CheckAndReserveAsync(List<OrderItem> items)
    {
        Console.WriteLine($"[Inventory] ตรวจสอบและจอง {items.Count} รายการ");
        await Task.Delay(50);
        return true;
    }
    
    public async Task ReleaseReservationAsync(List<OrderItem> items)
    {
        Console.WriteLine($"[Inventory] ยกเลิกการจอง {items.Count} รายการ");
        await Task.Delay(30);
    }
}

public class PaymentProcessor
{
    public async Task<string> ProcessAsync(decimal amount, PaymentInfo payment)
    {
        Console.WriteLine($"[Payment] ดำเนินการชำระเงิน {amount:N2} บาท");
        await Task.Delay(100);
        return $"PAY-{Guid.NewGuid():N}"[..12];
    }
    
    public async Task<bool> RefundAsync(string paymentId, decimal amount)
    {
        Console.WriteLine($"[Payment] คืนเงิน {amount:N2} บาท สำหรับ {paymentId}");
        await Task.Delay(100);
        return true;
    }
}

public class ShippingService
{
    public async Task<string> CreateShipmentAsync(Order order, Address address)
    {
        Console.WriteLine($"[Shipping] สร้างการจัดส่งไปยัง {address}");
        await Task.Delay(50);
        return $"SHIP-{Guid.NewGuid():N}"[..10];
    }
}

public class NotificationSystem
{
    public async Task SendOrderConfirmationAsync(string email, Order order)
    {
        Console.WriteLine($"[Notification] ส่งการยืนยันไปยัง {email}");
        await Task.Delay(30);
    }
    
    public async Task SendShipmentNotificationAsync(string email, string trackingNumber)
    {
        Console.WriteLine($"[Notification] ส่งหมายเลขติดตาม {trackingNumber} ไปยัง {email}");
        await Task.Delay(30);
    }
}

// Facade - ซ่อนความซับซ้อนทั้งหมด
public class OrderFacade
{
    private readonly OrderValidator _validator;
    private readonly InventorySystem _inventory;
    private readonly PaymentProcessor _payment;
    private readonly ShippingService _shipping;
    private readonly NotificationSystem _notification;
    
    public OrderFacade(
        OrderValidator validator,
        InventorySystem inventory,
        PaymentProcessor payment,
        ShippingService shipping,
        NotificationSystem notification)
    {
        _validator = validator;
        _inventory = inventory;
        _payment = payment;
        _shipping = shipping;
        _notification = notification;
    }
    
    // Client เรียกแค่ method เดียว ไม่ต้องรู้ความซับซ้อนข้างใน
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        Console.WriteLine("=== เริ่มกระบวนการสั่งซื้อ ===");
        
        // 1. Validate
        var (isValid, errors) = _validator.Validate(request);
        if (!isValid)
            return new OrderResult(false, null, errors);
        
        // 2. Check & Reserve inventory
        bool reserved = await _inventory.CheckAndReserveAsync(request.Items);
        if (!reserved)
            return new OrderResult(false, null, new[] { "สินค้าไม่เพียงพอ" });
        
        try
        {
            // 3. Process payment
            var paymentId = await _payment.ProcessAsync(
                request.TotalAmount, request.PaymentInfo);
            
            // 4. Create order
            var order = new Order(request, paymentId);
            
            // 5. Create shipment
            var trackingNumber = await _shipping.CreateShipmentAsync(
                order, request.ShippingAddress);
            
            // 6. Send notifications
            await _notification.SendOrderConfirmationAsync(
                request.CustomerEmail, order);
            await _notification.SendShipmentNotificationAsync(
                request.CustomerEmail, trackingNumber);
            
            Console.WriteLine("=== สั่งซื้อสำเร็จ ===\n");
            return new OrderResult(true, order, null, trackingNumber);
        }
        catch (Exception ex)
        {
            // Compensate - ยกเลิกการจอง inventory
            await _inventory.ReleaseReservationAsync(request.Items);
            return new OrderResult(false, null, new[] { ex.Message });
        }
    }
}
```

---

## 4. Proxy Pattern

Proxy เป็นตัวแทนของ object จริง ควบคุมการเข้าถึง

```csharp
public interface IProductService
{
    Task<Product?> GetByIdAsync(Guid id);
    Task<List<Product>> SearchAsync(string query);
    Task SaveAsync(Product product);
}

// Real Service
public class ProductService : IProductService
{
    private readonly AppDbContext _context;
    
    public ProductService(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<Product?> GetByIdAsync(Guid id)
        => await _context.Products.FindAsync(id);
    
    public async Task<List<Product>> SearchAsync(string query)
        => await _context.Products
            .Where(p => p.Name.Contains(query))
            .ToListAsync();
    
    public async Task SaveAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
    }
}

// Caching Proxy
public class CachingProductServiceProxy : IProductService
{
    private readonly IProductService _realService;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheExpiry = TimeSpan.FromMinutes(5);
    
    public CachingProductServiceProxy(IProductService realService, IMemoryCache cache)
    {
        _realService = realService;
        _cache = cache;
    }
    
    public async Task<Product?> GetByIdAsync(Guid id)
    {
        var cacheKey = $"product:{id}";
        
        if (_cache.TryGetValue(cacheKey, out Product? cached))
        {
            Console.WriteLine($"[Cache HIT] Product {id}");
            return cached;
        }
        
        Console.WriteLine($"[Cache MISS] Product {id}");
        var product = await _realService.GetByIdAsync(id);
        
        if (product != null)
            _cache.Set(cacheKey, product, _cacheExpiry);
        
        return product;
    }
    
    public async Task<List<Product>> SearchAsync(string query)
    {
        var cacheKey = $"search:{query.ToLower()}";
        
        if (_cache.TryGetValue(cacheKey, out List<Product>? results))
        {
            Console.WriteLine($"[Cache HIT] Search '{query}'");
            return results!;
        }
        
        results = await _realService.SearchAsync(query);
        _cache.Set(cacheKey, results, TimeSpan.FromMinutes(1));
        return results;
    }
    
    public async Task SaveAsync(Product product)
    {
        await _realService.SaveAsync(product);
        // Invalidate cache
        _cache.Remove($"product:{product.Id}");
        Console.WriteLine($"[Cache] Invalidated product:{product.Id}");
    }
}

// Authorization Proxy
public class AuthorizedProductServiceProxy : IProductService
{
    private readonly IProductService _realService;
    private readonly ICurrentUser _currentUser;
    
    public AuthorizedProductServiceProxy(
        IProductService realService, ICurrentUser currentUser)
    {
        _realService = realService;
        _currentUser = currentUser;
    }
    
    public Task<Product?> GetByIdAsync(Guid id)
    {
        if (!_currentUser.IsAuthenticated)
            throw new UnauthorizedException("ต้องเข้าสู่ระบบก่อน");
        return _realService.GetByIdAsync(id);
    }
    
    public Task<List<Product>> SearchAsync(string query)
    {
        if (!_currentUser.IsAuthenticated)
            throw new UnauthorizedException("ต้องเข้าสู่ระบบก่อน");
        return _realService.SearchAsync(query);
    }
    
    public Task SaveAsync(Product product)
    {
        if (!_currentUser.IsInRole("Admin") && !_currentUser.IsInRole("Manager"))
            throw new ForbiddenException("ไม่มีสิทธิ์บันทึกสินค้า");
        return _realService.SaveAsync(product);
    }
}
```

---

## 5. Composite Pattern

Composite ให้จัดการ individual objects และ collections ด้วย interface เดียวกัน

```csharp
// Component Interface
public interface IFileSystemItem
{
    string Name { get; }
    long GetSize();
    void Display(int depth = 0);
    IFileSystemItem? Find(string name);
}

// Leaf
public class FileItem : IFileSystemItem
{
    public string Name { get; }
    private readonly long _size;
    
    public FileItem(string name, long sizeBytes)
    {
        Name = name;
        _size = sizeBytes;
    }
    
    public long GetSize() => _size;
    
    public void Display(int depth = 0)
    {
        var indent = new string(' ', depth * 2);
        Console.WriteLine($"{indent}📄 {Name} ({FormatSize(_size)})");
    }
    
    public IFileSystemItem? Find(string name)
        => Name.Equals(name, StringComparison.OrdinalIgnoreCase) ? this : null;
    
    private static string FormatSize(long bytes)
    {
        if (bytes < 1024) return $"{bytes} B";
        if (bytes < 1024 * 1024) return $"{bytes / 1024:N0} KB";
        return $"{bytes / (1024 * 1024):N0} MB";
    }
}

// Composite
public class DirectoryItem : IFileSystemItem
{
    public string Name { get; }
    private readonly List<IFileSystemItem> _children = new();
    
    public DirectoryItem(string name)
    {
        Name = name;
    }
    
    public void Add(IFileSystemItem item) => _children.Add(item);
    public void Remove(IFileSystemItem item) => _children.Remove(item);
    
    public long GetSize() => _children.Sum(c => c.GetSize());
    
    public void Display(int depth = 0)
    {
        var indent = new string(' ', depth * 2);
        Console.WriteLine($"{indent}📁 {Name}/ ({FormatSize(GetSize())})");
        foreach (var child in _children)
            child.Display(depth + 1);
    }
    
    public IFileSystemItem? Find(string name)
    {
        if (Name.Equals(name, StringComparison.OrdinalIgnoreCase)) return this;
        return _children.Select(c => c.Find(name)).FirstOrDefault(r => r != null);
    }
    
    private static string FormatSize(long bytes)
    {
        if (bytes < 1024) return $"{bytes} B";
        if (bytes < 1024 * 1024) return $"{bytes / 1024:N0} KB";
        return $"{bytes / (1024 * 1024):N0} MB";
    }
}
```

---

## โปรแกรมตัวอย่าง: Payment Gateway Adapter

```csharp
// ===== Existing Payment Providers =====
// StripeAPI (external library - ไม่สามารถแก้ไขได้)
public class StripePaymentApi
{
    public string CreateCharge(
        long amountInCents, string currency, string source, string description)
    {
        Console.WriteLine($"[Stripe] Charging {amountInCents} {currency.ToUpper()} cents");
        return $"ch_{Guid.NewGuid():N}"[..20];
    }
    
    public bool Refund(string chargeId, long amountInCents)
    {
        Console.WriteLine($"[Stripe] Refunding {amountInCents} cents from {chargeId}");
        return true;
    }
    
    public Dictionary<string, object> RetrieveBalance()
        => new() { ["available"] = 100000L, ["pending"] = 5000L };
}

// OmiseAPI (external library - ไม่สามารถแก้ไขได้)
public class OmisePaymentApi
{
    public OmiseChargeResult CreateCharge(OmiseChargeRequest request)
    {
        Console.WriteLine($"[Omise] Charging {request.Amount} {request.Currency}");
        return new OmiseChargeResult 
        { 
            Id = $"chrg_{Guid.NewGuid():N}"[..20],
            Status = "successful",
            Amount = request.Amount
        };
    }
    
    public bool CreateRefund(string chargeId, int amount)
    {
        Console.WriteLine($"[Omise] Refunding {amount} from {chargeId}");
        return true;
    }
}

public class OmiseChargeRequest
{
    public int Amount { get; set; }
    public string Currency { get; set; } = "thb";
    public string? Token { get; set; }
    public string? Description { get; set; }
}

public class OmiseChargeResult
{
    public string Id { get; set; } = "";
    public string Status { get; set; } = "";
    public int Amount { get; set; }
}

// ===== Our System's Interface =====
public interface IPaymentGateway
{
    string Name { get; }
    Task<PaymentResult> ChargeAsync(PaymentRequest request);
    Task<RefundResult> RefundAsync(string transactionId, decimal amount);
    Task<decimal> GetBalanceAsync();
}

public record PaymentRequest(
    decimal Amount, string Currency, string Token, 
    string Description = "Payment");
public record PaymentResult(bool Success, string TransactionId, string? ErrorMessage = null);
public record RefundResult(bool Success, string? RefundId = null, string? ErrorMessage = null);

// ===== Adapters =====
public class StripeAdapter : IPaymentGateway
{
    private readonly StripePaymentApi _stripe;
    
    public string Name => "Stripe";
    
    public StripeAdapter(StripePaymentApi stripe)
    {
        _stripe = stripe;
    }
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest request)
    {
        try
        {
            // แปลง decimal บาท → long สตางค์
            var amountInCents = (long)(request.Amount * 100);
            
            var chargeId = _stripe.CreateCharge(
                amountInCents,
                request.Currency.ToLower(),
                request.Token,
                request.Description);
            
            return new PaymentResult(true, chargeId);
        }
        catch (Exception ex)
        {
            return new PaymentResult(false, "", ex.Message);
        }
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        var amountInCents = (long)(amount * 100);
        var success = _stripe.Refund(transactionId, amountInCents);
        return new RefundResult(success, success ? $"re_{transactionId}" : null);
    }
    
    public async Task<decimal> GetBalanceAsync()
    {
        var balance = _stripe.RetrieveBalance();
        var available = (long)balance["available"];
        return available / 100m; // แปลง cents → baht
    }
}

public class OmiseAdapter : IPaymentGateway
{
    private readonly OmisePaymentApi _omise;
    
    public string Name => "Omise";
    
    public OmiseAdapter(OmisePaymentApi omise)
    {
        _omise = omise;
    }
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest request)
    {
        try
        {
            // Omise ใช้ int สตางค์ (100 สตางค์ = 1 บาท)
            var result = _omise.CreateCharge(new OmiseChargeRequest
            {
                Amount = (int)(request.Amount * 100),
                Currency = request.Currency.ToLower(),
                Token = request.Token,
                Description = request.Description
            });
            
            var success = result.Status == "successful";
            return new PaymentResult(success, result.Id,
                success ? null : $"Charge status: {result.Status}");
        }
        catch (Exception ex)
        {
            return new PaymentResult(false, "", ex.Message);
        }
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        var success = _omise.CreateRefund(transactionId, (int)(amount * 100));
        return new RefundResult(success);
    }
    
    public async Task<decimal> GetBalanceAsync()
    {
        // Omise ไม่มี balance API โดยตรง ใช้ default value
        return 0m;
    }
}

// ===== Decorator สำหรับเพิ่ม features =====
public class LoggingPaymentDecorator : IPaymentGateway
{
    private readonly IPaymentGateway _gateway;
    
    public string Name => _gateway.Name;
    
    public LoggingPaymentDecorator(IPaymentGateway gateway)
    {
        _gateway = gateway;
    }
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest request)
    {
        Console.WriteLine($"[{Name}] เริ่มชำระเงิน {request.Amount:N2} {request.Currency}");
        var result = await _gateway.ChargeAsync(request);
        Console.WriteLine($"[{Name}] ผลลัพธ์: {(result.Success ? "สำเร็จ" : "ล้มเหลว")} " +
            $"TransactionId: {result.TransactionId}");
        return result;
    }
    
    public async Task<RefundResult> RefundAsync(string transactionId, decimal amount)
    {
        Console.WriteLine($"[{Name}] คืนเงิน {amount:N2} สำหรับ {transactionId}");
        return await _gateway.RefundAsync(transactionId, amount);
    }
    
    public Task<decimal> GetBalanceAsync() => _gateway.GetBalanceAsync();
}

// ===== Facade สำหรับ Payment System =====
public class PaymentFacade
{
    private readonly Dictionary<string, IPaymentGateway> _gateways;
    private readonly string _defaultGateway;
    
    public PaymentFacade(IEnumerable<IPaymentGateway> gateways, string defaultGateway)
    {
        _gateways = gateways.ToDictionary(g => g.Name.ToLower());
        _defaultGateway = defaultGateway.ToLower();
    }
    
    public async Task<PaymentResult> PayAsync(
        decimal amount, string token, string currency = "thb",
        string? preferredGateway = null)
    {
        var gatewayName = (preferredGateway ?? _defaultGateway).ToLower();
        
        if (!_gateways.TryGetValue(gatewayName, out var gateway))
            throw new InvalidOperationException($"ไม่พบ payment gateway: {gatewayName}");
        
        return await gateway.ChargeAsync(
            new PaymentRequest(amount, currency, token));
    }
    
    public async Task<RefundResult> RefundAsync(
        string transactionId, decimal amount, string? gateway = null)
    {
        var g = _gateways.GetValueOrDefault((gateway ?? _defaultGateway).ToLower())
            ?? throw new InvalidOperationException("ไม่พบ gateway");
        return await g.RefundAsync(transactionId, amount);
    }
}

// ===== Main Program =====
Console.WriteLine("=== Payment Gateway Adapter Demo ===\n");

// สร้าง Adapters
var stripeAdapter = new LoggingPaymentDecorator(
    new StripeAdapter(new StripePaymentApi()));
var omiseAdapter = new LoggingPaymentDecorator(
    new OmiseAdapter(new OmisePaymentApi()));

// ใช้งานผ่าน unified interface
Console.WriteLine("1. ชำระเงินผ่าน Stripe:");
var stripeResult = await stripeAdapter.ChargeAsync(
    new PaymentRequest(1500m, "thb", "tok_test_123"));
Console.WriteLine($"   Transaction: {stripeResult.TransactionId}\n");

Console.WriteLine("2. ชำระเงินผ่าน Omise:");
var omiseResult = await omiseAdapter.ChargeAsync(
    new PaymentRequest(2500m, "thb", "tokn_test_456"));
Console.WriteLine($"   Transaction: {omiseResult.TransactionId}\n");

// Facade
Console.WriteLine("3. ใช้ Facade (เลือก gateway อัตโนมัติ):");
var facade = new PaymentFacade(
    new IPaymentGateway[] { stripeAdapter, omiseAdapter }, "stripe");

var facadeResult = await facade.PayAsync(3000m, "tok_789");
Console.WriteLine($"   Transaction: {facadeResult.TransactionId}\n");

// Composite - File System
Console.WriteLine("4. Composite Pattern - File System:");
var root = new DirectoryItem("project");
var src = new DirectoryItem("src");
src.Add(new FileItem("Program.cs", 2048));
src.Add(new FileItem("Startup.cs", 4096));
var controllers = new DirectoryItem("Controllers");
controllers.Add(new FileItem("HomeController.cs", 1024));
controllers.Add(new FileItem("ApiController.cs", 3072));
src.Add(controllers);
root.Add(src);
root.Add(new FileItem("README.md", 512));
root.Add(new FileItem(".gitignore", 256));
root.Display();
Console.WriteLine($"\nTotal Size: {root.GetSize():N0} bytes");
```

---

## Exercises

1. **Exercise 1**: สร้าง `CachingDecorator<T>` ที่ใช้กับ Repository ใดๆ ก็ได้

2. **Exercise 2**: Implement `CircuitBreakerProxy` ที่หยุดส่ง request เมื่อ error rate สูงเกินไป

3. **Exercise 3**: สร้าง `UIThemeFacade` ที่ซ่อนความซับซ้อนของการเปลี่ยน theme

4. **Exercise 4**: ใช้ Composite Pattern สร้าง Menu System:
   - MenuItem (leaf)
   - MenuGroup (composite)
   - เรียก `Render()` แสดง nested menu

5. **Exercise 5**: สร้าง `CloudStorageAdapter` ที่รองรับ AWS S3, Azure Blob และ Google Cloud Storage

---

## สรุป

Structural Patterns ช่วยจัดโครงสร้าง objects:
- **Adapter**: แปลง interface ที่ไม่เข้ากัน
- **Decorator**: เพิ่ม behavior โดยไม่แก้ class เดิม
- **Facade**: ซ่อนความซับซ้อน ให้ interface ง่ายขึ้น
- **Proxy**: ควบคุมการเข้าถึง object (caching, auth, logging)
- **Composite**: จัดการ tree structure ด้วย interface เดียว

---

## Part ถัดไป

ใน Part 79 เราจะเรียนรู้ **Design Patterns: Behavioral** เช่น Observer, Strategy, Command

---

*Part 78/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
