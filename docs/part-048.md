# Part 048: MVC - Models และ ViewModels

## เนื้อหาใน Part นี้
- Model classes
- Data Annotations validation
- ViewModel pattern
- Model validation
- Fluent Validation เบื้องต้น
- โปรแกรมตัวอย่าง: User registration form

---

## 1. Model คืออะไร?

Model คือ class ที่แสดงถึงข้อมูลใน application มี 2 ประเภทหลัก:

1. **Domain Model** - แสดงถึง business entity (เช่น User, Product, Order)
2. **ViewModel** - แสดงถึงข้อมูลที่ View ต้องการ (อาจรวมหลาย domain models)

```
Domain Model           ViewModel
(Database/Business)    (View-specific)
     │                      │
     User                   UserProfileViewModel
     ├── Id                 ├── FullName (First + Last)
     ├── FirstName          ├── Email
     ├── LastName           ├── AvatarUrl
     ├── Email              └── MemberSince (formatted)
     ├── PasswordHash
     └── CreatedAt
```

---

## 2. Domain Model

```csharp
// Models/User.cs
public class User
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public DateTime DateOfBirth { get; set; }
    public bool IsActive { get; set; } = true;
    public bool IsEmailVerified { get; set; }
    public UserRole Role { get; set; } = UserRole.User;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
    
    // Navigation properties (สำหรับ EF Core)
    public List<Order> Orders { get; set; } = [];
    
    // Computed property
    public string FullName => $"{FirstName} {LastName}";
    public int Age => (DateTime.Today - DateOfBirth).Days / 365;
}

public enum UserRole { User, Admin, Manager }
```

---

## 3. Data Annotations Validation

Data Annotations ใช้ validate ข้อมูลได้ทั้งฝั่ง client และ server

### 3.1 Built-in Validation Attributes

```csharp
// ViewModels/RegisterViewModel.cs
public class RegisterViewModel
{
    // Required - ห้ามว่าง
    [Required(ErrorMessage = "กรุณาระบุชื่อ")]
    public string FirstName { get; set; } = string.Empty;

    [Required(ErrorMessage = "กรุณาระบุนามสกุล")]
    public string LastName { get; set; } = string.Empty;

    // StringLength - ความยาว
    [Required]
    [StringLength(100, MinimumLength = 3, 
        ErrorMessage = "ชื่อผู้ใช้ต้องมีความยาว 3-100 ตัวอักษร")]
    public string Username { get; set; } = string.Empty;

    // EmailAddress - format email
    [Required(ErrorMessage = "กรุณาระบุ email")]
    [EmailAddress(ErrorMessage = "รูปแบบ email ไม่ถูกต้อง")]
    public string Email { get; set; } = string.Empty;

    // MinLength/MaxLength
    [Required]
    [MinLength(8, ErrorMessage = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")]
    [MaxLength(100, ErrorMessage = "รหัสผ่านต้องไม่เกิน 100 ตัวอักษร")]
    [DataType(DataType.Password)]
    public string Password { get; set; } = string.Empty;

    // Compare - เปรียบเทียบค่ากับ property อื่น
    [Required]
    [Compare(nameof(Password), ErrorMessage = "รหัสผ่านไม่ตรงกัน")]
    [DataType(DataType.Password)]
    public string ConfirmPassword { get; set; } = string.Empty;

    // Phone - format phone number
    [Phone(ErrorMessage = "รูปแบบเบอร์โทรไม่ถูกต้อง")]
    public string? PhoneNumber { get; set; }

    // Range - ช่วงค่า
    [Required]
    [Range(typeof(DateTime), "1900-01-01", "2010-01-01",
        ErrorMessage = "วันเกิดต้องอยู่ระหว่าง ค.ศ. 1900-2010")]
    public DateTime DateOfBirth { get; set; }

    // Url - format URL
    [Url(ErrorMessage = "รูปแบบ URL ไม่ถูกต้อง")]
    public string? WebsiteUrl { get; set; }

    // RegularExpression - regex pattern
    [RegularExpression(@"^[a-zA-Z0-9_]{3,20}$",
        ErrorMessage = "ชื่อผู้ใช้ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _ (3-20 ตัว)")]
    public string? PreferredUsername { get; set; }

    // CreditCard
    [CreditCard(ErrorMessage = "หมายเลขบัตรไม่ถูกต้อง")]
    public string? CreditCardNumber { get; set; }

    // DataType - hint สำหรับ formatting (ไม่ validate)
    [DataType(DataType.Date)]
    public DateTime? EventDate { get; set; }

    [DataType(DataType.Currency)]
    public decimal Price { get; set; }

    [DataType(DataType.MultilineText)]
    public string? Bio { get; set; }

    // Display - ชื่อที่แสดงใน UI
    [Display(Name = "ที่อยู่อีเมล")]
    [Required]
    [EmailAddress]
    public string ContactEmail { get; set; } = string.Empty;
}
```

