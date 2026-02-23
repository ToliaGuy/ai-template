# Reference Repositories

This project includes reference repositories as git subtrees in `repos/` for pattern discovery and API lookup.

## Effect (`repos/effect/`)

The core Effect library for functional TypeScript.

### Core Package (`repos/effect/packages/effect/src/`)

Essential modules for this project:

| Module | Path | Use For |
|--------|------|---------|
| **Effect** | `Effect.ts` | Core effect type, operations |
| **Schema** | `Schema.ts` | Validation, encoding/decoding |
| **Context** | `Context.ts` | Dependency injection |
| **Layer** | `Layer.ts` | Service composition |
| **Data** | `Data.ts` | Value objects, tagged unions |
| **Brand** | `Brand.ts` | Branded types (UserId, etc.) |
| **Either** | `Either.ts` | Error handling |
| **Option** | `Option.ts` | Optional values |
| **Match** | `Match.ts` | Pattern matching |
| **BigDecimal** | `BigDecimal.ts` | Monetary calculations |
| **DateTime** | `DateTime.ts` | Date/time handling |
| **Duration** | `Duration.ts` | Time durations |
| **Struct** | `Struct.ts` | Object operations |
| **Record** | `Record.ts` | Record utilities |
| **Array** | `Array.ts` | Array utilities |
| **Stream** | `Stream.ts` | Streaming data |
| **Ref** | `Ref.ts` | Mutable references |
| **Queue** | `Queue.ts` | Async queues |
| **Config** | `Config.ts` | Configuration |
| **Fiber** | `Fiber.ts` | Concurrent execution |

**Search patterns in Effect:**
```bash
# Find service definitions
grep -r "Context.Tag" repos/effect/packages/effect/src/

# Find schema examples
grep -r "Schema.Struct" repos/effect/packages/effect/src/

# Find branded type examples
grep -r "Brand.nominal" repos/effect/packages/effect/src/

# Find Layer patterns
grep -r "Layer.succeed" repos/effect/packages/effect/src/
```

### SQL Packages (`repos/effect/packages/sql*/`)

Database integration for Effect:

| Package | Path | Description |
|---------|------|-------------|
| **sql** | `repos/effect/packages/sql/` | Core SQL abstractions |
| **sql-pg** | `repos/effect/packages/sql-pg/` | PostgreSQL client |
| **sql-drizzle** | `repos/effect/packages/sql-drizzle/` | Drizzle ORM integration |

**Search patterns for SQL:**
```bash
# Find repository patterns
grep -r "SqlClient" repos/effect/packages/sql/src/

# Find transaction patterns
grep -r "Effect.acquireRelease" repos/effect/packages/sql*/src/

# Find query patterns
grep -rn "sql\`" repos/effect/packages/sql*/src/
```

### Platform Packages (`repos/effect/packages/platform*/`)

Runtime platform abstractions:

| Package | Path | Description |
|---------|------|-------------|
| **platform** | `repos/effect/packages/platform/` | Core platform abstractions |
| **platform-node** | `repos/effect/packages/platform-node/` | Node.js runtime |
| **platform-browser** | `repos/effect/packages/platform-browser/` | Browser runtime |

**Key modules:**
- `FileSystem.ts` - File operations
- `Terminal.ts` - CLI utilities
- `HttpClient.ts` - HTTP client (used as RPC transport layer)
- `HttpServer.ts` - HTTP server (used as RPC transport layer)

### HttpApi Packages (Core to this template)

The HTTP API framework used for defining typed endpoints:

**Search patterns for HttpApi:**
```bash
# Find HttpApiEndpoint definitions
grep -r "HttpApiEndpoint" repos/effect/packages/platform/src/

# Find HttpApiGroup patterns
grep -r "HttpApiGroup" repos/effect/packages/platform/src/

# Find HttpApi composition
grep -r "HttpApi.make" repos/effect/packages/platform/src/

# Find HttpApiBuilder patterns (handler implementation)
grep -r "HttpApiBuilder" repos/effect/packages/platform/src/

# Find HttpApiSchema error annotations
grep -r "HttpApiSchema" repos/effect/packages/platform/src/

# Find HttpApiMiddleware patterns
grep -r "HttpApiMiddleware" repos/effect/packages/platform/src/

# Find HttpApiSecurity patterns (auth)
grep -r "HttpApiSecurity" repos/effect/packages/platform/src/
```

### Other Effect Packages

| Package | Path | Description |
|---------|------|-------------|
| **vitest** | `repos/effect/packages/vitest/` | Testing utilities |
| **cli** | `repos/effect/packages/cli/` | CLI framework |
| **cluster** | `repos/effect/packages/cluster/` | Distributed systems |
| **workflow** | `repos/effect/packages/workflow/` | Workflow engine |
| **experimental** | `repos/effect/packages/experimental/` | Experimental features |

