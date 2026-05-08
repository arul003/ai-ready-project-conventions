# OrderHub — Database Naming Conventions

This document defines naming conventions for everything **inside** the OrderHub database — tables, columns, indexes, stored procedures, views, and more. For Azure resource naming (SQL Server, databases as Azure resources, etc.), see [`AZURE_CONVENTIONS.md`](./AZURE_CONVENTIONS.md).

## Stack assumptions

- **Database engine:** Azure SQL Database (SQL Server)
- **ORM:** Entity Framework Core
- **Application:** C# / .NET 10 (`OrderHub.API`)
- **Default collation:** Case-insensitive (`SQL_Latin1_General_CP1_CI_AS`)

## Core principle

**PascalCase everywhere, plural for collections, singular for entities.** This matches Entity Framework's default conventions, the broader SQL Server ecosystem (AdventureWorks, Northwind, WideWorldImporters), and the casing used throughout `OrderHub.API` C# code.

When EF defaults match your naming, you write less code and avoid scattering `[Table]` and `[Column]` attributes throughout the domain layer.

## Quick reference

| Object | Convention | Example |
|---|---|---|
| Database | PascalCase | `OrderHub` |
| Schema | lowercase | `dbo`, `audit`, `identity`, `reporting` |
| Table | PascalCase, **plural** | `Users`, `Orders`, `OrderItems` |
| Column | PascalCase, **singular** | `UserId`, `FirstName`, `CreatedDate` |
| Primary key column | `<Entity>Id` | `UserId`, `OrderId`, `ProductId` |
| Foreign key column | `<Entity>Id` | `CustomerId`, `OrderId` |
| Primary key constraint | `PK_<Table>` | `PK_Users` |
| Foreign key constraint | `FK_<Table>_<ReferencedTable>` | `FK_Orders_Customers` |
| Unique constraint | `UQ_<Table>_<Column>` | `UQ_Users_EmailAddress` |
| Check constraint | `CK_<Table>_<Column>` | `CK_Products_Price` |
| Default constraint | `DF_<Table>_<Column>` | `DF_Users_CreatedDate` |
| Index (non-clustered) | `IX_<Table>_<Columns>` | `IX_Users_EmailAddress` |
| Unique index | `UX_<Table>_<Columns>` | `UX_Users_Username` |
| Stored procedure | `usp_<VerbNoun>` | `usp_GetUserById`, `usp_CreateOrder` |
| Function (scalar) | `ufn_<VerbNoun>` | `ufn_CalculateOrderTotal` |
| Function (table-valued) | `tvf_<VerbNoun>` | `tvf_GetActiveUsers` |
| View | `vw_<Name>` | `vw_ActiveUsers`, `vw_OrderSummary` |
| Trigger | `tr_<Table>_<Action>` | `tr_Users_AfterUpdate` |
| Sequence | `sq_<Name>` | `sq_OrderNumber` |

## Tables

### Naming

- **PascalCase, plural noun.** A table represents a collection of entities, so the name is plural.
- The corresponding C# entity class is the **singular** form.

