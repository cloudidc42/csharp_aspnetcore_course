# Part 068: Email Service

## เนื้อหาใน Part นี้
- MailKit และ MimeKit - ส่งอีเมลผ่าน SMTP
- การตั้งค่า SMTP
- Template Emails ด้วย RazorLight
- SendGrid Integration
- โปรแกรมตัวอย่าง: Email Notification Service

---

## 1. MailKit และ MimeKit

**MailKit** เป็น .NET library สำหรับส่งและรับอีเมลที่ทรงพลังและ cross-platform ใช้ร่วมกับ **MimeKit** สำหรับสร้าง MIME messages

### การติดตั้ง

```bash
dotnet add package MailKit
dotnet add package MimeKit
```

### การส่งอีเมลพื้นฐาน

```csharp
// Services/BasicEmailService.cs
using MailKit.Net.Smtp;
using MailKit.Security;
using MimeKit;

namespace EmailService.Services;

public class BasicEmailService
{
    private readonly SmtpSettings _settings;
    private readonly ILogger<BasicEmailService> _logger;

    public BasicEmailService(SmtpSettings settings, ILogger<BasicEmailService> logger)
    {
        _settings = settings;
        _logger = logger;
    }

    public async Task SendAsync(
        string to,
        string subject,
        string htmlBody,
        string? textBody = null)
    {
        var message = new MimeMessage();

        // From
        message.From.Add(new MailboxAddress(_settings.SenderName, _settings.SenderEmail));

        // To (รองรับหลาย addresses)
        foreach (var email in to.Split(',', StringSplitOptions.RemoveEmptyEntries))
        {
            message.To.Add(new MailboxAddress(string.Empty, email.Trim()));
        }

        message.Subject = subject;

        // สร้าง body (HTML + Text fallback)
        var builder = new BodyBuilder
        {
            HtmlBody = htmlBody,
            TextBody = textBody ?? StripHtml(htmlBody)
        };

        message.Body = builder.ToMessageBody();

        await SendMessageAsync(message);
    }

    public async Task SendWithAttachmentsAsync(
        string to,
        string subject,
        string htmlBody,
        List<EmailAttachment> attachments)
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress(_settings.SenderName, _settings.SenderEmail));
        message.To.Add(new MailboxAddress(string.Empty, to));
        message.Subject = subject;

        var builder = new BodyBuilder { HtmlBody = htmlBody };

        foreach (var attachment in attachments)
        {
            if (attachment.Content != null)
            {
                builder.Attachments.Add(
                    attachment.FileName,
                    attachment.Content,
                    ContentType.Parse(attachment.ContentType));
            }
            else if (attachment.FilePath != null)
            {
                builder.Attachments.Add(attachment.FilePath);
            }
        }

        message.Body = builder.ToMessageBody();
        await SendMessageAsync(message);
    }

    private async Task SendMessageAsync(MimeMessage message)
    {
        using var client = new SmtpClient();

        try
        {
            await client.ConnectAsync(
                _settings.Host,
                _settings.Port,
                _settings.UseSsl ? SecureSocketOptions.SslOnConnect : SecureSocketOptions.StartTls);

            if (!string.IsNullOrEmpty(_settings.Username))
            {
                await client.AuthenticateAsync(_settings.Username, _settings.Password);
            }

            await client.SendAsync(message);
            await client.DisconnectAsync(true);

            _logger.LogInformation(
                "Email sent to {To}: {Subject}",
                message.To.ToString(),
                message.Subject);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to send email to {To}",
                message.To.ToString());
            throw;
        }
    }

    private string StripHtml(string html)
    {
        return System.Text.RegularExpressions.Regex.Replace(html, "<[^>]*>", string.Empty);
    }
}

public class EmailAttachment
{
    public string FileName { get; set; } = string.Empty;
    public byte[]? Content { get; set; }
    public string? FilePath { get; set; }
    public string ContentType { get; set; } = "application/octet-stream";
}
```

---

## 2. การตั้งค่า SMTP

