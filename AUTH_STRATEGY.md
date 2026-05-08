# OrderHub — REST API Naming Conventions

This document defines REST API conventions for the OrderHub backend (`orderhub-api`, ASP.NET Core 10). It covers route structure, HTTP verbs, status codes, request/response shapes, versioning, and error handling.

## Stack assumptions

- **Backend:** ASP.NET Core 10 Web API (`OrderHub.API`)
- **Routing:** Attribute routing (`[Route]`, `[HttpGet]`, etc.)
- **Serialization:** `System.Text.Json` (default in ASP.NET Core)
- **Documentation:** OpenAPI / Swagger via Swashbuckle or NSwag
- **Auth:** JWT Bearer tokens (Azure AD / Entra ID assumed)

## Core principles

1. **Resources, not actions.** URLs identify *things*; HTTP verbs describe *actions on them*. `POST /orders` to create an order, not `POST /createOrder`.
2. **Consistency over cleverness.** A predictable API is easier to use than a clever one.
3. **kebab-case in URLs, camelCase in JSON, PascalCase in C#.** Each layer follows its own ecosystem's convention.
4. **Stable contracts.** Once published, an endpoint shape doesn't change without a version bump.

## Naming style glossary

Different layers of OrderHub use different naming styles. This isn't inconsistency — each ecosystem has its own convention, and following each one makes the code feel native in its context. Here's a reference for the four styles you'll encounter:

| Style | Format | Example |
|---|---|---|
| **kebab-case** | lowercase, hyphens between words | `order-items`, `payment-methods` |
| **camelCase** | lowercase first letter, capital for each subsequent word | `orderItems`, `firstName` |
| **PascalCase** | capital first letter, capital for each subsequent word | `OrderItems`, `FirstName` |
| **snake_case** | lowercase, underscores between words | `order_items`, `first_name` |
| **SCREAMING_SNAKE_CASE** | uppercase, underscores between words | `ORDER_ITEMS`, `API_KEY` |

### Where each style is used in OrderHub

| Context | Style | Example |
|---|---|---|
| GitHub repo names | kebab-case | `orderhub-web`, `orderhub-api` |
| Azure resource names (most) | kebab-case | `prod-orderhub-app-api` |
| REST API URL paths | kebab-case | `/api/v1/order-items` |
| HTTP query parameters | camelCase | `?pageSize=20&sortBy=createdDate` |
| JSON request/response fields | camelCase | `{ "firstName": "Alex" }` |
| C# class names | PascalCase | `OrderController`, `OrderDto` |
| C# property names | PascalCase | `public string FirstName { get; set; }` |
| SQL table & column names | PascalCase | `Orders`, `FirstName` |
| Database name | PascalCase | `OrderHub` |
| Environment variables | SCREAMING_SNAKE_CASE | `DATABASE_CONNECTION_STRING` |
| Config keys (`appsettings.json`) | PascalCase | `"ConnectionStrings:OrderHubDb"` |
| Next.js component files | PascalCase | `OrderList.tsx`, `UserProfile.tsx` |
| Next.js route folders | kebab-case | `/order-items/page.tsx` |
| CSS class names | kebab-case | `.order-item-row`, `.btn-primary` |

### One concept, five forms

The same logical concept ("order items") shows up in different forms throughout the stack:

```
GitHub repo:     orderhub-api               ← kebab-case
URL path:        /api/v1/order-items        ← kebab-case
Query param:     ?orderItemId=123           ← camelCase
JSON field:      "orderItems"               ← camelCase
C# class:        OrderItem, OrderItemDto    ← PascalCase
SQL table:       OrderItems                 ← PascalCase
SQL column:      OrderItemId                ← PascalCase
Env variable:    ORDER_ITEM_API_KEY         ← SCREAMING_SNAKE_CASE
```

ASP.NET Core's `System.Text.Json` automatically converts between PascalCase (C#) and camelCase (JSON), so you write C# normally and the framework handles the casing translation at the API boundary.

### Why these specific choices?

- **kebab-case in URLs** — Hyphens are URL-friendly, SEO-recommended (Google prefers hyphens to underscores), and avoid case-sensitivity issues across servers
- **camelCase in JSON** — JavaScript's native variable convention; consumers (especially the Next.js frontend) get fields that match their language idioms
- **PascalCase in C# and SQL Server** — Microsoft ecosystem convention going back decades; matches `.cs` files, namespaces, and SQL Server tooling like SSMS
- **SCREAMING_SNAKE_CASE for env variables** — POSIX convention; visually distinct from code identifiers

