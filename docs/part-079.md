# Part 79: Design Patterns: Behavioral

## เนื้อหาใน Part นี้
- Observer Pattern
- Strategy Pattern
- Command Pattern
- Chain of Responsibility
- Template Method Pattern
- โปรแกรมตัวอย่าง: Notification System

---

## Behavioral Patterns

Behavioral Patterns เกี่ยวกับการสื่อสารระหว่าง objects และ responsibilities

```
Behavioral Patterns:
├── Observer           - แจ้งเตือนเมื่อสถานะเปลี่ยน
├── Strategy           - เลือก algorithm ได้ตอน runtime
├── Command            - ห่อ action เป็น object
├── Chain of Responsibility - ส่ง request ผ่าน chain ของ handlers
├── Template Method    - กำหนดโครงสร้าง algorithm ใน base class
├── State              - เปลี่ยน behavior ตามสถานะ
├── Iterator           - traverse collection โดยไม่รู้ implementation
├── Mediator           - ลด coupling ระหว่าง objects
├── Memento            - บันทึกและ restore สถานะ
└── Visitor            - เพิ่ม operations โดยไม่แก้ class
```

---

## 1. Observer Pattern

Observer กำหนดความสัมพันธ์ one-to-many เมื่อ subject เปลี่ยน observers ทั้งหมดถูกแจ้งเตือน

```csharp
// Event-based Observer (แบบ C#)
public class StockPrice
{
    private decimal _price;
    
    // Events แทน manual observer registration
    public event EventHandler<PriceChangedEventArgs>? PriceChanged;
    public event EventHandler<PriceChangedEventArgs>? PriceSurged; // เพิ่มขึ้น > 5%
    public event EventHandler<PriceChangedEventArgs>? PriceDropped; // ลดลง > 5%
    
    public string Symbol { get; }
    
    public decimal Price
    {
        get => _price;
        set
        {
            if (_price == value) return;
            
            var oldPrice = _price;
            _price = value;
            
            var args = new PriceChangedEventArgs(Symbol, oldPrice, value);
            PriceChanged?.Invoke(this, args);
            
            if (oldPrice > 0)
            {
                var changePercent = (value - oldPrice) / oldPrice * 100;
                if (changePercent > 5) PriceSurged?.Invoke(this, args);
                if (changePercent < -5) PriceDropped?.Invoke(this, args);
            }
        }
    }
    
    public StockPrice(string symbol, decimal initialPrice)
    {
        Symbol = symbol;
        _price = initialPrice;
    }
}

public class PriceChangedEventArgs : EventArgs
{
    public string Symbol { get; }
    public decimal OldPrice { get; }
    public decimal NewPrice { get; }
    public decimal ChangePercent => OldPrice > 0 ? (NewPrice - OldPrice) / OldPrice * 100 : 0;
    public bool IsIncrease => NewPrice > OldPrice;
    
    public PriceChangedEventArgs(string symbol, decimal oldPrice, decimal newPrice)
    {
        Symbol = symbol;
        OldPrice = oldPrice;
        NewPrice = newPrice;
    }
}

// Observers
public class PriceAlertObserver
{
    private readonly string _name;
    private readonly decimal _alertPrice;
    private readonly bool _alertWhenAbove;
    
    public PriceAlertObserver(string name, decimal alertPrice, bool alertWhenAbove)
    {
        _name = name;
        _alertPrice = alertPrice;
        _alertWhenAbove = alertWhenAbove;
    }
    
    public void OnPriceChanged(object? sender, PriceChangedEventArgs e)
    {
        bool shouldAlert = _alertWhenAbove 
            ? e.NewPrice >= _alertPrice 
            : e.NewPrice <= _alertPrice;
            
        if (shouldAlert)
            Console.WriteLine($"[Alert:{_name}] {e.Symbol} ถึง {e.NewPrice:N2} บาท " +
                $"({(e.IsIncrease ? "↑" : "↓")} {Math.Abs(e.ChangePercent):N2}%)");
    }
}

public class PortfolioTracker
{
    private readonly Dictionary<string, (int Shares, decimal AveragePrice)> _holdings = new();
    
    public void AddHolding(string symbol, int shares, decimal avgPrice)
        => _holdings[symbol] = (shares, avgPrice);
    
    public void OnPriceChanged(object? sender, PriceChangedEventArgs e)
    {
        if (!_holdings.TryGetValue(e.Symbol, out var holding)) return;
        
        var pnl = (e.NewPrice - holding.AveragePrice) * holding.Shares;
        var pnlPercent = (e.NewPrice - holding.AveragePrice) / holding.AveragePrice * 100;
        
        Console.WriteLine($"[Portfolio] {e.Symbol}: {holding.Shares} หุ้น | " +
            $"P&L: {(pnl >= 0 ? "+" : "")}{pnl:N2} ({(pnlPercent >= 0 ? "+" : "")}{pnlPercent:N2}%)");
    }
}

// IObservable Interface แบบ classic
public interface IObservable<T>
{
    void Subscribe(IObserver<T> observer);
    void Unsubscribe(IObserver<T> observer);
    void NotifyAll(T data);
}

public interface IObserver<T>
{
    void Update(T data);
}
```