| Table (DB) | Entity (C#) |
|---|---|
| `Users` | `User` |
| `Orders` | `Order` |
| `OrderItems` | `OrderItem` |
| `Categories` | `Category` |

### Schema usage

- **`dbo`** — Default schema for the main application data
- **`audit`** — Audit log tables (`audit.UserChanges`, `audit.OrderHistory`)
- **`identity`** — Auth/user-management tables if not using a separate database
- **`reporting`** — Read-only views and denormalized tables for reporting
- **`staging`** — ETL or import staging tables
- **`config`** — Configuration and generic lookup data (see [Lookup tables and reference data](#lookup-tables-and-reference-data))

Always reference tables with the schema prefix in queries and stored procs (`dbo.Users`, not just `Users`) — improves the query optimizer's plan caching and makes intent explicit.

### Avoid

- ❌ Reserved words as table names: `User`, `Order`, `Group`, `Transaction` — these require bracket-quoting (`[User]`) which is friction. Use plurals to dodge most reserved words automatically (`Users`, `Orders`).
- ❌ Hungarian notation prefixes: `tblUsers`, `tbl_Users` — outdated, redundant.
- ❌ Abbreviations unless universally understood: `Cust` instead of `Customers`, `Trx` instead of `Transactions`.

### Pluralization edge cases

Some English words have awkward, irregular, or uncountable plural forms. Here's how to handle them under our plural-table convention.

#### Common awkward plurals (use as-is)

| Singular (Entity) | Plural (Table) | Notes |
|---|---|---|
| `Status` | `Statuses` | Awkward double-s but grammatically correct |
| `Address` | `Addresses` | Same pattern, reads fine in practice |
| `Category` | `Categories` | y → ies, EF Core pluralizer handles automatically |
| `Country` | `Countries` | Same y → ies pattern |
| `Company` | `Companies` | Same |
| `Activity` | `Activities` | Same |
| `Index` | `Indexes` | Both `Indexes` and `Indices` are valid; `Indexes` is more common in DB contexts |
| `Person` | `People` | Irregular — EF Core's pluralizer recognizes this |
| `Child` | `Children` | Irregular — EF Core handles it |

#### Generic names: qualify with a domain prefix instead

Generic single-word names like `Status`, `Type`, `Category`, `Group`, `Role` are usually too vague — and the plural form (`Statuses`, `Types`) looks awkward. **Prefer domain-qualified names:**

| ❌ Generic | ✅ Domain-qualified |
|---|---|
| `Statuses` | `OrderStatuses`, `PaymentStatuses`, `TicketStatuses` |
| `Types` | `ProductTypes`, `AccountTypes`, `DocumentTypes` |
| `Categories` | `ProductCategories`, `ArticleCategories` |
| `Groups` | `UserGroups`, `PermissionGroups` |
| `Levels` | `AccessLevels`, `SubscriptionLevels` |

This solves three problems at once:

1. **The plural reads naturally** — `OrderStatuses` is broken up enough that the double-s feels fine.
2. **The table name is self-documenting** — anyone seeing `OrderStatuses` immediately knows what it holds.
3. **Avoids name collisions** — multiple teams can have their own status/type tables (`OrderStatuses`, `PaymentStatuses`) without conflict.

The corresponding C# entity follows the same pattern: `OrderStatus`, `PaymentStatus`, `ProductType`.

#### Uncountable nouns

Some nouns don't have a plural form in English (`Data`, `Equipment`, `Information`, `News`, `Software`, `Series`). Don't try to force one (`Datas`, `Equipments` — both wrong). Two valid approaches:

**Option A: Leave as singular** (the noun is already plural in meaning)
```
Equipment       (table, but contains many equipment rows)
Software        (table)
```

**Option B: Rename to a countable noun** (preferred — keeps the plural-table convention consistent)

| Uncountable | Countable replacement |
|---|---|
| `Data` | `DataPoints`, `DataEntries`, `DataRecords` |
| `Equipment` | `EquipmentItems`, `Assets` |
| `Information` | `Notes`, `Records`, `Details` |
| `News` | `NewsItems`, `Articles`, `Announcements` |
| `Software` | `Applications`, `SoftwarePackages` |
| `Feedback` | `FeedbackItems`, `Comments`, `Reviews` |

**Recommendation:** Default to Option B — renaming to a countable noun keeps the table-naming convention uniform across the entire schema. Reserve Option A for cases where renaming feels forced.

#### EF Core pluralizer

EF Core's built-in pluralizer (used when generating tables from entities) handles most cases automatically:

```csharp
public DbSet<User> Users { get; set; }              // → Users
public DbSet<Category> Categories { get; set; }     // → Categories
public DbSet<Status> Statuses { get; set; }         // → Statuses
public DbSet<Person> People { get; set; }           // → People
public DbSet<Child> Children { get; set; }          // → Children
```

For irregular cases where the pluralizer gets it wrong, override with `[Table]`:

```csharp
[Table("EquipmentItems")]
public class EquipmentItem { ... }
```

## Lookup tables and reference data

Most applications accumulate dozens of small "lookup" tables — order statuses, countries, currencies, payment methods, contact preferences, etc. Two common approaches exist for handling these, and neither is right in every case.

### The decision rule

For every lookup, ask one question:

> **"Will any code, query, or business rule ever care about specific values in this list?"**

| Answer | Approach | Schema |
|---|---|---|
| **Yes** | Separate dedicated table | `dbo` (or domain schema) |
| **No** | One row in shared `DropDownLists` table | `config` |

### Approach 1: Separate tables (for business-meaningful lookups)

Use a dedicated table when the lookup has any of these characteristics:

- **Foreign key relationships** with other tables (referential integrity matters)
- **Type-specific columns** beyond just `Code` and `DisplayValue` (workflow flags, business rules)
- **Code references specific values** (`WHERE OrderStatusId = 2` or `WHERE Code = 'PENDING'`)
- **Domain semantics** that other developers must understand

**Examples for OrderHub:**

```sql
dbo.OrderStatuses          -- workflow logic, FK from Orders
dbo.PaymentStatuses        -- payment processing rules, FK from Payments
dbo.Countries              -- FK from Addresses
dbo.Currencies             -- FK from Orders, Transactions
dbo.UserRoles              -- permissions logic
dbo.ProductCategories      -- FK from Products
```

**Why dedicated tables win for these:**
- SQL Server enforces referential integrity at the database level
- EF Core can map them to typed entities or enums (compile-time safety)
- Type-specific columns (e.g., `OrderStatuses.AllowsCancellation`) live where they belong
- Schema is self-documenting — anyone browsing tables understands the domain

### Approach 2: Generic `config.DropDownLists` table (for trivial UI values)

Use the generic table for lookups that are **purely cosmetic** — values displayed in dropdowns that don't drive any business logic.

**Schema:**

```sql
CREATE TABLE config.DropDownLists (
    DropDownListId    INT IDENTITY(1,1) NOT NULL,
    Category          NVARCHAR(50)  NOT NULL,    -- e.g. 'ContactMethod', 'TShirtSize'
    Code              NVARCHAR(50)  NOT NULL,    -- e.g. 'EMAIL', 'MEDIUM'
    DisplayValue      NVARCHAR(200) NOT NULL,    -- e.g. 'Email', 'Medium (M)'
    SortOrder         INT           NOT NULL,
    IsActive          BIT           NOT NULL,
    CreatedDate       DATETIME2     NOT NULL,
    CreatedBy         INT           NOT NULL,
    ModifiedDate      DATETIME2     NULL,
    ModifiedBy        INT           NULL,

    CONSTRAINT PK_DropDownLists PRIMARY KEY (DropDownListId),
    CONSTRAINT UQ_DropDownLists_Category_Code UNIQUE (Category, Code),
    CONSTRAINT DF_DropDownLists_IsActive DEFAULT (1) FOR IsActive,
    CONSTRAINT DF_DropDownLists_SortOrder DEFAULT (0) FOR SortOrder
);

CREATE INDEX IX_DropDownLists_Category ON config.DropDownLists (Category, SortOrder)
    WHERE IsActive = 1;
```

**Examples of values that belong here:**

| Category | Sample values |
|---|---|
| `ContactMethod` | Email, Phone, Mail, Text |
| `TShirtSize` | XS, S, M, L, XL, XXL |
| `HearAboutUs` | Google, Friend, Advertisement, Other |
| `PreferredContactTime` | Morning, Afternoon, Evening |
| `SubscriptionInterest` | Newsletter, Product Updates, Promotions |

**Why a generic table works here:**
- One admin UI manages all of them — pick category, edit values, save
- No EF entity or migration needed for each new dropdown category
- Business users can add new dropdown options without developer involvement
- Values genuinely don't have logic tied to them — typo on `'EMAIL'` vs `'EMIAL'` doesn't break anything critical

### What does NOT belong in `config.DropDownLists`

> ⚠️ **The generic table has strong gravitational pull.** Resist the temptation to add business-meaningful lookups "just to save time creating a table."

Never put these in `DropDownLists`:

- ❌ `OrderStatus` — drives workflow, needs FK from Orders
- ❌ `PaymentStatus` — drives payment logic
- ❌ `UserRole` — drives permissions
- ❌ `Country` — referenced by addresses, billing, shipping
- ❌ `Currency` — referenced by transactions, prices
- ❌ Anything you'd write a `WHERE` clause against in business logic
- ❌ Anything that needs additional columns beyond Code + DisplayValue

If you find yourself wanting to add a new column to `DropDownLists` for a specific category, that's a strong signal the category should be promoted to its own table.

### Promoting a value from `DropDownLists` to its own table

If a lookup outgrows the generic table (gains business logic, needs FK relationships, requires extra columns), promote it:

1. Create the dedicated table with proper PK, columns, and constraints
2. Migrate existing values from `DropDownLists` (preserve IDs if anything references them)
3. Update application code to reference the new table
4. Remove rows for that category from `DropDownLists`

Plan for this. It's normal for the line between "trivial" and "business-meaningful" to shift as the product evolves.

### C# / EF Core usage

**Dedicated lookup tables** map to entities or enums:

```csharp
public class OrderStatus
{
    public int OrderStatusId { get; set; }
    public string Code { get; set; }
    public string DisplayValue { get; set; }
    public bool AllowsCancellation { get; set; }
}

public class Order
{
    public int OrderId { get; set; }
    public int OrderStatusId { get; set; }
    public OrderStatus OrderStatus { get; set; }  // navigation property
}
```

**`config.DropDownLists`** maps to a single repository/service that fetches by category:

```csharp
public interface IDropDownService
{
    Task<IReadOnlyList<DropDownItem>> GetByCategoryAsync(string category);
}

// Usage in a controller or service
var sizes = await _dropDowns.GetByCategoryAsync("TShirtSize");
```

Cache the results — these values rarely change and are read constantly.

### Summary

| Question | Use dedicated table | Use `DropDownLists` |
|---|---|---|
| Will code reference specific values? | ✅ Yes | ❌ No |
| Need FK from another table? | ✅ Yes | ❌ No |
| Has type-specific columns? | ✅ Yes | ❌ No |
| Drives business/workflow logic? | ✅ Yes | ❌ No |
| Pure UI dropdown, admin-managed? | ❌ No | ✅ Yes |

**Default to dedicated tables when in doubt.** The cost of an extra table is small. The cost of moving business logic out of `DropDownLists` later is large.

## Columns

### Naming

- **PascalCase, singular.**
- Be explicit: `EmailAddress` reads better than `Email`, `PhoneNumber` better than `Phone`.

### Primary keys

**Standard for OrderHub: `<Entity>Id`**

```sql
CREATE TABLE Users (
    UserId       INT IDENTITY(1,1) NOT NULL,
    FirstName    NVARCHAR(100) NOT NULL,
    ...
    CONSTRAINT PK_Users PRIMARY KEY (UserId)
);

CREATE TABLE Orders (
    OrderId      INT IDENTITY(1,1) NOT NULL,
    CustomerId   INT NOT NULL,  -- FK to Customers.CustomerId
    ...
    CONSTRAINT PK_Orders PRIMARY KEY (OrderId),
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerId)
        REFERENCES Customers(CustomerId)
);
```

### Why `<Entity>Id` over plain `Id`

- **Self-documenting in joins.** No ambiguity about which `Id` you're referencing:
  ```sql
  SELECT u.UserId, o.OrderId
  FROM Users u
  INNER JOIN Orders o ON o.CustomerId = u.UserId;
  ```
  vs. the `Id`-only style:
  ```sql
  SELECT u.Id, o.Id  -- which Id is which?
  FROM Users u
  INNER JOIN Orders o ON o.CustomerId = u.Id;
  ```
- **Cleaner ad-hoc SQL.** When DBAs or developers write troubleshooting queries, they don't need to alias every table to disambiguate columns.
- **Foreign key naming becomes consistent.** The PK column and the FK column both use the same name (`UserId`), making the relationship obvious.
- **Common in enterprise SQL Server codebases**, including a lot of existing Microsoft documentation.

### EF Core configuration

Since EF Core's default is plain `Id`, you'll need to either:

**Option 1 (recommended): Use `[Key]` attribute on each entity**
```csharp
public class User
{
    [Key]
    public int UserId { get; set; }
    public string FirstName { get; set; }
}
```

**Option 2: Configure globally via convention**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        var entityName = entityType.ClrType.Name;
        var keyProperty = entityType.FindProperty($"{entityName}Id");
        if (keyProperty != null)
        {
            entityType.SetPrimaryKey(keyProperty);
        }
    }
}
```

EF Core also recognizes `<EntityName>Id` as a primary key by convention (after plain `Id`), so often **just naming the property `UserId` in the entity class works automatically** — no attribute or configuration needed:

```csharp
public class User
{
    public int UserId { get; set; }  // EF Core treats this as PK by convention
    public string FirstName { get; set; }
}
```

### Foreign keys

Foreign keys reference the parent table's primary key column directly. Since we use `<Entity>Id` for primary keys, the foreign key column has the **same name as the PK it references**:

```sql
CREATE TABLE Orders (
    OrderId            INT IDENTITY(1,1) NOT NULL,
    CustomerId         INT NOT NULL,  -- FK to Customers.CustomerId
    ShippingAddressId  INT NULL,      -- FK to Addresses.AddressId
    ...
    CONSTRAINT PK_Orders PRIMARY KEY (OrderId),
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerId)
        REFERENCES Customers(CustomerId),
    CONSTRAINT FK_Orders_Addresses FOREIGN KEY (ShippingAddressId)
        REFERENCES Addresses(AddressId)
);
```

If a table has **two foreign keys to the same table**, prefix with the role:

```sql
CREATE TABLE Messages (
    MessageId     INT IDENTITY(1,1) NOT NULL,
    SenderId      INT NOT NULL,  -- FK to Users.UserId
    RecipientId   INT NOT NULL,  -- FK to Users.UserId
    ...
    CONSTRAINT PK_Messages PRIMARY KEY (MessageId),
    CONSTRAINT FK_Messages_Sender FOREIGN KEY (SenderId)
        REFERENCES Users(UserId),
    CONSTRAINT FK_Messages_Recipient FOREIGN KEY (RecipientId)
        REFERENCES Users(UserId)
);
```

### Boolean columns

- Prefix with `Is`, `Has`, or `Can`: `IsActive`, `IsDeleted`, `HasShipped`, `CanLogin`
- Use `BIT` data type
- Never use `0`/`1` integers when a boolean is meant

### Date/time columns

- **`CreatedDate`** / **`ModifiedDate`** — when the row was created/last changed
- **`CreatedBy`** / **`ModifiedBy`** — who created/changed it (user ID)
- Use **`DATETIME2`** (not `DATETIME` — better precision and range)
- Use **`DATETIMEOFFSET`** if timezone matters
- For date-only values, use **`DATE`** (not `DATETIME2`)

### Money/decimal columns

- Use `DECIMAL(precision, scale)` — never `FLOAT` or `REAL` for currency
- Standard money columns: `DECIMAL(19, 4)` (handles up to ~$1 quadrillion with 4 decimal places)
- Always include a unit indicator if multiple currencies are possible: `PriceUsd`, `AmountEur`

### String columns

- **`NVARCHAR`** for any user-facing text (Unicode support) — never `VARCHAR` for names, addresses, descriptions
- **`VARCHAR`** is acceptable for ASCII-only fields like ISO codes, hex strings, internal identifiers
- Always specify a length — never `NVARCHAR(MAX)` unless truly unbounded
- Common lengths: `NVARCHAR(50)` for short codes, `NVARCHAR(100)` for names, `NVARCHAR(255)` for emails/URLs, `NVARCHAR(2000)` for descriptions

### Avoid

- ❌ Reserved words: `User`, `Order`, `Date`, `Status` (use `OrderStatus`, `CreatedDate`)
- ❌ Cryptic abbreviations: `Qty` over `Quantity`, `Desc` over `Description`
- ❌ Hungarian notation: `strFirstName`, `intUserId`
- ❌ Mixing case styles: `userId` (camelCase) or `user_id` (snake_case) — stay PascalCase

## Constraints and indexes

### Naming pattern

```
<Type>_<Table>_<Columns>
```

Where `<Type>` is:

| Prefix | Type |
|---|---|
| `PK` | Primary key |
| `FK` | Foreign key |
| `UQ` | Unique constraint |
| `CK` | Check constraint |
| `DF` | Default constraint |
| `IX` | Non-clustered index |
| `UX` | Unique index |
| `CIX` | Clustered index (when explicitly named) |

### Examples

```sql
-- Primary key
CONSTRAINT PK_Users PRIMARY KEY (UserId)

