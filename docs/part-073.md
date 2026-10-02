# Part 73: Domain-Driven Design (DDD) เบื้องต้น

## เนื้อหาใน Part นี้
- Bounded Context
- Entities vs Value Objects
- Aggregates
- Domain Events
- Repository (DDD style)
- Domain Services
- โปรแกรมตัวอย่าง: Shopping cart domain model

---

## Domain-Driven Design คืออะไร?

**Domain-Driven Design (DDD)** คือแนวทางการพัฒนาซอฟต์แวร์ที่วาง **Business Domain** เป็นศูนย์กลางของการออกแบบ โดย Eric Evans เป็นผู้คิดค้นขึ้นในปี 2003

### แนวคิดหลัก

> "ความซับซ้อนส่วนมากในซอฟต์แวร์ไม่ได้อยู่ที่เทคนิค แต่อยู่ที่ Domain ของธุรกิจ"

DDD เน้น:
1. **Ubiquitous Language** - ภาษากลางระหว่าง developer และ domain expert
2. **Model ที่สะท้อน Domain จริง** - โค้ดควรตรงกับวิธีที่ธุรกิจคิด
3. **Bounded Contexts** - แบ่งระบบออกเป็น context ย่อยๆ

---

## Bounded Context

Bounded Context คือขอบเขตที่ model และ language ใช้ได้ในความหมายเดียวกัน

### ตัวอย่าง: ระบบ E-Commerce

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   Order Context  │    │ Catalog Context  │    │ Shipping Context │
│                  │    │                  │    │                  │
│  Order           │    │  Product         │    │  Shipment        │
│  OrderItem       │    │  Category        │    │  Address         │
│  Payment         │    │  Price           │    │  TrackingInfo    │
│                  │    │                  │    │                  │
│  "Product"       │    │  "Product"       │    │  "Product"       │
│  = ชื่อ+ราคา    │    │  = รายละเอียด    │    │  = น้ำหนัก+ขนาด │
│    ณ เวลาสั่ง    │    │    สินค้าเต็ม   │    │    สำหรับจัดส่ง  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

แม้ว่าทั้ง 3 context ใช้คำว่า "Product" แต่ความหมายและ model ต่างกัน

```csharp
// Order Context - Product คือสิ่งที่ถูกสั่งซื้อ
namespace OrderContext.Domain;

public class OrderLine // ใน Order context ไม่เรียก "OrderItem"
{
    public string ProductSku { get; private set; }  // แค่ SKU พอ
    public string ProductName { get; private set; } // ชื่อ ณ เวลาสั่ง
    public Money Price { get; private set; }        // ราคา ณ เวลาสั่ง
    public int Quantity { get; private set; }
}

// Catalog Context - Product คือสินค้าที่ขาย
namespace CatalogContext.Domain;

public class Product
{
    public Sku Sku { get; private set; }
    public ProductName Name { get; private set; }
    public Money ListPrice { get; private set; }
    public ProductDescription Description { get; private set; }
    public Category Category { get; private set; }
    public List<ProductImage> Images { get; private set; }
    public StockLevel Stock { get; private set; }
}

// Shipping Context - Product คือสิ่งที่ต้องจัดส่ง
namespace ShippingContext.Domain;

public class ShipmentItem
{
    public string Sku { get; private set; }
    public Weight Weight { get; private set; }
    public Dimension Dimension { get; private set; }
    public bool IsFragile { get; private set; }
    public bool RequiresRefrigeration { get; private set; }
}
```

---

## Entities vs Value Objects

### Entities

Entity คือ object ที่มี **Identity** (ตัวตน) ที่คงอยู่ตลอดเวลา

