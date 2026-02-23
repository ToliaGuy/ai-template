# Project Template - Claude Code Guide

This document provides guidance for Claude Code when working on projects based on this template.

## Project Overview

This is a full-stack TypeScript application using:
- **Convex** - Backend: database, server functions, and HTTP actions
- **Clerk** - Authentication: user sign-up, sign-in, session management
- **Vite + React** - Frontend: fast dev server, simple SPA
- **Effect HttpApi** - Type-safe HTTP API definition (HttpApiGroup, HttpApiEndpoint)
- **Effect Atom + AtomHttpApi** - Reactive frontend data loading from the HTTP API
- **shadcn/ui** - UI component library (Radix UI + Tailwind CSS)
- **Tailwind CSS** - Utility-first styling

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      FRONTEND                                │
│  Vite + React + shadcn/ui + Effect Atom + AtomHttpApi        │
│  Data loading via Effect Atom (useAtomValue, Result.match)   │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP API (Effect HttpApi)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       CONVEX BACKEND                         │
│  Convex functions + Effect HttpApi definitions                │
│  Database, server logic - all handled by Convex               │
│  Authentication via Clerk (JWT validation in Convex functions) │
└─────────────────────────────────────────────────────────────┘
```

## CRITICAL: BACKEND AND FRONTEND MUST STAY ALIGNED

**This is a HARD REQUIREMENT. When implementing features:**

1. **NEVER do frontend-only changes** for features that need backend work
2. **ALWAYS update both layers** - frontend AND Convex functions
3. **Frontend workarounds are NOT acceptable** - if the spec says "update API", update the API
4. **Run tests** - `pnpm test && pnpm typecheck` MUST pass before marking work complete

**Data flow**: Frontend (Effect Atom) → HTTP API → Convex
**All layers must be consistent.**

## Project Structure

```
project/
├── src/                 # Frontend (Vite + React)
│   ├── api/             # AtomHttpApi client setup
│   ├── atoms/           # Atom definitions (queries, mutations)
│   ├── components/      # React components
│   ├── routes/          # Page components
│   └── styles/          # Tailwind CSS
├── convex/              # Convex backend
│   ├── schema.ts        # Database schema
│   ├── functions/       # Convex queries, mutations, actions
│   └── http.ts          # HTTP API routes
├── specs/               # All specs and context
│   ├── guides/          # Consolidated how-to guides (START HERE)
│   ├── architecture/    # System architecture documentation
│   ├── pending/         # Features not yet implemented
│   ├── completed/       # Historical reference (completed work)
│   └── reference/       # External references
└── repos/               # Reference repositories (git subtrees)
    ├── effect/          # Effect-TS source (includes HttpApi, Atom)
    └── effect-atom/     # Effect Atom state management
