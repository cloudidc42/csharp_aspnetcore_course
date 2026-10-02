# Part 016: Polymorphism

## เนื้อหาใน Part นี้
- virtual และ override
- abstract methods และ abstract class
- Upcasting และ Downcasting
- is และ as keywords
- Pattern matching กับ type
- Method hiding ด้วย new keyword
- โปรแกรมตัวอย่าง: Shape hierarchy

---

## 1. Polymorphism คืออะไร?

Polymorphism (พหุสัณฐาน) หมายถึงความสามารถของ object ในการมีรูปแบบหลายแบบ หรือพูดง่ายๆ คือ "ชื่อเดียว หลายพฤติกรรม"

**2 ประเภทหลัก:**
- **Compile-time (Static)** - Method overloading, Operator overloading
- **Runtime (Dynamic)** - Method overriding ผ่าน `virtual` + `override`

```csharp
// ตัวอย่างง่ายของ Polymorphism
public class Animal
{
    public virtual void Speak()
    {
        Console.WriteLine("...");
    }
}

public class Dog : Animal
{
    public override void Speak() => Console.WriteLine("โฮ่ง!");
}

public class Cat : Animal
{
    public override void Speak() => Console.WriteLine("เมี๊ยว~");
}

public class Duck : Animal
{
    public override void Speak() => Console.WriteLine("กวั้กๆ");
}

// Polymorphism ทำงาน - ตัวแปรประเภท Animal แต่ชี้ไปที่ object ต่างๆ
Animal[] animals = [new Dog(), new Cat(), new Duck(), new Animal()];

foreach (Animal animal in animals)
{
    animal.Speak(); // แต่ละตัวส่งเสียงต่างกัน
}
// โฮ่ง!
// เมี๊ยว~
// กวั้กๆ
// ...
```

---

## 2. virtual และ override

```csharp
public class BankAccount
{
    protected decimal _balance;
    public string AccountNumber { get; }
    public string OwnerName { get; }

    public BankAccount(string accountNumber, string ownerName, decimal initialBalance)
    {
        AccountNumber = accountNumber;
        OwnerName = ownerName;
        _balance = initialBalance;
    }

    public decimal Balance => _balance;

    // virtual - สามารถ override ได้ใน derived classes
    public virtual decimal CalculateMonthlyFee()
    {
        return 50m; // ค่าธรรมเนียมพื้นฐาน
    }

    public virtual bool CanWithdraw(decimal amount)
    {
        return _balance >= amount;
    }

    public virtual void Withdraw(decimal amount)
    {
        if (!CanWithdraw(amount))
            throw new InvalidOperationException("ยอดเงินไม่เพียงพอ");

        _balance -= amount;
        Console.WriteLine($"[{AccountNumber}] ถอน {amount:N0} บาท คงเหลือ {_balance:N0} บาท");
    }

    public virtual void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("จำนวนต้องมากกว่า 0");
        _balance += amount;
        Console.WriteLine($"[{AccountNumber}] ฝาก {amount:N0} บาท คงเหลือ {_balance:N0} บาท");
    }

    public virtual string GetAccountInfo()
    {
        return $"[{AccountNumber}] {OwnerName}: {_balance:N0} บาท ({GetType().Name})";
    }
}

// บัญชีออมทรัพย์
public class SavingsAccount : BankAccount
{
    public double AnnualInterestRate { get; }
    private decimal _minimumBalance;

    public SavingsAccount(string accountNumber, string ownerName,
        decimal initialBalance, double interestRate = 0.015, decimal minBalance = 500)
        : base(accountNumber, ownerName, initialBalance)
    {
        AnnualInterestRate = interestRate;
        _minimumBalance = minBalance;
    }

    // override - เปลี่ยน logic การถอน
    public override bool CanWithdraw(decimal amount)
    {
        return _balance - amount >= _minimumBalance;
    }

    // override - ค่าธรรมเนียมต่างกัน
    public override decimal CalculateMonthlyFee()
    {
        return _balance >= 2000 ? 0m : 25m; // ไม่มีค่าธรรมเนียมถ้ามียอดพอ
    }

    public void AddInterest()
    {
        decimal interest = _balance * (decimal)(AnnualInterestRate / 12);
        _balance += interest;
        Console.WriteLine($"[{AccountNumber}] ดอกเบี้ย +{interest:N2} บาท คงเหลือ {_balance:N2} บาท");
    }

    public override string GetAccountInfo()
    {
        return $"{base.GetAccountInfo()} [ดอกเบี้ย {AnnualInterestRate:P2}/ปี]";
    }
}

// บัญชีกระแสรายวัน (overdraft ได้)
public class CheckingAccount : BankAccount
{
    public decimal OverdraftLimit { get; }

    public CheckingAccount(string accountNumber, string ownerName,
        decimal initialBalance, decimal overdraftLimit = 5000)
        : base(accountNumber, ownerName, initialBalance)
    {
        OverdraftLimit = overdraftLimit;
    }

    // override - อนุญาต overdraft
    public override bool CanWithdraw(decimal amount)
    {
        return _balance + OverdraftLimit >= amount;
    }

    public override decimal CalculateMonthlyFee()
    {
        return 150m; // ค่าธรรมเนียมคงที่
    }

    public override string GetAccountInfo()
    {
        return $"{base.GetAccountInfo()} [Overdraft: {OverdraftLimit:N0} บาท]";
    }
}

// Polymorphism ในการใช้งาน
BankAccount[] accounts =
[
    new BankAccount("001", "พื้นฐาน", 1000),
    new SavingsAccount("002", "ออมทรัพย์", 5000, 0.02),
    new CheckingAccount("003", "กระแสรายวัน", 2000, 10000),
];

foreach (BankAccount account in accounts)
{
    Console.WriteLine(account.GetAccountInfo());
    Console.WriteLine($"  ค่าธรรมเนียม: {account.CalculateMonthlyFee():N0} บาท/เดือน");
}
```

