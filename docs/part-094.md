# Part 094: Real-world Project: Social Platform

## เนื้อหาใน Part นี้
- User management และ Profile
- Posts, Comments, Likes
- Follow system
- Notifications
- Feed algorithm
- Real-time features

---

## 1. Project Overview

สร้าง Social Platform API ที่รองรับ:
- User profiles พร้อม bio และ avatar
- Posts (text/image) พร้อม hashtags
- Comments และ nested replies
- Like / Unlike
- Follow / Unfollow
- Notifications แบบ real-time
- Personalized feed

### Tech Stack
```
- ASP.NET Core 9 (Minimal API + SignalR)
- PostgreSQL + EF Core 9
- Redis (caching + pub/sub)
- Azure Blob Storage (images)
- MediatR + CQRS
- SignalR (real-time notifications)
```

---

## 2. Domain Models

```csharp
// Domain/Entities/User.cs
public class User
{
    public Guid Id { get; private set; }
    public string Username { get; private set; } = string.Empty;
    public string Email { get; private set; } = string.Empty;
    public string PasswordHash { get; private set; } = string.Empty;
    public string? DisplayName { get; private set; }
    public string? Bio { get; private set; }
    public string? AvatarUrl { get; private set; }
    public string? WebsiteUrl { get; private set; }
    public bool IsVerified { get; private set; }
    public bool IsPrivate { get; private set; }
    public int PostCount { get; private set; }
    public int FollowerCount { get; private set; }
    public int FollowingCount { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private User() { }

    public static User Create(string username, string email, string passwordHash)
    {
        if (string.IsNullOrWhiteSpace(username)) throw new DomainException("Username is required");
        if (username.Length < 3 || username.Length > 30) throw new DomainException("Username must be 3-30 chars");

        return new User
        {
            Id = Guid.NewGuid(),
            Username = username.ToLower(),
            Email = email.ToLower(),
            PasswordHash = passwordHash,
            DisplayName = username,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void UpdateProfile(string? displayName, string? bio, string? website, bool isPrivate)
    {
        DisplayName = displayName;
        Bio = bio;
        WebsiteUrl = website;
        IsPrivate = isPrivate;
    }

    public void UpdateAvatar(string avatarUrl) => AvatarUrl = avatarUrl;

    public void IncrementPostCount() => PostCount++;
    public void DecrementPostCount() => PostCount = Math.Max(0, PostCount - 1);
    public void IncrementFollowerCount() => FollowerCount++;
    public void DecrementFollowerCount() => FollowerCount = Math.Max(0, FollowerCount - 1);
    public void IncrementFollowingCount() => FollowingCount++;
    public void DecrementFollowingCount() => FollowingCount = Math.Max(0, FollowingCount - 1);
}

// Domain/Entities/Post.cs
public class Post
{
    public Guid Id { get; private set; }
    public Guid AuthorId { get; private set; }
    public string Content { get; private set; } = string.Empty;
    public List<string> MediaUrls { get; private set; } = new();
    public List<string> Hashtags { get; private set; } = new();
    public List<string> Mentions { get; private set; } = new();
    public int LikeCount { get; private set; }
    public int CommentCount { get; private set; }
    public int ShareCount { get; private set; }
    public bool IsDeleted { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private Post() { }

    public static Post Create(Guid authorId, string content, List<string>? mediaUrls = null)
    {
        if (string.IsNullOrWhiteSpace(content)) throw new DomainException("Content is required");
        if (content.Length > 2000) throw new DomainException("Content too long (max 2000 chars)");

        var post = new Post
        {
            Id = Guid.NewGuid(),
            AuthorId = authorId,
            Content = content,
            MediaUrls = mediaUrls ?? new(),
            CreatedAt = DateTime.UtcNow
        };

        post.ParseHashtags();
        post.ParseMentions();
        return post;
    }

    public void Delete()
    {
        IsDeleted = true;
        UpdatedAt = DateTime.UtcNow;
    }

    public void IncrementLike() => LikeCount++;
    public void DecrementLike() => LikeCount = Math.Max(0, LikeCount - 1);
    public void IncrementComment() => CommentCount++;
    public void DecrementComment() => CommentCount = Math.Max(0, CommentCount - 1);

    private void ParseHashtags()
    {
        Hashtags = Regex.Matches(Content, @"#(\w+)")
            .Select(m => m.Groups[1].Value.ToLower())
            .Distinct()
            .ToList();
    }

    private void ParseMentions()
    {
        Mentions = Regex.Matches(Content, @"@(\w+)")
            .Select(m => m.Groups[1].Value.ToLower())
            .Distinct()
            .ToList();
    }
}

// Domain/Entities/Follow.cs
public class Follow
{
    public Guid FollowerId { get; private set; }
    public Guid FolloweeId { get; private set; }
    public FollowStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private Follow() { }

    public static Follow Create(Guid followerId, Guid followeeId)
    {
        if (followerId == followeeId) throw new DomainException("Cannot follow yourself");

        return new Follow
        {
            FollowerId = followerId,
            FolloweeId = followeeId,
            Status = FollowStatus.Active,
            CreatedAt = DateTime.UtcNow
        };
    }
}

public enum FollowStatus { Active, Requested, Rejected }

// Domain/Entities/Notification.cs
public class Notification
{
    public Guid Id { get; private set; }
    public Guid RecipientId { get; private set; }
    public Guid? ActorId { get; private set; }
    public NotificationType Type { get; private set; }
    public string? ReferenceId { get; private set; }
    public string Message { get; private set; } = string.Empty;
    public bool IsRead { get; private set; }
    public DateTime CreatedAt { get; private set; }

    private Notification() { }

    public static Notification Create(
        Guid recipientId, Guid? actorId, NotificationType type,
        string message, string? referenceId = null)
    {
        return new Notification
        {
            Id = Guid.NewGuid(),
            RecipientId = recipientId,
            ActorId = actorId,
            Type = type,
            Message = message,
            ReferenceId = referenceId,
            CreatedAt = DateTime.UtcNow
        };
    }

    public void MarkAsRead() => IsRead = true;
}

public enum NotificationType
{
    NewFollower,
    PostLike,
    PostComment,
    CommentReply,
    Mention
}
```

