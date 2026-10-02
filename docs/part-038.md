# Part 038: Records (C# 9+)

## เนื้อหาใน Part นี้
- record class vs record struct
- Init-only properties
- Positional record syntax
- With expressions (non-destructive mutation)
- Value equality
- Inheritance กับ records
- โปรแกรมตัวอย่าง: Immutable Domain Models

---

## 1. record class vs record struct

```csharp
using System;
using System.Collections.Generic;

// record class (default) - reference type, heap allocated
public record class PersonRecord
{
    public string Name { get; init; } = string.Empty;
    public int Age { get; init; }
}

// record struct (C# 10+) - value type, stack allocated
public record struct PointStruct
{
    public double X { get; init; }
    public double Y { get; init; }
}

// readonly record struct - fully immutable
public readonly record struct ColorStruct(byte R, byte G, byte B);

class RecordComparison
{
    static void Main()
    {
        // record class
        var p1 = new PersonRecord { Name = "สมชาย", Age = 25 };
        var p2 = new PersonRecord { Name = "สมชาย", Age = 25 };
        var p3 = p1; // reference copy
        
        Console.WriteLine("--- record class ---");
        Console.WriteLine($"p1 == p2: {p1 == p2}");          // true (value equality)
        Console.WriteLine($"ReferenceEqual(p1, p2): {ReferenceEquals(p1, p2)}"); // false
        Console.WriteLine($"ReferenceEqual(p1, p3): {ReferenceEquals(p1, p3)}"); // true
        
        // record struct
        var pt1 = new PointStruct { X = 1.0, Y = 2.0 };
        var pt2 = new PointStruct { X = 1.0, Y = 2.0 };
        var pt3 = pt1; // value copy
        pt3 = pt3 with { X = 99 }; // ไม่กระทบ pt1
        
        Console.WriteLine("\n--- record struct ---");
        Console.WriteLine($"pt1 == pt2: {pt1 == pt2}");         // true
        Console.WriteLine($"pt1 == pt3: {pt1 == pt3}");         // false
        Console.WriteLine($"pt1.X: {pt1.X}, pt3.X: {pt3.X}");  // 1.0, 99
        
        // ข้อแตกต่างสำคัญ
        Console.WriteLine("\n--- Type Info ---");
        Console.WriteLine($"PersonRecord IsClass: {typeof(PersonRecord).IsClass}");   // true
        Console.WriteLine($"PointStruct IsValueType: {typeof(PointStruct).IsValueType}"); // true
        Console.WriteLine($"ColorStruct IsValueType: {typeof(ColorStruct).IsValueType}"); // true
        
        // ToString อัตโนมัติ
        Console.WriteLine($"\nToString: {p1}");  // PersonRecord { Name = สมชาย, Age = 25 }
        Console.WriteLine($"ToString: {pt1}");   // PointStruct { X = 1, Y = 2 }
        
        // GetHashCode สอดคล้องกับ equality
        Console.WriteLine($"\np1.GetHashCode() == p2.GetHashCode(): {p1.GetHashCode() == p2.GetHashCode()}"); // true
    }
}
```

---

## 2. Init-only Properties

```csharp
using System;

class InitOnlyExample
{
    // init accessor - set ได้แค่ตอน object initialization
    public class Configuration
    {
        public string ConnectionString { get; init; } = string.Empty;
        public int MaxRetries { get; init; } = 3;
        public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
        
        // ใช้ required (C# 11) บังคับให้ต้องกำหนดค่า
        public required string DatabaseName { get; init; }
    }
    
    static void Main()
    {
        // ✅ กำหนดค่าตอน init
        var config = new Configuration
        {
            DatabaseName = "MyDB",
            ConnectionString = "Server=localhost;",
            MaxRetries = 5
        };
        
        Console.WriteLine($"DB: {config.DatabaseName}");
        Console.WriteLine($"Timeout: {config.Timeout.TotalSeconds}s");
        
        // ❌ แก้ไขหลัง init ไม่ได้
        // config.DatabaseName = "NewDB"; // Compile error!
        
        // ✅ ใช้ with expression สร้าง copy ที่แก้ไขแล้ว
        var newConfig = config with { MaxRetries = 10 };
        Console.WriteLine($"\nOriginal MaxRetries: {config.MaxRetries}");
        Console.WriteLine($"New MaxRetries: {newConfig.MaxRetries}");
        
        // required กับ object initializer
        // var missingRequired = new Configuration(); // Compile error - DatabaseName ต้องกำหนด
    }
}
```

