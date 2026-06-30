---
repo: order-service
spec_type: non_functional
commit: 7b86e423571ffb8383d4aa35805b403edd43970b
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: d70aba380427e929d2e224cd6d6906f79deb3c687b47675f3f65a74db96c1048
generated_at: 2026-06-30T17:02:46.078941258+02:00
generator: specsync
---

## Performance

- **Server port:** 8083; served by the embedded Tomcat instance bundled with `spring-boot-starter-web` (Spring Boot 3.3.4). No custom thread-pool or Tomcat connector tuning is present in `application.properties` or any `@Configuration` class, so default Spring Boot settings apply (200 max threads, 8 min spare threads).
- **Outbound HTTP (WebClient):** Two `WebClient` beans (`userWebClient`, `productWebClient`) are constructed without a custom `HttpClient`, meaning no explicit connection-pool size, pending-acquire timeout, max-idle time, or read/write timeouts are configured. Calls to user-service and product-service are made synchronously via `.block()`, which parks a Tomcat thread for the duration of each remote call. No explicit timeout is set on those blocking calls.
- **Database:** H2 in-memory database (`jdbc:h2:mem:orderdb`). JPA/Hibernate is the access layer; `spring.jpa.show-sql=false`. No HikariCP pool properties are overridden, so the Spring Boot default pool of 10 connections applies. `FetchType.LAZY` is used for `Order.items` and `OrderItem.order`, avoiding N+1 fetches where collections are not required.
- **Caching:** No caching layer (no `@EnableCaching`, no Redis, no Caffeine) is configured.
- **Latency targets:** _Not determinable from code._

## Scalability

- **State:** The service uses an H2 **in-memory** database (`jdbc:h2:mem:orderdb` with `DB_CLOSE_DELAY=-1`). This makes each JVM instance fully stateful; data does not survive restarts and is not shared across instances. **Horizontal scaling to multiple replicas is not viable with the current persistence configuration.**
- **Replica count / autoscaling:** No Kubernetes manifests, Docker Compose files, or container resource limits are present in the snapshot. _Not determinable from code._
- **Vertical scaling:** Limited by the single in-memory H2 instance and the blocking WebClient calls that hold Tomcat threads during downstream calls.
- **Partitioning / sharding:** _Not determinable from code._
- **Statelessness (application layer):** The HTTP layer itself carries no server-side session state; all order state is persisted to the database, which is suitable for stateless HTTP once the database is externalised (target).

## Security

- **AuthN / AuthZ:** No authentication or authorisation mechanism is present. There is no `spring-security` dependency, no JWT/OAuth2 configuration, and no security filter chain. All endpoints are publicly accessible.
- **Transport security (TLS):** No TLS (`server.ssl.*`) configuration is present; the service listens on plain HTTP.
- **Secrets handling:** The H2 datasource password is an empty string stored in plain text in `application.properties`. No secrets-management integration (Vault, AWS Secrets Manager, environment variable injection) is evident.
- **H2 Console:** Exposed on `/h2-console` with `web-allow-others=true`, meaning it is reachable from any host — a significant attack surface in any non-local environment.
- **Input validation:** Jakarta Bean Validation (`spring-boot-starter-validation`) is applied to incoming DTOs. Constraints include `@NotNull`, `@NotBlank`, `@NotEmpty`, and `@Min(1)` on request fields; `@Valid` is used at the controller layer for `CreateOrderRequest` and `UpdateOrderStatusRequest`.
- **Security middleware:** _None detected._

## Observability

- **Logging:** Application-level logging via SLF4J/Lombok `@Slf4j`. Log levels set in `application.properties`:
  - `com.specsync.orderservice` → `INFO`
  - `org.springframework.web` → `INFO`
  - Client classes log `WARN` for missing resources and `ERROR` for upstream failures, including structured context (entity IDs, HTTP status codes).
- **Metrics:** No Micrometer, Prometheus, or other metrics registry dependency is declared. _Not determinable from code._
- **Tracing:** No distributed tracing dependency (e.g., Micrometer Tracing, Zipkin, OpenTelemetry) is present.
- **Health / readiness endpoints:** Spring Boot Actuator is **not** declared as a dependency. The README suggests `curl http://localhost:8083/actuator/health` as a health check but provides `curl http://localhost:8083/api/orders` as a fallback — indicating no Actuator endpoint is reliably available. No formal liveness or readiness probe is configured.
- **API documentation:** Swagger UI available at `/swagger-ui.html`; OpenAPI JSON at `/v3/api-docs` (springdoc-openapi 2.6.0).

## Reliability

- **Resilience patterns:** No circuit-breaker library (Resilience4j, Hystrix) is present. No retry mechanism is configured on WebClient calls.
- **Timeout handling:** No connection timeout, read timeout, or request-level deadline is set on either WebClient bean. A hung upstream service will block a Tomcat thread indefinitely until the OS-level TCP timeout fires.
- **Failure handling for downstream services:**
  - 4xx responses from user-service or product-service are treated as "not found" (HTTP 422 returned to caller).
  - 5xx responses or connection failures throw a `RuntimeException`, which the controller maps to HTTP 503 (`SERVICE_UNAVAILABLE`).
  - This is a manual, code-level degradation strategy rather than a structured resilience pattern.
- **Idempotency:** No idempotency keys or deduplication logic is implemented. Retried `POST /api/orders` requests will create duplicate orders.
- **Data durability:** The H2 in-memory database (`create-drop` DDL strategy) loses all data on application restart. This is unsuitable for production and implies zero persistence guarantees.
- **Transaction management:** Spring Data JPA repository methods are transactional by default. No explicit `@Transactional` boundaries or rollback policies are visible in the provided source samples beyond what Spring Data provides automatically.
- **Availability target:** _Not determinable from code._