---

## 3. Follow System

```csharp
// Application/Features/Follow/Commands/FollowUserCommand.cs
public record FollowUserCommand(Guid FollowerId, Guid FolloweeId) : IRequest<FollowResult>;

public class FollowUserCommandHandler : IRequestHandler<FollowUserCommand, FollowResult>
{
    private readonly IUserRepository _userRepo;
    private readonly IFollowRepository _followRepo;
    private readonly INotificationService _notificationService;

    public FollowUserCommandHandler(
        IUserRepository userRepo,
        IFollowRepository followRepo,
        INotificationService notificationService)
    {
        _userRepo = userRepo;
        _followRepo = followRepo;
        _notificationService = notificationService;
    }

    public async Task<FollowResult> Handle(FollowUserCommand cmd, CancellationToken ct)
    {
        var follower = await _userRepo.GetByIdAsync(cmd.FollowerId, ct)
            ?? throw new NotFoundException("Follower not found");
        var followee = await _userRepo.GetByIdAsync(cmd.FolloweeId, ct)
            ?? throw new NotFoundException("User not found");

        var existing = await _followRepo.GetFollowAsync(cmd.FollowerId, cmd.FolloweeId, ct);
        if (existing != null)
            return new FollowResult(false, "Already following");

        var follow = Follow.Create(cmd.FollowerId, cmd.FolloweeId);
        await _followRepo.SaveAsync(follow, ct);

        follower.IncrementFollowingCount();
        followee.IncrementFollowerCount();
        await _userRepo.UpdateAsync(follower, ct);
        await _userRepo.UpdateAsync(followee, ct);

        // Notify followee
        await _notificationService.SendNotificationAsync(new SendNotificationRequest
        {
            RecipientId = cmd.FolloweeId,
            ActorId = cmd.FollowerId,
            Type = NotificationType.NewFollower,
            Message = $"{follower.Username} started following you"
        });

        return new FollowResult(true, "Followed successfully");
    }
}

public record FollowResult(bool Success, string Message);
```

---

## 4. Feed Algorithm