---

## 3. Positional Record Syntax

```csharp
using System;
using System.Collections.Generic;

// Positional syntax - compact กว่า
public record Point(double X, double Y);

// เหมือนกับ:
// public record Point
// {
//     public double X { get; init; }
//     public double Y { get; init; }
//     public Point(double x, double y) { X = x; Y = y; }
//     // + Deconstruct, ==, GetHashCode, ToString
// }

// Positional record struct
public record struct Size(double Width, double Height);

// ผสมกัน
public record Person(string FirstName, string LastName)
{
    // เพิ่ม computed property
    public string FullName => $"{FirstName} {LastName}";
    
    // เพิ่ม method
    public Person WithTitle(string title) => this with { FirstName = $"{title} {FirstName}" };
    
    // เพิ่ม property ที่ไม่ใช่ positional
    public string? Email { get; init; }
    public DateTime? BirthDate { get; init; }
}

class PositionalRecordExample
{
    static void Main()
    {
        // สร้างด้วย positional constructor
        var p1 = new Point(3.0, 4.0);
        Console.WriteLine($"Point: {p1}");
        Console.WriteLine($"Distance: {Math.Sqrt(p1.X * p1.X + p1.Y * p1.Y):F2}");
        
        // Deconstruct
        var (x, y) = p1;
        Console.WriteLine($"Destructured: x={x}, y={y}");
        
        // Person กับ extra properties
        var person = new Person("สมชาย", "ใจดี")
        {
            Email = "somchai@example.com",
            BirthDate = new DateTime(1990, 1, 15)
        };
        
        Console.WriteLine($"\n{person.FullName} ({person.Email})");
        
        // Deconstruct ได้ positional parameters เท่านั้น
        var (first, last) = person;
        Console.WriteLine($"Deconstructed: {first} {last}");
        
        // WithTitle method
        var dr = person.WithTitle("Dr.");
        Console.WriteLine($"With title: {dr.FullName}");
        
        // Collection of records
        var people = new List<Person>
        {
            new("อนุชา", "สมบูรณ์") { Email = "anucha@test.com" },
            new("วิภาวดี", "มั่นคง") { Email = "viphavadee@test.com" },
            new("ธนาคาร", "ดีเลิศ"),
        };
        
        var withEmail = people.Where(p => p.Email != null);
        foreach (var p in withEmail)
        {
            Console.WriteLine($"  {p.FullName}: {p.Email}");
        }
    }
}
```

---

## 4. With Expressions (Non-destructive Mutation)