```csharp
// Entity - มี Id ที่ไม่เปลี่ยน
public abstract class Entity
{
    public Guid Id { get; protected set; }
    
    protected Entity() { }
    protected Entity(Guid id) { Id = id; }
    
    // Two entities are equal if they have the same Id
    public override bool Equals(object? obj)
    {
        if (obj is not Entity other) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return Id == other.Id;
    }
    
    public override int GetHashCode() => Id.GetHashCode();
    
    public static bool operator ==(Entity? a, Entity? b)
    {
        if (a is null && b is null) return true;
        if (a is null || b is null) return false;
        return a.Equals(b);
    }
    
    public static bool operator !=(Entity? a, Entity? b) => !(a == b);
}

// Customer Entity - ตัวตนสำคัญกว่าค่า
public class Customer : Entity
{
    public string FirstName { get; private set; }
    public string LastName { get; private set; }
    public Email Email { get; private set; }
    public DateTime RegisteredAt { get; private set; }
    
    protected Customer() { }
    
    public static Customer Create(string firstName, string lastName, string email)
    {
        return new Customer
        {
            Id = Guid.NewGuid(),
            FirstName = firstName,
            LastName = lastName,
            Email = Email.Create(email),
            RegisteredAt = DateTime.UtcNow
        };
    }
    
    public void UpdateEmail(string newEmail)
    {
        Email = Email.Create(newEmail);
        // ทุก Customer มี Id ของตัวเอง แม้เปลี่ยน email ก็ยังเป็น entity เดิม
    }
    
    public string FullName => $"{FirstName} {LastName}";
}
```

### Value Objects

Value Object คือ object ที่ถูกกำหนดโดย **ค่า** ไม่มี identity

```csharp
// Value Object - ไม่มี Id, เท่ากันถ้าค่าเท่ากัน
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();
    
    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        var other = (ValueObject)obj;
        return GetEqualityComponents().SequenceEqual(other.GetEqualityComponents());
    }
    
    public override int GetHashCode()
        => GetEqualityComponents()
            .Select(x => x?.GetHashCode() ?? 0)
            .Aggregate((x, y) => x ^ y);
}

// Email Value Object
public class Email : ValueObject
{
    public string Value { get; private set; }
    
    private Email(string value) { Value = value; }
    
    public static Email Create(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            throw new DomainException("Email ต้องไม่ว่างเปล่า");
            
        email = email.Trim().ToLowerInvariant();
        
        if (!IsValidEmail(email))
            throw new DomainException($"รูปแบบ email ไม่ถูกต้อง: {email}");
            
        return new Email(email);
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
    
    protected override IEnumerable<object?> GetEqualityComponents()
        => new[] { Value };
        
    public override string ToString() => Value;
    
    public static implicit operator string(Email email) => email.Value;
}

// Money Value Object
public class Money : ValueObject
{
    public decimal Amount { get; private set; }
    public string Currency { get; private set; }
    
    private Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
    
    public static Money Of(decimal amount, string currency = "THB")
    {
        if (amount < 0)
            throw new DomainException("จำนวนเงินต้องไม่ติดลบ");
        if (string.IsNullOrWhiteSpace(currency))
            throw new DomainException("สกุลเงินต้องไม่ว่าง");
            
        return new Money(Math.Round(amount, 2), currency.ToUpperInvariant());
    }
    
    public static Money Zero(string currency = "THB") => Of(0, currency);
    
    public Money Add(Money other)
    {
        EnsureSameCurrency(other);
        return Of(Amount + other.Amount, Currency);
    }
    
    public Money Subtract(Money other)
    {
        EnsureSameCurrency(other);
        var result = Amount - other.Amount;
        if (result < 0) throw new DomainException("จำนวนเงินติดลบ");
        return Of(result, Currency);
    }
    
    public Money Multiply(int factor)
        => Of(Amount * factor, Currency);
        
    public Money ApplyDiscount(decimal percentage)
    {
        if (percentage < 0 || percentage > 100)
            throw new DomainException("เปอร์เซ็นต์ส่วนลดต้องอยู่ระหว่าง 0-100");
        return Of(Amount * (1 - percentage / 100), Currency);
    }
    
    private void EnsureSameCurrency(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException($"ไม่สามารถดำเนินการกับสกุลเงินต่างกัน: {Currency} และ {other.Currency}");
    }
    
    protected override IEnumerable<object?> GetEqualityComponents()
        => new object[] { Amount, Currency };
        
    public override string ToString() => $"{Amount:N2} {Currency}";
}

// Address Value Object
public class Address : ValueObject
{
    public string Street { get; private set; }
    public string City { get; private set; }
    public string Province { get; private set; }
    public string PostalCode { get; private set; }
    public string Country { get; private set; }
    
    private Address(string street, string city, string province, string postalCode, string country)
    {
        Street = street;
        City = city;
        Province = province;
        PostalCode = postalCode;
        Country = country;
    }
    
    public static Address Create(
        string street, string city, string province, 
        string postalCode, string country = "TH")
    {
        if (string.IsNullOrWhiteSpace(street)) throw new DomainException("ที่อยู่ต้องไม่ว่าง");
        if (string.IsNullOrWhiteSpace(city)) throw new DomainException("เมืองต้องไม่ว่าง");
        if (string.IsNullOrWhiteSpace(postalCode)) throw new DomainException("รหัสไปรษณีย์ต้องไม่ว่าง");
        
        return new Address(street.Trim(), city.Trim(), province.Trim(), postalCode.Trim(), country.Trim());
    }
    
    protected override IEnumerable<object?> GetEqualityComponents()
        => new[] { Street, City, Province, PostalCode, Country };
        
    public override string ToString() 
        => $"{Street}, {City}, {Province} {PostalCode}, {Country}";
}
```