---

## 2. Strategy Pattern

Strategy ให้เลือก algorithm ได้ตอน runtime โดยไม่แก้ client code

```csharp
// Strategy Interface
public interface ISortStrategy
{
    void Sort(List<int> data);
    string Name { get; }
}

// Concrete Strategies
public class BubbleSortStrategy : ISortStrategy
{
    public string Name => "Bubble Sort";
    
    public void Sort(List<int> data)
    {
        int n = data.Count;
        for (int i = 0; i < n - 1; i++)
            for (int j = 0; j < n - i - 1; j++)
                if (data[j] > data[j + 1])
                    (data[j], data[j + 1]) = (data[j + 1], data[j]);
    }
}

public class QuickSortStrategy : ISortStrategy
{
    public string Name => "Quick Sort";
    
    public void Sort(List<int> data)
    {
        QuickSort(data, 0, data.Count - 1);
    }
    
    private void QuickSort(List<int> data, int low, int high)
    {
        if (low < high)
        {
            int pivot = Partition(data, low, high);
            QuickSort(data, low, pivot - 1);
            QuickSort(data, pivot + 1, high);
        }
    }
    
    private int Partition(List<int> data, int low, int high)
    {
        int pivot = data[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++)
        {
            if (data[j] <= pivot)
            {
                i++;
                (data[i], data[j]) = (data[j], data[i]);
            }
        }
        
        (data[i + 1], data[high]) = (data[high], data[i + 1]);
        return i + 1;
    }
}

public class LinqSortStrategy : ISortStrategy
{
    public string Name => "LINQ Sort";
    
    public void Sort(List<int> data)
    {
        var sorted = data.Order().ToList();
        data.Clear();
        data.AddRange(sorted);
    }
}

// Context
public class DataSorter
{
    private ISortStrategy _strategy;
    
    public DataSorter(ISortStrategy strategy)
    {
        _strategy = strategy;
    }
    
    // เปลี่ยน strategy ได้ตอน runtime
    public void SetStrategy(ISortStrategy strategy)
    {
        Console.WriteLine($"เปลี่ยน strategy เป็น {strategy.Name}");
        _strategy = strategy;
    }
    
    public List<int> Sort(List<int> data)
    {
        var copy = new List<int>(data);
        var sw = Stopwatch.StartNew();
        _strategy.Sort(copy);
        sw.Stop();
        Console.WriteLine($"[{_strategy.Name}] เรียงลำดับ {data.Count} elements ใช้เวลา {sw.ElapsedMicroseconds} μs");
        return copy;
    }
}

// Strategy Factory - เลือก strategy ตาม data size
public class AdaptiveSortStrategyFactory
{
    public static ISortStrategy GetBestStrategy(int dataSize)
    {
        return dataSize switch
        {
            < 10 => new BubbleSortStrategy(),
            < 1000 => new QuickSortStrategy(),
            _ => new LinqSortStrategy()
        };
    }
}

// Payment Strategy
public interface IPaymentStrategy
{
    string MethodName { get; }
    bool Validate(decimal amount);
    Task<PaymentResult> PayAsync(decimal amount, Dictionary<string, string> details);
}

public class CreditCardStrategy : IPaymentStrategy
{
    public string MethodName => "Credit Card";
    
    public bool Validate(decimal amount) => amount > 0 && amount <= 500000m;
    
    public async Task<PaymentResult> PayAsync(decimal amount, Dictionary<string, string> details)
    {
        Console.WriteLine($"[Credit Card] ชำระ {amount:N2} บาท ด้วยบัตร {details.GetValueOrDefault("card_number", "****")}");
        await Task.Delay(200);
        return new PaymentResult(true, $"CC-{Guid.NewGuid():N}"[..12]);
    }
}

public class BankTransferStrategy : IPaymentStrategy
{
    public string MethodName => "Bank Transfer";
    
    public bool Validate(decimal amount) => amount >= 1m;
    
    public async Task<PaymentResult> PayAsync(decimal amount, Dictionary<string, string> details)
    {
        Console.WriteLine($"[Bank Transfer] โอนเงิน {amount:N2} บาท ไปยัง {details.GetValueOrDefault("account", "unknown")}");
        await Task.Delay(500);
        return new PaymentResult(true, $"TXN-{Guid.NewGuid():N}"[..12]);
    }
}

public class QRPaymentStrategy : IPaymentStrategy
{
    public string MethodName => "QR Payment";
    
    public bool Validate(decimal amount) => amount > 0 && amount <= 100000m;
    
    public async Task<PaymentResult> PayAsync(decimal amount, Dictionary<string, string> details)
    {
        Console.WriteLine($"[QR] ชำระ {amount:N2} บาท ผ่าน QR Code");
        await Task.Delay(300);
        return new PaymentResult(true, $"QR-{Guid.NewGuid():N}"[..12]);
    }
}
```

