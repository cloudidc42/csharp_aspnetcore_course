# Part 067: File Upload/Download

## เนื้อหาใน Part นี้
- IFormFile - การรับไฟล์จาก client
- Multiple File Upload
- File Validation
- Streaming Large Files
- Azure Blob Storage เบื้องต้น
- โปรแกรมตัวอย่าง: Image Upload Service

---

## 1. IFormFile - การรับไฟล์พื้นฐาน

**IFormFile** เป็น interface หลักสำหรับรับไฟล์ที่ถูก upload ผ่าน multipart/form-data

### การรับไฟล์เดียว

```csharp
// Controllers/FilesController.cs
using Microsoft.AspNetCore.Mvc;

namespace FileUploadDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class FilesController : ControllerBase
{
    private readonly IWebHostEnvironment _env;
    private readonly ILogger<FilesController> _logger;

    public FilesController(IWebHostEnvironment env, ILogger<FilesController> logger)
    {
        _env = env;
        _logger = logger;
    }

    [HttpPost("upload")]
    public async Task<IActionResult> Upload(IFormFile file)
    {
        if (file == null || file.Length == 0)
            return BadRequest("ไม่มีไฟล์");

        _logger.LogInformation(
            "Uploading file: {FileName}, Size: {Size}, ContentType: {ContentType}",
            file.FileName,
            file.Length,
            file.ContentType);

        // สร้าง unique filename
        var uniqueFileName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
        var uploadsFolder = Path.Combine(_env.ContentRootPath, "uploads");

        Directory.CreateDirectory(uploadsFolder);

        var filePath = Path.Combine(uploadsFolder, uniqueFileName);

        using var stream = new FileStream(filePath, FileMode.Create);
        await file.CopyToAsync(stream);

        return Ok(new
        {
            FileName = uniqueFileName,
            OriginalName = file.FileName,
            Size = file.Length,
            ContentType = file.ContentType,
            Url = $"/uploads/{uniqueFileName}"
        });
    }
}
```

### ข้อมูลใน IFormFile

```csharp
public interface IFormFile
{
    string ContentType { get; }          // "image/jpeg"
    string ContentDisposition { get; }  // "form-data; name=\"file\"; filename=\"photo.jpg\""
    IHeaderDictionary Headers { get; }
    long Length { get; }                 // ขนาดไฟล์ (bytes)
    string Name { get; }                 // ชื่อ form field
    string FileName { get; }            // ชื่อไฟล์ต้นฉบับ
    Stream OpenReadStream();             // เปิด stream สำหรับอ่าน
    void CopyTo(Stream target);
    Task CopyToAsync(Stream target, CancellationToken cancellationToken = default);
}
```

---

## 2. Multiple File Upload

### รับหลายไฟล์พร้อมกัน

```csharp
[HttpPost("upload-multiple")]
public async Task<IActionResult> UploadMultiple(List<IFormFile> files)
{
    if (files == null || !files.Any())
        return BadRequest("ไม่มีไฟล์");

    var results = new List<object>();
    var uploadsFolder = Path.Combine(_env.ContentRootPath, "uploads");
    Directory.CreateDirectory(uploadsFolder);

    foreach (var file in files)
    {
        if (file.Length > 0)
        {
            var uniqueFileName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";
            var filePath = Path.Combine(uploadsFolder, uniqueFileName);

            await using var stream = new FileStream(filePath, FileMode.Create);
            await file.CopyToAsync(stream);

            results.Add(new
            {
                OriginalName = file.FileName,
                SavedAs = uniqueFileName,
                Size = file.Length
            });
        }
    }

    return Ok(new { Uploaded = results.Count, Files = results });
}

// รับ mixed form data (ไฟล์ + ข้อมูล)
[HttpPost("upload-with-data")]
public async Task<IActionResult> UploadWithData(
    [FromForm] string title,
    [FromForm] string description,
    IFormFile thumbnail,
    List<IFormFile>? attachments)
{
    var result = new
    {
        Title = title,
        Description = description,
        ThumbnailUploaded = thumbnail != null,
        AttachmentsCount = attachments?.Count ?? 0
    };

    // Process files...

    return Ok(result);
}
```

