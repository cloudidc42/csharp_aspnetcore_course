# Part 015: Inheritance (การสืบทอด)

## เนื้อหาใน Part นี้
- Base และ Derived classes
- base keyword
- Method hiding vs Overriding
- sealed class/method
- Object class (root of all classes)
- โปรแกรมตัวอย่าง: Animal hierarchy

---

## 1. Inheritance คืออะไร?

Inheritance (การสืบทอด) คือกลไกที่ทำให้ class หนึ่ง (derived/child class) รับคุณสมบัติและพฤติกรรมจากอีก class หนึ่ง (base/parent class)

**ประโยชน์:**
- **Code reuse** - ไม่ต้องเขียนโค้ดซ้ำ
- **Extensibility** - ขยาย class ได้โดยไม่แก้ไข base class
- **Polymorphism** - ใช้ base type อ้างถึง derived objects ได้
- **IS-A relationship** - Dog IS-A Animal

```csharp
// Base class (parent)
public class Animal
{
    public string Name { get; set; }
    public int Age { get; set; }

    public Animal(string name, int age)
    {
        Name = name;
        Age = age;
    }

    public void Breathe()
    {
        Console.WriteLine($"{Name} กำลังหายใจ");
    }

    public void Sleep()
    {
        Console.WriteLine($"{Name} กำลังนอนหลับ");
    }

    public virtual void MakeSound()
    {
        Console.WriteLine($"{Name} ส่งเสียง...");
    }
}

// Derived class (child) - รับทุกอย่างจาก Animal
public class Dog : Animal
{
    public string Breed { get; set; }

    // Constructor ต้องเรียก base constructor
    public Dog(string name, int age, string breed)
        : base(name, age) // เรียก Animal constructor
    {
        Breed = breed;
    }

    // Override method จาก base
    public override void MakeSound()
    {
        Console.WriteLine($"{Name} เห่า: โฮ่ง โฮ่ง!");
    }

    // Method เพิ่มเติม (เฉพาะ Dog)
    public void Fetch()
    {
        Console.WriteLine($"{Name} วิ่งไปเก็บลูกบอล");
    }
}

// การใช้งาน
var dog = new Dog("บัดดี้", 3, "Labrador");
dog.Breathe();       // จาก Animal
dog.Sleep();         // จาก Animal
dog.MakeSound();     // Override ใน Dog
dog.Fetch();         // เฉพาะ Dog

Console.WriteLine($"{dog.Name}, อายุ {dog.Age}, สายพันธุ์ {dog.Breed}");
```

---

## 2. Access Modifiers ใน Inheritance

```csharp
public class BaseClass
{
    public string PublicField = "Public";        // เข้าถึงได้ทุกที่
    protected string ProtectedField = "Protected"; // เข้าถึงได้ใน class และ derived
    private string PrivateField = "Private";     // เฉพาะ class เท่านั้น
    internal string InternalField = "Internal";  // เฉพาะ assembly เดียวกัน

    public void PublicMethod() { }
    protected void ProtectedMethod() { }
    private void PrivateMethod() { }

    protected internal void ProtectedInternal() { } // protected หรือ internal
    private protected void PrivateProtected() { }   // protected และ private
}

public class DerivedClass : BaseClass
{
    public void TestAccess()
    {
        Console.WriteLine(PublicField);     // ✅ OK
        Console.WriteLine(ProtectedField);  // ✅ OK (protected)
        // Console.WriteLine(PrivateField); // ❌ Error! private
        Console.WriteLine(InternalField);   // ✅ OK (same assembly)

        PublicMethod();                     // ✅ OK
        ProtectedMethod();                  // ✅ OK (protected)
        // PrivateMethod();                 // ❌ Error! private
    }
}

// นอก class
var derived = new DerivedClass();
derived.PublicMethod();     // ✅ OK
// derived.ProtectedMethod(); // ❌ Error!
```

---

## 3. base Keyword

`base` ใช้เข้าถึง base class constructor และ members

