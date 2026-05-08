# OrderHub — C# / .NET Naming Conventions (Vertical Slice Architecture)

This document defines the project structure, naming standards, and code conventions for the OrderHub backend (`orderhub-api` repository, `OrderHub.API` solution). It uses **Vertical Slice Architecture (VSA)** — code is organized by *feature*, not by technical layer.

## Stack assumptions

- **Runtime:** .NET 10 (LTS, supported through November 2028)
- **Solution:** `OrderHub.API.sln`
- **Architecture:** Vertical Slice Architecture (VSA)
- **API style:** Minimal APIs (not Controllers)
- **ORM:** Entity Framework Core 10 (used directly — no repository pattern)
- **Validation:** FluentValidation
- **Testing:** xUnit + FluentAssertions + (Moq or NSubstitute)
- **Logging / Observability:** Built-in `ILogger<T>` + Azure Application Insights (no Serilog)

## Why VSA

OrderHub uses Vertical Slice Architecture because it optimizes for **feature delivery speed** and **code locality** — the two things that matter most for a SaaS product being actively built.

**Compared to traditional Clean Architecture:**
- Adding a feature touches **one folder**, not 4+ projects
- Code that changes together lives together
- Features can be deleted by removing a folder
- Each slice can be as simple or complex as it genuinely needs to be
- No premature abstraction (repositories, mappers, application services)

**The tradeoff we accept:** Less prescriptive structure, requires team discipline around when to extract shared code, slightly less name recognition than Clean Architecture. We mitigate this with strong reference slices and clear conventions in this document.

## Solution structure

### Repository layout

```
orderhub-api/                          ← repo (lowercase-kebab)
├── .github/
│   └── workflows/                     ← GitHub Actions CI/CD
├── src/
│   ├── OrderHub.API/                  ← Web API + all feature slices
│   ├── OrderHub.Domain/               ← Entities, value objects, domain logic
│   └── OrderHub.Infrastructure/       ← EF Core, external service clients
├── tests/
│   ├── OrderHub.API.Tests/            ← Slice tests (or co-located in Features/)
│   └── OrderHub.Domain.Tests/         ← Domain logic tests
├── OrderHub.API.sln
├── README.md
├── .editorconfig                      ← Code style enforcement
└── Directory.Build.props              ← Shared MSBuild properties
```

### Three projects, no separate Application layer

| Project | Purpose | Depends on |
|---|---|---|
| `OrderHub.API` | All feature slices, endpoint registration, middleware, DI | Domain, Infrastructure |
| `OrderHub.Domain` | Entities, value objects, enums, domain rules | (nothing) |
| `OrderHub.Infrastructure` | `DbContext`, EF configurations, external service adapters | Domain |

**Dependency direction:**

```
OrderHub.API ─────────► OrderHub.Domain
     │                        ▲
     ▼                        │
OrderHub.Infrastructure ──────┘
```

**Key rule:** `Domain` depends on **nothing**. Slices in `API` use `DbContext` directly from `Infrastructure`.

### Why no separate `OrderHub.Application` project

In Clean Architecture, "application logic" is a separate layer. In VSA, application logic **is the slice**. There's no orchestration layer that needs its own project — each feature contains its own orchestration inline.

If genuinely-shared logic emerges across slices, it goes in:
- **`OrderHub.Domain`** if it's a business rule
- **`OrderHub.API/Common/`** if it's infrastructure plumbing (e.g., a base validator, a shared error helper)

### When to add more projects

Don't pre-split. Add new projects only when there's a clear reason:

| Project | When to add |
|---|---|
| `OrderHub.Worker` | Background jobs that run as a separate process |
| `OrderHub.Migrations` | If migrations need to ship as a separate executable |
| `OrderHub.Contracts` | If you publish a NuGet package of DTOs for external customers |
| `OrderHub.IntegrationTests` | When integration tests grow large enough to separate |

## Project naming

All projects follow `OrderHub.<Name>` and live in a folder of the same name. The solution file is `OrderHub.API.sln`.

## Folder structure within `OrderHub.API`

```
OrderHub.API/
├── Features/                          ← All feature slices live here
│   ├── Orders/
│   │   ├── CreateOrder/
│   │   │   ├── CreateOrder.cs         ← Complete slice (Command, Validator, Handler, Endpoint)
│   │   │   └── CreateOrderTests.cs    ← Optional co-located tests
│   │   ├── GetOrder/
│   │   │   ├── GetOrder.cs
│   │   │   └── GetOrderTests.cs
│   │   ├── ListOrders/
│   │   │   └── ListOrders.cs
│   │   ├── CancelOrder/
│   │   │   └── CancelOrder.cs
│   │   └── ShipOrder/
│   │       └── ShipOrder.cs
│   ├── Users/
│   │   ├── RegisterUser/
│   │   ├── UpdateProfile/
│   │   └── ChangePassword/
│   ├── Products/
│   └── Payments/
├── Common/                            ← Cross-cutting, shared by multiple slices
│   ├── Endpoints/
│   │   └── EndpointRegistration.cs    ← Maps all slice endpoints
│   ├── Errors/
│   │   └── ProblemDetailsExtensions.cs
│   ├── Validation/
│   │   └── ValidationExtensions.cs
│   ├── Authentication/
│   │   └── CurrentUserService.cs
│   └── Behaviors/                     ← (only if needed)
├── Configuration/                     ← Strongly-typed config classes
│   ├── JwtSettings.cs
│   └── CorsSettings.cs
├── appsettings.json
├── appsettings.Development.json
├── appsettings.Production.json
├── Program.cs
└── OrderHub.API.csproj
```

