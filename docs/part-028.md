# Part 028: Delegates และ Events

## เนื้อหาใน Part นี้
- Delegate คืออะไร และการประกาศ
- Multicast Delegates
- Func\<T\>, Action\<T\>, Predicate\<T\>
- Event Declaration และการใช้งาน
- Event Subscription และ Unsubscription
- EventHandler Pattern
- Thread-safe Events
- โปรแกรมตัวอย่าง: Button Click Event System

---

## 1. Delegate คืออะไร?

Delegate คือ Type ที่แทนการอ้างอิงถึง method ให้ส่ง method เป็น argument ได้

```csharp
// ===== การประกาศ Delegate =====
// delegate keyword กำหนด signature ของ method ที่รองรับ
public delegate int MathOperation(int a, int b);
public delegate string Formatter(string input);
public delegate bool Validator(string value);

// ===== การใช้งาน Delegate =====
int Add(int a, int b) => a + b;
int Subtract(int a, int b) => a - b;
int Multiply(int a, int b) => a * b;

// สร้าง delegate instance
MathOperation op = Add;
Console.WriteLine(op(3, 4));  // 7

op = Subtract;
Console.WriteLine(op(10, 3)); // 7

op = Multiply;
Console.WriteLine(op(3, 4));  // 12

// ===== Delegate เป็น Parameter =====
int ApplyOperation(int x, int y, MathOperation operation)
{
    return operation(x, y);
}

Console.WriteLine(ApplyOperation(5, 3, Add));       // 8
Console.WriteLine(ApplyOperation(5, 3, Subtract));  // 2
Console.WriteLine(ApplyOperation(5, 3, Multiply));  // 15

// ===== Anonymous Method (เก่า) =====
MathOperation squareFirst = delegate(int a, int b) { return a * a + b; };
Console.WriteLine(squareFirst(3, 4)); // 13

// ===== Lambda Expression (ใหม่กว่า) =====
MathOperation add = (a, b) => a + b;
Formatter upper = s => s.ToUpper();
Validator notEmpty = s => !string.IsNullOrEmpty(s);

Console.WriteLine(add(2, 3));     // 5
Console.WriteLine(upper("hello")); // HELLO
Console.WriteLine(notEmpty(""));   // false

// ===== Delegate เป็น Return Value =====
MathOperation GetOperation(string name) => name switch
{
    "add" => (a, b) => a + b,
    "sub" => (a, b) => a - b,
    "mul" => (a, b) => a * b,
    _ => throw new ArgumentException($"Unknown operation: {name}")
};

var addOp = GetOperation("add");
Console.WriteLine(addOp(10, 5)); // 15
```

---

## 2. Multicast Delegates

Delegate สามารถรวมหลาย methods ได้ (Invocation List)

```csharp
public delegate void Logger(string message);

// สร้าง logger
Logger consoleLogger = msg => Console.WriteLine($"[Console] {msg}");
Logger fileLogger = msg => Console.WriteLine($"[File] {msg}"); // simulated
Logger emailLogger = msg => Console.WriteLine($"[Email] {msg}"); // simulated

// Multicast: รวมหลาย delegates ด้วย +=
Logger allLoggers = consoleLogger;
allLoggers += fileLogger;
allLoggers += emailLogger;

// เรียกครั้งเดียว - ทั้ง 3 methods ทำงาน
allLoggers("System started");
// [Console] System started
// [File] System started
// [Email] System started

// ลบ delegate ด้วย -=
allLoggers -= emailLogger;
allLoggers("Now only 2 loggers");
// [Console] Now only 2 loggers
// [File] Now only 2 loggers

// GetInvocationList: ดู methods ที่ registered
Delegate[] list = allLoggers.GetInvocationList();
Console.WriteLine($"Loggers registered: {list.Length}"); // 2

// ===== Multicast กับ Return Value =====
// ระวัง: ถ้า delegate มี return value, ได้แค่ค่าสุดท้าย!
public delegate int Calculator(int n);

Calculator calc = n => n + 1;
calc += n => n * 2;
calc += n => n - 3;

int result = calc(5); // ได้แค่ n - 3 = 2 (method สุดท้าย)
Console.WriteLine(result); // 2

// ถ้าต้องการทุก result - ใช้ GetInvocationList
foreach (Calculator c in calc.GetInvocationList())
    Console.WriteLine(c(5)); // 6, 10, 2
```