-- Foreign key
CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerId) REFERENCES Customers(CustomerId)

-- Unique constraint
CONSTRAINT UQ_Users_EmailAddress UNIQUE (EmailAddress)

-- Check constraint
CONSTRAINT CK_Products_Price CHECK (Price >= 0)

-- Default constraint
CONSTRAINT DF_Users_CreatedDate DEFAULT (SYSUTCDATETIME()) FOR CreatedDate

-- Non-clustered index
CREATE INDEX IX_Orders_CustomerId ON Orders (CustomerId);

-- Composite index (multiple columns)
CREATE INDEX IX_Orders_CustomerId_OrderDate ON Orders (CustomerId, OrderDate);

-- Filtered index
CREATE INDEX IX_Orders_Active ON Orders (CustomerId)
    WHERE IsDeleted = 0;
```

### Why explicit constraint names matter

If you don't name a constraint, SQL Server auto-generates something like `PK__Users__3214EC07A1B2C3D4`. This is:
- Hard to read in error messages
- Different across environments (dev/stg/prod), making schema diffs noisy
- Painful to drop/alter later

**Always explicitly name every constraint.**

## Stored procedures

### Naming

**`usp_<Verb><Noun>`** — `usp` stands for "user stored procedure" and disambiguates from system procs (which start with `sp_`).

> ⚠️ **Never start a stored proc name with `sp_`** — SQL Server treats `sp_*` as a system procedure and adds lookup overhead. The `usp_` prefix prevents this.

### Verb conventions

| Verb | Meaning |
|---|---|
| `Get` | Retrieve data (read-only) |
| `List` | Retrieve multiple rows (read-only) |
| `Create` | Insert new row(s) |
| `Update` | Modify existing row(s) |
| `Delete` | Remove row(s) |
| `Upsert` | Insert or update (merge) |
| `Search` | Filtered/paginated retrieval |
| `Validate` | Check business rules without modifying data |
| `Process` | Multi-step business operation |

### Examples

```sql
usp_GetUserById                   -- Single row by primary key
usp_GetUserByEmailAddress         -- Single row by alternate key
usp_ListActiveUsers               -- Multiple rows, filtered
usp_SearchOrdersByDateRange       -- Filtered + paginated
usp_CreateOrder                   -- Insert
usp_UpdateUserProfile             -- Update
usp_DeleteOrderItem               -- Delete
usp_UpsertCustomer                -- Insert or update
usp_ProcessMonthlyBilling         -- Multi-step business operation
usp_ValidateOrderEligibility      -- Read-only business check
```

### Schema-qualified procs

For procs that operate on a specific subsystem, prefix or schema-qualify:

```sql
audit.usp_LogUserChange
identity.usp_CreateUser
reporting.usp_GetMonthlySales
```

### Parameter conventions

- Prefix parameters with `@` (SQL Server requirement)
- PascalCase: `@UserId`, `@StartDate`, `@EmailAddress`
- Match the column name when possible: `@UserId` parameter for `UserId` column
- Output parameters: suffix with `Output` for clarity in code, e.g. `@NewOrderIdOutput`

```sql
CREATE PROCEDURE usp_CreateOrder
    @CustomerId       INT,
    @OrderDate        DATETIME2,
    @TotalAmount      DECIMAL(19, 4),
    @NewOrderId       INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO Orders (CustomerId, OrderDate, TotalAmount)
    VALUES (@CustomerId, @OrderDate, @TotalAmount);

    SET @NewOrderId = SCOPE_IDENTITY();
