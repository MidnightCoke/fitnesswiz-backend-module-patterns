# Backend Module Pattern Guide

## Module Structure (5 files per module)

Each module is located in `src/modules/{moduleName}/` with exactly 5 files:

1. **{module}.routes.ts** - Route registration
2. **{module}.handler.ts** - HTTP request handlers
3. **{module}.service.ts** - Business logic
4. **{module}.repo.ts** - Database queries (optional for some modules)
5. **{module}.schema.ts** - Zod validation schemas & TypeScript types

## File Patterns

### Routes File Pattern
```typescript
import type { Router } from "../../lib/router";
import { {module}Handler } from "./{module}.handler";
import { requireAuth } from "../../middlewares/auth";
import { requireAdminAuth } from "../../middlewares/admin";

export function register{Module}Routes(router: Router) {
  router.get("/path", requireAuth((req, userId) => handler.method(req, userId)));
  router.post("/path", requireAdminAuth((req, adminUserId, params) => handler.method(req, params)));
}
```

### Handler File Pattern
```typescript
import { json, error } from "../../lib/response";
import { handleError } from "../../lib/errors";
import { {module}Service } from "./{module}.service";
import { {schema} } from "./{module}.schema";
import type { RouteParams } from "../../middlewares/auth";

export const {module}Handler = {
  async methodName(req: Request, userId?: number, params?: RouteParams) {
    try {
      const body = await req.json();
      const validatedData = schema.parse(body);
      const result = await {module}Service.methodName(validatedData);
      return json(result, 201);
    } catch (err) {
      const { message, statusCode } = handleError(err);
      return error(message, statusCode);
    }
  }
};
```

### Service File Pattern
```typescript
import { {module}Repository } from "./{module}.repo";
import { NotFoundError, ValidationError } from "../../lib/errors";

export class {Module}Service {
  async methodName(data: InputType) {
    const result = await {module}Repository.method(data);
    if (!result) throw new NotFoundError("Not found");
    return result;
  }
}

export const {module}Service = new {Module}Service();
```

### Repository File Pattern
```typescript
import prisma from "../../lib/prisma";

export class {Module}Repository {
  async findAll(page?: number, limit?: number) {
    return prisma.{model}.findMany({
      include: { /* relations */ },
      skip: (page - 1) * limit,
      take: limit
    });
  }

  async findById(id: number) {
    return prisma.{model}.findUnique({ where: { id } });
  }
}

export const {module}Repository = new {Module}Repository();
```

### Schema File Pattern
```typescript
import { z } from "zod";

export const createSchema = z.object({
  field: z.string().min(1, "Required")
});

export const updateSchema = z.object({
  field: z.string().optional()
});

export type CreateInput = z.infer<typeof createSchema>;
export type UpdateInput = z.infer<typeof updateSchema>;
export interface Response { /* ... */ }
```

## Core Patterns

### Module Registration (app.ts)
All modules registered in one place:
```typescript
import { register{Module}Routes } from "./modules/{module}/{module}.routes";
register{Module}Routes(router);
```

### Authentication Patterns
- **`requireAuth(handler)`** - Standard user auth (userId required)
- **`requireAdminAuth(handler)`** - Admin-only (checks admin role in DB + JWT)

### Error Handling
- Use custom error classes: `ValidationError`, `NotFoundError`, `UnauthorizedError`, `ConflictError`
- Handler template: `const { message, statusCode } = handleError(err);`
- Uses `ErrorTypes` enum for predefined messages

### Response Patterns
- Success: `json(data, statusCode)` - default 200, can specify 201
- Error: `error(message, statusCode)`
- Both accept optional `headers` in options

### Validation Pattern
- Import `{schema}` from `./{module}.schema`
- Validate: `const data = schema.parse(body)`
- Zod handles both validation + type inference

### Caching Pattern
```typescript
const cached = cache.get(key);
if (cached) return cached;
const result = await repo.method();
cache.set(key, result, CACHE.TTL_SECONDS);
```

### Database Transactions
```typescript
await prisma.$transaction(async (tx) => {
  await tx.model.create({ data });
  await tx.model.update({ data });
});
```

## Key Infrastructure Files

- **src/lib/router.ts** - Custom Router (no Express) with pattern matching
- **src/lib/prisma.ts** - Prisma client singleton
- **src/lib/response.ts** - json() & error() helpers with BigInt support
- **src/lib/validation.ts** - Validation utility functions
- **src/lib/errors.ts** - Custom error classes + handleError()
- **src/middlewares/auth.ts** - requireAuth() wrapper
- **src/middlewares/admin.ts** - requireAdminAuth() wrapper (JWT + DB check)
- **src/config/env.ts** - Environment variables with validation
- **src/config/constants.ts** - HTTP status codes, size limits, timeouts, cache TTLs

## Prisma Patterns

- Use `select` for field projection, not in routes/handlers
- Use `include` in repository for relations
- `@relation("name")` for back-references
- `onDelete: Cascade` for cleanup
- `@unique`, `@index` for DB optimization