---

## 3. Abstract Classes และ Abstract Methods

`abstract` บังคับให้ derived classes ต้อง implement method

```csharp
// Abstract class - ไม่สามารถ new ได้โดยตรง
public abstract class Shape
{
    public string Color { get; set; }
    public string Name { get; }

    protected Shape(string name, string color = "Black")
    {
        Name = name;
        Color = color;
    }

    // Abstract methods - ต้อง implement ใน derived classes
    public abstract double GetArea();
    public abstract double GetPerimeter();

    // Virtual method - มี default implementation แต่ override ได้
    public virtual void Draw()
    {
        Console.WriteLine($"Drawing {Name} [{Color}] " +
            $"(Area: {GetArea():F2}, Perimeter: {GetPerimeter():F2})");
    }

    // Regular method - ใช้ abstract methods
    public bool IsLargerThan(Shape other)
    {
        return GetArea() > other.GetArea();
    }

    public double GetAreaToPerimeterRatio()
    {
        double perimeter = GetPerimeter();
        return perimeter == 0 ? 0 : GetArea() / perimeter;
    }

    public override string ToString()
        => $"{Name}({Color}) Area={GetArea():F2} Perimeter={GetPerimeter():F2}";
}

public class Circle : Shape
{
    public double Radius { get; }

    public Circle(double radius, string color = "Red")
        : base("Circle", color)
    {
        Radius = radius > 0 ? radius
            : throw new ArgumentOutOfRangeException(nameof(radius));
    }

    public override double GetArea() => Math.PI * Radius * Radius;
    public override double GetPerimeter() => 2 * Math.PI * Radius;

    public override void Draw()
    {
        base.Draw();
        Console.WriteLine($"  Radius: {Radius}");
    }
}

public class Rectangle : Shape
{
    public double Width { get; }
    public double Height { get; }

    public Rectangle(double width, double height, string color = "Blue")
        : base("Rectangle", color)
    {
        Width = width;
        Height = height;
    }

    public override double GetArea() => Width * Height;
    public override double GetPerimeter() => 2 * (Width + Height);

    public bool IsSquare => Math.Abs(Width - Height) < 0.0001;
}

public class Triangle : Shape
{
    public double SideA { get; }
    public double SideB { get; }
    public double SideC { get; }

    public Triangle(double a, double b, double c, string color = "Green")
        : base("Triangle", color)
    {
        // ตรวจสอบความถูกต้องของสามเหลี่ยม
        if (a + b <= c || a + c <= b || b + c <= a)
            throw new ArgumentException("ด้านทั้งสามไม่สามารถสร้างสามเหลี่ยมได้");
        SideA = a; SideB = b; SideC = c;
    }

    public override double GetPerimeter() => SideA + SideB + SideC;

    // Heron's formula
    public override double GetArea()
    {
        double s = GetPerimeter() / 2;
        return Math.Sqrt(s * (s - SideA) * (s - SideB) * (s - SideC));
    }

    public string GetTriangleType()
    {
        bool isEquilateral = Math.Abs(SideA - SideB) < 0.001 && Math.Abs(SideB - SideC) < 0.001;
        bool isIsosceles = Math.Abs(SideA - SideB) < 0.001 ||
                           Math.Abs(SideB - SideC) < 0.001 ||
                           Math.Abs(SideA - SideC) < 0.001;

        return isEquilateral ? "Equilateral" : isIsosceles ? "Isosceles" : "Scalene";
    }
}

// การใช้งาน abstract class
var shapes = new List<Shape>
{
    new Circle(5, "Red"),
    new Rectangle(4, 6, "Blue"),
    new Triangle(3, 4, 5, "Green"),
    new Circle(3),
    new Rectangle(5, 5)
};

// Polymorphism - เรียก method เดียวกัน result ต่างกัน
foreach (Shape shape in shapes)
{
    shape.Draw();
}

// หาพื้นที่รวม
double totalArea = shapes.Sum(s => s.GetArea());
Console.WriteLine($"\nพื้นที่รวม: {totalArea:F2}");

// หา shape ที่ใหญ่ที่สุด
Shape largest = shapes.OrderByDescending(s => s.GetArea()).First();
Console.WriteLine($"Shape ใหญ่สุด: {largest}");
```