END
```

## Functions

### Scalar functions: `ufn_<VerbNoun>`

Return a single value.

```sql
CREATE FUNCTION ufn_CalculateOrderTotal (@OrderId INT)
RETURNS DECIMAL(19, 4)
AS ...
```

### Table-valued functions: `tvf_<VerbNoun>`

Return a table.

```sql
CREATE FUNCTION tvf_GetUserOrders (@UserId INT)
RETURNS TABLE
AS ...
```

### Why prefix?

Distinguishes functions from stored procs and tables in IntelliSense, code search, and execution plans.

## Views

**`vw_<DescriptiveName>`** — PascalCase, descriptive, often plural.

```sql
vw_ActiveUsers
vw_OrderSummary
vw_MonthlyRevenue
vw_CustomerLifetimeValue
```

### Schema for views

Reporting and aggregate views often live in a `reporting` schema:

```sql
reporting.vw_DailyOrderTotals
reporting.vw_TopCustomersByRevenue
```

## Triggers

**`tr_<Table>_<Timing><Action>`**

| Timing | Action |
|---|---|
| `After` | After the operation completes |
| `Instead` | Replaces the operation |

```sql
tr_Users_AfterInsert
tr_Users_AfterUpdate
tr_Orders_AfterDelete
tr_AuditLog_InsteadOfUpdate
```

> 💡 **Use triggers sparingly.** They're hard to debug and can cause unexpected behavior. Prefer application-layer logic or stored procs for business rules. Triggers are appropriate for cross-cutting concerns like audit logging.

## Sequences

**`sq_<Purpose>`**

```sql
sq_OrderNumber
sq_InvoiceNumber
sq_TicketNumber
```

## SQL formatting standards

### Keyword casing

**UPPERCASE for SQL keywords**, PascalCase for identifiers.

```sql
-- ✅ Good
SELECT u.UserId, u.FirstName, u.LastName
FROM Users AS u
INNER JOIN Orders AS o ON o.CustomerId = u.UserId
WHERE u.IsActive = 1
ORDER BY u.LastName;