```csharp
using System;
using System.Collections.Generic;
using System.Collections.Immutable;

public record Address(
    string Street,
    string City,
    string State,
    string Country,
    string PostalCode
);

public record Customer(
    int Id,
    string Name,
    string Email,
    Address BillingAddress,
    Address ShippingAddress
)
{
    public bool IsVerified { get; init; }
    public ImmutableList<string> Tags { get; init; } = ImmutableList<string>.Empty;
}

class WithExpressionExample
{
    static void Main()
    {
        var original = new Customer(
            Id: 1,
            Name: "สมชาย ใจดี",
            Email: "somchai@example.com",
            BillingAddress: new Address("123 ถนนสุขุมวิท", "Bangkok", "BKK", "TH", "10110"),
            ShippingAddress: new Address("456 ถนนพหลโยธิน", "Bangkok", "BKK", "TH", "10900")
        );
        
        // with expression - สร้าง copy ที่แก้ไขเฉพาะส่วนที่ระบุ
        var updated = original with { Email = "new@example.com" };
        
        Console.WriteLine($"Original email: {original.Email}");
        Console.WriteLine($"Updated email: {updated.Email}");
        Console.WriteLine($"Same Id: {original.Id == updated.Id}"); // true
        
        // กับ nested records
        var movedCustomer = original with
        {
            ShippingAddress = original.ShippingAddress with
            {
                City = "Chiang Mai",
                PostalCode = "50000"
            }
        };
        
        Console.WriteLine($"\nOriginal shipping: {original.ShippingAddress.City}");
        Console.WriteLine($"New shipping: {movedCustomer.ShippingAddress.City}");
        
        // กับ ImmutableList
        var taggedCustomer = original with
        {
            IsVerified = true,
            Tags = original.Tags.Add("premium").Add("verified")
        };
        
        Console.WriteLine($"\nOriginal tags: [{string.Join(", ", original.Tags)}]");
        Console.WriteLine($"Tagged: [{string.Join(", ", taggedCustomer.Tags)}]");
        
        // เปรียบเทียบ
        Console.WriteLine($"\noriginal == updated: {original == updated}"); // false
        Console.WriteLine($"updated == movedCustomer: {updated == movedCustomer}"); // false
        
        // Version history pattern
        var v1 = new ProductVersion("iPhone 16", 49900, "1.0");
        var v2 = v1 with { Version = "1.1", Price = 48900 };
        var v3 = v2 with { Version = "1.2" };
        
        var history = new[] { v1, v2, v3 };
        Console.WriteLine("\nProduct version history:");
        foreach (var v in history)
        {
            Console.WriteLine($"  v{v.Version}: {v.Name} = {v.Price:N0}");
        }
    }
}

record ProductVersion(string Name, decimal Price, string Version);
```

---

## 5. Value Equality

```csharp
using System;
using System.Collections.Generic;

class ValueEqualityExample
{
    // Regular class - reference equality
    class OrderClass
    {
        public int Id { get; set; }
        public string Product { get; set; } = string.Empty;
    }
    
    // Record - value equality
    record OrderRecord(int Id, string Product);
    
    static void Main()
    {
        // Class equality
        var c1 = new OrderClass { Id = 1, Product = "iPhone" };
        var c2 = new OrderClass { Id = 1, Product = "iPhone" };
        Console.WriteLine($"Class: c1 == c2: {c1 == c2}");     // false
        Console.WriteLine($"Class: Equals: {c1.Equals(c2)}");  // false
        
        // Record equality
        var r1 = new OrderRecord(1, "iPhone");
        var r2 = new OrderRecord(1, "iPhone");
        var r3 = r1;
        
        Console.WriteLine($"\nRecord: r1 == r2: {r1 == r2}");     // true!
        Console.WriteLine($"Record: r1 == r3: {r1 == r3}");       // true
        Console.WriteLine($"Record: Equals: {r1.Equals(r2)}");    // true
        Console.WriteLine($"HashCode equal: {r1.GetHashCode() == r2.GetHashCode()}"); // true
        
        // ใช้ใน HashSet/Dictionary
        var set = new HashSet<OrderRecord> { r1, r2 }; // จะมีแค่ 1 element!
        Console.WriteLine($"\nHashSet count: {set.Count}"); // 1
        
        var dict = new Dictionary<OrderRecord, string>
        {
            [r1] = "First entry"
        };
        Console.WriteLine($"Dict[r2]: {dict[r2]}"); // "First entry" - เข้าถึงด้วย r2 ได้!
        
        // Record พร้อม custom equality
        var addresses = new List<Address>
        {
            new("Main St", "Bangkok"),
            new("Main St", "Bangkok"),  // duplicate
            new("Second St", "Bangkok")
        };
        
        var unique = addresses.Distinct().ToList();
        Console.WriteLine($"\nAll addresses: {addresses.Count}");
        Console.WriteLine($"Unique addresses: {unique.Count}"); // 2
        
        // Null equality
        OrderRecord? nullRecord = null;
        Console.WriteLine($"\nnull == null: {nullRecord == null}"); // true
        Console.WriteLine($"r1 != null: {r1 != null}");            // true
    }
}

record Address(string Street, string City);
```