## Folder structure within `OrderHub.Domain`

```
OrderHub.Domain/
├── Entities/                          ← Order, User, Product, etc.
│   ├── Order.cs
│   ├── OrderItem.cs
│   ├── User.cs
│   └── Product.cs
├── Enums/                             ← OrderStatus, PaymentMethod, UserRole
├── ValueObjects/                      ← Money, Address, EmailAddress
├── Exceptions/
│   └── DomainException.cs
└── OrderHub.Domain.csproj
```

## Folder structure within `OrderHub.Infrastructure`

```
OrderHub.Infrastructure/
├── Persistence/
│   ├── OrderHubDbContext.cs
│   ├── Configurations/                ← EF Core entity configurations
│   │   ├── OrderConfiguration.cs
│   │   ├── UserConfiguration.cs
│   │   └── ProductConfiguration.cs
│   └── Migrations/                    ← Auto-generated EF migrations
├── ExternalServices/
│   ├── Email/
│   │   └── SendGridEmailService.cs
│   ├── Storage/
│   │   └── AzureBlobStorageService.cs
│   └── Payments/
│       └── StripePaymentService.cs
├── Authentication/
│   └── JwtTokenService.cs
├── DependencyInjection.cs             ← Service registration extension
└── OrderHub.Infrastructure.csproj
```

## Vertical slice conventions

This is the heart of the document. **Read this section carefully** — it defines how every feature in OrderHub is organized.

### Slice = one folder per feature

Each feature gets its own folder under `Features/<Domain>/<FeatureName>/`. Folder name uses **PascalCase**, matches the operation (e.g., `CreateOrder`, `GetOrder`, `CancelOrder`).

```
Features/Orders/CreateOrder/
Features/Orders/GetOrder/
Features/Orders/CancelOrder/
Features/Users/RegisterUser/
Features/Users/ChangePassword/
```

### One file per slice (default)

By default, **everything for a slice lives in a single `.cs` file** named after the feature:

```
Features/Orders/CreateOrder/CreateOrder.cs
```

This file contains:
1. The `Command` or `Query` record (request shape)
2. Sub-records (e.g., `OrderItemRequest`)
3. The `Response` record (response shape)
4. The `Validator` class (FluentValidation)
5. The `HandleAsync` method (business logic)
6. The `MapEndpoint` method (route registration)

**Why one file:**
- Everything for the feature in one place — no tab-switching while developing
- Easy navigation — open one file, see the whole slice
- Easy code review — reviewer reads one file, sees full context
- Easy deletion — remove the folder, feature is gone
- Faster development — no boilerplate of creating multiple files for each feature

### Standard file structure (top to bottom)

Always organize the slice file in this exact order. Predictability beats personal preference.

```csharp
namespace OrderHub.API.Features.<Domain>.<FeatureName>;

public static class <FeatureName>
{
    // 1. Request shape (Command for writes, Query for reads)
    public record Command(...);

    // 2. Sub-records used by the request
    public record OrderItemRequest(int ProductId, int Quantity);

    // 3. Response shape
    public record Response(...);

    // 4. Validator (FluentValidation)
    public class Validator : AbstractValidator<Command> { ... }

    // 5. Handler (the actual business logic)
    public static async Task<IResult> HandleAsync(...) { ... }

    // 6. Endpoint registration
    public static void MapEndpoint(IEndpointRouteBuilder app) { ... }
}
```

After 5 slices written, this becomes muscle memory.

### Command vs. Query — naming by intent

Use **`Command`** for writes, **`Query`** for reads. This is a critical signal of intent.

| Operation type | Record name | Example |
|---|---|---|
| Create / Update / Delete / state change | `Command` | `public record Command(int CustomerId, ...);` |
| Get / List / Search / read-only | `Query` | `public record Query(int OrderId);` |

```csharp
// Write slice
namespace OrderHub.API.Features.Orders.CreateOrder;
public static class CreateOrder
{
    public record Command(int CustomerId, List<OrderItemRequest> Items);
    // ...
}

// Read slice
namespace OrderHub.API.Features.Orders.GetOrder;
public static class GetOrder
{
    public record Query(int OrderId);
    // ...
}
```

### Class container: static class with the same name as the feature

The outer container is a **`public static class`** with the same name as the feature folder. This keeps the namespace clean and groups related types under a single scope.

```csharp
// Folder:    Features/Orders/CreateOrder/
// File:      CreateOrder.cs
// Namespace: OrderHub.API.Features.Orders.CreateOrder
// Class:     public static class CreateOrder
```

The handler and endpoint methods become `CreateOrder.HandleAsync(...)` and `CreateOrder.MapEndpoint(app)` — readable, discoverable.

### When to split into multiple files

Split only when one of these is genuinely true:

| Condition | What to do |
|---|---|
| Handler logic exceeds ~150 lines | Move to `<Feature>.Handler.cs` |
| Validator has 30+ rules | Move to `<Feature>.Validator.cs` |
| Multiple slices share business logic | Extract to `OrderHub.Domain` (entity method or domain service) |
| The Response shape is reused by another slice | Move to `Common/Models/` |

When splitting, use the naming pattern `<Feature>.<Concern>.cs`:

```
Features/Orders/CreateOrder/
├── CreateOrder.cs              ← Command, Response, Validator, Endpoint
└── CreateOrder.Handler.cs      ← Just the handler (when it's huge)
```

**Default to NOT splitting.** A 250-line single-file slice is easier to navigate than five 50-line files. Split only when the file genuinely becomes hard to scan.

### The "extract" rule

The biggest VSA failure mode is over-eager extraction of "shared services" before duplication actually exists. The rule:

> **Wait until you see the same pattern duplicated three times before extracting it.**

Two duplicates might be coincidence. Three is a pattern worth abstracting. Pre-extracting based on imagined future reuse leads to the same over-engineered mess Clean Architecture suffers from.

## Complete reference slices

These are templates the team can copy-paste for new features.

### Reference slice 1: Write operation (Command)

```csharp
// File: Features/Orders/CreateOrder/CreateOrder.cs
using FluentValidation;
using OrderHub.Domain.Entities;
using OrderHub.Domain.Enums;
using OrderHub.Infrastructure.Persistence;

namespace OrderHub.API.Features.Orders.CreateOrder;

public static class CreateOrder
{
    public record Command(
        int CustomerId,
        List<OrderItemRequest> Items,
        string ShippingAddress);

    public record OrderItemRequest(int ProductId, int Quantity);

    public record Response(
        int OrderId,
        string Status,
        decimal TotalAmount,
        DateTime CreatedDate);

    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(x => x.CustomerId).GreaterThan(0);
            RuleFor(x => x.Items).NotEmpty()
                .WithMessage("Order must have at least one item.");
            RuleForEach(x => x.Items).ChildRules(item =>
            {
                item.RuleFor(i => i.ProductId).GreaterThan(0);
                item.RuleFor(i => i.Quantity).GreaterThan(0);
            });
            RuleFor(x => x.ShippingAddress).NotEmpty().MaximumLength(500);
        }
    }

    public static async Task<IResult> HandleAsync(
        Command command,
        OrderHubDbContext db,
        IValidator<Command> validator,
        CancellationToken ct)
    {
        var validation = await validator.ValidateAsync(command, ct);
        if (!validation.IsValid)
            return Results.ValidationProblem(validation.ToDictionary());

        var customer = await db.Customers.FindAsync([command.CustomerId], ct);
        if (customer is null)
            return Results.NotFound(new { Message = "Customer not found." });

        var order = new Order
        {
            CustomerId = command.CustomerId,
            ShippingAddress = command.ShippingAddress,
            CreatedDate = DateTime.UtcNow,
            Status = OrderStatus.Pending
        };

        foreach (var item in command.Items)
        {
            order.Items.Add(new OrderItem
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity
            });
        }

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        var response = new Response(
            order.OrderId,
            order.Status.ToString(),
            order.TotalAmount,
            order.CreatedDate);

        return Results.Created($"/api/v1/orders/{order.OrderId}", response);
    }

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/v1/orders", HandleAsync)
            .WithName("CreateOrder")
            .WithTags("Orders")
            .Produces<Response>(StatusCodes.Status201Created)
            .ProducesValidationProblem()
            .Produces(StatusCodes.Status404NotFound)
            .RequireAuthorization();
    }
}
```

### Reference slice 2: Read operation (Query)

```csharp
// File: Features/Orders/GetOrder/GetOrder.cs
using Microsoft.EntityFrameworkCore;
using OrderHub.Infrastructure.Persistence;

namespace OrderHub.API.Features.Orders.GetOrder;

public static class GetOrder
{
    public record Response(
        int OrderId,
        int CustomerId,
        string Status,
        decimal TotalAmount,
        DateTime CreatedDate,
        List<OrderItemResponse> Items);

    public record OrderItemResponse(
        int ProductId,
        string ProductName,
        int Quantity,
        decimal Price);

    public static async Task<IResult> HandleAsync(
        int orderId,
        OrderHubDbContext db,
        CancellationToken ct)
    {
        var order = await db.Orders
            .AsNoTracking()
            .Where(o => o.OrderId == orderId)
            .Select(o => new Response(
                o.OrderId,
                o.CustomerId,
                o.Status.ToString(),
                o.TotalAmount,
                o.CreatedDate,
                o.Items.Select(i => new OrderItemResponse(
                    i.ProductId,
                    i.Product.Name,
                    i.Quantity,
                    i.Price)).ToList()))
            .FirstOrDefaultAsync(ct);

        return order is null
            ? Results.NotFound()
            : Results.Ok(order);
    }

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/v1/orders/{orderId:int}", HandleAsync)
            .WithName("GetOrder")
            .WithTags("Orders")
            .Produces<Response>(StatusCodes.Status200OK)
            .Produces(StatusCodes.Status404NotFound)
            .RequireAuthorization();
    }
}
```

Notice the read slice is much shorter — no `Command`, no `Validator`. The query takes the route parameter directly, projects straight to `Response`, returns. This is intentional: simple operations stay simple.

### Reference slice 3: List with pagination (Query)