---

## 3. Func\<T\>, Action\<T\>, Predicate\<T\>

.NET มี built-in delegate types พร้อมใช้

```csharp
// ===== Func<TResult> =====
// Func ที่มี return value
// Func<T1, T2, ..., TResult> - parameter สุดท้ายคือ return type

Func<int> getNumber = () => 42;
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<string, int, string> repeat = (s, n) => string.Concat(Enumerable.Repeat(s, n));
Func<double, double> sqrt = Math.Sqrt; // Method Group

Console.WriteLine(getNumber());        // 42
Console.WriteLine(square(5));          // 25
Console.WriteLine(add(3, 4));          // 7
Console.WriteLine(repeat("ab", 3));    // ababab
Console.WriteLine(sqrt(16));           // 4

// ===== Action<T> =====
// Action ที่ไม่มี return value (void)

Action greet = () => Console.WriteLine("Hello!");
Action<string> sayHello = name => Console.WriteLine($"Hello, {name}!");
Action<string, int> print = (msg, count) =>
{
    for (int i = 0; i < count; i++)
        Console.WriteLine(msg);
};
Action<int, int> logDivide = (a, b) =>
{
    if (b == 0) Console.WriteLine("Cannot divide by zero");
    else Console.WriteLine($"{a} / {b} = {a / b}");
};

greet();             // Hello!
sayHello("Alice");   // Hello, Alice!
print("repeat", 3);  // repeat x3
logDivide(10, 2);    // 10 / 2 = 5

// ===== Predicate<T> =====
// ย่อจาก Func<T, bool>

Predicate<int> isPositive = n => n > 0;
Predicate<string> isLong = s => s.Length > 5;
Predicate<DateTime> isPast = d => d < DateTime.Now;

Console.WriteLine(isPositive(5));         // true
Console.WriteLine(isPositive(-1));        // false
Console.WriteLine(isLong("Hello"));       // false
Console.WriteLine(isLong("Hello World")); // true

// ใช้งานร่วมกับ List<T>
var numbers = new List<int> { -3, -1, 0, 2, 4, 6 };
int firstPositive = numbers.Find(isPositive);   // 2
List<int> allPositive = numbers.FindAll(isPositive); // [2, 4, 6]
bool hasNegative = numbers.Exists(n => n < 0);  // true

// ===== Higher-Order Functions =====
// Functions ที่รับ/คืน functions อื่น

// Memoize: Cache function results
Func<T, TResult> Memoize<T, TResult>(Func<T, TResult> func) where T : notnull
{
    var cache = new Dictionary<T, TResult>();
    return input =>
    {
        if (cache.TryGetValue(input, out TResult? result))
            return result;
        result = func(input);
        cache[input] = result;
        return result;
    };
}

// Fibonacci with memoization
var fib = Memoize<int, long>(n =>
{
    if (n <= 1) return n;
    return n - 1 + n - 2; // simplified - real version needs recursion
});

// Compose: รวม 2 functions
Func<T, TResult> Compose<T, TMiddle, TResult>(
    Func<T, TMiddle> first,
    Func<TMiddle, TResult> second)
{
    return input => second(first(input));
}

Func<string, string> trimAndUpper = Compose<string, string, string>(
    s => s.Trim(),
    s => s.ToUpper()
);
Console.WriteLine(trimAndUpper("  hello  ")); // HELLO

// Curry
Func<int, Func<int, int>> curriedAdd = a => b => a + b;
Func<int, int> add5 = curriedAdd(5);
Console.WriteLine(add5(3));  // 8
Console.WriteLine(add5(10)); // 15
```

