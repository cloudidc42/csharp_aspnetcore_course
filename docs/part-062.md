# Part 062: Background Services

## เนื้อหาใน Part นี้
- IHostedService - Interface หลัก
- BackgroundService - Abstract class สำหรับ background work
- Hosted Service lifecycle
- Worker Service template
- Scheduled tasks ด้วย Timer และ Cron
- โปรแกรมตัวอย่าง: Email Reminder Service

---

## 1. IHostedService

**IHostedService** เป็น interface หลักที่ใช้สร้าง background service ใน ASP.NET Core มี 2 method:

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

### การ Implement IHostedService ตรงๆ

```csharp
// Services/SimpleHostedService.cs
using Microsoft.Extensions.Hosting;

namespace BackgroundServiceDemo.Services;

public class SimpleHostedService : IHostedService, IDisposable
{
    private readonly ILogger<SimpleHostedService> _logger;
    private Timer? _timer;

    public SimpleHostedService(ILogger<SimpleHostedService> logger)
    {
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("SimpleHostedService เริ่มต้นทำงาน");

        // ทำงานทุก 5 วินาที
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(5));

        return Task.CompletedTask;
    }

    private void DoWork(object? state)
    {
        _logger.LogInformation("SimpleHostedService กำลังทำงาน: {Time}", DateTimeOffset.Now);
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("SimpleHostedService หยุดทำงาน");

        _timer?.Change(Timeout.Infinite, 0);

        return Task.CompletedTask;
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

### การลงทะเบียน Hosted Service

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// วิธีที่ 1: AddHostedService
builder.Services.AddHostedService<SimpleHostedService>();

// วิธีที่ 2: AddHostedService แบบ factory
builder.Services.AddHostedService(sp =>
    new SimpleHostedService(sp.GetRequiredService<ILogger<SimpleHostedService>>()));

var app = builder.Build();
app.Run();
```

---

## 2. BackgroundService - Abstract Base Class

**BackgroundService** เป็น abstract class ที่ implement `IHostedService` และให้เราเขียนแค่ `ExecuteAsync`:

```csharp
public abstract class BackgroundService : IHostedService, IDisposable
{
    protected abstract Task ExecuteAsync(CancellationToken stoppingToken);

    public virtual Task StartAsync(CancellationToken cancellationToken) { ... }
    public virtual Task StopAsync(CancellationToken cancellationToken) { ... }
    public virtual void Dispose() { ... }
}
```

### ตัวอย่าง BackgroundService อย่างง่าย

```csharp
// Services/PrintTimeService.cs
using Microsoft.Extensions.Hosting;

namespace BackgroundServiceDemo.Services;

public class PrintTimeService : BackgroundService
{
    private readonly ILogger<PrintTimeService> _logger;

    public PrintTimeService(ILogger<PrintTimeService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("PrintTimeService เริ่มต้น");

        // วน loop ตราบที่ยังไม่ถูก cancel
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("เวลาปัจจุบัน: {Time}", DateTimeOffset.Now);

            // รอ 10 วินาทีก่อนรอบถัดไป
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }

        _logger.LogInformation("PrintTimeService สิ้นสุด");
    }
}
```

---

## 3. Hosted Service Lifecycle

ลำดับการทำงานของ Hosted Services:

```
Application Start
    ↓
IHostedService.StartAsync() สำหรับทุก registered service (ตามลำดับ)
    ↓
Application Running (Web server accepts requests)
    ↓
Shutdown Signal (Ctrl+C หรือ SIGTERM)
    ↓
IHostedService.StopAsync() สำหรับทุก registered service (ย้อนกลับ)
    ↓
Application Exit
```

### ตัวอย่าง Lifecycle ที่สมบูรณ์