---

## 6. Inheritance กับ Records

```csharp
using System;
using System.Collections.Generic;

// Base record
public abstract record Animal(string Name, int Age);

// Derived records
public record Dog(string Name, int Age, string Breed) : Animal(Name, Age)
{
    public bool IsVaccinated { get; init; }
}

public record Cat(string Name, int Age, bool IsIndoor) : Animal(Name, Age);

public record Bird(string Name, int Age, bool CanFly) : Animal(Name, Age);

// ข้อควรระวัง: inheritance ใน records มีข้อจำกัด

class RecordInheritance
{
    static void PrintAnimal(Animal animal)
    {
        string description = animal switch
        {
            Dog { Breed: "Labrador", IsVaccinated: true } d =>
                $"🐕 {d.Name}: Vaccinated Labrador",
            Dog { IsVaccinated: false } d =>
                $"🐕 {d.Name} ({d.Breed}): ยังไม่ได้ฉีดวัคซีน!",
            Dog d =>
                $"🐕 {d.Name} ({d.Breed})",
            Cat { IsIndoor: true } c =>
                $"🐱 {c.Name}: แมวในบ้าน",
            Cat c =>
                $"🐱 {c.Name}: แมวนอกบ้าน",
            Bird { CanFly: true } b =>
                $"🐦 {b.Name}: บินได้",
            Bird b =>
                $"🐦 {b.Name}: บินไม่ได้ (เช่น นกกระจอกเทศ)",
            _ =>
                $"❓ {animal.Name}"
        };
        
        Console.WriteLine($"  {description} (อายุ {animal.Age} ปี)");
    }
    
    static void Main()
    {
        var animals = new Animal[]
        {
            new Dog("บัดดี้", 3, "Labrador") { IsVaccinated = true },
            new Dog("แม็กซ์", 2, "Poodle") { IsVaccinated = false },
            new Cat("มิสตี้", 5, IsIndoor: true),
            new Cat("ไวลด์", 2, IsIndoor: false),
            new Bird("ทวีตตี้", 1, CanFly: true),
            new Bird("เพนกวิน", 4, CanFly: false),
        };
        
        Console.WriteLine("Animals:");
        foreach (var animal in animals)
        {
            PrintAnimal(animal);
        }
        
        // Equality กับ inheritance
        var dog1 = new Dog("บัดดี้", 3, "Labrador") { IsVaccinated = true };
        var dog2 = new Dog("บัดดี้", 3, "Labrador") { IsVaccinated = true };
        Animal animalRef = dog1;
        
        Console.WriteLine($"\ndog1 == dog2: {dog1 == dog2}"); // true
        Console.WriteLine($"animalRef == dog1: {animalRef == dog1}"); // true
        
        // ข้อควรระวัง: subtype ต่างกัน = ไม่เท่ากัน
        var cat = new Cat("บัดดี้", 3, true);
        // dog1 == cat; // Compile error - ต่าง type
        Console.WriteLine($"dog1 Equals cat: {dog1.Equals(cat)}"); // false
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Immutable Domain Models

```csharp
using System;
using System.Collections.Generic;
using System.Collections.Immutable;
using System.Linq;
using System.Text.Json;
using System.Text.Json.Serialization;

// ===== Domain Models =====

public record Money(decimal Amount, string Currency)
{
    public static readonly Money Zero = new(0, "THB");
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return this with { Amount = Amount + other.Amount };
    }
    
    public Money Multiply(decimal factor) => this with { Amount = Amount * factor };
    public Money ApplyDiscount(decimal discountPercent) =>
        Multiply(1 - discountPercent / 100);
    
    public override string ToString() => $"{Amount:N2} {Currency}";
    
    public static Money operator +(Money a, Money b) => a.Add(b);
    public static Money operator *(Money m, decimal factor) => m.Multiply(factor);
    public static bool operator >(Money a, Money b) => a.Amount > b.Amount;
    public static bool operator <(Money a, Money b) => a.Amount < b.Amount;
}

