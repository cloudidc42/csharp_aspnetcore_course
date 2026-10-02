# Part 091: Event Sourcing

## เนื้อหาใน Part นี้
- Event Sourcing pattern
- Event store
- Projections
- Snapshots
- CQRS + Event Sourcing
- โปรแกรมตัวอย่าง: Account balance with events

---

## 1. Event Sourcing Pattern

Event Sourcing เก็บ state ของ application เป็น sequence ของ events แทนที่จะเก็บ current state

### Traditional vs Event Sourcing

```
Traditional Approach:
Table: Accounts
| id  | balance | status   | updated_at |
|-----|---------|----------|------------|
| 001 | 5000    | active   | 2024-01-15 |

ปัญหา: เราไม่รู้ว่า balance เปลี่ยนไปอย่างไร และทำไม

Event Sourcing Approach:
Table: AccountEvents
| id | account_id | type              | data                 | occurred_at |
|----|------------|-------------------|----------------------|-------------|
| 1  | 001        | AccountOpened     | {balance: 0}         | 2024-01-01  |
| 2  | 001        | MoneyDeposited    | {amount: 10000}      | 2024-01-05  |
| 3  | 001        | MoneyWithdrawn    | {amount: 3000}       | 2024-01-10  |
| 4  | 001        | MoneyDeposited    | {amount: 1000}       | 2024-01-12  |
| 5  | 001        | AccountSuspended  | {reason: "fraud"}    | 2024-01-14  |
| 6  | 001        | AccountReinstated | {}                   | 2024-01-15  |

ข้อดี:
- Full audit trail
- Time travel (replay to any point)
- ไม่มี data loss
- Event-driven integration
```

---

## 2. Events และ Aggregate

### Domain Events

```csharp
// Events/AccountEvents.cs

// Base event
public abstract record DomainEvent
{
    public Guid EventId { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
    public int Version { get; init; }
}

// Account events
public record AccountOpened : DomainEvent
{
    public Guid AccountId { get; init; }
    public Guid OwnerId { get; init; }
    public string AccountNumber { get; init; } = string.Empty;
    public decimal InitialDeposit { get; init; }
    public string AccountType { get; init; } = string.Empty;
}

public record MoneyDeposited : DomainEvent
{
    public Guid AccountId { get; init; }
    public decimal Amount { get; init; }
    public string Description { get; init; } = string.Empty;
    public string TransactionId { get; init; } = string.Empty;
}

public record MoneyWithdrawn : DomainEvent
{
    public Guid AccountId { get; init; }
    public decimal Amount { get; init; }
    public string Description { get; init; } = string.Empty;
    public string TransactionId { get; init; } = string.Empty;
}

public record MoneyTransferred : DomainEvent
{
    public Guid SourceAccountId { get; init; }
    public Guid TargetAccountId { get; init; }
    public decimal Amount { get; init; }
    public string Description { get; init; } = string.Empty;
}

public record AccountSuspended : DomainEvent
{
    public Guid AccountId { get; init; }
    public string Reason { get; init; } = string.Empty;
    public Guid SuspendedBy { get; init; }
}

public record AccountClosed : DomainEvent
{
    public Guid AccountId { get; init; }
    public string Reason { get; init; } = string.Empty;
}
```

### Account Aggregate