## URL structure

### Pattern

```
https://<host>/api/v<version>/<resource>[/<id>][/<sub-resource>][/<id>]
```

Examples:
```
GET    /api/v1/orders                          # List orders
GET    /api/v1/orders/12345                    # Get one order
POST   /api/v1/orders                          # Create order
PUT    /api/v1/orders/12345                    # Replace order
PATCH  /api/v1/orders/12345                    # Partial update
DELETE /api/v1/orders/12345                    # Delete order
GET    /api/v1/orders/12345/items              # List items in an order
GET    /api/v1/users/12345/orders              # List orders for a user
```

### Casing rules

| Layer | Convention | Example |
|---|---|---|
| URL path | **kebab-case** | `/api/v1/order-items`, `/api/v1/payment-methods` |
| Query parameters | **camelCase** | `?pageSize=20&sortBy=createdDate` |
| JSON request/response | **camelCase** | `{ "firstName": "Alex", "createdDate": "..." }` |
| C# class names | **PascalCase** | `OrderController`, `OrderDto` |
| C# property names | **PascalCase** | `public string FirstName { get; set; }` |
| Route attribute | **kebab-case** | `[Route("api/v1/order-items")]` |

ASP.NET Core handles the C# PascalCase ↔ JSON camelCase conversion automatically via `System.Text.Json` defaults — no manual configuration needed.

### Resource names

