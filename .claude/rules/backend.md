---
paths:
  - "src/app/api/**"
  - "src/lib/**"
  - "prisma/**"
---

# Backend Development Rules

## Database (MariaDB + Prisma)
- Schema lives in `prisma/schema.prisma`; every change goes through a migration (`npx prisma migrate dev --name <change>`), never manual DDL on a shared database
- Production applies migrations with `npx prisma migrate deploy` only
- Add indexes (`@@index`) on columns used in WHERE, ORDER BY, and JOIN clauses
- Use foreign keys with explicit `onDelete` behavior (Cascade only where data truly belongs to the parent)
- Money: `Decimal` / `DECIMAL(p,2)`, never Float
- Use `utf8mb4` and store timestamps in UTC

## Authorization (no RLS in MariaDB)
- MariaDB has no Row Level Security: every API route and Server Action MUST check session AND role/ownership on the server
- Centralize checks in helpers (e.g. `requireUser()`, `requireRole('admin')`) and call them first in every handler
- Scope every query by the allowed owner/tenant (`where: { userId }`) - never fetch first and filter in JS
- UI hiding is not access control

## API Routes / Server Actions
- Validate all inputs using Zod schemas before processing
- Return meaningful error messages with appropriate HTTP status codes (400, 401, 403, 404, 409)
- Use `take` (limit) and pagination on all list queries

## Query Patterns
- Use Prisma `include`/`select` or a single joined query instead of N+1 loops
- Use transactions (`prisma.$transaction`) for multi-step writes
- Reuse one PrismaClient instance (`src/lib/db.ts`), do not create clients per request
- Use `unstable_cache` from Next.js for rarely-changing data

## Security
- Never hardcode secrets in source code
- Use environment variables for all credentials (`DATABASE_URL` etc.)
- Validate and sanitize all user input
- Raw SQL only via `prisma.$queryRaw` tagged templates (parameterized), never `$queryRawUnsafe` with user input
- The app's DB user gets only the privileges it needs on its own schema (no root)