```csharp
// Domain/Account.cs
public class Account
{
    private readonly List<DomainEvent> _uncommittedEvents = new();
    
    // State
    public Guid Id { get; private set; }
    public Guid OwnerId { get; private set; }
    public string AccountNumber { get; private set; } = string.Empty;
    public decimal Balance { get; private set; }
    public AccountStatus Status { get; private set; }
    public string AccountType { get; private set; } = string.Empty;
    public int Version { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    private Account() { }

    // Factory method - สร้าง account ใหม่
    public static Account Open(Guid ownerId, string accountType, decimal initialDeposit = 0)
    {
        if (initialDeposit < 0)
            throw new DomainException("Initial deposit cannot be negative");

        var account = new Account();
        
        account.Apply(new AccountOpened
        {
            AccountId = Guid.NewGuid(),
            OwnerId = ownerId,
            AccountNumber = GenerateAccountNumber(),
            AccountType = accountType,
            InitialDeposit = initialDeposit
        });
        
        if (initialDeposit > 0)
        {
            account.Apply(new MoneyDeposited
            {
                AccountId = account.Id,
                Amount = initialDeposit,
                Description = "Initial deposit",
                TransactionId = Guid.NewGuid().ToString()
            });
        }
        
        return account;
    }

    // Commands
    public void Deposit(decimal amount, string description, string transactionId)
    {
        if (amount <= 0) throw new DomainException("Amount must be positive");
        if (Status != AccountStatus.Active) throw new DomainException($"Cannot deposit to {Status} account");
        
        Apply(new MoneyDeposited
        {
            AccountId = Id,
            Amount = amount,
            Description = description,
            TransactionId = transactionId
        });
    }

    public void Withdraw(decimal amount, string description, string transactionId)
    {
        if (amount <= 0) throw new DomainException("Amount must be positive");
        if (Status != AccountStatus.Active) throw new DomainException($"Cannot withdraw from {Status} account");
        if (Balance < amount) throw new DomainException($"Insufficient balance. Available: {Balance:N2}");
        
        Apply(new MoneyWithdrawn
        {
            AccountId = Id,
            Amount = amount,
            Description = description,
            TransactionId = transactionId
        });
    }

    public void Transfer(decimal amount, Guid targetAccountId, string description)
    {
        if (amount <= 0) throw new DomainException("Amount must be positive");
        if (Status != AccountStatus.Active) throw new DomainException("Account is not active");
        if (Balance < amount) throw new DomainException("Insufficient balance");
        
        Apply(new MoneyTransferred
        {
            SourceAccountId = Id,
            TargetAccountId = targetAccountId,
            Amount = amount,
            Description = description
        });
    }

    public void Suspend(string reason, Guid suspendedBy)
    {
        if (Status == AccountStatus.Suspended) throw new DomainException("Account already suspended");
        if (Status == AccountStatus.Closed) throw new DomainException("Cannot suspend closed account");
        
        Apply(new AccountSuspended
        {
            AccountId = Id,
            Reason = reason,
            SuspendedBy = suspendedBy
        });
    }

    // Event application - update state from events
    private void Apply(DomainEvent @event)
    {
        // Update state
        When(@event);
        
        // Track version
        Version++;
        var eventWithVersion = @event with { Version = Version };
        
        // Add to uncommitted events
        _uncommittedEvents.Add(eventWithVersion);
    }

    // Rebuild state from stored events
    public static Account Reconstruct(IEnumerable<DomainEvent> events)
    {
        var account = new Account();
        foreach (var @event in events)
        {
            account.When(@event);
            account.Version = @event.Version;
        }
        return account;
    }

    // Apply state changes
    private void When(DomainEvent @event)
    {
        switch (@event)
        {
            case AccountOpened e:
                Id = e.AccountId;
                OwnerId = e.OwnerId;
                AccountNumber = e.AccountNumber;
                AccountType = e.AccountType;
                Balance = 0;
                Status = AccountStatus.Active;
                CreatedAt = e.OccurredAt;
                break;
            
            case MoneyDeposited e:
                Balance += e.Amount;
                UpdatedAt = e.OccurredAt;
                break;
            
            case MoneyWithdrawn e:
                Balance -= e.Amount;
                UpdatedAt = e.OccurredAt;
                break;
            
            case MoneyTransferred e:
                if (e.SourceAccountId == Id)
                    Balance -= e.Amount;
                else
                    Balance += e.Amount;
                UpdatedAt = e.OccurredAt;
                break;
            
            case AccountSuspended e:
                Status = AccountStatus.Suspended;
                UpdatedAt = e.OccurredAt;
                break;
            
            case AccountClosed:
                Status = AccountStatus.Closed;
                UpdatedAt = @event.OccurredAt;
                break;
        }
    }

    public IReadOnlyList<DomainEvent> GetUncommittedEvents() => _uncommittedEvents.AsReadOnly();
    public void ClearUncommittedEvents() => _uncommittedEvents.Clear();
    
    private static string GenerateAccountNumber()
    {
        return $"ACC{DateTime.UtcNow:yyyyMMdd}{Random.Shared.Next(1000, 9999)}";
    }
}

public enum AccountStatus { Active, Suspended, Closed }

public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```

---

## 3. Event Store