---

## 3. Command Pattern

Command ห่อ request เป็น object ทำให้ parameterize, queue, log และ undo/redo ได้

```csharp
// Command Interface
public interface ICommand
{
    void Execute();
    void Undo();
    string Description { get; }
}

// Receiver
public class TextDocument
{
    private readonly StringBuilder _content = new();
    
    public string Content => _content.ToString();
    
    public void InsertText(int position, string text)
    {
        _content.Insert(position, text);
        Console.WriteLine($"[Doc] เพิ่มข้อความ '{text}' ที่ตำแหน่ง {position}");
    }
    
    public void DeleteText(int position, int length)
    {
        var deleted = _content.ToString().Substring(position, Math.Min(length, _content.Length - position));
        _content.Remove(position, Math.Min(length, _content.Length - position));
        Console.WriteLine($"[Doc] ลบข้อความ '{deleted}' ที่ตำแหน่ง {position}");
    }
    
    public void ReplaceText(int position, int length, string newText)
    {
        DeleteText(position, length);
        InsertText(position, newText);
    }
}

// Concrete Commands
public class InsertTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly string _text;
    
    public string Description => $"เพิ่ม '{_text}' ที่ตำแหน่ง {_position}";
    
    public InsertTextCommand(TextDocument document, int position, string text)
    {
        _document = document;
        _position = position;
        _text = text;
    }
    
    public void Execute() => _document.InsertText(_position, _text);
    
    public void Undo() => _document.DeleteText(_position, _text.Length);
}

public class DeleteTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly int _length;
    private string _deletedText = "";
    
    public string Description => $"ลบ {_length} ตัวอักษรที่ตำแหน่ง {_position}";
    
    public DeleteTextCommand(TextDocument document, int position, int length)
    {
        _document = document;
        _position = position;
        _length = length;
    }
    
    public void Execute()
    {
        _deletedText = _document.Content.Substring(
            _position, Math.Min(_length, _document.Content.Length - _position));
        _document.DeleteText(_position, _length);
    }
    
    public void Undo() => _document.InsertText(_position, _deletedText);
}

// Invoker - จัดการ command history
public class CommandHistory
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();
    private readonly int _maxHistory;
    
    public CommandHistory(int maxHistory = 100)
    {
        _maxHistory = maxHistory;
    }
    
    public void Execute(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear(); // Clear redo stack เมื่อมี command ใหม่
        
        // จำกัด history size
        if (_undoStack.Count > _maxHistory)
        {
            var items = _undoStack.ToArray();
            _undoStack.Clear();
            foreach (var item in items.Take(_maxHistory))
                _undoStack.Push(item);
        }
        
        Console.WriteLine($"[History] Execute: {command.Description}");
    }
    
    public bool Undo()
    {
        if (!_undoStack.TryPop(out var command)) return false;
        
        command.Undo();
        _redoStack.Push(command);
        Console.WriteLine($"[History] Undo: {command.Description}");
        return true;
    }
    
    public bool Redo()
    {
        if (!_redoStack.TryPop(out var command)) return false;
        
        command.Execute();
        _undoStack.Push(command);
        Console.WriteLine($"[History] Redo: {command.Description}");
        return true;
    }
    
    public bool CanUndo => _undoStack.Count > 0;
    public bool CanRedo => _redoStack.Count > 0;
    
    public IEnumerable<string> GetHistory() 
        => _undoStack.Select(c => c.Description);
}
```

