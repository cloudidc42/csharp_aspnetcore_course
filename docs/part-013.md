# Part 013: Properties และ Fields

## เนื้อหาใน Part นี้
- Auto-implemented properties
- Getters และ Setters
- Read-only properties
- Init-only properties (C# 9+)
- Computed/Derived properties
- Property validation
- โปรแกรมตัวอย่าง: Product class

---

## 1. ทำไมต้องใช้ Properties แทน Fields?

```csharp
// ❌ วิธีไม่ดี: Public fields
public class BadPerson
{
    public string Name;   // ใครก็เข้าถึงและแก้ไขได้โดยตรง
    public int Age;       // ไม่มี validation
}

var bad = new BadPerson();
bad.Age = -999; // ค่าไม่สมเหตุสมผล แต่ไม่มีอะไรหยุด

// ✅ วิธีดี: Properties พร้อม validation
public class GoodPerson
{
    private string _name = "";
    private int _age;

    public string Name
    {
        get => _name;
        set => _name = string.IsNullOrWhiteSpace(value)
            ? throw new ArgumentException("Name cannot be empty")
            : value.Trim();
    }

    public int Age
    {
        get => _age;
        set => _age = value >= 0 && value <= 150
            ? value
            : throw new ArgumentOutOfRangeException(nameof(value), "Age must be 0-150");
    }
}

var good = new GoodPerson();
good.Name = "สมชาย";
try { good.Age = -999; }
catch (ArgumentOutOfRangeException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}
```

**เหตุผลที่ใช้ Properties:**
1. **Encapsulation** - ซ่อน implementation details
2. **Validation** - ตรวจสอบค่าก่อนกำหนด
3. **Computed values** - คำนวณค่าจาก fields อื่น
4. **Debugging** - ใส่ breakpoint ใน getter/setter ได้
5. **Change notification** - แจ้งเมื่อค่าเปลี่ยน
6. **Lazy loading** - โหลดข้อมูลเมื่อจำเป็น

---

## 2. Auto-Implemented Properties

```csharp
public class Product
{
    // Auto-implemented property - compiler สร้าง backing field ให้
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }

    // Auto-property พร้อม default value
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.Now;
    public List<string> Tags { get; set; } = new List<string>();

    // Auto-property ที่ set ได้เฉพาะภายใน class
    public Guid Id { get; private set; } = Guid.NewGuid();

    // Auto-property ที่ set ได้เฉพาะภายใน class และ derived
    public string Category { get; protected set; } = "Uncategorized";
}

// Object initializer ใช้ได้กับ auto-properties
var product = new Product
{
    Name = "Laptop",
    Price = 35000m,
    Stock = 10,
    Tags = new List<string> { "Electronics", "Computer" }
};

Console.WriteLine($"{product.Id}: {product.Name} - {product.Price:C}");
Console.WriteLine($"Active: {product.IsActive}, Created: {product.CreatedAt:dd/MM/yyyy}");
```

---

## 3. Full Properties (Manual Getter/Setter)

```csharp
public class Employee
{
    // Backing fields (convention: underscore prefix)
    private string _firstName = "";
    private string _lastName = "";
    private decimal _salary;
    private int _yearsOfExperience;
    private string _email = "";

    // Full property พร้อม validation
    public string FirstName
    {
        get => _firstName;
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("First name cannot be empty.");
            _firstName = value.Trim();
        }
    }

    public string LastName
    {
        get => _lastName;
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Last name cannot be empty.");
            _lastName = value.Trim();
        }
    }

    public decimal Salary
    {
        get => _salary;
        set
        {
            if (value < 0)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Salary cannot be negative.");
            if (value > 10_000_000)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Salary exceeds maximum limit.");
            _salary = value;
        }
    }

    public int YearsOfExperience
    {
        get => _yearsOfExperience;
        set
        {
            if (value < 0 || value > 60)
                throw new ArgumentOutOfRangeException(nameof(value),
                    "Years of experience must be between 0 and 60.");
            _yearsOfExperience = value;
        }
    }

    public string Email
    {
        get => _email;
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Email cannot be empty.");
            // Simple email validation
            if (!value.Contains('@') || !value.Contains('.'))
                throw new ArgumentException("Invalid email format.");
            _email = value.ToLower().Trim();
        }
    }

    // Computed property
    public string FullName => $"{FirstName} {LastName}";

    public string SalaryGrade
    {
        get
        {
            return _salary switch
            {
                < 20000 => "C",
                < 40000 => "B",
                < 80000 => "A",
                _ => "S"
            };
        }
    }
}

var emp = new Employee();
emp.FirstName = "สมชาย";
emp.LastName = "ใจดี";
emp.Salary = 45000;
emp.Email = "somchai@company.com";

Console.WriteLine($"{emp.FullName} Grade {emp.SalaryGrade}");
```

---

## 4. Read-Only Properties

```csharp
public class Circle
{
    // Read-only auto-property: กำหนดได้ใน constructor เท่านั้น
    public double Radius { get; }

    // Read-only ที่ computed
    public double Diameter => Radius * 2;
    public double Area => Math.PI * Radius * Radius;
    public double Circumference => 2 * Math.PI * Radius;

    public Circle(double radius)
    {
        if (radius <= 0)
            throw new ArgumentOutOfRangeException(nameof(radius),
                "Radius must be positive.");
        Radius = radius;
    }

    public override string ToString()
        => $"Circle(r={Radius:F2}, area={Area:F2}, circumference={Circumference:F2})";
}

var c = new Circle(5);
Console.WriteLine(c);
// c.Radius = 10; // Error! Read-only

// Readonly field vs readonly property
public class Config
{
    // readonly field - กำหนดได้ใน constructor หรือ field initializer
    private readonly string _connectionString;

    // Readonly property สำหรับ access จากภายนอก
    public string ConnectionString => _connectionString;

    public Config(string connStr)
    {
        _connectionString = connStr;
    }
}
```

---

## 5. Init-Only Properties (C# 9+)

Init-only properties กำหนดค่าได้เฉพาะตอน object initialization (ใน constructor หรือ object initializer)

```csharp
public class OrderItem
{
    // init: กำหนดได้เฉพาะตอน initialization
    public int Id { get; init; }
    public string ProductName { get; init; } = "";
    public decimal UnitPrice { get; init; }
    public int Quantity { get; init; }

    // set ปกติ - เปลี่ยนได้ภายหลัง
    public string Notes { get; set; } = "";

    // Computed readonly
    public decimal TotalPrice => UnitPrice * Quantity;

    // Validation ใน constructor
    public OrderItem()
    {
        Id = Random.Shared.Next(1000, 9999);
    }
}

// Object initializer ใช้ init ได้
var item = new OrderItem
{
    ProductName = "Laptop",
    UnitPrice = 35000m,
    Quantity = 2,
    Notes = "ต้องการสีดำ"
};

Console.WriteLine($"{item.ProductName}: {item.Quantity} x {item.UnitPrice:C} = {item.TotalPrice:C}");

// item.ProductName = "Desktop"; // Error! init-only ไม่สามารถแก้ไขได้หลัง init

// With expression - copy และเปลี่ยนบาง property (ใช้กับ record)
public record Product2(int Id, string Name, decimal Price, int Stock);

var p1 = new Product2(1, "Laptop", 35000, 10);
var p2 = p1 with { Price = 32000, Stock = 8 }; // Copy และเปลี่ยน Price และ Stock
var p3 = p1 with { Name = "Gaming Laptop" };

Console.WriteLine(p1); // Product2 { Id = 1, Name = Laptop, Price = 35000, Stock = 10 }
Console.WriteLine(p2); // Product2 { Id = 1, Name = Laptop, Price = 32000, Stock = 8 }
Console.WriteLine(p3); // Product2 { Id = 1, Name = Gaming Laptop, Price = 35000, Stock = 10 }
```

---

## 6. Computed Properties ขั้นสูง

```csharp
public class ShoppingCart
{
    private readonly List<CartItem> _items = new();

    // Read-only computed properties
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    public int ItemCount => _items.Sum(i => i.Quantity);
    public int UniqueItemCount => _items.Count;
    public bool IsEmpty => _items.Count == 0;

    // ราคาก่อนลด
    public decimal SubTotal => _items.Sum(i => i.TotalPrice);

    // ราคาส่วนลด (คำนวณจาก SubTotal)
    public decimal DiscountAmount
    {
        get
        {
            if (SubTotal >= 5000) return SubTotal * 0.1m;   // ลด 10%
            if (SubTotal >= 2000) return SubTotal * 0.05m;  // ลด 5%
            return 0;
        }
    }

    // ค่าจัดส่ง (ขึ้นกับเงื่อนไข)
    public decimal ShippingFee
    {
        get
        {
            if (IsEmpty) return 0;
            if (SubTotal >= 1000) return 0; // ส่งฟรีเมื่อซื้อ >= 1000
            return 50m;
        }
    }

    // ราคารวม
    public decimal Total => SubTotal - DiscountAmount + ShippingFee;

    // Discount tier description
    public string DiscountTier
    {
        get
        {
            if (DiscountAmount == 0) return "ไม่มีส่วนลด";
            decimal rate = DiscountAmount / SubTotal;
            return $"ลด {rate:P0} ({DiscountAmount:N2} บาท)";
        }
    }

    public void AddItem(CartItem item) => _items.Add(item);
    public void RemoveItem(int productId)
        => _items.RemoveAll(i => i.ProductId == productId);
}

public class CartItem
{
    public int ProductId { get; init; }
    public string ProductName { get; init; } = "";
    public decimal UnitPrice { get; init; }
    public int Quantity { get; set; }
    public decimal TotalPrice => UnitPrice * Quantity;

    public override string ToString()
        => $"{ProductName}: {Quantity} x {UnitPrice:N2} = {TotalPrice:N2}";
}

// การใช้งาน
var cart = new ShoppingCart();
cart.AddItem(new CartItem { ProductId = 1, ProductName = "Laptop", UnitPrice = 25000, Quantity = 1 });
cart.AddItem(new CartItem { ProductId = 2, ProductName = "Mouse", UnitPrice = 500, Quantity = 2 });
cart.AddItem(new CartItem { ProductId = 3, ProductName = "Keyboard", UnitPrice = 800, Quantity = 1 });

Console.WriteLine($"รายการ: {cart.UniqueItemCount} รายการ ({cart.ItemCount} ชิ้น)");
Console.WriteLine($"ราคารวม: {cart.SubTotal:N2} บาท");
Console.WriteLine($"ส่วนลด: {cart.DiscountTier}");
Console.WriteLine($"ค่าส่ง: {cart.ShippingFee:N2} บาท");
Console.WriteLine($"รวมทั้งหมด: {cart.Total:N2} บาท");
```

---

## 7. Property Validation Patterns

```csharp
// Pattern 1: Guard clauses ใน setter
public class DateRange
{
    private DateTime _startDate;
    private DateTime _endDate;

    public DateTime StartDate
    {
        get => _startDate;
        set
        {
            if (value > _endDate && _endDate != default)
                throw new ArgumentException("Start date cannot be after end date.");
            _startDate = value;
        }
    }

    public DateTime EndDate
    {
        get => _endDate;
        set
        {
            if (value < _startDate)
                throw new ArgumentException("End date cannot be before start date.");
            _endDate = value;
        }
    }

    public TimeSpan Duration => _endDate - _startDate;
    public int DurationInDays => (int)Duration.TotalDays;

    public DateRange(DateTime start, DateTime end)
    {
        if (end < start)
            throw new ArgumentException("End date must be after start date.");
        _startDate = start;
        _endDate = end;
    }
}

// Pattern 2: INotifyPropertyChanged สำหรับ data binding
using System.ComponentModel;

public class ObservableProduct : INotifyPropertyChanged
{
    private string _name = "";
    private decimal _price;
    private int _stock;

    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged(string propertyName)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }

    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged(nameof(Name));
            }
        }
    }

    public decimal Price
    {
        get => _price;
        set
        {
            if (_price != value)
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException(nameof(value));
                _price = value;
                OnPropertyChanged(nameof(Price));
            }
        }
    }

    public int Stock
    {
        get => _stock;
        set
        {
            if (_stock != value)
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException(nameof(value));
                _stock = value;
                OnPropertyChanged(nameof(Stock));
                // แจ้ง computed property ด้วย
                OnPropertyChanged(nameof(IsAvailable));
                OnPropertyChanged(nameof(StockStatus));
            }
        }
    }

    public bool IsAvailable => _stock > 0;

    public string StockStatus => _stock switch
    {
        0 => "หมดสต็อก",
        <= 5 => $"เหลือน้อย ({_stock})",
        <= 20 => $"มีสต็อก ({_stock})",
        _ => $"มีสต็อกมาก ({_stock})"
    };
}

// Pattern 3: Lazy Loading
public class HeavyDataClass
{
    private List<string>? _expensiveData;

    // โหลดเมื่อมีการเข้าถึงครั้งแรก
    public List<string> ExpensiveData
    {
        get
        {
            if (_expensiveData == null)
            {
                Console.WriteLine("Loading expensive data...");
                _expensiveData = LoadDataFromDatabase();
            }
            return _expensiveData;
        }
    }

    // Thread-safe lazy loading
    private Lazy<List<string>> _threadSafeData = new Lazy<List<string>>(
        () => LoadDataFromDatabaseStatic(),
        LazyThreadSafetyMode.ExecutionAndPublication);

    public List<string> ThreadSafeData => _threadSafeData.Value;

    private List<string> LoadDataFromDatabase()
    {
        System.Threading.Thread.Sleep(100); // จำลองการโหลด
        return new List<string> { "Data1", "Data2", "Data3" };
    }

    private static List<string> LoadDataFromDatabaseStatic()
    {
        return new List<string> { "Data1", "Data2", "Data3" };
    }
}
```

---

## 8. Property Attributes

```csharp
using System.ComponentModel.DataAnnotations;
using System.Text.Json.Serialization;

public class UserProfile
{
    // Data Annotations สำหรับ validation
    [Required(ErrorMessage = "กรุณากรอกชื่อผู้ใช้")]
    [StringLength(50, MinimumLength = 3,
        ErrorMessage = "ชื่อผู้ใช้ต้องมีความยาว 3-50 ตัวอักษร")]
    public string Username { get; set; } = "";

    [Required(ErrorMessage = "กรุณากรอก Email")]
    [EmailAddress(ErrorMessage = "รูปแบบ Email ไม่ถูกต้อง")]
    public string Email { get; set; } = "";

    [Range(13, 120, ErrorMessage = "อายุต้องอยู่ระหว่าง 13-120 ปี")]
    public int Age { get; set; }

    [Phone(ErrorMessage = "รูปแบบเบอร์โทรไม่ถูกต้อง")]
    public string? PhoneNumber { get; set; }

    [MaxLength(500, ErrorMessage = "Bio ต้องไม่เกิน 500 ตัวอักษร")]
    public string? Bio { get; set; }

    // JSON attributes
    [JsonPropertyName("user_id")]
    public int UserId { get; set; }

    [JsonIgnore]
    public string PasswordHash { get; set; } = "";

    [JsonPropertyName("display_name")]
    public string DisplayName => $"{Username} <{Email}>";
}

// Validation ด้วย DataAnnotations
using System.ComponentModel.DataAnnotations;

static List<string> ValidateModel(object model)
{
    var context = new ValidationContext(model);
    var results = new List<ValidationResult>();

    Validator.TryValidateObject(model, context, results, true);

    return results.Select(r => r.ErrorMessage ?? "Unknown error").ToList();
}

var user = new UserProfile { Username = "ab", Email = "not-email", Age = 10 };
var errors = ValidateModel(user);
foreach (var error in errors)
{
    Console.WriteLine($"❌ {error}");
}
```

---

## 9. โปรแกรมตัวอย่าง: Product Class

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Linq;

namespace ProductExample
{
    /// <summary>
    /// ประเภทสินค้า
    /// </summary>
    public enum ProductCategory
    {
        Electronics,
        Clothing,
        Food,
        Books,
        Sports,
        Other
    }

    /// <summary>
    /// ข้อมูลสินค้า
    /// </summary>
    public class Product : INotifyPropertyChanged
    {
        // Backing fields
        private int _id;
        private string _name = "";
        private string _description = "";
        private decimal _price;
        private decimal _costPrice;
        private int _stockQuantity;
        private ProductCategory _category;
        private bool _isActive;
        private string _sku = "";

        // Static
        private static int _nextId = 1;
        private static readonly Dictionary<int, Product> _registry = new();

        // Event สำหรับ change notification
        public event PropertyChangedEventHandler? PropertyChanged;

        // --- Properties ---

        public int Id
        {
            get => _id;
            private init => _id = value;
        }

        public string SKU
        {
            get => _sku;
            init
            {
                if (string.IsNullOrWhiteSpace(value))
                    throw new ArgumentException("SKU cannot be empty.");
                _sku = value.ToUpper().Trim();
            }
        }

        public string Name
        {
            get => _name;
            set
            {
                if (string.IsNullOrWhiteSpace(value))
                    throw new ArgumentException("Product name cannot be empty.");
                string trimmed = value.Trim();
                if (_name != trimmed)
                {
                    _name = trimmed;
                    NotifyPropertyChanged(nameof(Name));
                }
            }
        }

        public string Description
        {
            get => _description;
            set
            {
                string trimmed = (value ?? "").Trim();
                if (_description != trimmed)
                {
                    _description = trimmed;
                    NotifyPropertyChanged(nameof(Description));
                }
            }
        }

        public decimal Price
        {
            get => _price;
            set
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException(nameof(value),
                        "Price cannot be negative.");
                if (_price != value)
                {
                    _price = value;
                    NotifyPropertyChanged(nameof(Price));
                    NotifyPropertyChanged(nameof(ProfitMargin));
                    NotifyPropertyChanged(nameof(PriceCategory));
                }
            }
        }

        public decimal CostPrice
        {
            get => _costPrice;
            set
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException(nameof(value),
                        "Cost price cannot be negative.");
                if (_costPrice != value)
                {
                    _costPrice = value;
                    NotifyPropertyChanged(nameof(CostPrice));
                    NotifyPropertyChanged(nameof(ProfitMargin));
                    NotifyPropertyChanged(nameof(GrossProfit));
                }
            }
        }

        public int StockQuantity
        {
            get => _stockQuantity;
            set
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException(nameof(value),
                        "Stock quantity cannot be negative.");
                if (_stockQuantity != value)
                {
                    _stockQuantity = value;
                    NotifyPropertyChanged(nameof(StockQuantity));
                    NotifyPropertyChanged(nameof(IsInStock));
                    NotifyPropertyChanged(nameof(StockStatus));
                    NotifyPropertyChanged(nameof(InventoryValue));
                }
            }
        }

        public ProductCategory Category
        {
            get => _category;
            set
            {
                if (_category != value)
                {
                    _category = value;
                    NotifyPropertyChanged(nameof(Category));
                }
            }
        }

        public bool IsActive
        {
            get => _isActive;
            set
            {
                if (_isActive != value)
                {
                    _isActive = value;
                    NotifyPropertyChanged(nameof(IsActive));
                }
            }
        }

        // Dates - init only
        public DateTime CreatedAt { get; init; }
        public DateTime UpdatedAt { get; private set; }

        // Tags
        private readonly List<string> _tags = new();
        public IReadOnlyList<string> Tags => _tags.AsReadOnly();

        // --- Computed Properties ---

        public decimal GrossProfit => _price - _costPrice;

        public decimal ProfitMargin
        {
            get
            {
                if (_price == 0) return 0;
                return (GrossProfit / _price) * 100;
            }
        }

        public bool IsInStock => _stockQuantity > 0;

        public string StockStatus
        {
            get
            {
                if (!_isActive) return "ไม่ใช้งาน";
                return _stockQuantity switch
                {
                    0 => "หมดสต็อก",
                    <= 5 => $"ใกล้หมด ({_stockQuantity} ชิ้น)",
                    <= 20 => $"สต็อกปกติ ({_stockQuantity} ชิ้น)",
                    _ => $"สต็อกเพียงพอ ({_stockQuantity} ชิ้น)"
                };
            }
        }

        public decimal InventoryValue => _price * _stockQuantity;

        public string PriceCategory
        {
            get
            {
                return _price switch
                {
                    < 100 => "ราคาประหยัด",
                    < 500 => "ราคากลาง",
                    < 2000 => "ราคาสูง",
                    _ => "ราคาพรีเมียม"
                };
            }
        }

        public string DisplayName => $"[{SKU}] {_name}";

        // Static property
        public static int TotalProducts => _registry.Count;

        // --- Constructor ---

        public Product(string sku, string name, decimal price, decimal costPrice,
            ProductCategory category = ProductCategory.Other)
        {
            _id = _nextId++;
            SKU = sku;
            Name = name;
            Price = price;
            CostPrice = costPrice;
            Category = category;
            IsActive = true;
            CreatedAt = DateTime.Now;
            UpdatedAt = DateTime.Now;

            _registry[_id] = this;
        }

        // --- Methods ---

        public void AddTag(string tag)
        {
            string normalized = tag.Trim().ToLower();
            if (!_tags.Contains(normalized))
            {
                _tags.Add(normalized);
                UpdatedAt = DateTime.Now;
                NotifyPropertyChanged(nameof(Tags));
            }
        }

        public void RemoveTag(string tag)
        {
            string normalized = tag.Trim().ToLower();
            if (_tags.Remove(normalized))
            {
                UpdatedAt = DateTime.Now;
                NotifyPropertyChanged(nameof(Tags));
            }
        }

        public bool AddStock(int quantity, string reason = "")
        {
            if (quantity <= 0) return false;
            StockQuantity += quantity;
            UpdatedAt = DateTime.Now;
            Console.WriteLine($"เพิ่มสต็อก {quantity} ชิ้น → คงเหลือ {StockQuantity}" +
                (string.IsNullOrEmpty(reason) ? "" : $" ({reason})"));
            return true;
        }

        public bool ReduceStock(int quantity, string reason = "")
        {
            if (quantity <= 0 || quantity > _stockQuantity) return false;
            StockQuantity -= quantity;
            UpdatedAt = DateTime.Now;
            Console.WriteLine($"ลดสต็อก {quantity} ชิ้น → คงเหลือ {StockQuantity}" +
                (string.IsNullOrEmpty(reason) ? "" : $" ({reason})"));
            return true;
        }

        public void ApplyDiscount(decimal discountPercent)
        {
            if (discountPercent < 0 || discountPercent > 100)
                throw new ArgumentOutOfRangeException(nameof(discountPercent));

            decimal discountAmount = Price * (discountPercent / 100);
            Price -= discountAmount;
            UpdatedAt = DateTime.Now;
            Console.WriteLine($"ลดราคา {discountPercent}% = {discountAmount:N2} บาท → ราคาใหม่ {Price:N2} บาท");
        }

        // Static methods
        public static Product? FindById(int id)
            => _registry.TryGetValue(id, out var p) ? p : null;

        public static IEnumerable<Product> FindByCategory(ProductCategory category)
            => _registry.Values.Where(p => p.Category == category && p.IsActive);

        public static IEnumerable<Product> GetLowStockProducts(int threshold = 5)
            => _registry.Values.Where(p => p.IsActive && p.StockQuantity <= threshold);

        public static decimal GetTotalInventoryValue()
            => _registry.Values.Where(p => p.IsActive).Sum(p => p.InventoryValue);

        // Private helper
        private void NotifyPropertyChanged(string propertyName)
        {
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
        }

        public override string ToString()
        {
            return $"{DisplayName} | ราคา: {Price:N2} | กำไร: {ProfitMargin:F1}% | {StockStatus}";
        }

        public string GetDetailReport()
        {
            return $"""
                === Product Details ===
                ID     : {Id}
                SKU    : {SKU}
                Name   : {Name}
                Cat    : {Category}
                Price  : {Price:N2} บาท ({PriceCategory})
                Cost   : {CostPrice:N2} บาท
                Profit : {GrossProfit:N2} บาท ({ProfitMargin:F1}%)
                Stock  : {StockStatus}
                Value  : {InventoryValue:N2} บาท
                Tags   : {(Tags.Count > 0 ? string.Join(", ", Tags) : "ไม่มี")}
                Active : {(IsActive ? "Yes" : "No")}
                Created: {CreatedAt:dd/MM/yyyy HH:mm}
                Updated: {UpdatedAt:dd/MM/yyyy HH:mm}
                """;
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("=== ระบบจัดการสินค้า ===\n");

            // สร้างสินค้า
            var laptop = new Product("LAP-001", "Laptop Pro 15\"", 35000, 22000,
                ProductCategory.Electronics);
            laptop.Description = "Laptop สำหรับมืออาชีพ CPU Intel i7";
            laptop.AddTag("laptop");
            laptop.AddTag("computer");
            laptop.AddTag("sale");
            laptop.AddStock(10, "สต็อกเริ่มต้น");

            var mouse = new Product("MOU-001", "Wireless Mouse", 850, 400,
                ProductCategory.Electronics);
            mouse.AddStock(50, "สต็อกเริ่มต้น");

            var shirt = new Product("CLO-001", "เสื้อโปโล", 290, 120,
                ProductCategory.Clothing);
            shirt.AddStock(3, "สต็อกน้อย - ต้องสั่งเพิ่ม");

            // แสดงรายละเอียด
            Console.WriteLine(laptop.GetDetailReport());

            // ทดสอบ property changes
            Console.WriteLine("\n=== การเปลี่ยนแปลง ===");
            laptop.PropertyChanged += (sender, e) =>
            {
                Console.WriteLine($"Property '{e.PropertyName}' changed on {((Product)sender!).Name}");
            };

            laptop.Price = 32000; // ลดราคา
            laptop.ReduceStock(2, "ขาย 2 เครื่อง");
            laptop.AddTag("bestseller");

            // สรุปสต็อก
            Console.WriteLine("\n=== สรุปสต็อกต่ำ ===");
            foreach (var p in Product.GetLowStockProducts(5))
            {
                Console.WriteLine($"  ⚠️ {p}");
            }

            Console.WriteLine($"\nมูลค่าสินค้าทั้งหมด: {Product.GetTotalInventoryValue():N2} บาท");
            Console.WriteLine($"จำนวนสินค้าทั้งหมด: {Product.TotalProducts} รายการ");
        }
    }
}
```

---

## Exercises

### Exercise 1: Temperature Converter
```csharp
// TODO: สร้าง class TemperatureUnit ที่มี:
// - Properties: Celsius, Fahrenheit, Kelvin (แต่ละอันเป็นทั้ง getter และ setter)
// - เมื่อ set ค่าใดค่าหนึ่ง อีกสองค่าต้องอัปเดตอัตโนมัติ
// - Property: Scale (string ว่าอยู่ใน scale ไหน)
// - Validation: Kelvin ต้องไม่ต่ำกว่า 0
// ตัวอย่าง: ถ้า set Celsius = 100 แล้ว Fahrenheit = 212, Kelvin = 373.15