### Request สำหรับ Multiple Files

```csharp
// DTOs/UploadRequest.cs
using Microsoft.AspNetCore.Http;
using System.ComponentModel.DataAnnotations;

namespace FileUploadDemo.DTOs;

public class ProductUploadRequest
{
    [Required]
    public string ProductName { get; set; } = string.Empty;

    [Required]
    public IFormFile MainImage { get; set; } = null!;

    public List<IFormFile>? GalleryImages { get; set; }

    public IFormFile? Manual { get; set; }
}

// Controller
[HttpPost("products")]
public async Task<IActionResult> CreateProduct([FromForm] ProductUploadRequest request)
{
    // ใช้ request.MainImage, request.GalleryImages, etc.
    return Ok();
}
```

---

## 3. File Validation

### Validation Service

```csharp
// Services/FileValidationService.cs
namespace FileUploadDemo.Services;

public class FileValidationResult
{
    public bool IsValid { get; set; }
    public List<string> Errors { get; set; } = new();
}

public interface IFileValidationService
{
    Task<FileValidationResult> ValidateImageAsync(IFormFile file);
    Task<FileValidationResult> ValidateDocumentAsync(IFormFile file);
    Task<FileValidationResult> ValidateAsync(IFormFile file, FileValidationOptions options);
}

public class FileValidationOptions
{
    public long MaxSizeBytes { get; set; } = 10 * 1024 * 1024; // 10MB default
    public List<string> AllowedExtensions { get; set; } = new();
    public List<string> AllowedContentTypes { get; set; } = new();
    public bool ValidateMagicBytes { get; set; } = true;
}

public class FileValidationService : IFileValidationService
{
    // Magic bytes สำหรับตรวจสอบไฟล์จริงๆ (ไม่ใช่แค่ extension)
    private static readonly Dictionary<string, byte[]> MagicBytes = new()
    {
        { "jpg", new byte[] { 0xFF, 0xD8, 0xFF } },
        { "png", new byte[] { 0x89, 0x50, 0x4E, 0x47 } },
        { "gif", new byte[] { 0x47, 0x49, 0x46 } },
        { "webp", new byte[] { 0x52, 0x49, 0x46, 0x46 } },
        { "pdf", new byte[] { 0x25, 0x50, 0x44, 0x46 } },
        { "zip", new byte[] { 0x50, 0x4B, 0x03, 0x04 } },
    };

    public async Task<FileValidationResult> ValidateImageAsync(IFormFile file)
    {
        return await ValidateAsync(file, new FileValidationOptions
        {
            MaxSizeBytes = 5 * 1024 * 1024, // 5MB
            AllowedExtensions = new() { ".jpg", ".jpeg", ".png", ".gif", ".webp" },
            AllowedContentTypes = new() { "image/jpeg", "image/png", "image/gif", "image/webp" },
            ValidateMagicBytes = true
        });
    }

    public async Task<FileValidationResult> ValidateDocumentAsync(IFormFile file)
    {
        return await ValidateAsync(file, new FileValidationOptions
        {
            MaxSizeBytes = 20 * 1024 * 1024, // 20MB
            AllowedExtensions = new() { ".pdf", ".doc", ".docx", ".xlsx" },
            AllowedContentTypes = new() {
                "application/pdf",
                "application/msword",
                "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
            }
        });
    }

    public async Task<FileValidationResult> ValidateAsync(IFormFile file, FileValidationOptions options)
    {
        var result = new FileValidationResult { IsValid = true };

        // ตรวจสอบ null
        if (file == null || file.Length == 0)
        {
            result.IsValid = false;
            result.Errors.Add("ไม่มีไฟล์หรือไฟล์ว่าง");
            return result;
        }

        // ตรวจสอบขนาด
        if (file.Length > options.MaxSizeBytes)
        {
            result.Errors.Add(
                $"ไฟล์ขนาดใหญ่เกินไป: {file.Length / 1024 / 1024}MB " +
                $"(สูงสุด: {options.MaxSizeBytes / 1024 / 1024}MB)");
        }

        // ตรวจสอบ extension
        var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
        if (options.AllowedExtensions.Any() && !options.AllowedExtensions.Contains(extension))
        {
            result.Errors.Add(
                $"ประเภทไฟล์ไม่ได้รับอนุญาต: {extension} " +
                $"(อนุญาต: {string.Join(", ", options.AllowedExtensions)})");
        }

        // ตรวจสอบ Content-Type
        if (options.AllowedContentTypes.Any() && !options.AllowedContentTypes.Contains(file.ContentType))
        {
            result.Errors.Add($"Content-Type ไม่ถูกต้อง: {file.ContentType}");
        }

        // ตรวจสอบ Magic Bytes
        if (options.ValidateMagicBytes && result.Errors.Count == 0)
        {
            var isValidMagicBytes = await ValidateMagicBytesAsync(file, extension);
            if (!isValidMagicBytes)
            {
                result.Errors.Add("เนื้อหาไฟล์ไม่ตรงกับประเภทที่ระบุ (อาจเป็นไฟล์ปลอม)");
            }
        }

        result.IsValid = !result.Errors.Any();
        return result;
    }

    private async Task<bool> ValidateMagicBytesAsync(IFormFile file, string extension)
    {
        var ext = extension.TrimStart('.');

        if (!MagicBytes.TryGetValue(ext, out var expectedBytes))
            return true; // ถ้าไม่มี magic bytes ให้ผ่านไป

        var buffer = new byte[expectedBytes.Length];
        await using var stream = file.OpenReadStream();

        var bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);

        if (bytesRead < expectedBytes.Length)
            return false;

        return buffer.Take(expectedBytes.Length).SequenceEqual(expectedBytes);
    }
}
```

