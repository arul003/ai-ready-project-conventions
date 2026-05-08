# OrderHub — Next.js / Frontend Naming Conventions

This document defines the project structure, naming standards, and code conventions for the OrderHub frontend (`orderhub-web` repository). It mirrors the structure of `CSHARP_CONVENTIONS.md` for the backend, adapted for the Next.js / React ecosystem.

## Stack assumptions

- **Framework:** Next.js 16 with **App Router** (Pages Router is in maintenance mode — don't use it)
- **React:** React 19.2 (bundled with Next.js 16's App Router)
- **Language:** TypeScript (strict mode)
- **Bundler:** Turbopack (stable default in Next.js 16)
- **Styling:** Tailwind CSS
- **Component library:** shadcn/ui (recommended) or Radix UI primitives
- **State management:** TanStack Query (React Query) for server state, React state/context for UI state
- **Forms:** React Hook Form + Zod
- **API client:** Generated TypeScript client from OpenAPI (or hand-written `fetch` wrappers)
- **Auth:** Auth.js (formerly NextAuth.js) or custom JWT integration with the C# API
- **Hosting:** Azure Static Web App (`prod-orderhub-stapp-web`) — see `AZURE_CONVENTIONS.md`
- **Package manager:** pnpm (recommended for speed and disk efficiency)

## Why these choices

- **App Router** — As of 2026, the Pages Router is in maintenance mode. All new Next.js projects should use App Router.
- **Turbopack** — Stable in Next.js 16, ~5–10x faster than Webpack for dev builds. Default; no flag needed.
- **TypeScript strict** — Catches bugs at compile time. Cost of typing is repaid many times over.
- **TanStack Query** — Industry standard for server-state management. Caching, refetching, optimistic updates handled for you.
- **Tailwind + shadcn/ui** — Tailwind for utility styling, shadcn/ui for accessible primitives you copy into your codebase (not a dependency).
- **React Hook Form + Zod** — Same Zod schemas can be used on both client and (optionally) server, eliminating duplicate validation logic.

## Repository layout

```
orderhub-web/                              ← repo (lowercase-kebab)
├── .github/
│   └── workflows/                         ← GitHub Actions CI/CD
├── public/                                ← Static assets (images, fonts, favicon)
├── src/
│   ├── app/                               ← App Router (routes, layouts, pages)
│   ├── components/                        ← Reusable React components
│   ├── features/                          ← Feature-based organization (mirrors backend slices)
│   ├── lib/                               ← Utilities, helpers, configs
│   ├── hooks/                             ← Shared custom React hooks
│   ├── types/                             ← Shared TypeScript types
│   └── styles/                            ← Global CSS
├── tests/                                 ← E2E tests (Playwright)
├── .env.example                           ← Template for environment variables
├── .env.local                             ← Local secrets (gitignored)
├── next.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── package.json
└── README.md
```

The `src/` directory is the convention — keeps build artifacts and configs at the root, app code separated.

## App Router structure

The App Router uses **file-system-based routing**: folder names become URL paths, special files (`page.tsx`, `layout.tsx`) define behavior at each level.

### Folder structure within `src/app/`

```
src/app/
├── layout.tsx                             ← Root layout (wraps every page)
├── page.tsx                               ← / (home page)
├── globals.css                            ← Global styles imported in root layout
├── loading.tsx                            ← Global loading UI
├── error.tsx                              ← Global error boundary
├── not-found.tsx                          ← 404 page
│
├── (auth)/                                ← Route group: auth pages share a layout
│   ├── layout.tsx                         ← Auth layout (centered, no nav)
│   ├── login/
│   │   └── page.tsx                       ← /login
│   ├── register/
│   │   └── page.tsx                       ← /register
│   └── forgot-password/
│       └── page.tsx                       ← /forgot-password
│
├── (app)/                                 ← Route group: authenticated pages
│   ├── layout.tsx                         ← App layout (with sidebar, header)
│   ├── dashboard/
│   │   └── page.tsx                       ← /dashboard
│   ├── orders/
│   │   ├── page.tsx                       ← /orders (list)
│   │   ├── new/
│   │   │   └── page.tsx                   ← /orders/new (create form)
│   │   └── [orderId]/
│   │       ├── page.tsx                   ← /orders/123 (detail)
│   │       └── edit/
│   │           └── page.tsx               ← /orders/123/edit
│   ├── users/
│   └── settings/
│
└── api/                                   ← Route handlers (serverless functions)
    └── webhook/
        └── route.ts                       ← /api/webhook (POST handler)
```

### Special file naming

Next.js App Router has reserved file names with specific meanings. Use them exactly as-is (lowercase, no exceptions):

| File | Purpose |
|---|---|
| `page.tsx` | The route's main UI (makes the folder publicly accessible) |
| `layout.tsx` | Wraps `page.tsx` and child routes; persists across navigation |
| `loading.tsx` | Loading UI shown while page streams in |
| `error.tsx` | Error boundary for this route segment |
| `not-found.tsx` | 404 UI for this segment |
| `template.tsx` | Like layout but re-mounts on navigation |
| `route.ts` | API route handler (in `app/api/...`) |
| `default.tsx` | Fallback for parallel routes |
| `middleware.ts` | Edge middleware (lives at `src/` or root, not inside `app/`) |

### Folder naming for routes

| Convention | Folder | Resulting URL |
|---|---|---|
| Static segment | `kebab-case` | `/order-items` |
| Dynamic segment | `[paramName]` (camelCase inside brackets) | `/orders/[orderId]` → `/orders/123` |
| Catch-all | `[...slug]` | `/docs/[...slug]` → `/docs/a/b/c` |
| Optional catch-all | `[[...slug]]` | matches with or without slug |
| Route group (no URL effect) | `(groupName)` | `(auth)`, `(app)` — for grouping without affecting URLs |
| Private folder (excluded from routing) | `_folderName` | `_components`, `_utils` |

**URL paths use kebab-case.** This matches our API URL convention from `API_CONVENTIONS.md` — `/order-items`, `/payment-methods`, `/forgot-password`.

## Component organization

Two valid approaches for organizing React components — pick one and stick to it.

### Recommended: Feature-based with shared components

```
src/
├── components/                            ← Truly reusable, app-wide components
│   ├── ui/                                ← shadcn/ui primitives
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── dialog.tsx
│   │   └── ...
│   ├── layout/                            ← Page-level layout components
│   │   ├── Sidebar.tsx
│   │   ├── Header.tsx
│   │   └── PageContainer.tsx
│   └── shared/                            ← Reusable composite components
│       ├── DataTable.tsx
│       ├── PageHeader.tsx
│       └── EmptyState.tsx
│
├── features/                              ← Feature-specific components/logic
│   ├── orders/
│   │   ├── components/
│   │   │   ├── OrderList.tsx
│   │   │   ├── OrderCard.tsx
│   │   │   ├── OrderDetailView.tsx
│   │   │   └── CreateOrderForm.tsx
│   │   ├── hooks/
│   │   │   ├── useOrders.ts
│   │   │   ├── useOrder.ts
│   │   │   └── useCreateOrder.ts
│   │   ├── api/
│   │   │   └── orders.api.ts              ← TanStack Query functions
│   │   ├── schemas/
│   │   │   └── order.schema.ts            ← Zod validation schemas
│   │   └── types.ts                       ← TypeScript types for this feature
│   ├── users/
│   ├── products/
│   └── auth/
```

**Rule of thumb:** Components used by **only one feature** live in that feature's folder. Components used by **multiple features** live in `src/components/`.

This mirrors the backend's vertical slice organization — features are the unit of organization on both sides.

## File and component naming

### Components: PascalCase

```typescript
// File: src/features/orders/components/OrderCard.tsx
export function OrderCard({ order }: OrderCardProps) {
  return <div>...</div>;
}
```

- **File name matches component name**: `OrderCard.tsx` exports `OrderCard`
- **PascalCase** for both the file and the component
- One main component per file (small helper components can share the file)

### Hooks: camelCase, prefixed with `use`

```typescript
// File: src/features/orders/hooks/useOrders.ts
export function useOrders() { ... }
export function useOrder(orderId: number) { ... }
export function useCreateOrder() { ... }
```

- **`use` prefix is mandatory** — React enforces this and lint rules check it
- **camelCase**, descriptive
- File name typically matches the primary export

### Utility / library files: camelCase

```
src/lib/api-client.ts
src/lib/format-currency.ts
src/lib/date-utils.ts
```

Or **kebab-case** if you prefer (also common in the JS ecosystem). Pick one and be consistent. **Recommendation: kebab-case** for utility files (matches Node.js conventions, easier to scan).

### Pages and layouts: lowercase (Next.js requirement)

`page.tsx`, `layout.tsx`, `loading.tsx` etc. are **always lowercase** — Next.js requires this. The folder names around them follow the route-naming conventions above.

### TypeScript type files

```
src/features/orders/types.ts
src/types/api.ts
src/types/global.d.ts
```

- camelCase or kebab-case (be consistent — recommendation: `kebab-case`)
- `.d.ts` for ambient declarations (rarely needed)

### CSS/styling files

```
src/app/globals.css
src/styles/animations.css
```

- kebab-case
- For Tailwind, most styling lives in component files via `className`; standalone CSS is for true globals

## Component conventions

### Function components only

```typescript
// ✅ Preferred — function component
export function OrderCard({ order }: OrderCardProps) {
  return <div>...</div>;
}

// ❌ Avoid — class components are legacy
export class OrderCard extends React.Component { ... }
```

### Named exports, not default exports

```typescript
// ✅ Good — named export
export function OrderCard({ order }: OrderCardProps) { ... }

// Import:
import { OrderCard } from '@/features/orders/components/OrderCard';

// ❌ Avoid — default export
export default function OrderCard() { ... }
```

**Why named exports:**
- Refactoring tools rename consistently across the codebase
- Imports show the actual component name (no aliasing)
- Easier to grep for usages
- Better autocomplete in IDEs

**Exception:** Next.js requires default exports for `page.tsx`, `layout.tsx`, `error.tsx`, `loading.tsx`, etc. Use default exports there because the framework requires it.

### Props interface naming

```typescript
interface OrderCardProps {
  order: Order;
  onSelect?: (orderId: number) => void;
}

export function OrderCard({ order, onSelect }: OrderCardProps) {
  return <div>...</div>;
}
```

- **`<ComponentName>Props`** suffix
- `interface` for props (extensible) over `type` (more common convention)
- Optional props use `?`
- Event handler props prefixed with `on`: `onSelect`, `onSubmit`, `onCancel`

### Server Components vs. Client Components

Next.js App Router uses **React Server Components by default**. Add `"use client"` only when the component needs interactivity (event handlers, hooks, browser APIs).

```typescript
// ✅ Server Component (default — runs on server, zero JS sent to client)
// File: src/app/(app)/orders/page.tsx
import { getOrders } from '@/features/orders/api/orders.api';
import { OrderList } from '@/features/orders/components/OrderList';

export default async function OrdersPage() {
  const orders = await getOrders();  // runs on server
  return <OrderList orders={orders} />;
}
```

```typescript
// ✅ Client Component (only when needed)
// File: src/features/orders/components/CreateOrderForm.tsx
"use client";

import { useState } from 'react';

export function CreateOrderForm() {
  const [items, setItems] = useState([]);  // hooks need client component
  return <form>...</form>;
}
```

**Rule of thumb:**
- **Default to Server Components** — they're faster, send less JS, run secure code on the server
- **Only mark `"use client"` when needed** — interactivity, state, effects, browser APIs
- **Push `"use client"` as deep into the tree as possible** — keep parents as Server Components

### Component file structure

Standard order within a component file (top to bottom):

```typescript
// 1. "use client" directive (if applicable)
"use client";

// 2. Imports
import { useState } from 'react';
import { Button } from '@/components/ui/button';
import type { Order } from '@/features/orders/types';

// 3. Type/interface definitions
interface OrderCardProps {
  order: Order;
  onSelect?: (orderId: number) => void;
}

// 4. Constants (if any)
const MAX_ITEMS_DISPLAY = 5;

// 5. The component
export function OrderCard({ order, onSelect }: OrderCardProps) {
  // 5a. Hooks (in order: state, derived values, effects, callbacks)
  const [isExpanded, setIsExpanded] = useState(false);

  // 5b. Event handlers
  function handleClick() {
    onSelect?.(order.orderId);
  }

  // 5c. Render
  return (
    <div onClick={handleClick}>
      ...
    </div>
  );
}

// 6. Helper components (if small and only used here)
function OrderStatusBadge({ status }: { status: string }) { ... }
```

## TypeScript conventions

### Strict mode

`tsconfig.json` should have all strict flags enabled:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "preserve",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### Type vs. interface

- **`interface`** for object shapes that may be extended (props, public APIs)
- **`type`** for unions, intersections, mapped types, or anywhere `interface` doesn't fit

```typescript
// interface — extensible
interface OrderCardProps {
  order: Order;
}

// type — for unions
type OrderStatus = 'pending' | 'paid' | 'shipped' | 'cancelled';
```

### Naming conventions

| Element | Convention | Example |
|---|---|---|
| Type / interface | PascalCase | `Order`, `OrderCardProps` |
| Type alias for union | PascalCase | `OrderStatus` |
| Function | camelCase | `formatCurrency`, `getOrders` |
| Component | PascalCase | `OrderCard`, `CreateOrderForm` |
| Hook | camelCase, `use` prefix | `useOrders`, `useDebounce` |
| Constant | UPPER_SNAKE_CASE for module-level, camelCase for local | `MAX_PAGE_SIZE`, `defaultPageSize` |
| Variable | camelCase | `orderItems`, `isLoading` |
| Boolean | `is`/`has`/`can` prefix | `isLoading`, `hasError`, `canEdit` |
| Generic type parameter | `T` or `TName` | `T`, `TData`, `TError` |

### Avoid `any` — use `unknown` instead

```typescript
// ❌ Bad — bypasses type checking entirely
function processData(data: any) { ... }

// ✅ Good — forces you to narrow the type
function processData(data: unknown) {
  if (typeof data === 'object' && data !== null) {
    // narrowed
  }
}
```

If you genuinely don't know the type and can't narrow it, use `unknown`. `any` should be a last resort with a comment explaining why.

## Data fetching and API integration

OrderHub's frontend talks to the C# API at `/api/v1/...`. Two layers handle this cleanly:

1. **API client functions** — thin `fetch` wrappers that call the API and return typed responses
2. **TanStack Query hooks** — cache, refetch, mutate state from those API functions

### API client structure

```typescript
// File: src/features/orders/api/orders.api.ts
import { apiClient } from '@/lib/api-client';
import type { Order, CreateOrderRequest, PagedResult } from '@/features/orders/types';

export async function getOrders(params: ListOrdersParams): Promise<PagedResult<Order>> {
  return apiClient.get('/api/v1/orders', { params });
}

export async function getOrder(orderId: number): Promise<Order> {
  return apiClient.get(`/api/v1/orders/${orderId}`);
}

export async function createOrder(request: CreateOrderRequest): Promise<Order> {
  return apiClient.post('/api/v1/orders', request);
}

export async function cancelOrder(orderId: number): Promise<void> {
  return apiClient.post(`/api/v1/orders/${orderId}/cancellation`);
}
```

API functions live in `<feature>/api/<feature>.api.ts`. Each function maps to one endpoint.

### TanStack Query hooks

```typescript
// File: src/features/orders/hooks/useOrders.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { getOrders, getOrder, createOrder, cancelOrder } from '../api/orders.api';

export function useOrders(params: ListOrdersParams) {
  return useQuery({
    queryKey: ['orders', params],
    queryFn: () => getOrders(params),
  });
}

export function useOrder(orderId: number) {
  return useQuery({
    queryKey: ['orders', orderId],
    queryFn: () => getOrder(orderId),
    enabled: !!orderId,
  });
}

export function useCreateOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createOrder,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['orders'] });
    },
  });
}
```

### Query key conventions

Query keys are the cache identity in TanStack Query. Use a consistent structure:

```typescript
// ✅ Hierarchical, predictable
['orders']                          // List of all orders
['orders', { status: 'pending' }]   // Filtered list
['orders', orderId]                 // Single order
['orders', orderId, 'items']        // Sub-resource

// ❌ Inconsistent, hard to invalidate
['allOrders']
['order-detail-123']
```

The hierarchy lets you invalidate broadly (`['orders']` invalidates everything order-related) or narrowly (`['orders', 123]` invalidates just one order).

## Form handling

Use **React Hook Form + Zod** for forms. Same Zod schemas can validate on both client and (optionally) server.

```typescript
// File: src/features/orders/schemas/order.schema.ts
import { z } from 'zod';

export const createOrderSchema = z.object({
  customerId: z.number().positive(),
  items: z.array(
    z.object({
      productId: z.number().positive(),
      quantity: z.number().positive(),
    })
  ).min(1, 'Order must have at least one item'),
  shippingAddress: z.string().min(1).max(500),
});

export type CreateOrderRequest = z.infer<typeof createOrderSchema>;
```

```typescript
// File: src/features/orders/components/CreateOrderForm.tsx
"use client";

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createOrderSchema, type CreateOrderRequest } from '../schemas/order.schema';
import { useCreateOrder } from '../hooks/useOrders';

export function CreateOrderForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<CreateOrderRequest>({
    resolver: zodResolver(createOrderSchema),
  });

  const createOrder = useCreateOrder();

  function onSubmit(data: CreateOrderRequest) {
    createOrder.mutate(data);
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      ...
    </form>
  );
}
```

The Zod schema serves as the single source of truth for the form's shape, validation rules, and TypeScript type.

## Path aliases

Configure `@/*` to point to `src/*` in `tsconfig.json` (already shown above). Use it everywhere instead of relative imports:

```typescript
// ✅ Good — clean, refactorable
import { Button } from '@/components/ui/button';
import { useOrders } from '@/features/orders/hooks/useOrders';
import type { Order } from '@/features/orders/types';

// ❌ Avoid — fragile, breaks on file moves
import { Button } from '../../../components/ui/button';
```

## Environment variables

### Naming

- **`NEXT_PUBLIC_*`** — exposed to the browser (build-time)
- **No prefix** — server-only (runtime, route handlers, Server Components)

```env
# .env.example (committed — template only, no real values)
NEXT_PUBLIC_API_BASE_URL=https://api.orderhub.com
NEXT_PUBLIC_APPLICATION_INSIGHTS_CONNECTION_STRING=

# Server-only — never sent to browser
AUTH_SECRET=
DATABASE_URL=
```

### Strongly-typed env access

Don't use `process.env.X` directly throughout the codebase — typo-prone and lacks type safety. Wrap it once:

```typescript
// File: src/lib/env.ts
import { z } from 'zod';

const envSchema = z.object({
  NEXT_PUBLIC_API_BASE_URL: z.string().url(),
  NEXT_PUBLIC_APPLICATION_INSIGHTS_CONNECTION_STRING: z.string().optional(),
  AUTH_SECRET: z.string().min(32).optional(),  // server-only, may be undefined client-side
});

export const env = envSchema.parse({
  NEXT_PUBLIC_API_BASE_URL: process.env.NEXT_PUBLIC_API_BASE_URL,
  NEXT_PUBLIC_APPLICATION_INSIGHTS_CONNECTION_STRING: process.env.NEXT_PUBLIC_APPLICATION_INSIGHTS_CONNECTION_STRING,
  AUTH_SECRET: process.env.AUTH_SECRET,
});
```

Now use `env.NEXT_PUBLIC_API_BASE_URL` everywhere — fully typed, validated at startup.

### Secrets

- **Local development:** `.env.local` (gitignored)
- **Production / staging:** Set via Azure Static Web App configuration → Application settings
- **CI/CD:** GitHub Actions secrets

## Styling conventions

### Tailwind utility classes

```tsx
// ✅ Good — utility-first, no separate CSS file
<button className="rounded-md bg-blue-600 px-4 py-2 text-white hover:bg-blue-700">
  Save
</button>

// ❌ Avoid mixing utility classes with inline styles unless needed
<button style={{ borderRadius: '6px' }} className="bg-blue-600">Save</button>
```

### Component composition over deeply nested classNames

When `className` strings get long, extract to a variant utility:

```typescript
import { cva } from 'class-variance-authority';

const buttonVariants = cva('rounded-md font-medium transition-colors', {
  variants: {
    variant: {
      primary: 'bg-blue-600 text-white hover:bg-blue-700',
      secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300',
      danger: 'bg-red-600 text-white hover:bg-red-700',
    },
    size: {
      sm: 'px-3 py-1.5 text-sm',
      md: 'px-4 py-2 text-base',
      lg: 'px-6 py-3 text-lg',
    },
  },
  defaultVariants: { variant: 'primary', size: 'md' },
});
```

This is how shadcn/ui works internally — copy components into your codebase, use `cva` for variants.

### Class merging

Use `clsx` + `tailwind-merge` (or the combined `cn` utility from shadcn) for conditional classes:

```typescript
import { cn } from '@/lib/utils';

<button className={cn('px-4 py-2', isActive && 'bg-blue-600', className)}>
  ...
</button>
```

`cn()` deduplicates conflicting Tailwind classes (e.g., `px-2 px-4` becomes `px-4`).

## Testing

### Unit tests: Vitest + React Testing Library

```
tests/                                 ← OR co-locate with components
└── orders.test.tsx
```

Recommended: **co-locate tests with components** (`OrderCard.test.tsx` next to `OrderCard.tsx`). Easier to find, easier to keep in sync.

### E2E tests: Playwright

```
tests/e2e/
├── auth.spec.ts
├── orders.spec.ts
└── playwright.config.ts
```

E2E tests live in a separate `tests/e2e/` folder and run against deployed environments.

### Test naming

```typescript
describe('OrderCard', () => {
  it('displays the order ID and total', () => { ... });
  it('calls onSelect when clicked', () => { ... });
  it('shows the cancellation badge for cancelled orders', () => { ... });
});
```

## Code style enforcement

### ESLint + Prettier

```json
// .eslintrc.json
{
  "extends": [
    "next/core-web-vitals",
    "next/typescript",
    "prettier"
  ]
}
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5",
  "printWidth": 100,
  "tabWidth": 2,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

The `prettier-plugin-tailwindcss` plugin auto-sorts Tailwind classes into the recommended order — eliminates "where do I put this class" debates in code review.

### Pre-commit hooks

Use **Husky + lint-staged** to run linting and formatting before every commit:

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

This catches style violations before they hit the repo.

## Quick reference cheat sheet

| Element | Convention | Example |
|---|---|---|
| Repo | lowercase-kebab | `orderhub-web` |
| Component file | PascalCase, `.tsx` | `OrderCard.tsx` |
| Component | PascalCase, named export | `export function OrderCard()` |
| Hook file | camelCase, `.ts` | `useOrders.ts` |
| Hook | camelCase, `use` prefix | `useOrders`, `useCreateOrder` |
| Utility file | kebab-case, `.ts` | `format-currency.ts` |
| API client file | `<feature>.api.ts` | `orders.api.ts` |
| Schema file | `<entity>.schema.ts` | `order.schema.ts` |
| Type | PascalCase | `Order`, `OrderCardProps` |
| Function | camelCase | `formatCurrency` |
| Constant | UPPER_SNAKE_CASE | `MAX_PAGE_SIZE` |
| Variable | camelCase | `orderItems`, `isLoading` |
| Boolean | `is`/`has`/`can` prefix | `isLoading`, `hasError` |
| Route folder (static) | kebab-case | `app/order-items/` → `/order-items` |
| Route folder (dynamic) | `[paramName]` | `app/orders/[orderId]/` |
| Route group | `(groupName)` | `app/(auth)/`, `app/(app)/` |
| Special files | lowercase | `page.tsx`, `layout.tsx`, `loading.tsx` |
| Env (public) | `NEXT_PUBLIC_*` | `NEXT_PUBLIC_API_BASE_URL` |
| Env (server-only) | no prefix, UPPER_SNAKE_CASE | `AUTH_SECRET` |
| Path alias | `@/*` → `src/*` | `import { ... } from '@/features/...'` |

---

**Last updated:** 2026-04-30
**Owner:** OrderHub Engineering
**Related:**
- [`AZURE_CONVENTIONS.md`](./AZURE_CONVENTIONS.md) — Azure resource naming
- [`DATABASE_CONVENTIONS.md`](./DATABASE_CONVENTIONS.md) — Database object naming
- [`API_CONVENTIONS.md`](./API_CONVENTIONS.md) — REST API conventions
- [`CSHARP_CONVENTIONS.md`](./CSHARP_CONVENTIONS.md) — C# / .NET conventions
