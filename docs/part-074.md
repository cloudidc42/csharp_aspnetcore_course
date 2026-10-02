# Part 74: Unit Testing ด้วย xUnit

## เนื้อหาใน Part นี้
- Unit test คืออะไร
- xUnit: [Fact], [Theory], [InlineData]
- Arrange-Act-Assert pattern
- Test naming conventions
- Code coverage
- โปรแกรมตัวอย่าง: Testing Calculator class

---

## Unit Test คืออะไร?

**Unit Test** คือการทดสอบหน่วยย่อยที่สุดของโค้ด (function, method, class) โดยแยกออกจากส่วนอื่นๆ

### ประโยชน์ของ Unit Testing

1. **ค้นพบ Bug เร็ว** - พบปัญหาตั้งแต่ขั้นตอน development
2. **เอกสารที่มีชีวิต** - test บอกว่าโค้ดควรทำอะไร
3. **Refactor อย่างมั่นใจ** - เปลี่ยนโค้ดได้โดยไม่กลัวเสีย
4. **Design ที่ดีขึ้น** - โค้ดที่ test ได้มักออกแบบได้ดีกว่า
5. **Regression Protection** - ป้องกัน bug ที่เคยแก้แล้วกลับมา

### Test Pyramid

```
        /\
       /  \
      / E2E \     (น้อยที่สุด, ช้า, แพง)
     /--------\
    / Integration\  (ปานกลาง)
   /--------------\
  /   Unit Tests   \  (มากที่สุด, เร็ว, ถูก)
 /------------------\
```

---

## ติดตั้ง xUnit

```bash
# สร้าง test project
dotnet new xunit -n MyProject.Tests

# เพิ่ม reference ไปยัง project ที่ต้องการ test
dotnet add reference ../MyProject/MyProject.csproj

# เพิ่ม packages
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package coverlet.collector
```

### โครงสร้าง Test Project

```
MyProject.Tests/
├── MyProject.Tests.csproj
├── Calculators/
│   ├── BasicCalculatorTests.cs
│   └── ScientificCalculatorTests.cs
├── Validators/
│   └── EmailValidatorTests.cs
└── Services/
    └── OrderServiceTests.cs
```

---

## [Fact] - Test แบบพื้นฐาน

`[Fact]` ใช้สำหรับ test ที่ไม่รับ parameter และมีผลลัพธ์เดียว

```csharp
using Xunit;

public class BasicCalculatorTests
{
    private readonly BasicCalculator _calculator;
    
    public BasicCalculatorTests()
    {
        // สร้าง instance ใหม่สำหรับทุก test
        _calculator = new BasicCalculator();
    }
    
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsCorrectSum()
    {
        // Arrange
        int a = 5;
        int b = 3;
        
        // Act
        int result = _calculator.Add(a, b);
        
        // Assert
        Assert.Equal(8, result);
    }
    
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        // Arrange & Act & Assert
        Assert.Throws<DivideByZeroException>(() => _calculator.Divide(10, 0));
    }
    
    [Fact]
    public void Sqrt_NegativeNumber_ThrowsArgumentException()
    {
        // Arrange
        double input = -4;
        
        // Act & Assert
        var exception = Assert.Throws<ArgumentException>(
            () => _calculator.Sqrt(input));
            
        Assert.Contains("ไม่สามารถหารากที่สองของเลขติดลบ", exception.Message);
    }
    
    [Fact]
    public void GetHistory_AfterMultipleOperations_ReturnsAllOperations()
    {
        // Arrange
        _calculator.Add(1, 2);
        _calculator.Multiply(3, 4);
        _calculator.Subtract(10, 5);
        
        // Act
        var history = _calculator.GetHistory();
        
        // Assert
        Assert.NotNull(history);
        Assert.Equal(3, history.Count);
    }
}
```

---

## [Theory] และ [InlineData]

`[Theory]` ใช้สำหรับ test ที่ต้องการทดสอบด้วยข้อมูลหลายชุด