```csharp
// File: Features/Orders/ListOrders/ListOrders.cs
using Microsoft.EntityFrameworkCore;
using OrderHub.API.Common.Models;
using OrderHub.Infrastructure.Persistence;

namespace OrderHub.API.Features.Orders.ListOrders;

public static class ListOrders
{
    public record Query(
        int Page = 1,
        int PageSize = 20,
        string? Status = null,
        int? CustomerId = null,
        string SortBy = "createdDate",
        string SortOrder = "desc");

    public record OrderSummary(
        int OrderId,
        int CustomerId,
        string Status,
        decimal TotalAmount,
        DateTime CreatedDate);

    public static async Task<IResult> HandleAsync(
        [AsParameters] Query query,
        OrderHubDbContext db,
        CancellationToken ct)
    {
        var ordersQuery = db.Orders.AsNoTracking().AsQueryable();

        if (!string.IsNullOrEmpty(query.Status))
            ordersQuery = ordersQuery.Where(o => o.Status.ToString() == query.Status);

        if (query.CustomerId.HasValue)
            ordersQuery = ordersQuery.Where(o => o.CustomerId == query.CustomerId.Value);

        var totalItems = await ordersQuery.CountAsync(ct);

        var orders = await ordersQuery
            .OrderByDescending(o => o.CreatedDate)
            .Skip((query.Page - 1) * query.PageSize)
            .Take(query.PageSize)
            .Select(o => new OrderSummary(
                o.OrderId,
                o.CustomerId,
                o.Status.ToString(),
                o.TotalAmount,
                o.CreatedDate))
            .ToListAsync(ct);

        var result = new PagedResult<OrderSummary>(
            Data: orders,
            Page: query.Page,
            PageSize: query.PageSize,
            TotalItems: totalItems);

        return Results.Ok(result);
    }

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/v1/orders", HandleAsync)
            .WithName("ListOrders")
            .WithTags("Orders")
            .Produces<PagedResult<OrderSummary>>(StatusCodes.Status200OK)
            .RequireAuthorization();
    }
}
```

The shared `PagedResult<T>` lives in `Common/Models/PagedResult.cs` since it's used across many list endpoints.

## Endpoint registration

Each slice's `MapEndpoint` method must be called once at startup. We use **explicit manual registration** — clear, debuggable, traceable.

```csharp
// File: Common/Endpoints/EndpointRegistration.cs
using OrderHub.API.Features.Orders.CreateOrder;
using OrderHub.API.Features.Orders.GetOrder;
using OrderHub.API.Features.Orders.ListOrders;
using OrderHub.API.Features.Orders.CancelOrder;
using OrderHub.API.Features.Users.RegisterUser;
// ... etc

namespace OrderHub.API.Common.Endpoints;

public static class EndpointRegistration
{
    public static WebApplication MapEndpoints(this WebApplication app)
    {
        // Orders
        CreateOrder.MapEndpoint(app);
        GetOrder.MapEndpoint(app);
        ListOrders.MapEndpoint(app);
        CancelOrder.MapEndpoint(app);

        // Users
        RegisterUser.MapEndpoint(app);
        // ...

        return app;
    }
}
```

```csharp
// File: Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddInfrastructure(builder.Configuration)
    .AddValidatorsFromAssembly(typeof(Program).Assembly)  // FluentValidation
    .AddEndpointsApiExplorer()
    .AddSwaggerGen()
    .AddAuthentication(/* ... */)
    .AddAuthorization();

var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseAuthentication();
app.UseAuthorization();

app.MapEndpoints();  // ← all slices wired up here

app.Run();
```

**Why explicit registration:**
- Easy to find where a slice is registered (just search for the slice name)
- Debugging route conflicts is straightforward
- No reflection magic, no source generators to learn
- Adding a slice = adding one line — minimal friction

**When to switch to auto-discovery:** If the registration list grows past ~50 entries and feels unwieldy. At that point, a small reflection helper or source generator can replace manual registration.

## Naming conventions

### Standard C# naming (Microsoft conventions)

| Element | Convention | Example |
|---|---|---|
| Namespace | PascalCase, file-scoped | `namespace OrderHub.API.Features.Orders.CreateOrder;` |
| Class | PascalCase | `Order`, `OrderHubDbContext` |
| Static class (slice container) | PascalCase, matches feature | `public static class CreateOrder` |
| Interface | `I` + PascalCase | `IEmailService`, `ICurrentUserService` |
| Record | PascalCase | `Command`, `Query`, `Response` |
| Method | PascalCase, `Async` suffix when async | `HandleAsync`, `MapEndpoint` |
| Property | PascalCase | `OrderId`, `TotalAmount` |
| Boolean property | `Is`/`Has`/`Can` prefix | `IsActive`, `HasShipped`, `CanCancel` |
| Private field | `_camelCase` | `_dbContext`, `_logger` |
| Constant | PascalCase | `MaxRetryAttempts`, `DefaultPageSize` |
| Local variable | camelCase | `orderItems`, `totalAmount` |
| Parameter | camelCase | `orderId`, `cancellationToken` |
| Generic type | `T` or `TName` | `T`, `TSource`, `TDestination` |
| File name | Matches the public type | `Order.cs`, `CreateOrder.cs` |

### File-scoped namespaces (always)