---

## 4. Event Declaration

```csharp
// ===== Event Pattern =====
// Event ใช้ delegate แต่จำกัด: ภายนอกเรียกได้แค่ += และ -=

// ===== Simple Event =====
public class Timer
{
    // ประกาศ event ด้วย event keyword
    public event Action? Tick;
    public event Action<int>? TickWithCount;

    private int _count = 0;

    public void Start(int times)
    {
        for (int i = 0; i < times; i++)
        {
            _count++;
            // เรียก event (ใช้ ?. เพื่อความปลอดภัย)
            Tick?.Invoke();
            TickWithCount?.Invoke(_count);
            Thread.Sleep(100);
        }
    }
}

var timer = new Timer();

// Subscribe
timer.Tick += () => Console.WriteLine("Tick!");
timer.TickWithCount += count => Console.WriteLine($"Tick #{count}");

timer.Start(3);
// Tick!
// Tick #1
// Tick!
// Tick #2
// ...

// ===== EventHandler Pattern =====
// .NET standard: EventHandler<TEventArgs>
public class EventArgs<T> : EventArgs
{
    public T Data { get; }
    public DateTime Timestamp { get; }

    public EventArgs(T data)
    {
        Data = data;
        Timestamp = DateTime.Now;
    }
}

public class StockMarket
{
    // Standard EventHandler pattern
    public event EventHandler<EventArgs<decimal>>? PriceChanged;
    public event EventHandler<EventArgs<string>>? AlertTriggered;

    private decimal _price;
    private decimal _threshold = 1000m;

    public string Symbol { get; }

    public StockMarket(string symbol, decimal initialPrice)
    {
        Symbol = symbol;
        _price = initialPrice;
    }

    public decimal Price
    {
        get => _price;
        set
        {
            decimal oldPrice = _price;
            _price = value;

            // Raise event
            if (oldPrice != value)
                OnPriceChanged(value);

            if (value > _threshold)
                OnAlertTriggered($"{Symbol} exceeded {_threshold:C}");
        }
    }

    protected virtual void OnPriceChanged(decimal newPrice)
    {
        PriceChanged?.Invoke(this, new EventArgs<decimal>(newPrice));
    }

    protected virtual void OnAlertTriggered(string message)
    {
        AlertTriggered?.Invoke(this, new EventArgs<string>(message));
    }
}

// ใช้งาน
var stock = new StockMarket("AAPL", 900m);

stock.PriceChanged += (sender, e) =>
{
    var market = (StockMarket)sender!;
    Console.WriteLine($"{market.Symbol}: Price changed to {e.Data:C} at {e.Timestamp:HH:mm:ss}");
};

stock.AlertTriggered += (sender, e) =>
{
    Console.WriteLine($"ALERT: {e.Data}");
};

stock.Price = 950m;  // Price changed to ฿950.00
stock.Price = 1050m; // Price changed + Alert!
```

---

## 5. Event Subscription และ Unsubscription

