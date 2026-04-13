# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Project: Acme API

## Commands
npm run dev          # Start dev server
npm run test         # Run tests (Jest)
npm run lint         # ESLint + Prettier check
npm run build        # Production build

## Architecture
- Express REST API, Node 20
- PostgreSQL via Prisma ORM
- All handlers live in src/handlers/
- Shared types in src/types/

## Conventions
- Use zod for request validation in every handler
- Return shape is always { data, error }
- Never expose stack traces to the client
- Use the logger module, not console.log

## Watch out for
- Tests use a real local DB, not mocks. Run `npm run db:test:reset` first
- Strict TypeScript: no unused imports, ever

---

# API and Interface Design

Design stable, well-documented interfaces that are hard to misuse. Good interfaces make the right thing easy and the wrong thing hard. Applies to REST APIs, module boundaries, component props, and any surface where one piece of code talks to another.

## Core Principles

### Hyrum's Law

Every public behavior — including undocumented quirks, error message text, timing, and ordering — becomes a de facto contract once users depend on it.

- Be intentional about what you expose. Every observable behavior is a potential commitment.
- Don't leak implementation details. If users can observe it, they will depend on it.
- Plan for deprecation at design time.

### The One-Version Rule

Avoid forcing consumers to choose between multiple versions of the same API. Extend rather than fork. Design for a world where only one version exists at a time.

## Patterns

### Contract First

Define the interface before implementing it. The contract is the spec — implementation follows.

```typescript
interface TaskAPI {
  createTask(input: CreateTaskInput): Promise<Task>;
  listTasks(params: ListTasksParams): Promise<PaginatedResult<Task>>;
  getTask(id: string): Promise<Task>;          // throws NotFoundError if missing
  updateTask(id: string, input: UpdateTaskInput): Promise<Task>;  // partial update
  deleteTask(id: string): Promise<void>;        // idempotent
}
```

### Consistent Error Shape

Every error response uses the same shape:

```typescript
interface APIError {
  error: {
    code: string;     // "VALIDATION_ERROR"
    message: string;  // "Email is required"
    details?: unknown;
  };
}
// 400 invalid data | 401 unauthenticated | 403 unauthorized | 404 not found
// 409 conflict | 422 validation failed | 500 server error (never expose internals)
```

### Validate at Boundaries Only

Validate at system edges (user input, external API responses, env vars). Trust internal code and framework guarantees. Third-party API responses are untrusted — always validate their shape before use.

### Prefer Addition Over Modification

New fields must be optional and additive. Never change existing field types or remove fields.

### REST Resource Design

```
GET    /api/tasks              → List (use query params for filtering + pagination)
POST   /api/tasks              → Create
GET    /api/tasks/:id          → Get one
PATCH  /api/tasks/:id          → Partial update
DELETE /api/tasks/:id          → Delete (idempotent)
GET    /api/tasks/:id/comments → Sub-resource list
```

Pagination shape: `{ data: [...], pagination: { page, pageSize, totalItems, totalPages } }`

### Naming Conventions

| Target | Convention | Example |
|---|---|---|
| REST endpoints | Plural nouns, no verbs | `GET /api/tasks` |
| Query params | camelCase | `?sortBy=createdAt` |
| Response fields | camelCase | `{ createdAt, taskId }` |
| Boolean fields | is/has/can prefix | `isComplete` |
| Enum values | UPPER_SNAKE | `"IN_PROGRESS"` |

### TypeScript Patterns

Use discriminated unions for variants with different shapes. Separate Input types (caller-provided) from Output types (includes server-generated fields like `id`, `createdAt`). Use branded types for IDs to prevent mixing `TaskId` and `UserId`.

## Red Flags

- Endpoints that return different shapes depending on conditions
- Inconsistent error formats across endpoints
- Validation scattered inside internal functions instead of at boundaries
- Breaking changes to existing fields (type changes, removals)
- List endpoints without pagination
- Verbs in REST URLs (`/api/createTask`)
- Third-party API responses used without validation

## Checklist

- [ ] Every endpoint has typed input and output schemas
- [ ] All error responses share one consistent format
- [ ] Validation only at system boundaries
- [ ] List endpoints support pagination
- [ ] New fields are optional and additive
- [ ] Naming is consistent across all endpoints