# Part 061: SignalR - Real-time Communication

## เนื้อหาใน Part นี้
- SignalR คืออะไร และทำงานอย่างไร
- Hub - หัวใจหลักของ SignalR
- Client Connection และการจัดการ
- Broadcasting ข้อความ
- Groups - การจัดกลุ่ม Connection
- JavaScript Client
- โปรแกรมตัวอย่าง: Real-time Chat Application

---

## 1. SignalR คืออะไร

**ASP.NET Core SignalR** เป็น library ที่ทำให้การสื่อสารแบบ real-time ระหว่าง server และ client ทำได้ง่ายขึ้น แทนที่จะต้อง polling server ซ้ำๆ SignalR ช่วยให้ server สามารถ push ข้อมูลไปยัง client ได้ทันทีที่มีการเปลี่ยนแปลง

### โปรโตคอลที่ SignalR รองรับ

SignalR จะเลือกโปรโตคอลที่ดีที่สุดโดยอัตโนมัติ:

1. **WebSockets** - ดีที่สุด (bidirectional, full-duplex)
2. **Server-Sent Events (SSE)** - ส่งข้อมูลจาก server ไป client (HTTP/1.1)
3. **Long Polling** - fallback สุดท้าย

### การติดตั้ง

```bash
# ไม่ต้องติดตั้งเพิ่ม SignalR มาพร้อมกับ ASP.NET Core แล้ว
# แต่ถ้าต้องการ JavaScript client ต้องติดตั้ง npm package
npm install @microsoft/signalr
```

หรือใช้ CDN:
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.0/signalr.min.js"></script>
```

---

## 2. Hub - หัวใจหลักของ SignalR

**Hub** คือ class ที่เป็น central point ของ SignalR ใช้จัดการ connection, รับ และส่งข้อความระหว่าง server กับ clients

### การสร้าง Hub เบื้องต้น

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Hubs;

public class ChatHub : Hub
{
    // Method นี้ client สามารถเรียกได้
    public async Task SendMessage(string user, string message)
    {
        // ส่งข้อความไปยัง clients ทั้งหมด
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    // Override เมื่อ client เชื่อมต่อ
    public override async Task OnConnectedAsync()
    {
        Console.WriteLine($"Client connected: {Context.ConnectionId}");
        await base.OnConnectedAsync();
    }

    // Override เมื่อ client ตัดการเชื่อมต่อ
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        Console.WriteLine($"Client disconnected: {Context.ConnectionId}");
        await base.OnDisconnectedAsync(exception);
    }
}
```

### การลงทะเบียน Hub ใน Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// เพิ่ม SignalR service
builder.Services.AddSignalR();

