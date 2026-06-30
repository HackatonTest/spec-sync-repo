---
repo: order-service
spec_type: functional
commit: 7b86e423571ffb8383d4aa35805b403edd43970b
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: d70aba380427e929d2e224cd6d6906f79deb3c687b47675f3f65a74db96c1048
generated_at: 2026-06-30T17:02:46.078941258+02:00
generator: specsync
---

## Business Purpose

The Order Service manages the full lifecycle of customer orders within the SpecSync e-commerce platform. It handles order creation, status progression, cancellation, and deletion, serving as the authoritative system of record for all order data. It integrates with the User Service and Product Service to validate customers and resolve product pricing and availability at order creation time.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Order Management
- **Core aggregates / entities:**
  - `Order` — the root aggregate, owning status, total amount, shipping address, timestamps, and notes.
  - `OrderItem` — a line item within an order, capturing product ID, product name, quantity, unit price, and subtotal.
- **Relationships to neighbouring contexts:**
  - **Upstream — User Service** (`http://localhost:8081`): queried synchronously to validate that a customer exists before an order is created.
  - **Upstream — Product Service** (`http://localhost:8082`): queried synchronously to validate product existence, retrieve price, and verify stock availability before an order is created.
  - No downstream consumers are evident (no events published).

## Use Cases / User Stories

- **As an administrator**, I want to list all orders (`GET /api/orders`) so that I can get a full view of platform activity.
- **As an administrator or customer-support agent**, I want to retrieve an order by its ID (`GET /api/orders/{id}`) so that I can inspect its details.
- **As a customer or support agent**, I want to retrieve all orders belonging to a specific user (`GET /api/orders/user/{userId}`) so that I can review a customer's order history.
- **As an administrator**, I want to filter orders by status (`GET /api/orders/status/{status}`) so that I can monitor orders at a specific lifecycle stage.
- **As a customer**, I want to create a new order (`POST /api/orders`) with a list of products and a shipping address so that I can purchase items; the service validates my account and each product's availability before confirming.
- **As an administrator or fulfilment operator**, I want to update an order's status (`PATCH /api/orders/{id}/status`) so that I can advance the order through its lifecycle.
- **As a customer or administrator**, I want to cancel an order (`PATCH /api/orders/{id}/cancel`) so that I can stop processing of an unwanted order before it is shipped.
- **As an administrator**, I want to delete an order record (`DELETE /api/orders/{id}`) so that I can remove erroneous or test orders from the system.

## Business Rules

- **Order creation — user validation:** A `userId` must be provided and must resolve to an existing user in the User Service; if the user is not found, the request is rejected with HTTP 422.
- **Order creation — product validation:** Every line item must reference a product that exists in the Product Service; if any product is not found, the request is rejected with HTTP 422.
- **Order creation — stock validation:** The requested quantity for each product must not exceed available stock reported by the Product Service; violation returns HTTP 422. (inferred — evidenced by README statement "check stock" and `stockQuantity` field on `ProductResponse`)
- **Order creation — item quantity:** Each `OrderItemRequest.quantity` must be at least 1 (validated via `@Min(1)`).
- **Order creation — required fields:** `userId` (not null), `shippingAddress` (not blank), and `items` (not empty) are mandatory on the create request.
- **Order creation — initial status:** New orders are always created in the `PENDING` status (set in the `@PrePersist` callback).
- **Order creation — total amount:** The order's `totalAmount` is computed from per-item unit prices fetched from the Product Service at creation time; the snapshot price is stored on the `OrderItem`.
- **Status lifecycle:** The defined progression is `PENDING → CONFIRMED → PROCESSING → SHIPPED → DELIVERED`. `CANCELLED` and `REFUNDED` are additional terminal-adjacent states.
- **Cancellation constraint:** An order may only be cancelled if its current status is `PENDING` or `CONFIRMED`; any other status results in HTTP 422.
- **`REFUNDED` status:** Treated as a terminal state set manually (e.g., via the update-status endpoint); automatic refund triggering is not implemented.
- **External service unavailability:** If the User Service or Product Service is unreachable during order creation, the service returns HTTP 503 (Service Unavailable) rather than failing silently.
- **Timestamps:** `createdAt` is set at persist time and is immutable (`updatable = false`); `updatedAt` is refreshed on every update via `@PreUpdate`.
- **Notes field:** Optional on order creation; stored with a maximum length of 500 characters (inferred from `@Column(length = 500)`).