```csharp
// Services/LifecycleService.cs
using Microsoft.Extensions.Hosting;

namespace BackgroundServiceDemo.Services;

public class LifecycleService : BackgroundService
{
    private readonly ILogger<LifecycleService> _logger;

    public LifecycleService(ILogger<LifecycleService> logger)
    {
        _logger = logger;
    }

    public override async Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("LifecycleService: StartAsync - กำลัง initialize");

        // ทำ initialization ที่จำเป็น
        await InitializeAsync(cancellationToken);

        _logger.LogInformation("LifecycleService: เริ่ม ExecuteAsync");
        await base.StartAsync(cancellationToken);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                await DoWorkAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
            }
        }
        catch (OperationCanceledException)
        {
            // ปกติเมื่อ token ถูก cancel
            _logger.LogInformation("LifecycleService: ถูก cancel");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "LifecycleService: เกิด error ที่ไม่คาดคิด");
            throw; // re-throw เพื่อให้ application รู้ว่า service fail
        }
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("LifecycleService: StopAsync - กำลัง cleanup");

        // รอให้ ExecuteAsync จบ (หรือ timeout)
        await base.StopAsync(cancellationToken);

        // Cleanup resources
        await CleanupAsync();

        _logger.LogInformation("LifecycleService: หยุดแล้ว");
    }

    private async Task InitializeAsync(CancellationToken cancellationToken)
    {
        // เชื่อมต่อ database, load config, etc.
        await Task.Delay(1000, cancellationToken);
        _logger.LogInformation("LifecycleService: Initialization สำเร็จ");
    }

    private async Task DoWorkAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("LifecycleService: กำลังทำงาน...");
        await Task.Delay(500, stoppingToken);
    }

    private Task CleanupAsync()
    {
        _logger.LogInformation("LifecycleService: Cleanup สำเร็จ");
        return Task.CompletedTask;
    }
}
```

---

## 4. Worker Service Template

**Worker Service** เป็น .NET project template สำหรับ background processing:

```bash
# สร้าง Worker Service project
dotnet new worker -n MyWorkerService
cd MyWorkerService
```

### โครงสร้าง Worker Service

```
MyWorkerService/
├── Worker.cs          # BackgroundService หลัก
├── Program.cs         # Setup และ configuration
└── appsettings.json
```

### Worker.cs

```csharp
// Worker.cs
namespace MyWorkerService;

public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;

    public Worker(ILogger<Worker> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            if (_logger.IsEnabled(LogLevel.Information))
            {
                _logger.LogInformation("Worker running at: {time}", DateTimeOffset.Now);
            }
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

### Program.cs สำหรับ Worker Service

```csharp
// Program.cs
using MyWorkerService;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<Worker>();

var host = builder.Build();
host.Run();
```

### ติดตั้งเป็น Windows Service หรือ systemd

```bash
# ติดตั้ง package สำหรับ Windows Service
dotnet add package Microsoft.Extensions.Hosting.WindowsServices

# ติดตั้ง package สำหรับ Linux systemd
dotnet add package Microsoft.Extensions.Hosting.Systemd
```

```csharp
// Program.cs สำหรับ Windows Service
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddWindowsService(options =>
{
    options.ServiceName = "My Background Service";
});
builder.Services.AddHostedService<Worker>();

// Program.cs สำหรับ Linux systemd
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddSystemd();
builder.Services.AddHostedService<Worker>();
```

---

## 5. Scheduled Tasks

### วิธีที่ 1: ใช้ Timer อย่างง่าย

```csharp
// Services/TimerService.cs
using Microsoft.Extensions.Hosting;

namespace BackgroundServiceDemo.Services;

public class TimerService : BackgroundService
{
    private readonly ILogger<TimerService> _logger;
    private readonly TimeSpan _interval = TimeSpan.FromHours(1);

    public TimerService(ILogger<TimerService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // รอถึงเวลาที่กำหนด (เช่น ตี 2 ของวัน)
        await WaitForNextRunTime(stoppingToken);

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoScheduledWorkAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "เกิด error ใน scheduled task");
            }

            // รอจนถึงรอบถัดไป
            await Task.Delay(_interval, stoppingToken);
        }
    }

    private async Task WaitForNextRunTime(CancellationToken stoppingToken)
    {
        var now = DateTime.Now;
        var nextRun = now.Date.AddHours(2); // ตี 2

        if (now > nextRun)
            nextRun = nextRun.AddDays(1); // วันถัดไป

        var delay = nextRun - now;
        _logger.LogInformation("รอถึงเวลา {NextRun} (อีก {Delay})", nextRun, delay);

        await Task.Delay(delay, stoppingToken);
    }

    private async Task DoScheduledWorkAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("กำลังทำ scheduled work: {Time}", DateTime.Now);
        // ทำงานที่ต้องการ
        await Task.Delay(5000, stoppingToken);
    }
}
```

### วิธีที่ 2: ใช้ Cron Expression (NCronTab)

```bash
dotnet add package NCrontab
```

```csharp
// Services/CronService.cs
using NCrontab;
using Microsoft.Extensions.Hosting;