```csharp
// Application/Features/Feed/Queries/GetFeedQuery.cs
public record GetFeedQuery(Guid UserId, int Page = 1, int PageSize = 20) : IRequest<List<PostDto>>;

public class GetFeedQueryHandler : IRequestHandler<GetFeedQuery, List<PostDto>>
{
    private readonly IFollowRepository _followRepo;
    private readonly IPostRepository _postRepo;
    private readonly ICacheService _cache;

    public GetFeedQueryHandler(
        IFollowRepository followRepo,
        IPostRepository postRepo,
        ICacheService cache)
    {
        _followRepo = followRepo;
        _postRepo = postRepo;
        _cache = cache;
    }

    public async Task<List<PostDto>> Handle(GetFeedQuery query, CancellationToken ct)
    {
        // Get following list (cached)
        var cacheKey = $"following:{query.UserId}";
        var followingIds = await _cache.GetAsync<List<Guid>>(cacheKey);
        if (followingIds == null)
        {
            followingIds = await _followRepo.GetFollowingIdsAsync(query.UserId, ct);
            await _cache.SetAsync(cacheKey, followingIds, TimeSpan.FromMinutes(10));
        }

        // Include own posts
        followingIds.Add(query.UserId);

        // Get posts with scoring
        var posts = await _postRepo.GetFeedPostsAsync(followingIds, query.Page, query.PageSize, ct);
        return posts.Select(PostDto.From).ToList();
    }
}

// Infrastructure/Repositories/PostRepository.cs  
public async Task<List<Post>> GetFeedPostsAsync(
    List<Guid> authorIds, int page, int pageSize, CancellationToken ct)
{
    return await _context.Posts
        .Where(p => authorIds.Contains(p.AuthorId) && !p.IsDeleted)
        .OrderByDescending(p => p.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Include(p => p.Author)
        .ToListAsync(ct);
}

// For more advanced feed with engagement scoring:
public async Task<List<FeedPost>> GetScoredFeedAsync(
    Guid userId, List<Guid> authorIds, int page, int pageSize, CancellationToken ct)
{
    var now = DateTime.UtcNow;
    
    // Raw SQL for performance with custom scoring
    var posts = await _context.Posts
        .FromSqlRaw(@"
            SELECT p.*, 
                   (p.like_count * 2 + p.comment_count * 3) as engagement_score,
                   EXTRACT(EPOCH FROM (NOW() - p.created_at)) / 3600 as hours_old
            FROM posts p
            WHERE p.author_id = ANY(@authorIds) AND p.is_deleted = false
            ORDER BY 
                (p.like_count * 2 + p.comment_count * 3) / 
                POWER(EXTRACT(EPOCH FROM (NOW() - p.created_at)) / 3600 + 2, 1.5) DESC
            LIMIT @pageSize OFFSET @offset",
            new NpgsqlParameter("authorIds", authorIds.ToArray()),
            new NpgsqlParameter("pageSize", pageSize),
            new NpgsqlParameter("offset", (page - 1) * pageSize))
        .ToListAsync(ct);

    return posts;
}
```

---

## 5. Real-time Notifications ด้วย SignalR

```csharp
// Infrastructure/Hubs/NotificationHub.cs
[Authorize]
public class NotificationHub : Hub
{
    private readonly IConnectionTracker _connectionTracker;

    public NotificationHub(IConnectionTracker connectionTracker)
    {
        _connectionTracker = connectionTracker;
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        await _connectionTracker.AddConnectionAsync(userId, Context.ConnectionId);
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user:{userId}");
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier!;
        await _connectionTracker.RemoveConnectionAsync(userId, Context.ConnectionId);
        await base.OnDisconnectedAsync(exception);
    }

    public async Task MarkAsRead(string notificationId)
    {
        var userId = Context.UserIdentifier!;
        // Handle mark as read
    }
}

// Application/Services/NotificationService.cs
public class NotificationService : INotificationService
{
    private readonly IHubContext<NotificationHub> _hubContext;
    private readonly INotificationRepository _notifRepo;
    private readonly ILogger<NotificationService> _logger;

    public NotificationService(
        IHubContext<NotificationHub> hubContext,
        INotificationRepository notifRepo,
        ILogger<NotificationService> logger)
    {
        _hubContext = hubContext;
        _notifRepo = notifRepo;
        _logger = logger;
    }

    public async Task SendNotificationAsync(SendNotificationRequest request)
    {
        // Save to DB
        var notification = Notification.Create(
            request.RecipientId,
            request.ActorId,
            request.Type,
            request.Message,
            request.ReferenceId);

        await _notifRepo.SaveAsync(notification, CancellationToken.None);

        // Push real-time
        var payload = new NotificationPayload
        {
            Id = notification.Id,
            Type = request.Type.ToString(),
            Message = request.Message,
            CreatedAt = notification.CreatedAt
        };

        await _hubContext.Clients
            .Group($"user:{request.RecipientId}")
            .SendAsync("NewNotification", payload);

        _logger.LogInformation(
            "Notification sent to {UserId}: {Type}", request.RecipientId, request.Type);
    }
}

public record SendNotificationRequest
{
    public Guid RecipientId { get; init; }
    public Guid? ActorId { get; init; }
    public NotificationType Type { get; init; }
    public string Message { get; init; } = string.Empty;
    public string? ReferenceId { get; init; }
}

public record NotificationPayload
{
    public Guid Id { get; init; }
    public string Type { get; init; } = string.Empty;
    public string Message { get; init; } = string.Empty;
    public DateTime CreatedAt { get; init; }
}
```