### 3.2 Custom Validation Attributes

```csharp
// Attributes/ThaiIdCardAttribute.cs
public class ThaiIdCardAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value, ValidationContext validationContext)
    {
        if (value is null) return ValidationResult.Success; // ถ้าไม่จำเป็น

        var idCard = value.ToString()!;

        // ตรวจสอบรูปแบบ: 13 หลัก
        if (!System.Text.RegularExpressions.Regex.IsMatch(idCard, @"^\d{13}$"))
            return new ValidationResult("เลขบัตรประชาชนต้องเป็นตัวเลข 13 หลัก");

        // ตรวจสอบ checksum
        int sum = 0;
        for (int i = 0; i < 12; i++)
            sum += int.Parse(idCard[i].ToString()) * (13 - i);

        int checkDigit = (11 - (sum % 11)) % 10;
        if (checkDigit != int.Parse(idCard[12].ToString()))
            return new ValidationResult("เลขบัตรประชาชนไม่ถูกต้อง");

        return ValidationResult.Success;
    }
}

// Attributes/FutureDateAttribute.cs
public class FutureDateAttribute : ValidationAttribute
{
    private readonly int _daysFromNow;

    public FutureDateAttribute(int minDaysFromNow = 0)
    {
        _daysFromNow = minDaysFromNow;
        ErrorMessage = $"วันที่ต้องเป็นวันในอนาคต (อย่างน้อย {minDaysFromNow} วัน)";
    }

    public override bool IsValid(object? value)
    {
        if (value is DateTime date)
            return date >= DateTime.Today.AddDays(_daysFromNow);
        return true;
    }
}

// ใช้งาน
public class EventRegistrationViewModel
{
    [ThaiIdCard]
    public string IdCardNumber { get; set; } = string.Empty;

    [FutureDate(minDaysFromNow: 7)]
    public DateTime EventDate { get; set; }
}
```

### 3.3 IValidatableObject

```csharp
// Validation ที่ต้องการ logic ซับซ้อน
public class BookingViewModel : IValidatableObject
{
    [Required]
    public DateTime CheckInDate { get; set; }

    [Required]
    public DateTime CheckOutDate { get; set; }

    [Range(1, 10)]
    public int GuestCount { get; set; }

    [Range(1, 5)]
    public int RoomCount { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        // Cross-field validation
        if (CheckOutDate <= CheckInDate)
            yield return new ValidationResult(
                "วันเช็คเอาท์ต้องหลังวันเช็คอิน",
                new[] { nameof(CheckOutDate) });

        if (CheckInDate < DateTime.Today)
            yield return new ValidationResult(
                "วันเช็คอินต้องไม่อยู่ในอดีต",
                new[] { nameof(CheckInDate) });

        var nights = (CheckOutDate - CheckInDate).Days;
        if (nights > 30)
            yield return new ValidationResult(
                "การจองต้องไม่เกิน 30 คืน",
                new[] { nameof(CheckOutDate) });

        if (GuestCount > RoomCount * 4)
            yield return new ValidationResult(
                $"จำนวนแขกเกินกำลังรับ ({RoomCount} ห้องรับได้ {RoomCount * 4} คน)",
                new[] { nameof(GuestCount) });
    }
}
```

