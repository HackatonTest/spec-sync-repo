---
repo: user-service
spec_type: functional
commit: 4054b55114e1fe16363d8c451cd14db54fc695b0
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 1154734297395299feeb8345ae4561410ebf39ed7eff5a65094d69feb6a3876d
generated_at: 2026-06-30T17:04:50.789336815+02:00
generator: specsync
---

## Business Purpose

The `user-service` provides centralised user management, authentication, and role-based authorisation for the SpecSync platform. It issues and validates JWT tokens so that other services (or API consumers) can authenticate users without maintaining their own identity stores. The service exposes a self-service registration/login flow for end-users and a privileged administration API for managing user accounts.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Identity & Access Management (IAM).
- **Core aggregate:** `User` — owns `id`, `username`, `email`, `password` (BCrypt hash), `role` (`USER` | `ADMIN`), `createdAt`, `updatedAt`.
- **Supporting value objects:** `Role` enum (`USER`, `ADMIN`); JWT token (issued but not persisted).
- **Upstream context:** None detected — this service is a root identity provider.
- **Downstream contexts:** Any service that accepts the JWT issued here is a downstream consumer; no explicit downstream integrations are visible in this snapshot (no events, no outbound HTTP clients).

## Use Cases / User Stories

- **As an anonymous visitor**, I want to register a new account (`POST /api/auth/register`) so that I can obtain a JWT token and access protected resources.
- **As a registered user**, I want to log in with my credentials (`POST /api/auth/login`) so that I receive a fresh JWT token for subsequent requests.
- **As an ADMIN**, I want to list all registered users (`GET /api/users`) so that I can audit the user base.
- **As an ADMIN**, I want to retrieve a specific user by ID (`GET /api/users/{id}`) so that I can inspect their details.
- **As an ADMIN**, I want to update a user's email address and/or role (`PUT /api/users/{id}`) so that I can correct data or promote/demote users.
- **As an ADMIN**, I want to permanently delete a user account (`DELETE /api/users/{id}`) so that I can remove stale or unauthorised accounts.

## Business Rules

- **Username uniqueness:** `username` must be unique across all users; duplicate registration returns HTTP 409.
- **Email uniqueness:** `email` must be unique across all users; duplicate registration or update returns HTTP 409.
- **Username format:** Must be between 3 and 50 characters and not blank (`@NotBlank`, `@Size(min=3, max=50)`).
- **Email format:** Must be a syntactically valid e-mail address (`@Email`); required on registration, optional but validated on update.
- **Password length:** Must be at least 6 characters on registration (`@Size(min=6)`).
- **Password storage:** Passwords are stored as BCrypt hashes; plaintext is never persisted.
- **Default role:** Newly registered users receive the `USER` role unless explicitly set otherwise. (inferred — the `User` entity defaults `role` to `USER` and the registration flow does not accept a role field from the caller.)
- **JWT expiry:** Tokens are valid for exactly 24 hours from issuance; expired tokens are rejected.
- **JWT algorithm:** Tokens are signed with HMAC-SHA256 using a configurable secret of at least 32 characters.
- **Stateless sessions:** No server-side session is maintained; every request must carry a valid `Authorization: Bearer <token>` header for protected endpoints.
- **Admin-only user management:** All `/api/users/**` endpoints require the `ADMIN` role (enforced via `@PreAuthorize("hasRole('ADMIN')")`); a valid JWT without the ADMIN role returns HTTP 403.
- **Public auth endpoints:** `/api/auth/**` is fully accessible without authentication.
- **Immutable creation timestamp:** `createdAt` is set on first persist and never updated (`updatable = false`).
- **Updatable fields:** Only `email` and `role` can be changed via the update endpoint; `username` and `password` are not exposed for update. (inferred — based on `UpdateUserRequest` containing only `email` and `role`.)
- **Not-found handling:** Requests referencing a non-existent user ID return HTTP 404.
- **Invalid credential handling:** A failed login attempt returns HTTP 401.