```csharp
public class CalculatorTheoryTests
{
    private readonly BasicCalculator _calculator = new();
    
    // [InlineData] - ระบุข้อมูลโดยตรง
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    [InlineData(100, -50, 50)]
    [InlineData(int.MaxValue - 1, 1, int.MaxValue)]
    public void Add_VariousInputs_ReturnsCorrectResult(int a, int b, int expected)
    {
        // Act
        var result = _calculator.Add(a, b);
        
        // Assert
        Assert.Equal(expected, result);
    }
    
    [Theory]
    [InlineData(10, 2, 5)]
    [InlineData(15, 3, 5)]
    [InlineData(-10, 2, -5)]
    [InlineData(7, 2, 3.5)]
    public void Divide_ValidInputs_ReturnsCorrectResult(
        double dividend, double divisor, double expected)
    {
        var result = _calculator.Divide(dividend, divisor);
        Assert.Equal(expected, result, precision: 5);
    }
    
    // [MemberData] - ข้อมูลจาก property หรือ method
    [Theory]
    [MemberData(nameof(GetAddTestData))]
    public void Add_MemberData_ReturnsCorrectResult(int a, int b, int expected)
    {
        Assert.Equal(expected, _calculator.Add(a, b));
    }
    
    public static IEnumerable<object[]> GetAddTestData()
    {
        yield return new object[] { 1, 2, 3 };
        yield return new object[] { -5, 5, 0 };
        yield return new object[] { 100, 200, 300 };
        yield return new object[] { int.MinValue, 1, int.MinValue + 1 };
    }
    
    // [ClassData] - ข้อมูลจาก class แยก
    [Theory]
    [ClassData(typeof(MultiplicationTestData))]
    public void Multiply_ClassData_ReturnsCorrectResult(
        int a, int b, int expected)
    {
        Assert.Equal(expected, _calculator.Multiply(a, b));
    }
}

// Class สำหรับ test data
public class MultiplicationTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 2, 3, 6 };
        yield return new object[] { -4, 5, -20 };
        yield return new object[] { 0, 100, 0 };
        yield return new object[] { 7, 7, 49 };
    }
    
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

---

## Arrange-Act-Assert Pattern

AAA Pattern คือโครงสร้างมาตรฐานของ unit test

```csharp
public class ShoppingCartTests
{
    [Fact]
    public void AddItem_NewProduct_AddsToCart()
    {
        // Arrange - เตรียมข้อมูลและ objects ที่ต้องการ
        var cart = new ShoppingCart(Guid.NewGuid());
        var productId = Guid.NewGuid();
        var productName = "iPhone 15";
        var price = 45000m;
        var quantity = 1;
        
        // Act - เรียก method ที่ต้องการทดสอบ
        cart.AddItem(productId, productName, price, quantity);
        
        // Assert - ตรวจสอบผลลัพธ์
        Assert.Single(cart.Items);
        Assert.Equal(productId, cart.Items.First().ProductId);
        Assert.Equal(quantity, cart.Items.First().Quantity);
    }
    
    [Fact]
    public void AddItem_ExistingProduct_IncreasesQuantity()
    {
        // Arrange
        var cart = new ShoppingCart(Guid.NewGuid());
        var productId = Guid.NewGuid();
        cart.AddItem(productId, "iPhone", 45000m, 1);
        
        // Act - เพิ่มสินค้าเดิมอีกครั้ง
        cart.AddItem(productId, "iPhone", 45000m, 2);
        
        // Assert - ต้องมีแค่ 1 item แต่จำนวน = 3
        Assert.Single(cart.Items);
        Assert.Equal(3, cart.Items.First().Quantity);
    }
    
    [Fact]
    public void GetTotal_MultipleItems_ReturnCorrectTotal()
    {
        // Arrange
        var cart = new ShoppingCart(Guid.NewGuid());
        cart.AddItem(Guid.NewGuid(), "Product A", 100m, 2);  // 200
        cart.AddItem(Guid.NewGuid(), "Product B", 50m, 3);   // 150
        cart.AddItem(Guid.NewGuid(), "Product C", 75m, 1);   // 75
        
        // Act
        var total = cart.GetTotal();
        
        // Assert
        Assert.Equal(425m, total);
    }
    