```csharp
public class Vehicle
{
    public string Make { get; }
    public string Model { get; }
    public int Year { get; }
    public decimal BasePrice { get; protected set; }

    public Vehicle(string make, string model, int year, decimal basePrice)
    {
        Make = make;
        Model = model;
        Year = year;
        BasePrice = basePrice;
    }

    public virtual string GetDescription()
    {
        return $"{Year} {Make} {Model}";
    }

    public virtual decimal CalculateInsurance()
    {
        return BasePrice * 0.02m; // 2% ของราคา
    }
}

public class ElectricVehicle : Vehicle
{
    public int BatteryCapacityKwh { get; }
    public int RangeKm { get; }

    // เรียก base constructor ด้วย base()
    public ElectricVehicle(string make, string model, int year,
        decimal basePrice, int batteryKwh, int rangeKm)
        : base(make, model, year, basePrice) // เรียก Vehicle constructor
    {
        BatteryCapacityKwh = batteryKwh;
        RangeKm = rangeKm;
        BasePrice *= 1.2m; // EV แพงกว่า 20%
    }

    // Override และเรียก base method
    public override string GetDescription()
    {
        string vehicleDesc = base.GetDescription(); // เรียก Vehicle.GetDescription()
        return $"{vehicleDesc} (EV) - แบตเตอรี่ {BatteryCapacityKwh}kWh, ระยะ {RangeKm}km";
    }

    // Override calculation
    public override decimal CalculateInsurance()
    {
        decimal baseInsurance = base.CalculateInsurance(); // เรียก Vehicle.CalculateInsurance()
        return baseInsurance * 1.1m; // EV แพงกว่า 10%
    }

    // Method เพิ่มเติม
    public void Charge()
    {
        Console.WriteLine($"กำลังชาร์จ {Make} {Model}...");
    }
}

public class HybridVehicle : Vehicle
{
    public int ElectricRangeKm { get; }
    public double FuelEfficiencyKmL { get; }

    public HybridVehicle(string make, string model, int year,
        decimal basePrice, int electricRange, double fuelEfficiency)
        : base(make, model, year, basePrice)
    {
        ElectricRangeKm = electricRange;
        FuelEfficiencyKmL = fuelEfficiency;
        BasePrice *= 1.1m; // Hybrid แพงกว่า 10%
    }

    public override string GetDescription()
    {
        return $"{base.GetDescription()} (Hybrid) - " +
               $"ไฟฟ้า {ElectricRangeKm}km, น้ำมัน {FuelEfficiencyKmL}km/L";
    }
}

// การใช้งาน
var tesla = new ElectricVehicle("Tesla", "Model 3", 2024, 1_800_000m, 75, 500);
var prius = new HybridVehicle("Toyota", "Prius", 2024, 1_200_000m, 50, 26.5);

Console.WriteLine(tesla.GetDescription());
Console.WriteLine(prius.GetDescription());
Console.WriteLine($"Tesla Insurance: {tesla.CalculateInsurance():N0} บาท");
tesla.Charge();
```

---

## 4. Method Hiding vs Overriding

**Overriding** (แนะนำ) - แทนที่ implementation ของ base class  
**Hiding** (ไม่แนะนำ) - ซ่อน base class method โดยสร้าง method ใหม่

```csharp
public class BaseAnimal
{
    // virtual = สามารถ override ได้
    public virtual void Speak()
    {
        Console.WriteLine("BaseAnimal speaks");
    }

    // ไม่ virtual = ไม่สามารถ override ได้ (แต่ hide ได้)
    public void Move()
    {
        Console.WriteLine("BaseAnimal moves");
    }
}

// Overriding - ใช้ virtual + override
public class Cat : BaseAnimal
{
    // override - แทนที่ method จาก base
    public override void Speak()
    {
        Console.WriteLine("Meow! 🐱");
    }

    // new - ซ่อน method จาก base (Method Hiding)
    public new void Move()
    {
        Console.WriteLine("Cat sneaks quietly");
    }
}

// สาธิตความแตกต่าง
var cat = new Cat();
cat.Speak();  // "Meow! 🐱" (overridden)
cat.Move();   // "Cat sneaks quietly" (hidden)

// Polymorphism - ความแตกต่างที่สำคัญ
BaseAnimal animalRef = new Cat(); // reference ประเภท BaseAnimal
animalRef.Speak(); // "Meow! 🐱" - override ทำงาน (runtime dispatch)
animalRef.Move();  // "BaseAnimal moves" - hiding ไม่ทำงาน! (compile-time dispatch)

Console.WriteLine("\n=== Override vs Hide ===");
Console.WriteLine("cat.Speak():         Meow! (correct)");
Console.WriteLine("animalRef.Speak():   Meow! (override works!)");
Console.WriteLine("cat.Move():          Cat sneaks (correct)");
Console.WriteLine("animalRef.Move():    BaseAnimal! (hide doesn't work through base ref)");
```