---

## 4. Upcasting และ Downcasting

**Upcasting** - แปลง derived type → base type (implicit, ปลอดภัย)  
**Downcasting** - แปลง base type → derived type (explicit, อาจ fail)

```csharp
// Upcasting - Implicit (ปลอดภัย)
Dog myDog = new Dog("Rex", 3);
Animal animal = myDog;  // Upcasting - Dog เป็น Animal ด้วย
object obj = myDog;     // ทุก class เป็น object

// เมื่อ upcast แล้ว เข้าถึงได้เฉพาะ members ของ base type
animal.MakeSound();     // OK - method ของ Animal
// animal.Fetch();      // Error! - Fetch เป็นของ Dog เท่านั้น

// Downcasting - Explicit cast (อาจ throw InvalidCastException)
Animal animal2 = new Dog("Buddy", 2);

// วิธีที่ 1: Direct cast - throw exception ถ้า fail
Dog dog1 = (Dog)animal2;  // OK - animal2 เป็น Dog จริงๆ
dog1.Fetch();

// วิธีที่ 2: as operator - return null ถ้า fail
Dog? dog2 = animal2 as Dog;
if (dog2 != null)
{
    dog2.Fetch();
    Console.WriteLine("Cast สำเร็จ");
}

Cat? cat = animal2 as Cat; // null - animal2 ไม่ใช่ Cat
if (cat == null)
{
    Console.WriteLine("animal2 ไม่ใช่ Cat");
}

// วิธีที่ 3: is operator - ตรวจสอบก่อน cast
if (animal2 is Dog d)  // Pattern matching - C# 7+
{
    d.Fetch();
    Console.WriteLine($"{d.Name} fetch!");
}

// สาธิต Upcasting ใน collection
List<Animal> zoo = new List<Animal>
{
    new Dog("Rex", 3),
    new Cat("Whiskers", 2),
    new Dog("Buddy", 1),
    new Animal("Generic", 5)
};

// Polymorphism ทำงาน
foreach (Animal a in zoo)
{
    a.MakeSound(); // เรียก method ของแต่ละ type
}

// กรองเฉพาะ Dog โดยใช้ is
var dogs = zoo.Where(a => a is Dog).Cast<Dog>().ToList();
Console.WriteLine($"\nพบสุนัข {dogs.Count} ตัว:");
foreach (Dog dog in dogs)
{
    Console.WriteLine($"  {dog.Name}");
    dog.Fetch();
}
```

---

## 5. is Keyword และ Pattern Matching