    [Fact]
    public void Checkout_EmptyCart_ThrowsInvalidOperationException()
    {
        // Arrange
        var cart = new ShoppingCart(Guid.NewGuid());
        
        // Act & Assert - combined
        var exception = Assert.Throws<InvalidOperationException>(() => cart.Checkout());
        Assert.Equal("ไม่สามารถ checkout cart ที่ว่างได้", exception.Message);
    }
}
```

### Assert Methods ที่ใช้บ่อย

```csharp
public class AssertExamplesTests
{
    [Fact]
    public void AssertExamples()
    {
        // Equality
        Assert.Equal(5, 2 + 3);
        Assert.NotEqual(5, 2 + 4);
        
        // Null checks
        string? str = null;
        Assert.Null(str);
        str = "hello";
        Assert.NotNull(str);
        
        // Boolean
        Assert.True(5 > 3);
        Assert.False(3 > 5);
        
        // Collections
        var list = new List<int> { 1, 2, 3 };
        Assert.Contains(2, list);
        Assert.DoesNotContain(5, list);
        Assert.Equal(3, list.Count);
        Assert.Single(new List<int> { 42 });
        Assert.Empty(new List<int>());
        Assert.NotEmpty(list);
        
        // String
        Assert.StartsWith("hel", "hello");
        Assert.EndsWith("llo", "hello");
        Assert.Contains("ell", "hello");
        Assert.Matches(@"\d+", "123");
        
        // Type checking
        object obj = "text";
        Assert.IsType<string>(obj);
        Assert.IsAssignableFrom<IEnumerable<char>>(obj);
        
        // Exception
        Assert.Throws<ArgumentNullException>(() => 
            throw new ArgumentNullException("param"));
        
        // Floating point - ใช้ precision
        Assert.Equal(3.14, Math.PI, precision: 2);
    }
}
```

---

## Test Naming Conventions

การตั้งชื่อ test ที่ดีทำให้เข้าใจเจตนาของ test ได้ทันที

```csharp
// Convention: [MethodName]_[Scenario]_[ExpectedBehavior]
public class UserServiceTests
{
    // ✅ ชื่อดี - อ่านแล้วเข้าใจทันที
    [Fact]
    public void RegisterUser_ValidData_CreatesUserAndReturnsId()
    { /* ... */ }
    
    [Fact]
    public void RegisterUser_DuplicateEmail_ThrowsDuplicateEmailException()
    { /* ... */ }
    
    [Fact]
    public void RegisterUser_EmptyPassword_ThrowsArgumentException()
    { /* ... */ }
    
    [Fact]
    public void GetUserById_ExistingUser_ReturnsUserDetails()
    { /* ... */ }
    
    [Fact]
    public void GetUserById_NonExistentUser_ReturnsNull()
    { /* ... */ }
    
    [Fact]
    public void UpdateUserEmail_ValidNewEmail_UpdatesSuccessfully()
    { /* ... */ }
    
    [Fact]
    public void UpdateUserEmail_InvalidEmailFormat_ThrowsValidationException()
    { /* ... */ }
    
    // ❌ ชื่อไม่ดี - ไม่รู้ว่า test อะไร
    [Fact]
    public void Test1() { /* ... */ }
    
    [Fact]
    public void RegisterUserTest() { /* ... */ }
    
    [Fact]
    public void TestRegisterUser_DuplicateEmail() { /* ... */ }
}
```

---

## Code Coverage

Code Coverage วัดว่าโค้ดถูก test ไปแล้วกี่เปอร์เซ็นต์

```bash
# ติดตั้ง dotnet-coverage
dotnet tool install -g dotnet-coverage

# รัน test พร้อม coverage
dotnet test --collect:"XPlat Code Coverage"

# Generate report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator \
    -reports:"**/coverage.cobertura.xml" \
    -targetdir:"coveragereport" \
    -reporttypes:Html
```

### ตัวอย่าง Coverage-Driven Testing

```csharp
// Class ที่จะ test
public class PriceCalculator
{
    public decimal CalculateDiscount(decimal price, string customerType, int quantity)
    {
        decimal discount = 0;
        
        // Branch 1: Customer type discount
        if (customerType == "VIP")
            discount += price * 0.20m;
        else if (customerType == "Premium")
            discount += price * 0.10m;
        // else: no discount
        
        // Branch 2: Quantity discount
        if (quantity >= 100)
            discount += price * 0.15m;
        else if (quantity >= 50)
            discount += price * 0.10m;
        else if (quantity >= 10)
            discount += price * 0.05m;
        
        // Branch 3: Maximum discount cap
        if (discount > price * 0.30m)
            discount = price * 0.30m;
            
        return Math.Round(discount, 2);
    }
}

// Tests ที่ cover ทุก branch
public class PriceCalculatorTests
{
    private readonly PriceCalculator _calc = new();
    