---

## 4. Chain of Responsibility

Chain ส่ง request ผ่าน chain ของ handlers แต่ละตัวตัดสินใจว่าจะจัดการหรือส่งต่อ

```csharp
// Handler interface
public abstract class RequestHandler
{
    protected RequestHandler? _next;
    
    public RequestHandler SetNext(RequestHandler next)
    {
        _next = next;
        return next;
    }
    
    public abstract Task<bool> HandleAsync(HttpContext context);
}

// Middleware-style handlers
public class AuthenticationHandler : RequestHandler
{
    public override async Task<bool> HandleAsync(HttpContext context)
    {
        var token = context.Request.Headers["Authorization"].FirstOrDefault();
        
        if (string.IsNullOrEmpty(token))
        {
            Console.WriteLine("[Auth] ไม่มี Authorization header");
            context.Response.StatusCode = 401;
            return false;
        }
        
        Console.WriteLine("[Auth] Token ถูกต้อง");
        context.Items["UserId"] = "user-123";
        
        return _next == null || await _next.HandleAsync(context);
    }
}

public class AuthorizationHandler : RequestHandler
{
    private readonly string _requiredRole;
    
    public AuthorizationHandler(string requiredRole)
    {
        _requiredRole = requiredRole;
    }
    
    public override async Task<bool> HandleAsync(HttpContext context)
    {
        var userRoles = context.Items["Roles"] as string[] ?? Array.Empty<string>();
        
        if (!userRoles.Contains(_requiredRole))
        {
            Console.WriteLine($"[Authz] ไม่มี role: {_requiredRole}");
            context.Response.StatusCode = 403;
            return false;
        }
        
        Console.WriteLine($"[Authz] มี role {_requiredRole}");
        return _next == null || await _next.HandleAsync(context);
    }
}

public class RateLimitHandler : RequestHandler
{
    private readonly Dictionary<string, Queue<DateTime>> _requests = new();
    private readonly int _maxRequests;
    private readonly TimeSpan _window;
    
    public RateLimitHandler(int maxRequests = 100, int windowSeconds = 60)
    {
        _maxRequests = maxRequests;
        _window = TimeSpan.FromSeconds(windowSeconds);
    }
    
    public override async Task<bool> HandleAsync(HttpContext context)
    {
        var clientId = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        
        if (!_requests.ContainsKey(clientId))
            _requests[clientId] = new Queue<DateTime>();
        
        var queue = _requests[clientId];
        var now = DateTime.UtcNow;
        
        while (queue.Count > 0 && (now - queue.Peek()) > _window)
            queue.Dequeue();
        
        if (queue.Count >= _maxRequests)
        {
            Console.WriteLine($"[RateLimit] {clientId} เกิน limit");
            context.Response.StatusCode = 429;
            return false;
        }
        
        queue.Enqueue(now);
        Console.WriteLine($"[RateLimit] {clientId}: {queue.Count}/{_maxRequests} requests");
        
        return _next == null || await _next.HandleAsync(context);
    }
}

// Discount Chain
public abstract class DiscountHandler
{
    protected DiscountHandler? _next;
    
    public DiscountHandler SetNext(DiscountHandler next)
    {
        _next = next;
        return next;
    }
    
    public abstract decimal CalculateDiscount(OrderContext context);
}

public class OrderContext
{
    public decimal OrderTotal { get; set; }
    public string CustomerTier { get; set; } = "Regular";
    public int ItemCount { get; set; }
    public bool HasCoupon { get; set; }
    public string? CouponCode { get; set; }
    public decimal AppliedDiscount { get; set; }
}

public class VipDiscountHandler : DiscountHandler
{
    public override decimal CalculateDiscount(OrderContext context)
    {
        if (context.CustomerTier == "VIP")
        {
            var discount = context.OrderTotal * 0.20m;
            context.AppliedDiscount += discount;
            Console.WriteLine($"[VIP Discount] -{discount:N2} บาท (20%)");
        }
        return _next?.CalculateDiscount(context) ?? context.AppliedDiscount;
    }
}

public class BulkDiscountHandler : DiscountHandler
{
    public override decimal CalculateDiscount(OrderContext context)
    {
        if (context.ItemCount >= 10)
        {
            var discount = context.OrderTotal * 0.10m;
            context.AppliedDiscount += discount;
            Console.WriteLine($"[Bulk Discount] -{discount:N2} บาท (10% สำหรับ {context.ItemCount} รายการ)");
        }
        return _next?.CalculateDiscount(context) ?? context.AppliedDiscount;
    }
}

public class CouponDiscountHandler : DiscountHandler
{
    private readonly Dictionary<string, decimal> _coupons = new()
    {
        ["SAVE100"] = 100m,
        ["SUMMER20"] = 0.20m,
        ["WELCOME50"] = 50m
    };
    
    public override decimal CalculateDiscount(OrderContext context)
    {
        if (context.HasCoupon && context.CouponCode != null &&
            _coupons.TryGetValue(context.CouponCode, out var value))
        {
            var discount = value < 1 
                ? context.OrderTotal * value 
                : value;
            context.AppliedDiscount += discount;
            Console.WriteLine($"[Coupon {context.CouponCode}] -{discount:N2} บาท");
        }
        return _next?.CalculateDiscount(context) ?? context.AppliedDiscount;
    }
}
```