-- ❌ Avoid
select u.userid, u.firstname, u.lastname
from users u
join orders o on o.customerid = u.userid
where u.isactive = 1
order by u.lastname;
```

### Aliases

- Use **meaningful, short aliases** with `AS`: `Users AS u`, `Orders AS o`
- Avoid single-letter aliases for long queries — use `cust` instead of `c`, `prod` instead of `p`

### Always use schema prefix

```sql
-- ✅ Good
SELECT * FROM dbo.Users;

-- ⚠️ Avoid (works, but worse for query plan caching)
SELECT * FROM Users;
```

### `SET NOCOUNT ON` in stored procs

Always include at the top of stored procs to suppress row count messages:

```sql
CREATE PROCEDURE usp_DoSomething
AS
BEGIN
    SET NOCOUNT ON;
    -- ...
END
```

## Migrations

OrderHub uses **EF Core migrations** for schema changes. Migration files should be:

- **PascalCase**: `AddUserEmailIndex`, `CreateOrdersTable`, `AddSoftDeleteToProducts`
- **Descriptive verbs**: start with `Add`, `Create`, `Remove`, `Rename`, `Alter`
- **Specific**: `AddIsActiveToUsers` not `UpdateUsers`

```bash
dotnet ef migrations add AddUserEmailAddressIndex
dotnet ef migrations add CreateOrdersTable
dotnet ef migrations add AddSoftDeleteToProducts
```

Migration files are timestamped automatically by EF, so chronological ordering is handled — focus on making the *name* describe the change.

## Standard audit columns

Every business-domain table should include these columns:

```sql
CreatedDate     DATETIME2     NOT NULL CONSTRAINT DF_<Table>_CreatedDate DEFAULT (SYSUTCDATETIME()),
CreatedBy       INT           NOT NULL,  -- FK to Users.UserId
ModifiedDate    DATETIME2     NULL,
ModifiedBy      INT           NULL,      -- FK to Users.UserId
IsDeleted       BIT           NOT NULL CONSTRAINT DF_<Table>_IsDeleted DEFAULT (0)
```

For high-audit tables (financial, identity), add a separate audit table in the `audit` schema rather than tracking changes inline.

## Examples: full table definition

```sql
CREATE TABLE dbo.Users (
    UserId          INT             IDENTITY(1,1) NOT NULL,
    EmailAddress    NVARCHAR(255)   NOT NULL,
    FirstName       NVARCHAR(100)   NOT NULL,
    LastName        NVARCHAR(100)   NOT NULL,
    PhoneNumber     NVARCHAR(20)    NULL,
    IsActive        BIT             NOT NULL,
    LastLoginDate   DATETIME2       NULL,
    CreatedDate     DATETIME2       NOT NULL,
    CreatedBy       INT             NOT NULL,
    ModifiedDate    DATETIME2       NULL,
    ModifiedBy      INT             NULL,
    IsDeleted       BIT             NOT NULL,

    CONSTRAINT PK_Users PRIMARY KEY CLUSTERED (UserId),
    CONSTRAINT UQ_Users_EmailAddress UNIQUE (EmailAddress),
    CONSTRAINT CK_Users_EmailAddress CHECK (EmailAddress LIKE '%_@_%._%'),
    CONSTRAINT DF_Users_IsActive DEFAULT (1) FOR IsActive,
    CONSTRAINT DF_Users_CreatedDate DEFAULT (SYSUTCDATETIME()) FOR CreatedDate,
    CONSTRAINT DF_Users_IsDeleted DEFAULT (0) FOR IsDeleted
);

CREATE INDEX IX_Users_LastName_FirstName ON dbo.Users (LastName, FirstName);
CREATE INDEX IX_Users_IsActive ON dbo.Users (IsActive) WHERE IsDeleted = 0;
```

## Examples: full stored procedure

```sql
CREATE OR ALTER PROCEDURE dbo.usp_GetUserByEmailAddress
    @EmailAddress   NVARCHAR(255)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        u.UserId,
        u.EmailAddress,
        u.FirstName,
        u.LastName,
        u.PhoneNumber,
        u.IsActive,
        u.LastLoginDate,
        u.CreatedDate
    FROM dbo.Users AS u
    WHERE u.EmailAddress = @EmailAddress
      AND u.IsDeleted = 0;
END
```

---

**Last updated:** 2026-04-30
**Owner:** OrderHub Engineering
**Related:** [`AZURE_CONVENTIONS.md`](./AZURE_CONVENTIONS.md) — Azure resource naming conventions