---

## Aggregates

Aggregate คือกลุ่มของ Entities และ Value Objects ที่ถูกจัดการเป็นหน่วยเดียว มี **Aggregate Root** เป็นจุดเข้าถึง

### กฎของ Aggregate

1. **Transactions** ต้องไม่ข้ามขอบเขต Aggregate
2. **References** ระหว่าง Aggregates ใช้ Id เท่านั้น
3. **Aggregate Root** เป็นเดียวที่ Repository เข้าถึงได้

```csharp
// Aggregate Root
public class ShoppingCart : Entity, IAggregateRoot
{
    private readonly List<CartItem> _items = new();
    private readonly List<IDomainEvent> _domainEvents = new();
    
    public IReadOnlyCollection<CartItem> Items => _items.AsReadOnly();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    public Guid CustomerId { get; private set; }
    public CartStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime LastModifiedAt { get; private set; }
    
    protected ShoppingCart() { }
    
    public static ShoppingCart Create(Guid customerId)
    {
        if (customerId == Guid.Empty)
            throw new DomainException("CustomerId ต้องไม่ว่าง");
            
        return new ShoppingCart
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = CartStatus.Active,
            CreatedAt = DateTime.UtcNow,
            LastModifiedAt = DateTime.UtcNow
        };
    }
    
    public void AddItem(Guid productId, string productName, Money price, int quantity)
    {
        EnsureCartIsActive();
        
        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        
        if (existingItem is not null)
        {
            existingItem.IncreaseQuantity(quantity);
        }
        else
        {
            var item = CartItem.Create(Id, productId, productName, price, quantity);
            _items.Add(item);
        }
        
        LastModifiedAt = DateTime.UtcNow;
        AddDomainEvent(new ItemAddedToCartEvent(Id, productId, quantity));
    }
    
    public void RemoveItem(Guid productId)
    {
        EnsureCartIsActive();
        
        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new DomainException($"ไม่พบสินค้าใน cart: {productId}");
            
        _items.Remove(item);
        LastModifiedAt = DateTime.UtcNow;
        
        AddDomainEvent(new ItemRemovedFromCartEvent(Id, productId));
    }
    
    public void UpdateItemQuantity(Guid productId, int newQuantity)
    {
        EnsureCartIsActive();
        
        if (newQuantity <= 0)
        {
            RemoveItem(productId);
            return;
        }
        
        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new DomainException($"ไม่พบสินค้าใน cart: {productId}");
            
        item.SetQuantity(newQuantity);
        LastModifiedAt = DateTime.UtcNow;
    }
    
    public void Clear()
    {
        EnsureCartIsActive();
        _items.Clear();
        LastModifiedAt = DateTime.UtcNow;
    }
    
    public Money GetTotal() 
        => _items.Aggregate(Money.Zero(), (total, item) => total.Add(item.GetSubtotal()));
    
    public int GetItemCount() => _items.Sum(i => i.Quantity);
    
    public void Checkout()
    {
        EnsureCartIsActive();
        if (!_items.Any()) throw new DomainException("ไม่สามารถ checkout cart ที่ว่างได้");
        
        Status = CartStatus.CheckedOut;
        AddDomainEvent(new CartCheckedOutEvent(Id, CustomerId, GetTotal()));
    }
    
    private void EnsureCartIsActive()
    {
        if (Status != CartStatus.Active)
            throw new DomainException($"ไม่สามารถแก้ไข cart ที่มีสถานะ {Status}");
    }
    
    private void AddDomainEvent(IDomainEvent @event) => _domainEvents.Add(@event);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

// Child Entity ภายใน Aggregate
public class CartItem : Entity
{
    public Guid CartId { get; private set; }
    public Guid ProductId { get; private set; }
    public string ProductName { get; private set; }
    public Money UnitPrice { get; private set; }
    public int Quantity { get; private set; }
    
    protected CartItem() { }
    
    public static CartItem Create(
        Guid cartId, Guid productId, string productName, 
        Money unitPrice, int quantity)
    {
        if (quantity <= 0)
            throw new DomainException("จำนวนสินค้าต้องมากกว่า 0");
            
        return new CartItem
        {
            Id = Guid.NewGuid(),
            CartId = cartId,
            ProductId = productId,
            ProductName = productName,
            UnitPrice = unitPrice,
            Quantity = quantity
        };
    }
    
    public void IncreaseQuantity(int additional)
    {
        if (additional <= 0)
            throw new DomainException("จำนวนที่เพิ่มต้องมากกว่า 0");
        Quantity += additional;
    }
    
    public void SetQuantity(int quantity)
    {
        if (quantity <= 0)
            throw new DomainException("จำนวนสินค้าต้องมากกว่า 0");
        Quantity = quantity;
    }
    
    public Money GetSubtotal() => UnitPrice.Multiply(Quantity);
}

public enum CartStatus { Active, CheckedOut, Abandoned }
```