Use file-scoped namespaces (C# 10+) — less indentation, cleaner files:

```csharp
// ✅ Preferred
namespace OrderHub.API.Features.Orders.CreateOrder;

public static class CreateOrder { ... }
```

```csharp
// ❌ Avoid (block-scoped)
namespace OrderHub.API.Features.Orders.CreateOrder
{
    public static class CreateOrder { ... }
}
```

Enforce in `.editorconfig`:

```ini
csharp_style_namespace_declarations = file_scoped:warning
```

### Async naming

Methods returning `Task` or `ValueTask` end in `Async`:

```csharp
// ✅ Good
public static async Task<IResult> HandleAsync(...) { }

// ❌ Bad — caller can't tell it's async
public static async Task<IResult> Handle(...) { }
```

### Always accept `CancellationToken`

Async methods that do I/O accept a `CancellationToken` parameter (usually last). ASP.NET Core auto-binds it from request context:

```csharp
public static async Task<IResult> HandleAsync(
    Command command,
    OrderHubDbContext db,
    IValidator<Command> validator,
    CancellationToken ct)  // ← always include
{
    var validation = await validator.ValidateAsync(command, ct);
    // ...
}
```

### Domain entity naming

Entities are PascalCase and live in `OrderHub.Domain/Entities/`:

```csharp
namespace OrderHub.Domain.Entities;

public class Order
{
    public int OrderId { get; set; }
    public int CustomerId { get; set; }
    public DateTime CreatedDate { get; set; }
    public OrderStatus Status { get; set; }
    public decimal TotalAmount { get; set; }
    public List<OrderItem> Items { get; set; } = new();
}
```

For complex domain logic, prefer **rich domain models** with methods on the entity:

```csharp
public class Order
{
    public int OrderId { get; private set; }
    public OrderStatus Status { get; private set; }
    // ...

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Cannot cancel a shipped order.");
        Status = OrderStatus.Cancelled;
    }
}
```

For simple data-only entities, plain property bags are acceptable. Don't force richness when there's no business logic to put on the entity.

### Common shared types

Shared models used across multiple slices live in `Common/Models/`:

```csharp
// File: Common/Models/PagedResult.cs
namespace OrderHub.API.Common.Models;

public record PagedResult<T>(
    IReadOnlyList<T> Data,
    int Page,
    int PageSize,
    int TotalItems)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalItems / PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrevious => Page > 1;
}
```

## Validation conventions

Use **FluentValidation** for request validation. Each slice's validator lives inside the slice file.

```csharp
public class Validator : AbstractValidator<Command>
{
    public Validator()
    {
        RuleFor(x => x.CustomerId).GreaterThan(0);
        RuleFor(x => x.Items).NotEmpty();
        RuleFor(x => x.ShippingAddress).NotEmpty().MaximumLength(500);
    }
}
```

Register validators globally in `Program.cs`:

```csharp
builder.Services.AddValidatorsFromAssembly(typeof(Program).Assembly);
```

This auto-discovers every `AbstractValidator<T>` in the assembly. New slice with a validator? Just write it — registration is automatic.

## Dependency injection conventions

### Constructor / parameter injection

Minimal API endpoints inject dependencies via the handler method's parameters:

```csharp
public static async Task<IResult> HandleAsync(
    Command command,
    OrderHubDbContext db,                // ← injected
    IValidator<Command> validator,       // ← injected
    ICurrentUserService currentUser,     // ← injected
    CancellationToken ct)
{
    // ...
}
```

ASP.NET Core resolves these from the DI container automatically.

### Service lifetimes

| Lifetime | Use for |
|---|---|
| `Singleton` | Stateless services, expensive to create |
| `Scoped` | Most services participating in a single request (default) |
| `Transient` | Lightweight, stateless, short-lived |

**Default to `Scoped`** for application services and EF Core `DbContext`.

### Service registration

Group registrations into extension methods per project:

```csharp
// OrderHub.Infrastructure/DependencyInjection.cs
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<OrderHubDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("OrderHubDb")));

        services.AddScoped<IEmailService, SendGridEmailService>();
        services.AddScoped<IJwtTokenService, JwtTokenService>();
        // ...

        return services;
    }
}

// Program.cs
builder.Services.AddInfrastructure(builder.Configuration);
```

## Configuration conventions

### Strongly-typed configuration

Define a class per config section in `Configuration/`:

```csharp
// Configuration/JwtSettings.cs
public class JwtSettings
{
    public const string SectionName = "Jwt";

    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public string Secret { get; set; } = string.Empty;
    public int ExpirationMinutes { get; set; } = 60;
}
```

Bind in `Program.cs`:

```csharp
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));
```

### `appsettings.json` keys

Use **PascalCase** for config keys (matches C# property names):

```json
{
  "Jwt": {
    "Issuer": "https://api.orderhub.com",
    "Audience": "orderhub-api",
    "ExpirationMinutes": 60
  },
  "ConnectionStrings": {
    "OrderHubDb": "Server=...;Database=OrderHub;..."
  }
}
```

### Secrets

- **Never commit secrets** to any config file
- **Local development:** User Secrets (`dotnet user-secrets`)
- **Staging / production:** Azure Key Vault
- **CI/CD:** Pipeline variables / environment variables

## Logging and observability

OrderHub uses **Azure Application Insights** as the primary logging and observability platform, integrated through the built-in `ILogger<T>` interface. We do **not** use Serilog — the built-in logger plus App Insights covers everything we need with one less dependency to maintain.

### Why Application Insights

The Azure infrastructure already includes Application Insights (`prod-orderhub-appi`) and Log Analytics Workspace (`prod-orderhub-log`) — see `AZURE_CONVENTIONS.md`. With one line of setup, you get:

- **Automatic correlation** — every log entry tagged with the request that produced it
- **Distributed tracing** — see how a single request flows through HTTP → SQL → external APIs
- **Dependency tracking** — every SQL query, HTTP call, and Redis operation logged automatically
- **Exception grouping** — duplicate errors collapsed, with full stack traces and request context
- **Live Metrics Stream** — real-time view of incoming requests, failures, CPU/memory
- **Application Map** — auto-generated dependency diagram with failure rates
- **KQL queries** — answer production questions in seconds (see KQL examples below)

### Setup

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Wires up logging, metrics, and tracing — one line
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.EnableAdaptiveSampling = true;
});

