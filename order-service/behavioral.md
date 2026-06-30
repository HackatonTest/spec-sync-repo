---
repo: order-service
spec_type: behavioral
commit: 7b86e423571ffb8383d4aa35805b403edd43970b
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: d70aba380427e929d2e224cd6d6906f79deb3c687b47675f3f65a74db96c1048
generated_at: 2026-06-30T17:02:46.078941258+02:00
generator: specsync
---

## API Contracts

**Protocol:** REST over HTTP/1.1. The service exposes a JSON API on port **8083** under the base path `/api/orders`. An OpenAPI 3 specification is generated automatically by springdoc-openapi and is available at `/v3/api-docs`; Swagger UI is served at `/swagger-ui.html`.

| Method | Path | Purpose | Request Body | Success Response |
|--------|------|---------|--------------|-----------------|
| `GET` | `/api/orders` | List all orders | None | `200 OK` — `List<OrderDto>` |
| `GET` | `/api/orders/{id}` | Get order by ID | None | `200 OK` — `OrderDto` |
| `GET` | `/api/orders/user/{userId}` | Get all orders for a user | None | `200 OK` — `List<OrderDto>` |
| `GET` | `/api/orders/status/{status}` | Get orders by status | None | `200 OK` — `List<OrderDto>` |
| `POST` | `/api/orders` | Create a new order | `CreateOrderRequest` (JSON) | `201 Created` — `OrderDto` |
| `PATCH` | `/api/orders/{id}/status` | Update order status | `UpdateOrderStatusRequest` (JSON) | `200 OK` — `OrderDto` |
| `PATCH` | `/api/orders/{id}/cancel` | Cancel an order | None | `200 OK` — `OrderDto` |
| `DELETE` | `/api/orders/{id}` | Delete an order | None | `204 No Content` |

### Path Parameter Types