---

## Domain Events

Domain Events คือ events ที่เกิดขึ้นใน Domain และ Aggregates อื่นๆ หรือ services อื่นๆ สนใจ

```csharp
// Domain Events Interface
public interface IDomainEvent
{
    Guid EventId { get; }
    DateTime OccurredOn { get; }
}

public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

// Specific Domain Events
public record ItemAddedToCartEvent(
    Guid CartId, Guid ProductId, int Quantity) : DomainEvent;

public record ItemRemovedFromCartEvent(
    Guid CartId, Guid ProductId) : DomainEvent;

public record CartCheckedOutEvent(
    Guid CartId, Guid CustomerId, Money TotalAmount) : DomainEvent;

// Domain Event Dispatcher
public interface IDomainEventDispatcher
{
    Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct = default);
}

public class DomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IServiceProvider _serviceProvider;
    
    public DomainEventDispatcher(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct)
    {
        foreach (var @event in events)
        {
            var handlerType = typeof(IDomainEventHandler<>).MakeGenericType(@event.GetType());
            var handlers = _serviceProvider.GetServices(handlerType);
            
            foreach (var handler in handlers)
            {
                await (Task)handlerType
                    .GetMethod("HandleAsync")!
                    .Invoke(handler, new object[] { @event, ct })!;
            }
        }
    }
}

public interface IDomainEventHandler<TEvent> where TEvent : IDomainEvent
{
    Task HandleAsync(TEvent @event, CancellationToken ct = default);
}

// Event Handlers
public class CartCheckedOutEventHandler : IDomainEventHandler<CartCheckedOutEvent>
{
    private readonly ILogger<CartCheckedOutEventHandler> _logger;
    
    public CartCheckedOutEventHandler(ILogger<CartCheckedOutEventHandler> logger)
    {
        _logger = logger;
    }
    
    public async Task HandleAsync(CartCheckedOutEvent @event, CancellationToken ct)
    {
        _logger.LogInformation(
            "Cart {CartId} ถูก checkout โดยลูกค้า {CustomerId} ยอดรวม {Total}",
            @event.CartId, @event.CustomerId, @event.TotalAmount);
        
        // สามารถสร้าง Order จาก Cart ที่นี่ได้
        await Task.CompletedTask;
    }
}
```

