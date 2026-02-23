# API Guide

This is the consolidated guide for API development using Effect HttpApi for endpoint definitions and Effect Atom (AtomHttpApi) for frontend data loading. This follows the same patterns as the accountability project.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Defining HTTP Endpoints (Backend)](#defining-http-endpoints-backend)
3. [Request/Response Schemas](#requestresponse-schemas)
4. [Implementing Handlers](#implementing-handlers)
5. [Authentication & Middleware](#authentication--middleware)
6. [Frontend Client (AtomHttpApi)](#frontend-client-atomhttpapi)
7. [Data Loading Patterns](#data-loading-patterns)
8. [Mutations & Cache Invalidation](#mutations--cache-invalidation)
9. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)

---

## Architecture Overview

```
Frontend (Effect Atom + AtomHttpApi) → HTTP API (Effect HttpApi) → Convex Backend
```

**Backend:**
- `HttpApi` - Define typed API contracts using Schema
- `HttpApiGroup` - Group related endpoints
- `HttpApiEndpoint` - Individual endpoint definitions with typed path, query, body, response, and errors
- Auto-generates OpenAPI spec

**Frontend:**
- `AtomHttpApi.Tag` - Create typed HTTP API client from the same HttpApi definition
- `useAtomValue` - Subscribe to query results (returns `Result<A, E>`)
- `useAtomSet` - Trigger mutations
- `Result.match` - Handle loading/success/failure states
- No code generation needed - schemas are shared directly

---

## Defining HTTP Endpoints (Backend)

### Use Domain Schemas Directly

HttpApi automatically decodes path params, URL params, headers, and payloads:

```typescript
import { HttpApiEndpoint, HttpApiGroup, HttpApi, HttpApiSchema } from "@effect/platform"
import { Schema } from "effect"

// Define endpoints
const getItem = HttpApiEndpoint.get("getItem", "/items/:id")
  .setPath(Schema.Struct({ id: ItemId }))  // Branded type - auto decoded
  .addSuccess(Item)
  .addError(ItemNotFoundError)

const createItem = HttpApiEndpoint.post("createItem", "/items")
  .setPayload(CreateItemRequest)
  .addSuccess(Item, { status: 201 })
  .addError(ValidationError)

const listItems = HttpApiEndpoint.get("listItems", "/items")
  .setUrlParams(Schema.Struct({
    categoryId: Schema.optional(CategoryId),
    page: Schema.optional(Schema.NumberFromString),
    limit: Schema.optional(Schema.NumberFromString)
  }))
  .addSuccess(Schema.Array(Item))

const updateItem = HttpApiEndpoint.patch("updateItem", "/items/:id")
  .setPath(Schema.Struct({ id: ItemId }))
  .setPayload(UpdateItemRequest)
  .addSuccess(Item)
  .addError(ItemNotFoundError)
  .addError(ValidationError)

const deleteItem = HttpApiEndpoint.del("deleteItem", "/items/:id")
  .setPath(Schema.Struct({ id: ItemId }))
  .addSuccess(Schema.Void, { status: 204 })
  .addError(ItemNotFoundError)
```

### Creating API Groups

```typescript
// Group related endpoints
class ItemsApi extends HttpApiGroup.make("items")
  .add(getItem)
  .add(createItem)
  .add(listItems)
  .add(updateItem)
  .add(deleteItem)
  .prefix("/v1")
  .addError(UnauthorizedError, { status: 401 })
  .middleware(AuthMiddleware)
{}

// Compose into full API
class AppApi extends HttpApi.make("app")
  .add(ItemsApi)
  .add(CategoriesApi)
  .add(UsersApi)
  .prefix("/api")
  .addError(InternalServerError, { status: 500 })
{}
```

---

## Request/Response Schemas

### Path Parameters

Path parameters are strings. Use branded string schemas:

```typescript
export const ItemId = Schema.NonEmptyTrimmedString.pipe(
  Schema.brand("ItemId")
)

const getItem = HttpApiEndpoint.get("getItem", "/items/:id")
  .setPath(Schema.Struct({ id: ItemId }))
```

### URL Query Parameters

Use transforming schemas for non-string types:

```typescript
export const ItemListParams = Schema.Struct({
  categoryId: Schema.optional(CategoryId),
  isActive: Schema.optional(Schema.BooleanFromString),
  limit: Schema.optional(Schema.NumberFromString.pipe(
    Schema.int(),
    Schema.greaterThan(0)
  ))
})
```

### Request Bodies

Use domain schemas directly for JSON payloads:

```typescript
export class CreateItemRequest extends Schema.Class<CreateItemRequest>("CreateItemRequest")({
  name: Schema.NonEmptyTrimmedString,
  categoryId: CategoryId,
  parentId: Schema.OptionFromNullOr(ItemId)
}) {}
```

### Response Bodies

Return domain entities directly:

```typescript
const getItem = HttpApiEndpoint.get("getItem", "/items/:id")
  .addSuccess(Item)  // Domain entity as response

// For lists with metadata
export class ItemListResponse extends Schema.Class<ItemListResponse>("ItemListResponse")({
  items: Schema.Array(Item),
  total: Schema.Number,
  limit: Schema.Number,
  offset: Schema.Number
}) {}
```

### Error Responses

Annotate error schemas with HTTP status codes:

```typescript
class ItemNotFoundError extends Schema.TaggedError<ItemNotFoundError>()(
  "ItemNotFoundError",
  {
    itemId: ItemId,
    message: Schema.propertySignature(Schema.String).pipe(
      Schema.withConstructorDefault(() => "Item not found")
    )
  },
  HttpApiSchema.annotations({ status: 404 })
) {}

class ValidationError extends Schema.TaggedError<ValidationError>()(
  "ValidationError",
  { errors: Schema.Array(Schema.String) },
  HttpApiSchema.annotations({ status: 422 })
) {}

// Empty errors (status only, no body)
class UnauthorizedError extends HttpApiSchema.EmptyError<UnauthorizedError>()({
  tag: "UnauthorizedError",
  status: 401
}) {}
```

---

## Implementing Handlers

```typescript
const ItemsLive = HttpApiBuilder.group(
  AppApi,
  "items",
  (handlers) =>
    handlers
      .handle("getItem", ({ path }) =>
        Effect.gen(function* () {
          const item = yield* convex.query("items:getById", { id: path.id })
          return yield* Option.match(item, {
            onNone: () => Effect.fail(new ItemNotFoundError({ itemId: path.id })),
            onSome: Effect.succeed
          })
        })
      )
      .handle("createItem", ({ payload }) =>
        Effect.gen(function* () {
          return yield* convex.mutation("items:create", payload)
        })
      )
      .handle("listItems", ({ urlParams }) =>
        Effect.gen(function* () {
          return yield* convex.query("items:list", {
            categoryId: urlParams.categoryId,
            page: urlParams.page ?? 1,
            limit: urlParams.limit ?? 20
          })
        })
      )
)
```

---

## Authentication & Middleware

**This project uses [Clerk](https://clerk.com/) for authentication.** Clerk handles user sign-up, sign-in, session management, and user profiles. It integrates with Convex via the official Clerk + Convex integration.

### How It Works

1. **Frontend**: Clerk's `<ClerkProvider>` wraps the app. Clerk provides pre-built `<SignIn>`, `<SignUp>`, `<UserButton>` components and hooks like `useAuth()`, `useUser()`.
2. **Convex Integration**: Use `<ConvexProviderWithClerk>` to pass Clerk's auth context to Convex. Convex automatically validates the Clerk JWT.
3. **Backend**: Convex functions access the authenticated user via `ctx.auth.getUserIdentity()`. No custom session middleware needed.

### Frontend Setup

```typescript
// src/main.tsx
import { ClerkProvider, useAuth } from "@clerk/clerk-react"
import { ConvexProviderWithClerk } from "convex/react-clerk"
import { ConvexReactClient } from "convex/react"

const convex = new ConvexReactClient(import.meta.env.VITE_CONVEX_URL)

function App() {
  return (
    <ClerkProvider publishableKey={import.meta.env.VITE_CLERK_PUBLISHABLE_KEY}>
      <ConvexProviderWithClerk client={convex} useAuth={useAuth}>
        <RegistryProvider>
          <Router />
        </RegistryProvider>
      </ConvexProviderWithClerk>
    </ClerkProvider>
  )
}
```

### Environment Variables

```bash
VITE_CLERK_PUBLISHABLE_KEY=pk_test_...   # From Clerk Dashboard
VITE_CONVEX_URL=https://...convex.cloud  # From Convex Dashboard
```

### Protecting Routes (Frontend)

Use Clerk's components to gate authenticated content:

```typescript
import { SignedIn, SignedOut, RedirectToSignIn } from "@clerk/clerk-react"

function ProtectedRoute({ children }: { children: React.ReactNode }) {
  return (
    <>
      <SignedIn>{children}</SignedIn>
      <SignedOut><RedirectToSignIn /></SignedOut>
    </>
  )
}
```

### Accessing Auth in Convex Functions

```typescript
// convex/functions/items.ts
import { query, mutation } from "../_generated/server"

export const list = query({
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity()
    if (!identity) throw new Error("Unauthenticated")

    // identity.subject is the Clerk user ID
    // identity.email, identity.name, etc. are also available
    return await ctx.db.query("items")
      .filter((q) => q.eq(q.field("userId"), identity.subject))
      .collect()
  }
})
```

### Passing Auth Token to HTTP API

When using Effect HttpApi endpoints (not Convex functions directly), pass the Clerk session token via the Authorization header:

```typescript
import { useAuth } from "@clerk/clerk-react"

// Get the token from Clerk
const { getToken } = useAuth()
const token = await getToken({ template: "convex" })

// Pass as Bearer token to HTTP API endpoints
const headers = { Authorization: `Bearer ${token}` }
```

### HttpApi Auth Middleware (for Effect HttpApi endpoints)

```typescript
class AuthMiddleware extends HttpApiMiddleware.Tag<AuthMiddleware>()(
  "AuthMiddleware",
  {
    failure: UnauthorizedError,
    provides: CurrentUser,
    security: {
      bearer: HttpApiSecurity.bearer
    }
  }
) {}

const AuthMiddlewareLive = Layer.succeed(
  AuthMiddleware,
  AuthMiddleware.of({
    bearer: (token) =>
      Effect.gen(function* () {
        // Verify the Clerk JWT and extract user identity
        const user = yield* verifyClerkToken(token)
        return user
      })
  })
)

// Apply to group
class SecureApi extends HttpApiGroup.make("secure")
  .add(protectedEndpoint)
  .middleware(AuthMiddleware)
{}
```

### Clerk Documentation

- [Clerk React Quickstart](https://clerk.com/docs/quickstarts/react)
- [Clerk + Convex Integration](https://clerk.com/docs/guides/development/integrations/databases/convex)
- [Convex Auth with Clerk](https://docs.convex.dev/auth/clerk)

---

## Frontend Client (AtomHttpApi)

### Setting Up AtomHttpApi

```typescript
// src/api/client.ts
import { AtomHttpApi } from "@effect-atom/atom"
import { FetchHttpClient } from "@effect/platform"

class ApiClient extends AtomHttpApi.Tag<ApiClient>()("ApiClient", {
  api: AppApi,  // Same HttpApi definition used on backend
  httpClient: FetchHttpClient.layer,
  baseUrl: window.location.origin
}) {}
```

### Provider Setup

```typescript
// src/main.tsx
import { RegistryProvider } from "@effect-atom/atom-react"

function App() {
  return (
    <RegistryProvider>
      <Router />
    </RegistryProvider>
  )
}
```

---

## Data Loading Patterns

### Pattern 1: Simple Query

```typescript
import { useAtomValue } from "@effect-atom/atom-react"
import { Result } from "@effect-atom/atom"

function ItemDetail({ id }: { id: string }) {
  const result = useAtomValue(
    ApiClient.query("items", "getItem", {
      urlParams: { id },
      reactivityKeys: { items: [id] }
    })
  )

  return Result.match(result, {
    onInitial: () => <LoadingSkeleton />,
    onSuccess: (item) => <div>{item.name}</div>,
    onFailure: (cause) => <ErrorState cause={cause} />
  })
}
```

### Pattern 2: List Query

```typescript
function ItemsList() {
  const result = useAtomValue(
    ApiClient.query("items", "listItems", {
      urlParams: { limit: "20" },
      reactivityKeys: { items: true }
    })
  )

  return Result.match(result, {
    onInitial: () => <TableSkeleton rows={5} />,
    onSuccess: (items) =>
      items.length === 0
        ? <EmptyState title="No items yet" action={<CreateItemButton />} />
        : <ItemsTable items={items} />,
    onFailure: (cause) => <ErrorState cause={cause} />
  })
}
```

### Pattern 3: Atom Family (Parameterized Queries)

```typescript
// src/atoms/items.ts
import { Atom } from "@effect-atom/atom"

const itemAtom = Atom.family((id: string) =>
  ApiClient.query("items", "getItem", {
    urlParams: { id },
    reactivityKeys: { items: [id] }
  })
)

// Use in components - same id = same atom instance
function ItemCard({ id }: { id: string }) {
  const result = useAtomValue(itemAtom(id))

  return Result.match(result, {
    onInitial: () => <Skeleton />,
    onSuccess: (item) => <Card>{item.name}</Card>,
    onFailure: () => <ErrorCard />
  })
}
```

### Pattern 4: Result Builder (Complex UI)

```typescript
function ItemDetail({ id }: { id: string }) {
  const result = useAtomValue(itemAtom(id))

  return Result.builder(result)
    .onInitial(() => <LoadingSkeleton />)
    .onFailure((cause) => <ErrorState cause={cause} />)
    .onSuccess((item, { waiting }) => (
      <div className="space-y-4">
        <h1 className="text-2xl font-bold">{item.name}</h1>
        <p className="text-gray-600">{item.description}</p>
        {waiting && <span className="animate-pulse">Refreshing...</span>}
      </div>
    ))
    .render()
}
```

### Pattern 5: Suspense Integration

```typescript
import { useAtomSuspense } from "@effect-atom/atom-react"

function ItemDetail({ id }: { id: string }) {
  const item = useAtomSuspense(itemAtom(id))
  return <div>{item.name}</div>
}

// Parent wraps with Suspense
<Suspense fallback={<LoadingSkeleton />}>
  <ItemDetail id={id} />
</Suspense>
```

### Pattern 6: Manual Refresh

```typescript
import { useAtomRefresh } from "@effect-atom/atom-react"

function ItemDetail({ id }: { id: string }) {
  const result = useAtomValue(itemAtom(id))
  const refresh = useAtomRefresh(itemAtom(id))

  return (
    <div>
      {Result.match(result, { ... })}
      <button onClick={refresh}>Refresh</button>
    </div>
  )
}
```

---

## Mutations & Cache Invalidation

### Simple Mutation

```typescript
function CreateItemForm() {
  const createItem = useAtomSet(
    ApiClient.mutation("items", "createItem", {
      reactivityKeys: { items: true }  // Auto-invalidates all item queries
    }),
    { mode: "promiseExit" }
  )
  const [name, setName] = useState("")
  const [isSubmitting, setIsSubmitting] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    if (isSubmitting) return
    setIsSubmitting(true)
    setError(null)

    const result = await createItem({
      name,
      categoryId: CategoryId.make(selectedCategoryId)
    })

    if (Exit.isFailure(result)) {
      // Check error type via _tag
      const err = Cause.squash(result.cause)
      if ("_tag" in err && err._tag === "ValidationError") {
        setError(err.message)
      } else {
        setError("Failed to create item")
      }
      setIsSubmitting(false)
      return
    }

    // Queries with matching reactivityKeys auto-refresh
    navigate(`/items/${result.value.id}`)
  }

  return <form onSubmit={handleSubmit}>...</form>
}
```

### Update with Specific Invalidation

```typescript
function EditItemForm({ item }: { item: Item }) {
  const updateItem = useAtomSet(
    ApiClient.mutation("items", "updateItem", {
      urlParams: { id: item.id },
      reactivityKeys: { items: [item.id] }  // Only invalidates this item's queries
    }),
    { mode: "promiseExit" }
  )

  const handleSave = async () => {
    const result = await updateItem({ name })
    if (Exit.isSuccess(result)) {
      // Query for this item auto-refreshes via reactivityKeys
    }
  }
}
```

### Delete

```typescript
function DeleteItemButton({ id }: { id: string }) {
  const deleteItem = useAtomSet(
    ApiClient.mutation("items", "deleteItem", {
      urlParams: { id },
      reactivityKeys: { items: true }
    }),
    { mode: "promiseExit" }
  )

  const handleDelete = async () => {
    if (!confirm("Are you sure?")) return
    const result = await deleteItem({})
    if (Exit.isSuccess(result)) {
      navigate("/items")
    }
  }

  return <button onClick={handleDelete}>Delete</button>
}
```

---

## Anti-Patterns to Avoid

### DO NOT Use Raw fetch()

```typescript
// WRONG - bypasses type safety
const response = await fetch("/api/v1/items")
const data = await response.json()

// CORRECT - use AtomHttpApi
const result = useAtomValue(
  ApiClient.query("items", "listItems", {})
)
```

### DO NOT Use Effect.runPromise for Data Loading

```typescript
// WRONG - bypasses Effect Atom's reactive system
const items = await Effect.runPromise(someEffect)

// CORRECT - use AtomHttpApi for reactive data loading
const result = useAtomValue(ApiClient.query("items", "listItems", {}))
```

### DO NOT Manage Loading State Manually

```typescript
// WRONG - manual loading/error state
const [loading, setLoading] = useState(true)
const [error, setError] = useState(null)
const [data, setData] = useState(null)

useEffect(() => {
  fetch("/api/v1/items").then(r => r.json()).then(setData).catch(setError)
}, [])

// CORRECT - Effect Atom handles all of this
const result = useAtomValue(ApiClient.query("items", "listItems", {}))
// Result has loading/success/failure states built in
```

### DO NOT Manually Decode in Handlers

```typescript
// WRONG - manual decoding
.handle("getItem", (_) =>
  Effect.gen(function* () {
    const itemId = yield* Schema.decodeUnknown(ItemId)(_.path.id)
  })
)

// CORRECT - use domain schema in endpoint definition, auto-decoded
.setPath(Schema.Struct({ id: ItemId }))
.handle("getItem", (_) =>
  Effect.gen(function* () {
    // _.path.id is already ItemId
  })
)
```

---

## Checklist

1. **Path params**: Use branded string schemas (`ItemId`, `CategoryId`)
2. **Query params**: Use branded strings + transforming schemas (`BooleanFromString`, `NumberFromString`)
3. **Request body**: Use domain schemas with `OptionFromNullOr` for optional fields
4. **Response body**: Return domain entities or wrap in response classes
5. **Errors**: `Schema.TaggedError` with `HttpApiSchema.annotations({ status })` for HTTP mapping
6. **No manual decoding**: If calling `Schema.decode*` in a handler, reconsider the API schema
7. **AtomHttpApi for frontend**: Use `query()` for reads, `mutation()` for writes
8. **reactivityKeys**: Tag queries and mutations for automatic cache invalidation
9. **Result.match**: Always handle all three states (initial, success, failure)