---

## 5. sealed Class และ sealed Method

`sealed` ป้องกันการ inherit หรือ override

```csharp
// sealed class - ไม่สามารถสืบทอดได้
public sealed class StringHelper
{
    public static string Capitalize(string s)
        => string.IsNullOrEmpty(s) ? s : char.ToUpper(s[0]) + s[1..].ToLower();
}

// public class ExtendedHelper : StringHelper { } // Error! sealed

// sealed method - ป้องกันการ override ต่อไป
public class Animal2
{
    public virtual void Eat() { Console.WriteLine("Animal eats"); }
}

public class Dog2 : Animal2
{
    // sealed override - override ได้ครั้งนี้แต่ derived ต่อไปไม่ได้
    public sealed override void Eat()
    {
        Console.WriteLine("Dog eats kibble");
    }
}

public class GoldenRetriever : Dog2
{
    // public override void Eat() { } // Error! sealed in Dog2
}

// ตัวอย่างการใช้ sealed ใน real-world
public abstract class BaseRepository<T>
{
    // sealed เพื่อป้องกันการ override ใน derived classes
    public sealed T? GetById(int id)
    {
        Console.WriteLine($"Getting {typeof(T).Name} with id {id}");
        return FindById(id); // เรียก abstract method
    }

    // abstract - subclass ต้อง implement
    protected abstract T? FindById(int id);
}
```

---

## 6. Object Class

`Object` (หรือ `object`) คือ root base class ของทุก class ใน .NET

```csharp
// ทุก class สืบทอดจาก object โดยปริยาย
public class MyClass { }
// เหมือนกับ:
// public class MyClass : object { }

// Methods ที่ทุก class ได้รับจาก object:
// - ToString() - แปลงเป็น string
// - Equals(object) - เปรียบเทียบ
// - GetHashCode() - hash code
// - GetType() - รับ type info
// - MemberwiseClone() - shallow copy (protected)
// - ReferenceEquals() - เปรียบเทียบ reference (static)

object obj = new object();
Console.WriteLine(obj.ToString());    // "System.Object"
Console.WriteLine(obj.GetType());     // "System.Object"
Console.WriteLine(obj.GetHashCode()); // hash code

// Boxing และ Unboxing
int value = 42;
object boxed = value;     // Boxing - value type -> reference type
int unboxed = (int)boxed; // Unboxing - reference type -> value type

Console.WriteLine($"boxed: {boxed}, unboxed: {unboxed}");

// ใช้ object เป็น container (ไม่แนะนำ - ใช้ Generics แทน)
object[] mixed = [42, "Hello", 3.14, true, new DateTime(2024, 1, 1)];
foreach (object item in mixed)
{
    Console.WriteLine($"{item.GetType().Name}: {item}");
}

// Override ToString, Equals, GetHashCode
public class Color
{
    public byte R { get; }
    public byte G { get; }
    public byte B { get; }

    public Color(byte r, byte g, byte b)
    {
        R = r; G = g; B = b;
    }

    public override string ToString() => $"RGB({R}, {G}, {B})";

    public override bool Equals(object? obj)
    {
        if (obj is Color other)
            return R == other.R && G == other.G && B == other.B;
        return false;
    }

    public override int GetHashCode() => HashCode.Combine(R, G, B);

    // ทำให้ == ทำงานถูกต้อง
    public static bool operator ==(Color? a, Color? b)
        => a?.Equals(b) ?? b is null;
    public static bool operator !=(Color? a, Color? b) => !(a == b);
}

var red = new Color(255, 0, 0);
var red2 = new Color(255, 0, 0);
var blue = new Color(0, 0, 255);

Console.WriteLine(red);         // "RGB(255, 0, 0)"
Console.WriteLine(red == red2); // True (value equality)
Console.WriteLine(red == blue); // False
```