```csharp
public abstract class Vehicle
{
    public string Brand { get; set; }
    public Vehicle(string brand) { Brand = brand; }
    public abstract int GetPassengerCapacity();
}

public class Car : Vehicle
{
    public int Doors { get; set; }
    public Car(string brand, int doors) : base(brand) { Doors = doors; }
    public override int GetPassengerCapacity() => 5;
}

public class Bus : Vehicle
{
    public int SeatCount { get; set; }
    public Bus(string brand, int seats) : base(brand) { SeatCount = seats; }
    public override int GetPassengerCapacity() => SeatCount;
}

public class Truck : Vehicle
{
    public double MaxLoadTons { get; set; }
    public Truck(string brand, double load) : base(brand) { MaxLoadTons = load; }
    public override int GetPassengerCapacity() => 2;
}

// Pattern matching
void ProcessVehicle(Vehicle vehicle)
{
    // is with pattern variable
    if (vehicle is Car car)
    {
        Console.WriteLine($"Car: {car.Brand} ({car.Doors} ประตู)");
    }
    else if (vehicle is Bus bus)
    {
        Console.WriteLine($"Bus: {bus.Brand} ({bus.SeatCount} ที่นั่ง)");
    }
    else if (vehicle is Truck truck when truck.MaxLoadTons > 5)
    {
        Console.WriteLine($"Heavy Truck: {truck.Brand} (รับ {truck.MaxLoadTons} ตัน)");
    }
    else if (vehicle is Truck lightTruck)
    {
        Console.WriteLine($"Light Truck: {lightTruck.Brand} (รับ {lightTruck.MaxLoadTons} ตัน)");
    }

    // Switch expression (C# 8+)
    string category = vehicle switch
    {
        Car c when c.Doors >= 4 => "Family Car",
        Car c => "Sports Car",
        Bus b when b.SeatCount > 40 => "Large Bus",
        Bus => "Minibus",
        Truck t when t.MaxLoadTons > 10 => "Heavy Truck",
        Truck => "Light Truck",
        _ => "Unknown Vehicle"
    };
    Console.WriteLine($"  Category: {category}");
    Console.WriteLine($"  ความจุ: {vehicle.GetPassengerCapacity()} คน");
}

// การใช้งาน
var vehicles = new List<Vehicle>
{
    new Car("Toyota", 4),
    new Car("Ferrari", 2),
    new Bus("Mercedes", 50),
    new Bus("Toyota", 25),
    new Truck("Hino", 15),
    new Truck("Isuzu", 3)
};

foreach (var v in vehicles)
{
    ProcessVehicle(v);
    Console.WriteLine();
}

// Null pattern
Vehicle? nullVehicle = null;
if (nullVehicle is null)
    Console.WriteLine("nullVehicle เป็น null");
if (nullVehicle is not null)
    Console.WriteLine("nullVehicle ไม่ใช่ null");
```

---

## 6. as Keyword

```csharp
public interface IFlyable
{
    void Fly();
}

public interface ISwimmable
{
    void Swim();
}

public class Bird : Animal
{
    public Bird(string name) : base(name, 2) { }
    public override void MakeSound() => Console.WriteLine($"{Name}: Tweet!");
}

public class Eagle : Bird, IFlyable
{
    public Eagle(string name) : base(name) { }
    public void Fly() => Console.WriteLine($"{Name} บินสูง");
}

public class Penguin : Bird, ISwimmable
{
    public Penguin(string name) : base(name) { }
    public void Swim() => Console.WriteLine($"{Name} ว่ายน้ำ");
}

public class Duck : Bird, IFlyable, ISwimmable
{
    public Duck(string name) : base(name) { }
    public void Fly() => Console.WriteLine($"{Name} บินต่ำ");
    public void Swim() => Console.WriteLine($"{Name} ว่ายน้ำในบึง");
}

// ใช้ as เพื่อ cast ไปยัง interface
void TryActivities(Animal animal)
{
    Console.WriteLine($"\n{animal.Name}:");
    animal.MakeSound();

    // as ตรวจสอบ interface
    if (animal is IFlyable flyer)
    {
        flyer.Fly();
    }

    if (animal is ISwimmable swimmer)
    {
        swimmer.Swim();
    }

    // as operator
    IFlyable? flyable = animal as IFlyable;
    ISwimmable? swimmable = animal as ISwimmable;

    Console.WriteLine($"  บินได้: {flyable != null}, ว่ายน้ำได้: {swimmable != null}");
}

var animals2 = new List<Animal>
{
    new Eagle("อินทรีย์"),
    new Penguin("เพนกวิน"),
    new Duck("เป็ด")
};

foreach (var a in animals2)
{
    TryActivities(a);
}
```

---