### SmtpSettings Model

```csharp
// Settings/SmtpSettings.cs
namespace EmailService.Settings;

public class SmtpSettings
{
    public const string SectionName = "SmtpSettings";

    public string Host { get; set; } = string.Empty;
    public int Port { get; set; } = 587;
    public bool UseSsl { get; set; } = false;
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string SenderEmail { get; set; } = string.Empty;
    public string SenderName { get; set; } = string.Empty;
}
```

### appsettings.json

```json
{
  "SmtpSettings": {
    "Host": "smtp.gmail.com",
    "Port": 587,
    "UseSsl": false,
    "Username": "your-email@gmail.com",
    "Password": "your-app-password",
    "SenderEmail": "noreply@yourapp.com",
    "SenderName": "Your App Name"
  }
}
```

### การลงทะเบียนใน DI

```csharp
// Program.cs
builder.Services.Configure<SmtpSettings>(
    builder.Configuration.GetSection(SmtpSettings.SectionName));

builder.Services.AddScoped<IEmailService, EmailService>();
```

### ProviderSpecific Settings

```json
// Gmail
{
  "SmtpSettings": {
    "Host": "smtp.gmail.com",
    "Port": 587,
    "UseSsl": false
  }
}

// Outlook/Office365
{
  "SmtpSettings": {
    "Host": "smtp.office365.com",
    "Port": 587,
    "UseSsl": false
  }
}

// Amazon SES
{
  "SmtpSettings": {
    "Host": "email-smtp.us-east-1.amazonaws.com",
    "Port": 587,
    "UseSsl": false
  }
}

// Local Development (Mailhog)
{
  "SmtpSettings": {
    "Host": "localhost",
    "Port": 1025,
    "UseSsl": false,
    "Username": "",
    "Password": ""
  }
}
```

```bash
# รัน Mailhog สำหรับ development
docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog
# เปิด UI ที่ http://localhost:8025
```

---

## 3. Template Emails ด้วย RazorLight

**RazorLight** ให้เราใช้ Razor syntax ในการสร้าง email templates

### การติดตั้ง

```bash
dotnet add package RazorLight
```

### Email Templates

```html
<!-- Templates/WelcomeEmail.cshtml -->
@model EmailService.Models.WelcomeEmailModel
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ยินดีต้อนรับ</title>
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; background: #f4f4f4; margin: 0; padding: 0; }
        .container { max-width: 600px; margin: 20px auto; background: white; border-radius: 8px; overflow: hidden; }
        .header { background: #2c3e50; color: white; padding: 30px; text-align: center; }
        .content { padding: 30px; }
        .button { display: inline-block; background: #3498db; color: white; padding: 12px 30px;
                  text-decoration: none; border-radius: 4px; margin: 20px 0; }
        .footer { background: #f8f9fa; padding: 20px; text-align: center; color: #666; font-size: 12px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>@Model.AppName</h1>
            <p>ยินดีต้อนรับสู่ระบบ</p>
        </div>
        <div class="content">
            <h2>สวัสดีคุณ @Model.UserName</h2>
            <p>ขอบคุณที่สมัครสมาชิกกับ @Model.AppName</p>
            <p>บัญชีของคุณถูกสร้างเรียบร้อยแล้ว คลิกปุ่มด้านล่างเพื่อยืนยันอีเมล</p>

            <a href="@Model.ConfirmationUrl" class="button">ยืนยันอีเมล</a>

            <p style="color: #666; font-size: 14px;">
                ลิงก์นี้จะหมดอายุใน @Model.ExpiryHours ชั่วโมง
            </p>

            @if (!string.IsNullOrEmpty(Model.ReferralCode))
            {
                <div style="background: #e8f5e9; padding: 15px; border-radius: 4px; margin-top: 20px;">
                    <p><strong>รหัสแนะนำของคุณ: @Model.ReferralCode</strong></p>
                    <p>แชร์รหัสนี้เพื่อรับสิทธิพิเศษ</p>
                </div>
            }
        </div>
        <div class="footer">
            <p>หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้</p>
            <p>© @DateTime.Now.Year @Model.AppName. All rights reserved.</p>
        </div>
    </div>
</body>
</html>
```