---

## 6. API Endpoints

```csharp
// Endpoints/SocialEndpoints.cs
public static class SocialEndpoints
{
    public static void MapSocialEndpoints(this IEndpointRouteBuilder app)
    {
        var users = app.MapGroup("/api/users").WithTags("Users");
        users.MapGet("/{username}", GetProfile);
        users.MapGet("/{username}/posts", GetUserPosts);
        users.MapGet("/{username}/followers", GetFollowers);
        users.MapGet("/{username}/following", GetFollowing);
        users.MapPost("/{username}/follow", FollowUser).RequireAuthorization();
        users.MapDelete("/{username}/follow", UnfollowUser).RequireAuthorization();

        var feed = app.MapGroup("/api/feed").RequireAuthorization().WithTags("Feed");
        feed.MapGet("/", GetFeed);
        feed.MapGet("/trending", GetTrending);
        feed.MapGet("/hashtag/{tag}", GetHashtagFeed);

        var posts = app.MapGroup("/api/posts").WithTags("Posts");
        posts.MapPost("/", CreatePost).RequireAuthorization();
        posts.MapGet("/{id:guid}", GetPost);
        posts.MapDelete("/{id:guid}", DeletePost).RequireAuthorization();
        posts.MapPost("/{id:guid}/like", LikePost).RequireAuthorization();
        posts.MapDelete("/{id:guid}/like", UnlikePost).RequireAuthorization();
        posts.MapGet("/{id:guid}/comments", GetComments);
        posts.MapPost("/{id:guid}/comments", AddComment).RequireAuthorization();

        var notifications = app.MapGroup("/api/notifications")
            .RequireAuthorization().WithTags("Notifications");
        notifications.MapGet("/", GetNotifications);
        notifications.MapPost("/{id:guid}/read", MarkAsRead);
        notifications.MapPost("/read-all", MarkAllAsRead);
    }

    private static async Task<IResult> CreatePost(
        CreatePostRequest request,
        HttpContext ctx,
        ISender sender)
    {
        var userId = ctx.GetUserId();
        var post = await sender.Send(new CreatePostCommand(userId, request.Content, request.MediaUrls));
        return Results.Created($"/api/posts/{post.Id}", post);
    }

    private static async Task<IResult> FollowUser(
        string username,
        HttpContext ctx,
        ISender sender)
    {
        var followerId = ctx.GetUserId();
        var result = await sender.Send(new FollowUserByUsernameCommand(followerId, username));
        return result.Success ? Results.Ok(result) : Results.Conflict(result);
    }

    private static async Task<IResult> GetFeed(
        HttpContext ctx,
        ISender sender,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20)
    {
        var userId = ctx.GetUserId();
        var posts = await sender.Send(new GetFeedQuery(userId, page, pageSize));
        return Results.Ok(posts);
    }

    private static async Task<IResult> LikePost(
        Guid id,
        HttpContext ctx,
        ISender sender)
    {
        var userId = ctx.GetUserId();
        await sender.Send(new LikePostCommand(userId, id));
        return Results.Ok();
    }
}

// Endpoints/MediaEndpoints.cs
public static class MediaEndpoints
{
    public static void MapMediaEndpoints(this IEndpointRouteBuilder app)
    {
        var media = app.MapGroup("/api/media")
            .RequireAuthorization()
            .WithTags("Media");

        media.MapPost("/avatar", UploadAvatar);
        media.MapPost("/post-image", UploadPostImage);
    }

    private static async Task<IResult> UploadAvatar(
        IFormFile file,
        HttpContext ctx,
        IMediaUploadService uploadService,
        ISender sender)
    {
        if (file.Length > 5 * 1024 * 1024)
            return Results.BadRequest("File too large (max 5MB)");

        var allowed = new[] { "image/jpeg", "image/png", "image/webp" };
        if (!allowed.Contains(file.ContentType))
            return Results.BadRequest("Only JPEG, PNG, WebP allowed");

        var userId = ctx.GetUserId();
        
        await using var stream = file.OpenReadStream();
        var url = await uploadService.UploadAvatarAsync(userId, stream, file.ContentType);
        
        await sender.Send(new UpdateAvatarCommand(userId, url));
        
        return Results.Ok(new { url });
    }
}
```