// ... rest of the setup
```

The connection string comes from configuration:

```json
// appsettings.json
{
  "ApplicationInsights": {
    "ConnectionString": ""
  }
}
```

In production, store the connection string in **Azure Key Vault** (`prod-orderhub-kv`) and reference it via Key Vault references in App Service configuration. Never commit it to source control.

### Cost-tuning (REQUIRED for production)

**Application Insights bills by gigabytes of data ingested.** Default settings will blow through the free tier (5 GB/month) quickly on a moderately busy app. The configuration below keeps OrderHub within the free tier or close to it for early-stage traffic.

> ⚠️ **Skipping these settings is the #1 cause of unexpected Azure bills.** This is required setup, not optional optimization.

#### 1. Adaptive sampling

Sampling tells App Insights "for every N successful requests, only keep one — but keep ALL errors." You see the patterns without paying for every successful call. **Never sample exceptions or your own application logs.**

```csharp
builder.Services.Configure<TelemetryConfiguration>(config =>
{
    config.DefaultTelemetrySink.TelemetryProcessorChainBuilder
        .UseAdaptiveSampling(
            maxTelemetryItemsPerSecond: 2,
            excludedTypes: "Exception;Trace")  // Never sample these
        .Build();
});
```

This single setting typically cuts data volume by 60–80%.

#### 2. Filter out noise

Health checks, Swagger requests, and static asset fetches generate volume without value. Drop them entirely:

```csharp
// File: Common/Telemetry/NoiseFilterProcessor.cs
namespace OrderHub.API.Common.Telemetry;

public class NoiseFilterProcessor : ITelemetryProcessor
{
    private readonly ITelemetryProcessor _next;

    public NoiseFilterProcessor(ITelemetryProcessor next) => _next = next;

    public void Process(ITelemetry item)
    {
        if (item is RequestTelemetry request)
        {
            var path = request.Url?.AbsolutePath ?? string.Empty;

            if (path.StartsWith("/health", StringComparison.OrdinalIgnoreCase) ||
                path.StartsWith("/swagger", StringComparison.OrdinalIgnoreCase) ||
                path.StartsWith("/favicon", StringComparison.OrdinalIgnoreCase))
            {
                return;  // Drop it
            }
        }
        _next.Process(item);
    }
}
```

Register it:

```csharp
builder.Services.AddApplicationInsightsTelemetryProcessor<NoiseFilterProcessor>();
```

#### 3. Right-size log levels for production

```json
// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "OrderHub": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "ApplicationInsights": {
      "LogLevel": {
        "Default": "Information"
      }
    }
  }
}
```

This keeps OrderHub's own `Information` logs flowing while suppressing framework noise.

#### 4. Set a daily ingestion cap (safety net)

A daily cap prevents runaway costs from a logging bug. Configure on the Log Analytics Workspace:

**Azure Portal:** `prod-orderhub-log` → Usage and estimated costs → Daily cap → Set to **1 GB/day** initially.

You'll get an alert when approaching the cap. Better to lose some logs than to blow the budget on a Friday night.

#### 5. Use 30-day retention (the free default)

Don't extend retention beyond the default 30 days unless required for compliance. Longer retention costs per GB-month. For long-term archival, export to blob storage instead.

### Structured logging

Always use **structured logging** with named placeholders. The placeholders become **searchable properties** in App Insights — you can query "all orders created for customer 12345 today" in KQL.

```csharp
// ✅ Good — structured, searchable
logger.LogInformation("Order {OrderId} created for customer {CustomerId} with total {TotalAmount}",
    order.OrderId, order.CustomerId, order.TotalAmount);

// ❌ Bad — string interpolation defeats structured logging
logger.LogInformation($"Order {order.OrderId} created for customer {order.CustomerId}");
```

### Log identifiers, not payloads

Logging entire objects burns through your data quota fast and exposes sensitive fields. Log the IDs you'd actually search on.

```csharp
// ❌ Expensive — logs entire object as JSON
logger.LogInformation("Order created: {Order}", JsonSerializer.Serialize(order));

// ✅ Cheap — logs just the searchable identifiers
logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
    order.OrderId, order.CustomerId);