---

## 4. ViewModel Pattern

### 4.1 ทำไมต้องใช้ ViewModel?

```csharp
// ❌ ไม่ดี: ส่ง Domain Model โดยตรงไปยัง View
public class User
{
    public int Id { get; set; }
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;  // ข้อมูลลับ!
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    // ...
}

return View(user); // ส่ง PasswordHash ไปด้วย!

// ✅ ดี: ใช้ ViewModel ที่มีเฉพาะข้อมูลที่ View ต้องการ
public class UserProfileViewModel
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string AvatarUrl { get; set; } = string.Empty;
    public string MemberSince { get; set; } = string.Empty;
    public int OrderCount { get; set; }
    // ไม่มี PasswordHash!
}
```

### 4.2 ประเภทของ ViewModels

```csharp
// 1. Display ViewModel - สำหรับแสดงข้อมูล
public class ProductDetailsViewModel
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public string FormattedPrice => Price.ToString("N2") + " ฿";
    public bool IsInStock => Stock > 0;
    public string StockStatus => Stock switch
    {
        0 => "สินค้าหมด",
        <= 5 => $"เหลือ {Stock} ชิ้น",
        _ => "มีสินค้า"
    };
    public int Stock { get; set; }
    public List<RelatedProductViewModel> RelatedProducts { get; set; } = [];
}

// 2. Form ViewModel - สำหรับ input/form
public class RegisterViewModel
{
    [Required, StringLength(50)]
    public string FirstName { get; set; } = string.Empty;

    [Required, StringLength(50)]
    public string LastName { get; set; } = string.Empty;

    [Required, EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required, MinLength(8)]
    [DataType(DataType.Password)]
    public string Password { get; set; } = string.Empty;

    [Required, Compare(nameof(Password))]
    [DataType(DataType.Password)]
    public string ConfirmPassword { get; set; } = string.Empty;
}

// 3. List ViewModel - สำหรับ list/grid
public class UserListViewModel
{
    public List<UserRowViewModel> Users { get; set; } = [];
    public int TotalCount { get; set; }
    public int CurrentPage { get; set; }
    public int TotalPages { get; set; }
    public string? SearchTerm { get; set; }
    public string? RoleFilter { get; set; }
    public List<string> AvailableRoles { get; set; } = [];
}

public class UserRowViewModel
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Role { get; set; } = string.Empty;
    public bool IsActive { get; set; }
    public string CreatedAt { get; set; } = string.Empty;
}
```

---

## 5. Model Validation ใน Controller

### 5.1 Manual Validation Check

```csharp
[HttpPost]
public async Task<IActionResult> Register(RegisterViewModel model)
{
    // ตรวจสอบ ModelState
    if (!ModelState.IsValid)
    {
        return View(model);
    }

    // ตรวจสอบเพิ่มเติม (business rules)
    if (await _userService.EmailExistsAsync(model.Email))
    {
        ModelState.AddModelError(nameof(model.Email), "Email นี้ถูกใช้งานแล้ว");
        return View(model);
    }

    // สร้าง user
    var user = await _userService.RegisterAsync(model);
    TempData["Success"] = "สมัครสมาชิกสำเร็จ!";
    return RedirectToAction("Login");
}
```

### 5.2 ModelState Methods

