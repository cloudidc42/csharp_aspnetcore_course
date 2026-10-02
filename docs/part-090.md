# Part 090: GraphQL ใน .NET

## เนื้อหาใน Part นี้
- GraphQL vs REST
- Hot Chocolate library
- Queries, Mutations, Subscriptions
- Types, Resolvers
- DataLoader
- โปรแกรมตัวอย่าง: GraphQL API

---

## 1. GraphQL vs REST

GraphQL เป็น query language สำหรับ APIs ที่ให้ client กำหนดเองว่าต้องการ data อะไร

### ปัญหาของ REST ที่ GraphQL แก้

```
Over-fetching: ดึงข้อมูลมากเกิน
GET /api/users/123
Response: { id, name, email, address, phone, createdAt, ... }
แต่ต้องการแค่ name และ email

Under-fetching: ต้องเรียกหลาย endpoints
GET /api/orders/1
GET /api/orders/1/items
GET /api/users/456
GET /api/products/789

GraphQL:
query {
  order(id: "1") {
    id
    status
    items {
      product { name, price }
      quantity
    }
    customer { name, email }
  }
}
```

### GraphQL Operations

```graphql
# Query - อ่านข้อมูล
query GetProduct($id: ID!) {
  product(id: $id) {
    id
    name
    price
    reviews {
      rating
      text
    }
  }
}

# Mutation - เขียนข้อมูล
mutation CreateProduct($input: CreateProductInput!) {
  createProduct(input: $input) {
    id
    name
    price
  }
}

# Subscription - real-time updates
subscription OnOrderStatusChanged($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    status
    updatedAt
  }
}
```

---

## 2. Hot Chocolate Library

Hot Chocolate เป็น GraphQL server ที่ feature-rich สำหรับ .NET

### การติดตั้ง

```bash
dotnet new web -n ProductGraphQL
dotnet add package HotChocolate.AspNetCore
dotnet add package HotChocolate.Data
dotnet add package HotChocolate.Data.EntityFramework
dotnet add package HotChocolate.Subscriptions.InMemory
```

### Schema-First vs Code-First

Hot Chocolate รองรับทั้ง 2 แบบ

**Code-First (แนะนำ):**
```csharp
// Schema สร้างจาก C# classes โดยอัตโนมัติ
[GraphQLName("Product")]
public class ProductType
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}
```

**Schema-First:**
```graphql
type Product {
  id: ID!
  name: String!
  price: Float!
}
```

---

## 3. Types และ Schema

### Type Definitions

```csharp
// Types/ProductType.cs
using HotChocolate.Types;

// Entity
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public string Category { get; set; } = string.Empty;
    public int Stock { get; set; }
    public bool IsActive { get; set; }
    public DateTime CreatedAt { get; set; }
    
    // Navigation
    public ICollection<Review> Reviews { get; set; } = new List<Review>();
    public Guid CategoryId { get; set; }
}

// GraphQL type configuration
public class ProductType : ObjectType<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor.Description("Represents a product in the catalog");
        
        descriptor.Field(p => p.Id)
            .Description("The unique identifier");
        
        descriptor.Field(p => p.Name)
            .Description("The product name");
        
        descriptor.Field(p => p.Price)
            .Description("The product price in THB");
        
        // Computed field
        descriptor.Field("displayPrice")
            .Type<NonNullType<StringType>>()
            .Description("Formatted price")
            .Resolve(ctx =>
            {
                var product = ctx.Parent<Product>();
                return $"฿{product.Price:N2}";
            });
        
        // Hide internal field
        descriptor.Field(p => p.IsActive).Ignore();
    }
}

// Input types สำหรับ Mutations
public record CreateProductInput(
    string Name,
    string? Description,
    decimal Price,
    string Category,
    int InitialStock = 0);

public record UpdateProductInput(
    Guid Id,
    string? Name,
    string? Description,
    decimal? Price,
    int? Stock);

// Connection type สำหรับ pagination
public class ProductFilterInput : FilterInputType<Product>
{
    protected override void Configure(IFilterInputTypeDescriptor<Product> descriptor)
    {
        descriptor.Field(p => p.Name).Name("name");
        descriptor.Field(p => p.Price).Name("price");
        descriptor.Field(p => p.Category).Name("category");
    }
}

public class ProductSortInput : SortInputType<Product>
{
    protected override void Configure(ISortInputTypeDescriptor<Product> descriptor)
    {
        descriptor.Field(p => p.Name).Name("name");
        descriptor.Field(p => p.Price).Name("price");
        descriptor.Field(p => p.CreatedAt).Name("createdAt");
    }
}
```