    // Branch 1 tests
    [Fact]
    public void CalculateDiscount_VipCustomer_Applies20PercentDiscount()
    {
        var result = _calc.CalculateDiscount(1000m, "VIP", 1);
        Assert.Equal(200m, result);
    }
    
    [Fact]
    public void CalculateDiscount_PremiumCustomer_Applies10PercentDiscount()
    {
        var result = _calc.CalculateDiscount(1000m, "Premium", 1);
        Assert.Equal(100m, result);
    }
    
    [Fact]
    public void CalculateDiscount_RegularCustomer_NoTypeDiscount()
    {
        var result = _calc.CalculateDiscount(1000m, "Regular", 1);
        Assert.Equal(0m, result);
    }
    
    // Branch 2 tests
    [Theory]
    [InlineData(100, 0.15)]
    [InlineData(150, 0.15)]
    [InlineData(50, 0.10)]
    [InlineData(75, 0.10)]
    [InlineData(10, 0.05)]
    [InlineData(25, 0.05)]
    [InlineData(9, 0)]
    [InlineData(1, 0)]
    public void CalculateDiscount_QuantityTiers_AppliesCorrectDiscount(
        int quantity, decimal expectedDiscountRate)
    {
        var price = 1000m;
        var result = _calc.CalculateDiscount(price, "Regular", quantity);
        Assert.Equal(price * expectedDiscountRate, result);
    }
    
    // Branch 3: Max cap test
    [Fact]
    public void CalculateDiscount_VipWith100Items_CapsAt30Percent()
    {
        // VIP (20%) + Quantity>=100 (15%) = 35% -> capped at 30%
        var result = _calc.CalculateDiscount(1000m, "VIP", 100);
        Assert.Equal(300m, result);
    }
}
```

---

## โปรแกรมตัวอย่าง: Testing Calculator Class

```csharp
// ===== Calculator Class =====
namespace Calculator;

public class CalculationResult
{
    public double Value { get; set; }
    public string Operation { get; set; } = "";
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    
    public override string ToString() => $"{Operation} = {Value}";
}

public class ScientificCalculator
{
    private readonly List<CalculationResult> _history = new();
    
    public double Add(double a, double b)
    {
        var result = a + b;
        RecordHistory($"{a} + {b}", result);
        return result;
    }
    
    public double Subtract(double a, double b)
    {
        var result = a - b;
        RecordHistory($"{a} - {b}", result);
        return result;
    }
    
    public double Multiply(double a, double b)
    {
        var result = a * b;
        RecordHistory($"{a} * {b}", result);
        return result;
    }
    
    public double Divide(double a, double b)
    {
        if (b == 0) throw new DivideByZeroException("ไม่สามารถหารด้วย 0");
        var result = a / b;
        RecordHistory($"{a} / {b}", result);
        return result;
    }
    
    public double Sqrt(double a)
    {
        if (a < 0) throw new ArgumentException("ไม่สามารถหารากที่สองของเลขติดลบ", nameof(a));
        var result = Math.Sqrt(a);
        RecordHistory($"√{a}", result);
        return result;
    }
    
    public double Power(double @base, double exponent)
    {
        var result = Math.Pow(@base, exponent);
        RecordHistory($"{@base}^{exponent}", result);
        return result;
    }
    
    public double Factorial(int n)
    {
        if (n < 0) throw new ArgumentException("Factorial ของเลขติดลบไม่มีความหมาย", nameof(n));
        if (n > 20) throw new ArgumentException("ค่าที่ใส่ใหญ่เกินไป (max: 20)", nameof(n));
        
        double result = 1;
        for (int i = 2; i <= n; i++) result *= i;
        
        RecordHistory($"{n}!", result);
        return result;
    }
    
    public double Percentage(double value, double percentage)
    {
        var result = value * percentage / 100;
        RecordHistory($"{percentage}% of {value}", result);
        return result;
    }
    
    public IReadOnlyList<CalculationResult> GetHistory() => _history.AsReadOnly();
    public void ClearHistory() => _history.Clear();
    
    private void RecordHistory(string operation, double result)
        => _history.Add(new CalculationResult { Operation = operation, Value = result });
}

// ===== Tests =====
namespace Calculator.Tests;

public class ScientificCalculatorTests : IDisposable
{
    private readonly ScientificCalculator _calc;
    