| Parameter | Type | Description |
|-----------|------|-------------|
| `{id}` | `Long` | Order primary key |
| `{userId}` | `Long` | User identifier |
| `{status}` | `OrderStatus` enum | One of `PENDING`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`, `REFUNDED` |

### DTO Schemas

**`CreateOrderRequest`** (POST `/api/orders` request body):

| Field | Type | Constraints |
|-------|------|-------------|
| `userId` | `Long` | Required (`@NotNull`) |
| `shippingAddress` | `String` | Required, non-blank (`@NotBlank`) |
| `notes` | `String` | Optional |
| `items` | `List<OrderItemRequest>` | Required, non-empty (`@NotEmpty`); each item validated (`@Valid`) |

**`OrderItemRequest`** (nested in `CreateOrderRequest.items`):

| Field | Type | Constraints |
|-------|------|-------------|
| `productId` | `Long` | Required (`@NotNull`) |
| `quantity` | `int` | Minimum 1 (`@Min(1)`) |

**`UpdateOrderStatusRequest`** (PATCH `/{id}/status` request body):

| Field | Type | Constraints |
|-------|------|-------------|
| `status` | `OrderStatus` | Required (`@NotNull`) |

**`OrderDto`** (standard response object):

| Field | Type |
|-------|------|
| `id` | `Long` |
| `userId` | `Long` |
| `status` | `OrderStatus` |
| `totalAmount` | `BigDecimal` |
| `shippingAddress` | `String` |
| `notes` | `String` |
| `createdAt` | `LocalDateTime` |
| `updatedAt` | `LocalDateTime` |
| `items` | `List<OrderItemDto>` |

**`OrderItemDto`** (nested in `OrderDto.items`):

| Field | Type |
|-------|------|
| `id` | `Long` |
| `productId` | `Long` |
| `productName` | `String` |
| `quantity` | `int` |
| `unitPrice` | `BigDecimal` |
| `subtotal` | `BigDecimal` |

### `OrderStatus` Enum Values

`PENDING` · `CONFIRMED` · `PROCESSING` · `SHIPPED` · `DELIVERED` · `CANCELLED` · `REFUNDED`

Lifecycle: `PENDING` → `CONFIRMED` → `PROCESSING` → `SHIPPED` → `DELIVERED`. Cancellation is allowed only from `PENDING` or `CONFIRMED`. `REFUNDED` is a terminal state set manually.

---

## Event Schemas

_Not determinable from code._

No messaging broker dependency (Kafka, RabbitMQ, etc.) is present in `pom.xml`, `application.properties`, or any source file. The service communicates with upstream services exclusively via synchronous HTTP (WebClient).

---

## Input / Output Formats

**Content type:** `application/json` for all request bodies and responses. WebClient beans are configured with `Content-Type: application/json` and `Accept: application/json` default headers, consistent with the controller contract.

**Serialization:** JSON via Jackson (Spring Boot default). `BigDecimal` fields for monetary amounts are declared with precision 12/10, scale 2 in the JPA layer; serialized as JSON numbers. `LocalDateTime` fields (`createdAt`, `updatedAt`) are serialized as ISO-8601 datetime strings by default Spring Boot/Jackson configuration.

**Pagination:** No pagination mechanism is implemented. All collection endpoints (`GET /api/orders`, `GET /api/orders/user/{userId}`, `GET /api/orders/status/{status}`) return a plain `List` (unbounded array).

**Request envelope:** No wrapper envelope — request bodies are flat JSON objects matching the DTO schemas above.

**Response envelope:** No wrapper envelope — response bodies are either a single `OrderDto` object or a JSON array of `OrderDto` objects. Error responses use a simple `ErrorResponse` object with a single `message` string field (evidenced by `new ErrorResponse(e.getMessage())` in the controller).

---

## Error Handling

### Status Code Mapping

| HTTP Status | Trigger |
|-------------|---------|
| `200 OK` | Successful read or update |
| `201 Created` | Order successfully created (`POST /api/orders`) |
| `204 No Content` | Order successfully deleted (`DELETE /{id}`) |
| `400 Bad Request` | Bean Validation failure on request body (`@Valid`); invalid `status` enum value in path/body |
| `404 Not Found` | `NoSuchElementException` thrown by service layer (order not found by ID) |
| `422 Unprocessable Entity` | `IllegalArgumentException` or `IllegalStateException` from service — e.g., user not found, product not found, insufficient stock, invalid cancellation state |
| `503 Service Unavailable` | `RuntimeException` propagated when user-service or product-service is unreachable |

### Error Response Payload

For `422` and `503` responses the body is a JSON object with the following structure (based on `ErrorResponse` instantiation in the controller):

```json
{
  "message": "<human-readable error description>"
}
```

For `404` responses, the body is empty (`ResponseEntity.notFound().build()`).

For `400` responses caused by Bean Validation, Spring Boot's default `MethodArgumentNotValidException` handling applies; the exact structure of the validation error payload is not overridden in the visible code and therefore follows Spring Boot defaults (`timestamp`, `status`, `errors` array, etc.).

### Validation Behaviour

Bean Validation (`@Valid`) is applied to `CreateOrderRequest` and `UpdateOrderStatusRequest` request bodies. Constraint violations produce a `400 Bad Request` before the controller logic is reached. Specific constraints enforced:

- `CreateOrderRequest.userId` — must not be null
- `CreateOrderRequest.shippingAddress` — must not be blank
- `CreateOrderRequest.items` — must not be empty
- `OrderItemRequest.productId` — must not be null
- `OrderItemRequest.quantity` — must be ≥ 1
- `UpdateOrderStatusRequest.status` — must not be null

### Outbound-Service Error Propagation

- **User not found** (4xx from user-service): `Optional.empty()` returned → service throws `IllegalArgumentException` → controller responds `422`.
- **Product not found** (4xx from product-service): `Optional.empty()` returned → service throws `IllegalArgumentException` → controller responds `422`.
- **user-service / product-service unreachable or 5xx**: `RuntimeException("…-service is unavailable")` thrown → controller responds `503`.

---

## Versioning

No explicit API versioning strategy is implemented. There is no URI version prefix (e.g., `/v1/`), no version request header, and no content-type negotiation based on version. The OpenAPI document declares version `"1.0"` in the `Info` object (`OpenApiConfig`), which is a documentation label only.

The service artifact version is `1.0.0` (from `pom.xml`). Schema evolution is not addressed in any visible code.