---

## Repository (DDD Style)

Repository ใน DDD ทำหน้าที่เป็น in-memory collection ของ Aggregate Roots

```csharp
// Generic Repository Interface
public interface IRepository<T> where T : IAggregateRoot
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
}

// Specific Repository
public interface IShoppingCartRepository : IRepository<ShoppingCart>
{
    Task<ShoppingCart?> GetByCustomerIdAsync(Guid customerId, CancellationToken ct = default);
    Task<bool> ExistsAsync(Guid customerId, CancellationToken ct = default);
}

// EF Core Implementation
public class EfShoppingCartRepository : IShoppingCartRepository
{
    private readonly AppDbContext _context;
    private readonly IDomainEventDispatcher _eventDispatcher;
    
    public EfShoppingCartRepository(
        AppDbContext context, IDomainEventDispatcher eventDispatcher)
    {
        _context = context;
        _eventDispatcher = eventDispatcher;
    }
    
    public async Task<ShoppingCart?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _context.ShoppingCarts
            .Include(c => c.Items)
            .FirstOrDefaultAsync(c => c.Id == id, ct);
    
    public async Task<ShoppingCart?> GetByCustomerIdAsync(Guid customerId, CancellationToken ct)
        => await _context.ShoppingCarts
            .Include(c => c.Items)
            .FirstOrDefaultAsync(c => c.CustomerId == customerId 
                && c.Status == CartStatus.Active, ct);
    
    public async Task<bool> ExistsAsync(Guid customerId, CancellationToken ct)
        => await _context.ShoppingCarts
            .AnyAsync(c => c.CustomerId == customerId 
                && c.Status == CartStatus.Active, ct);
    
    public async Task AddAsync(ShoppingCart cart, CancellationToken ct)
    {
        await _context.ShoppingCarts.AddAsync(cart, ct);
        await SaveAndDispatchEventsAsync(cart, ct);
    }
    
    public async Task UpdateAsync(ShoppingCart cart, CancellationToken ct)
    {
        _context.ShoppingCarts.Update(cart);
        await SaveAndDispatchEventsAsync(cart, ct);
    }
    
    public async Task DeleteAsync(ShoppingCart cart, CancellationToken ct)
    {
        _context.ShoppingCarts.Remove(cart);
        await _context.SaveChangesAsync(ct);
    }
    
    private async Task SaveAndDispatchEventsAsync(
        ShoppingCart cart, CancellationToken ct)
    {
        var events = cart.DomainEvents.ToList();
        cart.ClearDomainEvents();
        
        await _context.SaveChangesAsync(ct);
        await _eventDispatcher.DispatchAsync(events, ct);
    }
}
```

---

## Domain Services