    public ScientificCalculatorTests()
    {
        _calc = new ScientificCalculator();
    }
    
    public void Dispose()
    {
        _calc.ClearHistory();
    }
    
    // Addition Tests
    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        Assert.Equal(8, _calc.Add(5, 3));
    }
    
    [Fact]
    public void Add_NegativeNumbers_ReturnsCorrectSum()
    {
        Assert.Equal(-2, _calc.Add(-5, 3));
    }
    
    [Theory]
    [InlineData(0, 0, 0)]
    [InlineData(1.5, 2.5, 4.0)]
    [InlineData(-3, -4, -7)]
    [InlineData(double.MaxValue / 2, double.MaxValue / 2, double.MaxValue)]
    public void Add_VariousInputs_ReturnsCorrectResult(double a, double b, double expected)
    {
        Assert.Equal(expected, _calc.Add(a, b), precision: 10);
    }
    
    // Division Tests
    [Fact]
    public void Divide_ValidNumbers_ReturnsQuotient()
    {
        Assert.Equal(2.5, _calc.Divide(5, 2));
    }
    
    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        var ex = Assert.Throws<DivideByZeroException>(() => _calc.Divide(10, 0));
        Assert.Equal("ไม่สามารถหารด้วย 0", ex.Message);
    }
    
    // Square Root Tests
    [Theory]
    [InlineData(4, 2)]
    [InlineData(9, 3)]
    [InlineData(0, 0)]
    [InlineData(2, 1.4142135623)]
    public void Sqrt_ValidInput_ReturnsCorrectRoot(double input, double expected)
    {
        var result = _calc.Sqrt(input);
        Assert.Equal(expected, result, precision: 5);
    }
    
    [Fact]
    public void Sqrt_NegativeNumber_ThrowsArgumentException()
    {
        var ex = Assert.Throws<ArgumentException>(() => _calc.Sqrt(-1));
        Assert.Equal("a", ex.ParamName);
        Assert.Contains("ไม่สามารถหารากที่สองของเลขติดลบ", ex.Message);
    }
    
    // Factorial Tests
    [Theory]
    [InlineData(0, 1)]
    [InlineData(1, 1)]
    [InlineData(5, 120)]
    [InlineData(10, 3628800)]
    public void Factorial_ValidInput_ReturnsCorrectResult(int n, double expected)
    {
        Assert.Equal(expected, _calc.Factorial(n));
    }
    
    [Fact]
    public void Factorial_NegativeNumber_ThrowsArgumentException()
    {
        Assert.Throws<ArgumentException>(() => _calc.Factorial(-1));
    }
    
    [Fact]
    public void Factorial_TooLargeNumber_ThrowsArgumentException()
    {
        Assert.Throws<ArgumentException>(() => _calc.Factorial(21));
    }
    
    // History Tests
    [Fact]
    public void GetHistory_AfterOperations_ReturnsAllOperations()
    {
        _calc.Add(1, 2);
        _calc.Multiply(3, 4);
        _calc.Divide(10, 2);
        
        var history = _calc.GetHistory();
        
        Assert.Equal(3, history.Count);
    }
    
    [Fact]
    public void GetHistory_ContainsCorrectOperations()
    {
        _calc.Add(5, 3);
        _calc.Subtract(10, 4);
        
        var history = _calc.GetHistory();
        
        Assert.Collection(history,
            item => Assert.Equal(8, item.Value),
            item => Assert.Equal(6, item.Value));
    }
    
    [Fact]
    public void ClearHistory_AfterOperations_EmptiesHistory()
    {
        _calc.Add(1, 2);
        _calc.Add(3, 4);
        
        _calc.ClearHistory();
        
        Assert.Empty(_calc.GetHistory());
    }
    
    // Percentage Tests
    [Theory]
    [InlineData(1000, 10, 100)]
    [InlineData(500, 20, 100)]
    [InlineData(1000, 0, 0)]
    [InlineData(1000, 100, 1000)]
    public void Percentage_ValidInputs_ReturnsCorrectAmount(
        double value, double pct, double expected)
    {
        Assert.Equal(expected, _calc.Percentage(value, pct));
    }
}

// ===== Test Fixtures สำหรับ shared setup =====
public class CalculatorFixture : IDisposable
{
    public ScientificCalculator Calculator { get; } = new ScientificCalculator();
    