---

## 5. Template Method Pattern

Template Method กำหนดโครงสร้าง algorithm ใน base class ให้ subclass implement ส่วนย่อย

```csharp
// Abstract Template
public abstract class DataExporter
{
    // Template Method - กำหนด algorithm skeleton
    public sealed async Task<string> ExportAsync(IEnumerable<object> data)
    {
        Console.WriteLine($"[{GetType().Name}] เริ่มการ export");
        
        ValidateData(data);
        var prepared = await PrepareDataAsync(data);
        var header = BuildHeader();
        var content = FormatData(prepared);
        var footer = BuildFooter();
        var result = Combine(header, content, footer);
        await PostProcessAsync(result);
        
        Console.WriteLine($"[{GetType().Name}] Export สำเร็จ");
        return result;
    }
    
    protected virtual void ValidateData(IEnumerable<object> data)
    {
        if (!data.Any()) throw new InvalidOperationException("ไม่มีข้อมูล");
    }
    
    protected virtual Task<IEnumerable<object>> PrepareDataAsync(IEnumerable<object> data)
        => Task.FromResult(data);
    
    protected abstract string BuildHeader();
    protected abstract string FormatData(IEnumerable<object> data);
    
    protected virtual string BuildFooter() => "";
    
    protected virtual string Combine(string header, string content, string footer)
        => $"{header}{content}{footer}";
    
    protected virtual Task PostProcessAsync(string result) => Task.CompletedTask;
}

// Concrete Templates
public class CsvExporter : DataExporter
{
    private readonly string[] _columns;
    
    public CsvExporter(string[] columns)
    {
        _columns = columns;
    }
    
    protected override string BuildHeader()
        => string.Join(",", _columns) + "\n";
    
    protected override string FormatData(IEnumerable<object> data)
    {
        var sb = new StringBuilder();
        foreach (var item in data)
        {
            var props = item.GetType().GetProperties()
                .Select(p => $"\"{p.GetValue(item)?.ToString()?.Replace("\"", "\"\"")}\"");
            sb.AppendLine(string.Join(",", props));
        }
        return sb.ToString();
    }
}

public class JsonExporter : DataExporter
{
    protected override string BuildHeader() => "[\n";
    protected override string BuildFooter() => "\n]";
    
    protected override string FormatData(IEnumerable<object> data)
    {
        var items = data.Select(d => JsonSerializer.Serialize(d));
        return string.Join(",\n", items);
    }
    
    protected override string Combine(string header, string content, string footer)
        => header + content + footer;
}

public class HtmlExporter : DataExporter
{
    private readonly string _title;
    
    public HtmlExporter(string title = "Data Export")
    {
        _title = title;
    }
    
    protected override string BuildHeader()
        => $"<html><head><title>{_title}</title></head><body><table border='1'>\n";
    
    protected override string BuildFooter()
        => "</table></body></html>";
    
    protected override string FormatData(IEnumerable<object> data)
    {
        var sb = new StringBuilder();
        foreach (var item in data)
        {
            sb.Append("<tr>");
            foreach (var prop in item.GetType().GetProperties())
                sb.Append($"<td>{prop.GetValue(item)}</td>");
            sb.AppendLine("</tr>");
        }
        return sb.ToString();
    }
}
```

---

## โปรแกรมตัวอย่าง: Notification System