namespace BackgroundServiceDemo.Services;

public abstract class CronJobService : BackgroundService
{
    private readonly CrontabSchedule _schedule;
    private DateTime _nextRun;

    protected CronJobService(string cronExpression)
    {
        // Parse cron expression (5 fields: minute hour day month dayofweek)
        _schedule = CrontabSchedule.Parse(cronExpression);
        _nextRun = _schedule.GetNextOccurrence(DateTime.Now);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var now = DateTime.Now;

            if (now >= _nextRun)
            {
                await ProcessAsync(stoppingToken);
                _nextRun = _schedule.GetNextOccurrence(DateTime.Now);
            }

            await Task.Delay(1000, stoppingToken); // ตรวจสอบทุก 1 วินาที
        }
    }

    protected abstract Task ProcessAsync(CancellationToken stoppingToken);
}

// ตัวอย่างการใช้งาน
public class DailyReportService : CronJobService
{
    private readonly ILogger<DailyReportService> _logger;

    public DailyReportService(ILogger<DailyReportService> logger)
        : base("0 9 * * *") // ทุกวันตอน 9:00 AM
    {
        _logger = logger;
    }

    protected override async Task ProcessAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("กำลังสร้าง daily report...");
        await Task.Delay(2000, stoppingToken); // จำลองการทำงาน
        _logger.LogInformation("Daily report สร้างเสร็จแล้ว");
    }
}
```

### วิธีที่ 3: Quartz.NET (สำหรับ Enterprise)

```bash
dotnet add package Quartz
dotnet add package Quartz.Extensions.Hosting
dotnet add package Quartz.Extensions.DependencyInjection
```

```csharp
// Jobs/CleanupJob.cs
using Quartz;

namespace BackgroundServiceDemo.Jobs;

[DisallowConcurrentExecution]
public class CleanupJob : IJob
{
    private readonly ILogger<CleanupJob> _logger;

    public CleanupJob(ILogger<CleanupJob> logger)
    {
        _logger = logger;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        _logger.LogInformation("กำลัง cleanup เวลา: {Time}", DateTimeOffset.Now);
        await Task.Delay(1000);
        _logger.LogInformation("Cleanup เสร็จแล้ว");
    }
}

// Program.cs
builder.Services.AddQuartz(q =>
{
    var jobKey = new JobKey("CleanupJob");

    q.AddJob<CleanupJob>(opts => opts.WithIdentity(jobKey));

    q.AddTrigger(opts => opts
        .ForJob(jobKey)
        .WithIdentity("CleanupJob-trigger")
        .WithCronSchedule("0 0 0 * * ?") // ทุกวันเที่ยงคืน
    );
});

builder.Services.AddQuartzHostedService(q => q.WaitForJobsToComplete = true);
```

---

## 6. Scoped Services ใน Background Service

Background Service เป็น Singleton ดังนั้นต้องใช้ `IServiceScopeFactory` เพื่อเข้าถึง Scoped services:

```csharp
// Services/ScopedBackgroundService.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace BackgroundServiceDemo.Services;

public class ScopedBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<ScopedBackgroundService> _logger;

    public ScopedBackgroundService(
        IServiceScopeFactory scopeFactory,
        ILogger<ScopedBackgroundService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // สร้าง scope ใหม่ทุกรอบ
            await using var scope = _scopeFactory.CreateAsyncScope();

            // ดึง scoped service
            var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();

            await ProcessAsync(dbContext, emailService, stoppingToken);

            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    private async Task ProcessAsync(
        AppDbContext db,
        IEmailService emailService,
        CancellationToken stoppingToken)
    {
        _logger.LogInformation("Processing...");
        // ทำงานกับ database และ services
    }
}
```

---

## โปรแกรมตัวอย่าง: Email Reminder Service

ระบบส่งอีเมลเตือนความจำ โดยตรวจสอบ appointments ที่ใกล้ถึงและส่งอีเมลล่วงหน้า 24 ชั่วโมง

### Models

```csharp
// Models/Appointment.cs
namespace EmailReminderService.Models;