Domain Service ใช้เมื่อ logic ไม่เข้ากับ Entity หรือ Value Object ใดๆ เพราะต้องการหลาย Entities

```csharp
// Domain Service Interface
public interface IOrderPricingService
{
    Task<PricingResult> CalculatePriceAsync(
        ShoppingCart cart, 
        Guid? couponCode, 
        CancellationToken ct = default);
}

// Domain Service Implementation  
public class OrderPricingService : IOrderPricingService
{
    private readonly ICouponRepository _couponRepository;
    private readonly ITaxService _taxService;
    
    public OrderPricingService(
        ICouponRepository couponRepository, 
        ITaxService taxService)
    {
        _couponRepository = couponRepository;
        _taxService = taxService;
    }
    
    public async Task<PricingResult> CalculatePriceAsync(
        ShoppingCart cart, Guid? couponCode, CancellationToken ct)
    {
        if (!cart.Items.Any())
            throw new DomainException("ไม่สามารถคำนวณราคาสำหรับ cart ที่ว่างได้");
        
        var subtotal = cart.GetTotal();
        var discount = Money.Zero();
        
        // ใช้ Coupon ถ้ามี
        if (couponCode.HasValue)
        {
            var coupon = await _couponRepository.GetByIdAsync(couponCode.Value, ct);
            if (coupon is not null && coupon.IsValid(subtotal))
            {
                discount = coupon.CalculateDiscount(subtotal);
            }
        }
        
        var afterDiscount = subtotal.Subtract(discount);
        
        // คำนวณภาษี
        var tax = await _taxService.CalculateTaxAsync(afterDiscount, ct);
        var total = afterDiscount.Add(tax);
        
        return new PricingResult(
            Subtotal: subtotal,
            Discount: discount,
            Tax: tax,
            Total: total);
    }
}

public record PricingResult(
    Money Subtotal,
    Money Discount,
    Money Tax,
    Money Total);
```

---

## โปรแกรมตัวอย่าง: Shopping Cart Domain Model

