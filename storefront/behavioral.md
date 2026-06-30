---
repo: storefront
spec_type: behavioral
commit: aeb8a560fd0d89f7906350b77d76a9b0ac58449c
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 3ac71d18a2e48b5c79e887cae2f954d2a9191b766b1a2147673a85a417de765c
generated_at: 2026-06-30T17:03:39.742940382+02:00
generator: specsync
---

## API Contracts

This microservice is an **Angular 17 single-page application (frontend client)**. It does not expose its own HTTP server API; instead it consumes three downstream backend REST services over HTTP/JSON. The protocol used for all outbound calls is **REST over HTTP**.

### Outbound REST calls consumed by this frontend

#### User Service (`http://localhost:8081`)

| Method | Path | Purpose | Request Body | Response Body |
|--------|------|---------|--------------|---------------|
| POST | _Not determinable from code._ `/login` (inferred from `userService.login()`) | Authenticate user | `{ username: string, password: string }` | `{ token: string, username: string, email: string, role: string, userId: number }` |
| POST | _Not determinable from code._ `/register` (inferred from `userService.register()`) | Register new user | `{ username: string, email: string, password: string, role?: string }` | `User` object |
| GET | _Not determinable from code._ (inferred from `userService.getUsers()`) | List all users | — | `User[]` |
| DELETE | _Not determinable from code._ (inferred from `userService.deleteUser(id)`) | Delete a user by ID | — | — |

#### Product Service (`http://localhost:8082`)

| Method | Path | Purpose | Request Body | Response Body |
|--------|------|---------|--------------|---------------|
| GET | _Not determinable from code._ (inferred from `productService.getProducts()`) | List all products | — | `Product[]` |
| GET | _Not determinable from code._ (inferred from `productService.getProductById(id)`) | Get single product | — | `Product` |
| POST | _Not determinable from code._ (inferred from `productService.createProduct()`) | Create a product | `CreateProductRequest` | `Product` |
| PUT/PATCH | _Not determinable from code._ (inferred from `productService.updateProduct(id, body)`) | Update a product | `UpdateProductRequest` | `Product` |
| DELETE | _Not determinable from code._ (inferred from `productService.deleteProduct(id)`) | Delete a product | — | — |

#### Order Service (`http://localhost:8083`)

| Method | Path | Purpose | Request Body | Response Body |
|--------|------|---------|--------------|---------------|
| GET | _Not determinable from code._ (inferred from `orderService.getOrders()`) | List all orders | — | `Order[]` |
| POST | _Not determinable from code._ (inferred from `orderService.createOrder()`) | Create an order | `CreateOrderRequest` | `Order` |
| PUT/PATCH | _Not determinable from code._ (inferred from `orderService.updateOrderStatus(id, body)`) | Update order status | `{ status: OrderStatus }` | `Order` |
| DELETE | _Not determinable from code._ (inferred from `orderService.deleteOrder(id)`) | Delete an order | — | — |

### Key data shapes evidenced in models

**`User`**: `{ id: number, username: string, email: string, role: string, createdAt: string }`

**`Product`**: `{ id: number, name: string, description: string, price: number, stockQuantity: number, category: string, imageUrl?: string, active: boolean }`

**`Order`**: `{ id: number, userId: number, status: OrderStatus, totalAmount: number, shippingAddress: string, items: OrderItem[], createdAt?: string, updatedAt?: string }`

**`OrderItem`**: `{ productId: number, productName: string, quantity: number, unitPrice: number, subtotal: number }`

**`OrderStatus`** (enum): `PENDING | CONFIRMED | SHIPPED | DELIVERED | CANCELLED`

**`AuthResponse`**: `{ token: string, username: string, email: string, role: string, userId: number }`

**`CreateOrderRequest`**: `{ userId: number, shippingAddress: string, items: { productId: number, quantity: number }[] }`

---

## Event Schemas

_Not determinable from code._

No asynchronous messaging (Kafka, RabbitMQ, etc.) is present. This is a browser-based Angular frontend with no event broker integration.

---

## Input / Output Formats

- **Content type**: JSON (`application/json`) for all REST communication — evidenced by Angular's `HttpClient` default behaviour and the structured TypeScript model interfaces.
- **Serialization**: JSON only; no Protobuf or Avro usage detected.
- **Authentication**: JWT-based. The `AuthResponse.token` field is stored client-side after login and applied to subsequent requests (exact header attachment mechanism is _not determinable from code_ as the HTTP interceptor source is not included in the snapshot).
- **Pagination**: Client-side pagination only, via Angular Material `MatPaginator` on the Users and Orders list views. No server-side pagination parameters (e.g. `page`, `size`) are evidenced.
- **Filtering**: Client-side filtering on product name/description/category and order status. The `getProducts()` call fetches the full list; filtering is done in-browser.
- **Request envelopes**: No wrapper envelope; request and response bodies are flat JSON objects or arrays matching the model interfaces directly.
- **Response envelopes**: No wrapper envelope evidenced; responses are expected to deserialize directly to typed model arrays or objects.

---

## Error Handling

Error handling is entirely client-side (UI feedback); the frontend does not define server error schemas but does branch on specific status codes:

| Condition | Handling |
|-----------|----------|
| HTTP `401` on login | Displays `"Invalid username or password"` via `MatSnackBar` |
| Any other HTTP error on login | Displays `err.error?.message \|\| err.message \|\| 'Login failed. Please try again.'` |
| Failed list/load operations (users, products, orders) | Displays `'Failed to load [resource]: ' + err.message` via snackbar (4 s duration) |
| Failed create/update/delete operations | Displays `err.error?.message \|\| err.message \|\| 'Unknown error'` via snackbar (4 s duration) |
| Form validation failure | Submission is blocked (`if (this.form.invalid) return`); no server call is made |

**Client-side validation rules evidenced:**

| Field | Rules |
|-------|-------|
| `username` (register) | Required, minLength 3, maxLength 50 |
| `email` (register) | Required, valid email format |
| `password` (register) | Required, minLength 6 |
| `role` (register) | Required; allowed values: `USER`, `ADMIN`, `MANAGER` |
| `username` (login) | Required |
| `password` (login) | Required |
| `userId` (create order) | Required, numeric pattern `^[0-9]+$` |
| `shippingAddress` (create order) | Required |
| `productId` (order item) | Required, numeric pattern `^[0-9]+$` |
| `quantity` (order item) | Required, min 1 |
| Product `name` | Required |
| Product `price` | Required, min 0 |
| Product `stockQuantity` | Required, min 0 |
| Product `category` | Required |

---

## Versioning

No API versioning strategy (URI prefixes such as `/v1/`, version headers, or schema versioning annotations) is determinable from the frontend source code. The backend base URLs are configured as plain host:port strings (`http://localhost:808x`) with no version segment visible in this snapshot.

_Not determinable from code._