```html
<!-- Templates/OrderConfirmation.cshtml -->
@model EmailService.Models.OrderConfirmationModel
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>ยืนยันคำสั่งซื้อ</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f4f4f4; }
        .container { max-width: 600px; margin: 20px auto; background: white; border-radius: 8px; }
        .header { background: #27ae60; color: white; padding: 30px; text-align: center; }
        .content { padding: 30px; }
        .order-table { width: 100%; border-collapse: collapse; }
        .order-table th, .order-table td { padding: 10px; text-align: left; border-bottom: 1px solid #eee; }
        .order-table th { background: #f8f9fa; }
        .total { font-weight: bold; font-size: 18px; text-align: right; margin-top: 15px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>✓ ยืนยันคำสั่งซื้อ</h1>
            <p>หมายเลขคำสั่งซื้อ: #@Model.OrderNumber</p>
        </div>
        <div class="content">
            <p>สวัสดีคุณ @Model.CustomerName</p>
            <p>เราได้รับคำสั่งซื้อของคุณเรียบร้อยแล้ว</p>

            <h3>รายการสินค้า</h3>
            <table class="order-table">
                <thead>
                    <tr>
                        <th>สินค้า</th>
                        <th>จำนวน</th>
                        <th>ราคา</th>
                        <th>รวม</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var item in Model.Items)
                    {
                        <tr>
                            <td>@item.ProductName</td>
                            <td>@item.Quantity</td>
                            <td>@item.UnitPrice.ToString("N2")</td>
                            <td>@(item.UnitPrice * item.Quantity).ToString("N2")</td>
                        </tr>
                    }
                </tbody>
            </table>

            <div class="total">
                ยอดรวม: ฿@Model.TotalAmount.ToString("N2")
            </div>

            <div style="margin-top: 20px; padding: 15px; background: #f8f9fa; border-radius: 4px;">
                <h4>ที่อยู่จัดส่ง</h4>
                <p>@Model.ShippingAddress.Name</p>
                <p>@Model.ShippingAddress.Street</p>
                <p>@Model.ShippingAddress.City @Model.ShippingAddress.PostalCode</p>
            </div>

            <p style="margin-top: 20px;">
                <a href="@Model.TrackingUrl"
                   style="background: #3498db; color: white; padding: 10px 20px;
                          text-decoration: none; border-radius: 4px;">
                    ติดตามสถานะ
                </a>
            </p>
        </div>
    </div>
</body>
</html>
```

### Template Service

```csharp
// Services/EmailTemplateService.cs
using RazorLight;

namespace EmailService.Services;

public interface IEmailTemplateService
{
    Task<string> RenderAsync<T>(string templateName, T model);
}

public class RazorEmailTemplateService : IEmailTemplateService
{
    private readonly RazorLightEngine _engine;

    public RazorEmailTemplateService(IWebHostEnvironment env)
    {
        var templatesPath = Path.Combine(env.ContentRootPath, "EmailTemplates");

        _engine = new RazorLightEngineBuilder()
            .UseFileSystemProject(templatesPath)
            .UseMemoryCachingProvider()
            .EnableDebugMode()
            .Build();
    }

    public async Task<string> RenderAsync<T>(string templateName, T model)
    {
        return await _engine.CompileRenderAsync($"{templateName}.cshtml", model);
    }
}
```

---

## 4. SendGrid Integration

**SendGrid** เป็น cloud email service ที่ reliable และ scalable

### การติดตั้ง

```bash
dotnet add package SendGrid
```

### SendGrid Service

