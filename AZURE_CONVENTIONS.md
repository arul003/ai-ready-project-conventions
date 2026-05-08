# OrderHub — Authentication & Authorization Strategy

This document defines the authentication and authorization architecture for OrderHub. Authentication is delegated to **Auth0** (free tier); authorization is handled within **`OrderHub.API`** using a local roles/permissions model.

## TL;DR

- **Authentication** (who is this user?) → **Auth0 free tier** (email/password, single tenant)
- **Authorization** (what can they do?) → **`OrderHub.API`** with a local `Roles` table
- **User identity bridge** → `Users.ExternalUserId` column maps Auth0's `sub` claim to your local `UserId`
- **Just-in-time provisioning** → Local `Users` row is created on first successful login
- **Dev/prod isolation** → Two separate Auth0 free accounts (`orderhub-dev` and `orderhub-prod` tenants)
- **Customer API keys** → Deferred to Phase 2 (when external customer integrations begin)

## Why this architecture

### Why Auth0 over building it ourselves

Building your own authentication is a significant undertaking that's easy to get wrong:

- **Password hashing** — bcrypt, argon2, salting, peppering, work factors that age correctly
- **Brute-force protection** — rate limiting, lockouts, IP-based throttling
- **Password reset flows** — secure tokens, expiry, single-use, email delivery
- **Email verification** — token generation, validation, expiry
- **Session management** — secure cookies, CSRF protection, session invalidation
- **Breach detection** — monitoring known-breached password lists
- **Compliance** — SOC 2, GDPR, audit logging

Auth0 handles all of this for free up to 25,000 monthly active users. The engineering hours saved are worth far more than any cost we'd pay later when scaling beyond free.

### Why authorization in our app, not in Auth0