---

## 4. Queries

```csharp
// Queries/ProductQuery.cs
using HotChocolate.Data;

[QueryType]
public class ProductQuery
{
    // Single product
    [GraphQLDescription("Get a product by ID")]
    [UseFirstOrDefault]
    [UseProjection]
    public IQueryable<Product> GetProduct(
        Guid id,
        AppDbContext dbContext)
    {
        return dbContext.Products.Where(p => p.Id == id && p.IsActive);
    }

    // List with filtering, sorting, pagination
    [GraphQLDescription("Get paginated list of products")]
    [UsePaging(MaxPageSize = 50)]
    [UseProjection]
    [UseFiltering<ProductFilterInput>]
    [UseSorting<ProductSortInput>]
    public IQueryable<Product> GetProducts(AppDbContext dbContext)
    {
        return dbContext.Products
            .AsNoTracking()
            .Where(p => p.IsActive);
    }

    // Search
    [GraphQLDescription("Search products by text")]
    public async Task<IEnumerable<Product>> SearchProducts(
        string query,
        AppDbContext dbContext,
        CancellationToken cancellationToken)
    {
        return await dbContext.Products
            .AsNoTracking()
            .Where(p => p.IsActive && 
                (p.Name.Contains(query) || 
                 (p.Description != null && p.Description.Contains(query))))
            .OrderByDescending(p => EF.Functions.Like(p.Name, $"%{query}%"))
            .Take(20)
            .ToListAsync(cancellationToken);
    }

    // Categories
    public async Task<IEnumerable<string>> GetCategories(
        AppDbContext dbContext,
        CancellationToken ct)
    {
        return await dbContext.Products
            .AsNoTracking()
            .Where(p => p.IsActive)
            .Select(p => p.Category)
            .Distinct()
            .OrderBy(c => c)
            .ToListAsync(ct);
    }
}
```

### Nested Resolvers

```csharp
// Resolvers/ProductReviewResolver.cs
public class ReviewExtension : ObjectTypeExtension<Product>
{
    protected override void Configure(IObjectTypeDescriptor<Product> descriptor)
    {
        descriptor
            .Field("reviews")
            .Description("Get product reviews")
            .Argument("limit", a => a.DefaultValue(10))
            .Type<ListType<NonNullType<ReviewType>>>()
            .Resolve(async (ctx, ct) =>
            {
                var product = ctx.Parent<Product>();
                var dbContext = ctx.Service<AppDbContext>();
                var limit = ctx.ArgumentValue<int>("limit");
                
                return await dbContext.Reviews
                    .AsNoTracking()
                    .Where(r => r.ProductId == product.Id)
                    .OrderByDescending(r => r.CreatedAt)
                    .Take(limit)
                    .ToListAsync(ct);
            });
        
        descriptor
            .Field("averageRating")
            .Type<FloatType>()
            .Resolve(async (ctx, ct) =>
            {
                var product = ctx.Parent<Product>();
                var dbContext = ctx.Service<AppDbContext>();
                
                var avg = await dbContext.Reviews
                    .Where(r => r.ProductId == product.Id)
                    .AverageAsync(r => (double?)r.Rating, ct);
                
                return avg;
            });
    }
}
```

---

## 5. Mutations