- **Plural, kebab-case nouns** for collection resources: `/orders`, `/users`, `/order-items`, `/payment-methods`
- **Singular for singletons** (resources where there's only one per parent): `/users/12345/profile`, `/orders/12345/shipping-address`
- **Avoid verbs in URLs** — verbs belong in the HTTP method, not the path

### Good vs. bad examples

| ❌ Bad | ✅ Good | Why |
|---|---|---|
| `POST /api/v1/createOrder` | `POST /api/v1/orders` | Verb belongs in HTTP method |
| `GET /api/v1/getOrders` | `GET /api/v1/orders` | Same — `GET` already means "retrieve" |
| `GET /api/v1/Orders` | `GET /api/v1/orders` | URLs are case-sensitive; lowercase is standard |
| `GET /api/v1/order_items` | `GET /api/v1/order-items` | Hyphens, not underscores, in URL paths |
| `GET /api/v1/orderItems` | `GET /api/v1/order-items` | camelCase in URLs is unusual; prefer kebab-case |
| `GET /api/v1/order` | `GET /api/v1/orders` | Resource collections are plural |
| `DELETE /api/v1/orders/delete/12345` | `DELETE /api/v1/orders/12345` | `DELETE` verb already implies the action |

## HTTP verbs

| Verb | Purpose | Idempotent? | Safe? | Body? |
|---|---|---|---|---|
| `GET` | Retrieve resource(s) | ✅ Yes | ✅ Yes | ❌ No |
| `POST` | Create new resource | ❌ No | ❌ No | ✅ Yes |
| `PUT` | Replace entire resource | ✅ Yes | ❌ No | ✅ Yes |
| `PATCH` | Partial update | ❌ No | ❌ No | ✅ Yes |
| `DELETE` | Remove resource | ✅ Yes | ❌ No | ❌ Usually no |

### When to use PUT vs. PATCH

- **`PUT`** — Client sends the **complete** resource. Server replaces it entirely. If a field is omitted, it's set to default/null.
- **`PATCH`** — Client sends **only the fields to change**. Server merges them with the existing resource. Omitted fields are left untouched.

For most CRUD operations, **`PATCH` is the better default** — `PUT` requires the client to fetch the resource first to avoid wiping fields they didn't intend to change.

### Bulk operations

For operations on multiple resources at once:

```
POST   /api/v1/orders/bulk           # Create multiple orders
PATCH  /api/v1/orders/bulk           # Update multiple orders
DELETE /api/v1/orders/bulk           # Delete multiple orders (with body listing IDs)
```

Bulk endpoints accept and return arrays. Always paginate response sizes.

### Actions that don't fit CRUD

Some operations don't map cleanly to standard verbs (e.g., "approve", "cancel", "send"). Use a **sub-resource representing the action's outcome**:

```
POST /api/v1/orders/12345/cancellation         # Cancel an order
POST /api/v1/orders/12345/shipment             # Ship an order
POST /api/v1/users/12345/password-reset        # Request password reset
POST /api/v1/invoices/12345/payment            # Pay an invoice
```

The pattern: `POST /resource/{id}/<noun-form-of-action>`. The HTTP verb is `POST` because the action creates a new state (a cancellation, a shipment, a payment).

Avoid the alternative `POST /orders/12345/cancel` (verb in URL) — it works, but mixing verbs into URLs makes the API less consistent.

## Status codes

Use the right status code. The body explains *what*; the status explains *what kind*.

### Success codes

| Code | Meaning | When to use |
|---|---|---|
| **200 OK** | Success with response body | `GET`, `PUT`, `PATCH` returning the updated resource |
| **201 Created** | Resource created | `POST` that creates a new resource. Include `Location` header pointing to the new resource |
| **202 Accepted** | Request accepted, processing async | Long-running operations, queued jobs |
| **204 No Content** | Success with no response body | `DELETE`, or updates where the response body adds no value |

### Client error codes (4xx)

| Code | Meaning | When to use |
|---|---|---|
| **400 Bad Request** | Malformed request, validation failed | Invalid JSON, missing required fields, business rule violation |
| **401 Unauthorized** | Authentication missing or invalid | No token, expired token, invalid token |
| **403 Forbidden** | Authenticated but not allowed | Valid token, but user lacks permission for this resource |
| **404 Not Found** | Resource doesn't exist | `GET /orders/99999` where 99999 doesn't exist |
| **405 Method Not Allowed** | Endpoint exists, verb doesn't | `DELETE /orders` (collection) when only `DELETE /orders/{id}` is supported |
| **409 Conflict** | Request conflicts with current state | Duplicate creation, optimistic concurrency failure |
| **422 Unprocessable Entity** | Syntactically valid but semantically wrong | Often used for validation errors (some teams prefer `400`) |
| **429 Too Many Requests** | Rate limit exceeded | Include `Retry-After` header |

### Server error codes (5xx)

| Code | Meaning |
|---|---|
| **500 Internal Server Error** | Unhandled exception |
| **502 Bad Gateway** | Upstream service failed |
| **503 Service Unavailable** | Server temporarily down (maintenance, overload) |
| **504 Gateway Timeout** | Upstream service timed out |

### 401 vs. 403 — common confusion

- **401** = "I don't know who you are" (no/invalid credentials)
- **403** = "I know who you are, but you can't do this" (authenticated but unauthorized)

## Request and response shapes

### JSON casing

Always **camelCase** in JSON bodies — both requests and responses. ASP.NET Core's `System.Text.Json` does this by default.

```json
{
  "userId": 12345,
  "firstName": "Alex",
  "emailAddress": "alex@example.com",
  "createdDate": "2026-04-30T14:32:00Z",
  "isActive": true
}
```

### Date/time format

- Always **ISO 8601 with explicit timezone**: `"2026-04-30T14:32:00Z"` (UTC) or `"2026-04-30T14:32:00-07:00"`
- Prefer **UTC (`Z` suffix)** for storage and APIs; let clients convert to local time
- For date-only fields: `"2026-04-30"` (no time component)

### Money / decimal values

Send as **strings** in JSON to avoid floating-point precision issues:

```json
{
  "totalAmount": "1234.56",
  "currency": "USD"
}
```

Use C# `decimal` on the server side. Configure `System.Text.Json` to serialize decimals as strings if needed.

### IDs

- Use the same casing/type as the database: `userId`, `orderId` — `int` or `long` for sequential IDs, `string` (Guid) for UUIDs
- **Never expose database internal IDs** if the resource has a public-facing ID (e.g., order number `ORD-2026-001234`)

### Pagination

For any list endpoint that could return many items:

**Request (query parameters):**
```
GET /api/v1/orders?page=1&pageSize=20&sortBy=createdDate&sortOrder=desc
```

**Response shape:**
```json
{
  "data": [
    { "orderId": 1, "totalAmount": "100.00", ... },
    { "orderId": 2, "totalAmount": "250.00", ... }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": false
  }
}
```

Standard query parameters:

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `page` | int | 1 | 1-indexed |
| `pageSize` | int | 20 | Cap at 100 to prevent abuse |
| `sortBy` | string | (resource-specific) | Field name in camelCase |
| `sortOrder` | string | `asc` | `asc` or `desc` |

### Filtering

For simple filters, use query parameters:

```
GET /api/v1/orders?status=pending&customerId=12345
GET /api/v1/users?isActive=true&createdAfter=2026-01-01
```

For complex queries (multiple operators, ranges), consider a `POST /search` endpoint with a structured body:

```http
POST /api/v1/orders/search
Content-Type: application/json

{
  "filters": {
    "status": ["pending", "processing"],
    "totalAmount": { "min": "100.00", "max": "500.00" },
    "createdDate": { "from": "2026-01-01", "to": "2026-04-30" }
  },
  "sortBy": "createdDate",
  "sortOrder": "desc",
  "page": 1,
  "pageSize": 20
}
```

`POST` is acceptable here despite being a "read" operation — the request body would be too long for a query string.

### Response envelope: yes or no?

Two valid approaches:

**Option A: Bare resources (preferred for most cases)**
```json
{ "orderId": 12345, "totalAmount": "100.00", ... }
```

**Option B: Wrapped in envelope**
```json
{
  "data": { "orderId": 12345, "totalAmount": "100.00", ... },
  "metadata": { ... }
}
```

**Recommendation:** Use **bare resources for single-resource endpoints** (`GET /orders/12345`) and **envelope only for paginated lists** (which need the pagination metadata anyway). This keeps responses clean while still supporting metadata where it matters.

## Error response shape

All error responses use a consistent shape based on **RFC 7807 (Problem Details for HTTP APIs)** — ASP.NET Core supports this natively.

```json
{
  "type": "https://api.orderhub.com/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more fields failed validation.",
  "instance": "/api/v1/orders",
  "traceId": "00-abc123-def456-00",
  "errors": {
    "emailAddress": ["Email address is required."],
    "totalAmount": ["Total amount must be greater than zero."]
  }
}
```

Standard fields:

| Field | Required | Purpose |
|---|---|---|
| `type` | recommended | URI identifying the error type |
| `title` | ✅ yes | Short, human-readable summary |
| `status` | ✅ yes | HTTP status code (matches response code) |
| `detail` | recommended | Human-readable explanation of *this* occurrence |
| `instance` | optional | URI of the specific request |
| `traceId` | recommended | Correlation ID for log lookup |
| `errors` | for validation | Field-level errors (validation only) |

### ASP.NET Core implementation

Use the built-in `ProblemDetails` and `ValidationProblemDetails`:

```csharp
[HttpGet("{orderId}")]
public ActionResult<OrderDto> GetOrder(int orderId)
{
    var order = _orderService.GetOrder(orderId);
    if (order == null)
    {
        return Problem(
            title: "Order not found",
            detail: $"No order exists with ID {orderId}.",
            statusCode: StatusCodes.Status404NotFound);
    }
    return Ok(order);
}
```

Configure global exception handling in `Program.cs` to convert all unhandled exceptions to Problem Details responses.

## Versioning

### Strategy: URL path versioning

OrderHub uses **URL path versioning** (`/api/v1/...`) rather than header-based versioning.

**Why:**
- Visible and obvious in logs, browser bars, and Postman collections
- Easy to route to different controllers in ASP.NET Core
- Standard convention most consumers expect
- Caches work correctly (different URLs = different cache entries)

### Why include `v1` from day one

OrderHub plans to expose APIs to external customers. Once external consumers exist, versioning shifts from "nice to have" to "necessary":

- Customers build integrations on their timeline, not yours
- You can't force them to upgrade when something changes
- A breaking change without versioning means broken customer integrations
- Adding versioning **after** customers exist requires a coordinated migration

The cost of including `v1` in URLs from day one is essentially zero. The cost of retrofitting it later is high. So we include it.

### One API surface for everyone

OrderHub uses **a single `/api/v1/` surface for all consumers** — both `orderhub-web` (our frontend) and external customers hit the same endpoints. We deliberately do **not** split URLs by audience (no `/api/internal/`, no `/api/customer/`, etc.).

**Why one surface:**
- Most endpoints are useful to both audiences (fetching an order, updating a profile, etc.) — splitting would mean maintaining duplicate endpoints
- Audience-based access is an **authorization** concern, not a URL concern — solved with `[Authorize]` attributes and role-based policies
- Frontend-specific flexibility (extra fields, additional query parameters) can be handled within a single endpoint via auth-aware response shaping or optional `?include=` parameters
- One Swagger doc, one mental model, one place for every endpoint

**When admin-only endpoints are eventually needed**, group them under `/api/v1/admin/...` and protect them with `[Authorize(Roles = "Admin")]`. The role-based protection — not the URL — is what keeps customers out. There's no need to build this structure preemptively; add it when you actually have admin functionality.

### Version numbering

- Use **integer major versions** in the URL: `v1`, `v2`, `v3`
- Don't expose minor versions in URLs (`v1.2`) — handle minor changes in a backward-compatible way within the same major version
- A new major version is needed only for **breaking changes**

### What constitutes a breaking change

- Removing or renaming a field
- Changing a field's type or format
- Changing the meaning of a field
- Removing an endpoint
- Changing required fields
- Changing authentication requirements

### What does NOT require a new version

- Adding new optional fields to responses
- Adding new endpoints
- Adding new optional query parameters
- Adding new error types (within existing status codes)
- Performance improvements

### Deprecation

When deprecating an endpoint:

1. Add the `Deprecation` HTTP header on responses: `Deprecation: true`
2. Add a `Sunset` header indicating retirement date: `Sunset: Wed, 31 Dec 2026 23:59:59 GMT`
3. Add a `Link` header pointing to the replacement: `Link: </api/v2/orders>; rel="successor-version"`
4. Document the deprecation in the OpenAPI spec
5. Email known consumers with a timeline (minimum 6 months)

## C# controller conventions

### File naming

- One controller per resource: `OrdersController.cs`, `UsersController.cs`
- File location: `OrderHub.API/Controllers/V1/OrdersController.cs` (versioned subfolder)

### Class structure

```csharp
namespace OrderHub.API.Controllers.V1;

[ApiController]
[Route("api/v1/orders")]
[Authorize]
[Produces("application/json")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService orderService, ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [HttpGet]
    [ProducesResponseType(typeof(PagedResult<OrderDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResult<OrderDto>>> GetOrders(
        [FromQuery] OrderQueryParameters parameters,
        CancellationToken cancellationToken)
    {
        var result = await _orderService.GetOrdersAsync(parameters, cancellationToken);
        return Ok(result);
    }

    [HttpGet("{orderId:int}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> GetOrder(
        int orderId,
        CancellationToken cancellationToken)
    {
        var order = await _orderService.GetOrderAsync(orderId, cancellationToken);
        return order is null ? NotFound() : Ok(order);
    }

    [HttpPost]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ValidationProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<OrderDto>> CreateOrder(
        [FromBody] CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        var order = await _orderService.CreateOrderAsync(request, cancellationToken);
        return CreatedAtAction(
            nameof(GetOrder),
            new { orderId = order.OrderId },
            order);
    }
}
```

### Controller naming

- **Plural noun + `Controller`**: `OrdersController`, `UsersController`, `OrderItemsController`
- Match the URL resource: `OrdersController` → `/api/v1/orders`

### Action method naming

- Match the operation: `GetOrders`, `GetOrder`, `CreateOrder`, `UpdateOrder`, `DeleteOrder`
- For sub-resources: `GetOrderItems`, `AddOrderItem`, `RemoveOrderItem`
- For action endpoints: `CancelOrder`, `ShipOrder`, `ResetPassword`

### Always include `CancellationToken`

Every async action method should accept a `CancellationToken` parameter. ASP.NET Core auto-binds it from the request context. This allows long-running queries to be canceled when the client disconnects.

## DTO conventions

DTOs (Data Transfer Objects) are the contract between API and clients. **Never expose EF Core entities directly** — they leak database structure and create tight coupling.

### Naming

| Type | Suffix | Example |
|---|---|---|
| Response DTO | `Dto` | `OrderDto`, `UserDto` |
| Request body for create | `Request` | `CreateOrderRequest` |
| Request body for update | `Request` | `UpdateOrderRequest`, `UpdateUserProfileRequest` |
| Query parameter object | `Parameters` | `OrderQueryParameters`, `UserSearchParameters` |
| Paged response | `PagedResult<T>` | `PagedResult<OrderDto>` |

### Location

```
OrderHub.API/
├── Controllers/
│   └── V1/
│       └── OrdersController.cs
└── Models/
    └── V1/
        ├── OrderDto.cs
        ├── CreateOrderRequest.cs
        ├── UpdateOrderRequest.cs
        └── OrderQueryParameters.cs
```

(Final folder structure will be defined in `CSHARP_CONVENTIONS.md`.)

### Validation

Use Data Annotations or FluentValidation:

```csharp
public class CreateOrderRequest
{
    [Required]
    public int CustomerId { get; set; }

    [Required]
    [MinLength(1)]
    public List<CreateOrderItemRequest> Items { get; set; }

    [Range(0, double.MaxValue)]
    public decimal ShippingAmount { get; set; }
}
```

Validation failures automatically produce `ValidationProblemDetails` (HTTP 400) when `[ApiController]` is applied to the controller.

## Authentication and authorization

### Header convention

```
Authorization: Bearer <jwt-token>
```

### Endpoint protection

- Default: **all endpoints require authentication** (`[Authorize]` at controller or globally)
- Public endpoints: explicitly mark with `[AllowAnonymous]`
- Role-based: `[Authorize(Roles = "Admin")]`
- Policy-based (preferred for complex rules): `[Authorize(Policy = "CanManageOrders")]`

### CORS

Define allowed origins per environment in configuration. Don't use `*` in production.

```csharp
// Development
"Cors": {
  "AllowedOrigins": ["http://localhost:3000"]
}

// Production
"Cors": {
  "AllowedOrigins": ["https://app.orderhub.com"]
}
```

## Health and metadata endpoints

Standard non-versioned endpoints (outside `/api/v1`):

| Endpoint | Purpose | Auth |
|---|---|---|
| `GET /health` | Liveness probe (is the app running?) | None |
| `GET /health/ready` | Readiness probe (can it serve traffic? checks DB, dependencies) | None |
| `GET /version` | Build version and git commit | None |
| `GET /swagger` | OpenAPI UI | Disabled in prod or auth-protected |

These are platform/operational endpoints, not part of the versioned API surface.

## Examples: complete endpoint patterns

### Standard CRUD on a resource

```
GET    /api/v1/orders                    # List with pagination
GET    /api/v1/orders/{orderId}          # Get one
POST   /api/v1/orders                    # Create
PATCH  /api/v1/orders/{orderId}          # Partial update
DELETE /api/v1/orders/{orderId}          # Delete
```

### Nested resources

```
GET    /api/v1/orders/{orderId}/items                   # List items in order
POST   /api/v1/orders/{orderId}/items                   # Add item to order
GET    /api/v1/orders/{orderId}/items/{itemId}          # Get specific item
PATCH  /api/v1/orders/{orderId}/items/{itemId}          # Update item
DELETE /api/v1/orders/{orderId}/items/{itemId}          # Remove item
```

### Action endpoints

```
POST   /api/v1/orders/{orderId}/cancellation     # Cancel order
POST   /api/v1/orders/{orderId}/shipment         # Ship order
POST   /api/v1/users/{userId}/password-reset     # Initiate password reset
POST   /api/v1/users/{userId}/email-verification # Send verification email
```

### Search

```
GET    /api/v1/orders?status=pending&customerId=12345    # Simple filters
POST   /api/v1/orders/search                             # Complex query body
```

## OpenAPI / Swagger documentation

- Every endpoint must have XML doc comments on the action method (auto-extracted into Swagger)
- Every DTO field must have a description (XML comment or `[Description]` attribute)
- Use `[ProducesResponseType]` to declare all possible response types and status codes
- Group related endpoints with consistent tags (auto-derived from controller name)

```csharp
/// <summary>
/// Retrieves a single order by its identifier.
/// </summary>
/// <param name="orderId">The unique identifier of the order.</param>
/// <returns>The order details.</returns>
/// <response code="200">Order found and returned.</response>
/// <response code="404">No order exists with the given ID.</response>
[HttpGet("{orderId:int}")]
[ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<ActionResult<OrderDto>> GetOrder(int orderId, CancellationToken ct) { ... }
```

## Quick reference cheat sheet

| Element | Convention | Example |
|---|---|---|
| Base path | `/api/v{n}` | `/api/v1` |
| Resources | plural, kebab-case | `/orders`, `/order-items` |
| IDs in path | `{resourceId}` | `/orders/{orderId}` |
| Query params | camelCase | `?pageSize=20&sortBy=createdDate` |
| JSON fields | camelCase | `"firstName": "Alex"` |
| Dates | ISO 8601 UTC | `"2026-04-30T14:32:00Z"` |
| Money | string + currency | `"totalAmount": "100.00", "currency": "USD"` |
| Errors | RFC 7807 | `{ "type": "...", "title": "...", "status": 400 }` |
| Pagination | envelope with `data` + `pagination` | `{ "data": [...], "pagination": {...} }` |
| Versioning | URL path | `/api/v1`, `/api/v2` |

---

**Last updated:** 2026-04-30
**Owner:** OrderHub Engineering
**Related:**
- [`AZURE_CONVENTIONS.md`](./AZURE_CONVENTIONS.md) — Azure resource naming conventions
- [`DATABASE_CONVENTIONS.md`](./DATABASE_CONVENTIONS.md) — Database object naming conventions