// เพิ่ม CORS สำหรับ development
builder.Services.AddCors(options =>
{
    options.AddPolicy("CorsPolicy", policy =>
    {
        policy.WithOrigins("http://localhost:3000")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

var app = builder.Build();

app.UseCors("CorsPolicy");

// Map Hub ไปยัง route
app.MapHub<ChatHub>("/chathub");

app.Run();
```

---

## 3. Client Connection

### JavaScript Client

```javascript
// wwwroot/js/chat.js
import * as signalR from "@microsoft/signalr";

// สร้าง connection
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chathub")
    .withAutomaticReconnect()  // reconnect อัตโนมัติ
    .configureLogging(signalR.LogLevel.Information)
    .build();

// รับข้อความจาก server
connection.on("ReceiveMessage", (user, message) => {
    const li = document.createElement("li");
    li.textContent = `${user}: ${message}`;
    document.getElementById("messagesList").appendChild(li);
});

// เชื่อมต่อ
async function startConnection() {
    try {
        await connection.start();
        console.log("SignalR Connected.");
    } catch (err) {
        console.error(err);
        setTimeout(startConnection, 5000); // retry หลัง 5 วินาที
    }
}

// จัดการสถานะการเชื่อมต่อ
connection.onreconnecting(error => {
    console.log("Reconnecting...", error);
});

connection.onreconnected(connectionId => {
    console.log("Reconnected. ConnectionId:", connectionId);
});

connection.onclose(error => {
    console.log("Connection closed", error);
});

startConnection();

// ส่งข้อความ
document.getElementById("sendButton").addEventListener("click", async () => {
    const user = document.getElementById("userInput").value;
    const message = document.getElementById("messageInput").value;

    try {
        await connection.invoke("SendMessage", user, message);
        document.getElementById("messageInput").value = "";
    } catch (err) {
        console.error(err);
    }
});
```

### .NET Client (สำหรับ server-to-server)

```csharp
// ใช้ใน Console App หรือ Service อื่น
using Microsoft.AspNetCore.SignalR.Client;

var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chathub")
    .WithAutomaticReconnect()
    .Build();

connection.On<string, string>("ReceiveMessage", (user, message) =>
{
    Console.WriteLine($"{user}: {message}");
});

await connection.StartAsync();

// ส่งข้อความ
await connection.InvokeAsync("SendMessage", "Bot", "Hello from .NET client!");
```

---

## 4. Broadcasting ข้อความ

SignalR มีหลายวิธีในการส่งข้อความ:

```csharp
// Hubs/BroadcastHub.cs
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Hubs;

public class BroadcastHub : Hub
{
    // ส่งไปยัง clients ทั้งหมด
    public async Task SendToAll(string message)
    {
        await Clients.All.SendAsync("ReceiveMessage", message);
    }

    // ส่งเฉพาะ caller (ผู้ส่ง)
    public async Task SendToCaller(string message)
    {
        await Clients.Caller.SendAsync("ReceiveMessage", message);
    }

    // ส่งไปยัง connection ID ที่ระบุ
    public async Task SendToSpecific(string connectionId, string message)
    {
        await Clients.Client(connectionId).SendAsync("ReceiveMessage", message);
    }

    // ส่งไปยังหลาย connection IDs
    public async Task SendToMultiple(List<string> connectionIds, string message)
    {
        await Clients.Clients(connectionIds).SendAsync("ReceiveMessage", message);
    }

    // ส่งไปยัง clients ทั้งหมด ยกเว้น caller
    public async Task SendToOthers(string message)
    {
        await Clients.Others.SendAsync("ReceiveMessage", message);
    }

    // ส่งไปยัง clients ทั้งหมด ยกเว้น connections ที่ระบุ
    public async Task SendToAllExcept(string message, List<string> excludedConnections)
    {
        await Clients.AllExcept(excludedConnections).SendAsync("ReceiveMessage", message);
    }
}
```

### การส่งข้อความจากนอก Hub (IHubContext)

```csharp
// Services/NotificationService.cs
using Microsoft.AspNetCore.SignalR;
using SignalRDemo.Hubs;

namespace SignalRDemo.Services;

public class NotificationService
{
    private readonly IHubContext<ChatHub> _hubContext;

    public NotificationService(IHubContext<ChatHub> hubContext)
    {
        _hubContext = hubContext;
    }

    // ส่ง notification ไปยัง clients ทั้งหมดจาก service
    public async Task SendNotificationAsync(string message)
    {
        await _hubContext.Clients.All.SendAsync("ReceiveNotification", message);
    }

    // ส่งไปยัง user ที่ระบุ
    public async Task SendToUserAsync(string userId, string message)
    {
        await _hubContext.Clients.User(userId).SendAsync("ReceiveNotification", message);
    }
}

// ลงทะเบียน service
// Program.cs
builder.Services.AddScoped<NotificationService>();
```

---

## 5. Groups - การจัดกลุ่ม Connection

Groups ช่วยให้เราสามารถส่งข้อความไปยังกลุ่มของ connections ได้

```csharp
// Hubs/GroupHub.cs
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Hubs;

public class GroupHub : Hub
{
    // เข้าร่วมกลุ่ม
    public async Task JoinGroup(string groupName)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName)
            .SendAsync("SystemMessage", $"{Context.ConnectionId} ได้เข้าร่วมกลุ่ม {groupName}");
    }

    // ออกจากกลุ่ม
    public async Task LeaveGroup(string groupName)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);
        await Clients.Group(groupName)
            .SendAsync("SystemMessage", $"{Context.ConnectionId} ได้ออกจากกลุ่ม {groupName}");
    }

    // ส่งข้อความไปยังกลุ่ม
    public async Task SendMessageToGroup(string groupName, string user, string message)
    {
        await Clients.Group(groupName).SendAsync("ReceiveMessage", user, message);
    }

    // ส่งไปยังกลุ่ม ยกเว้น caller
    public async Task SendMessageToGroupExcluding(string groupName, string message)
    {
        await Clients.GroupExcept(groupName, Context.ConnectionId)
            .SendAsync("ReceiveMessage", message);
    }

    // ส่งไปยังหลายกลุ่ม
    public async Task SendToMultipleGroups(List<string> groupNames, string message)
    {
        await Clients.Groups(groupNames).SendAsync("ReceiveMessage", message);
    }

    // ออกจากกลุ่มทั้งหมดเมื่อ disconnect
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        // SignalR จะ remove connection ออกจากทุก group อัตโนมัติ
        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## 6. Authentication ใน SignalR

```csharp
// Hubs/AuthenticatedHub.cs
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Hubs;

[Authorize]  // ต้อง authenticate ก่อน
public class SecureHub : Hub
{
    public async Task SendMessage(string message)
    {
        // ดึงข้อมูล user จาก Claims
        var userId = Context.UserIdentifier;
        var userName = Context.User?.Identity?.Name ?? "Unknown";

        await Clients.All.SendAsync("ReceiveMessage", userName, message);
    }
}
```

### JavaScript Client พร้อม Token

```javascript
// ส่ง JWT token ผ่าน query string (สำหรับ WebSocket)
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/securehub", {
        accessTokenFactory: () => {
            return localStorage.getItem("authToken");
        }
    })
    .build();
```

---

## 7. Typed Hubs - Type Safety

```csharp
// Hubs/ITypedChatClient.cs
namespace SignalRDemo.Hubs;

public interface ITypedChatClient
{
    Task ReceiveMessage(string user, string message);
    Task ReceiveNotification(string message);
    Task UserJoined(string userId);
    Task UserLeft(string userId);
}

// Hubs/TypedChatHub.cs
using Microsoft.AspNetCore.SignalR;

namespace SignalRDemo.Hubs;

public class TypedChatHub : Hub<ITypedChatClient>
{
    public async Task SendMessage(string user, string message)
    {
        // Strongly typed - ไม่ต้องใช้ string method name
        await Clients.All.ReceiveMessage(user, message);
    }

    public async Task SendNotification(string message)
    {
        await Clients.Others.ReceiveNotification(message);
    }

    public override async Task OnConnectedAsync()
    {
        var userId = Context.ConnectionId;
        await Clients.Others.UserJoined(userId);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.ConnectionId;
        await Clients.Others.UserLeft(userId);
        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## โปรแกรมตัวอย่าง: Real-time Chat Application

### โครงสร้างโปรเจค

```
ChatApp/
├── Hubs/
│   └── ChatHub.cs
├── Models/
│   ├── Message.cs
│   └── ChatRoom.cs
├── Services/
│   └── ChatService.cs
├── Controllers/
│   └── ChatController.cs
├── wwwroot/
│   ├── css/
│   │   └── chat.css
│   └── js/
│       └── chat.js
├── Pages/
│   └── Index.cshtml
└── Program.cs
```

### Models

```csharp
// Models/Message.cs
namespace ChatApp.Models;

public class Message
{
    public int Id { get; set; }
    public string RoomId { get; set; } = string.Empty;
    public string UserId { get; set; } = string.Empty;
    public string UserName { get; set; } = string.Empty;
    public string Content { get; set; } = string.Empty;
    public DateTime SentAt { get; set; } = DateTime.UtcNow;
    public MessageType Type { get; set; } = MessageType.User;
}

public enum MessageType
{
    User,
    System,
    Bot
}

// Models/ChatRoom.cs
namespace ChatApp.Models;

public class ChatRoom
{
    public string Id { get; set; } = Guid.NewGuid().ToString();
    public string Name { get; set; } = string.Empty;
    public List<string> Members { get; set; } = new();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public int MemberCount => Members.Count;
}
```

### ChatHub

```csharp
// Hubs/ChatHub.cs
using Microsoft.AspNetCore.SignalR;
using ChatApp.Models;
using ChatApp.Services;

namespace ChatApp.Hubs;

public interface IChatClient
{
    Task ReceiveMessage(Message message);
    Task UserJoined(string roomId, string userName, int memberCount);
    Task UserLeft(string roomId, string userName, int memberCount);
    Task RoomCreated(ChatRoom room);
    Task Error(string errorMessage);
}

public class ChatHub : Hub<IChatClient>
{
    private readonly ChatService _chatService;
    private readonly ILogger<ChatHub> _logger;

    public ChatHub(ChatService chatService, ILogger<ChatHub> logger)
    {
        _chatService = chatService;
        _logger = logger;
    }

    // เข้าร่วมห้องแชท
    public async Task JoinRoom(string roomId, string userName)
    {
        try
        {
            var room = await _chatService.JoinRoomAsync(roomId, Context.ConnectionId, userName);

            if (room == null)
            {
                await Clients.Caller.Error("ไม่พบห้องแชทที่ระบุ");
                return;
            }

            await Groups.AddToGroupAsync(Context.ConnectionId, roomId);

            // บอก client คนอื่นในห้อง
            await Clients.OthersInGroup(roomId).UserJoined(roomId, userName, room.MemberCount);

            // ส่งประวัติข้อความให้ผู้เข้าร่วมใหม่
            var history = await _chatService.GetMessageHistoryAsync(roomId, 50);
            foreach (var msg in history)
            {
                await Clients.Caller.ReceiveMessage(msg);
            }

            // ส่ง system message
            var systemMsg = new Message
            {
                RoomId = roomId,
                UserName = "System",
                Content = $"{userName} ได้เข้าร่วมห้องแชท",
                Type = MessageType.System
            };
            await Clients.Group(roomId).ReceiveMessage(systemMsg);

            _logger.LogInformation("User {UserName} joined room {RoomId}", userName, roomId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error joining room {RoomId}", roomId);
            await Clients.Caller.Error("เกิดข้อผิดพลาดในการเข้าร่วมห้อง");
        }
    }

    // ออกจากห้องแชท
    public async Task LeaveRoom(string roomId, string userName)
    {
        var room = await _chatService.LeaveRoomAsync(roomId, Context.ConnectionId);

        await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomId);

        var systemMsg = new Message
        {
            RoomId = roomId,
            UserName = "System",
            Content = $"{userName} ได้ออกจากห้องแชท",
            Type = MessageType.System
        };

        await Clients.Group(roomId).ReceiveMessage(systemMsg);

        if (room != null)
        {
            await Clients.OthersInGroup(roomId).UserLeft(roomId, userName, room.MemberCount);
        }
    }

    // ส่งข้อความ
    public async Task SendMessage(string roomId, string userName, string content)
    {
        if (string.IsNullOrWhiteSpace(content))
        {
            await Clients.Caller.Error("ข้อความไม่สามารถว่างได้");
            return;
        }

        if (content.Length > 500)
        {
            await Clients.Caller.Error("ข้อความยาวเกินไป (สูงสุด 500 ตัวอักษร)");
            return;
        }

        var message = new Message
        {
            RoomId = roomId,
            UserId = Context.ConnectionId,
            UserName = userName,
            Content = content,
            SentAt = DateTime.UtcNow
        };

        await _chatService.SaveMessageAsync(message);

        // ส่งไปยัง clients ทั้งหมดในกลุ่ม
        await Clients.Group(roomId).ReceiveMessage(message);
    }

    // สร้างห้องใหม่
    public async Task CreateRoom(string roomName)
    {
        var room = await _chatService.CreateRoomAsync(roomName);
        await Clients.All.RoomCreated(room);
    }

    // Typing indicator
    public async Task Typing(string roomId, string userName, bool isTyping)
    {
        await Clients.OthersInGroup(roomId).SendAsync("UserTyping", userName, isTyping);
    }

    // Disconnect
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userInfo = await _chatService.GetUserInfoAsync(Context.ConnectionId);

        if (userInfo != null)
        {
            foreach (var roomId in userInfo.Rooms)
            {
                var room = await _chatService.LeaveRoomAsync(roomId, Context.ConnectionId);

                var systemMsg = new Message
                {
                    RoomId = roomId,
                    UserName = "System",
                    Content = $"{userInfo.UserName} ออกจากระบบ",
                    Type = MessageType.System
                };

                await Clients.Group(roomId).ReceiveMessage(systemMsg);
            }

            await _chatService.RemoveUserAsync(Context.ConnectionId);
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

### ChatService

```csharp
// Services/ChatService.cs
using ChatApp.Models;

namespace ChatApp.Services;

public class UserInfo
{
    public string ConnectionId { get; set; } = string.Empty;
    public string UserName { get; set; } = string.Empty;
    public List<string> Rooms { get; set; } = new();
}

public class ChatService
{
    private readonly Dictionary<string, ChatRoom> _rooms = new();
    private readonly Dictionary<string, List<Message>> _messageHistory = new();
    private readonly Dictionary<string, UserInfo> _users = new();
    private readonly object _lock = new();

    public Task<ChatRoom?> JoinRoomAsync(string roomId, string connectionId, string userName)
    {
        lock (_lock)
        {
            if (!_rooms.TryGetValue(roomId, out var room))
                return Task.FromResult<ChatRoom?>(null);

            if (!room.Members.Contains(connectionId))
                room.Members.Add(connectionId);

            // บันทึกข้อมูล user
            if (!_users.ContainsKey(connectionId))
            {
                _users[connectionId] = new UserInfo
                {
                    ConnectionId = connectionId,
                    UserName = userName
                };
            }

            _users[connectionId].Rooms.Add(roomId);

            return Task.FromResult<ChatRoom?>(room);
        }
    }

    public Task<ChatRoom?> LeaveRoomAsync(string roomId, string connectionId)
    {
        lock (_lock)
        {
            if (!_rooms.TryGetValue(roomId, out var room))
                return Task.FromResult<ChatRoom?>(null);

            room.Members.Remove(connectionId);

            if (_users.TryGetValue(connectionId, out var user))
                user.Rooms.Remove(roomId);

            return Task.FromResult<ChatRoom?>(room);
        }
    }

    public Task<ChatRoom> CreateRoomAsync(string roomName)
    {
        var room = new ChatRoom { Name = roomName };

        lock (_lock)
        {
            _rooms[room.Id] = room;
            _messageHistory[room.Id] = new List<Message>();
        }

        return Task.FromResult(room);
    }

    public Task SaveMessageAsync(Message message)
    {
        lock (_lock)
        {
            if (!_messageHistory.ContainsKey(message.RoomId))
                _messageHistory[message.RoomId] = new List<Message>();

            message.Id = _messageHistory[message.RoomId].Count + 1;
            _messageHistory[message.RoomId].Add(message);

            // เก็บแค่ 1000 ข้อความล่าสุด
            if (_messageHistory[message.RoomId].Count > 1000)
                _messageHistory[message.RoomId].RemoveAt(0);
        }

        return Task.CompletedTask;
    }

    public Task<List<Message>> GetMessageHistoryAsync(string roomId, int count)
    {
        lock (_lock)
        {
            if (!_messageHistory.TryGetValue(roomId, out var messages))
                return Task.FromResult(new List<Message>());

            return Task.FromResult(messages.TakeLast(count).ToList());
        }
    }

    public Task<UserInfo?> GetUserInfoAsync(string connectionId)
    {
        lock (_lock)
        {
            _users.TryGetValue(connectionId, out var user);
            return Task.FromResult(user);
        }
    }

    public Task RemoveUserAsync(string connectionId)
    {
        lock (_lock)
        {
            _users.Remove(connectionId);
        }
        return Task.CompletedTask;
    }

    public Task<List<ChatRoom>> GetRoomsAsync()
    {
        lock (_lock)
        {
            return Task.FromResult(_rooms.Values.ToList());
        }
    }
}
```

### Program.cs

```csharp
// Program.cs
using ChatApp.Hubs;
using ChatApp.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorPages();
builder.Services.AddSignalR(options =>
{
    options.EnableDetailedErrors = builder.Environment.IsDevelopment();
    options.MaximumReceiveMessageSize = 32 * 1024; // 32 KB
    options.HandshakeTimeout = TimeSpan.FromSeconds(15);
    options.KeepAliveInterval = TimeSpan.FromSeconds(15);
});

builder.Services.AddSingleton<ChatService>();

builder.Services.AddCors(options =>
{
    options.AddPolicy("ChatPolicy", policy =>
    {
        policy.WithOrigins("http://localhost:3000", "https://mychat.app")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseCors("ChatPolicy");
app.UseAuthorization();
app.MapRazorPages();
app.MapHub<ChatHub>("/chathub");

// สร้างห้อง default
var chatService = app.Services.GetRequiredService<ChatService>();
await chatService.CreateRoomAsync("General");
await chatService.CreateRoomAsync("Tech Talk");
await chatService.CreateRoomAsync("Random");

app.Run();
```

### HTML/JavaScript Client

```html
<!-- Pages/Index.cshtml -->
@page
@model IndexModel
@{
    ViewData["Title"] = "Real-time Chat";
}

<!DOCTYPE html>
<html>
<head>
    <title>Chat App</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', sans-serif; background: #f0f2f5; }

        .chat-container {
            display: flex;
            height: 100vh;
            max-width: 1200px;
            margin: 0 auto;
        }

        .sidebar {
            width: 250px;
            background: #2c3e50;
            color: white;
            padding: 20px;
        }

        .room-list { list-style: none; margin-top: 20px; }

        .room-item {
            padding: 10px;
            cursor: pointer;
            border-radius: 8px;
            margin-bottom: 5px;
        }

        .room-item:hover, .room-item.active {
            background: #3d5166;
        }

        .chat-area {
            flex: 1;
            display: flex;
            flex-direction: column;
        }

        .messages {
            flex: 1;
            padding: 20px;
            overflow-y: auto;
        }

        .message {
            background: white;
            border-radius: 12px;
            padding: 12px 16px;
            margin-bottom: 12px;
            max-width: 70%;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }

        .message.own { margin-left: auto; background: #dcf8c6; }
        .message.system { background: transparent; text-align: center; font-size: 0.8em; color: #888; }

        .message-author { font-weight: bold; color: #2c3e50; font-size: 0.85em; }
        .message-time { font-size: 0.75em; color: #888; }

        .input-area {
            padding: 20px;
            background: white;
            border-top: 1px solid #eee;
            display: flex;
            gap: 10px;
        }

        .input-area input {
            flex: 1;
            padding: 12px;
            border: 1px solid #ddd;
            border-radius: 24px;
            font-size: 14px;
        }

        .input-area button {
            padding: 12px 24px;
            background: #2c3e50;
            color: white;
            border: none;
            border-radius: 24px;
            cursor: pointer;
        }

        .status-bar {
            padding: 10px 20px;
            background: #2c3e50;
            color: white;
            font-size: 0.85em;
        }

        .typing-indicator { color: #888; font-size: 0.8em; padding: 5px 20px; }
    </style>
</head>
<body>
    <div class="chat-container">
        <div class="sidebar">
            <h2>Chat Rooms</h2>
            <div>
                <input id="userNameInput" placeholder="ชื่อของคุณ" style="width:100%; padding:8px; margin-top:10px; border-radius:4px; border:none;">
            </div>
            <ul class="room-list" id="roomList"></ul>
        </div>
        <div class="chat-area">
            <div class="status-bar" id="statusBar">กำลังเชื่อมต่อ...</div>
            <div class="messages" id="messagesList"></div>
            <div class="typing-indicator" id="typingIndicator"></div>
            <div class="input-area">
                <input type="text" id="messageInput" placeholder="พิมพ์ข้อความ..." />
                <button id="sendButton">ส่ง</button>
            </div>
        </div>
    </div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/microsoft-signalr/8.0.0/signalr.min.js"></script>
    <script>
        let currentRoom = null;
        let typingTimer = null;

        const connection = new signalR.HubConnectionBuilder()
            .withUrl("/chathub")
            .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
            .configureLogging(signalR.LogLevel.Warning)
            .build();

        // รับข้อความ
        connection.on("ReceiveMessage", (message) => {
            if (message.roomId !== currentRoom) return;

            const messageEl = document.createElement("div");
            const isOwn = message.userId === connection.connectionId;
            const isSystem = message.type === 1; // MessageType.System

            messageEl.className = `message ${isSystem ? 'system' : (isOwn ? 'own' : '')}`;

            if (!isSystem) {
                messageEl.innerHTML = `
                    <div class="message-author">${message.userName}</div>
                    <div class="message-content">${escapeHtml(message.content)}</div>
                    <div class="message-time">${new Date(message.sentAt).toLocaleTimeString()}</div>
                `;
            } else {
                messageEl.textContent = message.content;
            }

            document.getElementById("messagesList").appendChild(messageEl);
            messageEl.scrollIntoView({ behavior: 'smooth' });
        });

        // เมื่อมีคนพิมพ์
        connection.on("UserTyping", (userName, isTyping) => {
            const indicator = document.getElementById("typingIndicator");
            indicator.textContent = isTyping ? `${userName} กำลังพิมพ์...` : '';
        });

        // ห้องใหม่ถูกสร้าง
        connection.on("RoomCreated", (room) => {
            addRoomToList(room);
        });

        // เริ่ม connection
        async function start() {
            try {
                await connection.start();
                document.getElementById("statusBar").textContent = "เชื่อมต่อแล้ว";

                // โหลดรายการห้อง
                const response = await fetch('/api/rooms');
                const rooms = await response.json();
                rooms.forEach(addRoomToList);

                // เข้าร่วมห้องแรก
                if (rooms.length > 0) {
                    joinRoom(rooms[0].id);
                }
            } catch (err) {
                console.error(err);
                document.getElementById("statusBar").textContent = "เชื่อมต่อไม่สำเร็จ";
            }
        }

        function addRoomToList(room) {
            const li = document.createElement("li");
            li.className = "room-item";
            li.textContent = room.name;
            li.onclick = () => joinRoom(room.id);
            li.dataset.roomId = room.id;
            document.getElementById("roomList").appendChild(li);
        }

        async function joinRoom(roomId) {
            const userName = document.getElementById("userNameInput").value || "Anonymous";

            if (currentRoom) {
                await connection.invoke("LeaveRoom", currentRoom, userName);
            }

            currentRoom = roomId;
            document.getElementById("messagesList").innerHTML = '';

            // อัพเดท UI
            document.querySelectorAll(".room-item").forEach(el => {
                el.classList.toggle("active", el.dataset.roomId === roomId);
            });

            await connection.invoke("JoinRoom", roomId, userName);
        }

        // ส่งข้อความ
        async function sendMessage() {
            const content = document.getElementById("messageInput").value.trim();
            const userName = document.getElementById("userNameInput").value || "Anonymous";

            if (!content || !currentRoom) return;

            await connection.invoke("SendMessage", currentRoom, userName, content);
            document.getElementById("messageInput").value = '';
        }

        // Typing indicator
        document.getElementById("messageInput").addEventListener("input", () => {
            const userName = document.getElementById("userNameInput").value || "Anonymous";

            if (currentRoom) {
                connection.invoke("Typing", currentRoom, userName, true);

                clearTimeout(typingTimer);
                typingTimer = setTimeout(() => {
                    connection.invoke("Typing", currentRoom, userName, false);
                }, 2000);
            }
        });

        document.getElementById("sendButton").addEventListener("click", sendMessage);
        document.getElementById("messageInput").addEventListener("keypress", (e) => {
            if (e.key === 'Enter') sendMessage();
        });

        function escapeHtml(text) {
            const div = document.createElement('div');
            div.appendChild(document.createTextNode(text));
            return div.innerHTML;
        }

        // Reconnection events
        connection.onreconnecting(() => {
            document.getElementById("statusBar").textContent = "กำลังเชื่อมต่อใหม่...";
        });

        connection.onreconnected(() => {
            document.getElementById("statusBar").textContent = "เชื่อมต่อแล้ว";
            if (currentRoom) {
                const userName = document.getElementById("userNameInput").value || "Anonymous";
                connection.invoke("JoinRoom", currentRoom, userName);
            }
        });

        start();
    </script>
</body>
</html>
```

---

## Exercises

### Exercise 1: Private Messaging
เพิ่มฟีเจอร์ส่งข้อความส่วนตัวระหว่าง users สองคน โดยใช้ `Clients.User(userId)`

### Exercise 2: Online Users List
แสดงรายการ users ที่ online อยู่ใน hub และอัปเดตแบบ real-time เมื่อมีคน join/leave

### Exercise 3: Message Reactions
เพิ่มฟีเจอร์ให้ users สามารถ react ต่อข้อความด้วย emoji ได้

### Exercise 4: File Sharing
ส่งไฟล์ผ่าน SignalR โดยใช้การ encode เป็น Base64

### Exercise 5: Read Receipts
เพิ่ม read receipt เพื่อแสดงว่าใครอ่านข้อความไปแล้วบ้าง

---

## สรุป

- **SignalR** ทำให้การสื่อสาร real-time ระหว่าง server และ client ทำได้ง่าย
- **Hub** เป็น central point ที่จัดการ connections และข้อความ
- ใช้ **Groups** สำหรับการจัดกลุ่ม connections
- **IHubContext** ใช้ส่งข้อความจากนอก Hub
- **Typed Hubs** ช่วยให้มี type safety
- ควร handle reconnection และ error อย่างเหมาะสม

---

## Part ถัดไป

**Part 062: Background Services** - เรียนรู้การสร้าง service ที่ทำงานเบื้องหลังด้วย `IHostedService` และ `BackgroundService`

---

*Part 061/700 | Phase 4: ASP.NET Core ระดับกลาง | หลักสูตร C# และ ASP.NET Core*