---

## 7. Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, cfg) => cfg.ReadFrom.Configuration(ctx.Configuration));

// DB
builder.Services.AddDbContext<SocialDbContext>(opt =>
    opt.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Redis
builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = builder.Configuration["Redis:ConnectionString"]);

// MediatR
builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssemblyContaining<GetFeedQuery>());

// SignalR
builder.Services.AddSignalR()
    .AddStackExchangeRedis(builder.Configuration["Redis:ConnectionString"]!);

// Repos & Services
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IPostRepository, PostRepository>();
builder.Services.AddScoped<IFollowRepository, FollowRepository>();
builder.Services.AddScoped<INotificationRepository, NotificationRepository>();
builder.Services.AddScoped<INotificationService, NotificationService>();
builder.Services.AddScoped<IMediaUploadService, AzureBlobMediaUploadService>();
builder.Services.AddSingleton<IConnectionTracker, RedisConnectionTracker>();

// Auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]!))
        };
        // Allow SignalR to get token from query string
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = ctx =>
            {
                var token = ctx.Request.Query["access_token"];
                if (!string.IsNullOrEmpty(token) &&
                    ctx.HttpContext.Request.Path.StartsWithSegments("/hubs"))
                {
                    ctx.Token = token;
                }
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

builder.Services.AddHealthChecks()
    .AddNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")!)
    .AddRedis(builder.Configuration["Redis:ConnectionString"]!);

var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<SocialDbContext>();
    await db.Database.MigrateAsync();
}

app.UseExceptionHandler();
app.UseSwagger();
app.UseSwaggerUI();
app.UseSerilogRequestLogging();
app.UseAuthentication();
app.UseAuthorization();

app.MapSocialEndpoints();
app.MapMediaEndpoints();
app.MapAuthEndpoints();
app.MapHub<NotificationHub>("/hubs/notifications");
app.MapHealthChecks("/health");

app.Run();
```

---

## Exercises / Project Tasks

### Exercise 1: Stories Feature
เพิ่ม Stories (24-hour posts):
- StoryEntity มี expiry time
- Background service ลบ stories หมดอายุ
- Get stories feed จาก following

### Exercise 2: Trending Hashtags
สร้าง trending system:
- นับ hashtag usage ใน 24 ชั่วโมง
- Cache trending list (refresh ทุก 15 นาที)
- `/api/feed/trending` endpoint

### Exercise 3: Block User
เพิ่ม block feature:
- BlockEntity
- ซ่อน blocked users จาก feed
- ป้องกัน blocked users comment/like

### Exercise 4: Direct Messages
สร้าง DM system:
- Conversation / Message entities
- Real-time messaging ด้วย SignalR
- Typing indicators
- Read receipts

---

## สรุป

- **SignalR** สำหรับ real-time notifications และ messaging
- **Feed Algorithm** ใช้ engagement scoring บน SQL
- **Follow System** รองรับ private accounts
- **Hashtag Parsing** ด้วย regex ใน domain
- **Media Upload** ไปยัง Azure Blob Storage
- **Redis** ทั้ง caching, session, และ SignalR backplane

---

## Part ถัดไป

**Part 095: Real-world Project: SaaS Starter** - สร้าง multi-tenant SaaS template

---

*Part 094/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
