# Frontend Guide

This is the consolidated guide for frontend development, covering React patterns with Vite, Effect Atom for data loading, styling, components, and UX best practices.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Data Loading with Effect Atom](#data-loading-with-effect-atom)
3. [State Management](#state-management)
4. [Mutations & Forms](#mutations--forms)
5. [Styling with Tailwind](#styling-with-tailwind)
6. [Component Patterns](#component-patterns)
7. [Page Templates](#page-templates)
8. [Empty, Loading, and Error States](#empty-loading-and-error-states)
9. [Form Design](#form-design)
10. [Navigation & Layout](#navigation--layout)
11. [Accessibility](#accessibility)
12. [Design System Reference](#design-system-reference)

---

## Architecture Overview

The frontend uses:
- **Vite + React** - Fast dev server, simple SPA
- **Effect Atom + AtomHttpApi** - Reactive data loading from the HTTP API
- **Tailwind CSS** - Utility-first CSS framework

**Data Flow:**
```
Component (useAtomValue) → Effect Atom → AtomHttpApi → HTTP API → Convex
```

### File Structure

```
src/
├── api/                 # AtomHttpApi client setup
│   └── client.ts        # ApiClient definition
├── atoms/               # Atom definitions
│   └── items.ts         # Feature-specific atoms (families, derived)
├── components/          # Shared components
│   ├── ui/              # Base UI components (Button, Card, etc.)
│   ├── layout/          # AppLayout, Sidebar, Header, Breadcrumbs
│   └── [feature]/       # Feature-specific components
├── routes/              # Page components
│   ├── items/
│   │   ├── ItemsList.tsx
│   │   ├── ItemDetail.tsx
│   │   └── CreateItem.tsx
│   └── auth/
│       ├── Login.tsx
│       └── Register.tsx
├── styles/
│   └── globals.css      # Tailwind imports
├── App.tsx              # Router + RegistryProvider
└── main.tsx             # Vite entry point
```

---

## Data Loading with Effect Atom

### Setup: RegistryProvider

All Effect Atom hooks require a `RegistryProvider` at the root:

```typescript
// src/App.tsx
import { RegistryProvider } from "@effect-atom/atom-react"

function App() {
  return (
    <RegistryProvider>
      <Router />
    </RegistryProvider>
  )
}
```

### Setup: AtomHttpApi Client

```typescript
// src/api/client.ts
import { AtomHttpApi } from "@effect-atom/atom"
import { FetchHttpClient } from "@effect/platform"
import { AppApi } from "@/shared/AppApi"  // Shared HttpApi definition (see convex/)

class ApiClient extends AtomHttpApi.Tag<ApiClient>()("ApiClient", {
  api: AppApi,
  httpClient: FetchHttpClient.layer,
  baseUrl: window.location.origin
}) {}

export { ApiClient }
```

### Basic Query

```typescript
import { useAtomValue } from "@effect-atom/atom-react"
import { Result } from "@effect-atom/atom"
import { ApiClient } from "@/api/client"

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

### Parameterized Query (Atom Family)

```typescript
// src/atoms/items.ts
import { Atom } from "@effect-atom/atom"
import { ApiClient } from "@/api/client"

export const itemAtom = Atom.family((id: string) =>
  ApiClient.query("items", "getItem", {
    urlParams: { id },
    reactivityKeys: { items: [id] }
  })
)

// src/routes/items/ItemDetail.tsx
function ItemDetail({ id }: { id: string }) {
  const result = useAtomValue(itemAtom(id))

  return Result.match(result, {
    onInitial: () => <LoadingSkeleton />,
    onSuccess: (item, { waiting }) => (
      <div>
        <h1 className="text-2xl font-bold">{item.name}</h1>
        {waiting && <span className="text-gray-400 animate-pulse">Refreshing...</span>}
      </div>
    ),
    onFailure: (cause) => <ErrorState cause={cause} />
  })
}
```

### Result Builder (Complex UI)

For more complex rendering logic, use the builder pattern:

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
        {waiting && <RefreshingBanner />}
      </div>
    ))
    .render()
}
```

### Suspense Integration

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

### Manual Refresh

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

## State Management

### Server State: Effect Atom (AtomHttpApi)

All server data is loaded via Effect Atom. Never fetch in useEffect.

```typescript
// CORRECT - Effect Atom handles loading, caching, and reactivity
const result = useAtomValue(ApiClient.query("items", "listItems", {}))

// WRONG - manual fetching
useEffect(() => {
  fetch("/api/v1/items").then(r => r.json()).then(setData)
}, [])
```

### Local State: useState

Use `useState` for local, ephemeral UI state:

```typescript
function FilterPanel() {
  const [isExpanded, setIsExpanded] = useState(false)
  const [searchQuery, setSearchQuery] = useState("")
  // ...
}
```

### Derived Atoms

For computed values that depend on other atoms:

```typescript
const selectedIdAtom = Atom.make<string | null>(null)

const selectedItemAtom = Atom.make((get) => {
  const id = get(selectedIdAtom)
  if (id === null) return Effect.succeed(null)
  return Effect.gen(function* () {
    return yield* get.result(itemAtom(id))
  })
})
```

### Anti-Patterns

```typescript
// DON'T fetch data in useEffect
useEffect(() => {
  fetch("/api/v1/items").then(r => r.json()).then(setData)
}, [])

// DON'T use Effect.runPromise for data loading
const data = await Effect.runPromise(someEffect)

// DON'T store server data in useState
const [localItems, setLocalItems] = useState(items)  // Syncing nightmare

// DON'T manage loading/error state manually
const [loading, setLoading] = useState(false)
const [error, setError] = useState(null)
const [data, setData] = useState(null)
```

---

## Mutations & Forms

### Basic Mutation

```typescript
import { useAtomSet } from "@effect-atom/atom-react"
import { Exit, Cause } from "effect"

function CreateItemForm() {
  const createItem = useAtomSet(
    ApiClient.mutation("items", "createItem", {
      reactivityKeys: { items: true }  // Auto-invalidates item queries
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
      const err = Cause.squash(result.cause)
      const errorMessage = typeof err === "object" && "message" in err
        ? String(err.message)
        : "Failed to create item"
      setError(errorMessage)
      setIsSubmitting(false)
      return
    }

    // Queries with matching reactivityKeys auto-refresh
    navigate(`/items/${result.value.id}`)
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="name" className="block text-sm font-medium mb-1">
        Name
      </label>
      <input
        id="name"
        value={name}
        onChange={(e) => setName(e.target.value)}
        className="w-full border rounded-md px-3 py-2"
      />
      {error && <p className="text-red-600 text-sm mt-1">{error}</p>}
      <button
        type="submit"
        disabled={isSubmitting}
        className="mt-4 px-4 py-2 bg-blue-600 text-white rounded-md disabled:opacity-50"
      >
        {isSubmitting ? "Creating..." : "Create Item"}
      </button>
    </form>
  )
}
```

### Error Type Discrimination

Errors from the API have a `_tag` field for type discrimination:

```typescript
const err = Cause.squash(result.cause)

if (typeof err === "object" && "_tag" in err) {
  switch (err._tag) {
    case "ValidationError":
      setError(err.message)
      break
    case "ItemNotFoundError":
      setError("Item no longer exists")
      break
    default:
      setError("An unexpected error occurred")
  }
}
```

### Update with Specific Invalidation

```typescript
function EditItemForm({ item }: { item: Item }) {
  const updateItem = useAtomSet(
    ApiClient.mutation("items", "updateItem", {
      urlParams: { id: item.id },
      reactivityKeys: { items: [item.id] }  // Only invalidates this item
    }),
    { mode: "promiseExit" }
  )

  const handleSave = async () => {
    const result = await updateItem({ name })
    if (Exit.isSuccess(result)) {
      // Query for this item auto-refreshes
    }
  }
}
```

### Delete with Confirmation

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

  return (
    <button onClick={handleDelete} className="text-red-600 hover:text-red-700">
      Delete
    </button>
  )
}
```

---

## Styling with Tailwind

### Use Tailwind Classes Directly

```typescript
function Button({ children, variant = "primary" }: Props) {
  const baseClasses = "px-4 py-2 rounded-md font-medium transition-colors"
  const variantClasses = {
    primary: "bg-blue-600 text-white hover:bg-blue-700",
    secondary: "bg-gray-200 text-gray-800 hover:bg-gray-300",
    danger: "bg-red-600 text-white hover:bg-red-700"
  }

  return (
    <button className={`${baseClasses} ${variantClasses[variant]}`}>
      {children}
    </button>
  )
}
```

### Use clsx for Conditional Classes

```typescript
import { clsx } from "clsx"

function Button({ disabled, loading, className, children }: Props) {
  return (
    <button
      className={clsx(
        "px-4 py-2 rounded-md font-medium",
        "bg-blue-600 text-white hover:bg-blue-700",
        "disabled:opacity-50 disabled:cursor-not-allowed",
        loading && "animate-pulse",
        className
      )}
      disabled={disabled || loading}
    >
      {children}
    </button>
  )
}
```

---

## Component Patterns

### Composition Over Props

```typescript
// GOOD: Composition
<Card>
  <CardHeader><h2>Title</h2></CardHeader>
  <CardBody><p>Content</p></CardBody>
</Card>

// BAD: Prop drilling
<Card
  title="Title"
  content={<p>Content</p>}
  showHeader={true}
/>
```

---

## Page Templates

### List Page Template

```
+------------------------------------------+
|  [Breadcrumbs]                           |
|  Page Title                              |
+------------------------------------------+
|  [Filters/Search] [Primary Action Button]|
+------------------------------------------+
|  +------------------------------------+  |
|  | Data Table / Card Grid             |  |
|  | - Column headers                   |  |
|  | - Row actions                      |  |
|  | - Pagination                       |  |
|  +------------------------------------+  |
+------------------------------------------+
```

### Detail Page Template

```
+------------------------------------------+
|  [Breadcrumbs]                           |
|  Title [Status Badge]         [Actions]  |
+------------------------------------------+
|  +------------------------------------+  |
|  | Info Card                          |  |
|  | Key-value pairs in grid            |  |
|  +------------------------------------+  |
|                                          |
|  +------------------------------------+  |
|  | Related Data Section               |  |
|  | Table with row actions             |  |
|  +------------------------------------+  |
+------------------------------------------+
```

### Form Page Template

```
+------------------------------------------+
|  [Breadcrumbs]                           |
|  Create/Edit [Entity]                    |
+------------------------------------------+
|  +------------------------------------+  |
|  | Form Card                          |  |
|  | - Form fields with labels          |  |
|  | - Validation messages              |  |
|  | - [Cancel] [Submit]                |  |
|  +------------------------------------+  |
+------------------------------------------+
```

---

## Empty, Loading, and Error States

### Handling with Result.match

Effect Atom's `Result` type gives you all three states automatically:

```typescript
const result = useAtomValue(ApiClient.query("items", "listItems", {}))

return Result.match(result, {
  onInitial: () => <LoadingState />,
  onSuccess: (items) =>
    items.length === 0
      ? <EmptyState icon={Package} title="No items yet" action={<CreateButton />} />
      : <ItemsTable items={items} />,
  onFailure: (cause) => <ErrorState cause={cause} />
})
```

### Empty States

Every list page must have an empty state:

```typescript
function EmptyState({ icon: Icon, title, description, action }: Props) {
  return (
    <div className="flex flex-col items-center py-12">
      <div className="rounded-full bg-gray-100 p-4 mb-4">
        <Icon className="h-8 w-8 text-gray-400" />
      </div>
      <h3 className="text-lg font-medium text-gray-900 mb-2">{title}</h3>
      <p className="text-gray-500 text-center max-w-sm mb-6">{description}</p>
      {action}
    </div>
  )
}
```

### Loading States (via Result.onInitial)

```typescript
function TableSkeleton({ rows = 5 }: { rows?: number }) {
  return (
    <div className="space-y-3">
      {Array.from({ length: rows }).map((_, i) => (
        <div key={i} className="h-16 bg-gray-100 rounded animate-pulse" />
      ))}
    </div>
  )
}
```

### Error States (via Result.onFailure)

```typescript
function ErrorState({ cause, onRetry }: { cause: Cause.Cause<unknown>; onRetry?: () => void }) {
  return (
    <div className="flex flex-col items-center py-12">
      <h3 className="text-lg font-medium text-gray-900 mb-2">
        Something went wrong
      </h3>
      <p className="text-gray-500 mb-6">{Cause.pretty(cause)}</p>
      {onRetry && <button onClick={onRetry}>Try again</button>}
    </div>
  )
}
```

---

## Form Design

### Labels and Placeholders

Labels are required for accessibility:

```typescript
<label htmlFor="item-name" className="block text-sm font-medium mb-1">
  Item Name
</label>
<input id="item-name" placeholder="e.g., My Item" />
```

### Validation

- Validate on blur, not on change
- Show inline errors next to fields
- Don't clear form on failed submission

### Prevent Double Submission

```typescript
const [isSubmitting, setIsSubmitting] = useState(false)

const handleSubmit = async () => {
  if (isSubmitting) return
  setIsSubmitting(true)
  try {
    await submitForm()
  } finally {
    setIsSubmitting(false)
  }
}

<Button disabled={isSubmitting}>
  {isSubmitting ? "Submitting..." : "Submit"}
</Button>
```

---

## Navigation & Layout

### AppLayout

ALL authenticated pages use AppLayout with sidebar and header:

```typescript
function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex flex-1 flex-col">
        <Header />
        <main className="flex-1 overflow-y-auto p-6">
          {children}
        </main>
      </div>
    </div>
  )
}
```

### Breadcrumbs

Use the shared Breadcrumbs component for pages 2+ levels deep:

```typescript
<Breadcrumbs items={[
  { label: "Items", href: "/items" },
  { label: itemName }
]} />
```

---

## Accessibility

### Keyboard Navigation

All interactive elements must be keyboard accessible:

```typescript
<div
  role="button"
  tabIndex={0}
  onKeyDown={(e) => {
    if (e.key === "Enter" || e.key === " ") {
      e.preventDefault()
      onClick()
    }
  }}
  onClick={onClick}
>
```

### ARIA Labels

```typescript
<button aria-label="Close modal">
  <X className="h-5 w-5" />
</button>

<div aria-live="polite" aria-busy={isLoading}>
  {isLoading ? "Loading..." : content}
</div>
```

### Touch Targets

Minimum touch target size is 44x44 pixels:

```typescript
<button className="min-h-[44px] min-w-[44px] p-2">
  <Icon />
</button>
```

---

## Design System Reference

### Color Palette

| Purpose | Class | Usage |
|---------|-------|-------|
| Primary | `bg-blue-600` | Primary buttons, active states |
| Success | `bg-green-100 text-green-800` | Completed, approved |
| Warning | `bg-yellow-100 text-yellow-800` | Pending review |
| Error | `bg-red-100 text-red-800` | Failed, rejected |

### Typography

| Level | Class | Usage |
|-------|-------|-------|
| Page Title | `text-2xl font-bold` | Main headings |
| Section Title | `text-xl font-semibold` | Section headings |
| Card Header | `text-lg font-semibold` | Card titles |
| Body | `text-base` | Default text |
| Small | `text-sm` | Secondary text |
| Caption | `text-xs` | Badges, table headers |

### Spacing

| Token | Value | Usage |
|-------|-------|-------|
| `gap-2` | 8px | Standard internal spacing |
| `gap-4` | 16px | Section spacing |
| `p-6` | 24px | Card padding |

### Buttons

| Variant | Usage |
|---------|-------|
| `primary` | Main actions (Save, Create) |
| `secondary` | Secondary actions (Cancel) |
| `danger` | Destructive actions (Delete) |
| `ghost` | Tertiary actions |

---

## Quick Reference Checklist

Before marking any UI task complete:

### Data Loading
- [ ] Uses `useAtomValue` with AtomHttpApi (not useEffect/fetch)
- [ ] `Result.match` handles all three states (initial, success, failure)
- [ ] `reactivityKeys` set on queries and mutations

### Navigation
- [ ] Breadcrumbs on nested pages
- [ ] Active route highlighted

### Forms
- [ ] First field auto-focused
- [ ] Labels are clickable (htmlFor)
- [ ] Validation on blur with inline errors
- [ ] Submit button shows loading state
- [ ] Button disabled during submission
- [ ] Mutation uses `reactivityKeys` for auto-invalidation

### States
- [ ] Empty state with illustration + CTA
- [ ] Loading state via `Result.onInitial`
- [ ] Error state via `Result.onFailure` with WHAT/WHY/action

### Accessibility
- [ ] All elements keyboard navigable
- [ ] Focus indicators visible
- [ ] Touch targets 44px minimum
- [ ] ARIA labels on icon buttons