```csharp
// Mutations/ProductMutation.cs
[MutationType]
public class ProductMutation
{
    [GraphQLDescription("Create a new product")]
    [Error<ProductNameAlreadyExistsError>]
    [Error<ValidationError>]
    public async Task<Product> CreateProduct(
        CreateProductInput input,
        AppDbContext dbContext,
        [Service] IValidator<CreateProductInput> validator,
        CancellationToken ct)
    {
        // Validate
        var validationResult = await validator.ValidateAsync(input, ct);
        if (!validationResult.IsValid)
        {
            throw new ValidationError(validationResult.Errors
                .Select(e => e.ErrorMessage).ToList());
        }
        
        // Check duplicate name
        if (await dbContext.Products.AnyAsync(p => p.Name == input.Name, ct))
        {
            throw new ProductNameAlreadyExistsError(input.Name);
        }

        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = input.Name,
            Description = input.Description,
            Price = input.Price,
            Category = input.Category,
            Stock = input.InitialStock,
            IsActive = true,
            CreatedAt = DateTime.UtcNow
        };

        dbContext.Products.Add(product);
        await dbContext.SaveChangesAsync(ct);

        return product;
    }

    [GraphQLDescription("Update an existing product")]
    public async Task<Product?> UpdateProduct(
        UpdateProductInput input,
        AppDbContext dbContext,
        CancellationToken ct)
    {
        var product = await dbContext.Products.FindAsync(new object[] { input.Id }, ct);
        
        if (product == null) return null;
        
        if (input.Name != null) product.Name = input.Name;
        if (input.Description != null) product.Description = input.Description;
        if (input.Price.HasValue) product.Price = input.Price.Value;
        if (input.Stock.HasValue) product.Stock = input.Stock.Value;
        
        await dbContext.SaveChangesAsync(ct);
        return product;
    }

    [GraphQLDescription("Delete a product")]
    public async Task<bool> DeleteProduct(
        Guid id,
        AppDbContext dbContext,
        CancellationToken ct)
    {
        var product = await dbContext.Products.FindAsync(new object[] { id }, ct);
        if (product == null) return false;
        
        product.IsActive = false;  // Soft delete
        await dbContext.SaveChangesAsync(ct);
        return true;
    }
    
    [GraphQLDescription("Add a review to a product")]
    public async Task<Review> AddReview(
        Guid productId,
        string text,
        int rating,
        AppDbContext dbContext,
        [GlobalState("userId")] Guid userId,
        CancellationToken ct)
    {
        if (rating < 1 || rating > 5)
            throw new ArgumentException("Rating must be between 1 and 5");
        
        var review = new Review
        {
            Id = Guid.NewGuid(),
            ProductId = productId,
            UserId = userId,
            Text = text,
            Rating = rating,
            CreatedAt = DateTime.UtcNow
        };
        
        dbContext.Reviews.Add(review);
        await dbContext.SaveChangesAsync(ct);
        
        return review;
    }
}

// Error types
public class ProductNameAlreadyExistsError : Exception
{
    public string ProductName { get; }
    
    public ProductNameAlreadyExistsError(string productName)
        : base($"A product with name '{productName}' already exists")
    {
        ProductName = productName;
    }
}

public class ValidationError : Exception
{
    public IReadOnlyList<string> Errors { get; }
    
    public ValidationError(List<string> errors)
        : base("Validation failed")
    {
        Errors = errors;
    }
}
```

---

## 6. Subscriptions

```csharp
// Subscriptions/ProductSubscription.cs
using HotChocolate.Subscriptions;

[SubscriptionType]
public class ProductSubscription
{
    [Subscribe]
    [Topic("{orderId}")]
    [GraphQLDescription("Subscribe to order status changes")]
    public async Task<Order> OnOrderStatusChanged(
        [EventMessage] Order order,
        Guid orderId)
    {
        return order;
    }
    
    [Subscribe]
    [Topic]
    public async Task<Product> OnProductCreated(
        [EventMessage] Product product)
    {
        return product;
    }
    
    [Subscribe]
    [Topic]
    public async Task<ProductPriceChanged> OnProductPriceChanged(
        [EventMessage] ProductPriceChanged change,
        Guid productId)
    {
        return change;
    }
}

public record ProductPriceChanged(Guid ProductId, decimal OldPrice, decimal NewPrice);

// Publishing events ใน Mutation
public class ProductMutationWithSubscription
{
    public async Task<Product> UpdateProductPrice(
        Guid productId,
        decimal newPrice,
        AppDbContext dbContext,
        [Service] ITopicEventSender eventSender,
        CancellationToken ct)
    {
        var product = await dbContext.Products.FindAsync(new object[] { productId }, ct)
            ?? throw new KeyNotFoundException($"Product {productId} not found");
        
        var oldPrice = product.Price;
        product.Price = newPrice;
        await dbContext.SaveChangesAsync(ct);
        
        // Send subscription event
        await eventSender.SendAsync(
            $"OnProductPriceChanged_{productId}",
            new ProductPriceChanged(productId, oldPrice, newPrice),
            ct);
        
        return product;
    }
}
```

---

## 7. DataLoader

DataLoader แก้ N+1 problem ใน GraphQL

