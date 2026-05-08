# AGENTS.md — Context for AI Coding Agents

This file is the entry point for AI coding agents (Claude Code, Cursor, GitHub Copilot, Codex, etc.) working in this repository. Read this first to understand the project's conventions before generating, modifying, or reviewing code.

If you're a human reader: you can find the same information in more detail in the individual `*_CONVENTIONS.md` files referenced below.

## Project overview

**OrderHub** is a SaaS order-management product. The codebase consists of:

- **`orderhub-api`** — C# / .NET 10 backend using Vertical Slice Architecture
- **`orderhub-web`** — Next.js 16 frontend using App Router
- **`orderhub-infra`** — Bicep / Terraform infrastructure-as-code (Azure)

Production runs on Azure: App Service for the API, Static Web App for the frontend, Azure SQL Database, Auth0 for authentication.

## Critical conventions

### Naming style by context

The same logical concept is written differently depending on the layer. This is intentional — each ecosystem has its own convention.

| Context | Style | Example |
|---|---|---|
| Repo names | lowercase-kebab | `orderhub-api` |
| Azure resources | lowercase-kebab | `prod-orderhub-app-api` |
| URL paths | lowercase-kebab | `/api/v1/order-items` |
| Query params | camelCase | `?pageSize=20` |
| JSON fields | camelCase | `"firstName": "Alex"` |
| C# classes / methods / properties | PascalCase | `OrderService`, `GetOrderAsync` |
| C# private fields | `_camelCase` | `_dbContext` |
| SQL table / column names | PascalCase | `Orders`, `OrderId` |
| Database name | PascalCase | `OrderHub` |
| TypeScript / React components | PascalCase | `OrderCard.tsx` |
| TypeScript hooks | camelCase, `use` prefix | `useOrders` |
| Environment variables | SCREAMING_SNAKE_CASE | `DATABASE_CONNECTION_STRING` |

### Database (SQL Server)

- **Primary keys:** `<Entity>Id` (e.g., `OrderId`, `UserId`) — NOT plain `Id`
- **Tables:** plural, PascalCase (`Orders`, `Users`, `OrderItems`)
- **Schema-qualified queries:** always use `dbo.Orders`, not `Orders`
- **Stored procedures:** `usp_<VerbNoun>` prefix (never `sp_` — system procs only)
- **Foreign keys:** match the referenced PK name (`Orders.CustomerId` → `Customers.CustomerId`)
- **Use plural tables**: `Statuses`, not `Status` (and prefer domain-qualified names like `OrderStatuses`)
- **Lookup tables**: dedicated tables for business-meaningful lookups; `config.DropDownLists` only for trivial UI dropdowns

See `DATABASE_CONVENTIONS.md` for the full reference.

### REST API

- **Single API surface:** `/api/v1/...` — no `/api/internal/`, no audience splits
- **Resources, not actions:** `POST /api/v1/orders` (create) — never `POST /api/v1/createOrder`
- **Plural resource names:** `/orders`, `/users`, `/order-items`
- **HTTP verbs:** GET (read), POST (create), PATCH (partial update), PUT (replace), DELETE
- **Status codes:** 200 OK, 201 Created (with `Location` header), 204 No Content, 400 Validation, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict
- **Errors:** RFC 7807 Problem Details
- **Pagination:** envelope with `data` + `pagination` for list endpoints
- **Action endpoints:** `POST /api/v1/orders/{id}/cancellation` (noun form), not `POST /api/v1/orders/{id}/cancel`

See `API_CONVENTIONS.md` for the full reference.

### C# backend (Vertical Slice Architecture)

**Architecture:** Vertical Slice — code organized by feature, not technical layer.

**Solution structure (3 projects only):**
```
src/
├── OrderHub.API/              ← All feature slices, endpoint registration
├── OrderHub.Domain/           ← Entities, value objects (depends on nothing)
└── OrderHub.Infrastructure/   ← DbContext, EF configurations, external services
```

**No separate `Application` layer.** Application logic lives inside each feature slice.

**Slice structure:** One folder per feature, one file per slice (default).

```
Features/Orders/CreateOrder/
└── CreateOrder.cs    ← Command, Validator, Handler, Endpoint all in one file
```

**File structure within a slice (top to bottom):**
1. `Command` (writes) or `Query` (reads) record
2. Sub-records used by the request
3. `Response` record
4. `Validator` class (FluentValidation)
5. `HandleAsync` method (the logic)
6. `MapEndpoint` method (route registration)

**Key technology choices:**
- **Minimal APIs** (not Controllers)
- **EF Core directly** (no repository pattern)
- **No MediatR** (direct method calls — `CreateOrder.HandleAsync(...)`)
- **FluentValidation** for request validation
- **`ILogger<T>` + Application Insights** (not Serilog)
- **JWT Bearer authentication** validating Auth0 tokens

**Async conventions:**
- All async methods end in `Async`
- Always accept `CancellationToken` (last parameter)
- Never `async void` (except event handlers)

See `CSHARP_CONVENTIONS.md` for the full reference.

### Next.js frontend

- **App Router only** (Pages Router is in maintenance mode)
- **Server Components by default**, mark `"use client"` only when needed
- **Named exports** for components (except `page.tsx`, `layout.tsx` etc. which Next.js requires as default exports)
- **Feature-based folders:** `src/features/<domain>/{components,hooks,api,schemas}/`
- **Path alias:** `@/*` → `src/*`
- **TanStack Query** for server state, **React Hook Form + Zod** for forms
- **Tailwind CSS** with `cn()` for conditional classes
- **shadcn/ui** for component primitives (copy into codebase, not a dependency)

See `NEXTJS_CONVENTIONS.md` for the full reference.

### Git workflow