## 7. Method Hiding ด้วย new Keyword

```csharp
// new keyword ใน method = Method Hiding (ไม่แนะนำ)
public class Parent
{
    public void Print() => Console.WriteLine("Parent.Print()");
    public virtual void VirtualPrint() => Console.WriteLine("Parent.VirtualPrint()");
}

public class Child : Parent
{
    // Method Hiding - ซ่อน Parent.Print()
    public new void Print() => Console.WriteLine("Child.Print()");

    // Override - แทนที่ Parent.VirtualPrint()
    public override void VirtualPrint() => Console.WriteLine("Child.VirtualPrint()");
}

var child = new Child();
child.Print();         // "Child.Print()" - ตามที่คาด
child.VirtualPrint();  // "Child.VirtualPrint()" - ตามที่คาด

// ผ่าน base type reference
Parent parent = child;
parent.Print();         // "Parent.Print()" ⚠️ - ไม่เรียก Child! (Hiding)
parent.VirtualPrint();  // "Child.VirtualPrint()" ✅ (Override ทำงาน)

Console.WriteLine("\n=== Override vs Hide Summary ===");
Console.WriteLine("child.Print():         " + "Child (ถูกต้อง)");
Console.WriteLine("parent.Print():        " + "Parent (⚠️ hiding ไม่ทำงานผ่าน base ref)");
Console.WriteLine("child.VirtualPrint():  " + "Child (ถูกต้อง)");
Console.WriteLine("parent.VirtualPrint(): " + "Child (✅ override ทำงานเสมอ)");
```

---

## 8. Covariance และ Contravariance

```csharp
// Covariance - IEnumerable<Derived> สามารถใช้แทน IEnumerable<Base>
List<Dog> dogs = new List<Dog> { new Dog("Rex", 2), new Dog("Buddy", 3) };
IEnumerable<Animal> animals = dogs; // Covariance ทำงานกับ IEnumerable<T>

foreach (Animal a in animals)
{
    a.MakeSound();
}

// Arrays ก็รองรับ covariance
Dog[] dogArray = [new Dog("Max", 1)];
Animal[] animalArray = dogArray; // OK
// animalArray[0] = new Cat("Kitty", 2); // Runtime error! ArrayTypeMismatchException

// Generic covariance ด้วย out
public interface IProducer<out T>
{
    T Produce();
}

public class DogFactory : IProducer<Dog>
{
    public Dog Produce() => new Dog("NewDog", 0);
}

IProducer<Animal> producer = new DogFactory(); // Covariance
Animal producedAnimal = producer.Produce();

// Generic contravariance ด้วย in
public interface IConsumer<in T>
{
    void Consume(T item);
}

public class AnimalHandler : IConsumer<Animal>
{
    public void Consume(Animal animal)
    {
        Console.WriteLine($"Handling {animal.Name}");
        animal.MakeSound();
    }
}

IConsumer<Dog> dogHandler = new AnimalHandler(); // Contravariance
dogHandler.Consume(new Dog("Fido", 2));
```

---