public record ProductId(Guid Value)
{
    public static ProductId New() => new(Guid.NewGuid());
    public override string ToString() => Value.ToString("N")[..8];
}

public record Product(
    ProductId Id,
    string Name,
    string Description,
    Money Price,
    int StockQuantity,
    string Category
)
{
    public bool IsInStock => StockQuantity > 0;
    public bool IsLowStock => StockQuantity > 0 && StockQuantity <= 5;
    
    public Product WithPrice(Money newPrice) => this with { Price = newPrice };
    public Product WithStock(int newStock) => this with { StockQuantity = newStock };
    public Product Restock(int quantity) => this with { StockQuantity = StockQuantity + quantity };
}

public record OrderLineItem(
    Product Product,
    int Quantity,
    Money UnitPrice
)
{
    public Money Subtotal => UnitPrice * Quantity;
    
    public OrderLineItem WithQuantity(int qty) => this with { Quantity = qty };
}

public enum OrderStatus
{
    Draft,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

public record Order(
    Guid Id,
    string CustomerId,
    ImmutableList<OrderLineItem> Items,
    OrderStatus Status,
    DateTime CreatedAt
)
{
    public static Order Create(string customerId) => new(
        Id: Guid.NewGuid(),
        CustomerId: customerId,
        Items: ImmutableList<OrderLineItem>.Empty,
        Status: OrderStatus.Draft,
        CreatedAt: DateTime.UtcNow
    );
    
    public Money Total => Items.Aggregate(Money.Zero, (sum, item) => sum + item.Subtotal);
    public int TotalQuantity => Items.Sum(i => i.Quantity);
    public bool IsEmpty => !Items.Any();
    
    // Non-destructive operations return new Order
    public Order AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Only draft orders can be modified");
        
        var existing = Items.FirstOrDefault(i => i.Product.Id == product.Id);
        
        ImmutableList<OrderLineItem> newItems;
        if (existing != null)
        {
            newItems = Items.Replace(existing, existing.WithQuantity(existing.Quantity + quantity));
        }
        else
        {
            newItems = Items.Add(new OrderLineItem(product, quantity, product.Price));
        }
        
        return this with { Items = newItems };
    }
    
    public Order RemoveItem(ProductId productId)
    {
        var item = Items.FirstOrDefault(i => i.Product.Id == productId);
        if (item == null) return this;
        return this with { Items = Items.Remove(item) };
    }
    
    public Order Confirm()
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException($"Cannot confirm order in {Status} status");
        if (IsEmpty)
            throw new InvalidOperationException("Cannot confirm empty order");
        
        return this with { Status = OrderStatus.Confirmed };
    }
    
    public Order Transition(OrderStatus newStatus)
    {
        bool isValid = (Status, newStatus) switch
        {
            (OrderStatus.Draft, OrderStatus.Confirmed) => true,
            (OrderStatus.Confirmed, OrderStatus.Processing) => true,
            (OrderStatus.Processing, OrderStatus.Shipped) => true,
            (OrderStatus.Shipped, OrderStatus.Delivered) => true,
            (OrderStatus.Draft or OrderStatus.Confirmed, OrderStatus.Cancelled) => true,
            (OrderStatus.Delivered, OrderStatus.Refunded) => true,
            _ => false
        };
        
        if (!isValid)
            throw new InvalidOperationException($"Invalid transition: {Status} -> {newStatus}");
        
        return this with { Status = newStatus };
    }
}

// ===== Event Sourcing Pattern =====

public abstract record DomainEvent(Guid AggregateId, DateTime OccurredAt);

public record OrderCreated(Guid AggregateId, string CustomerId, DateTime OccurredAt)
    : DomainEvent(AggregateId, OccurredAt);

public record ItemAdded(Guid AggregateId, ProductId ProductId, int Quantity, Money Price, DateTime OccurredAt)
    : DomainEvent(AggregateId, OccurredAt);

