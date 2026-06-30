---
repo: product-service
spec_type: non_functional
commit: 4b4f2e1f136fd78b1352d3f16ac6e5489a5c8e13
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: e49333137049b1873579da841b8a4813883e9324476a408b17905fe59ee37194
generated_at: 2026-06-30T17:02:40.200968382+02:00
generator: specsync
---

## Performance

- The service runs on a single embedded Tomcat instance (Spring Boot default) listening on **port 8082**. No explicit thread-pool tuning (`server.tomcat.threads.*`) is present in `application.properties`; Spring Boot defaults apply (max 200 worker threads, min-spare 10). (target)
- Persistence is backed by an **H2 in-memory database** (`jdbc:h2:mem:productdb`). Because all data lives in process memory, read/write latency is sub-millisecond for the database layer itself, but the entire dataset is lost on restart.
- No explicit connection-pool configuration (`spring.datasource.hikari.*`) is present; HikariCP defaults are used (maximum pool size 10). _Not determinable from code._
- All read operations in `ProductService` are annotated `@Transactional(readOnly = true)`, which lets Hibernate and the JDBC driver apply read-optimisation hints.
- No HTTP response caching (e.g., `Cache-Control` headers, Spring Cache abstraction, Redis) is configured.
- No explicit request-timeout or async/reactive processing is configured; all endpoints are synchronous blocking.
- SQL logging is disabled (`spring.jpa.show-sql=false`), avoiding per-request log overhead in production.

## Scalability

- The service is packaged as a self-contained JAR and is horizontally scalable in principle, **but** the H2 in-memory database is embedded and instance-local; multiple replicas would each maintain independent, non-shared state. Horizontal scaling beyond a single instance is therefore **not viable** without replacing the persistence layer.
- No replica count, Kubernetes `Deployment`, Docker Compose, or resource-limit configuration is present in the repository. _Not determinable from code._
- No autoscaling configuration (HPA, KEDA, etc.) is evident. _Not determinable from code._
- The application is otherwise stateless at the HTTP layer (no in-process session state), which would support horizontal scaling once an external, shared database is introduced. (target)
- No database partitioning or sharding is configured.

## Security

- **Authentication / Authorisation:** No `spring-security` dependency is declared in `pom.xml` and no security filter chain is configured. All endpoints are unauthenticated and publicly accessible.
- **Transport security (TLS):** No `server.ssl.*` properties are set; the service listens on plain HTTP. TLS must be terminated externally (e.g., by an ingress or API gateway) if required. _Not determinable from code._
- **Secrets handling:** The H2 datasource password is an empty string stored in plain text in `application.properties`. No secrets-management integration (Vault, Kubernetes Secrets, environment-variable injection) is evident.
- **H2 Console:** The web console is **enabled** (`spring.h2.console.enabled=true`) and is reachable at `/h2-console`. Remote access is explicitly restricted (`spring.h2.console.settings.web-allow-others=false`), limiting access to `localhost` only. The console should be disabled in any internet-facing deployment.
- **Input validation:** Bean Validation (Jakarta Validation / Hibernate Validator via `spring-boot-starter-validation`) is applied to all mutating request DTOs (`@Valid` on controller parameters). Constraints include `@NotBlank`, `@Size`, `@DecimalMin`, `@Digits`, `@Min`, and `@NotNull`. Category and price-range inputs are validated in the service layer with appropriate `400`/`409` responses.
- **CORS / CSRF:** No CORS configuration and no CSRF protection are configured. Because Spring Security is absent, CSRF filters are not active.
- **Swagger UI / OpenAPI:** The API documentation UI (`/swagger-ui.html`, `/api-docs`) is enabled with no access restrictions. The `try-it-out` feature is enabled, allowing unauthenticated write operations directly from the UI.

## Observability

- **Logging:** Spring Boot default Logback logging is active. SQL statement logging is explicitly disabled (`spring.jpa.show-sql=false`). No structured/JSON logging configuration or log-shipping integration is present. _Not determinable from code._
- **Metrics:** `spring-boot-actuator` is **not** declared as a dependency; no Micrometer metrics endpoint (Prometheus, etc.) is configured.
- **Distributed tracing:** No tracing library (Micrometer Tracing, OpenTelemetry, Sleuth) is present.
- **Health / readiness endpoints:** Actuator is absent, so no `/actuator/health` or `/actuator/readiness` endpoint is available. No custom health endpoint is implemented.
- **API documentation:** Springdoc OpenAPI 2.6.0 exposes a Swagger UI at `/swagger-ui.html` and the OpenAPI JSON schema at `/api-docs`, providing a form of runtime introspection of the API surface.

## Reliability

- **Retries:** No retry mechanism (Spring Retry, Resilience4j) is configured at the service or HTTP client level.
- **Circuit breakers:** No circuit-breaker pattern (Resilience4j, Hystrix) is present. The service has no outbound HTTP calls to other services, so this risk is limited to the database connection.
- **Timeouts:** No explicit database query timeout or HTTP request timeout is configured.
- **Idempotency:** `PUT /{id}` and `PATCH /{id}/stock` operations are effectively idempotent for the same input values. `POST /api/products` does not enforce client-supplied idempotency keys; repeated identical requests will create duplicate records.
- **Soft deletes:** Deletion is implemented as a soft-delete (setting `active = false`) rather than a physical `DELETE`, preserving data recoverability without a separate backup strategy.
- **Stock integrity:** Insufficient-stock protection for the `SUBTRACT` operation returns HTTP `409 Conflict` before mutation, preventing negative inventory. Operations are wrapped in `@Transactional`, ensuring atomicity at the database level.
- **Data durability:** Because the database is H2 in-memory with `ddl-auto=create-drop`, **all data is lost on every restart**. There is no persistence, backup, or replication. This makes the service unsuitable for production use without replacing the storage layer.
- **No message queue / event sourcing:** No asynchronous messaging (Kafka, RabbitMQ) is present, so there is no event-driven recovery mechanism.