```csharp
// ===== Complete Shopping Cart Example =====
using System;
using System.Collections.Generic;
using System.Linq;

// Value Objects
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();
    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        return GetEqualityComponents()
            .SequenceEqual(((ValueObject)obj).GetEqualityComponents());
    }
    public override int GetHashCode()
        => GetEqualityComponents()
            .Select(x => x?.GetHashCode() ?? 0)
            .Aggregate((x, y) => x ^ y);
}

public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }
    public Money(decimal amount, string currency = "THB")
    {
        Amount = Math.Round(amount, 2);
        Currency = currency;
    }
    public static Money Zero(string currency = "THB") => new(0, currency);
    public Money Add(Money other) => new(Amount + other.Amount, Currency);
    public Money Subtract(Money other) => new(Math.Max(0, Amount - other.Amount), Currency);
    public Money Multiply(int qty) => new(Amount * qty, Currency);
    protected override IEnumerable<object?> GetEqualityComponents() => new object[] { Amount, Currency };
    public override string ToString() => $"{Amount:N2} {Currency}";
}

// Entities
public abstract class Entity
{
    public Guid Id { get; protected set; }
}

// Domain Events
public interface IDomainEvent { DateTime OccurredOn { get; } }
public record ItemAddedEvent(Guid CartId, Guid ProductId, int Qty) : IDomainEvent
{ public DateTime OccurredOn { get; } = DateTime.UtcNow; }
public record CartCheckedOutEvent(Guid CartId, Money Total) : IDomainEvent
{ public DateTime OccurredOn { get; } = DateTime.UtcNow; }

// Aggregate
public enum CartStatus { Active, CheckedOut, Abandoned }

public class CartItem : Entity
{
    public Guid CartId { get; private set; }
    public Guid ProductId { get; private set; }
    public string ProductName { get; private set; }
    public Money UnitPrice { get; private set; }
    public int Quantity { get; private set; }
    
    protected CartItem() { }
    
    public static CartItem Create(Guid cartId, Guid productId, 
        string name, Money price, int qty)
    {
        return new CartItem 
        { 
            Id = Guid.NewGuid(), CartId = cartId,
            ProductId = productId, ProductName = name,
            UnitPrice = price, Quantity = qty 
        };
    }
    
    public void AddQuantity(int qty) => Quantity += qty;
    public void SetQuantity(int qty) => Quantity = qty;
    public Money GetSubtotal() => UnitPrice.Multiply(Quantity);
}

public class ShoppingCart : Entity
{
    private readonly List<CartItem> _items = new();
    private readonly List<IDomainEvent> _events = new();
    
    public Guid CustomerId { get; private set; }
    public CartStatus Status { get; private set; }
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    public IReadOnlyList<IDomainEvent> DomainEvents => _events.AsReadOnly();
    
    public static ShoppingCart Create(Guid customerId) => new()
    {
        Id = Guid.NewGuid(),
        CustomerId = customerId,
        Status = CartStatus.Active
    };
    
    public void AddItem(Guid productId, string name, decimal price, int qty)
    {
        if (Status != CartStatus.Active)
            throw new InvalidOperationException("Cart ไม่ Active");
        
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
            existing.AddQuantity(qty);
        else
            _items.Add(CartItem.Create(Id, productId, name, new Money(price), qty));
            
        _events.Add(new ItemAddedEvent(Id, productId, qty));
    }
    
    public void RemoveItem(Guid productId)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new InvalidOperationException("ไม่พบสินค้าใน cart");
        _items.Remove(item);
    }
    
    public void UpdateQuantity(Guid productId, int qty)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new InvalidOperationException("ไม่พบสินค้าใน cart");
        if (qty <= 0) _items.Remove(item);
        else item.SetQuantity(qty);
    }
    
    public Money GetTotal() => _items.Aggregate(Money.Zero(), (t, i) => t.Add(i.GetSubtotal()));
    
    public void Checkout()
    {
        if (!_items.Any()) throw new InvalidOperationException("Cart ว่าง");
        if (Status != CartStatus.Active) throw new InvalidOperationException("Cart ไม่ Active");
        Status = CartStatus.CheckedOut;
        _events.Add(new CartCheckedOutEvent(Id, GetTotal()));
    }
    
    public void ClearEvents() => _events.Clear();
}

// In-Memory Repository
public class InMemoryCartRepository
{
    private readonly Dictionary<Guid, ShoppingCart> _carts = new();
    
    public ShoppingCart? GetByCustomerId(Guid customerId)
        => _carts.Values.FirstOrDefault(c => c.CustomerId == customerId && c.Status == CartStatus.Active);
    
    public void Save(ShoppingCart cart)
    {
        _carts[cart.Id] = cart;
        foreach (var evt in cart.DomainEvents)
        {
            Console.WriteLine($"  [Event] {evt.GetType().Name} ที่ {evt.OccurredOn:HH:mm:ss}");
        }
        cart.ClearEvents();
    }
}

// Domain Service
public class CartCheckoutService
{
    private readonly InMemoryCartRepository _repo;
    
    public CartCheckoutService(InMemoryCartRepository repo)
    {
        _repo = repo;
    }
    
    public (bool success, string message, Money? total) Checkout(Guid customerId)
    {
        var cart = _repo.GetByCustomerId(customerId);
        if (cart is null) return (false, "ไม่พบ cart", null);
        
        try
        {
            cart.Checkout();
            _repo.Save(cart);
            return (true, "Checkout สำเร็จ", cart.GetTotal());
        }
        catch (Exception ex)
        {
            return (false, ex.Message, null);
        }
    }
}

// ===== Main Program =====
Console.WriteLine("=== Shopping Cart Domain Model Demo ===\n");

var repo = new InMemoryCartRepository();
var checkoutService = new CartCheckoutService(repo);
var customerId = Guid.NewGuid();

// สร้าง Cart
var cart = ShoppingCart.Create(customerId);
Console.WriteLine($"สร้าง Cart: {cart.Id}");

// เพิ่มสินค้า
Console.WriteLine("\n--- เพิ่มสินค้า ---");
cart.AddItem(Guid.NewGuid(), "iPhone 15 Pro", 45000m, 1);
repo.Save(cart);

cart.AddItem(Guid.NewGuid(), "AirPods Pro", 9000m, 2);
repo.Save(cart);

// เพิ่มสินค้าเดิมซ้ำ (ควรเพิ่มจำนวน)
var ipadId = Guid.NewGuid();
cart.AddItem(ipadId, "iPad Pro", 32000m, 1);
cart.AddItem(ipadId, "iPad Pro", 32000m, 1); // เพิ่มอีก 1 ชิ้น
repo.Save(cart);

// แสดงรายการ
Console.WriteLine("\n--- รายการใน Cart ---");
foreach (var item in cart.Items)
{
    Console.WriteLine($"  {item.ProductName}: {item.Quantity} x {item.UnitPrice} = {item.GetSubtotal()}");
}
Console.WriteLine($"  ยอดรวม: {cart.GetTotal()}");

// อัพเดทจำนวน
Console.WriteLine("\n--- อัพเดทจำนวน AirPods Pro เป็น 1 ---");
cart.UpdateQuantity(cart.Items[1].ProductId, 1);
Console.WriteLine($"  ยอดรวมใหม่: {cart.GetTotal()}");

// Checkout
Console.WriteLine("\n--- Checkout ---");
var (success, message, total) = checkoutService.Checkout(customerId);
Console.WriteLine($"  ผลลัพธ์: {message}");
if (success) Console.WriteLine($"  ยอดสุดท้าย: {total}");

// ลองเพิ่มสินค้าหลัง Checkout
Console.WriteLine("\n--- ลองเพิ่มสินค้าหลัง Checkout ---");
try
{
    cart.AddItem(Guid.NewGuid(), "Apple Watch", 15000m, 1);
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"  Error: {ex.Message}");
}
```