---

## 4. Streaming Large Files

### Upload ไฟล์ขนาดใหญ่ด้วย Streaming

```csharp
// Program.cs - ปรับ request limits
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 500 * 1024 * 1024; // 500MB
    options.ValueLengthLimit = int.MaxValue;
    options.MemoryBufferThreshold = int.MaxValue;
});

builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.Limits.MaxRequestBodySize = 500 * 1024 * 1024; // 500MB
});
```

```csharp
// Controllers/LargeFileController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.WebUtilities;
using Microsoft.Net.Http.Headers;

namespace FileUploadDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
public class LargeFileController : ControllerBase
{
    private readonly IWebHostEnvironment _env;

    public LargeFileController(IWebHostEnvironment env)
    {
        _env = env;
    }

    // Upload ด้วย streaming (ไม่ buffer ทั้งหมดในหน่วยความจำ)
    [HttpPost("stream-upload")]
    [DisableFormValueModelBinding]
    [RequestSizeLimit(500_000_000)] // 500MB
    public async Task<IActionResult> StreamUpload()
    {
        if (!MultipartRequestHelper.IsMultipartContentType(Request.ContentType))
            return BadRequest("Expected multipart request");

        var boundary = MultipartRequestHelper.GetBoundary(
            MediaTypeHeaderValue.Parse(Request.ContentType),
            lengthLimit: 70);

        var reader = new MultipartReader(boundary, HttpContext.Request.Body);
        var section = await reader.ReadNextSectionAsync();

        string? savedFileName = null;

        while (section != null)
        {
            if (ContentDispositionHeaderValue.TryParse(
                section.ContentDisposition, out var contentDisposition))
            {
                if (MultipartRequestHelper.HasFileContentDisposition(contentDisposition))
                {
                    var fileName = contentDisposition.FileName.Value
                        ?? contentDisposition.FileNameStar.Value
                        ?? "unknown";

                    fileName = Path.GetFileName(fileName); // ป้องกัน path traversal
                    var uniqueName = $"{Guid.NewGuid()}{Path.GetExtension(fileName)}";
                    var savePath = Path.Combine(_env.ContentRootPath, "uploads", uniqueName);

                    Directory.CreateDirectory(Path.GetDirectoryName(savePath)!);

                    await using var stream = new FileStream(
                        savePath,
                        FileMode.Create,
                        FileAccess.Write,
                        FileShare.None,
                        bufferSize: 4096,
                        useAsync: true);

                    await section.Body.CopyToAsync(stream);
                    savedFileName = uniqueName;
                }
            }

            section = await reader.ReadNextSectionAsync();
        }

        if (savedFileName == null)
            return BadRequest("No file found in request");

        return Ok(new { FileName = savedFileName, Url = $"/uploads/{savedFileName}" });
    }

    // Download ด้วย streaming
    [HttpGet("download/{fileName}")]
    public IActionResult Download(string fileName)
    {
        // ป้องกัน path traversal
        fileName = Path.GetFileName(fileName);
        var filePath = Path.Combine(_env.ContentRootPath, "uploads", fileName);

        if (!System.IO.File.Exists(filePath))
            return NotFound();

        var contentType = GetContentType(fileName);
        var fileStream = new FileStream(
            filePath,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            bufferSize: 4096,
            useAsync: true);

        return File(fileStream, contentType, fileName);
    }

    // Download พร้อม progress (Range requests)
    [HttpGet("download-range/{fileName}")]
    public IActionResult DownloadWithRange(string fileName)
    {
        fileName = Path.GetFileName(fileName);
        var filePath = Path.Combine(_env.ContentRootPath, "uploads", fileName);

        if (!System.IO.File.Exists(filePath))
            return NotFound();

        var contentType = GetContentType(fileName);

        // PhysicalFile รองรับ Range requests อัตโนมัติ
        return PhysicalFile(filePath, contentType, enableRangeProcessing: true);
    }

    private string GetContentType(string fileName)
    {
        var extension = Path.GetExtension(fileName).ToLowerInvariant();
        return extension switch
        {
            ".jpg" or ".jpeg" => "image/jpeg",
            ".png" => "image/png",
            ".gif" => "image/gif",
            ".pdf" => "application/pdf",
            ".mp4" => "video/mp4",
            ".txt" => "text/plain",
            _ => "application/octet-stream"
        };
    }
}

// Helper class
public static class MultipartRequestHelper
{
    public static bool IsMultipartContentType(string? contentType)
        => !string.IsNullOrEmpty(contentType) && contentType.Contains("multipart/", StringComparison.OrdinalIgnoreCase);

    public static string GetBoundary(MediaTypeHeaderValue contentType, int lengthLimit)
    {
        var boundary = HeaderUtilities.RemoveQuotes(contentType.Boundary).Value;
        if (string.IsNullOrWhiteSpace(boundary))
            throw new InvalidDataException("Missing content-type boundary.");
        if (boundary.Length > lengthLimit)
            throw new InvalidDataException($"Multipart boundary too long.");
        return boundary;
    }

    public static bool HasFileContentDisposition(ContentDispositionHeaderValue? contentDisposition)
        => contentDisposition != null
            && contentDisposition.DispositionType.Equals("form-data")
            && (!string.IsNullOrEmpty(contentDisposition.FileName.Value)
                || !string.IsNullOrEmpty(contentDisposition.FileNameStar.Value));
}
```