## 9. โปรแกรมตัวอย่าง: Shape Hierarchy

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace ShapeHierarchyExample
{
    /// <summary>
    /// Base class สำหรับทุก Shape
    /// </summary>
    public abstract class Shape
    {
        private static int _nextId = 1;

        public int Id { get; }
        public string Color { get; set; }
        public bool IsFilled { get; set; }
        public DateTime CreatedAt { get; }

        protected Shape(string color = "Black", bool isFilled = true)
        {
            Id = _nextId++;
            Color = color;
            IsFilled = isFilled;
            CreatedAt = DateTime.Now;
        }

        // Abstract methods
        public abstract double GetArea();
        public abstract double GetPerimeter();
        public abstract string GetShapeType();

        // Virtual methods
        public virtual void Draw()
        {
            string fillStr = IsFilled ? $"[filled:{Color}]" : $"[outline:{Color}]";
            Console.WriteLine($"#{Id} {GetShapeType()} {fillStr} " +
                $"Area={GetArea():F2} Perimeter={GetPerimeter():F2}");
        }

        public virtual void Scale(double factor)
        {
            if (factor <= 0)
                throw new ArgumentOutOfRangeException(nameof(factor));
            Console.WriteLine($"Scaling {GetShapeType()} by {factor}x");
        }

        // Non-virtual (template)
        public string GetDescription()
        {
            return $"{GetShapeType()} | Color: {Color} | " +
                   $"Area: {GetArea():F2} | Perimeter: {GetPerimeter():F2} | " +
                   $"A:P ratio: {(GetPerimeter() > 0 ? GetArea() / GetPerimeter() : 0):F2}";
        }

        public bool IsLargerThan(Shape other)
            => GetArea() > other.GetArea();

        public override string ToString() => GetDescription();
    }

    /// <summary>
    /// วงกลม
    /// </summary>
    public class Circle : Shape
    {
        private double _radius;

        public double Radius
        {
            get => _radius;
            set => _radius = value > 0 ? value
                : throw new ArgumentOutOfRangeException(nameof(value));
        }

        public Circle(double radius, string color = "Red", bool isFilled = true)
            : base(color, isFilled)
        {
            Radius = radius;
        }

        public override double GetArea() => Math.PI * _radius * _radius;
        public override double GetPerimeter() => 2 * Math.PI * _radius;
        public override string GetShapeType() => "Circle";

        public override void Draw()
        {
            Console.Write("⭕ ");
            base.Draw();
        }

        public override void Scale(double factor)
        {
            base.Scale(factor);
            Radius *= factor;
        }

        // Circle-specific
        public Circle GetLargerCircle(double additionalRadius)
            => new Circle(_radius + additionalRadius, Color, IsFilled);
    }

    /// <summary>
    /// สี่เหลี่ยมมุมฉาก
    /// </summary>
    public class Rectangle : Shape
    {
        private double _width;
        private double _height;

        public double Width
        {
            get => _width;
            set => _width = value > 0 ? value
                : throw new ArgumentOutOfRangeException(nameof(value));
        }

        public double Height
        {
            get => _height;
            set => _height = value > 0 ? value
                : throw new ArgumentOutOfRangeException(nameof(value));
        }

        public bool IsSquare => Math.Abs(_width - _height) < 0.0001;
        public double Diagonal => Math.Sqrt(_width * _width + _height * _height);

        public Rectangle(double width, double height,
            string color = "Blue", bool isFilled = true)
            : base(color, isFilled)
        {
            Width = width;
            Height = height;
        }

        public override double GetArea() => _width * _height;
        public override double GetPerimeter() => 2 * (_width + _height);
        public override string GetShapeType() => IsSquare ? "Square" : "Rectangle";

        public override void Draw()
        {
            Console.Write("⬜ ");
            base.Draw();
            if (IsSquare) Console.WriteLine("  (Perfect Square!)");
        }

        public override void Scale(double factor)
        {
            base.Scale(factor);
            Width *= factor;
            Height *= factor;
        }

        public Rectangle Rotate90()
            => new Rectangle(_height, _width, Color, IsFilled);
    }

    /// <summary>
    /// สามเหลี่ยม
    /// </summary>
    public class Triangle : Shape
    {
        public double SideA { get; private set; }
        public double SideB { get; private set; }
        public double SideC { get; private set; }

        public string TriangleType
        {
            get
            {
                bool isRight = IsRightTriangle();
                bool isEqui = Math.Abs(SideA - SideB) < 0.001 && Math.Abs(SideB - SideC) < 0.001;
                bool isIso = Math.Abs(SideA - SideB) < 0.001 ||
                             Math.Abs(SideB - SideC) < 0.001 ||
                             Math.Abs(SideA - SideC) < 0.001;

                if (isEqui) return "Equilateral";
                if (isRight && isIso) return "Right Isosceles";
                if (isRight) return "Right";
                if (isIso) return "Isosceles";
                return "Scalene";
            }
        }

        public Triangle(double a, double b, double c,
            string color = "Green", bool isFilled = true)
            : base(color, isFilled)
        {
            if (a + b <= c || a + c <= b || b + c <= a)
                throw new ArgumentException("ไม่สามารถสร้างสามเหลี่ยมได้จากด้านที่กำหนด");
            SideA = a; SideB = b; SideC = c;
        }

        public override double GetPerimeter() => SideA + SideB + SideC;

        public override double GetArea()
        {
            double s = GetPerimeter() / 2;
            return Math.Sqrt(s * (s - SideA) * (s - SideB) * (s - SideC));
        }

        public override string GetShapeType() => $"Triangle({TriangleType})";

        public override void Draw()
        {
            Console.Write("🔺 ");
            base.Draw();
        }

        public override void Scale(double factor)
        {
            base.Scale(factor);
            SideA *= factor;
            SideB *= factor;
            SideC *= factor;
        }

        private bool IsRightTriangle()
        {
            double[] sides = [SideA, SideB, SideC];
            Array.Sort(sides);
            return Math.Abs(sides[0] * sides[0] + sides[1] * sides[1] - sides[2] * sides[2]) < 0.001;
        }
    }

    /// <summary>
    /// วงรี
    /// </summary>
    public class Ellipse : Shape
    {
        public double SemiMajorAxis { get; private set; }
        public double SemiMinorAxis { get; private set; }

        public Ellipse(double semiMajor, double semiMinor,
            string color = "Purple", bool isFilled = true)
            : base(color, isFilled)
        {
            SemiMajorAxis = semiMajor > 0 ? semiMajor : throw new ArgumentOutOfRangeException();
            SemiMinorAxis = semiMinor > 0 ? semiMinor : throw new ArgumentOutOfRangeException();
        }

        public override double GetArea() => Math.PI * SemiMajorAxis * SemiMinorAxis;

        // Ramanujan approximation
        public override double GetPerimeter()
        {
            double h = Math.Pow(SemiMajorAxis - SemiMinorAxis, 2) /
                       Math.Pow(SemiMajorAxis + SemiMinorAxis, 2);
            return Math.PI * (SemiMajorAxis + SemiMinorAxis) *
                   (1 + 3 * h / (10 + Math.Sqrt(4 - 3 * h)));
        }

        public override string GetShapeType() => "Ellipse";
    }

    /// <summary>
    /// Canvas สำหรับจัดการ shapes
    /// </summary>
    public class DrawingCanvas
    {
        private readonly List<Shape> _shapes = new();
        public string Name { get; }

        public DrawingCanvas(string name) => Name = name;

        public void Add(Shape shape)
        {
            _shapes.Add(shape);
        }

        public void DrawAll()
        {
            Console.WriteLine($"\n=== Canvas: {Name} ({_shapes.Count} shapes) ===");
            foreach (var shape in _shapes)
            {
                shape.Draw();
            }
        }

        public void ScaleAll(double factor)
        {
            Console.WriteLine($"\n=== Scaling all shapes by {factor}x ===");
            foreach (var shape in _shapes)
            {
                shape.Scale(factor);
            }
        }

        public Shape? GetLargestShape()
            => _shapes.MaxBy(s => s.GetArea());

        public Shape? GetSmallestShape()
            => _shapes.MinBy(s => s.GetArea());

        public double GetTotalArea()
            => _shapes.Sum(s => s.GetArea());

        public double GetTotalPerimeter()
            => _shapes.Sum(s => s.GetPerimeter());

        public Dictionary<string, List<Shape>> GroupByType()
            => _shapes.GroupBy(s => s.GetShapeType())
                      .ToDictionary(g => g.Key, g => g.ToList());

        public IEnumerable<T> GetShapesByType<T>() where T : Shape
            => _shapes.OfType<T>();

        public void PrintStatistics()
        {
            Console.WriteLine($"\n=== Statistics: {Name} ===");
            Console.WriteLine($"จำนวน Shape: {_shapes.Count}");
            Console.WriteLine($"พื้นที่รวม: {GetTotalArea():F2}");
            Console.WriteLine($"เส้นรอบรูปรวม: {GetTotalPerimeter():F2}");

            var largest = GetLargestShape();
            var smallest = GetSmallestShape();

            if (largest != null)
                Console.WriteLine($"Shape ใหญ่สุด: {largest.GetDescription()}");
            if (smallest != null)
                Console.WriteLine($"Shape เล็กสุด: {smallest.GetDescription()}");

            Console.WriteLine("\nจำนวนตามประเภท:");
            foreach (var (type, shapes) in GroupByType())
            {
                Console.WriteLine($"  {type}: {shapes.Count}");
            }

            Console.WriteLine($"\nวงกลม: {GetShapesByType<Circle>().Count()}");
            Console.WriteLine($"สี่เหลี่ยม: {GetShapesByType<Rectangle>().Count()}");
            Console.WriteLine($"สามเหลี่ยม: {GetShapesByType<Triangle>().Count()}");
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            var canvas = new DrawingCanvas("My Drawing");

            // เพิ่ม shapes ต่างๆ
            canvas.Add(new Circle(5, "Red"));
            canvas.Add(new Circle(3, "Orange"));
            canvas.Add(new Rectangle(4, 6, "Blue"));
            canvas.Add(new Rectangle(5, 5, "Navy")); // Square
            canvas.Add(new Triangle(3, 4, 5, "Green")); // Right triangle
            canvas.Add(new Triangle(5, 5, 5, "DarkGreen")); // Equilateral
            canvas.Add(new Ellipse(6, 4, "Purple"));

            // วาด shapes
            canvas.DrawAll();

            // สถิติ
            canvas.PrintStatistics();

            // Polymorphism demo
            Console.WriteLine("\n=== Polymorphism Demo ===");
            List<Shape> shapes = new List<Shape>
            {
                new Circle(1),
                new Rectangle(2, 3),
                new Triangle(3, 4, 5)
            };

            foreach (Shape s in shapes)
            {
                // เรียก method เดียวกัน แต่ผลลัพธ์ต่างกัน
                Console.WriteLine($"{s.GetShapeType()}: Area={s.GetArea():F2}");
            }

            // Type checking
            Console.WriteLine("\n=== Type Checking ===");
            foreach (Shape s in shapes)
            {
                if (s is Circle c)
                    Console.WriteLine($"Circle with r={c.Radius}");
                else if (s is Rectangle r)
                    Console.WriteLine($"Rectangle {r.Width}x{r.Height}{(r.IsSquare ? " (square)" : "")}");
                else if (s is Triangle t)
                    Console.WriteLine($"Triangle type: {t.TriangleType}");
            }
        }
    }
}
```

---

## Exercises

### Exercise 1: Animal Sound System
```csharp
// TODO: สร้าง hierarchy สำหรับสัตว์ที่ส่งเสียงต่างกัน:
// Animal (abstract) -> Dog, Cat, Cow, Duck, Frog
// แต่ละตัวมี MakeSound() ต่างกัน
// สร้าง SoundOrchestra class ที่:
// - รับ list ของ animals
// - MakeAllSounds() - ทุกตัวส่งเสียง
// - GetLoudest() - หาตัวที่ดังสุด (property SoundLevel)
// - GetByType<T>() - กรองตามประเภท