```csharp
// ===== Observer Pattern สำหรับ Notification System =====

// Notification Types
public enum NotificationType { Email, SMS, Push, Webhook, Slack }

public record Notification(
    string Title,
    string Message,
    NotificationType Type,
    string Recipient,
    Dictionary<string, string>? Metadata = null);

// Observer Interface
public interface INotificationObserver
{
    string Name { get; }
    NotificationType[] SupportedTypes { get; }
    Task HandleAsync(Notification notification, CancellationToken ct = default);
}

// Subject/Observable
public class NotificationDispatcher
{
    private readonly List<INotificationObserver> _observers = new();
    private readonly List<NotificationLog> _log = new();
    
    public void Subscribe(INotificationObserver observer)
    {
        _observers.Add(observer);
        Console.WriteLine($"[Dispatcher] ลงทะเบียน {observer.Name}");
    }
    
    public void Unsubscribe(INotificationObserver observer)
    {
        _observers.Remove(observer);
        Console.WriteLine($"[Dispatcher] ยกเลิกการลงทะเบียน {observer.Name}");
    }
    
    public async Task DispatchAsync(Notification notification, CancellationToken ct = default)
    {
        Console.WriteLine($"\n[Dispatcher] ส่ง {notification.Type} notification: {notification.Title}");
        
        var handlers = _observers
            .Where(o => o.SupportedTypes.Contains(notification.Type))
            .ToList();
        
        if (!handlers.Any())
        {
            Console.WriteLine($"[Dispatcher] ไม่พบ handler สำหรับ {notification.Type}");
            return;
        }
        
        var tasks = handlers.Select(h => h.HandleAsync(notification, ct));
        await Task.WhenAll(tasks);
        
        _log.Add(new NotificationLog(notification, DateTime.UtcNow, handlers.Count));
    }
    
    public IReadOnlyList<NotificationLog> GetLog() => _log.AsReadOnly();
}

public record NotificationLog(
    Notification Notification, DateTime SentAt, int HandlersCount);

// Concrete Observers - Strategy สำหรับแต่ละ channel
public class EmailNotificationObserver : INotificationObserver
{
    public string Name => "EmailObserver";
    public NotificationType[] SupportedTypes => new[] { NotificationType.Email };
    
    public async Task HandleAsync(Notification notification, CancellationToken ct)
    {
        Console.WriteLine($"  📧 Email → {notification.Recipient}: {notification.Title}");
        Console.WriteLine($"     {notification.Message[..Math.Min(50, notification.Message.Length)]}...");
        await Task.Delay(50, ct);
    }
}

public class SmsNotificationObserver : INotificationObserver
{
    public string Name => "SMSObserver";
    public NotificationType[] SupportedTypes => new[] { NotificationType.SMS };
    
    public async Task HandleAsync(Notification notification, CancellationToken ct)
    {
        // SMS มี limit 160 ตัวอักษร
        var truncated = notification.Message[..Math.Min(160, notification.Message.Length)];
        Console.WriteLine($"  📱 SMS → {notification.Recipient}: {truncated}");
        await Task.Delay(30, ct);
    }
}

public class PushNotificationObserver : INotificationObserver
{
    public string Name => "PushObserver";
    public NotificationType[] SupportedTypes => new[] { NotificationType.Push };
    
    public async Task HandleAsync(Notification notification, CancellationToken ct)
    {
        Console.WriteLine($"  🔔 Push → {notification.Recipient}: {notification.Title}");
        await Task.Delay(20, ct);
    }
}

public class SlackNotificationObserver : INotificationObserver
{
    private readonly string _webhookUrl;
    
    public string Name => "SlackObserver";
    public NotificationType[] SupportedTypes => 
        new[] { NotificationType.Slack, NotificationType.Webhook };
    
    public SlackNotificationObserver(string webhookUrl)
    {
        _webhookUrl = webhookUrl;
    }
    
    public async Task HandleAsync(Notification notification, CancellationToken ct)
    {
        var channel = notification.Metadata?.GetValueOrDefault("channel") ?? "#general";
        Console.WriteLine($"  💬 Slack [{channel}]: *{notification.Title}*\n     {notification.Message}");
        await Task.Delay(40, ct);
    }
}

// Command Pattern สำหรับ Notification Actions
public interface INotificationCommand
{
    string Description { get; }
    Task ExecuteAsync(CancellationToken ct = default);
}

public class SendNotificationCommand : INotificationCommand
{
    private readonly NotificationDispatcher _dispatcher;
    private readonly Notification _notification;
    
    public string Description => $"Send {_notification.Type} to {_notification.Recipient}";
    
    public SendNotificationCommand(NotificationDispatcher dispatcher, Notification notification)
    {
        _dispatcher = dispatcher;
        _notification = notification;
    }
    
    public Task ExecuteAsync(CancellationToken ct)
        => _dispatcher.DispatchAsync(_notification, ct);
}

// Template Method - Notification templates
public abstract class NotificationTemplateBuilder
{
    // Template Method
    public Notification BuildNotification(string recipient, Dictionary<string, string> data)
    {
        var title = BuildTitle(data);
        var message = BuildMessage(data);
        var type = GetNotificationType();
        var metadata = BuildMetadata(data);
        
        return new Notification(title, message, type, recipient, metadata);
    }
    
    protected abstract string BuildTitle(Dictionary<string, string> data);
    protected abstract string BuildMessage(Dictionary<string, string> data);
    protected abstract NotificationType GetNotificationType();
    protected virtual Dictionary<string, string>? BuildMetadata(Dictionary<string, string> data) => null;
}

public class OrderConfirmationEmailTemplate : NotificationTemplateBuilder
{
    protected override string BuildTitle(Dictionary<string, string> data)
        => $"ยืนยันการสั่งซื้อ #{data.GetValueOrDefault("order_number", "N/A")}";
    
    protected override string BuildMessage(Dictionary<string, string> data)
        => $"สวัสดีคุณ {data.GetValueOrDefault("customer_name", "ลูกค้า")},\n" +
           $"คำสั่งซื้อหมายเลข #{data.GetValueOrDefault("order_number")} " +
           $"ได้รับการยืนยันแล้ว\n" +
           $"ยอดรวม: {data.GetValueOrDefault("total", "0")} บาท\n" +
           $"ขอบคุณที่ใช้บริการ";
    
    protected override NotificationType GetNotificationType() => NotificationType.Email;
}

public class LowStockAlertSlackTemplate : NotificationTemplateBuilder
{
    protected override string BuildTitle(Dictionary<string, string> data)
        => $"⚠️ สินค้าใกล้หมด: {data.GetValueOrDefault("product_name")}";
    
    protected override string BuildMessage(Dictionary<string, string> data)
        => $"สินค้า *{data.GetValueOrDefault("product_name")}* เหลือเพียง " +
           $"*{data.GetValueOrDefault("stock")} ชิ้น*\n" +
           $"<{data.GetValueOrDefault("product_url")}|คลิกเพื่อดูรายละเอียด>";
    
    protected override NotificationType GetNotificationType() => NotificationType.Slack;
    
    protected override Dictionary<string, string>? BuildMetadata(Dictionary<string, string> data)
        => new() { ["channel"] = "#inventory-alerts" };
}

// Chain of Responsibility - Notification validation pipeline
public abstract class NotificationValidator
{
    protected NotificationValidator? _next;
    
    public NotificationValidator SetNext(NotificationValidator next)
    {
        _next = next;
        return next;
    }
    
    public abstract (bool IsValid, string? Error) Validate(Notification notification);
}

public class RecipientValidator : NotificationValidator
{
    public override (bool IsValid, string? Error) Validate(Notification notification)
    {
        if (string.IsNullOrWhiteSpace(notification.Recipient))
            return (false, "Recipient ต้องไม่ว่าง");
        
        if (notification.Type == NotificationType.Email && 
            !notification.Recipient.Contains('@'))
            return (false, "Email address ไม่ถูกต้อง");
        
        return _next?.Validate(notification) ?? (true, null);
    }
}

public class ContentValidator : NotificationValidator
{
    public override (bool IsValid, string? Error) Validate(Notification notification)
    {
        if (string.IsNullOrWhiteSpace(notification.Title))
            return (false, "Title ต้องไม่ว่าง");
        if (string.IsNullOrWhiteSpace(notification.Message))
            return (false, "Message ต้องไม่ว่าง");
        if (notification.Type == NotificationType.SMS && notification.Message.Length > 160)
            return (false, "SMS ต้องมีความยาวไม่เกิน 160 ตัวอักษร");
        
        return _next?.Validate(notification) ?? (true, null);
    }
}

// ===== Main Program =====
Console.WriteLine("=== Notification System Demo ===\n");

// Setup Dispatcher (Observer)
var dispatcher = new NotificationDispatcher();
dispatcher.Subscribe(new EmailNotificationObserver());
dispatcher.Subscribe(new SmsNotificationObserver());
dispatcher.Subscribe(new PushNotificationObserver());
dispatcher.Subscribe(new SlackNotificationObserver("https://hooks.slack.com/services/test"));

// Setup Validators (Chain of Responsibility)
var recipientValidator = new RecipientValidator();
var contentValidator = new ContentValidator();
recipientValidator.SetNext(contentValidator);

// Template Method
Console.WriteLine("\n--- Template Method: สร้าง Email จาก Template ---");
var orderTemplate = new OrderConfirmationEmailTemplate();
var emailNotification = orderTemplate.BuildNotification(
    "customer@example.com",
    new Dictionary<string, string>
    {
        ["order_number"] = "ORD-2024-001",
        ["customer_name"] = "สมชาย",
        ["total"] = "45,000"
    });

// Validate (Chain of Responsibility)
var (isValid, error) = recipientValidator.Validate(emailNotification);
if (!isValid)
{
    Console.WriteLine($"Validation error: {error}");
}
else
{
    await dispatcher.DispatchAsync(emailNotification);
}

// Slack Notification
Console.WriteLine("\n--- Low Stock Alert (Slack) ---");
var slackTemplate = new LowStockAlertSlackTemplate();
var slackNotification = slackTemplate.BuildNotification(
    "#inventory",
    new Dictionary<string, string>
    {
        ["product_name"] = "iPhone 15 Pro",
        ["stock"] = "3",
        ["product_url"] = "https://shop.example.com/products/iphone15pro"
    });
await dispatcher.DispatchAsync(slackNotification);

// Observer: Strategy ด้วย Sort
Console.WriteLine("\n--- Strategy Pattern: Sort Algorithms ---");
var data = Enumerable.Range(1, 1000).OrderBy(_ => Random.Shared.Next()).ToList();

var sorter = new DataSorter(AdaptiveSortStrategyFactory.GetBestStrategy(data.Count));
var sorted = sorter.Sort(new List<int>(data));
Console.WriteLine($"ผลลัพธ์ (5 ค่าแรก): {string.Join(", ", sorted.Take(5))}");

// Command Pattern: Text Editor
Console.WriteLine("\n--- Command Pattern: Text Document ---");
var doc = new TextDocument();
var history = new CommandHistory();

history.Execute(new InsertTextCommand(doc, 0, "สวัสดี"));
history.Execute(new InsertTextCommand(doc, 6, " โลก"));
Console.WriteLine($"Content: {doc.Content}");

history.Undo();
Console.WriteLine($"หลัง Undo: {doc.Content}");

history.Redo();
Console.WriteLine($"หลัง Redo: {doc.Content}");

// Notification Log
Console.WriteLine("\n--- Notification Log ---");
foreach (var log in dispatcher.GetLog())
{
    Console.WriteLine($"  [{log.SentAt:HH:mm:ss}] {log.Notification.Type}: " +
        $"{log.Notification.Title} → {log.HandlersCount} handlers");
}
```