---

## 5. Azure Blob Storage เบื้องต้น

```bash
dotnet add package Azure.Storage.Blobs
```

### Azure Blob Storage Service

```csharp
// Services/AzureBlobStorageService.cs
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Storage.Sas;

namespace FileUploadDemo.Services;

public interface IStorageService
{
    Task<string> UploadAsync(IFormFile file, string? folder = null);
    Task<string> UploadAsync(Stream stream, string fileName, string contentType, string? folder = null);
    Task DeleteAsync(string blobName);
    Task<Stream> DownloadAsync(string blobName);
    Task<string> GenerateSasUrlAsync(string blobName, TimeSpan expiry);
    Task<bool> ExistsAsync(string blobName);
}

public class AzureBlobStorageService : IStorageService
{
    private readonly BlobContainerClient _containerClient;
    private readonly ILogger<AzureBlobStorageService> _logger;

    public AzureBlobStorageService(
        IConfiguration configuration,
        ILogger<AzureBlobStorageService> logger)
    {
        _logger = logger;

        var connectionString = configuration["Azure:StorageConnectionString"]
            ?? throw new InvalidOperationException("Azure storage connection string not configured");
        var containerName = configuration["Azure:ContainerName"] ?? "uploads";

        _containerClient = new BlobContainerClient(connectionString, containerName);
        _containerClient.CreateIfNotExists(PublicAccessType.None);
    }

    public async Task<string> UploadAsync(IFormFile file, string? folder = null)
    {
        var blobName = BuildBlobName(file.FileName, folder);

        await using var stream = file.OpenReadStream();
        return await UploadAsync(stream, blobName, file.ContentType, null);
    }

    public async Task<string> UploadAsync(
        Stream stream,
        string fileName,
        string contentType,
        string? folder = null)
    {
        var blobName = BuildBlobName(fileName, folder);
        var blobClient = _containerClient.GetBlobClient(blobName);

        var uploadOptions = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders
            {
                ContentType = contentType,
                CacheControl = "public, max-age=3600"
            },
            Metadata = new Dictionary<string, string>
            {
                { "UploadedAt", DateTime.UtcNow.ToString("O") },
                { "OriginalName", Path.GetFileName(fileName) }
            }
        };

        await blobClient.UploadAsync(stream, uploadOptions);

        _logger.LogInformation("Uploaded blob: {BlobName}", blobName);

        return blobName;
    }

    public async Task DeleteAsync(string blobName)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        await blobClient.DeleteIfExistsAsync(DeleteSnapshotsOption.IncludeSnapshots);
        _logger.LogInformation("Deleted blob: {BlobName}", blobName);
    }

    public async Task<Stream> DownloadAsync(string blobName)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        var response = await blobClient.DownloadStreamingAsync();
        return response.Value.Content;
    }

    public async Task<string> GenerateSasUrlAsync(string blobName, TimeSpan expiry)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);

        if (!await blobClient.ExistsAsync())
            throw new FileNotFoundException($"Blob not found: {blobName}");

        var sasUri = blobClient.GenerateSasUri(
            BlobSasPermissions.Read,
            DateTimeOffset.UtcNow.Add(expiry));

        return sasUri.ToString();
    }

    public async Task<bool> ExistsAsync(string blobName)
    {
        var blobClient = _containerClient.GetBlobClient(blobName);
        return await blobClient.ExistsAsync();
    }

    private string BuildBlobName(string fileName, string? folder)
    {
        var uniqueName = $"{Guid.NewGuid()}{Path.GetExtension(fileName)}";
        return folder != null ? $"{folder.TrimEnd('/')}/{uniqueName}" : uniqueName;
    }
}
```