```csharp
// Services/SendGridEmailService.cs
using SendGrid;
using SendGrid.Helpers.Mail;

namespace EmailService.Services;

public class SendGridEmailService : IEmailService
{
    private readonly SendGridClient _client;
    private readonly SendGridSettings _settings;
    private readonly IEmailTemplateService _templateService;
    private readonly ILogger<SendGridEmailService> _logger;

    public SendGridEmailService(
        SendGridSettings settings,
        IEmailTemplateService templateService,
        ILogger<SendGridEmailService> logger)
    {
        _client = new SendGridClient(settings.ApiKey);
        _settings = settings;
        _templateService = templateService;
        _logger = logger;
    }

    public async Task SendAsync(EmailMessage message)
    {
        var msg = new SendGridMessage
        {
            From = new EmailAddress(_settings.SenderEmail, _settings.SenderName),
            Subject = message.Subject,
        };

        msg.AddTo(new EmailAddress(message.To, message.ToName));

        if (!string.IsNullOrEmpty(message.HtmlContent))
            msg.HtmlContent = message.HtmlContent;

        if (!string.IsNullOrEmpty(message.TextContent))
            msg.PlainTextContent = message.TextContent;

        // เพิ่ม CC และ BCC
        if (message.CcAddresses.Any())
        {
            foreach (var cc in message.CcAddresses)
                msg.AddCc(new EmailAddress(cc));
        }

        if (message.BccAddresses.Any())
        {
            foreach (var bcc in message.BccAddresses)
                msg.AddBcc(new EmailAddress(bcc));
        }

        // Attachments
        foreach (var attachment in message.Attachments)
        {
            msg.AddAttachment(
                attachment.FileName,
                Convert.ToBase64String(attachment.Content!),
                attachment.ContentType);
        }

        // Custom headers
        msg.AddHeader("X-App-Name", _settings.AppName);

        try
        {
            var response = await _client.SendEmailAsync(msg);

            if (response.IsSuccessStatusCode)
            {
                _logger.LogInformation(
                    "Email sent via SendGrid to {To}: {Subject}",
                    message.To,
                    message.Subject);
            }
            else
            {
                var body = await response.Body.ReadAsStringAsync();
                _logger.LogError(
                    "SendGrid error {StatusCode}: {Body}",
                    response.StatusCode,
                    body);
                throw new Exception($"SendGrid returned {response.StatusCode}");
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send email via SendGrid to {To}", message.To);
            throw;
        }
    }

    // ใช้ SendGrid Dynamic Templates
    public async Task SendTemplateAsync(
        string to,
        string templateId,
        object templateData)
    {
        var msg = new SendGridMessage();
        msg.SetFrom(new EmailAddress(_settings.SenderEmail, _settings.SenderName));
        msg.AddTo(new EmailAddress(to));
        msg.SetTemplateId(templateId);
        msg.SetTemplateData(templateData);

        var response = await _client.SendEmailAsync(msg);

        if (!response.IsSuccessStatusCode)
        {
            var body = await response.Body.ReadAsStringAsync();
            throw new Exception($"SendGrid template error: {body}");
        }
    }
}
```

---

## โปรแกรมตัวอย่าง: Email Notification Service

ระบบส่ง notification emails ประเภทต่างๆ

### โครงสร้าง

```
EmailNotificationService/
├── EmailTemplates/
│   ├── WelcomeEmail.cshtml
│   ├── OrderConfirmation.cshtml
│   ├── PasswordReset.cshtml
│   └── PaymentReceipt.cshtml
├── Models/
│   ├── EmailMessage.cs
│   └── TemplateModels.cs
├── Services/
│   ├── IEmailService.cs
│   ├── SmtpEmailService.cs
│   └── NotificationService.cs
├── Settings/
│   └── EmailSettings.cs
└── Program.cs
```

### Email Service Interface

```csharp
// Services/IEmailService.cs
namespace EmailNotificationService.Services;

public interface IEmailService
{
    Task SendAsync(EmailMessage message);
    Task SendWelcomeAsync(string to, string userName, string confirmationUrl);
    Task SendOrderConfirmationAsync(string to, OrderConfirmationData order);
    Task SendPasswordResetAsync(string to, string resetUrl);
    Task SendPaymentReceiptAsync(string to, PaymentReceiptData payment);
}
```