```csharp
// Infrastructure/EventStore.cs
using System.Text.Json;

public interface IEventStore
{
    Task AppendEventsAsync(Guid streamId, IEnumerable<DomainEvent> events, int expectedVersion, CancellationToken ct = default);
    Task<IEnumerable<DomainEvent>> LoadEventsAsync(Guid streamId, CancellationToken ct = default);
    Task<IEnumerable<DomainEvent>> LoadEventsAsync(Guid streamId, int fromVersion, CancellationToken ct = default);
}

// Database entity
public class StoredEvent
{
    public long Id { get; set; }
    public Guid StreamId { get; set; }
    public int Version { get; set; }
    public string EventType { get; set; } = string.Empty;
    public string EventData { get; set; } = string.Empty;
    public DateTime OccurredAt { get; set; }
}

// EF Core Event Store
public class EfEventStore : IEventStore
{
    private readonly EventStoreDbContext _dbContext;
    private readonly ILogger<EfEventStore> _logger;
    
    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        Converters = { new System.Text.Json.Serialization.JsonStringEnumConverter() }
    };
    
    // Event type registry
    private static readonly Dictionary<string, Type> EventTypes = new()
    {
        { nameof(AccountOpened), typeof(AccountOpened) },
        { nameof(MoneyDeposited), typeof(MoneyDeposited) },
        { nameof(MoneyWithdrawn), typeof(MoneyWithdrawn) },
        { nameof(MoneyTransferred), typeof(MoneyTransferred) },
        { nameof(AccountSuspended), typeof(AccountSuspended) },
        { nameof(AccountClosed), typeof(AccountClosed) },
    };

    public EfEventStore(EventStoreDbContext dbContext, ILogger<EfEventStore> logger)
    {
        _dbContext = dbContext;
        _logger = logger;
    }

    public async Task AppendEventsAsync(
        Guid streamId,
        IEnumerable<DomainEvent> events,
        int expectedVersion,
        CancellationToken ct = default)
    {
        await using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);
        
        try
        {
            // Optimistic concurrency check
            var currentVersion = await _dbContext.Events
                .Where(e => e.StreamId == streamId)
                .MaxAsync(e => (int?)e.Version, ct) ?? 0;
            
            if (currentVersion != expectedVersion)
            {
                throw new ConcurrencyException(
                    $"Expected version {expectedVersion} but found {currentVersion}");
            }
            
            // Save events
            int version = expectedVersion;
            foreach (var @event in events)
            {
                version++;
                _dbContext.Events.Add(new StoredEvent
                {
                    StreamId = streamId,
                    Version = version,
                    EventType = @event.EventType,
                    EventData = JsonSerializer.Serialize(@event, @event.GetType(), JsonOptions),
                    OccurredAt = @event.OccurredAt
                });
            }
            
            await _dbContext.SaveChangesAsync(ct);
            await transaction.CommitAsync(ct);
            
            _logger.LogDebug("Appended {Count} events to stream {StreamId}", 
                events.Count(), streamId);
        }
        catch
        {
            await transaction.RollbackAsync(ct);
            throw;
        }
    }

    public async Task<IEnumerable<DomainEvent>> LoadEventsAsync(
        Guid streamId, 
        CancellationToken ct = default)
    {
        return await LoadEventsAsync(streamId, 0, ct);
    }

    public async Task<IEnumerable<DomainEvent>> LoadEventsAsync(
        Guid streamId,
        int fromVersion,
        CancellationToken ct = default)
    {
        var storedEvents = await _dbContext.Events
            .AsNoTracking()
            .Where(e => e.StreamId == streamId && e.Version > fromVersion)
            .OrderBy(e => e.Version)
            .ToListAsync(ct);
        
        return storedEvents.Select(Deserialize).Where(e => e != null).Cast<DomainEvent>();
    }

    private DomainEvent? Deserialize(StoredEvent storedEvent)
    {
        if (!EventTypes.TryGetValue(storedEvent.EventType, out var eventType))
        {
            _logger.LogWarning("Unknown event type: {EventType}", storedEvent.EventType);
            return null;
        }
        
        return (DomainEvent?)JsonSerializer.Deserialize(storedEvent.EventData, eventType, JsonOptions);
    }
}

public class ConcurrencyException : Exception
{
    public ConcurrencyException(string message) : base(message) { }
}
```

### Account Repository