```csharp
// DataLoaders/ProductDataLoader.cs
using GreenDonut;

// Batch DataLoader
public class ProductByIdDataLoader : BatchDataLoader<Guid, Product>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public ProductByIdDataLoader(
        IDbContextFactory<AppDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<IReadOnlyDictionary<Guid, Product>> LoadBatchAsync(
        IReadOnlyList<Guid> keys,
        CancellationToken cancellationToken)
    {
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);
        
        // แทนที่จะ query แยกกัน N ครั้ง, query ครั้งเดียว
        return await dbContext.Products
            .AsNoTracking()
            .Where(p => keys.Contains(p.Id))
            .ToDictionaryAsync(p => p.Id, cancellationToken);
    }
}

// Review DataLoader (one-to-many)
public class ReviewsByProductIdDataLoader : GroupedDataLoader<Guid, Review>
{
    private readonly IDbContextFactory<AppDbContext> _dbContextFactory;

    public ReviewsByProductIdDataLoader(
        IDbContextFactory<AppDbContext> dbContextFactory,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _dbContextFactory = dbContextFactory;
    }

    protected override async Task<ILookup<Guid, Review>> LoadGroupedBatchAsync(
        IReadOnlyList<Guid> keys,
        CancellationToken cancellationToken)
    {
        await using var dbContext = await _dbContextFactory.CreateDbContextAsync(cancellationToken);
        
        var reviews = await dbContext.Reviews
            .AsNoTracking()
            .Where(r => keys.Contains(r.ProductId))
            .ToListAsync(cancellationToken);
        
        return reviews.ToLookup(r => r.ProductId);
    }
}

// ใช้ DataLoader ใน Resolver
public class OrderItemType : ObjectType<OrderItem>
{
    protected override void Configure(IObjectTypeDescriptor<OrderItem> descriptor)
    {
        descriptor
            .Field(i => i.ProductId)
            .Ignore();
        
        descriptor
            .Field("product")
            .ResolveWith<OrderItemResolvers>(r => r.GetProductAsync(default!, default!, default!));
    }
}

public class OrderItemResolvers
{
    public async Task<Product?> GetProductAsync(
        [Parent] OrderItem orderItem,
        ProductByIdDataLoader productLoader,
        CancellationToken ct)
    {
        // DataLoader batches multiple productId lookups into single query
        return await productLoader.LoadAsync(orderItem.ProductId, ct);
    }
}
```

---

## 8. Authentication กับ GraphQL

```csharp
// Authorization ใน GraphQL
[QueryType]
public class SecuredQuery
{
    [Authorize]  // ต้องล็อกอิน
    public async Task<IEnumerable<Order>> GetMyOrders(
        [GlobalState("userId")] Guid userId,
        AppDbContext dbContext,
        CancellationToken ct)
    {
        return await dbContext.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == userId)
            .ToListAsync(ct);
    }
    
    [Authorize(Roles = new[] { "Admin" })]  // Admin เท่านั้น
    public async Task<IEnumerable<User>> GetAllUsers(
        AppDbContext dbContext,
        CancellationToken ct)
    {
        return await dbContext.Users.AsNoTracking().ToListAsync(ct);
    }
}

// Program.cs - Setup
builder.Services
    .AddGraphQLServer()
    .AddQueryType<ProductQuery>()
    .AddMutationType<ProductMutation>()
    .AddSubscriptionType<ProductSubscription>()
    .AddTypeExtension<ReviewExtension>()
    .AddType<ProductType>()
    .AddProjections()
    .AddFiltering<CustomFilterConvention>()
    .AddSorting()
    .AddAuthorization()
    .AddInMemorySubscriptions()
    .AddDbContextCursorPagingProvider()
    .RegisterDbContextFactory<AppDbContext>()
    .AddDataLoader<ProductByIdDataLoader>()
    .AddDataLoader<ReviewsByProductIdDataLoader>();

// Authentication state ใน GraphQL context
builder.Services.AddHttpContextAccessor();
builder.Services.AddGlobalObjectIdentifier();
```

---

## 9. โปรแกรมตัวอย่าง: GraphQL API สมบูรณ์

### Program.cs

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContextFactory<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Authentication
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
        
        options.Events = new JwtBearerEvents
        {
            OnMessageReceived = context =>
            {
                // Support token in query string for subscriptions
                var token = context.Request.Query["access_token"];
                var path = context.HttpContext.Request.Path;
                if (!string.IsNullOrEmpty(token) && path.StartsWithSegments("/graphql"))
                {
                    context.Token = token;
                }
                return Task.CompletedTask;
            }
        };
    });

builder.Services.AddAuthorization();