---

## โปรแกรมตัวอย่าง: Image Upload Service

ระบบ upload และจัดการรูปภาพสำหรับ product catalog

### โครงสร้าง

```
ImageUploadService/
├── Models/
│   └── ProductImage.cs
├── Services/
│   ├── IStorageService.cs
│   ├── LocalStorageService.cs
│   ├── ImageProcessingService.cs
│   └── ProductImageService.cs
├── Controllers/
│   └── ImagesController.cs
└── Program.cs
```

### Image Processing Service

```csharp
// Services/ImageProcessingService.cs
// ต้องติดตั้ง: dotnet add package SixLabors.ImageSharp
using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Processing;
using SixLabors.ImageSharp.Formats.Jpeg;
using SixLabors.ImageSharp.Formats.Webp;

namespace ImageUploadService.Services;

public class ImageSize
{
    public int Width { get; set; }
    public int Height { get; set; }
    public string Suffix { get; set; } = string.Empty;
}

public class ImageProcessingService
{
    private readonly ILogger<ImageProcessingService> _logger;

    // กำหนด thumbnail sizes
    private static readonly List<ImageSize> ThumbnailSizes = new()
    {
        new() { Width = 150, Height = 150, Suffix = "thumb" },
        new() { Width = 400, Height = 400, Suffix = "medium" },
        new() { Width = 800, Height = 800, Suffix = "large" }
    };

    public ImageProcessingService(ILogger<ImageProcessingService> logger)
    {
        _logger = logger;
    }

    // Process รูปภาพ: resize, compress, generate thumbnails
    public async Task<ProcessedImageResult> ProcessImageAsync(
        Stream inputStream,
        string originalFileName)
    {
        using var image = await Image.LoadAsync(inputStream);

        var result = new ProcessedImageResult
        {
            OriginalWidth = image.Width,
            OriginalHeight = image.Height,
            Format = image.Metadata.DecodedImageFormat?.Name ?? "unknown"
        };

        var baseName = Path.GetFileNameWithoutExtension(originalFileName);
        var variants = new Dictionary<string, byte[]>();

        // สร้าง WebP version ของรูปต้นฉบับ
        var originalWebp = await ConvertToWebpAsync(image);
        variants["original"] = originalWebp;
        result.FileSizes["original"] = originalWebp.Length;

        // สร้าง thumbnails
        foreach (var size in ThumbnailSizes)
        {
            var thumbnail = await CreateThumbnailAsync(image, size.Width, size.Height);
            variants[size.Suffix] = thumbnail;
            result.FileSizes[size.Suffix] = thumbnail.Length;
        }

        result.Variants = variants;

        _logger.LogInformation(
            "Processed image {FileName}: {Width}x{Height} -> {VariantCount} variants",
            originalFileName, image.Width, image.Height, variants.Count);

        return result;
    }

    private async Task<byte[]> ConvertToWebpAsync(Image image)
    {
        using var ms = new MemoryStream();

        // Clone เพื่อไม่แก้ไข original
        using var clone = image.Clone(ctx => ctx);

        // Resize ถ้าใหญ่เกินไป
        if (clone.Width > 1920 || clone.Height > 1920)
        {
            clone.Mutate(x => x.Resize(new ResizeOptions
            {
                Size = new Size(1920, 1920),
                Mode = ResizeMode.Max
            }));
        }

        await clone.SaveAsWebpAsync(ms, new WebpEncoder { Quality = 85 });
        return ms.ToArray();
    }

    private async Task<byte[]> CreateThumbnailAsync(Image image, int width, int height)
    {
        using var ms = new MemoryStream();
        using var thumbnail = image.Clone(ctx =>
        {
            ctx.Resize(new ResizeOptions
            {
                Size = new Size(width, height),
                Mode = ResizeMode.Crop,
                Position = AnchorPositionMode.Center
            });
        });

        await thumbnail.SaveAsWebpAsync(ms, new WebpEncoder { Quality = 80 });
        return ms.ToArray();
    }
}

public class ProcessedImageResult
{
    public int OriginalWidth { get; set; }
    public int OriginalHeight { get; set; }
    public string Format { get; set; } = string.Empty;
    public Dictionary<string, byte[]> Variants { get; set; } = new();
    public Dictionary<string, int> FileSizes { get; set; } = new();
}
```