```csharp
// Infrastructure/AccountRepository.cs
public interface IAccountRepository
{
    Task<Account?> LoadAsync(Guid accountId, CancellationToken ct = default);
    Task SaveAsync(Account account, CancellationToken ct = default);
}

public class AccountRepository : IAccountRepository
{
    private readonly IEventStore _eventStore;
    private readonly ISnapshotStore _snapshotStore;

    public AccountRepository(IEventStore eventStore, ISnapshotStore snapshotStore)
    {
        _eventStore = eventStore;
        _snapshotStore = snapshotStore;
    }

    public async Task<Account?> LoadAsync(Guid accountId, CancellationToken ct = default)
    {
        // Try to load from snapshot first
        var snapshot = await _snapshotStore.LoadAsync<AccountSnapshot>(accountId, ct);
        
        if (snapshot != null)
        {
            // Load only events after snapshot
            var events = await _eventStore.LoadEventsAsync(accountId, snapshot.Version, ct);
            var account = Account.ReconstructFromSnapshot(snapshot, events);
            return account;
        }
        else
        {
            var events = await _eventStore.LoadEventsAsync(accountId, ct);
            if (!events.Any()) return null;
            
            return Account.Reconstruct(events);
        }
    }

    public async Task SaveAsync(Account account, CancellationToken ct = default)
    {
        var uncommittedEvents = account.GetUncommittedEvents();
        if (!uncommittedEvents.Any()) return;
        
        var expectedVersion = account.Version - uncommittedEvents.Count;
        
        await _eventStore.AppendEventsAsync(
            account.Id, 
            uncommittedEvents, 
            expectedVersion, 
            ct);
        
        account.ClearUncommittedEvents();
        
        // Create snapshot every 50 events
        if (account.Version % 50 == 0)
        {
            await _snapshotStore.SaveAsync(
                account.Id,
                AccountSnapshot.From(account),
                ct);
        }
    }
}
```

---

## 4. Snapshots

```csharp
// Infrastructure/SnapshotStore.cs
public interface ISnapshotStore
{
    Task<T?> LoadAsync<T>(Guid streamId, CancellationToken ct = default) where T : class;
    Task SaveAsync<T>(Guid streamId, T snapshot, CancellationToken ct = default) where T : class;
}

// Account snapshot
public class AccountSnapshot
{
    public Guid AccountId { get; set; }
    public Guid OwnerId { get; set; }
    public string AccountNumber { get; set; } = string.Empty;
    public decimal Balance { get; set; }
    public AccountStatus Status { get; set; }
    public int Version { get; set; }
    public DateTime SnapshotAt { get; set; }
    
    public static AccountSnapshot From(Account account) => new()
    {
        AccountId = account.Id,
        OwnerId = account.OwnerId,
        AccountNumber = account.AccountNumber,
        Balance = account.Balance,
        Status = account.Status,
        Version = account.Version,
        SnapshotAt = DateTime.UtcNow
    };
}

// EF Core Snapshot Store
public class EfSnapshotStore : ISnapshotStore
{
    private readonly EventStoreDbContext _dbContext;
    
    public EfSnapshotStore(EventStoreDbContext dbContext)
    {
        _dbContext = dbContext;
    }
    
    public async Task<T?> LoadAsync<T>(Guid streamId, CancellationToken ct = default) where T : class
    {
        var stored = await _dbContext.Snapshots
            .AsNoTracking()
            .Where(s => s.StreamId == streamId)
            .OrderByDescending(s => s.Version)
            .FirstOrDefaultAsync(ct);
        
        if (stored == null) return null;
        
        return JsonSerializer.Deserialize<T>(stored.Data);
    }
    
    public async Task SaveAsync<T>(Guid streamId, T snapshot, CancellationToken ct = default) where T : class
    {
        var version = snapshot is AccountSnapshot acSnap ? acSnap.Version : 0;
        
        _dbContext.Snapshots.Add(new StoredSnapshot
        {
            StreamId = streamId,
            Version = version,
            SnapshotType = typeof(T).Name,
            Data = JsonSerializer.Serialize(snapshot),
            CreatedAt = DateTime.UtcNow
        });
        
        await _dbContext.SaveChangesAsync(ct);
    }
}
```

---

## 5. Projections

Projections แปลง events เป็น read models ที่เหมาะสำหรับ query