```

If you genuinely need the full object for debugging, log it at `Debug` level (which is filtered out in production by the log levels above).

### Injecting `ILogger<T>` in slices

Inject `ILogger<T>` as a method parameter on the handler. The `T` is the slice container — this tags every log line with the feature name automatically:

```csharp
public static async Task<IResult> HandleAsync(
    Command command,
    OrderHubDbContext db,
    IValidator<Command> validator,
    ILogger<CreateOrder> logger,    // ← T is the slice container
    CancellationToken ct)
{
    logger.LogInformation("Processing order creation for customer {CustomerId}",
        command.CustomerId);

    // ... handler logic ...

    logger.LogInformation("Order {OrderId} created successfully", order.OrderId);
    return Results.Created(...);
}
```

In App Insights, you can now filter logs by feature: `SourceContext == "OrderHub.API.Features.Orders.CreateOrder.CreateOrder"`.

### Log levels

| Level | Use for | Production? |
|---|---|---|
| `LogTrace` | Extremely verbose diagnostic data | ❌ No |
| `LogDebug` | Internal state for development | ❌ No |
| `LogInformation` | Normal application flow milestones | ✅ Yes (for OrderHub code) |
| `LogWarning` | Unexpected but recoverable conditions | ✅ Yes |
| `LogError` | Errors that need attention | ✅ Yes |
| `LogCritical` | Application-failure-level errors | ✅ Yes |

### Never log sensitive data

These should never appear in any log output:

- Passwords, tokens, API keys, secrets of any kind
- Full credit card numbers, SSNs, government IDs
- Full request/response bodies (may contain PII)
- PII (email, phone, address) unless required and properly redacted
- Connection strings

If a log message is constructed from user input, treat the input as untrusted — don't include it verbatim if it could contain sensitive data.

### Useful KQL queries

App Insights data is queryable via **KQL (Kusto Query Language)** in the Azure portal. Bookmark these queries — they're the operational toolkit your team will use most.

```kql
// Recent 5xx errors with full request context
requests
| where timestamp > ago(1h)
| where resultCode startswith "5"
| project timestamp, name, resultCode, duration, operation_Id
| order by timestamp desc
```

```kql
// Slow database queries (over 1 second)
dependencies
| where timestamp > ago(24h)
| where type == "SQL"
| where duration > 1000
| project timestamp, target, data, duration
| order by duration desc
| take 50
```

```kql
// Trace a specific request end-to-end (paste operation_Id from a request)
union requests, dependencies, exceptions, traces
| where operation_Id == "<paste-id-here>"
| order by timestamp asc
```

```kql
// Top 10 slowest endpoints by 95th percentile
requests
| where timestamp > ago(24h)
| summarize p95 = percentile(duration, 95), count = count() by name
| order by p95 desc
| take 10
```

```kql
// Exception rate by endpoint
exceptions
| where timestamp > ago(24h)
| join kind=inner (requests) on operation_Id
| summarize count() by name, type
| order by count_ desc
```

```kql
// Find all logs for a specific customer (using structured properties)
traces
| where timestamp > ago(24h)
| where customDimensions.CustomerId == "12345"
| project timestamp, message, severityLevel, customDimensions
| order by timestamp asc
```

### Setup checklist

When deploying OrderHub to a new environment, verify these are configured:

1. ☐ `AddApplicationInsightsTelemetry()` called in `Program.cs`
2. ☐ Connection string sourced from Key Vault (production) or User Secrets (development)
3. ☐ Adaptive sampling configured with `excludedTypes: "Exception;Trace"`
4. ☐ `NoiseFilterProcessor` registered to drop `/health` and `/swagger` traffic
5. ☐ Production `LogLevel` set to Warning for framework, Information for OrderHub
6. ☐ Daily ingestion cap set on Log Analytics Workspace (start at 1 GB/day)
7. ☐ Retention left at 30-day default
8. ☐ Cost alert configured at 50% of expected monthly budget

### Cost expectations

With the configuration above, rough monthly cost estimates for OrderHub:

| Stage | Approximate traffic | Monthly cost |
|---|---|---|
| Dev/staging only | Low | $0 (within free tier) |
| Early production | ~10k requests/day | $0–5 |
| Growing | ~100k requests/day | $5–20 |
| Established | ~1M requests/day | $30–80 |

Verify with the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) before committing to any tier — pricing changes and varies by region.

### Future direction: OpenTelemetry

Microsoft is gradually shifting .NET observability toward **OpenTelemetry** (vendor-neutral instrumentation). The `Azure.Monitor.OpenTelemetry.AspNetCore` package is the migration path. Both the `ApplicationInsights` SDK and OpenTelemetry feed the same Azure Monitor backend, so the migration when we're ready will be straightforward.

**For now:** stick with `AddApplicationInsightsTelemetry()`. Revisit OpenTelemetry in 1–2 years as the ecosystem matures.

## Testing conventions

### Test project structure

```
tests/
├── OrderHub.API.Tests/
│   ├── Features/
│   │   └── Orders/
│   │       ├── CreateOrderTests.cs
│   │       ├── GetOrderTests.cs
│   │       └── ListOrdersTests.cs
│   └── Common/
└── OrderHub.Domain.Tests/
    └── Entities/
        └── OrderTests.cs
```

Test folders mirror the feature folders. Easy to find tests for any slice.

### Co-located vs. separate test project

**Two valid options** — pick one and stick to it:

**Option A: Separate test project** (recommended)
- Tests in `tests/OrderHub.API.Tests/Features/Orders/CreateOrderTests.cs`
- Cleaner production assembly
- Test dependencies don't leak into production

**Option B: Co-located tests**
- Tests next to the slice: `Features/Orders/CreateOrder/CreateOrderTests.cs`
- Maximum locality — see slice and its tests in same folder
- Requires excluding test files from production builds

**Recommendation: Option A.** The locality benefit of co-location is real but doesn't outweigh the build/deployment concerns.

### Test class and method naming

Test class: `<Feature>Tests`
Test method: `<Scenario>_<ExpectedBehavior>` or `<Method>_<Scenario>_<Expected>`

```csharp
public class CreateOrderTests
{
    [Fact]
    public async Task ValidRequest_CreatesOrderAndReturnsCreated() { }

    [Fact]
    public async Task EmptyItems_ReturnsValidationError() { }

