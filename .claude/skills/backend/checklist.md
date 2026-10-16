# Backend Implementation Checklist

## Core Checklist
- [ ] Checked existing tables/APIs via git before creating new ones
- [ ] Models added to `prisma/schema.prisma` and migration created
- [ ] Migration applies cleanly on an empty database (`npx prisma migrate reset` locally)
- [ ] Server-side authorization (session + role/ownership) in EVERY route and Server Action
- [ ] Indexes created on performance-critical columns
- [ ] Foreign keys set with appropriate ON DELETE behavior
- [ ] All planned API endpoints implemented in `/src/app/api/`
- [ ] Authentication verified (no access without valid session, wrong role gets 403)
- [ ] Input validation with Zod on all POST/PUT requests
- [ ] Meaningful error messages with correct HTTP status codes
- [ ] No TypeScript errors in API routes
- [ ] All endpoints tested manually
- [ ] No hardcoded secrets in source code
- [ ] Frontend connected to real API endpoints
- [ ] User has reviewed and approved

## Verification (run before marking complete)
- [ ] `npm run build` passes without errors
- [ ] All acceptance criteria from feature spec addressed in API
- [ ] All API endpoints return correct status codes (test with curl or browser)
- [ ] `features/INDEX.md` status updated to "In Progress"
- [ ] Code committed to git

## Performance Checklist
- [ ] All frequently filtered columns have indexes
- [ ] No N+1 queries (use Prisma `include`/`select` instead of loops)
- [ ] All list queries use `take` / pagination
- [ ] Zod validation on all write endpoints
- [ ] Slow queries cached where appropriate (optional for MVP)
- [ ] Rate limiting on public-facing APIs (optional for MVP)