```csharp
// Projections/AccountProjection.cs

// Read model
public class AccountReadModel
{
    public Guid AccountId { get; set; }
    public Guid OwnerId { get; set; }
    public string AccountNumber { get; set; } = string.Empty;
    public decimal Balance { get; set; }
    public string Status { get; set; } = string.Empty;
    public string AccountType { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

public class TransactionReadModel
{
    public Guid Id { get; set; }
    public Guid AccountId { get; set; }
    public string Type { get; set; } = string.Empty;
    public decimal Amount { get; set; }
    public string Description { get; set; } = string.Empty;
    public decimal BalanceAfter { get; set; }
    public DateTime OccurredAt { get; set; }
}

// Projection processor
public interface IProjection
{
    Task ProjectAsync(DomainEvent @event, CancellationToken ct = default);
}

public class AccountProjection : IProjection
{
    private readonly ReadDbContext _readDb;
    private readonly ILogger<AccountProjection> _logger;

    public AccountProjection(ReadDbContext readDb, ILogger<AccountProjection> logger)
    {
        _readDb = readDb;
        _logger = logger;
    }

    public async Task ProjectAsync(DomainEvent @event, CancellationToken ct = default)
    {
        switch (@event)
        {
            case AccountOpened e:
                await HandleAsync(e, ct);
                break;
            case MoneyDeposited e:
                await HandleAsync(e, ct);
                break;
            case MoneyWithdrawn e:
                await HandleAsync(e, ct);
                break;
            case MoneyTransferred e:
                await HandleAsync(e, ct);
                break;
            case AccountSuspended e:
                await HandleAsync(e, ct);
                break;
            case AccountClosed e:
                await HandleAsync(e, ct);
                break;
        }
    }

    private async Task HandleAsync(AccountOpened e, CancellationToken ct)
    {
        _readDb.Accounts.Add(new AccountReadModel
        {
            AccountId = e.AccountId,
            OwnerId = e.OwnerId,
            AccountNumber = e.AccountNumber,
            Balance = e.InitialDeposit,
            Status = "Active",
            AccountType = e.AccountType,
            CreatedAt = e.OccurredAt
        });
        await _readDb.SaveChangesAsync(ct);
    }

    private async Task HandleAsync(MoneyDeposited e, CancellationToken ct)
    {
        var account = await _readDb.Accounts.FindAsync(new object[] { e.AccountId }, ct);
        if (account != null)
        {
            account.Balance += e.Amount;
            account.UpdatedAt = e.OccurredAt;
        }
        
        _readDb.Transactions.Add(new TransactionReadModel
        {
            Id = e.EventId,
            AccountId = e.AccountId,
            Type = "Deposit",
            Amount = e.Amount,
            Description = e.Description,
            BalanceAfter = account?.Balance ?? 0,
            OccurredAt = e.OccurredAt
        });
        
        await _readDb.SaveChangesAsync(ct);
    }

    private async Task HandleAsync(MoneyWithdrawn e, CancellationToken ct)
    {
        var account = await _readDb.Accounts.FindAsync(new object[] { e.AccountId }, ct);
        if (account != null)
        {
            account.Balance -= e.Amount;
            account.UpdatedAt = e.OccurredAt;
        }
        
        _readDb.Transactions.Add(new TransactionReadModel
        {
            Id = e.EventId,
            AccountId = e.AccountId,
            Type = "Withdrawal",
            Amount = e.Amount,
            Description = e.Description,
            BalanceAfter = account?.Balance ?? 0,
            OccurredAt = e.OccurredAt
        });
        
        await _readDb.SaveChangesAsync(ct);
    }

    private async Task HandleAsync(AccountSuspended e, CancellationToken ct)
    {
        var account = await _readDb.Accounts.FindAsync(new object[] { e.AccountId }, ct);
        if (account != null)
        {
            account.Status = "Suspended";
            account.UpdatedAt = e.OccurredAt;
            await _readDb.SaveChangesAsync(ct);
        }
    }

    private async Task HandleAsync(AccountClosed e, CancellationToken ct)
    {
        var account = await _readDb.Accounts.FindAsync(new object[] { e.AccountId }, ct);
        if (account != null)
        {
            account.Status = "Closed";
            account.UpdatedAt = e.OccurredAt;
            await _readDb.SaveChangesAsync(ct);
        }
    }
}
```

---

## 6. CQRS + Event Sourcing

