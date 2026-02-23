# Effect Guide

This is the consolidated guide for using Effect in projects based on this template. It covers coding patterns, error handling, Schema, Layers, and testing.

---

## Table of Contents

1. [Critical Rules](#critical-rules)
2. [Schema Patterns](#schema-patterns)
3. [Error Handling](#error-handling)
4. [Service Pattern (Context.Tag + Layer)](#service-pattern-contexttag--layer)
5. [Layer Memoization & Composition](#layer-memoization--composition)
6. [Testing Patterns (@effect/vitest)](#testing-patterns-effectvitest)

---

## Critical Rules

### 1. NEVER Use `any` or Type Casts (`x as Y`)

```typescript
// WRONG - never use any
const result: any = someValue
const data = value as any

// WRONG - never use type casts
const account = data as Account

// CORRECT - use proper types, generics, or unknown
const result: SomeType = someValue

// CORRECT - use Schema.make() for branded types
const id = UserId.make(rawId)

// CORRECT - use Schema.decodeUnknown for parsing
const account = yield* Schema.decodeUnknown(Account)(data)
```

**AVOID `eslint-disable-next-line @typescript-eslint/consistent-type-assertions`** - Using this disable comment should be an absolute last resort. Before adding it, exhaust ALL alternatives:

1. Use `Schema.make()` for branded types
2. Use `Schema.decodeUnknown()` for parsing unknown data
3. Use `Option.some<T>()` / `Option.none<T>()` for explicit Option types
4. Use `identity<T>()` from `effect/Function` for compile-time type verification
5. Use proper generics and type parameters

#### Type-Safe Alternatives to Casting

```typescript
// When you need to specify the type for Option.some/none, use type parameters
const someValue = Option.some<Account>(account)  // Option<Account>
const noneValue = Option.none<Account>()         // Option<Account>

// Use Option.fromNullable for nullable values
const desc = Option.fromNullable(row.description)

// When you need to assert a value matches a type (without casting), use identity
import { identity } from "effect/Function"
const verified = identity<Account>(x)  // Compile error if x isn't Account
```

### 2. NEVER Use `catchAll` When Error Type Is `never`

```typescript
// If the effect never fails, error type is `never`
const infallibleEffect: Effect.Effect<Result, never> = Effect.succeed(value)

// WRONG - catchAll on never is useless
infallibleEffect.pipe(
  Effect.catchAll((e) => Effect.fail(new SomeError({ message: String(e) })))
)

// CORRECT - if error is never, just use the effect directly
infallibleEffect
```

### 3. NEVER Use Global `Error` in Effect Error Channel

```typescript
// WRONG - global Error breaks Effect's typed error handling
const bad: Effect.Effect<Result, Error> = Effect.fail(new Error("failed"))

// CORRECT - use Schema.TaggedError for all domain errors
export class ValidationError extends Schema.TaggedError<ValidationError>()(
  "ValidationError",
  { message: Schema.String }
) {}

const good: Effect.Effect<Result, ValidationError> = Effect.fail(
  new ValidationError({ message: "failed" })
)
```

### 4. NEVER Use `{ disableValidation: true }` - It Is Banned

```typescript
// WRONG - disableValidation is banned
const item = Item.make(data, { disableValidation: true })

// CORRECT - let Schema validate the data, always
const item = Item.make(data)
```

### 5. Don't Wrap Safe Operations in Effect Unnecessarily

```typescript
// WRONG - Effect.try is for operations that might throw
const mapped = Effect.try(() => array.map(fn))  // array.map doesn't throw

// CORRECT - for pure transformations, just use a normal function
const mapItems = (rows: Row[]): Item[] => rows.map(toItem)

// CORRECT - use Effect.try ONLY for operations that might throw
const parsed = Effect.try(() => JSON.parse(jsonString))
```

### 6. NEVER Use `Effect.catchAllCause` to Wrap Errors

```typescript
// WRONG - catchAllCause catches BOTH errors AND defects (bugs)
const wrapError = (operation: string) =>
  <A, E, R>(effect: Effect.Effect<A, E, R>): Effect.Effect<A, AppError, R> =>
    Effect.catchAllCause(effect, (cause) =>
      Effect.fail(new AppError({ operation, cause: Cause.squash(cause) }))
    )

// CORRECT - use mapError to transform only expected errors
const wrapError = (operation: string) =>
  <A, E, R>(effect: Effect.Effect<A, E, R>): Effect.Effect<A, AppError, R> =>
    Effect.mapError(effect, (error) =>
      new AppError({ operation, cause: error })
    )
```

**Why this matters:**
- **Errors** are expected failures (user not found, validation failed) - they should be handled
- **Defects** are bugs (null pointer, division by zero) - they should crash and be fixed
- `catchAllCause` catches both, hiding bugs that should be fixed

### 7. NEVER Silently Swallow Errors

**Principle: If something can fail, the failure MUST be visible in the Effect's error channel `E`.**

```typescript
// WRONG - silently swallowing errors hides bugs
yield* auditLogService.log(entry).pipe(
  Effect.catchTag("AuditLogError", () => Effect.void)
)

// WRONG - Effect.ignore silently discards errors
yield* someEffect.pipe(Effect.ignore)

// CORRECT - let error propagate (caller decides how to handle)
yield* auditLogService.log(entry)

// CORRECT - transform error to different type (still visible in types)
yield* auditLogService.log(entry).pipe(
  Effect.mapError((e) => new MyError({ cause: e }))
)

// CORRECT - provide fallback value (for queries, not side effects)
const result = yield* findItem(id).pipe(
  Effect.catchTag("NotFoundError", () => Effect.succeed(null))
)
```

---

## Pipe Composition

When composing many operations, chain `.pipe()` calls for readability:

```typescript
// Chained pipes - easier to read
const result = effect
  .pipe(Effect.map(transformA))
  .pipe(Effect.flatMap(fetchRelated))
  .pipe(Effect.catchTag("NotFound", handleNotFound))
  .pipe(Effect.withSpan("myOperation"))
```

**Note:** `.pipe()` has a maximum of 20 arguments - long pipelines must be split.

---

## Schema Patterns

### Always Use Schema.Class for Domain Entities

**Never manually implement Equal/Hash** - Schema.Class provides them automatically.

```typescript
export class Account extends Schema.Class<Account>("Account")({
  id: AccountId,
  code: Schema.NonEmptyTrimmedString,
  name: Schema.NonEmptyTrimmedString,
  type: Schema.Literal("Asset", "Liability", "Equity", "Revenue", "Expense"),
  isActive: Schema.Boolean
}) {
  get isExpenseOrRevenue(): boolean {
    return this.type === "Expense" || this.type === "Revenue"
  }
}

// Usage - .make() is automatic, Equal/Hash work automatically
const account = Account.make({ id, code: "1000", name: "Cash", ... })
Equal.equals(account1, account2)  // Works without manual implementation
```

### Branded Types for IDs

```typescript
export const UserId = Schema.NonEmptyTrimmedString.pipe(
  Schema.brand("UserId")
)
export type UserId = typeof UserId.Type

const id = UserId.make("usr_123")  // Creates branded UserId
```

### Schema.TaggedError for Domain Errors

```typescript
export class ItemNotFound extends Schema.TaggedError<ItemNotFound>()(
  "ItemNotFound",
  { itemId: ItemId }
) {
  get message(): string {
    return `Item not found: ${this.itemId}`
  }
}

// Type guards are automatically derived
export const isItemNotFound = Schema.is(ItemNotFound)
```

### Recursive Schema Classes

For self-referencing types, use `Schema.suspend()`:

```typescript
export interface TreeNodeEncoded extends Schema.Schema.Encoded<typeof TreeNode> {}

export class TreeNode extends Schema.TaggedClass<TreeNode>()("TreeNode", {
  value: Schema.String,
  children: Schema.Array(Schema.suspend((): Schema.Schema<TreeNode, TreeNodeEncoded> => TreeNode))
}) {}
```

### Never Use *FromSelf Schemas

```typescript
// WRONG
parentId: Schema.OptionFromSelf(ItemId)  // NO!

// CORRECT
parentId: Schema.Option(ItemId)  // YES - encodes to JSON properly
```

### Use Schema.decodeUnknown (Not Sync)

```typescript
// WRONG - throws exceptions
const item = Schema.decodeUnknownSync(Item)(data)

// CORRECT - returns Effect
const item = yield* Schema.decodeUnknown(Item)(data)
```

### Use Chunk Instead of Array for Structural Equality

```typescript
// Arrays break structural equality - WRONG
Equal.equals([1, 2, 3], [1, 2, 3])  // false!

// Use Chunk - CORRECT
Equal.equals(Chunk.make(1, 2, 3), Chunk.make(1, 2, 3))  // true!
```

---

## Error Handling

### Error Architecture

Errors flow from Data Layer -> Domain -> RPC:

1. **Data Layer** - Convex errors, external API errors
2. **Domain Layer** - Business logic errors (ValidationError, NotFoundError)
3. **RPC Layer** - Typed RPC error responses (handled by Effect Atom's Result on frontend)

### Using Effect.catchTag

```typescript
const program = fetchItem(itemId).pipe(
  Effect.catchTag("ItemNotFound", (error) =>
    Effect.succeed(createDefaultItem())
  ),
  Effect.catchTag("ConvexError", (error) =>
    Effect.logError(`Database error: ${error.cause}`)
  )
)

// Or use Match for exhaustive handling
const handleError = Match.type<ItemError>().pipe(
  Match.tag("ItemNotFound", (err) => `Item ${err.itemId} not found`),
  Match.tag("ConvexError", (err) => `Database error occurred`),
  Match.exhaustive
)
```

---

## Service Pattern (Context.Tag + Layer)

```typescript
// Service interface
export interface ItemService {
  readonly findById: (id: ItemId) => Effect.Effect<Item, ItemNotFound>
  readonly findAll: () => Effect.Effect<ReadonlyArray<Item>>
  readonly create: (item: Item) => Effect.Effect<Item, ValidationError>
}

// Service tag
export class ItemService extends Context.Tag("ItemService")<
  ItemService,
  ItemService
>() {}
```

### Creating Layers

**Layer.effect** - When service creation is effectful but doesn't need cleanup:

```typescript
const make = Effect.gen(function* () {
  const convex = yield* ConvexClient

  return {
    findById: (id) => Effect.gen(function* () { ... }),
    findAll: () => Effect.gen(function* () { ... }),
    create: (item) => Effect.gen(function* () { ... })
  }
})

export const ItemServiceLive = Layer.effect(ItemService, make)
```

**Layer.scoped** - When the service needs resource cleanup:

```typescript
const make = Effect.gen(function* () {
  const changes = yield* PubSub.unbounded<ItemChange>()

  yield* Effect.forkScoped(
    subscribeToChanges().pipe(
      Stream.runForEach((change) => PubSub.publish(changes, change))
    )
  )

  return {
    findById: (id) => Effect.gen(function* () { ... }),
    subscribe: PubSub.subscribe(changes)
  }
})

export const ItemServiceLive = Layer.scoped(ItemService, make)
```

### Two Types of Context: Global vs Per-Request

**Global Context** - Provided via Layers at startup (Services, Config)

**Per-Request Context** - Provided via `Effect.provideService` for each request:

```typescript
// CORRECT - per-request context
const handleRequest = (req: Request) =>
  Effect.gen(function* () {
    const result = yield* businessLogic()
    return result
  }).pipe(
    Effect.provideService(CurrentUserId, extractUserId(req))
  )

// WRONG - don't use Layers for per-request data
Effect.provide(Layer.succeed(CurrentUserId, extractUserId(req)))  // NO!
```

---

## Layer Memoization & Composition

### Core Principle: Identity-Based Memoization

**Layers are memoized by object identity (reference equality), not by type or value.**

```typescript
// SAME layer reference used twice = ONE instance
const layer = Layer.effect(MyTag, Effect.succeed("value"))
const composed = layer.pipe(Layer.merge(layer)) // Single instance!

// DIFFERENT layer references = TWO instances
const layer1 = Layer.effect(MyTag, Effect.succeed("value"))
const layer2 = Layer.effect(MyTag, Effect.succeed("value"))
const composed2 = layer1.pipe(Layer.merge(layer2)) // Two instances!
```

### Layer.fresh: Escaping Memoization

`Layer.fresh(layer)` wraps a layer to escape memoization:

```typescript
const layer = Layer.effect(MyTag, createResource())

// Without fresh: single instance
const single = layer.pipe(Layer.merge(layer))

// With fresh: two separate instances
const two = layer.pipe(Layer.merge(Layer.fresh(layer)))
```

---

## Testing Patterns (@effect/vitest)

### Test Variants

| Method | TestServices | Scope | Use Case |
|--------|--------------|-------|----------|
| `it.effect` | TestClock | No | Most tests - deterministic time |
| `it.live` | Real clock | No | Tests needing real time/IO |
| `it.scoped` | TestClock | Yes | Tests with resources (acquireRelease) |
| `it.scopedLive` | Real clock | Yes | Real time + resources |

### it.effect - Use for Most Tests

```typescript
it.effect("processes after delay", () =>
  Effect.gen(function* () {
    const fiber = yield* Effect.fork(
      Effect.sleep(Duration.minutes(5)).pipe(Effect.map(() => "done"))
    )

    yield* TestClock.adjust(Duration.minutes(5))  // No real waiting!

    const result = yield* Fiber.join(fiber)
    expect(result).toBe("done")
  })
)
```

### Sharing Layers Between Tests

```typescript
layer(ItemServiceLive)("ItemService", (it) => {
  it.effect("finds item by id", () =>
    Effect.gen(function* () {
      const service = yield* ItemService
      const item = yield* service.findById(testItemId)
      expect(item.name).toBe("Test")
    })
  )
})
```

### Property-Based Testing

```typescript
import { it } from "@effect/vitest"
import { FastCheck, Schema, Arbitrary } from "effect"

// Synchronous property test
it.prop("symmetry", [Schema.Number, FastCheck.integer()], ([a, b]) =>
  a + b === b + a
)

// Effectful property test
it.effect.prop("symmetry in effect", [Schema.Number, FastCheck.integer()], ([a, b]) =>
  Effect.gen(function* () {
    yield* Effect.void
    return a + b === b + a
  })
)
```

### When to Use Layer.fresh in Tests

**Use Layer.fresh for module-level constant layers with different configs:**

```typescript
export const AuthServiceLive = Layer.effect(AuthService, make)

// In tests with different configs - NEED Layer.fresh
const createTestLayer = (options: { autoProvisionUsers?: boolean }) => {
  const AuthConfigLayer = Layer.effect(AuthServiceConfig, Effect.succeed({
    autoProvisionUsers: options.autoProvisionUsers ?? true
  }))

  return Layer.fresh(AuthServiceLive).pipe(
    Layer.provideMerge(AuthConfigLayer)
  )
}
```

**Don't use Layer.fresh for factory functions** - they already return new objects.