```csharp
// เพิ่ม error
ModelState.AddModelError("Email", "Email ถูกใช้งานแล้ว");
ModelState.AddModelError("", "เกิดข้อผิดพลาด กรุณาลองใหม่");  // global error

// ตรวจสอบ
if (!ModelState.IsValid)
{
    var errors = ModelState
        .Where(x => x.Value?.Errors.Count > 0)
        .Select(x => new
        {
            Field = x.Key,
            Errors = x.Value!.Errors.Select(e => e.ErrorMessage)
        })
        .ToList();
    
    foreach (var error in errors)
    {
        _logger.LogWarning("Validation error for {Field}: {Errors}", 
            error.Field, string.Join(", ", error.Errors));
    }
}

// ลบ error ของ field ที่กำหนด
ModelState.Remove("ConfirmPassword");

// ล้าง errors ทั้งหมด
ModelState.Clear();
```

### 5.3 ValidationProblemDetails สำหรับ API

```csharp
// [ApiController] จัดการ validation อัตโนมัติ
// แต่สามารถ customize ได้:
builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            var errors = context.ModelState
                .Where(x => x.Value?.Errors.Count > 0)
                .ToDictionary(
                    x => x.Key,
                    x => x.Value!.Errors.Select(e => e.ErrorMessage).ToArray()
                );

            return new BadRequestObjectResult(new
            {
                Status = 400,
                Title = "Validation Failed",
                Errors = errors
            });
        };
    });
```

---

## 6. Fluent Validation เบื้องต้น

FluentValidation เป็น library สำหรับ validation แบบ fluent API ที่ยืดหยุ่นกว่า Data Annotations

### 6.1 ติดตั้ง

```bash
dotnet add package FluentValidation.AspNetCore
```

### 6.2 สร้าง Validator

```csharp
// Validators/RegisterRequestValidator.cs
using FluentValidation;

public class RegisterRequestValidator : AbstractValidator<RegisterRequest>
{
    private readonly IUserRepository _userRepository;

    public RegisterRequestValidator(IUserRepository userRepository)
    {
        _userRepository = userRepository;

        // FirstName
        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("กรุณาระบุชื่อ")
            .Length(2, 50).WithMessage("ชื่อต้องมีความยาว 2-50 ตัวอักษร")
            .Matches(@"^[a-zA-Zก-ฮ\s]+$").WithMessage("ชื่อใช้ได้เฉพาะตัวอักษร");

        // Email
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("กรุณาระบุ email")
            .EmailAddress().WithMessage("รูปแบบ email ไม่ถูกต้อง")
            .MaximumLength(100).WithMessage("Email ต้องไม่เกิน 100 ตัวอักษร")
            .MustAsync(BeUniqueEmail).WithMessage("Email นี้ถูกใช้งานแล้ว");

        // Password
        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("กรุณาระบุรหัสผ่าน")
            .MinimumLength(8).WithMessage("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")
            .MaximumLength(100)
            .Matches(@"[A-Z]").WithMessage("รหัสผ่านต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
            .Matches(@"[a-z]").WithMessage("รหัสผ่านต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
            .Matches(@"[0-9]").WithMessage("รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว")
            .Matches(@"[!@#$%^&*]").WithMessage("รหัสผ่านต้องมีอักขระพิเศษอย่างน้อย 1 ตัว");

        // ConfirmPassword
        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("รหัสผ่านไม่ตรงกัน");

        // PhoneNumber (Optional)
        When(x => !string.IsNullOrWhiteSpace(x.PhoneNumber), () =>
        {
            RuleFor(x => x.PhoneNumber)
                .Matches(@"^(0[689]\d{8}|0[2-5]\d{7})$")
                .WithMessage("เบอร์โทรไม่ถูกต้อง (ต้องเป็น 9-10 หลัก)");
        });

        // DateOfBirth
        RuleFor(x => x.DateOfBirth)
            .NotEmpty().WithMessage("กรุณาระบุวันเกิด")
            .LessThan(DateTime.Today.AddYears(-13))
            .WithMessage("ต้องมีอายุอย่างน้อย 13 ปี")
            .GreaterThan(DateTime.Today.AddYears(-120))
            .WithMessage("วันเกิดไม่ถูกต้อง");
    }

    private async Task<bool> BeUniqueEmail(
        string email, CancellationToken cancellationToken)
    {
        return !await _userRepository.EmailExistsAsync(email);
    }
}
```