```csharp
// ===== การ Subscribe/Unsubscribe =====
public class Button
{
    public event EventHandler? Clicked;
    public event EventHandler? HoverEnter;
    public event EventHandler? HoverLeave;
    public string Label { get; set; } = "";

    public void SimulateClick() => Clicked?.Invoke(this, EventArgs.Empty);
    public void SimulateHover(bool enter)
    {
        if (enter) HoverEnter?.Invoke(this, EventArgs.Empty);
        else HoverLeave?.Invoke(this, EventArgs.Empty);
    }
}

// Handler methods
void OnButtonClicked(object? sender, EventArgs e)
{
    var btn = sender as Button;
    Console.WriteLine($"Button '{btn?.Label}' clicked!");
}

EventHandler hoverHandler = (s, e) =>
{
    Console.WriteLine("Hovering...");
};

var btn = new Button { Label = "Submit" };

// Subscribe
btn.Clicked += OnButtonClicked;
btn.HoverEnter += hoverHandler;

btn.SimulateClick();   // Button 'Submit' clicked!
btn.SimulateHover(true); // Hovering...

// Unsubscribe
btn.Clicked -= OnButtonClicked;
btn.HoverEnter -= hoverHandler;

btn.SimulateClick(); // ไม่มีอะไรเกิดขึ้น

// ===== ระวัง Memory Leak! =====
// ถ้า subscribe แล้วไม่ unsubscribe - object จะไม่ถูก GC
public class EventSource
{
    public event Action? SomethingHappened;
    // ...
}

public class EventSubscriber : IDisposable
{
    private readonly EventSource _source;
    private bool _disposed = false;

    public EventSubscriber(EventSource source)
    {
        _source = source;
        _source.SomethingHappened += HandleEvent; // Subscribe
    }

    private void HandleEvent()
    {
        Console.WriteLine("Handled!");
    }

    public void Dispose()
    {
        if (!_disposed)
        {
            _source.SomethingHappened -= HandleEvent; // Unsubscribe!
            _disposed = true;
        }
    }
}
```

---

## 6. Thread-Safe Events

```csharp
// ===== Thread Safety =====
// การ raise event แบบ thread-safe

public class SafeEventPublisher
{
    // วิธีที่ 1: Copy-and-invoke (แนะนำ)
    private event EventHandler? _dataReceived;
    public event EventHandler? DataReceived
    {
        add => _dataReceived += value;
        remove => _dataReceived -= value;
    }

    protected virtual void OnDataReceived(EventArgs e)
    {
        // Copy local - thread-safe เพราะ delegate เป็น immutable
        EventHandler? handler = _dataReceived;
        handler?.Invoke(this, e);
    }

    // วิธีที่ 2: Interlocked
    private EventHandler? _statusChanged;
    public event EventHandler? StatusChanged
    {
        add => Interlocked.CompareExchange(ref _statusChanged, value, null);
        remove
        {
            // ซับซ้อน - ส่วนใหญ่ใช้วิธีที่ 1 แทน
        }
    }

    // วิธีที่ 3: Custom event accessor ด้วย lock
    private readonly object _lockObj = new();
    private EventHandler? _customEvent;

    public event EventHandler? CustomEvent
    {
        add { lock (_lockObj) _customEvent += value; }
        remove { lock (_lockObj) _customEvent -= value; }
    }

    protected void RaiseCustomEvent()
    {
        EventHandler? handler;
        lock (_lockObj) handler = _customEvent;
        handler?.Invoke(this, EventArgs.Empty);
    }
}

// ===== Event Aggregator Pattern =====
// สำหรับ decoupled event communication

public interface IEvent { }

public class EventAggregator
{
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    private readonly object _lock = new();

    public void Subscribe<T>(Action<T> handler) where T : IEvent
    {
        lock (_lock)
        {
            var type = typeof(T);
            if (!_handlers.TryGetValue(type, out var list))
            {
                list = new List<Delegate>();
                _handlers[type] = list;
            }
            list.Add(handler);
        }
    }

    public void Unsubscribe<T>(Action<T> handler) where T : IEvent
    {
        lock (_lock)
        {
            if (_handlers.TryGetValue(typeof(T), out var list))
                list.Remove(handler);
        }
    }

    public void Publish<T>(T @event) where T : IEvent
    {
        List<Delegate>? handlers;
        lock (_lock)
        {
            if (!_handlers.TryGetValue(typeof(T), out handlers))
                return;
            handlers = new List<Delegate>(handlers); // copy for thread safety
        }

        foreach (var handler in handlers)
            ((Action<T>)handler)(@event);
    }
}

// Event types
public record UserLoggedIn(string Username, DateTime Time) : IEvent;
public record OrderPlaced(int OrderId, decimal Amount) : IEvent;
public record PaymentReceived(int OrderId, string Method) : IEvent;

// ใช้งาน Event Aggregator
var aggregator = new EventAggregator();

// Subscribers
aggregator.Subscribe<UserLoggedIn>(e =>
    Console.WriteLine($"Security: {e.Username} logged in at {e.Time:HH:mm:ss}"));

aggregator.Subscribe<UserLoggedIn>(e =>
    Console.WriteLine($"Analytics: User {e.Username} session started"));

aggregator.Subscribe<OrderPlaced>(e =>
    Console.WriteLine($"Inventory: Processing order #{e.OrderId}"));

aggregator.Subscribe<OrderPlaced>(e =>
    Console.WriteLine($"Billing: Order #{e.OrderId} for {e.Amount:C}"));

aggregator.Subscribe<PaymentReceived>(e =>
    Console.WriteLine($"Finance: Payment for order #{e.OrderId} via {e.Method}"));

// Publishers
aggregator.Publish(new UserLoggedIn("Alice", DateTime.Now));
aggregator.Publish(new OrderPlaced(1001, 2500m));
aggregator.Publish(new PaymentReceived(1001, "Credit Card"));
```