---

## Effect Atom (`repos/effect-atom/`) - Core to this template

Reactive state management for Effect. **AtomHttpApi is the primary frontend data loading mechanism.**

### Packages

| Package | Path | Description |
|---------|------|-------------|
| **atom** | `repos/effect-atom/packages/atom/` | Core atom primitives, AtomHttpApi |
| **atom-react** | `repos/effect-atom/packages/atom-react/` | React bindings (useAtomValue, useAtomSet) |
| **atom-vue** | `repos/effect-atom/packages/atom-vue/` | Vue bindings |
| **atom-livestore** | `repos/effect-atom/packages/atom-livestore/` | LiveStore integration |

### Key Files

| File | Purpose |
|------|---------|
| `packages/atom/src/Atom.ts` | Atom creation, family, derived atoms |
| `packages/atom/src/Result.ts` | Result type (Initial/Success/Failure) |
| `packages/atom/src/AtomHttpApi.ts` | HTTP API client integration |
| `packages/atom-react/src/Hooks.ts` | React hooks (useAtomValue, useAtomSet, etc.) |
| `packages/atom-react/src/RegistryContext.ts` | RegistryProvider |

**Search patterns:**
```bash
# Find atom creation patterns
grep -r "Atom.make" repos/effect-atom/packages/atom/src/

# Find AtomHttpApi patterns (key for this template)
grep -r "AtomHttpApi" repos/effect-atom/packages/atom/src/

# Find React hook patterns
grep -r "useAtomValue" repos/effect-atom/packages/atom-react/src/
grep -r "useAtomSet" repos/effect-atom/packages/atom-react/src/
grep -r "useAtomSuspense" repos/effect-atom/packages/atom-react/src/
grep -r "useAtomRefresh" repos/effect-atom/packages/atom-react/src/

# Find Result type patterns
grep -r "Result.match" repos/effect-atom/packages/atom/src/
grep -r "Result.builder" repos/effect-atom/packages/atom/src/

# Find Atom.family patterns
grep -r "Atom.family" repos/effect-atom/packages/atom/src/

# Find reactivity patterns
grep -r "reactivityKeys" repos/effect-atom/packages/atom/src/
```

---

## Common Search Patterns

### Finding Effect Patterns

```bash
# Schema definitions
grep -rn "Schema\." repos/effect/packages/effect/src/Schema.ts | head -50

# Service pattern (Context.Tag + Layer)
grep -rn "extends Context.Tag" repos/effect/packages/*/src/

# Error handling (tagged errors)
grep -rn "TaggedError" repos/effect/packages/effect/src/

# Branded types
grep -rn "Brand\." repos/effect/packages/effect/src/Brand.ts

# Effect.gen pattern
grep -rn "Effect.gen" repos/effect/packages/*/src/ | head -20

# BigDecimal operations
grep -rn "BigDecimal\." repos/effect/packages/effect/src/BigDecimal.ts
```

### Finding HttpApi Patterns

```bash
# HttpApiEndpoint definitions
grep -rn "HttpApiEndpoint" repos/effect/packages/platform/src/

# HttpApiGroup composition
grep -rn "HttpApiGroup.make" repos/effect/packages/platform/src/

# HttpApi composition
grep -rn "HttpApi.make" repos/effect/packages/platform/src/

# HttpApiBuilder handler patterns
grep -rn "HttpApiBuilder" repos/effect/packages/platform/src/

# Error status annotations
grep -rn "HttpApiSchema.annotations" repos/effect/packages/platform/src/

# Middleware patterns
grep -rn "HttpApiMiddleware" repos/effect/packages/platform/src/
```

### Finding Tests

```bash
# Effect tests
ls repos/effect/packages/effect/test/

# RPC tests
ls repos/effect/packages/rpc/test/

# Find specific test patterns
grep -rn "it\(" repos/effect/packages/rpc/test/

# Find vitest patterns
grep -rn "describe\(" repos/effect/packages/vitest/test/
```

### Finding Type Definitions

```bash
# Find interface definitions
grep -rn "^export interface" repos/effect/packages/effect/src/

# Find type aliases
grep -rn "^export type" repos/effect/packages/effect/src/ | head -30
```

---

## Package Versions

To check package versions in the reference repos:

```bash
# Effect version
jq '.version' repos/effect/packages/effect/package.json

# RPC version
jq '.version' repos/effect/packages/rpc/package.json

# Effect Atom version
jq '.version' repos/effect-atom/packages/atom/package.json
```