### 6.3 Custom Rules

```csharp
// Custom validator method
RuleFor(x => x.Username)
    .Must(username => !ContainsBadWords(username))
    .WithMessage("ชื่อผู้ใช้ไม่เหมาะสม");

// Conditional validation
RuleFor(x => x.CompanyName)
    .NotEmpty()
    .When(x => x.AccountType == AccountType.Business);

// Unless (เมื่อ condition เป็น false)
RuleFor(x => x.TaxId)
    .NotEmpty()
    .Unless(x => x.AccountType == AccountType.Personal);

// Custom async validator
RuleFor(x => x.Sku)
    .MustAsync(async (sku, cancellation) =>
    {
        var exists = await _productRepo.SkuExistsAsync(sku);
        return !exists;
    })
    .WithMessage("SKU นี้ถูกใช้งานแล้ว");
```

### 6.4 ลงทะเบียน FluentValidation

```csharp
// ลงทะเบียนอัตโนมัติ
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<RegisterRequestValidator>();

// หรือลงทะเบียนแบบ manual
builder.Services.AddScoped<IValidator<RegisterRequest>, RegisterRequestValidator>();
```

---

## 7. โปรแกรมตัวอย่าง: User Registration Form

### Program.cs

```csharp
using FluentValidation;
using FluentValidation.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

// FluentValidation
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Services
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<IUserRepository, InMemoryUserRepository>();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
    app.UseExceptionHandler("/Home/Error");

app.UseStaticFiles();
app.UseRouting();
app.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();

// ============================================
// MODELS
// ============================================

public class User
{
    public int Id { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public DateTime DateOfBirth { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public string FullName => $"{FirstName} {LastName}";
}

// ============================================
// VIEWMODELS
// ============================================

public class RegisterViewModel
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string ConfirmPassword { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public DateTime DateOfBirth { get; set; }
    public bool AgreeToTerms { get; set; }
}

public class UserProfileViewModel
{
    public int Id { get; set; }
    public string FullName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string? PhoneNumber { get; set; }
    public int Age { get; set; }
    public string MemberSince { get; set; } = string.Empty;
}

// ============================================
// VALIDATORS
// ============================================

public class RegisterViewModelValidator : AbstractValidator<RegisterViewModel>
{
    private readonly IUserRepository _userRepository;

    public RegisterViewModelValidator(IUserRepository userRepository)
    {
        _userRepository = userRepository;

        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("กรุณาระบุชื่อ")
            .Length(2, 50).WithMessage("ชื่อต้องมีความยาว 2-50 ตัวอักษร");

        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("กรุณาระบุนามสกุล")
            .Length(2, 50).WithMessage("นามสกุลต้องมีความยาว 2-50 ตัวอักษร");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("กรุณาระบุ email")
            .EmailAddress().WithMessage("รูปแบบ email ไม่ถูกต้อง")
            .MustAsync(BeUniqueEmail).WithMessage("Email นี้ถูกใช้งานแล้ว");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("กรุณาระบุรหัสผ่าน")
            .MinimumLength(8).WithMessage("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")
            .Matches(@"[A-Z]").WithMessage("ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
            .Matches(@"[0-9]").WithMessage("ต้องมีตัวเลขอย่างน้อย 1 ตัว");

        RuleFor(x => x.ConfirmPassword)
            .Equal(x => x.Password).WithMessage("รหัสผ่านไม่ตรงกัน");

        RuleFor(x => x.DateOfBirth)
            .NotEmpty().WithMessage("กรุณาระบุวันเกิด")
            .LessThan(DateTime.Today.AddYears(-13))
            .WithMessage("ต้องมีอายุอย่างน้อย 13 ปี");

        RuleFor(x => x.AgreeToTerms)
            .Equal(true).WithMessage("กรุณายอมรับเงื่อนไขการใช้งาน");
    }

    private async Task<bool> BeUniqueEmail(string email, CancellationToken ct)
        => !await _userRepository.EmailExistsAsync(email);
}

// ============================================
// CONTROLLER
// ============================================

public class AccountController : Controller
{
    private readonly IUserService _userService;
    private readonly ILogger<AccountController> _logger;

    public AccountController(IUserService userService, ILogger<AccountController> logger)
    {
        _userService = userService;
        _logger = logger;
    }

    [HttpGet]
    public IActionResult Register() => View(new RegisterViewModel());

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Register(RegisterViewModel model)
    {
        if (!ModelState.IsValid)
            return View(model);

        try
        {
            var user = await _userService.RegisterAsync(model);
            _logger.LogInformation("New user registered: {Email}", model.Email);
            TempData["Success"] = $"ยินดีต้อนรับ {user.FullName}! กรุณาตรวจสอบ email เพื่อยืนยัน";
            return RedirectToAction("RegisterSuccess");
        }
        catch (InvalidOperationException ex)
        {
            _logger.LogWarning(ex, "Registration failed for {Email}", model.Email);
            ModelState.AddModelError("", ex.Message);
            return View(model);
        }
    }

    [HttpGet]
    public IActionResult RegisterSuccess() => View();

    [HttpGet]
    public async Task<IActionResult> Profile(int id)
    {
        var user = await _userService.GetByIdAsync(id);
        if (user is null) return NotFound();

        var viewModel = new UserProfileViewModel
        {
            Id = user.Id,
            FullName = user.FullName,
            Email = user.Email,
            PhoneNumber = user.PhoneNumber,
            Age = (DateTime.Today - user.DateOfBirth).Days / 365,
            MemberSince = user.CreatedAt.ToString("MMMM yyyy")
        };

        return View(viewModel);
    }
}

// ============================================
// SERVICES
// ============================================

public interface IUserService
{
    Task<User> RegisterAsync(RegisterViewModel model);
    Task<User?> GetByIdAsync(int id);
}

public interface IUserRepository
{
    Task<bool> EmailExistsAsync(string email);
    Task<User> CreateAsync(User user);
    Task<User?> GetByIdAsync(int id);
}

public class UserService : IUserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }

    public async Task<User> RegisterAsync(RegisterViewModel model)
    {
        var user = new User
        {
            FirstName = model.FirstName,
            LastName = model.LastName,
            Email = model.Email.ToLower(),
            PasswordHash = HashPassword(model.Password),
            PhoneNumber = model.PhoneNumber,
            DateOfBirth = model.DateOfBirth
        };
        return await _repository.CreateAsync(user);
    }

    public Task<User?> GetByIdAsync(int id) => _repository.GetByIdAsync(id);

    private static string HashPassword(string password)
    {
        // ใน production ใช้ BCrypt หรือ PBKDF2
        using var sha256 = System.Security.Cryptography.SHA256.Create();
        var bytes = System.Text.Encoding.UTF8.GetBytes(password + "salt");
        var hash = sha256.ComputeHash(bytes);
        return Convert.ToBase64String(hash);
    }
}

public class InMemoryUserRepository : IUserRepository
{
    private readonly List<User> _users = [];
    private int _nextId = 1;
    private readonly object _lock = new();

    public Task<bool> EmailExistsAsync(string email)
        => Task.FromResult(_users.Any(u => u.Email.Equals(email, StringComparison.OrdinalIgnoreCase)));

    public Task<User> CreateAsync(User user)
    {
        lock (_lock)
        {
            user.Id = _nextId++;
            _users.Add(user);
            return Task.FromResult(user);
        }
    }

    public Task<User?> GetByIdAsync(int id)
        => Task.FromResult(_users.FirstOrDefault(u => u.Id == id));
}
```

