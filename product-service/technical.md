---
repo: product-service
spec_type: technical
commit: 4b4f2e1f136fd78b1352d3f16ac6e5489a5c8e13
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: e49333137049b1873579da841b8a4813883e9324476a408b17905fe59ee37194
generated_at: 2026-06-30T17:02:40.200968382+02:00
generator: specsync
---

## Tech Stack

| Component | Version |
|---|---|
| Language | Java 21 |
| Runtime | JVM (OpenJDK 21+) |
| Framework | Spring Boot 3.3.4 |
| Web layer | Spring MVC (`spring-boot-starter-web`) |
| Persistence | Spring Data JPA (`spring-boot-starter-data-jpa`), Hibernate (H2Dialect) |
| Validation | Jakarta Bean Validation (`spring-boot-starter-validation`) |
| Database | H2 in-memory (`com.h2database:h2`, runtime scope) |
| API documentation | springdoc-openapi 2.6.0 (`springdoc-openapi-starter-webmvc-ui`) |
| Code generation | Lombok (compile-time only; excluded from packaged JAR) |
| Build tool | Apache Maven 3.8+ with `spring-boot-maven-plugin` and `maven-compiler-plugin` |
| Test | `spring-boot-starter-test` (test scope) |

## Architecture Patterns

The service follows a classic **layered (n-tier) REST API** architecture with three main application layers:

- **Controller layer** – `ProductController` (`@RestController`) handles HTTP routing at `/api/products`, maps requests/responses to DTOs, and delegates all business logic downward.
- **Service layer** – `ProductService` (`@Service`, `@Transactional`) encapsulates business rules: soft-delete enforcement, stock add/subtract validation, price-range guards, and category parsing. Read operations use a `readOnly = true` transactional context; writes use a full transaction.
- **Repository layer** – `ProductRepository` (`JpaRepository<Product, Long>`) exposes derived-query methods for filtering by category, name (case-insensitive LIKE), price range, and active status. No custom JPQL or native queries are used.

Notable patterns:
- **DTO pattern** – Domain entities (`Product`) are never exposed directly; all responses use `ProductDto` and all writes use typed request objects (`CreateProductRequest`, `UpdateProductRequest`, `StockUpdateRequest`).
- **Soft-delete** – The `active` boolean flag on `Product` acts as a logical delete marker; the `DELETE /{id}` endpoint sets `active = false` rather than removing the row.
- **JPA lifecycle callbacks** – `@PrePersist` and `@PreUpdate` on the `Product` entity automatically manage `createdAt` and `updatedAt` timestamps.
- No CQRS, no event-driven messaging, and no asynchronous workers are present.

## Database & Data Ownership

| Aspect | Detail |
|---|---|
| Datastore type | Relational — H2 in-memory (embedded) |
| JDBC URL | `jdbc:h2:mem:productdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE` |
| DDL strategy | `spring.jpa.hibernate.ddl-auto=create-drop` — schema is generated from entity mappings at startup and dropped on shutdown |
| Seed data | `data.sql` executed via `spring.sql.init.mode=always` after schema creation; inserts 10 sample products |

**Owned table: `products`**

| Column | Type / Constraints |
|---|---|
| `id` | `BIGINT`, PK, auto-increment |
| `name` | `VARCHAR(255)`, NOT NULL |
| `description` | `VARCHAR(1000)`, nullable |
| `price` | `NUMERIC(12,2)`, NOT NULL |
| `stock_quantity` | `INT` |
| `category` | `VARCHAR(50)`, NOT NULL, stored as enum name (`ELECTRONICS`, `CLOTHING`, `BOOKS`, `FOOD`, `SPORTS`, `HOME`, `OTHER`) |
| `image_url` | `VARCHAR(512)`, nullable |
| `active` | `BOOLEAN`, NOT NULL, default `true` |
| `created_at` | `TIMESTAMP`, not updatable |
| `updated_at` | `TIMESTAMP` |

This service is the sole owner of the `products` table. Because the database is in-memory and `create-drop` is used, **all data is ephemeral and lost on restart**. No other service is known to share this datastore.

## Dependencies

### Runtime dependencies

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-web` | Embedded Tomcat, Spring MVC, JSON serialization (Jackson) |
| `spring-boot-starter-data-jpa` | Hibernate ORM, Spring Data repositories, transaction management |
| `spring-boot-starter-validation` | Jakarta Bean Validation (Hibernate Validator) for request DTOs |
| `springdoc-openapi-starter-webmvc-ui` 2.6.0 | Generates OpenAPI 3 spec at `/api-docs`; serves Swagger UI at `/swagger-ui.html` |
| `com.h2database:h2` | Embedded relational database; also exposes a web console at `/h2-console` |

### Build/compile-time only

| Dependency | Purpose |
|---|---|
| `lombok` | Annotation processor generating boilerplate (`@Data`, `@Builder`, `@NoArgsConstructor`, etc.); excluded from packaged JAR |
| `maven-compiler-plugin` | Configures Java 21 source/target and Lombok annotation processing |
| `spring-boot-maven-plugin` | Produces executable fat JAR |

### Test scope

| Dependency | Purpose |
|---|---|
| `spring-boot-starter-test` | JUnit 5, Mockito, Spring test context, MockMvc |

**No external service calls, no message brokers, no caches, and no third-party APIs** are used. All external integration surface is inbound HTTP only.

## Deployment Model

### Packaging
The service is packaged as an executable fat JAR (`product-service-1.0.0.jar`) via `spring-boot-maven-plugin`:
```bash
mvn clean package
java -jar target/product-service-1.0.0.jar
```

### Container / orchestration
_Not determinable from code._ No `Dockerfile`, Docker Compose file, Kubernetes manifests, or Helm chart are present in the repository snapshot.

### Port
The service listens on **HTTP port 8082** (`server.port=8082`).

### Environment configuration
All configuration is file-based via `src/main/resources/application.properties`. No externalized environment variable bindings or Spring profiles are defined in the available source.

### Exposed endpoints

| Path | Purpose |
|---|---|
| `/api/products/**` | Product REST API (see endpoint list) |
| `/swagger-ui.html` | Swagger UI (interactive API browser) |
| `/api-docs` | OpenAPI 3 JSON specification |
| `/h2-console` | H2 web console (restricted to localhost; `web-allow-others=false`) |

### Health / readiness
_Not determinable from code._ Spring Boot Actuator is not listed as a dependency; no `/actuator/health` or custom health endpoint is configured.