### Product Image Service

```csharp
// Services/ProductImageService.cs
using ImageUploadService.Models;
using Microsoft.EntityFrameworkCore;

namespace ImageUploadService.Services;

public class ProductImageService
{
    private readonly AppDbContext _db;
    private readonly IStorageService _storage;
    private readonly ImageProcessingService _imageProcessor;
    private readonly IFileValidationService _validator;
    private readonly ILogger<ProductImageService> _logger;

    public ProductImageService(
        AppDbContext db,
        IStorageService storage,
        ImageProcessingService imageProcessor,
        IFileValidationService validator,
        ILogger<ProductImageService> logger)
    {
        _db = db;
        _storage = storage;
        _imageProcessor = imageProcessor;
        _validator = validator;
        _logger = logger;
    }

    public async Task<ProductImage> UploadImageAsync(
        int productId,
        IFormFile file,
        bool isPrimary = false)
    {
        // Validate
        var validationResult = await _validator.ValidateImageAsync(file);
        if (!validationResult.IsValid)
            throw new ValidationException(string.Join(", ", validationResult.Errors));

        // Process image
        using var stream = file.OpenReadStream();
        var processed = await _imageProcessor.ProcessImageAsync(stream, file.FileName);

        // Upload all variants
        var uploadedUrls = new Dictionary<string, string>();

        foreach (var (variantName, imageBytes) in processed.Variants)
        {
            using var variantStream = new MemoryStream(imageBytes);
            var blobName = await _storage.UploadAsync(
                variantStream,
                $"{Path.GetFileNameWithoutExtension(file.FileName)}_{variantName}.webp",
                "image/webp",
                $"products/{productId}");

            uploadedUrls[variantName] = blobName;
        }

        // ถ้าเป็น primary image ให้ unset อันเก่า
        if (isPrimary)
        {
            var existingPrimary = await _db.ProductImages
                .Where(i => i.ProductId == productId && i.IsPrimary)
                .FirstOrDefaultAsync();

            if (existingPrimary != null)
                existingPrimary.IsPrimary = false;
        }

        // บันทึก metadata
        var productImage = new ProductImage
        {
            ProductId = productId,
            OriginalFileName = file.FileName,
            BlobName = uploadedUrls["original"],
            ThumbnailBlobName = uploadedUrls.GetValueOrDefault("thumb"),
            MediumBlobName = uploadedUrls.GetValueOrDefault("medium"),
            LargeBlobName = uploadedUrls.GetValueOrDefault("large"),
            ContentType = "image/webp",
            FileSizeBytes = processed.FileSizes.GetValueOrDefault("original", 0),
            Width = processed.OriginalWidth,
            Height = processed.OriginalHeight,
            IsPrimary = isPrimary,
            UploadedAt = DateTime.UtcNow
        };

        _db.ProductImages.Add(productImage);
        await _db.SaveChangesAsync();

        _logger.LogInformation(
            "Uploaded image {ImageId} for product {ProductId}",
            productImage.Id,
            productId);

        return productImage;
    }

    public async Task<bool> DeleteImageAsync(int imageId)
    {
        var image = await _db.ProductImages.FindAsync(imageId);
        if (image == null) return false;

        // ลบจาก storage
        var blobsToDelete = new[]
        {
            image.BlobName,
            image.ThumbnailBlobName,
            image.MediumBlobName,
            image.LargeBlobName
        }.Where(b => b != null);

        foreach (var blobName in blobsToDelete)
        {
            await _storage.DeleteAsync(blobName!);
        }

        _db.ProductImages.Remove(image);
        await _db.SaveChangesAsync();

        return true;
    }

    public async Task<string> GetImageUrlAsync(int imageId, string variant = "original")
    {
        var image = await _db.ProductImages.FindAsync(imageId);
        if (image == null) throw new FileNotFoundException();

        var blobName = variant switch
        {
            "thumb" or "thumbnail" => image.ThumbnailBlobName,
            "medium" => image.MediumBlobName,
            "large" => image.LargeBlobName,
            _ => image.BlobName
        } ?? image.BlobName;

        // Generate SAS URL (valid 1 hour)
        return await _storage.GenerateSasUrlAsync(blobName, TimeSpan.FromHours(1));
    }
}
```