---

## Exercises

1. **Exercise 1**: สร้าง `Discount` Value Object ที่รองรับ:
   - Percentage discount (เช่น 10%)
   - Fixed amount discount (เช่น 100 บาท)

2. **Exercise 2**: เพิ่ม `MaxQuantityPerItem` rule ใน `ShoppingCart` ไม่เกิน 10 ชิ้นต่อสินค้า

3. **Exercise 3**: สร้าง `WishList` Aggregate ที่ user สามารถย้ายสินค้าจาก WishList ไป Cart

4. **Exercise 4**: Implement `CartAbandonmentService` Domain Service ที่ mark cart ที่ไม่มี activity มากกว่า 30 นาทีเป็น Abandoned

5. **Exercise 5**: เพิ่ม `CouponCode` Value Object พร้อม validation

---

## สรุป

DDD ช่วยให้เราสร้างระบบที่:
- **Bounded Context**: แบ่งระบบออกเป็นส่วนที่มีความหมายชัดเจน
- **Entities**: วัตถุที่มีตัวตนและ lifecycle
- **Value Objects**: วัตถุที่กำหนดโดยค่า ไม่มี identity
- **Aggregates**: กลุ่มของ entities ที่ทำงานเป็นหน่วย
- **Domain Events**: เหตุการณ์ที่เกิดขึ้นใน domain
- **Repository**: จัดการ persistence ของ aggregates
- **Domain Services**: logic ที่ไม่เข้ากับ entity ใดๆ

---

## Part ถัดไป

ใน Part 74 เราจะเรียนรู้เรื่อง **Unit Testing ด้วย xUnit** เพื่อทดสอบ domain model ที่เราสร้างขึ้น

---

*Part 73/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
