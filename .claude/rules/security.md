---
paths:
  - "src/app/api/**"
  - "src/lib/**"
  - ".env*"
  - "prisma/**"
  - "next.config.*"
---

# Security Rules

## Secrets Management
- NEVER commit secrets, API keys, or credentials to git
- Use `.env.local` for local development (already in .gitignore)
- In production, set secrets as Windows environment variables of the service account / service definition, not in files inside the app folder
- Use `NEXT_PUBLIC_` prefix ONLY for values safe to expose in browser
- Document all required env vars in `.env.local.example` with dummy values

## Input Validation
- Validate ALL user input on the server side with Zod
- Never trust client-side validation alone
- Sanitize data before database insertion

## Authentication & Authorization
- Use an established auth library (Better Auth) - never hand-roll password hashing or session handling
- Passwords hashed with a slow hash (scrypt/argon2, library default)
- Session cookies: HttpOnly, Secure, SameSite=Lax
- Always verify session AND role before processing API requests or Server Actions (no RLS as second line of defense)
- Implement rate limiting and lockout on authentication endpoints
- Log security-relevant events (login, failed login, role changes, deletes) in an audit log

## Security Headers
- X-Frame-Options: DENY
- X-Content-Type-Options: nosniff
- Referrer-Policy: origin-when-cross-origin
- Strict-Transport-Security with includeSubDomains
- Set them in `next.config.*` headers; IIS must not strip them

## Code Review Triggers
- Any changes to authorization helpers or role checks require explicit user approval
- Any changes to authentication flow require explicit user approval
- Any new environment variables must be documented in .env.local.example