    [Fact]
    public async Task NonExistentCustomer_ReturnsNotFound() { }
}
```

### Arrange-Act-Assert structure

```csharp
[Fact]
public async Task ValidRequest_CreatesOrderAndReturnsCreated()
{
    // Arrange
    var db = CreateInMemoryDbContext();
    db.Customers.Add(new Customer { CustomerId = 1 });
    await db.SaveChangesAsync();

    var command = new CreateOrder.Command(
        CustomerId: 1,
        Items: [new(ProductId: 100, Quantity: 2)],
        ShippingAddress: "123 Main St");

    var validator = new CreateOrder.Validator();

    // Act
    var result = await CreateOrder.HandleAsync(command, db, validator, default);

    // Assert
    result.Should().BeOfType<Created<CreateOrder.Response>>();
}
```

Use **FluentAssertions** for readable assertions. Mock external dependencies via Moq or NSubstitute. For database tests, prefer EF Core's in-memory provider for unit tests; use TestContainers for integration tests.

## Error handling

### RFC 7807 Problem Details

ASP.NET Core supports Problem Details natively. Use `Results.Problem(...)`, `Results.NotFound()`, `Results.ValidationProblem(...)` from minimal APIs:

```csharp
return Results.Problem(
    title: "Order not found",
    detail: $"No order exists with ID {orderId}.",
    statusCode: StatusCodes.Status404NotFound);
```

### Global exception handler

Configure global exception handling in `Program.cs` to convert unhandled exceptions to Problem Details responses:

```csharp
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var problem = exception switch
        {
            DomainException de => Results.Problem(de.Message, statusCode: 400),
            _ => Results.Problem("An unexpected error occurred.", statusCode: 500)
        };
        await problem.ExecuteAsync(context);
    });
});
```

## Code style enforcement

### `.editorconfig`

Place at repo root to enforce style across the team and IDEs:

```ini
root = true

[*.{cs,csx}]
indent_style = space
indent_size = 4
charset = utf-8
end_of_line = crlf
insert_final_newline = true
trim_trailing_whitespace = true

# Namespace
csharp_style_namespace_declarations = file_scoped:warning

# var preferences
csharp_style_var_for_built_in_types = true:suggestion
csharp_style_var_when_type_is_apparent = true:suggestion
csharp_style_var_elsewhere = true:suggestion

# Expression-bodied members
csharp_style_expression_bodied_methods = when_on_single_line:suggestion
csharp_style_expression_bodied_properties = true:suggestion

# Naming - private fields with underscore
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.severity = warning
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.style = camel_case_underscore_style

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.camel_case_underscore_style.required_prefix = _
dotnet_naming_style.camel_case_underscore_style.capitalization = camel_case
```

### `Directory.Build.props`

Apply common MSBuild properties to all projects:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

`TreatWarningsAsErrors=true` and nullable reference types help prevent accumulation of issues. Set these from day one — adding them later is much harder.

## Slice creation checklist

When creating a new slice, follow this checklist:

1. ☐ Create folder: `Features/<Domain>/<FeatureName>/`
2. ☐ Create file: `<FeatureName>.cs`
3. ☐ Set namespace: `OrderHub.API.Features.<Domain>.<FeatureName>`
4. ☐ Define `public static class <FeatureName>`
5. ☐ Add `Command` (write) or `Query` (read) record
6. ☐ Add `Response` record
7. ☐ Add `Validator` class (if needed)
8. ☐ Implement `HandleAsync` method
9. ☐ Implement `MapEndpoint` method with full OpenAPI metadata
10. ☐ Register endpoint in `Common/Endpoints/EndpointRegistration.cs`
11. ☐ Write tests in `tests/OrderHub.API.Tests/Features/<Domain>/`
12. ☐ Verify Swagger doc renders the endpoint correctly

## Quick reference cheat sheet

| Element | Convention | Example |
|---|---|---|
| Solution | `OrderHub.API.sln` | One solution at repo root |
| Project | `OrderHub.<Name>` | `OrderHub.API`, `OrderHub.Domain`, `OrderHub.Infrastructure` |
| Repo | lowercase-kebab | `orderhub-api` |
| Slice folder | `Features/<Domain>/<FeatureName>/` | `Features/Orders/CreateOrder/` |
| Slice file | `<FeatureName>.cs` | `CreateOrder.cs` |
| Slice container | `public static class <FeatureName>` | `public static class CreateOrder` |
| Write request | `public record Command(...)` | Inside slice container |
| Read request | `public record Query(...)` | Inside slice container |
| Response | `public record Response(...)` | Inside slice container |
| Validator | `public class Validator : AbstractValidator<Command>` | Inside slice container |
| Handler | `public static async Task<IResult> HandleAsync(...)` | Inside slice container |
| Endpoint | `public static void MapEndpoint(IEndpointRouteBuilder app)` | Inside slice container |
| Namespace | `OrderHub.API.Features.<Domain>.<FeatureName>` | File-scoped |
| Test class | `<FeatureName>Tests` | `CreateOrderTests` |
| Test method | `<Scenario>_<Expected>` | `EmptyItems_ReturnsValidationError` |

---

**Last updated:** 2026-04-30
**Owner:** OrderHub Engineering
**Related:**
- [`AZURE_CONVENTIONS.md`](./AZURE_CONVENTIONS.md) — Azure resource naming
- [`DATABASE_CONVENTIONS.md`](./DATABASE_CONVENTIONS.md) — Database object naming
- [`API_CONVENTIONS.md`](./API_CONVENTIONS.md) — REST API conventions