This is a deliberate architectural choice, not just a workaround for the free tier (Auth0's RBAC requires the Essentials plan, $35/month):

1. **Authorization rules live close to the data they protect.** "Can this user cancel this order?" requires knowing the order's customer, status, age — data we already have in `OrderHub.API`. Putting that logic in Auth0 means a callback to our API anyway, defeating the purpose.

2. **Business rules belong in the domain.** "Admins can cancel any order; users can only cancel their own orders within 24 hours of placing them" is domain logic — it lives in `OrderHub.Domain` and slice handlers, where it's testable, reviewable, and version-controlled.

3. **Single source of truth.** Roles and permissions live in our database alongside the rest of the domain model. Adding a new role is a code change + migration, reviewed in PR.

4. **No vendor lock-in for authorization.** If we ever migrate off Auth0, only the authentication layer changes. All authorization stays in place.

5. **Easier to test.** Unit-testable C# code beats integration tests against an external service.

This is a well-established pattern, sometimes called **"identity provider for authentication, application for authorization."** It's how most mature SaaS products are built.

## Auth0 setup

### Tenant naming

Create **two separate Auth0 free accounts** — one per environment:

| Environment | Auth0 tenant | Registered to |
|---|---|---|
| Development | `orderhub-dev.us.auth0.com` | `auth-dev@orderhub.com` (or similar alias) |
| Production | `orderhub-prod.us.auth0.com` | `auth-prod@orderhub.com` |

Use email aliases (e.g., `+dev`, `+prod` Gmail-style) to manage multiple Auth0 accounts under one inbox if needed.

**Why two accounts:** The Auth0 free tier allows only 1 tenant per account. Mixing dev test users with production users is a recipe for accidents — accidentally promoting dev configurations, polluting production user lists, or leaking test data. Hard separation at the tenant level prevents this entirely.

### Application setup (in each tenant)

In each Auth0 tenant, create:

#### 1. Single Page Application (SPA) for `orderhub-web`

| Setting | Value |
|---|---|
| Application Type | Single Page Application |
| Application Name | `orderhub-web` |
| Allowed Callback URLs (dev) | `http://localhost:3000/api/auth/callback` |
| Allowed Callback URLs (prod) | `https://app.orderhub.com/api/auth/callback` |
| Allowed Logout URLs | `http://localhost:3000`, `https://app.orderhub.com` |
| Allowed Web Origins | `http://localhost:3000`, `https://app.orderhub.com` |
| Token Endpoint Authentication | None (SPA) |
| Grant Types | Authorization Code, Refresh Token |

#### 2. API for `OrderHub.API`

| Setting | Value |
|---|---|
| Name | `OrderHub API` |
| Identifier (Audience) | `https://api.orderhub.com` (use the same value in dev — it's just an identifier, not a real URL) |
| Signing Algorithm | RS256 |
| Allow Skipping User Consent | ✅ (for first-party apps) |
| Token Lifetime | 86400 seconds (24 hours) — adjust based on security needs |

The **Identifier (Audience)** is what `OrderHub.API` validates in the JWT's `aud` claim. Use the same identifier in both dev and prod — only the issuer differs.

### Auth0 settings to enable

In Auth0 dashboard for each tenant:

- ✅ **Email & Password** connection (Database connection)
- ✅ **Email verification on signup** (Auth0 → Authentication → Database → Settings)
- ✅ **Password breach detection** (Security → Attack Protection → Breached Password Detection)
- ✅ **Brute-force protection** (Security → Attack Protection → Brute-force Protection)
- ✅ **Password policy:** Good or Excellent (Authentication → Database → Password Policy)
- ⏸️ **Multi-factor authentication** — defer to later, basic MFA is available on free tier when you're ready

## Database schema

### Users table — bridge to Auth0 identity

```sql
CREATE TABLE Users (
    UserId          INT             IDENTITY(1,1) NOT NULL,
    ExternalUserId  NVARCHAR(255)   NOT NULL,        -- Auth0 'sub' claim, e.g. "auth0|abc123xyz"
    EmailAddress    NVARCHAR(255)   NOT NULL,
    FirstName       NVARCHAR(100)   NOT NULL,
    LastName        NVARCHAR(100)   NOT NULL,
    RoleId          INT             NOT NULL,        -- FK to Roles
    IsActive        BIT             NOT NULL,
    LastLoginDate   DATETIME2       NULL,
    CreatedDate     DATETIME2       NOT NULL,
    CreatedBy       INT             NULL,            -- self-reference for first user; nullable
    ModifiedDate    DATETIME2       NULL,
    ModifiedBy      INT             NULL,
    IsDeleted       BIT             NOT NULL,

    CONSTRAINT PK_Users PRIMARY KEY (UserId),
    CONSTRAINT UQ_Users_ExternalUserId UNIQUE (ExternalUserId),
    CONSTRAINT UQ_Users_EmailAddress UNIQUE (EmailAddress),
    CONSTRAINT FK_Users_Roles FOREIGN KEY (RoleId) REFERENCES Roles(RoleId),
    CONSTRAINT DF_Users_IsActive DEFAULT (1) FOR IsActive,
    CONSTRAINT DF_Users_CreatedDate DEFAULT (SYSUTCDATETIME()) FOR CreatedDate,
    CONSTRAINT DF_Users_IsDeleted DEFAULT (0) FOR IsDeleted
);

CREATE INDEX IX_Users_ExternalUserId ON Users (ExternalUserId);
CREATE INDEX IX_Users_EmailAddress ON Users (EmailAddress) WHERE IsDeleted = 0;
```

**Key points:**
- `ExternalUserId` stores Auth0's unique `sub` claim — the immutable identifier from Auth0
- `EmailAddress` is duplicated locally (also in Auth0) so we can query without calling Auth0
- **No password column.** Auth0 owns credentials.
- `RoleId` is our authorization, completely independent of Auth0

### Roles table — authorization model

```sql
CREATE TABLE Roles (
    RoleId      INT             NOT NULL,
    Name        NVARCHAR(50)    NOT NULL,    -- 'Admin', 'User', 'Manager', etc.
    Description NVARCHAR(500)   NULL,

    CONSTRAINT PK_Roles PRIMARY KEY (RoleId),
    CONSTRAINT UQ_Roles_Name UNIQUE (Name)
);

-- Seed initial roles
INSERT INTO Roles (RoleId, Name, Description) VALUES
    (1, 'User', 'Standard OrderHub customer user'),
    (2, 'Admin', 'OrderHub internal staff with administrative access');
```

Use **fixed integer IDs** for roles so they can be referenced in code:

```csharp
public static class RoleIds
{
    public const int User = 1;
    public const int Admin = 2;
}
```

### Permissions (optional — only if needed later)

For now, role-based authorization is sufficient. If you later need fine-grained permissions:

```sql
CREATE TABLE Permissions (
    PermissionId   INT             NOT NULL,
    Name           NVARCHAR(100)   NOT NULL,    -- 'orders.cancel', 'users.create', etc.
    Description    NVARCHAR(500)   NULL,

    CONSTRAINT PK_Permissions PRIMARY KEY (PermissionId),
    CONSTRAINT UQ_Permissions_Name UNIQUE (Name)
);

CREATE TABLE RolePermissions (
    RoleId         INT NOT NULL,
    PermissionId   INT NOT NULL,

    CONSTRAINT PK_RolePermissions PRIMARY KEY (RoleId, PermissionId),
    CONSTRAINT FK_RolePermissions_Roles FOREIGN KEY (RoleId) REFERENCES Roles(RoleId),
    CONSTRAINT FK_RolePermissions_Permissions FOREIGN KEY (PermissionId) REFERENCES Permissions(PermissionId)
);
```

**Recommendation:** Don't build this on day one. Start with roles only. Add permissions when a real use case requires it.

## Authentication flow

### Overview

```
1. User opens orderhub-web
2. orderhub-web redirects to Auth0 login page (auth0.com hosted)
3. User enters email/password
4. Auth0 validates → redirects back to orderhub-web with auth code
5. orderhub-web exchanges code for JWT (access token + ID token)
6. orderhub-web stores tokens, calls OrderHub.API with Bearer JWT
7. OrderHub.API validates JWT signature & claims (using Auth0 public keys)
8. OrderHub.API looks up local user by ExternalUserId
9. If first login: creates local Users row (just-in-time provisioning)
10. OrderHub.API checks user's RoleId for authorization
11. Returns response
```

### Sequence diagram

```
┌──────────────┐    ┌────────┐    ┌────────────────┐    ┌──────┐
│ orderhub-web │    │ Auth0  │    │  OrderHub.API  │    │ SQL  │
└──────┬───────┘    └────┬───┘    └────────┬───────┘    └──┬───┘
       │                 │                  │                │
       │ 1. Login click  │                  │                │
       ├────────────────►│                  │                │
       │ 2. Email/pwd    │                  │                │
       │◄────────────────┤                  │                │
       │ 3. JWT          │                  │                │
       │◄────────────────┤                  │                │
       │ 4. GET /me with Bearer JWT          │                │
       ├──────────────────────────────────────►│              │
       │                 │                  │ 5. Validate JWT│
       │                 │ (public key)     │                │
       │                 │◄─────────────────┤                │
       │                 │                  │ 6. Find user   │
       │                 │                  ├───────────────►│
       │                 │                  │ 7. Not found   │
       │                 │                  │◄───────────────┤
       │                 │                  │ 8. Create user │
       │                 │                  ├───────────────►│
       │                 │                  │ 9. User row    │
       │                 │                  │◄───────────────┤
       │ 10. User profile + role             │                │
       │◄──────────────────────────────────────┤              │
```

## Backend integration: `OrderHub.API`

### Required NuGet packages

```xml
<PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.0" />
```

That's it. Auth0 uses standard JWT Bearer authentication — no Auth0-specific NuGet package required.

### Configuration

```json
// appsettings.json
{
  "Auth0": {
    "Domain": "",           // e.g. "orderhub-dev.us.auth0.com" — pulled from Key Vault in prod
    "Audience": "https://api.orderhub.com"
  }
}
```

In `appsettings.Development.json`:

```json
{
  "Auth0": {
    "Domain": "orderhub-dev.us.auth0.com",
    "Audience": "https://api.orderhub.com"
  }
}
```

In production, the domain is pulled from Azure Key Vault (`prod-orderhub-kv`).

### Configuration class

```csharp
// File: src/OrderHub.API/Configuration/Auth0Settings.cs
namespace OrderHub.API.Configuration;

public class Auth0Settings
{
    public const string SectionName = "Auth0";

    public string Domain { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;

    public string Authority => $"https://{Domain}/";
}
```

### JWT Bearer setup in `Program.cs`

```csharp
// File: src/OrderHub.API/Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using OrderHub.API.Configuration;

var builder = WebApplication.CreateBuilder(args);

builder.Services.Configure<Auth0Settings>(
    builder.Configuration.GetSection(Auth0Settings.SectionName));

var auth0Settings = builder.Configuration
    .GetSection(Auth0Settings.SectionName)
    .Get<Auth0Settings>()!;

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = auth0Settings.Authority;
        options.Audience = auth0Settings.Audience;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            NameClaimType = ClaimTypes.NameIdentifier,
            RoleClaimType = ClaimTypes.Role
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Admin", policy => policy.RequireRole("Admin"));
    options.AddPolicy("User", policy => policy.RequireRole("User", "Admin"));
});

// ... rest of setup

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapEndpoints();
app.Run();
```

The `Authority` URL (e.g., `https://orderhub-dev.us.auth0.com/`) is where ASP.NET Core fetches the JWT signing keys from. Auth0 publishes these at `<Authority>/.well-known/jwks.json` automatically.

### Just-in-time user provisioning

The first time a user authenticates with OrderHub, they exist in Auth0 but not in our `Users` table. We create the local row on first request — a pattern called **just-in-time (JIT) provisioning**.

The cleanest place to do this is in a **middleware** that runs after authentication but before any slice handler:

```csharp
// File: src/OrderHub.API/Common/Authentication/UserProvisioningMiddleware.cs
namespace OrderHub.API.Common.Authentication;

public class UserProvisioningMiddleware
{
    private readonly RequestDelegate _next;

    public UserProvisioningMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(
        HttpContext context,
        OrderHubDbContext db,
        ILogger<UserProvisioningMiddleware> logger)
    {
        if (context.User.Identity?.IsAuthenticated != true)
        {
            await _next(context);
            return;
        }

        var externalUserId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (string.IsNullOrEmpty(externalUserId))
        {
            await _next(context);
            return;
        }

        var localUser = await db.Users
            .FirstOrDefaultAsync(u => u.ExternalUserId == externalUserId);

        if (localUser is null)
        {
            // First login — provision the user
            var emailAddress = context.User.FindFirst(ClaimTypes.Email)?.Value
                ?? throw new InvalidOperationException("Email claim missing from JWT.");
            var firstName = context.User.FindFirst("given_name")?.Value ?? string.Empty;
            var lastName = context.User.FindFirst("family_name")?.Value ?? string.Empty;

            localUser = new User
            {
                ExternalUserId = externalUserId,
                EmailAddress = emailAddress,
                FirstName = firstName,
                LastName = lastName,
                RoleId = RoleIds.User,    // Default role for new signups
                IsActive = true,
                CreatedDate = DateTime.UtcNow
            };

            db.Users.Add(localUser);
            await db.SaveChangesAsync();

            logger.LogInformation("Provisioned new local user {UserId} for Auth0 sub {ExternalUserId}",
                localUser.UserId, externalUserId);
        }
        else
        {
            // Existing user — update last login
            localUser.LastLoginDate = DateTime.UtcNow;
            await db.SaveChangesAsync();
        }

        // Stash the local user info on the request for downstream handlers
        context.Items["LocalUser"] = localUser;

        await _next(context);
    }
}
```

Register in `Program.cs`:

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<UserProvisioningMiddleware>();   // After auth, before endpoints
app.MapEndpoints();
```

### Accessing the current user in slices

Create a helper service that abstracts the current user lookup:

```csharp
// File: src/OrderHub.API/Common/Authentication/ICurrentUserService.cs
namespace OrderHub.API.Common.Authentication;

public interface ICurrentUserService
{
    int UserId { get; }
    int RoleId { get; }
    string EmailAddress { get; }
    bool IsAuthenticated { get; }
    bool IsAdmin { get; }
}

// File: src/OrderHub.API/Common/Authentication/CurrentUserService.cs
public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    private User? LocalUser =>
        _httpContextAccessor.HttpContext?.Items["LocalUser"] as User;

    public int UserId => LocalUser?.UserId
        ?? throw new InvalidOperationException("No authenticated user.");

    public int RoleId => LocalUser?.RoleId
        ?? throw new InvalidOperationException("No authenticated user.");

    public string EmailAddress => LocalUser?.EmailAddress ?? string.Empty;

    public bool IsAuthenticated => LocalUser is not null;

    public bool IsAdmin => LocalUser?.RoleId == RoleIds.Admin;
}
```

Register:

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<ICurrentUserService, CurrentUserService>();
```

Use in slice handlers:

```csharp
public static async Task<IResult> HandleAsync(
    Command command,
    OrderHubDbContext db,
    ICurrentUserService currentUser,
    CancellationToken ct)
{
    // Use currentUser.UserId, currentUser.IsAdmin, etc.
    var order = new Order
    {
        CustomerId = currentUser.UserId,
        // ...
    };

    // ...
}
```

### Authorization in slice handlers

Three layers of authorization, applied as needed:

#### 1. Endpoint-level: requires authentication

Default for nearly all endpoints — at the `MapEndpoint` level:

```csharp
app.MapPost("/api/v1/orders", HandleAsync)
    .RequireAuthorization();    // Any authenticated user
```

#### 2. Endpoint-level: requires specific role

```csharp
app.MapDelete("/api/v1/users/{userId}", HandleAsync)
    .RequireAuthorization("Admin");    // Only admins
```

The `"Admin"` matches the policy registered in `Program.cs`.

#### 3. Resource-level: business rules in the handler

For "user can only access their own data" rules, check inside the handler:

```csharp
public static async Task<IResult> HandleAsync(
    int orderId,
    OrderHubDbContext db,
    ICurrentUserService currentUser,
    CancellationToken ct)
{
    var order = await db.Orders.FindAsync([orderId], ct);
    if (order is null)
        return Results.NotFound();

    // Business rule: users can only see their own orders; admins can see all
    if (!currentUser.IsAdmin && order.CustomerId != currentUser.UserId)
        return Results.Forbid();

    return Results.Ok(order);
}
```

This is where the "authorization in the app" approach pays off — the rule lives next to the data it protects, fully testable, fully reviewable.

### The `/api/v1/me` endpoint

A standard pattern: expose a lightweight endpoint that returns the current user's profile and role. The frontend uses this to populate UI state (show/hide admin features, display name in header, etc.):

```csharp
// File: src/OrderHub.API/Features/Users/GetCurrentUser/GetCurrentUser.cs
namespace OrderHub.API.Features.Users.GetCurrentUser;

public static class GetCurrentUser
{
    public record Response(
        int UserId,
        string EmailAddress,
        string FirstName,
        string LastName,
        string RoleName,
        bool IsAdmin);

    public static async Task<IResult> HandleAsync(
        OrderHubDbContext db,
        ICurrentUserService currentUser,
        CancellationToken ct)
    {
        var user = await db.Users
            .Include(u => u.Role)
            .FirstOrDefaultAsync(u => u.UserId == currentUser.UserId, ct);

        if (user is null)
            return Results.NotFound();

        return Results.Ok(new Response(
            user.UserId,
            user.EmailAddress,
            user.FirstName,
            user.LastName,
            user.Role.Name,
            user.RoleId == RoleIds.Admin));
    }

    public static void MapEndpoint(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/v1/me", HandleAsync)
            .WithName("GetCurrentUser")
            .RequireAuthorization()
            .Produces<Response>(StatusCodes.Status200OK);
    }
}
```

## Frontend integration: `orderhub-web`

### Recommended SDK

Auth0 provides an official Next.js SDK that handles the OAuth dance:

```bash
pnpm add @auth0/nextjs-auth0
```

This SDK handles login/logout, callback routes, session cookies, and token refresh automatically.

### Environment variables

```env
# .env.local (gitignored)
AUTH0_SECRET=                              # generate via: openssl rand -hex 32
AUTH0_BASE_URL=http://localhost:3000
AUTH0_ISSUER_BASE_URL=https://orderhub-dev.us.auth0.com
AUTH0_CLIENT_ID=                           # from Auth0 SPA application
AUTH0_CLIENT_SECRET=                       # from Auth0 SPA application
AUTH0_AUDIENCE=https://api.orderhub.com    # matches OrderHub.API audience
AUTH0_SCOPE=openid profile email offline_access
```

In production (Azure Static Web App), set these in Application Settings, not in code.

### Auth routes

The Auth0 SDK auto-generates these route handlers in `src/app/api/auth/[auth0]/route.ts`:

```typescript
// File: src/app/api/auth/[auth0]/route.ts
import { handleAuth } from '@auth0/nextjs-auth0';

export const GET = handleAuth();
```

This single line creates `/api/auth/login`, `/api/auth/logout`, `/api/auth/callback`, and `/api/auth/me`.

### Wrapping the app

```typescript
// File: src/app/layout.tsx
import { UserProvider } from '@auth0/nextjs-auth0/client';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <UserProvider>{children}</UserProvider>
      </body>
    </html>
  );
}
```

### Login / logout

```typescript
// In any client component
import { useUser } from '@auth0/nextjs-auth0/client';

export function LoginButton() {
  const { user, isLoading } = useUser();

  if (isLoading) return <div>Loading...</div>;

  if (user) {
    return <a href="/api/auth/logout">Logout</a>;
  }

  return <a href="/api/auth/login">Login</a>;
}
```

### Calling `OrderHub.API` with the access token

The Auth0 SDK manages access tokens. Pass them on every API request:

```typescript
// File: src/lib/api-client.ts
import { getAccessToken } from '@auth0/nextjs-auth0';

export async function apiRequest<T>(path: string, init?: RequestInit): Promise<T> {
  const { accessToken } = await getAccessToken();

  const response = await fetch(`${process.env.NEXT_PUBLIC_API_BASE_URL}${path}`, {
    ...init,
    headers: {
      ...init?.headers,
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    },
  });

  if (!response.ok) {
    throw new Error(`API request failed: ${response.status}`);
  }

  return response.json();
}
```

### Server Components: `getSession()`

For Server Components that need user context server-side:

```typescript
// File: src/app/(app)/dashboard/page.tsx
import { getSession } from '@auth0/nextjs-auth0';

export default async function DashboardPage() {
  const session = await getSession();

  if (!session) {
    redirect('/api/auth/login');
  }

  // Server-side: fetch data scoped to the authenticated user
  const orders = await fetchOrders(session.accessToken);

  return <OrderList orders={orders} />;
}
```

### Showing/hiding UI based on role

The current user's role comes from `/api/v1/me`, not from Auth0:

```typescript
// File: src/features/auth/hooks/useCurrentUser.ts
import { useQuery } from '@tanstack/react-query';
import { apiRequest } from '@/lib/api-client';

interface CurrentUser {
  userId: number;
  emailAddress: string;
  firstName: string;
  lastName: string;
  roleName: string;
  isAdmin: boolean;
}

export function useCurrentUser() {
  return useQuery({
    queryKey: ['me'],
    queryFn: () => apiRequest<CurrentUser>('/api/v1/me'),
    staleTime: 5 * 60 * 1000,    // 5 minutes — role rarely changes mid-session
  });
}
```

```typescript
// Usage in components
function AdminPanelLink() {
  const { data: currentUser } = useCurrentUser();

  if (!currentUser?.isAdmin) return null;

  return <Link href="/admin">Admin Panel</Link>;
}
```

## Operational concerns

### Storing Auth0 secrets

| Environment | Storage |
|---|---|
| Local dev (frontend) | `.env.local` (gitignored) |
| Local dev (backend) | User Secrets (`dotnet user-secrets`) |
| Staging / Prod (frontend) | Azure Static Web App Application Settings |
| Staging / Prod (backend) | Azure Key Vault (`prod-orderhub-kv`), referenced via Key Vault references |

**Never commit Auth0 client secrets to git.** They live only in environment configuration.

### CI/CD considerations

Each environment has its own Auth0 tenant. Pipeline secrets:

- **Dev pipeline:** points to `orderhub-dev.us.auth0.com`
- **Prod pipeline:** points to `orderhub-prod.us.auth0.com`

Configure these as environment-specific variables in Azure DevOps.

### Logging auth events to Application Insights

Log key auth events as structured events for observability:

```csharp
// In UserProvisioningMiddleware
logger.LogInformation("User {UserId} authenticated from {RemoteIp}",
    localUser.UserId, context.Connection.RemoteIpAddress);

logger.LogInformation("New user {UserId} provisioned for email {EmailAddress}",
    localUser.UserId, localUser.EmailAddress);
```

For failed authentication attempts, Auth0's own dashboard shows the audit log. App Insights captures the resulting 401s on your API.

### Token lifetime

Default Auth0 access token lifetime is 24 hours, refresh token is unlimited (until revoked). For OrderHub:

- **Access token:** 24 hours (good balance — long enough that users aren't constantly refreshing, short enough that revocation matters)
- **Refresh token:** rotate on use, infinite expiry (the Auth0 SDK handles this)

If security requirements grow, shorten access tokens to 1 hour and rely on refresh tokens for silent renewal.

### Logout

A "logout" should:

1. Clear the session cookie in `orderhub-web`
2. Redirect to Auth0's logout URL to clear Auth0's session
3. Redirect back to OrderHub's home page

The `@auth0/nextjs-auth0` SDK handles all of this automatically when the user hits `/api/auth/logout`.

**Note:** Logout doesn't invalidate the JWT itself. Once issued, a JWT is valid until expiry. For high-security scenarios where instant logout is required, you'd need a token revocation list — overkill for OrderHub.

## Common pitfalls and debugging

### "401 Unauthorized" on every API call

Most common causes:
1. **Audience mismatch** — `appsettings.json` Audience doesn't match Auth0 API identifier
2. **Issuer mismatch** — Domain in config has typo or trailing slash issues (Auth0 requires trailing `/` in issuer)
3. **Wrong tenant** — frontend is hitting prod Auth0, backend is validating dev tokens
4. **Clock skew** — server clock is wildly off, JWT appears expired immediately

**Debug:** Decode the JWT at [jwt.io](https://jwt.io) and compare `iss` and `aud` claims to your API's expected values.

### "User authenticated but I get 403 Forbidden"

The JWT is valid (authentication works) but authorization is failing:
1. Check the user's `RoleId` in the database
2. Check the policy on the endpoint matches the user's role
3. Verify `UserProvisioningMiddleware` is running and creating the local user

### First-time user gets "User not found"

Just-in-time provisioning isn't running. Verify:
1. `UseMiddleware<UserProvisioningMiddleware>()` is registered after `UseAuthentication()`
2. The middleware reads claims correctly (especially `email`)
3. The Auth0 token includes the `email` claim — set scope to `openid profile email`

### CORS errors from `orderhub-web` to `OrderHub.API`

In `Program.cs`, configure CORS with the frontend's origin:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("OrderHubWeb", policy =>
    {
        policy.WithOrigins(
                "http://localhost:3000",
                "https://app.orderhub.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

// ...
app.UseCors("OrderHubWeb");
```

### Local development: Auth0 callback fails

Ensure `http://localhost:3000/api/auth/callback` is listed in **Allowed Callback URLs** in the Auth0 SPA application settings. Mismatches result in `Callback URL mismatch` errors.

## Phase 2: Customer API keys (future)

When external customers begin integrating with `OrderHub.API`, they'll need API access without going through user login flows. This is **deferred** — not built on day one.

### Approach: Auth0 Machine-to-Machine (M2M) applications

Auth0 supports M2M tokens for service-to-service auth. The flow:

1. Each customer gets one Auth0 M2M application (created in Auth0 dashboard)
2. Customer receives a **client ID** and **client secret**
3. Customer's backend exchanges these for an access token (client credentials grant)
4. Customer calls `OrderHub.API` with the access token in `Authorization: Bearer <token>`
5. `OrderHub.API` validates the token (same JWT validation as user tokens)
6. Token's `sub` claim distinguishes customers (e.g., `client_id@clients`)

### Free tier limitations for M2M

The Auth0 free tier includes a limited number of M2M tokens per month. Watch this metric — it can be the trigger that pushes us off the free tier before MAUs do.

### Implementation deferred

Detailed M2M implementation will be added to this document when external customer integrations begin. The architectural foundation (JWT validation, identity bridge) supports this seamlessly when needed.

## Quick reference

### Auth0 setup checklist

- [ ] Create Auth0 free account for dev tenant (`orderhub-dev`)
- [ ] Create Auth0 free account for prod tenant (`orderhub-prod`)
- [ ] Create SPA application (`orderhub-web`) in each tenant
- [ ] Create API (`OrderHub API`) in each tenant
- [ ] Configure Allowed Callback URLs and Logout URLs for each environment
- [ ] Enable email verification on signup
- [ ] Enable breached password detection
- [ ] Enable brute-force protection
- [ ] Set password policy to "Good" or "Excellent"

### Database setup checklist

- [ ] Create `Roles` table with seed data (`User`, `Admin`)
- [ ] Create `Users` table with `ExternalUserId` column
- [ ] Add unique constraint on `Users.ExternalUserId`
- [ ] Add index on `Users.ExternalUserId` and `Users.EmailAddress`

### Backend setup checklist

- [ ] Add `Microsoft.AspNetCore.Authentication.JwtBearer` package
- [ ] Configure `AddJwtBearer` with Auth0 Authority and Audience
- [ ] Add authorization policies for `Admin` and `User`
- [ ] Implement `UserProvisioningMiddleware`
- [ ] Implement `ICurrentUserService` and register
- [ ] Implement `/api/v1/me` endpoint
- [ ] Configure CORS for the frontend origin
- [ ] Store Auth0 secrets in Key Vault (production)

### Frontend setup checklist

- [ ] Install `@auth0/nextjs-auth0`
- [ ] Configure environment variables (`AUTH0_*`)
- [ ] Add `/api/auth/[auth0]/route.ts` handler
- [ ] Wrap app in `<UserProvider>`
- [ ] Implement API client that attaches Bearer token
- [ ] Implement `useCurrentUser` hook calling `/api/v1/me`
- [ ] Configure protected routes via middleware

---

**Last updated:** 2026-04-30
**Owner:** OrderHub Engineering
**Related:**
- [`AZURE_CONVENTIONS.md`](./AZURE_CONVENTIONS.md) — Azure resource naming
- [`DATABASE_CONVENTIONS.md`](./DATABASE_CONVENTIONS.md) — Database object naming (`Users`, `Roles` tables)
- [`API_CONVENTIONS.md`](./API_CONVENTIONS.md) — REST API conventions (`/api/v1/me` endpoint)
- [`CSHARP_CONVENTIONS.md`](./CSHARP_CONVENTIONS.md) — C# / .NET conventions (middleware, slices)
- [`NEXTJS_CONVENTIONS.md`](./NEXTJS_CONVENTIONS.md) — Next.js / frontend conventions (Auth0 SDK integration)
- [`GIT_CONVENTIONS.md`](./GIT_CONVENTIONS.md) — Branch and commit conventions