// GraphQL
builder.Services
    .AddGraphQLServer()
    .AddQueryType<ProductQuery>()
    .AddMutationType<ProductMutation>()
    .AddSubscriptionType<ProductSubscription>()
    .AddTypeExtension<ReviewExtension>()
    .AddTypeExtension<OrderItemResolvers>()
    // Projections, Filtering, Sorting
    .AddProjections()
    .AddFiltering()
    .AddSorting()
    // Authorization
    .AddAuthorization()
    // Subscriptions
    .AddInMemorySubscriptions()
    // Pagination
    .SetPagingOptions(new PagingOptions
    {
        MaxPageSize = 50,
        DefaultPageSize = 20,
        IncludeTotalCount = true
    })
    // DataLoaders
    .AddDataLoader<ProductByIdDataLoader>()
    .AddDataLoader<ReviewsByProductIdDataLoader>()
    // Error handling
    .AddErrorFilter<GraphQLErrorFilter>()
    // Performance
    .UseQueryDepth(10)  // ป้องกัน deeply nested queries
    .UseQueryComplexity(100)  // ป้องกัน expensive queries
    // Persisted queries
    .UsePersistedQueryPipeline()
    .AddReadOnlyFileSystemQueryStorage("./queries")
    // Introspection (disable in production)
    .ModifyOptions(o => o.EnableSchemaIntrospection = builder.Environment.IsDevelopment());

var app = builder.Build();

app.UseWebSockets();
app.UseAuthentication();
app.UseAuthorization();

// GraphQL endpoint
app.MapGraphQL("/graphql");

// Banana Cake Pop (GraphQL IDE) - development only
if (app.Environment.IsDevelopment())
{
    app.MapBananaCakePop("/graphql-ui");
}

app.Run();

// Error filter
public class GraphQLErrorFilter : IErrorFilter
{
    private readonly ILogger<GraphQLErrorFilter> _logger;
    private readonly IWebHostEnvironment _env;
    
    public GraphQLErrorFilter(ILogger<GraphQLErrorFilter> logger, IWebHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }
    
    public IError OnError(IError error)
    {
        if (error.Exception is not null)
        {
            _logger.LogError(error.Exception, "GraphQL error: {Message}", error.Message);
            
            if (!_env.IsDevelopment())
            {
                // Hide implementation details ใน production
                return error.WithMessage("An internal error occurred");
            }
        }
        
        return error;
    }
}
```

### ตัวอย่าง GraphQL Queries

```graphql
# Get products with filtering and sorting
query GetProducts {
  products(
    where: { 
      category: { eq: "Electronics" },
      price: { gte: 100, lte: 1000 }
    }
    order: [{ price: ASC }]
    first: 10
  ) {
    totalCount
    pageInfo {
      hasNextPage
      endCursor
    }
    nodes {
      id
      name
      displayPrice
      averageRating
      reviews(limit: 3) {
        rating
        text
      }
    }
  }
}

# Create product mutation
mutation CreateProduct {
  createProduct(input: {
    name: "iPhone 15 Pro"
    description: "Latest iPhone model"
    price: 38900
    category: "Electronics"
    initialStock: 100
  }) {
    id
    name
    price
    createdAt
  }
}

# Subscribe to price changes
subscription WatchPriceChange {
  onProductPriceChanged(productId: "some-guid") {
    productId
    oldPrice
    newPrice
  }
}
```

---

## Exercises / Project Tasks

### Exercise 1: Basic GraphQL
สร้าง GraphQL API สำหรับ Blog:
- Query: posts, post(id), comments(postId)
- Mutation: createPost, updatePost, deletePost, addComment
- Filtering by author, category

### Exercise 2: DataLoader
ใช้ DataLoader เพื่อแก้ N+1:
- AuthorByIdDataLoader
- CommentsByPostIdDataLoader
- ทดสอบว่า query count ลดลง

### Exercise 3: Subscriptions
เพิ่ม real-time features:
- Subscribe to new comments
- Subscribe to post likes
- Test ด้วย Banana Cake Pop

### Exercise 4: Authorization
เพิ่ม authorization:
- ต้อง login ถึงจะ create/update
- Admin เท่านั้นที่ delete ได้
- User เห็นเฉพาะ public posts

---

## สรุป

- **GraphQL** ให้ client กำหนดว่าต้องการ data อะไร แก้ over/under-fetching
- **Hot Chocolate** เป็น GraphQL server ที่ full-featured สำหรับ .NET
- **Queries** สำหรับ read, **Mutations** สำหรับ write, **Subscriptions** สำหรับ real-time
- **DataLoader** แก้ N+1 ด้วย batching requests
- **Filtering/Sorting/Pagination** built-in ด้วย Hot Chocolate
- **Authorization** ที่ field level ทำให้ fine-grained access control

---

## Part ถัดไป

**Part 091: Event Sourcing** - เรียนรู้ pattern ที่เก็บ history ของ state changes

---

*Part 090/100 | Phase 6/7: ระดับสูง | หลักสูตร C# และ ASP.NET Core*