### Complete Email Service

```csharp
// Services/SmtpEmailService.cs
using MailKit.Net.Smtp;
using MailKit.Security;
using MimeKit;
using Microsoft.Extensions.Options;

namespace EmailNotificationService.Services;

public class SmtpEmailService : IEmailService
{
    private readonly EmailSettings _settings;
    private readonly IEmailTemplateService _templates;
    private readonly ILogger<SmtpEmailService> _logger;

    public SmtpEmailService(
        IOptions<EmailSettings> settings,
        IEmailTemplateService templates,
        ILogger<SmtpEmailService> logger)
    {
        _settings = settings.Value;
        _templates = templates;
        _logger = logger;
    }

    public async Task SendAsync(EmailMessage message)
    {
        var mimeMessage = BuildMimeMessage(message);
        await SendMimeMessageAsync(mimeMessage);
    }

    public async Task SendWelcomeAsync(string to, string userName, string confirmationUrl)
    {
        var model = new WelcomeEmailModel
        {
            UserName = userName,
            ConfirmationUrl = confirmationUrl,
            AppName = _settings.AppName,
            ExpiryHours = 24
        };

        var htmlBody = await _templates.RenderAsync("WelcomeEmail", model);

        await SendAsync(new EmailMessage
        {
            To = to,
            Subject = $"ยินดีต้อนรับสู่ {_settings.AppName}",
            HtmlContent = htmlBody
        });
    }

    public async Task SendOrderConfirmationAsync(string to, OrderConfirmationData order)
    {
        var model = new OrderConfirmationModel
        {
            CustomerName = order.CustomerName,
            OrderNumber = order.OrderNumber,
            Items = order.Items.Select(i => new OrderItemModel
            {
                ProductName = i.ProductName,
                Quantity = i.Quantity,
                UnitPrice = i.UnitPrice
            }).ToList(),
            TotalAmount = order.TotalAmount,
            ShippingAddress = order.ShippingAddress,
            TrackingUrl = $"{_settings.AppBaseUrl}/orders/{order.OrderNumber}"
        };

        var htmlBody = await _templates.RenderAsync("OrderConfirmation", model);

        await SendAsync(new EmailMessage
        {
            To = to,
            Subject = $"ยืนยันคำสั่งซื้อ #{order.OrderNumber}",
            HtmlContent = htmlBody
        });
    }

    public async Task SendPasswordResetAsync(string to, string resetUrl)
    {
        var model = new PasswordResetModel
        {
            ResetUrl = resetUrl,
            AppName = _settings.AppName,
            ExpiryMinutes = 60
        };

        var htmlBody = await _templates.RenderAsync("PasswordReset", model);

        await SendAsync(new EmailMessage
        {
            To = to,
            Subject = "รีเซ็ตรหัสผ่าน",
            HtmlContent = htmlBody
        });
    }

    public async Task SendPaymentReceiptAsync(string to, PaymentReceiptData payment)
    {
        var model = new PaymentReceiptModel
        {
            CustomerName = payment.CustomerName,
            TransactionId = payment.TransactionId,
            Amount = payment.Amount,
            PaymentMethod = payment.PaymentMethod,
            PaidAt = payment.PaidAt,
            Items = payment.Items
        };

        var htmlBody = await _templates.RenderAsync("PaymentReceipt", model);

        await SendAsync(new EmailMessage
        {
            To = to,
            Subject = $"ใบเสร็จการชำระเงิน #{payment.TransactionId}",
            HtmlContent = htmlBody
        });
    }

    private MimeMessage BuildMimeMessage(EmailMessage message)
    {
        var mimeMessage = new MimeMessage();

        mimeMessage.From.Add(
            new MailboxAddress(_settings.SenderName, _settings.SenderEmail));

        mimeMessage.To.Add(
            new MailboxAddress(message.ToName ?? string.Empty, message.To));

        if (message.ReplyTo != null)
            mimeMessage.ReplyTo.Add(new MailboxAddress(string.Empty, message.ReplyTo));

        foreach (var cc in message.CcAddresses)
            mimeMessage.Cc.Add(new MailboxAddress(string.Empty, cc));

        foreach (var bcc in message.BccAddresses)
            mimeMessage.Bcc.Add(new MailboxAddress(string.Empty, bcc));

        mimeMessage.Subject = message.Subject;

        // Message ID สำหรับ tracking
        mimeMessage.MessageId = $"{Guid.NewGuid()}@{_settings.Domain}";

        // Build body
        var builder = new BodyBuilder();

        if (!string.IsNullOrEmpty(message.HtmlContent))
            builder.HtmlBody = message.HtmlContent;

        if (!string.IsNullOrEmpty(message.TextContent))
            builder.TextBody = message.TextContent;
        else if (!string.IsNullOrEmpty(message.HtmlContent))
            builder.TextBody = StripHtml(message.HtmlContent);

        foreach (var attachment in message.Attachments)
        {
            if (attachment.Content != null)
                builder.Attachments.Add(attachment.FileName, attachment.Content);
            else if (attachment.FilePath != null)
                builder.Attachments.Add(attachment.FilePath);
        }

        mimeMessage.Body = builder.ToMessageBody();
        return mimeMessage;
    }

    private async Task SendMimeMessageAsync(MimeMessage message)
    {
        using var client = new SmtpClient();

        try
        {
            var secureSocket = _settings.UseSsl
                ? SecureSocketOptions.SslOnConnect
                : SecureSocketOptions.StartTlsWhenAvailable;

            await client.ConnectAsync(_settings.Host, _settings.Port, secureSocket);

            if (!string.IsNullOrEmpty(_settings.Username))
                await client.AuthenticateAsync(_settings.Username, _settings.Password);

            await client.SendAsync(message);
            await client.DisconnectAsync(true);

            _logger.LogInformation(
                "Email sent successfully. To: {To}, Subject: {Subject}, MessageId: {MessageId}",
                message.To,
                message.Subject,
                message.MessageId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Failed to send email. To: {To}, Subject: {Subject}",
                message.To,
                message.Subject);
            throw;
        }
    }

    private static string StripHtml(string html)
    {
        return System.Text.RegularExpressions.Regex.Replace(html, "<[^>]*>", string.Empty);
    }
}
```