public abstract class SoundAnimal
{
    // TODO: Implement
}
```

### Exercise 2: Document Formatter
```csharp
// TODO: สร้าง Document hierarchy:
// Document (abstract) -> PlainText, HTML, Markdown, JSON
// แต่ละประเภทมี Format(string content) -> string ต่างกัน
// DocumentProcessor ที่:
// - ProcessAll(List<Document> docs) -> string[]
// - FindByType<T>() -> List<T>
// - GetDocumentStats() -> Dictionary<string, int>

public abstract class Document
{
    // TODO: Implement
}
```

### Exercise 3: Payment Calculator
```csharp
// TODO: สร้าง PaymentMethod hierarchy:
// PaymentMethod (abstract)
//   -> CashPayment: exact change หรือ overpay แล้วรับเงินทอน
//   -> CreditCardPayment: มีค่าธรรมเนียม, credit limit
//   -> DigitalWallet: มี balance, transfer fee
// แต่ละแบบมี ProcessPayment(decimal amount) -> PaymentResult
// PaymentResult มี Success, AmountPaid, Change, FeeCharged

public abstract class PaymentMethod
{
    // TODO: Implement
}
```

---

## สรุป

✅ Polymorphism = "ชื่อเดียว หลายพฤติกรรม" ทำงานผ่าน virtual + override  
✅ `virtual` ทำให้ method สามารถ override ได้ใน derived classes  
✅ `override` แทนที่ implementation ของ base class  
✅ `abstract` บังคับให้ derived classes ต้อง implement method  
✅ Upcasting ปลอดภัย (implicit), Downcasting ต้องระวัง (explicit)  
✅ `is` ตรวจสอบ type, `as` แปลง type (คืน null ถ้า fail)  
✅ Pattern matching (C# 7+) ทำให้ type checking สะดวกกว่า  
✅ Method hiding (`new`) ไม่ทำงานผ่าน base reference - หลีกเลี่ยง  

## Part ถัดไป
**Part 017: Interfaces** - เรียนรู้ interface declaration, implementation, multiple interfaces, default methods และ Common .NET interfaces

---
*Part 016/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
