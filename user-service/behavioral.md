---
repo: user-service
spec_type: behavioral
commit: 6556545879f4611dbd7eeec85ff3a878c61c64ef
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: df91828eb6b72b181ecfa3b174acca587720ed81320134be9494d4e16c8c8e22
generated_at: 2026-06-30T17:15:42.037590538+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST over HTTP. All endpoints are served on port `8081`. Authentication is enforced via JWT Bearer tokens except for the public `/api/auth/**` routes.

### Authentication (no JWT required)

| Method | Path | Purpose | Request Body | Response |
|--------|------|---------|--------------|----------|
| `POST` | `/api/auth/register` | Register a new user account | `RegisterRequest` (JSON) | `201 Created` → `AuthResponse` (JSON); `400` validation error; `409` duplicate username/email |
| `POST` | `/api/auth/login` | Authenticate and obtain a JWT | `LoginRequest` (JSON) | `200 OK` → `AuthResponse` (JSON); `401` invalid credentials |

#### `RegisterRequest` fields
| Field | Type | Constraints |
|-------|------|-------------|
| `username` | `String` | Required; 3–50 characters |
| `email` | `String` | Required; valid e-mail format |
| `password` | `String` | Required; minimum 6 characters |

#### `LoginRequest` fields
| Field | Type | Constraints |
|-------|------|-------------|
| `username` | `String` | Required; not blank |
| `password` | `String` | Required; not blank |

#### `AuthResponse` fields
| Field | Type |
|-------|------|
| `token` | `String` (JWT) |
| `username` | `String` |
| `role` | `Role` enum (`USER` \| `ADMIN`) |

---

### User Management (JWT required; `ADMIN` role unless noted)

| Method | Path | Purpose | Request | Response |
|--------|------|---------|---------|----------|
| `GET` | `/api/users` | List all users | — | `200 OK` → `List<UserDto>` |
| `GET` | `/api/users/search` | Paginated search/filter | Query params: `q` (string), `status` (`ACTIVE`\|`INACTIVE`\|`SUSPENDED`), `role` (`USER`\|`ADMIN`), Spring `Pageable` (`page`, `size`, `sort`; default size 20, sort `createdAt`) | `200 OK` → `Page<UserDto>` |
| `GET` | `/api/users/status/{status}` | List users by status | Path: `status` (`ACTIVE`\|`INACTIVE`\|`SUSPENDED`) | `200 OK` → `List<UserDto>` |
| `GET` | `/api/users/stats` | Aggregate user statistics | — | `200 OK` → `UserStatsDto` |
| `GET` | `/api/users/{id}` | Get a user by ID | Path: `id` (Long) | `200 OK` → `UserDto`; `404` not found |
| `PUT` | `/api/users/{id}` | Update user email and/or role | `UpdateUserRequest` (JSON) | `200 OK` → `UserDto`; `404` not found |
| `PATCH` | `/api/users/{id}/status` | Change a user's status | `UpdateStatusRequest` (JSON) | `200 OK` → `UserDto`; `404` not found |
| `PATCH` | `/api/users/{id}/password` | Change a user's password | `ChangePasswordRequest` (JSON) | `204 No Content`; `401` current password wrong; `404` not found |
| `DELETE` | `/api/users/{id}` | Permanently delete a user | Path: `id` (Long) | `204 No Content`; `404` not found |

> **Note on `PATCH /{id}/password`:** access is granted to callers with the `ADMIN` role **or** to the authenticated user whose `id` matches the path (`#id == authentication.principal.id`).

#### `UserDto` fields
| Field | Type |
|-------|------|
| `id` | `Long` |
| `username` | `String` |
| `email` | `String` |
| `role` | `Role` enum (`USER` \| `ADMIN`) |
| `status` | `UserStatus` enum (`ACTIVE` \| `INACTIVE` \| `SUSPENDED`) |
| `lastLoginAt` | `LocalDateTime` (ISO-8601) |
| `createdAt` | `LocalDateTime` (ISO-8601) |

#### `UpdateUserRequest` fields
| Field | Type | Constraints |
|-------|------|-------------|
| `email` | `String` | Optional; valid e-mail format if supplied |
| `role` | `Role` | Optional |

#### `UpdateStatusRequest` fields
| Field | Type | Constraints |
|-------|------|-------------|
| `status` | `UserStatus` | Required; not null |

#### `ChangePasswordRequest` fields
| Field | Type | Constraints |
|-------|------|-------------|
| `currentPassword` | `String` | Required; not blank |
| `newPassword` | `String` | Required; minimum 8 characters |

#### `UserStatsDto` fields
| Field | Type |
|-------|------|
| `totalUsers` | `long` |
| `activeUsers` | `long` |
| `inactiveUsers` | `long` |
| `suspendedUsers` | `long` |
| `adminCount` | `long` |
| `userCount` | `long` |
| `registeredLast7Days` | `long` |
| `registeredLast30Days` | `long` |

---

## Event Schemas

_Not determinable from code._

No message broker dependencies (Kafka, RabbitMQ, etc.) are declared; no event producers or consumers are present in the source.

---

## Input / Output Formats

- **Content-Type / Accept:** `application/json` for all request bodies and responses.
- **Serialization:** JSON via Jackson (included transitively through `spring-boot-starter-web` and `jjwt-jackson`).
- **Date/time fields:** serialized as ISO-8601 `LocalDateTime` strings (Jackson default with Spring Boot auto-configuration).
- **Pagination:** The `GET /api/users/search` endpoint accepts standard Spring Data `Pageable` query parameters (`page`, `size`, `sort`). The default page size is `20`, sorted by `createdAt`. The response is wrapped in a Spring Data `Page<UserDto>` envelope that includes `content`, `totalElements`, `totalPages`, `number`, `size`, and related pagination metadata.
- **Enum serialization:** `Role` (`USER`, `ADMIN`) and `UserStatus` (`ACTIVE`, `INACTIVE`, `SUSPENDED`) are stored and serialized as strings.
- **Authentication header:** `Authorization: Bearer <jwt-token>` on all protected endpoints.

---

## Error Handling

| HTTP Status | Trigger Condition |
|-------------|-------------------|
| `400 Bad Request` | Bean Validation failure on any `@Valid`-annotated request body (e.g., blank required fields, invalid e-mail format, password too short) |
| `401 Unauthorized` | Missing or invalid/expired JWT token; or incorrect `currentPassword` on `PATCH /{id}/password` |
| `403 Forbidden` | Valid JWT present but caller lacks the required role (`ADMIN`) for the requested endpoint |
| `404 Not Found` | No user exists with the given `{id}` |
| `409 Conflict` | `POST /api/auth/register` attempted with a `username` or `email` already in use |

The specific error payload structure (e.g., field names on the error response body) is _not determinable from code_—no custom `@ControllerAdvice` / `ExceptionHandler` class is present in the provided source samples; Spring Boot's default error response format applies.

Session management is stateless (`SessionCreationPolicy.STATELESS`); the JWT filter (`JwtAuthFilter`) intercepts every request before the standard `UsernamePasswordAuthenticationFilter`.

---

## Versioning

No URI version prefix (e.g., `/v1/`) is used. The base paths are `/api/auth` and `/api/users` with no version segment. The OpenAPI document reports version `"1.0"` in the info block (`OpenApiConfig`), but this is documentation metadata only and is not reflected in the URL structure. No header-based or content-negotiation versioning is evident.