### Images Controller

```csharp
// Controllers/ImagesController.cs
using Microsoft.AspNetCore.Mvc;
using ImageUploadService.Services;

namespace ImageUploadService.Controllers;

[ApiController]
[Route("api/products/{productId}/images")]
public class ImagesController : ControllerBase
{
    private readonly ProductImageService _imageService;

    public ImagesController(ProductImageService imageService)
    {
        _imageService = imageService;
    }

    [HttpPost]
    [RequestSizeLimit(10_000_000)] // 10MB
    public async Task<IActionResult> Upload(
        int productId,
        IFormFile file,
        [FromQuery] bool isPrimary = false)
    {
        try
        {
            var image = await _imageService.UploadImageAsync(productId, file, isPrimary);
            return CreatedAtAction(nameof(GetUrl), new { productId, imageId = image.Id }, new
            {
                image.Id,
                image.OriginalFileName,
                image.Width,
                image.Height,
                image.FileSizeBytes,
                image.IsPrimary
            });
        }
        catch (ValidationException ex)
        {
            return BadRequest(new { message = ex.Message });
        }
    }

    [HttpPost("batch")]
    [RequestSizeLimit(50_000_000)] // 50MB for batch
    public async Task<IActionResult> UploadBatch(
        int productId,
        List<IFormFile> files)
    {
        if (!files.Any()) return BadRequest("No files provided");

        var results = new List<object>();
        var errors = new List<object>();

        foreach (var file in files)
        {
            try
            {
                var image = await _imageService.UploadImageAsync(productId, file);
                results.Add(new { image.Id, image.OriginalFileName, Success = true });
            }
            catch (Exception ex)
            {
                errors.Add(new { file.FileName, Error = ex.Message });
            }
        }

        return Ok(new { Uploaded = results.Count, Errors = errors.Count, Results = results, Failures = errors });
    }

    [HttpGet("{imageId}/url")]
    public async Task<IActionResult> GetUrl(
        int productId,
        int imageId,
        [FromQuery] string variant = "original")
    {
        try
        {
            var url = await _imageService.GetImageUrlAsync(imageId, variant);
            return Ok(new { Url = url, ExpiresIn = "1 hour" });
        }
        catch (FileNotFoundException)
        {
            return NotFound();
        }
    }

    [HttpDelete("{imageId}")]
    public async Task<IActionResult> Delete(int productId, int imageId)
    {
        var deleted = await _imageService.DeleteImageAsync(imageId);
        if (!deleted) return NotFound();
        return NoContent();
    }
}
```