---

## Exercises

1. **Exercise 1**: Implement `EventBus` ด้วย Observer Pattern รองรับ:
   - Subscribe/Unsubscribe
   - Publish events แบบ async
   - Filter events ด้วย predicate

2. **Exercise 2**: สร้าง `FileConversionPipeline` ด้วย Chain of Responsibility:
   - ImageResizer
   - WatermarkAdder
   - FormatConverter
   - FileCompressor

3. **Exercise 3**: Implement `TextEditor` ด้วย Command Pattern พร้อม:
   - Undo/Redo สูงสุด 50 steps
   - Batch commands (macro)

4. **Exercise 4**: สร้าง `PricingStrategy` system ที่เลือก:
   - `DynamicPricingStrategy` - ปรับราคาตาม demand
   - `SeasonalPricingStrategy` - ลดราคาตามฤดูกาล
   - `LoyaltyPricingStrategy` - ให้ส่วนลดตาม loyalty points

5. **Exercise 5**: สร้าง Report Generator ด้วย Template Method รองรับ PDF, Excel, HTML

---

## สรุป

Behavioral Patterns จัดการการสื่อสารระหว่าง objects:
- **Observer**: แจ้งเตือน observers เมื่อ subject เปลี่ยน
- **Strategy**: เลือก algorithm ได้ตาม context
- **Command**: ห่อ action เป็น object, รองรับ undo/redo
- **Chain of Responsibility**: ส่ง request ผ่าน chain ของ handlers
- **Template Method**: กำหนด skeleton algorithm ใน base class

---

## Part ถัดไป

ใน Part 80 เราจะเรียนรู้เรื่อง **SOLID Principles** ซึ่งเป็นหลักการออกแบบ 5 ข้อที่ทำให้โค้ดมีคุณภาพ

---

*Part 79/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