### Notification Service

```csharp
// Services/NotificationService.cs
using System.Threading.Channels;

namespace EmailNotificationService.Services;

// Queue-based email sending เพื่อไม่บล็อก request
public class EmailQueueService : BackgroundService
{
    private readonly Channel<EmailMessage> _queue;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<EmailQueueService> _logger;

    public EmailQueueService(IServiceScopeFactory scopeFactory, ILogger<EmailQueueService> logger)
    {
        _queue = Channel.CreateBounded<EmailMessage>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.Wait
        });
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    public async Task QueueEmailAsync(EmailMessage message, CancellationToken ct = default)
    {
        await _queue.Writer.WriteAsync(message, ct);
        _logger.LogDebug("Email queued for {To}: {Subject}", message.To, message.Subject);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in _queue.Reader.ReadAllAsync(stoppingToken))
        {
            try
            {
                await using var scope = _scopeFactory.CreateAsyncScope();
                var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
                await emailService.SendAsync(message);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex,
                    "Failed to send queued email to {To}",
                    message.To);
            }
        }
    }
}
```

### Email Controller

```csharp
// Controllers/NotificationsController.cs
using Microsoft.AspNetCore.Mvc;
using EmailNotificationService.Services;

namespace EmailNotificationService.Controllers;

[ApiController]
[Route("api/[controller]")]
public class NotificationsController : ControllerBase
{
    private readonly IEmailService _emailService;
    private readonly EmailQueueService _emailQueue;

    public NotificationsController(
        IEmailService emailService,
        EmailQueueService emailQueue)
    {
        _emailService = emailService;
        _emailQueue = emailQueue;
    }

    [HttpPost("welcome")]
    public async Task<IActionResult> SendWelcome([FromBody] WelcomeRequest request)
    {
        await _emailQueue.QueueEmailAsync(new EmailMessage
        {
            To = request.Email,
            Subject = "ยินดีต้อนรับ",
            // จะถูก render ใน background service
        });

        return Ok(new { message = "Email queued" });
    }

    [HttpPost("order-confirmation")]
    public async Task<IActionResult> SendOrderConfirmation(
        [FromBody] OrderConfirmationRequest request)
    {
        // ส่ง immediately (synchronous)
        await _emailService.SendOrderConfirmationAsync(request.Email, request.Order);
        return Ok(new { message = "Email sent" });
    }

    [HttpPost("test")]
    public async Task<IActionResult> SendTest([FromQuery] string to)
    {
        await _emailService.SendAsync(new EmailMessage
        {
            To = to,
            Subject = "Test Email",
            HtmlContent = "<h1>Test</h1><p>This is a test email from ASP.NET Core</p>"
        });

        return Ok(new { message = $"Test email sent to {to}" });
    }
}
```