---

## 7. Inheritance Chains

```csharp
// Multi-level inheritance (C# ไม่รองรับ multiple inheritance สำหรับ class)
public class LivingThing
{
    public bool IsAlive { get; protected set; } = true;
    public virtual void Grow() { Console.WriteLine("Living thing grows"); }
}

public class Animal3 : LivingThing
{
    public string Name { get; set; }
    public int Age { get; set; }

    public Animal3(string name, int age)
    {
        Name = name;
        Age = age;
    }

    public override void Grow()
    {
        base.Grow();
        Age++;
        Console.WriteLine($"{Name} grows older, now {Age}");
    }

    public virtual void Feed(string food)
    {
        Console.WriteLine($"{Name} eats {food}");
    }
}

public class Mammal : Animal3
{
    public bool IsWarmBlooded { get; } = true;
    public double BodyTemperature { get; protected set; } = 37.0;

    public Mammal(string name, int age) : base(name, age) { }

    public virtual void RegulateTemperature()
    {
        Console.WriteLine($"{Name} maintains {BodyTemperature}°C body temperature");
    }
}

public class Dog3 : Mammal
{
    public string Breed { get; }

    public Dog3(string name, int age, string breed) : base(name, age)
    {
        Breed = breed;
    }

    public override void Feed(string food)
    {
        base.Feed(food);
        Console.WriteLine($"{Name} wags tail happily after eating {food}");
    }

    public void Bark() => Console.WriteLine($"{Name}: โฮ่ง!");
}

// ทุก class เป็น LivingThing, Animal3, Mammal ด้วย
var dog = new Dog3("Max", 2, "German Shepherd");
dog.Grow();
dog.Feed("kibble");
dog.RegulateTemperature();
dog.Bark();

// instanceof check
Console.WriteLine(dog is Dog3);       // True
Console.WriteLine(dog is Mammal);     // True
Console.WriteLine(dog is Animal3);    // True
Console.WriteLine(dog is LivingThing);// True
Console.WriteLine(dog is object);     // True
```

---

## 8. Calling Base Class Methods

```csharp
public class Shape
{
    public string Color { get; set; } = "Black";

    public Shape(string color = "Black")
    {
        Color = color;
    }

    public virtual double GetArea() => 0;
    public virtual double GetPerimeter() => 0;

    public virtual string GetInfo()
    {
        return $"Shape | Color: {Color} | Area: {GetArea():F2} | Perimeter: {GetPerimeter():F2}";
    }

    public virtual void Draw()
    {
        Console.WriteLine($"Drawing {GetType().Name} in {Color}");
    }
}

public class Rectangle2 : Shape
{
    public double Width { get; }
    public double Height { get; }

    public Rectangle2(double width, double height, string color = "Black")
        : base(color) // เรียก Shape constructor
    {
        Width = width;
        Height = height;
    }

    public override double GetArea() => Width * Height;
    public override double GetPerimeter() => 2 * (Width + Height);

    // ขยาย base method โดยเรียก base ก่อน
    public override string GetInfo()
    {
        string baseInfo = base.GetInfo(); // เรียก Shape.GetInfo()
        return $"{baseInfo} | Width: {Width}, Height: {Height}";
    }

    public override void Draw()
    {
        base.Draw(); // เรียก Shape.Draw() ก่อน
        Console.WriteLine($"  Size: {Width} x {Height}");
    }
}

public class Square : Rectangle2
{
    public double Side => Width; // Width == Height

    public Square(double side, string color = "Black")
        : base(side, side, color) { } // เรียก Rectangle2 constructor

    public override string GetInfo()
    {
        string baseInfo = base.GetInfo(); // เรียก Rectangle2.GetInfo()
        return $"{baseInfo} (Square with side {Side})";
    }
}

var sq = new Square(5, "Blue");
Console.WriteLine(sq.GetInfo());
sq.Draw();
```