```

## Specs Directory

All specifications and context documentation live in `specs/`.

### Consolidated Guides (Start Here)

| File | Description |
|------|-------------|
| [specs/guides/effect-guide.md](specs/guides/effect-guide.md) | Effect patterns, layers, errors, testing |
| [specs/guides/testing-guide.md](specs/guides/testing-guide.md) | Unit, integration, E2E testing |
| [specs/guides/frontend-guide.md](specs/guides/frontend-guide.md) | React, Vite, Effect Atom, UI, styling |
| [specs/guides/api-guide.md](specs/guides/api-guide.md) | HttpApi definitions, AtomHttpApi client |

### Reference

| File | Description |
|------|-------------|
| [specs/reference/reference-repos.md](specs/reference/reference-repos.md) | Reference repository documentation |

---

## Quick Start: Critical Rules

### Backend (Convex + Clerk)

1. **Define schema** in `convex/schema.ts`
2. **Use Convex functions** for all data operations (queries, mutations, actions)
3. **Use Effect HttpApi** to define typed HTTP endpoints
4. **Use Schema.TaggedError** for all domain errors
5. **Use Clerk for authentication** - `ctx.auth.getUserIdentity()` in Convex functions, see [api-guide.md](specs/guides/api-guide.md#authentication--middleware)

### Frontend (Vite + React + Effect Atom)

**Use Effect Atom + AtomHttpApi for all data loading.** Key patterns:

1. **Use `AtomHttpApi.Tag`** - define typed HTTP API client
2. **Use `useAtomValue`** - subscribe to query results (returns `Result<A, E>`)
3. **Use `useAtomSet`** - trigger mutations
4. **Use `Result.match`** - handle loading/success/failure states
5. **Use `Atom.family`** - parameterized atoms (e.g., per-ID queries)
6. **Use `useState` for UI** - local form state, modals, toggles
7. **Use shadcn/ui components** - for all base UI (buttons, inputs, dialogs, tables, cards, etc.)
8. **Use Tailwind CSS** - no inline styles, use `cn()` for conditional classes

### Effect Code Rules

**Read [specs/guides/effect-guide.md](specs/guides/effect-guide.md) first.** Key rules:

1. **NEVER use `any` or type casts** - use Schema.make(), decodeUnknown, identity
2. **NEVER use global `Error`** - use Schema.TaggedError for all domain errors
3. **NEVER use `catchAllCause`** - it catches defects (bugs); use catchAll or mapError
4. **NEVER use `disableValidation: true`** - banned by lint rule
5. **NEVER use `*FromSelf` schemas** - use standard variants
6. **NEVER use Sync variants** - use Schema.decodeUnknown not decodeUnknownSync
7. **NEVER create index.ts barrel files** - import from specific modules

### UI Architecture (CRITICAL)

**Read [specs/guides/frontend-guide.md](specs/guides/frontend-guide.md) for all UI work.** Key rules:

1. **ALL pages use AppLayout** - sidebar + header on EVERY authenticated page
2. **Consistent page templates** - use List, Detail, Form page patterns from spec
3. **Empty states required** - every list page needs an empty state with CTA

---

## Quick Reference Commands

```bash
# Development
pnpm dev                # Start Vite dev server + Convex
pnpm build              # Build for production

# Testing
pnpm test               # Run unit/integration tests (vitest)
pnpm test:verbose       # Run tests with full output
pnpm test:e2e           # Run Playwright E2E tests

# Code Quality
pnpm typecheck          # Check TypeScript types
pnpm lint               # Run ESLint
pnpm format             # Format code with Prettier

# Maintenance
pnpm clean              # Clean build outputs
```

---

## Implementation Guidelines

### Backend (Convex)

Convex handles all server-side concerns: database, auth, server logic.

**Guidelines:**
1. **Define schemas** with Convex schema builder
2. **Use Effect HttpApi** to define typed HTTP endpoint contracts
3. **Use Schema.TaggedError** for domain errors with HTTP status annotations
4. **Use branded types** for IDs
5. **Write tests** alongside implementation

### Frontend (Vite + React)

**Use Effect Atom + AtomHttpApi** for all data loading.

**Read these specs:**
- [specs/guides/frontend-guide.md](specs/guides/frontend-guide.md) - React patterns, Effect Atom, UI
- [specs/guides/api-guide.md](specs/guides/api-guide.md) - AtomHttpApi client patterns

**Guidelines:**
1. **Use AtomHttpApi for data loading** - `useAtomValue` for queries, `useAtomSet` for mutations
2. **Use `Result.match`** for loading/success/failure UI states
3. **Use `Atom.family`** for parameterized queries (per-ID, per-filter)
4. **Use `reactivityKeys`** for automatic cache invalidation after mutations
5. **Handle empty states** - show helpful messages when no data
6. **All pages use AppLayout** with Sidebar and Header
7. **Use Tailwind CSS** - consistent spacing, colors, typography

### Full Stack Features

When implementing a feature that spans layers:

1. **Convex schema** - define tables
2. **Convex functions** - queries, mutations, actions
3. **HttpApi definition** - typed endpoint contracts
4. **Frontend** - pages, components, AtomHttpApi queries/mutations