---

## 7. โปรแกรมตัวอย่าง: Button Click Event System

```csharp
using System;
using System.Collections.Generic;

namespace UIEventSystem
{
    // ===== Event Args =====
    public class MouseEventArgs : EventArgs
    {
        public int X { get; }
        public int Y { get; }
        public MouseButton Button { get; }
        public DateTime Timestamp { get; }

        public MouseEventArgs(int x, int y, MouseButton button = MouseButton.Left)
        {
            X = x; Y = y;
            Button = button;
            Timestamp = DateTime.Now;
        }
    }

    public class KeyEventArgs : EventArgs
    {
        public string Key { get; }
        public bool IsCtrlDown { get; }
        public bool IsShiftDown { get; }

        public KeyEventArgs(string key, bool ctrl = false, bool shift = false)
        {
            Key = key;
            IsCtrlDown = ctrl;
            IsShiftDown = shift;
        }
    }

    public class StateChangedEventArgs<T> : EventArgs
    {
        public T OldValue { get; }
        public T NewValue { get; }

        public StateChangedEventArgs(T oldValue, T newValue)
        {
            OldValue = oldValue;
            NewValue = newValue;
        }
    }

    public enum MouseButton { Left, Right, Middle }

    // ===== Base Control =====
    public abstract class Control
    {
        public string Name { get; set; } = "";
        public bool IsEnabled { get; protected set; } = true;
        public bool IsVisible { get; protected set; } = true;

        // Events
        public event EventHandler<MouseEventArgs>? MouseClick;
        public event EventHandler<MouseEventArgs>? MouseDoubleClick;
        public event EventHandler<MouseEventArgs>? MouseEnter;
        public event EventHandler<MouseEventArgs>? MouseLeave;
        public event EventHandler<KeyEventArgs>? KeyPress;
        public event EventHandler? GotFocus;
        public event EventHandler? LostFocus;
        public event EventHandler<StateChangedEventArgs<bool>>? EnabledChanged;

        public void Enable()
        {
            bool old = IsEnabled;
            IsEnabled = true;
            if (old != IsEnabled)
                EnabledChanged?.Invoke(this, new StateChangedEventArgs<bool>(old, IsEnabled));
        }

        public void Disable()
        {
            bool old = IsEnabled;
            IsEnabled = false;
            if (old != IsEnabled)
                EnabledChanged?.Invoke(this, new StateChangedEventArgs<bool>(old, IsEnabled));
        }

        // Methods to simulate events (for testing)
        public virtual void SimulateClick(int x = 0, int y = 0, MouseButton btn = MouseButton.Left)
        {
            if (!IsEnabled) return;
            MouseClick?.Invoke(this, new MouseEventArgs(x, y, btn));
        }

        public virtual void SimulateDoubleClick(int x = 0, int y = 0)
        {
            if (!IsEnabled) return;
            MouseDoubleClick?.Invoke(this, new MouseEventArgs(x, y));
        }

        public virtual void SimulateKeyPress(string key, bool ctrl = false, bool shift = false)
        {
            if (!IsEnabled) return;
            KeyPress?.Invoke(this, new KeyEventArgs(key, ctrl, shift));
        }

        public void SimulateHover(bool enter)
        {
            if (enter) MouseEnter?.Invoke(this, new MouseEventArgs(0, 0));
            else MouseLeave?.Invoke(this, new MouseEventArgs(0, 0));
        }

        public abstract string GetDescription();
    }

    // ===== Button =====
    public class Button : Control
    {
        public string Text { get; set; } = "";
        public string Style { get; set; } = "Default";

        private int _clickCount = 0;

        public Button(string name, string text)
        {
            Name = name;
            Text = text;
            // Default hover visual effect
            MouseEnter += (_, _) => Console.WriteLine($"  [{Name}] Hover on");
            MouseLeave += (_, _) => Console.WriteLine($"  [{Name}] Hover off");
        }

        public override void SimulateClick(int x = 0, int y = 0, MouseButton btn = MouseButton.Left)
        {
            if (!IsEnabled)
            {
                Console.WriteLine($"  [{Name}] Button is disabled");
                return;
            }
            _clickCount++;
            Console.WriteLine($"  [{Name}] Clicked ({_clickCount} times) at ({x},{y}) [{btn}]");
            base.SimulateClick(x, y, btn);
        }

        public int ClickCount => _clickCount;
        public void ResetCount() => _clickCount = 0;

        public override string GetDescription() =>
            $"Button[{Name}]: '{Text}' ({Style}) - Clicks: {_clickCount}";
    }

    // ===== TextBox =====
    public class TextBox : Control
    {
        private string _text = "";

        public event EventHandler<StateChangedEventArgs<string>>? TextChanged;

        public string Text
        {
            get => _text;
            set
            {
                string old = _text;
                _text = value;
                if (old != value)
                    TextChanged?.Invoke(this, new StateChangedEventArgs<string>(old, value));
            }
        }

        public TextBox(string name)
        {
            Name = name;
            KeyPress += (_, e) =>
            {
                if (e.Key.Length == 1)
                    Text += e.Key;
                else if (e.Key == "Backspace" && Text.Length > 0)
                    Text = Text[..^1];
            };
        }

        public override string GetDescription() =>
            $"TextBox[{Name}]: Value='{Text}'";
    }

    // ===== CheckBox =====
    public class CheckBox : Control
    {
        private bool _isChecked = false;

        public event EventHandler<StateChangedEventArgs<bool>>? CheckedChanged;

        public bool IsChecked
        {
            get => _isChecked;
            set
            {
                bool old = _isChecked;
                _isChecked = value;
                if (old != value)
                    CheckedChanged?.Invoke(this, new StateChangedEventArgs<bool>(old, value));
            }
        }

        public CheckBox(string name, bool initialState = false)
        {
            Name = name;
            _isChecked = initialState;
            MouseClick += (_, _) => IsChecked = !IsChecked;
        }

        public override string GetDescription() =>
            $"CheckBox[{Name}]: Checked={IsChecked}";
    }

    // ===== Form =====
    public class Form
    {
        private readonly List<Control> _controls = new();
        private readonly List<string> _eventLog = new();
        public string Title { get; set; } = "";

        public event EventHandler? FormLoaded;
        public event EventHandler? FormClosing;
        public event EventHandler<string>? ValidationFailed;

        public void Add(Control control) => _controls.Add(control);

        public T? GetControl<T>(string name) where T : Control =>
            _controls.OfType<T>().FirstOrDefault(c => c.Name == name);

        public void Load()
        {
            Console.WriteLine($"=== Form '{Title}' Loaded ===");
            Log("Form loaded");
            FormLoaded?.Invoke(this, EventArgs.Empty);
        }

        public void Close()
        {
            Log("Form closing");
            FormClosing?.Invoke(this, EventArgs.Empty);
            Console.WriteLine($"=== Form '{Title}' Closed ===");
        }

        public void Log(string message)
        {
            string entry = $"[{DateTime.Now:HH:mm:ss.fff}] {message}";
            _eventLog.Add(entry);
        }

        public void PrintLog()
        {
            Console.WriteLine($"\n--- Event Log for '{Title}' ---");
            foreach (string entry in _eventLog)
                Console.WriteLine($"  {entry}");
        }

        public void PrintStatus()
        {
            Console.WriteLine($"\n--- Control Status ---");
            foreach (Control ctrl in _controls)
                Console.WriteLine($"  {ctrl.GetDescription()}");
        }
    }

    // ===== Login Form Demo =====
    class Program
    {
        static void Main()
        {
            // สร้าง form
            var loginForm = new Form { Title = "Login Form" };

            // สร้าง controls
            var usernameBox = new TextBox("username");
            var passwordBox = new TextBox("password");
            var rememberMe = new CheckBox("rememberMe");
            var loginBtn = new Button("loginBtn", "Login");
            var cancelBtn = new Button("cancelBtn", "Cancel");
            var forgotBtn = new Button("forgotBtn", "Forgot Password");

            // เพิ่ม controls
            loginForm.Add(usernameBox);
            loginForm.Add(passwordBox);
            loginForm.Add(rememberMe);
            loginForm.Add(loginBtn);
            loginForm.Add(cancelBtn);
            loginForm.Add(forgotBtn);

            // ===== Wire up events =====

            // TextBox events
            usernameBox.TextChanged += (_, e) =>
                loginForm.Log($"Username changed: '{e.OldValue}' -> '{e.NewValue}'");

            // CheckBox events
            rememberMe.CheckedChanged += (_, e) =>
            {
                string state = e.NewValue ? "ON" : "OFF";
                Console.WriteLine($"Remember Me: {state}");
                loginForm.Log($"Remember Me toggled: {state}");
            };

            // Login button
            loginBtn.MouseClick += (_, e) =>
            {
                loginForm.Log($"Login attempted at ({e.X},{e.Y})");

                string username = usernameBox.Text;
                string password = passwordBox.Text;

                if (string.IsNullOrEmpty(username) || string.IsNullOrEmpty(password))
                {
                    Console.WriteLine("ERROR: Username and password required!");
                    loginForm.ValidationFailed?.Invoke(loginForm,
                        "Username and password required");
                    return;
                }

                // Simulate authentication
                bool success = username == "admin" && password == "1234";
                if (success)
                {
                    Console.WriteLine($"Welcome, {username}! (Remember: {rememberMe.IsChecked})");
                    loginForm.Log($"Login successful: {username}");
                }
                else
                {
                    Console.WriteLine("Invalid credentials!");
                    loginForm.Log("Login failed: invalid credentials");
                }
            };

            // Cancel button
            cancelBtn.MouseClick += (_, _) =>
            {
                loginForm.Log("User cancelled");
                loginForm.Close();
            };

            // Forgot password
            forgotBtn.MouseClick += (_, _) =>
            {
                Console.WriteLine("Opening password reset dialog...");
                loginForm.Log("Forgot password requested");
            };

            // Form events
            loginForm.FormLoaded += (_, _) =>
                Console.WriteLine("Initializing form...");

            loginForm.FormClosing += (_, _) =>
                Console.WriteLine("Saving state before close...");

            loginForm.ValidationFailed += (_, message) =>
                Console.WriteLine($"Validation: {message}");

            // Keyboard shortcuts
            loginBtn.KeyPress += (_, e) =>
            {
                if (e.Key == "Enter" || (e.Key == "L" && e.IsCtrlDown))
                    loginBtn.SimulateClick();
            };

            // ===== Simulate user interaction =====
            Console.WriteLine("\n=== Simulating User Interaction ===\n");

            loginForm.Load();

            Console.WriteLine("\n1. User types username:");
            foreach (char c in "admin")
                usernameBox.SimulateKeyPress(c.ToString());
            Console.WriteLine($"   Username: '{usernameBox.Text}'");

            Console.WriteLine("\n2. User types password:");
            foreach (char c in "1234")
                passwordBox.SimulateKeyPress(c.ToString());
            Console.WriteLine($"   Password: '{new string('*', passwordBox.Text.Length)}'");

            Console.WriteLine("\n3. Hover on Login button:");
            loginBtn.SimulateHover(true);
            loginBtn.SimulateHover(false);

            Console.WriteLine("\n4. Toggle Remember Me:");
            rememberMe.SimulateClick();
            rememberMe.SimulateClick();
            rememberMe.SimulateClick(); // checked

            Console.WriteLine("\n5. Click Login with wrong password:");
            passwordBox.Text = "wrong";
            loginBtn.SimulateClick(50, 30);

            Console.WriteLine("\n6. Click Login with correct password:");
            passwordBox.Text = "1234";
            loginBtn.SimulateClick(50, 30);

            Console.WriteLine("\n7. Disable login button:");
            loginBtn.Disable();
            loginBtn.SimulateClick(); // disabled

            Console.WriteLine("\n8. Try empty username:");
            usernameBox.Text = "";
            loginBtn.Enable();
            loginBtn.SimulateClick();

            loginForm.PrintStatus();
            loginForm.PrintLog();
        }
    }
}
```