public class TemperatureUnit
{
    // TODO: Implement
}
```

### Exercise 2: Rectangle class
```csharp
// TODO: สร้าง class Rectangle ที่มี:
// - Init-only properties: Width, Height
// - Computed: Area, Perimeter, Diagonal, IsSquare, AspectRatio
// - Method: Scale(double factor) -> Rectangle (new object)
// - Method: CanFitInside(Rectangle other) -> bool
// - Operator overloading: == (เปรียบเทียบขนาด)

public class Rectangle
{
    // TODO: Implement
}
```

### Exercise 3: BankAccount v2
```csharp
// TODO: ปรับปรุง BankAccount จาก Part 012 โดย:
// - ใช้ INotifyPropertyChanged
// - Balance เป็น read-only computed จาก transactions
// - เพิ่ม InterestRate property พร้อม validation (0-100%)
// - เพิ่ม AccountType enum (Savings/Checking/Fixed)
// - เพิ่ม computed MonthlyInterest
// - เพิ่ม IsOverdrawn computed property

public class BankAccountV2 : INotifyPropertyChanged
{
    // TODO: Implement
}
```

---

## สรุป

✅ Properties ดีกว่า public fields เพราะมี encapsulation และ validation  
✅ Auto-implemented properties ใช้เมื่อไม่ต้องการ logic พิเศษ  
✅ Full properties ใช้เมื่อต้องการ validation หรือ side effects  
✅ Read-only properties (get only) ป้องกันการแก้ไขจากภายนอก  
✅ Init-only properties (C# 9+) กำหนดได้ครั้งเดียวตอน initialization  
✅ Computed properties คำนวณค่าจาก fields อื่น ไม่ต้องเก็บซ้ำ  
✅ INotifyPropertyChanged ใช้สำหรับ data binding และ reactive UI  
✅ Lazy loading ใน property ช่วยประหยัด performance  

## Part ถัดไป
**Part 014: Constructors** - เรียนรู้ default/parameterized constructors, overloading, chaining, object initializers และ destructor

---
*Part 013/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