### Program.cs

```csharp
// Program.cs
using ImageUploadService.Services;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=images.db"));

// Storage service (local สำหรับ development, Azure สำหรับ production)
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddScoped<IStorageService, LocalStorageService>();
}
else
{
    builder.Services.AddSingleton<IStorageService, AzureBlobStorageService>();
}

builder.Services.AddScoped<IFileValidationService, FileValidationService>();
builder.Services.AddScoped<ImageProcessingService>();
builder.Services.AddScoped<ProductImageService>();

// ปรับ upload limits
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 100 * 1024 * 1024; // 100MB
});

builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.MaxRequestBodySize = 100 * 1024 * 1024;
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    // Serve static files สำหรับ local uploads
    app.UseStaticFiles();
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

---

## Exercises

### Exercise 1: Chunked Upload
สร้าง endpoint สำหรับ chunked upload เพื่อรองรับไฟล์ขนาดใหญ่มากๆ

### Exercise 2: Image Watermark
เพิ่มฟีเจอร์ใส่ watermark บนรูปภาพก่อนบันทึก

### Exercise 3: Virus Scanning
เพิ่ม virus scanning integration (เช่น ClamAV) ก่อนบันทึกไฟล์

### Exercise 4: Resumable Upload
สร้าง resumable upload protocol ที่สามารถ resume จากจุดที่หยุดได้

### Exercise 5: CDN Integration
ปรับ storage service ให้ใช้ CDN URL แทน direct blob URL

---

## สรุป

- **IFormFile** ใช้รับไฟล์จาก multipart form
- ตรวจสอบทั้ง extension, content-type, และ magic bytes เพื่อความปลอดภัย
- ใช้ **streaming** สำหรับไฟล์ขนาดใหญ่เพื่อประหยัด memory
- **Azure Blob Storage** เหมาะสำหรับ production ที่ต้องการ scalability
- ประมวลผลรูปภาพ (resize, compress) เพื่อลด storage และ bandwidth

---

## Part ถัดไป

**Part 068: Email Service** - เรียนรู้การส่งอีเมลด้วย MailKit, MimeKit, และ SendGrid

---

*Part 067/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