public class Appointment
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public string UserEmail { get; set; } = string.Empty;
    public string UserName { get; set; } = string.Empty;
    public DateTime ScheduledAt { get; set; }
    public bool ReminderSent { get; set; }
    public DateTime? ReminderSentAt { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

// Models/EmailMessage.cs
namespace EmailReminderService.Models;

public class EmailMessage
{
    public string To { get; set; } = string.Empty;
    public string Subject { get; set; } = string.Empty;
    public string Body { get; set; } = string.Empty;
    public bool IsHtml { get; set; } = true;
}
```

### AppDbContext

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using EmailReminderService.Models;

namespace EmailReminderService.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Appointment> Appointments => Set<Appointment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Appointment>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Title).IsRequired().HasMaxLength(200);
            entity.Property(e => e.UserEmail).IsRequired().HasMaxLength(100);

            // Index สำหรับ query ที่ใช้บ่อย
            entity.HasIndex(e => new { e.ScheduledAt, e.ReminderSent });
        });
    }
}
```

### Email Service

```csharp
// Services/IEmailService.cs
using EmailReminderService.Models;

namespace EmailReminderService.Services;

public interface IEmailService
{
    Task SendAsync(EmailMessage message);
    Task SendReminderAsync(Appointment appointment);
}

// Services/EmailService.cs
using MailKit.Net.Smtp;
using MimeKit;
using EmailReminderService.Models;

namespace EmailReminderService.Services;

public class EmailService : IEmailService
{
    private readonly IConfiguration _configuration;
    private readonly ILogger<EmailService> _logger;

    public EmailService(IConfiguration configuration, ILogger<EmailService> logger)
    {
        _configuration = configuration;
        _logger = logger;
    }

    public async Task SendAsync(EmailMessage message)
    {
        var emailMessage = new MimeMessage();

        emailMessage.From.Add(new MailboxAddress(
            _configuration["Email:SenderName"],
            _configuration["Email:SenderEmail"]));

        emailMessage.To.Add(new MailboxAddress(message.To, message.To));
        emailMessage.Subject = message.Subject;

        var builder = new BodyBuilder();
        if (message.IsHtml)
            builder.HtmlBody = message.Body;
        else
            builder.TextBody = message.Body;

        emailMessage.Body = builder.ToMessageBody();

        using var client = new SmtpClient();
        try
        {
            await client.ConnectAsync(
                _configuration["Email:SmtpHost"],
                int.Parse(_configuration["Email:SmtpPort"] ?? "587"),
                false);

            await client.AuthenticateAsync(
                _configuration["Email:Username"],
                _configuration["Email:Password"]);

            await client.SendAsync(emailMessage);
            await client.DisconnectAsync(true);

            _logger.LogInformation("ส่งอีเมลไปยัง {To} สำเร็จ", message.To);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "ไม่สามารถส่งอีเมลไปยัง {To}", message.To);
            throw;
        }
    }

    public async Task SendReminderAsync(Appointment appointment)
    {
        var body = $"""
            <html>
            <body style="font-family: Arial, sans-serif; padding: 20px;">
                <h2 style="color: #2c3e50;">เตือนความจำ: {appointment.Title}</h2>
                <p>สวัสดีคุณ {appointment.UserName},</p>
                <p>คุณมีนัดหมายในอีก 24 ชั่วโมง:</p>
                <div style="background: #f8f9fa; padding: 15px; border-radius: 8px; margin: 15px 0;">
                    <p><strong>หัวข้อ:</strong> {appointment.Title}</p>
                    <p><strong>รายละเอียด:</strong> {appointment.Description}</p>
                    <p><strong>เวลา:</strong> {appointment.ScheduledAt:dd/MM/yyyy HH:mm}</p>
                </div>
                <p>กรุณาอย่าลืมนัดหมายของคุณ</p>
                <p>ขอบคุณ,<br>ระบบแจ้งเตือนอัตโนมัติ</p>
            </body>
            </html>
            """;

        await SendAsync(new EmailMessage
        {
            To = appointment.UserEmail,
            Subject = $"เตือนความจำ: {appointment.Title} - {appointment.ScheduledAt:dd/MM/yyyy HH:mm}",
            Body = body
        });
    }
}
```

### Email Reminder Background Service

```csharp
// Services/EmailReminderBackgroundService.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using EmailReminderService.Data;

namespace EmailReminderService.Services;

public class EmailReminderBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<EmailReminderBackgroundService> _logger;

    // ตรวจสอบทุก 15 นาที
    private readonly TimeSpan _checkInterval = TimeSpan.FromMinutes(15);

    // ส่งเตือนล่วงหน้า 24 ชั่วโมง
    private readonly TimeSpan _reminderWindow = TimeSpan.FromHours(24);

    public EmailReminderBackgroundService(
        IServiceScopeFactory scopeFactory,
        ILogger<EmailReminderBackgroundService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Email Reminder Service เริ่มต้นทำงาน");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await CheckAndSendRemindersAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "เกิด error ใน Email Reminder Service");
            }

            await Task.Delay(_checkInterval, stoppingToken);
        }
    }

    private async Task CheckAndSendRemindersAsync(CancellationToken stoppingToken)
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();

        var now = DateTime.UtcNow;
        var reminderDeadline = now.Add(_reminderWindow);

        // ดึง appointments ที่ยังไม่ได้ส่ง reminder และอยู่ในช่วงเวลาที่กำหนด
        var pendingAppointments = await db.Appointments
            .Where(a => !a.ReminderSent
                     && a.ScheduledAt > now
                     && a.ScheduledAt <= reminderDeadline)
            .ToListAsync(stoppingToken);

        if (!pendingAppointments.Any())
        {
            _logger.LogDebug("ไม่มี appointment ที่ต้องส่ง reminder");
            return;
        }

        _logger.LogInformation("พบ {Count} appointments ที่ต้องส่ง reminder", pendingAppointments.Count);

        foreach (var appointment in pendingAppointments)
        {
            if (stoppingToken.IsCancellationRequested) break;

            try
            {
                await emailService.SendReminderAsync(appointment);

                appointment.ReminderSent = true;
                appointment.ReminderSentAt = DateTime.UtcNow;

                _logger.LogInformation(
                    "ส่ง reminder สำหรับ appointment {Id} ไปยัง {Email} สำเร็จ",
                    appointment.Id,
                    appointment.UserEmail);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "ไม่สามารถส่ง reminder สำหรับ appointment {Id}",
                    appointment.Id);
            }
        }

        // บันทึกการเปลี่ยนแปลงทั้งหมด
        await db.SaveChangesAsync(stoppingToken);
    }
}
```

### Appointment Service (สำหรับ CRUD)

```csharp
// Services/AppointmentService.cs
using Microsoft.EntityFrameworkCore;
using EmailReminderService.Data;
using EmailReminderService.Models;

namespace EmailReminderService.Services;

public class AppointmentService
{
    private readonly AppDbContext _db;

    public AppointmentService(AppDbContext db)
    {
        _db = db;
    }

    public async Task<List<Appointment>> GetAllAsync()
    {
        return await _db.Appointments
            .OrderBy(a => a.ScheduledAt)
            .ToListAsync();
    }

    public async Task<Appointment?> GetByIdAsync(int id)
    {
        return await _db.Appointments.FindAsync(id);
    }

    public async Task<Appointment> CreateAsync(Appointment appointment)
    {
        appointment.CreatedAt = DateTime.UtcNow;
        appointment.ReminderSent = false;

        _db.Appointments.Add(appointment);
        await _db.SaveChangesAsync();

        return appointment;
    }

    public async Task<bool> UpdateAsync(Appointment appointment)
    {
        var existing = await _db.Appointments.FindAsync(appointment.Id);
        if (existing == null) return false;

        existing.Title = appointment.Title;
        existing.Description = appointment.Description;
        existing.ScheduledAt = appointment.ScheduledAt;

        // ถ้าเปลี่ยนเวลา ต้องส่ง reminder ใหม่
        if (existing.ScheduledAt != appointment.ScheduledAt)
        {
            existing.ReminderSent = false;
            existing.ReminderSentAt = null;
        }

        await _db.SaveChangesAsync();
        return true;
    }

    public async Task<bool> DeleteAsync(int id)
    {
        var appointment = await _db.Appointments.FindAsync(id);
        if (appointment == null) return false;

        _db.Appointments.Remove(appointment);
        await _db.SaveChangesAsync();
        return true;
    }

    public async Task<List<Appointment>> GetUpcomingAsync(int days = 7)
    {
        var now = DateTime.UtcNow;
        var deadline = now.AddDays(days);

        return await _db.Appointments
            .Where(a => a.ScheduledAt >= now && a.ScheduledAt <= deadline)
            .OrderBy(a => a.ScheduledAt)
            .ToListAsync();
    }
}
```

### API Controller

```csharp
// Controllers/AppointmentsController.cs
using Microsoft.AspNetCore.Mvc;
using EmailReminderService.Models;
using EmailReminderService.Services;

namespace EmailReminderService.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AppointmentsController : ControllerBase
{
    private readonly AppointmentService _service;

    public AppointmentsController(AppointmentService service)
    {
        _service = service;
    }

    [HttpGet]
    public async Task<ActionResult<List<Appointment>>> GetAll()
    {
        return await _service.GetAllAsync();
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Appointment>> GetById(int id)
    {
        var appointment = await _service.GetByIdAsync(id);
        if (appointment == null) return NotFound();
        return appointment;
    }

    [HttpPost]
    public async Task<ActionResult<Appointment>> Create(Appointment appointment)
    {
        if (appointment.ScheduledAt <= DateTime.UtcNow)
            return BadRequest("เวลานัดหมายต้องเป็นเวลาในอนาคต");

        var created = await _service.CreateAsync(appointment);
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, Appointment appointment)
    {
        if (id != appointment.Id) return BadRequest();

        var updated = await _service.UpdateAsync(appointment);
        if (!updated) return NotFound();

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var deleted = await _service.DeleteAsync(id);
        if (!deleted) return NotFound();

        return NoContent();
    }

    [HttpGet("upcoming")]
    public async Task<ActionResult<List<Appointment>>> GetUpcoming([FromQuery] int days = 7)
    {
        return await _service.GetUpcomingAsync(days);
    }
}
```

### Program.cs

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;
using EmailReminderService.Data;
using EmailReminderService.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=appointments.db"));

builder.Services.AddScoped<AppointmentService>();
builder.Services.AddScoped<IEmailService, EmailService>();

// ลงทะเบียน Background Service
builder.Services.AddHostedService<EmailReminderBackgroundService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// สร้าง database
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### appsettings.json

```json
{
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": "587",
    "SenderName": "Appointment Reminder",
    "SenderEmail": "reminder@example.com",
    "Username": "your-email@gmail.com",
    "Password": "your-app-password"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=appointments.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "EmailReminderService.Services.EmailReminderBackgroundService": "Debug"
    }
  }
}
```

---

## Exercises

### Exercise 1: Retry Policy
เพิ่ม retry policy ใน EmailReminderBackgroundService เมื่อส่งอีเมลไม่สำเร็จ ให้ลองใหม่อีก 3 ครั้งโดยมี exponential backoff

### Exercise 2: Health Check Integration
เพิ่ม health check สำหรับ background service โดยรายงานสถานะล่าสุดของการทำงาน

### Exercise 3: Multiple Reminders
เพิ่มฟีเจอร์ส่ง reminder หลายครั้ง เช่น 1 สัปดาห์ล่วงหน้า, 24 ชั่วโมง, และ 1 ชั่วโมงก่อนถึงเวลา

### Exercise 4: Queue-based Processing
ใช้ `Channel<T>` เพื่อสร้าง producer-consumer pattern แทนการ polling database

### Exercise 5: Cancellation Handling
ปรับปรุง service ให้ handle cancellation อย่างถูกต้อง ให้ task ที่กำลังทำงานอยู่เสร็จสมบูรณ์ก่อนหยุด

---

## สรุป

- **IHostedService** เป็น interface พื้นฐานสำหรับ background services
- **BackgroundService** เป็น abstract class ที่ช่วยลด boilerplate code
- ใช้ **IServiceScopeFactory** เพื่อเข้าถึง Scoped services จาก Singleton background service
- **Worker Service** template เหมาะสำหรับ standalone background processing
- ใช้ **CancellationToken** เสมอใน async operations
- จัดการ exceptions ให้ดี เพื่อป้องกัน service crash

---

## Part ถัดไป

**Part 063: Caching ใน ASP.NET Core** - เรียนรู้การใช้ In-memory cache, Distributed cache, และ Redis

---

*Part 062/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