public record OrderConfirmed(Guid AggregateId, Money Total, DateTime OccurredAt)
    : DomainEvent(AggregateId, OccurredAt);

public record OrderCancelled(Guid AggregateId, string Reason, DateTime OccurredAt)
    : DomainEvent(AggregateId, OccurredAt);

// ===== Main Program =====

class Program
{
    static void Main()
    {
        Console.WriteLine("===== Immutable Domain Models Demo =====\n");
        
        // สร้าง products
        var products = new[]
        {
            new Product(ProductId.New(), "iPhone 16 Pro", "Apple iPhone 16 Pro 256GB",
                new Money(49900, "THB"), 10, "smartphones"),
            new Product(ProductId.New(), "AirPods Pro", "Apple AirPods Pro 2nd Gen",
                new Money(9900, "THB"), 25, "accessories"),
            new Product(ProductId.New(), "MacBook Air M3", "Apple MacBook Air M3 16GB",
                new Money(89900, "THB"), 3, "laptops"),
        };
        
        Console.WriteLine("Products:");
        foreach (var p in products)
        {
            Console.WriteLine($"  [{p.Id}] {p.Name}: {p.Price}" +
                $" (Stock: {p.StockQuantity}{(p.IsLowStock ? " ⚠️ LOW" : "")})");
        }
        
        // สร้าง order
        var order = Order.Create("customer-123");
        Console.WriteLine($"\n--- Order {order.Id:N}[..8] Created ---");
        Console.WriteLine($"Status: {order.Status}");
        
        // เพิ่ม items
        order = order.AddItem(products[0], 2);
        order = order.AddItem(products[1], 1);
        Console.WriteLine($"\nAfter adding items:");
        
        foreach (var item in order.Items)
        {
            Console.WriteLine($"  {item.Product.Name} x{item.Quantity} = {item.Subtotal}");
        }
        Console.WriteLine($"  Total: {order.Total}");
        
        // เพิ่มซ้ำ (merge)
        order = order.AddItem(products[1], 2); // รวมเป็น 3
        Console.WriteLine($"\nAfter adding more AirPods:");
        foreach (var item in order.Items)
        {
            Console.WriteLine($"  {item.Product.Name} x{item.Quantity} = {item.Subtotal}");
        }
        Console.WriteLine($"  Total: {order.Total}");
        
        // Confirm order
        order = order.Confirm();
        Console.WriteLine($"\nOrder confirmed. Status: {order.Status}");
        
        // State transitions
        Console.WriteLine("\n--- Order Lifecycle ---");
        var transitions = new[] { OrderStatus.Processing, OrderStatus.Shipped, OrderStatus.Delivered };
        foreach (var status in transitions)
        {
            order = order.Transition(status);
            Console.WriteLine($"  -> {order.Status}");
        }
        
        // Invalid transition
        try
        {
            order = order.Transition(OrderStatus.Cancelled);
        }
        catch (InvalidOperationException ex)
        {
            Console.WriteLine($"\nInvalid transition: {ex.Message}");
        }
        
        // Money operations
        Console.WriteLine("\n--- Money Operations ---");
        var price = new Money(1000, "THB");
        var taxed = price + new Money(70, "THB");
        var discounted = price.ApplyDiscount(10);
        
        Console.WriteLine($"Price: {price}");
        Console.WriteLine($"With tax: {taxed}");
        Console.WriteLine($"10% discount: {discounted}");
        Console.WriteLine($"price > discounted: {price > discounted}");
        
        // Value equality in collections
        Console.WriteLine("\n--- Value Equality ---");
        var productIds = products.Select(p => p.Id).ToList();
        var copy = products.Select(p => new Product(p.Id, p.Name, p.Description, 
            p.Price, p.StockQuantity, p.Category)).ToList();
        
        Console.WriteLine($"Products equal by value: {products[0] == copy[0]}"); // true
        Console.WriteLine($"Same reference: {ReferenceEquals(products[0], copy[0])}"); // false
        
        // Serialize/Deserialize (records work great with JSON)
        var json = JsonSerializer.Serialize(products[0], new JsonSerializerOptions
        {
            WriteIndented = true
        });
        Console.WriteLine($"\nJSON:\n{json[..200]}...");
    }
}
```

---

## Exercises

### Exercise 1: Shopping Cart
```csharp
// TODO: Implement immutable shopping cart