### Views/Account/Register.cshtml

```cshtml
@model RegisterViewModel
@{
    ViewData["Title"] = "สมัครสมาชิก";
}

<div class="row justify-content-center">
    <div class="col-md-6">
        <div class="card shadow">
            <div class="card-header bg-primary text-white">
                <h4 class="mb-0">📝 สมัครสมาชิก</h4>
            </div>
            <div class="card-body">

                @if (!ViewData.ModelState.IsValid && ViewData.ModelState.ErrorCount > 0)
                {
                    <div class="alert alert-danger">
                        <strong>กรุณาแก้ไขข้อผิดพลาด:</strong>
                        <ul class="mb-0 mt-2">
                            @foreach (var error in ViewData.ModelState.Values
                                .SelectMany(v => v.Errors)
                                .Select(e => e.ErrorMessage))
                            {
                                <li>@error</li>
                            }
                        </ul>
                    </div>
                }

                <form asp-action="Register" method="post" id="registerForm">
                    @Html.AntiForgeryToken()

                    <div class="row mb-3">
                        <div class="col">
                            <label asp-for="FirstName" class="form-label">ชื่อ *</label>
                            <input asp-for="FirstName" class="form-control"
                                   placeholder="ชื่อ" autocomplete="given-name" />
                            <span asp-validation-for="FirstName" class="text-danger small"></span>
                        </div>
                        <div class="col">
                            <label asp-for="LastName" class="form-label">นามสกุล *</label>
                            <input asp-for="LastName" class="form-control"
                                   placeholder="นามสกุล" autocomplete="family-name" />
                            <span asp-validation-for="LastName" class="text-danger small"></span>
                        </div>
                    </div>

                    <div class="mb-3">
                        <label asp-for="Email" class="form-label">Email *</label>
                        <input asp-for="Email" class="form-control"
                               type="email" placeholder="your@email.com"
                               autocomplete="email" />
                        <span asp-validation-for="Email" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="PhoneNumber" class="form-label">เบอร์โทรศัพท์</label>
                        <input asp-for="PhoneNumber" class="form-control"
                               type="tel" placeholder="0812345678" />
                        <span asp-validation-for="PhoneNumber" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="DateOfBirth" class="form-label">วันเกิด *</label>
                        <input asp-for="DateOfBirth" class="form-control"
                               type="date" />
                        <span asp-validation-for="DateOfBirth" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="Password" class="form-label">รหัสผ่าน *</label>
                        <div class="input-group">
                            <input asp-for="Password" class="form-control"
                                   type="password" id="password" placeholder="รหัสผ่าน" />
                            <button type="button" class="btn btn-outline-secondary"
                                    onclick="togglePassword('password', this)">👁</button>
                        </div>
                        <div class="password-strength mt-1" id="strengthBar"></div>
                        <span asp-validation-for="Password" class="text-danger small"></span>
                        <div class="form-text">
                            ต้องมีอย่างน้อย 8 ตัว, ตัวพิมพ์ใหญ่, และตัวเลข
                        </div>
                    </div>

                    <div class="mb-3">
                        <label asp-for="ConfirmPassword" class="form-label">ยืนยันรหัสผ่าน *</label>
                        <input asp-for="ConfirmPassword" class="form-control"
                               type="password" placeholder="ยืนยันรหัสผ่าน" />
                        <span asp-validation-for="ConfirmPassword" class="text-danger small"></span>
                    </div>

                    <div class="mb-4 form-check">
                        <input asp-for="AgreeToTerms" type="checkbox" class="form-check-input" />
                        <label asp-for="AgreeToTerms" class="form-check-label">
                            ฉันยอมรับ <a href="#" target="_blank">เงื่อนไขการใช้งาน</a>
                            และ <a href="#" target="_blank">นโยบายความเป็นส่วนตัว</a>
                        </label>
                        <span asp-validation-for="AgreeToTerms" class="text-danger small d-block"></span>
                    </div>

                    <button type="submit" class="btn btn-primary w-100" id="submitBtn">
                        สมัครสมาชิก
                    </button>
                </form>

                <hr />
                <p class="text-center mb-0">
                    มีบัญชีแล้ว?
                    <a asp-controller="Account" asp-action="Login">เข้าสู่ระบบ</a>
                </p>
            </div>
        </div>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        function togglePassword(id, btn) {
            const input = document.getElementById(id);
            input.type = input.type === 'password' ? 'text' : 'password';
            btn.textContent = input.type === 'password' ? '👁' : '🙈';
        }

        document.getElementById('password')?.addEventListener('input', function() {
            const val = this.value;
            const bar = document.getElementById('strengthBar');
            let strength = 0;
            if (val.length >= 8) strength++;
            if (/[A-Z]/.test(val)) strength++;
            if (/[0-9]/.test(val)) strength++;
            if (/[!@#$%^&*]/.test(val)) strength++;

            const colors = ['', 'danger', 'warning', 'info', 'success'];
            const labels = ['', 'อ่อนมาก', 'อ่อน', 'ปานกลาง', 'แข็งแกร่ง'];
            bar.innerHTML = strength > 0
                ? `<div class="progress"><div class="progress-bar bg-${colors[strength]}" 
                     style="width:${strength*25}%">${labels[strength]}</div></div>`
                : '';
        });

        document.getElementById('registerForm')?.addEventListener('submit', function() {
            document.getElementById('submitBtn').textContent = 'กำลังดำเนินการ...';
            document.getElementById('submitBtn').disabled = true;
        });
    </script>
}
```