```csharp
// Commands
public record OpenAccountCommand(Guid OwnerId, string AccountType, decimal InitialDeposit);
public record DepositMoneyCommand(Guid AccountId, decimal Amount, string Description);
public record WithdrawMoneyCommand(Guid AccountId, decimal Amount, string Description);
public record TransferMoneyCommand(Guid SourceAccountId, Guid TargetAccountId, decimal Amount, string Description);

// Command handlers
public class AccountCommandHandler
{
    private readonly IAccountRepository _repository;
    private readonly IProjection _projection;
    private readonly ILogger<AccountCommandHandler> _logger;

    public AccountCommandHandler(
        IAccountRepository repository,
        IProjection projection,
        ILogger<AccountCommandHandler> logger)
    {
        _repository = repository;
        _projection = projection;
        _logger = logger;
    }

    public async Task<Guid> HandleAsync(OpenAccountCommand command, CancellationToken ct = default)
    {
        var account = Account.Open(command.OwnerId, command.AccountType, command.InitialDeposit);
        
        await _repository.SaveAsync(account, ct);
        
        // Project events to read model
        foreach (var @event in account.GetUncommittedEvents())
            await _projection.ProjectAsync(@event, ct);
        
        _logger.LogInformation("Opened account {AccountId} for {OwnerId}", 
            account.Id, command.OwnerId);
        
        return account.Id;
    }

    public async Task HandleAsync(DepositMoneyCommand command, CancellationToken ct = default)
    {
        var account = await _repository.LoadAsync(command.AccountId, ct)
            ?? throw new KeyNotFoundException($"Account {command.AccountId} not found");
        
        account.Deposit(command.Amount, command.Description, Guid.NewGuid().ToString());
        
        await _repository.SaveAsync(account, ct);
        
        foreach (var @event in account.GetUncommittedEvents())
            await _projection.ProjectAsync(@event, ct);
    }

    public async Task HandleAsync(WithdrawMoneyCommand command, CancellationToken ct = default)
    {
        var account = await _repository.LoadAsync(command.AccountId, ct)
            ?? throw new KeyNotFoundException($"Account {command.AccountId} not found");
        
        account.Withdraw(command.Amount, command.Description, Guid.NewGuid().ToString());
        await _repository.SaveAsync(account, ct);
        
        foreach (var @event in account.GetUncommittedEvents())
            await _projection.ProjectAsync(@event, ct);
    }
}

// Query handlers (read from projections)
public class AccountQueryHandler
{
    private readonly ReadDbContext _readDb;

    public AccountQueryHandler(ReadDbContext readDb)
    {
        _readDb = readDb;
    }

    public async Task<AccountReadModel?> GetAccountAsync(Guid accountId, CancellationToken ct)
    {
        return await _readDb.Accounts.AsNoTracking()
            .FirstOrDefaultAsync(a => a.AccountId == accountId, ct);
    }

    public async Task<IEnumerable<TransactionReadModel>> GetTransactionsAsync(
        Guid accountId, 
        int page = 1, 
        int pageSize = 20,
        CancellationToken ct = default)
    {
        return await _readDb.Transactions.AsNoTracking()
            .Where(t => t.AccountId == accountId)
            .OrderByDescending(t => t.OccurredAt)
            .Skip((page - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync(ct);
    }
    
    // Time travel - ดู balance ณ วันที่ต้องการ
    public async Task<decimal> GetBalanceAtAsync(
        Guid accountId,
        DateTime at,
        IEventStore eventStore,
        CancellationToken ct)
    {
        var events = await eventStore.LoadEventsAsync(accountId, ct);
        var eventsUpTo = events.Where(e => e.OccurredAt <= at);
        
        var account = Account.Reconstruct(eventsUpTo);
        return account.Balance;
    }
}
```

---

## 7. โปรแกรมตัวอย่าง: Account Balance System

### API Endpoints

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Event Store DB
builder.Services.AddDbContext<EventStoreDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("EventStore")));

// Read Model DB
builder.Services.AddDbContext<ReadDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("ReadDb")));

builder.Services.AddScoped<IEventStore, EfEventStore>();
builder.Services.AddScoped<ISnapshotStore, EfSnapshotStore>();
builder.Services.AddScoped<IAccountRepository, AccountRepository>();
builder.Services.AddScoped<IProjection, AccountProjection>();
builder.Services.AddScoped<AccountCommandHandler>();
builder.Services.AddScoped<AccountQueryHandler>();

