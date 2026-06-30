---
repo: user-service
spec_type: technical
commit: 4054b55114e1fe16363d8c451cd14db54fc695b0
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 1154734297395299feeb8345ae4561410ebf39ed7eff5a65094d69feb6a3876d
generated_at: 2026-06-30T17:04:50.789336815+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|-----------|--------|
| Language | Java 21 |
| Runtime | JVM (OpenJDK-compatible) |
| Framework | Spring Boot 3.3.4 |
| Web layer | Spring MVC (`spring-boot-starter-web`) |
| Persistence | Spring Data JPA (`spring-boot-starter-data-jpa`) |
| Security | Spring Security 6 (`spring-boot-starter-security`) |
| Validation | Jakarta Bean Validation (`spring-boot-starter-validation`) |
| JWT | JJWT 0.12.6 (`jjwt-api`, `jjwt-impl`, `jjwt-jackson`) |
| API docs | springdoc-openapi 2.6.0 (Swagger UI + OpenAPI 3) |
| Database | H2 in-memory (runtime scope) |
| Code generation | Lombok (compile-time only; excluded from the fat JAR) |
| Build tool | Maven 3 (`spring-boot-maven-plugin`) |
| Test | Spring Boot Test + Spring Security Test (test scope) |

## Architecture Patterns

**Style:** Layered REST API (Controller → Service → Repository).

**Key patterns and components:**

- **Stateless REST API** — no HTTP sessions; every request is authenticated via a bearer token (`SessionCreationPolicy.STATELESS`).
- **JWT-based authentication** — a custom `OncePerRequestFilter` (`JwtAuthFilter`) intercepts every request, extracts and validates the HS256-signed token, and populates the `SecurityContext`.
- **Role-based access control (RBAC)** — Spring Security method-level security (`@EnableMethodSecurity`, `@PreAuthorize("hasRole('ADMIN')")`) gates all user-management endpoints behind the `ADMIN` role; authentication endpoints are fully public.
- **DTO projection** — internal `User` entity is never exposed directly; `UserDto`, `AuthResponse`, `LoginRequest`, `RegisterRequest`, and `UpdateUserRequest` form the public API surface.
- **Data seeding** — `spring.sql.init.mode=always` with `spring.jpa.defer-datasource-initialization=true` implies a SQL init script populates seed users at startup (three pre-seeded accounts: `admin`, `john.doe`, `jane.smith`).
- **OpenAPI-first documentation** — `OpenApiConfig` registers a global Bearer-auth security scheme so Swagger UI can test secured endpoints interactively.

Internal component map:

```
AuthController  ──┐
UserController  ──┤──► UserService ──► UserRepository ──► H2 (users table)
                  │         │
                  │         └──► JwtUtil (token generation / validation)
JwtAuthFilter ────┘──► JwtUtil + UserDetailsServiceImpl
```

## Database & Data Ownership

| Aspect | Detail |
|--------|--------|
| Datastore | H2 in-memory relational database (`jdbc:h2:mem:userdb`) |
| Schema strategy | `create-drop` — schema is created on startup and dropped on shutdown; data is not persisted across restarts |
| Owned table | `users` |

**`users` table** (derived from `User` entity):

| Column | Type | Constraints |
|--------|------|-------------|
| `id` | BIGINT (identity) | PK, auto-generated |
| `username` | VARCHAR(50) | UNIQUE, NOT NULL |
| `email` | VARCHAR(100) | UNIQUE, NOT NULL |
| `password` | VARCHAR | NOT NULL (BCrypt hash) |
| `role` | VARCHAR(10) | NOT NULL, default `USER`; enum: `USER`, `ADMIN` |
| `created_at` | TIMESTAMP | NOT NULL, set on insert, immutable |
| `updated_at` | TIMESTAMP | NOT NULL, updated on every write |

This service is the sole owner of the `users` table. No other service shares this datastore (H2 in-memory is not accessible externally).

## Dependencies

### Runtime dependencies

| Dependency | Purpose |
|------------|---------|
| Spring Boot 3.3.4 (web, data-jpa, security, validation) | Core application framework |
| H2 Database | Embedded in-memory RDBMS; not accessible outside the process |
| JJWT 0.12.6 (`jjwt-api`, `jjwt-impl`, `jjwt-jackson`) | HS256 JWT generation and parsing |
| springdoc-openapi-starter-webmvc-ui 2.6.0 | Auto-generated OpenAPI 3 spec + Swagger UI |
| Hibernate (transitive via Spring Data JPA) | ORM / DDL management |

### Build/compile-time only

| Dependency | Purpose |
|------------|---------|
| Lombok | Boilerplate reduction (`@Data`, `@Builder`, etc.); excluded from the packaged JAR |

### Test-scope

| Dependency | Purpose |
|------------|---------|
| `spring-boot-starter-test` | JUnit 5, Mockito, MockMvc |
| `spring-security-test` | Security context test utilities |

**No external service calls, message brokers, caches, or third-party HTTP APIs** are present. The service is fully self-contained at runtime.

## Deployment Model

**Build:**
```bash
mvn clean package -DskipTests
java -jar target/user-service-0.0.1-SNAPSHOT.jar
```
Packaged as a self-contained executable fat JAR by `spring-boot-maven-plugin`.

**Container / orchestration:** _Not determinable from code._ No `Dockerfile`, Docker Compose file, Kubernetes manifests, or Helm chart are present in the repository snapshot.

**Port:** `8081` (HTTP), configured via `server.port`.

**Environment configuration:**

All properties in `application.properties` can be overridden via environment variables (Spring Boot convention: replace `.` → `_`, uppercase):

| Property | Default | Notes |
|----------|---------|-------|
| `SERVER_PORT` | `8081` | HTTP listener port |
| `JWT_SECRET` | `specsync-user-service-secret-key-2024-long-enough-for-hs256` | HS256 signing key — **must be overridden in production** |
| `SPRING_DATASOURCE_URL` | `jdbc:h2:mem:userdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE` | In-memory only by default |
| `SPRING_JPA_HIBERNATE_DDL_AUTO` | `create-drop` | Change to `validate` or `none` against a persistent store |
| `SPRING_H2_CONSOLE_ENABLED` | `true` | Should be disabled in production |

**Notable URL paths at runtime:**

| Path | Purpose |
|------|---------|
| `/swagger-ui.html` | Interactive API documentation (public) |
| `/v3/api-docs` | OpenAPI 3 JSON spec (public) |
| `/h2-console` | H2 web console (public, `web-allow-others=false` limits to localhost) |

**Health/readiness endpoints:** _Not determinable from code._ Spring Boot Actuator is not listed as a dependency; no dedicated health endpoint is configured.