public record CartItem(ProductId ProductId, string Name, Money Price, int Quantity);

public record ShoppingCart(
    Guid Id,
    string SessionId,
    ImmutableList<CartItem> Items,
    DateTime LastModified
)
{
    public static ShoppingCart Create(string sessionId)
    {
        throw new NotImplementedException();
    }
    
    public ShoppingCart AddItem(CartItem item)
    {
        // ถ้ามีอยู่แล้วให้ merge quantity
        throw new NotImplementedException();
    }
    
    public ShoppingCart RemoveItem(ProductId productId)
    {
        throw new NotImplementedException();
    }
    
    public ShoppingCart UpdateQuantity(ProductId productId, int quantity)
    {
        throw new NotImplementedException();
    }
    
    public Money Total => throw new NotImplementedException();
    
    public ShoppingCart ApplyCoupon(string code, decimal discountPercent)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 2: Event Sourced Account
```csharp
// TODO: สร้าง bank account ด้วย event sourcing

public abstract record AccountEvent(Guid AccountId, DateTime Timestamp);
public record AccountOpened(Guid AccountId, string Owner, Money InitialBalance, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);
public record MoneyDeposited(Guid AccountId, Money Amount, string Description, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);
public record MoneyWithdrawn(Guid AccountId, Money Amount, string Description, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);
public record AccountFrozen(Guid AccountId, string Reason, DateTime Timestamp) : AccountEvent(AccountId, Timestamp);

public record BankAccount(
    Guid Id,
    string Owner,
    Money Balance,
    bool IsFrozen,
    ImmutableList<AccountEvent> Events
)
{
    // Replay events เพื่อสร้าง state
    public static BankAccount Replay(IEnumerable<AccountEvent> events)
    {
        throw new NotImplementedException();
    }
    
    public BankAccount Deposit(Money amount, string description)
    {
        throw new NotImplementedException();
    }
    
    public BankAccount Withdraw(Money amount, string description)
    {
        throw new NotImplementedException();
    }
}
```

### Exercise 3: Version-controlled Document
```csharp
// TODO: สร้าง document ที่มี version history
public record DocumentVersion(
    int Version,
    string Content,
    string Author,
    string ChangeMessage,
    DateTime Timestamp
);

public record VersionedDocument(
    Guid Id,
    string Title,
    ImmutableList<DocumentVersion> History
)
{
    public DocumentVersion? CurrentVersion => History.LastOrDefault();
    
    public VersionedDocument Edit(string newContent, string author, string changeMessage)
    {
        throw new NotImplementedException();
    }
    
    public VersionedDocument RollbackTo(int version)
    {
        throw new NotImplementedException();
    }
    
    public string Diff(int fromVersion, int toVersion)
    {
        throw new NotImplementedException();
    }
}
```

---

## สรุป

✅ **record class** เป็น reference type มี value equality, immutable by default

✅ **record struct** (C# 10) เป็น value type เหมาะสำหรับ small immutable structs

✅ **init-only properties** กำหนดค่าได้เฉพาะตอน initialization

✅ **required** (C# 11) บังคับต้องกำหนดค่าใน object initializer

✅ **with expression** สร้าง copy ที่แก้ไขบางส่วนโดยไม่กระทบ original

✅ **Value equality** records เปรียบเทียบด้วยค่า ไม่ใช่ reference

✅ Records ทำงานได้ดีกับ **HashSet**, **Dictionary** ด้วย value equality

✅ เหมาะมากสำหรับ **Domain Models**, **DTOs**, **Event Sourcing**, **Value Objects**

---

## Part ถัดไป

➡️ **Part 039**: Span\<T\> และ Memory\<T\> - High-performance memory management, ArrayPool, string parsing

---

*Part 038/700 | Phase 2: C# ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