var app = builder.Build();

// Command endpoints (write side)
var accountsGroup = app.MapGroup("/api/accounts");

accountsGroup.MapPost("/", async (
    OpenAccountRequest request,
    AccountCommandHandler handler,
    CancellationToken ct) =>
{
    var accountId = await handler.HandleAsync(
        new OpenAccountCommand(request.OwnerId, request.AccountType, request.InitialDeposit),
        ct);
    
    return Results.Created($"/api/accounts/{accountId}", new { AccountId = accountId });
});

accountsGroup.MapPost("/{accountId:guid}/deposit", async (
    Guid accountId,
    DepositRequest request,
    AccountCommandHandler handler,
    CancellationToken ct) =>
{
    await handler.HandleAsync(
        new DepositMoneyCommand(accountId, request.Amount, request.Description),
        ct);
    
    return Results.Ok(new { Message = "Deposit successful" });
});

accountsGroup.MapPost("/{accountId:guid}/withdraw", async (
    Guid accountId,
    WithdrawRequest request,
    AccountCommandHandler handler,
    CancellationToken ct) =>
{
    await handler.HandleAsync(
        new WithdrawMoneyCommand(accountId, request.Amount, request.Description),
        ct);
    
    return Results.Ok(new { Message = "Withdrawal successful" });
});

// Query endpoints (read side)
accountsGroup.MapGet("/{accountId:guid}", async (
    Guid accountId,
    AccountQueryHandler handler,
    CancellationToken ct) =>
{
    var account = await handler.GetAccountAsync(accountId, ct);
    return account is null ? Results.NotFound() : Results.Ok(account);
});

accountsGroup.MapGet("/{accountId:guid}/transactions", async (
    Guid accountId,
    AccountQueryHandler handler,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    CancellationToken ct = default) =>
{
    var transactions = await handler.GetTransactionsAsync(accountId, page, pageSize, ct);
    return Results.Ok(transactions);
});

// Time travel query
accountsGroup.MapGet("/{accountId:guid}/balance-at", async (
    Guid accountId,
    [FromQuery] DateTime at,
    AccountQueryHandler handler,
    IEventStore eventStore,
    CancellationToken ct) =>
{
    var balance = await handler.GetBalanceAtAsync(accountId, at, eventStore, ct);
    return Results.Ok(new { AccountId = accountId, At = at, Balance = balance });
});

app.Run();

// Request types
public record OpenAccountRequest(Guid OwnerId, string AccountType, decimal InitialDeposit = 0);
public record DepositRequest(decimal Amount, string Description);
public record WithdrawRequest(decimal Amount, string Description);
```

---

## Exercises / Project Tasks

### Exercise 1: Shopping Cart with Events
ออกแบบ event-sourced shopping cart:
- CartCreated
- ItemAdded, ItemRemoved, ItemQuantityChanged
- CartCheckedOut, CartAbandoned

### Exercise 2: Replay Events
สร้าง mechanism สำหรับ replay:
- Rebuild read model จาก event store
- Test ว่า balance ถูกต้อง

### Exercise 3: Snapshot Strategy
ทดสอบ snapshot performance:
- เปรียบเทียบเวลา load ก่อน/หลัง snapshot
- ทดสอบด้วย 100, 1000, 10000 events

### Exercise 4: Event Versioning
จัดการ event schema changes:
- V1: MoneyDeposited { amount: 100 }
- V2: MoneyDeposited { amount: 100, currency: "THB" }
- สร้าง upcaster สำหรับ V1 → V2

---

## สรุป

- **Event Sourcing** เก็บ state เป็น sequence ของ events แทน current state
- **Events เป็น facts** ที่ไม่เปลี่ยนแปลง, เก็บ "what happened" ไว้ตลอด
- **Aggregates** apply events เพื่อ rebuild state
- **Event Store** เป็น append-only store สำหรับ events
- **Projections** แปลง events เป็น read models ที่เหมาะสำหรับ query
- **Snapshots** เพิ่ม performance สำหรับ aggregates ที่มี events เยอะ
- **CQRS + Event Sourcing** แยก read/write model ให้ optimize แต่ละด้านได้

---

## Part ถัดไป

**Part 092: Observability: Metrics, Tracing, Logging** - เรียนรู้การ monitor production systems

---

*Part 091/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
