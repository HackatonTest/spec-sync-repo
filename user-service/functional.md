---
repo: user-service
spec_type: functional
commit: 6556545879f4611dbd7eeec85ff3a878c61c64ef
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: df91828eb6b72b181ecfa3b174acca587720ed81320134be9494d4e16c8c8e22
generated_at: 2026-06-30T17:15:42.037590538+02:00
generator: specsync
---

## Business Purpose

This service provides user management, authentication, and authorization capabilities for the SpecSync platform. It handles user self-registration and login via JWT-based stateless authentication, and exposes administrative operations (listing, searching, updating, and deleting users) restricted to privileged roles. It exists as a dedicated identity/access boundary so that other services can delegate authentication and user lifecycle management to a single source of truth.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Identity & Access Management (IAM)
- **Core aggregate:** `User` — owns username, email, hashed password, role, status, and audit timestamps (`createdAt`, `updatedAt`, `lastLoginAt`)
- **Enumerations owned:** `Role` (`USER`, `ADMIN`), `UserStatus` (`ACTIVE`, `INACTIVE`, `SUSPENDED`)
- **Neighbouring contexts:** No explicit upstream/downstream event integration is present (no messaging topics detected). Other services are expected to validate JWTs issued by this service; this service is therefore an upstream identity provider to any downstream consumer of those tokens.

## Use Cases / User Stories

- **As an anonymous user**, I want to register with a username, email, and password (`POST /api/auth/register`) so that I can obtain a JWT and access protected resources.
- **As a registered user**, I want to log in with my username and password (`POST /api/auth/login`) so that I receive a JWT token to authenticate subsequent requests.
- **As an ADMIN**, I want to list all users (`GET /api/users`) so that I can review every account in the system.
- **As an ADMIN**, I want to search and filter users by a query string, status, and/or role with pagination (`GET /api/users/search`) so that I can efficiently locate specific accounts.
- **As an ADMIN**, I want to retrieve all users in a given status (`GET /api/users/status/{status}`) so that I can act on a specific cohort (e.g., all SUSPENDED accounts).
- **As an ADMIN**, I want to view aggregate user statistics (`GET /api/users/stats`) so that I can monitor platform growth and account health.
- **As an ADMIN**, I want to retrieve a specific user by ID (`GET /api/users/{id}`) so that I can inspect individual account details.
- **As an ADMIN**, I want to update a user's email and/or role (`PUT /api/users/{id}`) so that I can correct account data or elevate/demote privileges.
- **As an ADMIN**, I want to change a user's status (`PATCH /api/users/{id}/status`) so that I can activate, deactivate, or suspend accounts.
- **As an ADMIN or the account owner**, I want to change a user's password (`PATCH /api/users/{id}/password`) after providing the current password so that account credentials can be updated securely.
- **As an ADMIN**, I want to permanently delete a user (`DELETE /api/users/{id}`) so that stale or invalid accounts can be removed.

## Business Rules

- **Username uniqueness:** Usernames must be unique across all users; duplicate registration returns HTTP 409. (enforced by DB `UNIQUE` constraint and service layer)
- **Email uniqueness:** Email addresses must be unique across all users; duplicate registration returns HTTP 409. (enforced by DB `UNIQUE` constraint and service layer)
- **Username length:** Must be between 3 and 50 characters, non-blank.
- **Email format:** Must be a well-formed email address (`@Email` validation on both registration and update requests).
- **Registration password minimum length:** At least 6 characters.
- **Password change minimum length:** New password must be at least 8 characters.
- **Current password verification:** A password change requires the caller to supply the correct current password; failure returns HTTP 401.
- **Default role on registration:** New users are assigned role `USER` unless explicitly overridden. (inferred — `Role.USER` is the `@Builder.Default`)
- **Default status on creation:** New users are created with status `ACTIVE`. (inferred — `UserStatus.ACTIVE` is the `@Builder.Default`)
- **Passwords are BCrypt-hashed:** Plaintext passwords are never stored; BCryptPasswordEncoder is configured as the sole `PasswordEncoder`.
- **Stateless sessions:** No server-side session is maintained; every request must carry a valid JWT (`SessionCreationPolicy.STATELESS`).
- **JWT signing algorithm:** HS256 with a secret key of at least 32 characters.
- **Admin-only user management:** All `/api/users/**` endpoints (except `PATCH /{id}/password` for the account owner) require the `ADMIN` role; non-admin authenticated users receive HTTP 403.
- **Password self-service:** A user may change their own password (`PATCH /api/users/{id}/password`) in addition to admins — enforced via `#id == authentication.principal.id` SpEL expression.
- **Valid status values:** The `UserStatus` field accepts only `ACTIVE`, `INACTIVE`, or `SUSPENDED`; null status on an update request returns HTTP 400.
- **Immutable creation timestamp:** `createdAt` is set on persist and is non-updatable.
- **`lastLoginAt` updated on login:** (inferred — field exists on the entity and is surfaced in `UserDto`; service is expected to record it at login time.)
- **Public endpoints:** `/api/auth/**`, Swagger UI, and H2 console paths are permit-all; all other requests require authentication.
- **H2 console restricted to localhost:** `spring.h2.console.settings.web-allow-others=false` prevents remote access to the database console.