### Program.cs

```csharp
// Program.cs
using EmailNotificationService.Services;
using EmailNotificationService.Settings;
using Microsoft.Extensions.Options;

var builder = WebApplication.CreateBuilder(args);

// Email settings
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection("EmailSettings"));

// Template service
builder.Services.AddSingleton<IEmailTemplateService, RazorEmailTemplateService>();

// Email service - เลือกตาม environment
if (builder.Environment.IsDevelopment())
{
    // ใช้ SmtpEmailService กับ Mailhog ใน development
    builder.Services.AddScoped<IEmailService, SmtpEmailService>();
}
else
{
    // ใช้ SendGrid ใน production
    builder.Services.AddScoped<IEmailService>(sp =>
    {
        var settings = sp.GetRequiredService<IOptions<EmailSettings>>().Value;
        var templates = sp.GetRequiredService<IEmailTemplateService>();
        var logger = sp.GetRequiredService<ILogger<SendGridEmailService>>();
        return new SendGridEmailService(settings.SendGridSettings!, templates, logger);
    });
}

// Email queue (background service)
builder.Services.AddSingleton<EmailQueueService>();
builder.Services.AddHostedService(sp => sp.GetRequiredService<EmailQueueService>());

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

---

## Exercises

### Exercise 1: Email Preview
สร้าง endpoint `/api/email-preview/{templateName}` ที่ render และแสดง HTML ของ email template

### Exercise 2: Bulk Email
สร้างฟีเจอร์ส่งอีเมลจำนวนมากพร้อมกัน โดยใช้ throttling เพื่อไม่ให้เกิน rate limit

### Exercise 3: Email Tracking
เพิ่ม pixel tracking เพื่อรู้ว่าใครเปิดอีเมลบ้าง และ click tracking สำหรับ links

### Exercise 4: Unsubscribe System
สร้างระบบ unsubscribe ที่ generate unique token และ handle unsubscribe requests

### Exercise 5: Email Templates Admin
สร้าง admin UI สำหรับแก้ไข email templates โดยไม่ต้อง redeploy application

---

## สรุป

- **MailKit/MimeKit** เป็น library ที่แนะนำสำหรับ SMTP email ใน .NET
- ใช้ **RazorLight** สร้าง HTML email templates ที่ dynamic
- **SendGrid** เหมาะสำหรับ high-volume email ใน production
- ใช้ **queue-based** approach เพื่อไม่ให้ email blocking request processing
- ทดสอบ email ด้วย **Mailhog** ใน development environment
- เพิ่ม **retry logic** สำหรับ transient failures

---

## Part ถัดไป

**Part 069: CORS และ Security Headers** - เรียนรู้การตั้งค่า security ใน ASP.NET Core

---

*Part 068/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