    public void Dispose() { }
}

// IClassFixture - share instance ระหว่าง tests ใน class เดียวกัน
public class CalculatorWithFixtureTests : IClassFixture<CalculatorFixture>
{
    private readonly ScientificCalculator _calc;
    
    public CalculatorWithFixtureTests(CalculatorFixture fixture)
    {
        _calc = fixture.Calculator;
    }
    
    [Fact]
    public void Power_PositiveExponent_ReturnsCorrectResult()
    {
        Assert.Equal(8, _calc.Power(2, 3));
    }
    
    [Fact]
    public void Power_ZeroExponent_ReturnsOne()
    {
        Assert.Equal(1, _calc.Power(5, 0));
    }
    
    [Fact]
    public void Power_NegativeExponent_ReturnsDecimal()
    {
        Assert.Equal(0.25, _calc.Power(2, -2));
    }
}

// ===== Main Program =====
using Calculator;

Console.WriteLine("=== Scientific Calculator Demo ===\n");

var calc = new ScientificCalculator();

Console.WriteLine("การคำนวณพื้นฐาน:");
Console.WriteLine($"  5 + 3 = {calc.Add(5, 3)}");
Console.WriteLine($"  10 - 4 = {calc.Subtract(10, 4)}");
Console.WriteLine($"  6 * 7 = {calc.Multiply(6, 7)}");
Console.WriteLine($"  15 / 4 = {calc.Divide(15, 4)}");

Console.WriteLine("\nการคำนวณขั้นสูง:");
Console.WriteLine($"  √16 = {calc.Sqrt(16)}");
Console.WriteLine($"  2^10 = {calc.Power(2, 10)}");
Console.WriteLine($"  5! = {calc.Factorial(5)}");
Console.WriteLine($"  15% of 1000 = {calc.Percentage(1000, 15)}");

Console.WriteLine("\nประวัติการคำนวณ:");
foreach (var record in calc.GetHistory())
{
    Console.WriteLine($"  {record.Timestamp:HH:mm:ss} | {record}");
}

Console.WriteLine("\nทดสอบ Exception:");
try
{
    calc.Divide(5, 0);
}
catch (DivideByZeroException ex)
{
    Console.WriteLine($"  DivideByZero: {ex.Message}");
}

try
{
    calc.Sqrt(-4);
}
catch (ArgumentException ex)
{
    Console.WriteLine($"  ArgumentException: {ex.Message}");
}
```

---

## Exercises

1. **Exercise 1**: เขียน test สำหรับ `StringCalculator` class:
   - `Add("")` → 0
   - `Add("1")` → 1
   - `Add("1,2")` → 3
   - `Add("1\n2,3")` → 6

2. **Exercise 2**: สร้าง `BankAccount` class และเขียน tests:
   - Deposit เงิน
   - Withdraw เงิน (ถ้าเงินไม่พอต้อง throw exception)
   - GetBalance
   - Transfer ระหว่าง account

3. **Exercise 3**: เขียน test สำหรับ `EmailValidator`:
   - Valid emails: user@example.com, user.name@domain.co.th
   - Invalid: no @, no domain, empty string

4. **Exercise 4**: ใช้ `IDisposable` / `IAsyncLifetime` สำหรับ cleanup ใน tests

5. **Exercise 5**: เรียนรู้ใช้ `Assert.Collection` สำหรับ test collections

---

## สรุป

Unit Testing ด้วย xUnit มีหลักการสำคัญ:
- **[Fact]**: test เดียว ไม่มี parameters
- **[Theory] + [InlineData/MemberData/ClassData]**: test หลาย scenarios
- **AAA Pattern**: Arrange, Act, Assert
- **ตั้งชื่อดี**: `MethodName_Scenario_ExpectedBehavior`
- **Code Coverage**: วัดว่า test ครอบคลุมโค้ดแค่ไหน
- **IDisposable**: cleanup หลัง test แต่ละ test
- **IClassFixture**: share setup ระหว่าง tests

---

## Part ถัดไป

ใน Part 75 เราจะเรียนรู้เรื่อง **Mocking ด้วย Moq** เพื่อ test code ที่มี dependencies เช่น database, email service

---

*Part 74/700 | Phase 5: ระดับมืออาชีพ | หลักสูตร C# และ ASP.NET Core*
