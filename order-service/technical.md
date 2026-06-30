---
repo: order-service
spec_type: technical
commit: 7b86e423571ffb8383d4aa35805b403edd43970b
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: d70aba380427e929d2e224cd6d6906f79deb3c687b47675f3f65a74db96c1048
generated_at: 2026-06-30T17:02:46.078941258+02:00
generator: specsync
---

## Tech Stack

| Component | Detail |
|---|---|
| Language | Java 21 |
| Runtime | JVM |
| Framework | Spring Boot 3.3.4 |
| Web layer | Spring MVC (`spring-boot-starter-web`) |
| Persistence | Spring Data JPA 3.3.4 (`spring-boot-starter-data-jpa`) |
| Reactive HTTP client | Spring WebFlux 3.3.4 (`spring-boot-starter-webflux`) — used solely for `WebClient` outbound calls |
| Validation | `spring-boot-starter-validation` (Jakarta Bean Validation) |
| API documentation | springdoc-openapi 2.6.0 (`springdoc-openapi-starter-webmvc-ui`) |
| Database | H2 in-memory (`runtime` scope) |
| Code generation | Lombok (compile-time, excluded from fat-jar) |
| Build tool | Maven 3 (`spring-boot-maven-plugin`) |
| Test | `spring-boot-starter-test` (scope: test) |

## Architecture Patterns

The service follows a classic **layered REST API** style with thin client adapters for outbound calls:

- **Controller layer** — `OrderController` (`/api/orders`): handles HTTP request/response mapping, input validation, and exception-to-status translation.
- **Service layer** — `OrderService`: encapsulates business logic including order lifecycle state transitions, validation orchestration, and total-amount calculation.
- **Repository layer** — Spring Data JPA repositories backed by H2 for `Order` and `OrderItem` entities.
- **Client adapters** — `UserServiceClient` and `ProductServiceClient` wrap `WebClient` (reactive) calls with blocking `.block()` to bridge into the synchronous service layer. Each client handles 4xx/5xx HTTP responses and connectivity failures distinctly, surfacing them as typed exceptions.
- **Configuration** — `WebClientConfig` creates two named `WebClient` beans (`userWebClient`, `productWebClient`) configured from application properties. `OpenApiConfig` defines the Swagger/OpenAPI metadata.
- **Domain model** — `Order` (aggregate root) → `OrderItem` (child, `CascadeType.ALL`); `OrderStatus` enum drives lifecycle rules (cancellation only from `PENDING` or `CONFIRMED`).

No CQRS, event sourcing, or async messaging patterns are present.

## Database & Data Ownership

| Aspect | Detail |
|---|---|
| Datastore | H2 in-memory relational database (`jdbc:h2:mem:orderdb`) |
| DDL strategy | `spring.jpa.hibernate.ddl-auto=create-drop` — schema is created on startup and dropped on shutdown |
| Seed data | `data.sql` is executed on startup (`spring.sql.init.mode=always`) inserting 5 sample orders |

**Owned tables:**

| Table | Entity class | Description |
|---|---|---|
| `orders` | `Order` | Core order record: `userId`, `status` (enum string), `totalAmount`, `shippingAddress`, `notes`, `createdAt`, `updatedAt` |
| `order_items` | `OrderItem` | Line items linked to an order: `productId`, `productName`, `quantity`, `unitPrice`, `subtotal` |

`order_items.order_id` is a foreign key to `orders.id`. All item lifecycle is managed via `CascadeType.ALL` / `orphanRemoval = true` from the `Order` aggregate.

This service is the **sole owner** of both tables. User and product data are not stored locally; only their IDs (and product name/price snapshots) are persisted.

## Dependencies

### Runtime — other internal services

| Service | Base URL (default) | Protocol | Purpose |
|---|---|---|---|
| `user-service` | `http://localhost:8081` | HTTP REST (WebClient) | Validate that the customer (`userId`) exists before order creation |
| `product-service` | `http://localhost:8082` | HTTP REST (WebClient) | Validate product existence, retrieve price and stock level for each order item |

Base URLs are configurable via `services.user-service.url` and `services.product-service.url` in `application.properties`. If either service is unreachable, `createOrder` returns HTTP 503; a missing user or product returns HTTP 422.

### Runtime — infrastructure

| Dependency | Role |
|---|---|
| H2 (in-memory) | Embedded relational database; data is non-persistent across restarts |

### Runtime — frameworks/libraries

| Library | Role |
|---|---|
| `spring-boot-starter-web` | Embedded Tomcat, Spring MVC |
| `spring-boot-starter-data-jpa` | JPA/Hibernate ORM |
| `spring-boot-starter-webflux` | `WebClient` for outbound HTTP |
| `spring-boot-starter-validation` | Bean Validation on DTOs |
| `springdoc-openapi-starter-webmvc-ui` 2.6.0 | Swagger UI + OpenAPI 3 doc generation |

### Build-time only

| Library | Role |
|---|---|
| Lombok | Boilerplate code generation (`@Data`, `@Builder`, etc.) — excluded from fat-jar |
| `spring-boot-starter-test` | Unit/integration test support |

No message brokers, caches, or third-party external APIs are used.

## Deployment Model

### Build

Built as an executable fat-jar via `spring-boot-maven-plugin`:

```bash
mvn clean package -DskipTests
java -jar target/order-service-1.0.0.jar
```

### Container / Orchestration

_Not determinable from code._ No `Dockerfile`, Docker Compose file, Kubernetes manifests, or Helm chart are present in the repository snapshot.

### Ports & Endpoints

| Item | Value |
|---|---|
| HTTP port | `8083` (`server.port=8083`) |
| API base path | `/api/orders` |
| Swagger UI | `http://localhost:8083/swagger-ui.html` |
| OpenAPI JSON | `http://localhost:8083/v3/api-docs` |
| H2 console | `http://localhost:8083/h2-console` (web access enabled) |
| Actuator health | `http://localhost:8083/actuator/health` (referenced in README; Spring Boot Actuator dependency not explicitly declared in `pom.xml`) |

### Environment Configuration

All externalisable settings are driven by `application.properties`. Key overridable properties:

| Property | Default | Purpose |
|---|---|---|
| `server.port` | `8083` | Listening port |
| `services.user-service.url` | `http://localhost:8081` | user-service base URL |
| `services.product-service.url` | `http://localhost:8082` | product-service base URL |
| `spring.datasource.url` | `jdbc:h2:mem:orderdb;…` | JDBC URL (in-memory; not persistent) |
| `spring.datasource.username` / `password` | `sa` / _(empty)_ | DB credentials |

### Health / Readiness

_Not determinable from code._ `spring-boot-starter-actuator` is not listed as a dependency in `pom.xml`; the `/actuator/health` path mentioned in the README may not be active unless the dependency is added at runtime or is transitively included.