---

## 9. โปรแกรมตัวอย่าง: Animal Hierarchy

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace AnimalHierarchyExample
{
    /// <summary>
    /// ประเภทอาหาร
    /// </summary>
    public enum DietType
    {
        Herbivore,   // กินพืช
        Carnivore,   // กินเนื้อ
        Omnivore,    // กินทั้งพืชและเนื้อ
        Insectivore  // กินแมลง
    }

    /// <summary>
    /// สิ่งมีชีวิต - Root class
    /// </summary>
    public abstract class LivingOrganism
    {
        public string Name { get; set; }
        public bool IsAlive { get; protected set; } = true;
        public DateTime BirthDate { get; }

        public int AgeInDays => (DateTime.Now - BirthDate).Days;

        protected LivingOrganism(string name, DateTime birthDate)
        {
            Name = name;
            BirthDate = birthDate;
        }

        public virtual void Grow()
        {
            Console.WriteLine($"{Name} กำลังเติบโต");
        }

        public virtual string GetStatus()
            => $"{Name} - {(IsAlive ? "มีชีวิต" : "ไม่มีชีวิต")} - อายุ {AgeInDays} วัน";
    }

    /// <summary>
    /// สัตว์ - สืบทอดจาก LivingOrganism
    /// </summary>
    public abstract class Animal : LivingOrganism
    {
        public DietType Diet { get; protected set; }
        public double Weight { get; protected set; } // กิโลกรัม
        public double Height { get; protected set; } // เซนติเมตร
        protected double EnergyLevel { get; set; } = 100;
        protected List<string> _foodHistory = new();

        protected Animal(string name, DateTime birthDate, DietType diet,
            double weight, double height)
            : base(name, birthDate)
        {
            Diet = diet;
            Weight = weight;
            Height = height;
        }

        // Abstract - ทุก Animal ต้องมีเสียงร้อง
        public abstract void MakeSound();

        // Virtual - มี default implementation แต่ override ได้
        public virtual void Eat(string food)
        {
            EnergyLevel = Math.Min(100, EnergyLevel + 20);
            _foodHistory.Add(food);
            Console.WriteLine($"{Name} กินอาหาร: {food} (Energy: {EnergyLevel:F0}%)");
        }

        public virtual void Sleep()
        {
            EnergyLevel = Math.Min(100, EnergyLevel + 40);
            Console.WriteLine($"{Name} นอนหลับ (Energy: {EnergyLevel:F0}%)");
        }

        public virtual void Move()
        {
            EnergyLevel = Math.Max(0, EnergyLevel - 10);
            Console.WriteLine($"{Name} เคลื่อนที่ (Energy: {EnergyLevel:F0}%)");
        }

        public override void Grow()
        {
            Weight *= 1.01;
            Height *= 1.005;
            Console.WriteLine($"{Name} เติบโต → น้ำหนัก {Weight:F1}kg, สูง {Height:F1}cm");
        }

        public override string GetStatus()
        {
            string baseStatus = base.GetStatus();
            return $"{baseStatus} | {Diet} | {Weight:F1}kg | Energy: {EnergyLevel:F0}%";
        }
    }

    /// <summary>
    /// สัตว์เลี้ยงลูกด้วยนม
    /// </summary>
    public abstract class Mammal : Animal
    {
        public bool IsWarmBlooded { get; } = true;
        public string FurColor { get; set; }
        protected int OffspringCount { get; private set; } = 0;

        protected Mammal(string name, DateTime birthDate, DietType diet,
            double weight, double height, string furColor)
            : base(name, birthDate, diet, weight, height)
        {
            FurColor = furColor;
        }

        public virtual void GiveBirth(string offspringName)
        {
            OffspringCount++;
            Console.WriteLine($"{Name} คลอดลูก: {offspringName} (ลูกคนที่ {OffspringCount})");
        }

        public override string GetStatus()
        {
            return $"{base.GetStatus()} | ขน: {FurColor} | ลูก: {OffspringCount}";
        }
    }

    /// <summary>
    /// นก
    /// </summary>
    public abstract class Bird : Animal
    {
        public double WingspanCm { get; }
        public bool CanFly { get; protected set; }

        protected Bird(string name, DateTime birthDate, DietType diet,
            double weight, double height, double wingspan, bool canFly = true)
            : base(name, birthDate, diet, weight, height)
        {
            WingspanCm = wingspan;
            CanFly = canFly;
        }

        public virtual void Fly()
        {
            if (!CanFly)
            {
                Console.WriteLine($"{Name} บินไม่ได้");
                return;
            }
            EnergyLevel = Math.Max(0, EnergyLevel - 15);
            Console.WriteLine($"{Name} กำลังบิน (Energy: {EnergyLevel:F0}%)");
        }

        public abstract void BuildNest();
    }

    /// <summary>
    /// สุนัข
    /// </summary>
    public class Dog : Mammal
    {
        public string Breed { get; }
        public bool IsTrainedForService { get; private set; }
        private int _trickCount = 0;
        private List<string> _tricks = new();

        public Dog(string name, DateTime birthDate, double weight,
            string breed, string furColor = "Brown")
            : base(name, birthDate, DietType.Omnivore, weight, 60, furColor)
        {
            Breed = breed;
        }

        public override void MakeSound()
        {
            Console.WriteLine($"{Name}: โฮ่ง! โฮ่ง! 🐕");
        }

        public void LearnTrick(string trick)
        {
            _tricks.Add(trick);
            _trickCount++;
            Console.WriteLine($"{Name} เรียนรู้ท่า: {trick} (รู้ {_trickCount} ท่า)");
        }

        public void PerformTrick()
        {
            if (_tricks.Count == 0)
            {
                Console.WriteLine($"{Name} ยังไม่รู้ท่าอะไร");
                return;
            }
            string trick = _tricks[Random.Shared.Next(_tricks.Count)];
            EnergyLevel = Math.Max(0, EnergyLevel - 5);
            Console.WriteLine($"{Name} แสดงท่า: {trick}!");
        }

        public void TrainForService()
        {
            IsTrainedForService = true;
            Console.WriteLine($"{Name} ผ่านการฝึกเป็นสุนัขรับใช้");
        }

        public override void Eat(string food)
        {
            base.Eat(food);
            if (food.Contains("bone") || food.Contains("กระดูก"))
                Console.WriteLine($"{Name} แทะกระดูกอย่างมีความสุข 🦴");
        }

        public override string GetStatus()
            => $"{base.GetStatus()} | {Breed} | {_trickCount} tricks";
    }

    /// <summary>
    /// แมว
    /// </summary>
    public class Cat : Mammal
    {
        public bool IsIndoor { get; set; }
        private int _livesLeft = 9; // แมว 9 ชีวิต 😄

        public Cat(string name, DateTime birthDate, double weight,
            string furColor = "Orange", bool isIndoor = true)
            : base(name, birthDate, DietType.Carnivore, weight, 30, furColor)
        {
            IsIndoor = isIndoor;
        }

        public override void MakeSound()
        {
            Console.WriteLine($"{Name}: เมี๊ยว~ 🐈");
        }

        public void Purr()
        {
            Console.WriteLine($"{Name}: purrr... purrrr... 😸");
        }

        public void UseOneLive()
        {
            if (_livesLeft > 0)
            {
                _livesLeft--;
                Console.WriteLine($"{Name} ใช้ชีวิต! เหลือ {_livesLeft} ชีวิต");
            }
        }

        public override void Move()
        {
            Console.WriteLine($"{Name} แอบเดินเงียบๆ");
            EnergyLevel = Math.Max(0, EnergyLevel - 5); // แมวเดินเงียบ ใช้พลังงานน้อย
        }
    }

    /// <summary>
    /// นกแก้ว
    /// </summary>
    public class Parrot : Bird
    {
        private List<string> _vocabulary = new();
        public string Species { get; }

        public Parrot(string name, DateTime birthDate, double weight,
            string species, double wingspan)
            : base(name, birthDate, DietType.Herbivore, weight, 30, wingspan)
        {
            Species = species;
        }

        public override void MakeSound()
        {
            if (_vocabulary.Count > 0)
            {
                string word = _vocabulary[Random.Shared.Next(_vocabulary.Count)];
                Console.WriteLine($"{Name}: \"{word}\" 🦜");
            }
            else
            {
                Console.WriteLine($"{Name}: อ๊าก! อ๊าก! 🦜");
            }
        }

        public void TeachWord(string word)
        {
            _vocabulary.Add(word);
            Console.WriteLine($"{Name} เรียนคำใหม่: \"{word}\"");
        }

        public override void BuildNest()
        {
            Console.WriteLine($"{Name} สร้างรังในโพรงไม้");
        }

        public override string GetStatus()
            => $"{base.GetStatus()} | {Species} | รู้ {_vocabulary.Count} คำ";
    }

    /// <summary>
    /// นกกระจอกเทศ - นกที่บินไม่ได้
    /// </summary>
    public class Ostrich : Bird
    {
        public double RunSpeedKmh { get; }

        public Ostrich(string name, DateTime birthDate, double weight)
            : base(name, birthDate, DietType.Omnivore, weight, 200, 200, false)
        {
            RunSpeedKmh = 70;
        }

        public override void MakeSound()
        {
            Console.WriteLine($"{Name}: บูม! บูม! (เสียงดังๆ) 🦤");
        }

        public override void Move()
        {
            EnergyLevel = Math.Max(0, EnergyLevel - 20);
            Console.WriteLine($"{Name} วิ่ง {RunSpeedKmh}km/h! (Energy: {EnergyLevel:F0}%)");
        }

        public override void BuildNest()
        {
            Console.WriteLine($"{Name} ขุดดินทำรัง");
        }
    }

    /// <summary>
    /// สวนสัตว์
    /// </summary>
    public class Zoo
    {
        private readonly List<Animal> _animals = new();
        public string Name { get; }

        public Zoo(string name) => Name = name;

        public void AddAnimal(Animal animal)
        {
            _animals.Add(animal);
            Console.WriteLine($"เพิ่ม {animal.GetType().Name} '{animal.Name}' เข้าสวนสัตว์ {Name}");
        }

        public void FeedingTime()
        {
            Console.WriteLine($"\n=== เวลาให้อาหาร ===");
            foreach (var animal in _animals.Where(a => a.IsAlive))
            {
                string food = animal.Diet switch
                {
                    DietType.Herbivore => "หญ้าและผักใบเขียว",
                    DietType.Carnivore => "เนื้อสด",
                    DietType.Omnivore => "อาหารผสม",
                    DietType.Insectivore => "แมลง",
                    _ => "อาหารสัตว์"
                };
                animal.Eat(food);
            }
        }

        public void SoundOff()
        {
            Console.WriteLine($"\n=== สวนสัตว์ส่งเสียง ===");
            foreach (var animal in _animals.Where(a => a.IsAlive))
            {
                animal.MakeSound();
            }
        }

        public void PrintStatus()
        {
            Console.WriteLine($"\n=== สถานะสัตว์ใน {Name} ===");
            foreach (var animal in _animals)
            {
                Console.WriteLine(animal.GetStatus());
            }
            Console.WriteLine($"\nรวม {_animals.Count} สัตว์");

            var byType = _animals.GroupBy(a => a.GetType().Name);
            foreach (var group in byType)
            {
                Console.WriteLine($"  {group.Key}: {group.Count()} ตัว");
            }
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            var zoo = new Zoo("สวนสัตว์ดุสิต");

            // สร้างสัตว์
            var dog = new Dog("บัดดี้", new DateTime(2022, 3, 15), 25, "Golden Retriever");
            dog.LearnTrick("นั่ง");
            dog.LearnTrick("นอน");
            dog.LearnTrick("จับมือ");

            var cat = new Cat("มีมี่", new DateTime(2021, 7, 20), 4, "Orange");
            var parrot = new Parrot("โพลลี่", new DateTime(2020, 1, 1), 0.5, "African Grey", 65);
            parrot.TeachWord("สวัสดี");
            parrot.TeachWord("อร่อย");
            parrot.TeachWord("บินไปไหน");

            var ostrich = new Ostrich("บิ๊กเบิร์ด", new DateTime(2019, 6, 10), 120);

            // เพิ่มเข้าสวนสัตว์
            zoo.AddAnimal(dog);
            zoo.AddAnimal(cat);
            zoo.AddAnimal(parrot);
            zoo.AddAnimal(ostrich);

            // กิจกรรม
            zoo.SoundOff();
            zoo.FeedingTime();

            dog.PerformTrick();
            cat.Purr();
            parrot.Fly();
            ostrich.Fly();   // บินไม่ได้!
            ostrich.Move();  // วิ่งแทน

            zoo.PrintStatus();

            // Polymorphism
            Console.WriteLine("\n=== Polymorphism Demo ===");
            List<Animal> animals = new List<Animal> { dog, cat, parrot, ostrich };

            foreach (Animal animal in animals)
            {
                Console.Write($"{animal.GetType().Name,-15}: ");
                animal.MakeSound(); // เรียก method ของแต่ละ class
            }
        }
    }
}
```

---

## Exercises

### Exercise 1: Vehicle Hierarchy
```csharp
// TODO: สร้าง hierarchy:
// Vehicle (abstract) -> MotorVehicle -> Car, Truck, Motorcycle
//                    -> NonMotorVehicle -> Bicycle, Skateboard
// แต่ละ class มี:
// - Properties ที่เหมาะสม
// - Constructor ที่เรียก base()
// - Override MakeSound(), Move(), GetFuelType()

