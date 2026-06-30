---
repo: user-service
spec_type: behavioral
commit: 4054b55114e1fe16363d8c451cd14db54fc695b0
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 1154734297395299feeb8345ae4561410ebf39ed7eff5a65094d69feb6a3876d
generated_at: 2026-06-30T17:04:50.789336815+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST over HTTP. Content type: `application/json`. Service listens on port `8081`. OpenAPI specification available at `/v3/api-docs`; interactive UI at `/swagger-ui.html`.

### Authentication endpoints — public (no JWT required)

| Method | Path | Purpose | Request Body | Response Body |
|--------|------|---------|--------------|---------------|
| `POST` | `/api/auth/register` | Register a new user account | `RegisterRequest` (see below) | `201 Created` → `AuthResponse`; `400` on validation failure; `409` if username/email already exists |
| `POST` | `/api/auth/login` | Authenticate and obtain JWT | `LoginRequest` (see below) | `200 OK` → `AuthResponse`; `401` on bad credentials |

### User-management endpoints — require `ADMIN` role (Bearer JWT)

| Method | Path | Purpose | Request Body | Response Body |
|--------|------|---------|--------------|---------------|
| `GET` | `/api/users` | List all registered users | _(none)_ | `200 OK` → `UserDto[]`; `401`; `403` |
| `GET` | `/api/users/{id}` | Retrieve a single user by ID | _(none)_ | `200 OK` → `UserDto`; `401`; `403`; `404` |
| `PUT` | `/api/users/{id}` | Update a user's email and/or role | `UpdateUserRequest` (see below) | `200 OK` → `UserDto`; `400`; `401`; `403`; `404`; `409` |
| `DELETE` | `/api/users/{id}` | Permanently delete a user | _(none)_ | `204 No Content`; `401`; `403`; `404` |

### DTO field reference

**`RegisterRequest`**

| Field | Type | Constraints |
|-------|------|-------------|
| `username` | `String` | `@NotBlank`, length 3–50 |
| `email` | `String` | `@NotBlank`, valid e-mail format |
| `password` | `String` | `@NotBlank`, min length 6 |

**`LoginRequest`**

| Field | Type | Constraints |
|-------|------|-------------|
| `username` | `String` | `@NotBlank` |
| `password` | `String` | `@NotBlank` |

**`AuthResponse`**

| Field | Type | Notes |
|-------|------|-------|
| `token` | `String` | Signed HS256 JWT, 24-hour expiry |
| `username` | `String` | |
| `role` | `Role` enum | `USER` or `ADMIN` |

**`UserDto`**

| Field | Type | Notes |
|-------|------|-------|
| `id` | `Long` | |
| `username` | `String` | |
| `email` | `String` | |
| `role` | `Role` enum | `USER` or `ADMIN` |
| `createdAt` | `LocalDateTime` | ISO-8601 |

**`UpdateUserRequest`**

| Field | Type | Constraints |
|-------|------|-------------|
| `email` | `String` | Optional; valid e-mail format if supplied |
| `role` | `Role` enum | Optional; `USER` or `ADMIN` |

---

## Event Schemas

_Not determinable from code._

No message broker, Kafka topics, RabbitMQ queues, or any other asynchronous event infrastructure is present in the codebase.

---

## Input / Output Formats

- **Content type:** `application/json` for all request and response bodies (Spring Boot default; no explicit content-type negotiation overrides detected).
- **Serialization:** JSON via Jackson (bundled with `spring-boot-starter-web`; additionally `jjwt-jackson` is on the classpath for JWT serialization).
- **Date/time:** `LocalDateTime` fields (e.g., `createdAt`) are serialized as ISO-8601 strings by default Jackson configuration.
- **Enumerations:** `Role` values are serialized as strings (`"USER"`, `"ADMIN"`).
- **Pagination:** _Not determinable from code._ No pagination parameters or `Page`/`Pageable` types are used; `GET /api/users` returns a plain `List<UserDto>`.
- **Request envelope:** No envelope wrapper — request bodies are flat DTO objects.
- **Response envelope:** No envelope wrapper — response bodies are either a single DTO object or a JSON array.
- **JWT transport:** Tokens must be supplied in the `Authorization` HTTP header using the `Bearer <token>` scheme.

---

## Error Handling

### HTTP status codes

| Status | Trigger |
|--------|---------|
| `200 OK` | Successful `GET`, `PUT`, `POST /login` |
| `201 Created` | Successful `POST /api/auth/register` |
| `204 No Content` | Successful `DELETE /api/users/{id}` |
| `400 Bad Request` | Bean Validation failure on a request body (triggered by `@Valid` + `spring-boot-starter-validation`) |
| `401 Unauthorized` | Missing or invalid/expired JWT; invalid login credentials |
| `403 Forbidden` | Valid JWT present but caller does not hold the `ADMIN` role (`@PreAuthorize("hasRole('ADMIN')")` guard fails) |
| `404 Not Found` | No `User` entity exists for the given `{id}` |
| `409 Conflict` | Duplicate `username` or `email` on register; duplicate `email` on update |

### Error payload structure

_Not determinable from code._ No custom `@ControllerAdvice` / `@ExceptionHandler` or problem-detail DTO is present in the provided source files. Error response bodies will therefore follow the default Spring Boot error format (the `BasicErrorController` response with `timestamp`, `status`, `error`, `message`, `path` fields), but the exact shape cannot be confirmed from the available code.

### Validation behaviour

- `@Valid` is applied to all request bodies that accept user input (`RegisterRequest`, `LoginRequest`, `UpdateUserRequest`).
- Violations produce a `400 Bad Request`. Field-level constraint messages are declared (e.g., `"Username must be between 3 and 50 characters"`, `"Email must be a valid email address"`, `"Password must be at least 6 characters"`).

### JWT validation

- The `JwtAuthFilter` silently skips authentication (calls `filterChain.doFilter`) when the `Authorization` header is absent, malformed, or throws a parsing exception; downstream security rules then produce `401`.
- Tokens are validated for: correct signature (HS256, shared secret), matching subject (`username`), and non-expiry (24-hour window).

---

## Versioning

- **URI prefix:** All endpoints are grouped under `/api/auth/**` and `/api/users/**`. No explicit version segment (e.g., `/v1/`) is present in any route.
- **OpenAPI document version:** The `Info` object declares version `"1.0"` (see `OpenApiConfig`).
- **Schema evolution strategy:** _Not determinable from code._ No header-based versioning, content-type negotiation versioning, or multi-version routing is evident.
