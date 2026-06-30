---
repo: user-service
spec_type: non_functional
commit: 4054b55114e1fe16363d8c451cd14db54fc695b0
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 1154734297395299feeb8345ae4561410ebf39ed7eff5a65094d69feb6a3876d
generated_at: 2026-06-30T17:04:50.789336815+02:00
generator: specsync
---

## Performance

- The service listens on **port 8081** with default Spring Boot embedded Tomcat settings; no custom thread-pool or connector tuning is configured in `application.properties` or the POM.
- **JWT tokens expire after 24 hours** (`EXPIRATION_MS = 24L * 60 * 60 * 1000`), hardcoded in `JwtUtil`.
- The backing store is an **H2 in-memory database** (`jdbc:h2:mem:userdb`). Because all data is held in JVM heap, read/write latency for the `users` table is very low under light load, but throughput is bounded by single-JVM memory and H2's concurrency model.
- Connection pooling uses the Spring Boot default (HikariCP with its out-of-the-box pool size); no explicit `spring.datasource.hikari.*` properties are set. _Not determinable from code_ whether pool size has been tuned.
- `spring.jpa.show-sql=false` and `hibernate.format_sql=false` reduce logging overhead in the hot path.
- No HTTP response caching, CDN, or application-level cache (e.g. Spring Cache) is configured.
- No explicit request/response timeout is configured beyond Tomcat defaults.

## Scalability

- Session management is configured as **`SessionCreationPolicy.STATELESS`**; all authentication state is carried in the JWT. This makes individual instances horizontally stateless.
- The H2 in-memory database with `ddl-auto=create-drop` is **not shared across JVM instances**, making true horizontal scaling (multiple replicas) impossible without replacing the database with an external, shared store. Each replica would start with an empty, independent schema.
- No replica count, Kubernetes `HorizontalPodAutoscaler`, or container resource limits/requests are specified in the repository.
- No partitioning, sharding, or event-driven fan-out is present.
- **(target)** A production deployment would require migrating to a persistent, network-accessible database and externalising JWT secret management before horizontal scaling is viable.

## Security

- **Authentication**: JWT Bearer tokens, signed with HMAC-SHA-256 (`Jwts.SIG.HS256`), 24-hour expiry. Tokens are validated per-request by `JwtAuthFilter` (a `OncePerRequestFilter`).
- **Authorisation**: Spring Security with `@EnableMethodSecurity`; user-management endpoints (`/api/users/**`) are guarded by `@PreAuthorize("hasRole('ADMIN')")`. Roles are `USER` and `ADMIN`.
- **Public endpoints**: `/api/auth/**`, `/h2-console/**`, Swagger UI paths (`/swagger-ui/**`, `/swagger-ui.html`, `/v3/api-docs/**`, `/api-docs/**`) are `permitAll()`.
- **Password storage**: BCrypt (`BCryptPasswordEncoder`) is used to hash passwords at rest.
- **Transport security**: No TLS/HTTPS configuration is present in the repository; the service listens on plain HTTP port 8081. TLS termination is not handled at the application layer. _Not determinable from code_ whether a reverse proxy provides TLS in deployment.
- **Secrets handling**: The JWT signing secret is stored in plaintext in `application.properties` (`jwt.secret=specsync-user-service-secret-key-2024-long-enough-for-hs256`). The README documents overriding via the `JWT_SECRET` environment variable, but no secrets-manager or encrypted-properties integration is configured.
- **CSRF**: Disabled (`AbstractHttpConfigurer::disable`). Acceptable for a stateless JWT API, but the H2 console (which uses cookies/sessions) is consequently also unprotected by CSRF controls.
- **Clickjacking (X-Frame-Options)**: Frame options are explicitly **disabled** (`frameOptions.disable()`) to allow the H2 console to render in an iframe.
- **Input validation**: Bean Validation (`jakarta.validation`) is applied to all inbound DTOs: `@NotBlank`, `@Size`, `@Email` constraints on `RegisterRequest`, `LoginRequest`, and `UpdateUserRequest`; Spring returns HTTP 400 on violation.
- **H2 console access**: Restricted to localhost by `spring.h2.console.settings.web-allow-others=false`.

## Observability

- **Logging**: Application-level logging set to `INFO` (`logging.level.com.specsync=INFO`); Spring Security logging suppressed to `WARN`. No structured/JSON logging format is configured; default Spring Boot console appender is used.
- **Metrics**: No Micrometer, Actuator (`spring-boot-starter-actuator`), or external metrics sink (Prometheus, Datadog, etc.) dependency is present. _Not determinable from code._
- **Tracing**: No distributed tracing library (Micrometer Tracing, Zipkin, OpenTelemetry) is configured. _Not determinable from code._
- **Health/readiness endpoints**: Spring Boot Actuator is not included as a dependency; no `/actuator/health` or `/actuator/ready` endpoint is exposed. _Not determinable from code._
- **API documentation**: Swagger UI is available at `/swagger-ui.html` and OpenAPI JSON at `/v3/api-docs`, served by `springdoc-openapi-starter-webmvc-ui 2.6.0`. These provide runtime introspection of the API surface.

## Reliability

- **Retries**: No retry logic (e.g. Spring Retry, Resilience4j) is present anywhere in the codebase.
- **Circuit breakers**: None configured.
- **Timeouts**: No explicit HTTP client or database query timeouts are set beyond framework defaults.
- **Idempotency**: `POST /api/auth/register` is not idempotent; duplicate usernames or emails return HTTP 409 (Conflict), enforced by unique database constraints (`@Column(unique = true)`). `PUT /api/users/{id}` is idempotent by nature of a full-field update.
- **Data durability**: The H2 in-memory database (`create-drop`) means **all data is lost on every application restart**. There is no persistence, backup, or replication mechanism.
- **Failure handling**: JWT parse failures in `JwtAuthFilter` are caught silently and the filter chain continues (unauthenticated), which prevents a malformed token from crashing the request pipeline.
- **Availability**: No liveness/readiness probes, no graceful-shutdown configuration, and no connection-drain timeout is specified. Single-replica, single-process deployment model is implied.
- **Transaction management**: JPA/Hibernate manages transactions for repository operations; no explicit `@Transactional` or saga/compensation patterns are defined at the service layer in the visible source.