public abstract class Vehicle
{
    // TODO: Implement base class
}
```

### Exercise 2: Employee Hierarchy
```csharp
// TODO: สร้าง hierarchy:
// Employee (base) -> FullTimeEmployee, PartTimeEmployee, Contractor
// แต่ละประเภทมีวิธีคำนวณเงินเดือนต่างกัน:
// FullTime: เงินเดือน + โบนัส
// PartTime: รายชั่วโมง * จำนวนชั่วโมง
// Contractor: ราคาต่อโปรเจค

public abstract class Employee
{
    // TODO: Implement
}
```

### Exercise 3: Game Character
```csharp
// TODO: สร้าง RPG character hierarchy:
// Character (abstract base)
//   - Warrior: melee damage, armor
//   - Mage: spell damage, mana
//   - Archer: ranged damage, arrows
// แต่ละ character มี Attack(), Defend(), UseSpecialAbility()
// สร้าง Battle(Character c1, Character c2) method

public abstract class Character
{
    // TODO: Implement
}
```

---

## สรุป

✅ Inheritance ช่วย reuse code และสร้าง IS-A relationships  
✅ `base` keyword ใช้เรียก base class constructor และ methods  
✅ `virtual` + `override` = Overriding (runtime dispatch - แนะนำ)  
✅ `new` keyword = Method Hiding (compile-time dispatch - หลีกเลี่ยง)  
✅ `sealed` ป้องกันการ inherit class หรือ override method  
✅ `Object` เป็น root class ของทุก class - มี ToString, Equals, GetHashCode  
✅ C# รองรับ single inheritance สำหรับ class (แต่ multiple สำหรับ interface)  

## Part ถัดไป
**Part 016: Polymorphism** - เรียนรู้ virtual/override, abstract methods, upcasting/downcasting, is/as keywords

---
*Part 015/700 | Phase 1: พื้นฐาน C# | หลักสูตร C# และ ASP.NET Core*