**Branches:**
- `prod` — production (NOT `main`)
- `dev` — integration branch (deploys to dev environment)
- `feature/*`, `bugfix/*`, `hotfix/*`, `refactor/*`, `chore/*` — short-lived branches off `dev`

**Flow:** `feature/* → PR → dev → Test & Validate → PR → prod`

**Hotfixes:** branch from `prod`, merge to `prod`, **then back-merge to `dev`** (this back-merge is critical — never skip it).

**Commit format:** `<type>: <subject>` (Conventional Commits — `feat:`, `fix:`, `refactor:`, etc.)

**Merge strategy:** squash merge (clean history, easy reverts).

See `GIT_CONVENTIONS.md` for the full reference.

### Authentication & authorization

- **Authentication:** Auth0 (free tier) — issues JWTs
- **Authorization:** in our app — `Roles` table in our database, `[Authorize(Roles = "Admin")]` attributes
- **User identity bridge:** `Users.ExternalUserId` column stores Auth0's `sub` claim
- **Just-in-time provisioning:** local `Users` row created on first successful login
- **Frontend SDK:** `@auth0/nextjs-auth0`
- **Backend:** `Microsoft.AspNetCore.Authentication.JwtBearer` (no Auth0-specific package needed)
- **`/api/v1/me` endpoint:** returns current user's local profile + role (frontend reads role from here, not from Auth0 token)

See `AUTH_STRATEGY.md` for the full reference.

## When generating code, an AI agent should

### ✅ Do

- Read the relevant `*_CONVENTIONS.md` file before generating non-trivial code in that area
- Match the casing conventions exactly — they're not negotiable
- Use the established patterns (slice structure, naming prefixes, conventions) without deviation
- Use plural table names, `<Entity>Id` primary keys, `usp_` stored proc prefix
- Put new C# features under `Features/<Domain>/<FeatureName>/` as a single-file slice
- Inject `CancellationToken` into all async methods
- Use structured logging: `_logger.LogInformation("Order {OrderId} created", order.OrderId)` — never string interpolation
- Keep PRs small (under ~400 lines of code changed)

### ❌ Don't

- Add a separate `Application` layer to the C# solution (we use VSA, not Clean Architecture)
- Add MediatR unless explicitly requested (slices use direct method calls)
- Add the repository pattern (we use `DbContext` directly)
- Use `Id` as a primary key column name — use `<Entity>Id`
- Use `sp_` prefix for stored procedures (this is reserved for SQL Server system procs)
- Use string interpolation in log messages (defeats structured logging)
- Use `any` in TypeScript without a written reason
- Put business logic in Auth0 — authorization stays in our app
- Mix conventions across the stack (kebab-case URLs but camelCase class names — each layer follows its own)

## Quick reference for common tasks

### Adding a new API endpoint (C#)

1. Create folder: `src/OrderHub.API/Features/<Domain>/<FeatureName>/`
2. Create file: `<FeatureName>.cs`
3. Inside: `public static class <FeatureName>` containing `Command`/`Query`, `Response`, `Validator`, `HandleAsync`, `MapEndpoint`
4. Register the endpoint in `Common/Endpoints/EndpointRegistration.cs`
5. Add tests in `tests/OrderHub.API.Tests/Features/<Domain>/`

### Adding a new database entity

1. Define entity in `src/OrderHub.Domain/Entities/<EntityName>.cs` (PascalCase, plural table)
2. Add EF configuration in `src/OrderHub.Infrastructure/Persistence/Configurations/<EntityName>Configuration.cs`
3. Add `DbSet<EntityName>` to `OrderHubDbContext`
4. Generate migration: `dotnet ef migrations add Add<EntityName>Table`
5. Standard audit columns required: `CreatedDate`, `CreatedBy`, `ModifiedDate`, `ModifiedBy`, `IsDeleted`
6. Primary key column: `<EntityName>Id` (not `Id`)

### Adding a new React component

1. Determine scope: feature-specific → `src/features/<domain>/components/`, app-wide → `src/components/`
2. File name: `ComponentName.tsx` (PascalCase)
3. Use **named export** (`export function ComponentName(...)`)
4. Server Component by default; `"use client"` only if interactivity is needed
5. Props interface: `<ComponentName>Props`

### Adding a new database query (in a slice)

```csharp
// Inside a Query slice, query DbContext directly:
public static async Task<IResult> HandleAsync(
    int orderId,
    OrderHubDbContext db,
    CancellationToken ct)
{
    var order = await db.Orders
        .AsNoTracking()
        .Where(o => o.OrderId == orderId)
        .Select(o => new Response(o.OrderId, o.TotalAmount, o.Status.ToString()))
        .FirstOrDefaultAsync(ct);

    return order is null ? Results.NotFound() : Results.Ok(order);
}
```

No repository, no service layer — query the `DbContext` directly inside the slice.

## Document index

| If you're working on... | Read first |
|---|---|
| Azure resources, infrastructure | `AZURE_CONVENTIONS.md` |
| Database schema, queries, migrations | `DATABASE_CONVENTIONS.md` |
| API endpoints, request/response shapes | `API_CONVENTIONS.md` |
| C# backend code, slices, services | `CSHARP_CONVENTIONS.md` |
| Next.js components, routes, hooks | `NEXTJS_CONVENTIONS.md` |
| Git, branches, commits, PRs | `GIT_CONVENTIONS.md` |
| Login, authentication, roles | `AUTH_STRATEGY.md` |

If unsure which document applies, start with this `AGENTS.md` and follow the pointers.

---

**Note for AI agents:** This document is intentionally compact. For detailed conventions, read the corresponding `*_CONVENTIONS.md` file. When generating non-trivial code, reading the full convention doc once is much cheaper than producing inconsistent code that needs revision.