---

## Exercises

### Exercise 1: Product Form Validation
สร้าง form สำหรับ product พร้อม FluentValidation:

```csharp
// Validator ต้องตรวจสอบ:
// - Name: required, 3-200 chars
// - SKU: format XX0000, unique
// - Price: > 0, ไม่เกิน 1,000,000
// - Stock: >= 0
// - Description: ไม่เกิน 2000 chars
// - Category: ต้องอยู่ในรายการที่กำหนด
// - Image URL: valid URL ถ้าระบุ
```

### Exercise 2: Password Change Form
สร้าง form สำหรับเปลี่ยนรหัสผ่าน:

```csharp
public class ChangePasswordViewModel
{
    // CurrentPassword - required
    // NewPassword - required, min 8 chars, complexity rules
    // ConfirmNewPassword - must equal NewPassword
    // NewPassword ต้องไม่เหมือน CurrentPassword
}
```

### Exercise 3: Address Form
สร้าง form สำหรับที่อยู่พร้อม conditional validation:

```csharp
public class AddressViewModel
{
    // AddressLine1 - required
    // AddressLine2 - optional
    // District - required ถ้า Province เป็น "กรุงเทพมหานคร"
    // Province - required
    // PostalCode - required, 5 digits
    // Country - required, default "TH"
    // PhoneNumber - required ถ้า IsDeliveryAddress = true
}
```

---

## สรุป

✅ Domain Model แสดงถึง business entity, ViewModel แสดงถึงข้อมูลที่ View ต้องการ  
✅ Data Annotations: `[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`, `[RegularExpression]`  
✅ Custom validation ทำได้ด้วย `ValidationAttribute` หรือ `IValidatableObject`  
✅ ViewModel pattern ปกป้อง sensitive data และแยก concerns ของ View  
✅ FluentValidation ยืดหยุ่นกว่า Data Annotations เหมาะกับ business rules ที่ซับซ้อน  
✅ `ModelState.IsValid` ตรวจสอบ validation, `ModelState.AddModelError()` เพิ่ม error  
✅ FluentValidation รองรับ async validation สำหรับ database checks  
✅ ใช้ `IValidatableObject` สำหรับ cross-field validation  

---

## Part ถัดไป

ใน **Part 049** เราจะเรียนรู้เรื่อง **Routing ใน ASP.NET Core** อย่างละเอียด:
- Convention-based routing
- Attribute routing
- Route constraints
- URL generation
- Area routing

---

*Part 048/700 | Phase 3: ASP.NET Core เบื้องต้น | หลักสูตร C# และ ASP.NET Core*