---

## Exercises

### Exercise 1: Observable List
สร้าง `ObservableList<T>` ที่:
- เหมือน `List<T>` แต่มี events
- `ItemAdded`, `ItemRemoved`, `ListCleared`
- `CollectionChanged` รวมทุก event

```csharp
public class ObservableList<T> : IList<T>
{
    public event EventHandler<ItemEventArgs<T>>? ItemAdded;
    public event EventHandler<ItemEventArgs<T>>? ItemRemoved;
    public event EventHandler? Cleared;
    // TODO: implement
}
```

### Exercise 2: Progress Reporting
สร้าง file downloader ที่รายงาน progress:
- `DownloadStarted`, `ProgressChanged`, `DownloadCompleted`
- แสดง progress bar ใน console
- Support cancellation

### Exercise 3: Command Pattern
ใช้ Delegates สร้าง Command system:
- `ICommand` interface ด้วย Execute/Undo
- `CommandManager` ที่มี Undo/Redo
- Events สำหรับ command execution

---

## สรุป

- ✅ Delegate คือ type-safe function pointer ที่ส่งเป็น argument ได้
- ✅ Multicast delegate รวมหลาย methods ด้วย `+=`
- ✅ `Func<T>` สำหรับ functions ที่มี return value
- ✅ `Action<T>` สำหรับ procedures ที่ไม่มี return value
- ✅ `Predicate<T>` ย่อจาก `Func<T, bool>`
- ✅ Event ใช้ delegate แต่ภายนอก subscribe/unsubscribe เท่านั้น
- ✅ `event EventHandler<TEventArgs>` คือ standard .NET pattern
- ✅ ใช้ `?.Invoke()` เพื่อ thread-safe null check
- ✅ Always unsubscribe เมื่อ subscriber destroyed - ป้องกัน memory leak
- ✅ Event Aggregator pattern ช่วย decouple publishers/subscribers

## Part ถัดไป
**Part 029** จะพูดถึง Lambda Expressions: syntax, closures, LINQ usage, Expression trees เบื้องต้น และ Anonymous methods

---
*Part 028/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
