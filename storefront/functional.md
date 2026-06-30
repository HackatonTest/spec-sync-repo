---
repo: storefront
spec_type: functional
commit: aeb8a560fd0d89f7906350b77d76a9b0ac58449c
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 3ac71d18a2e48b5c79e887cae2f954d2a9191b766b1a2147673a85a417de765c
generated_at: 2026-06-30T17:03:39.742940382+02:00
generator: specsync
---

## Business Purpose

This service is an Angular 17 single-page application (SPA) that provides the customer-facing and administrative storefront UI for the SpecSync Demo platform. It acts as the browser-side client integrating three backend microservices — User Service, Product Service, and Order Service — into a unified interface. It exists to allow users and administrators to manage accounts, browse and administer the product catalog, and create and track orders through a single web application accessible at `http://localhost:4200`.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Storefront UI / Frontend Shell — this service owns no persistent domain state itself; it is the presentation layer that composes and delegates to backend contexts.
- **Core domain entities / aggregates rendered:**
  - `User` (id, username, email, role, createdAt)
  - `Product` (id, name, description, price, stockQuantity, category, imageUrl, active)
  - `Order` / `OrderItem` (id, userId, status, totalAmount, shippingAddress, items, createdAt, updatedAt)
- **Neighbouring contexts (upstream):**
  - **User Service** (`http://localhost:8081`) — upstream authority for user accounts and JWT authentication.
  - **Product Service** (`http://localhost:8082`) — upstream authority for product catalog data.
  - **Order Service** (`http://localhost:8083`) — upstream authority for order lifecycle management.
- This service is purely downstream/consumer; it publishes no events and owns no database.

## Use Cases / User Stories

- **As a visitor**, I want to log in with my username and password so that I receive a JWT token and gain authenticated access to the storefront. _(→ `LoginComponent` → `UserService.login()` → POST to User Service)_
- **As an authenticated user**, I want to log out so that my session is terminated and I am redirected to the login page. _(→ `NavbarComponent.logout()`)_
- **As an administrator**, I want to list all registered users, search/filter by username, email, or role, so that I can manage user accounts. _(→ `/users` route → `UsersComponent` → `UserService.getUsers()`)_
- **As an administrator**, I want to register a new user with a username, email, password, and role so that new accounts are created. _(→ `RegisterDialogComponent` → `UserService.register()`)_
- **As an administrator**, I want to delete a user account so that unwanted accounts are removed. _(→ `UsersComponent.deleteUser()` → `UserService.deleteUser()`)_
- **As a user**, I want to browse the full product catalog and filter products by name/description and category so that I can find relevant products. _(→ `/products` route → `ProductsComponent`)_
- **As an administrator**, I want to create a new product with name, description, price, stock quantity, category, image URL, and active status so that it becomes available in the catalog. _(→ `ProductFormDialogComponent` → `ProductService.createProduct()`)_
- **As an administrator**, I want to edit an existing product's details so that catalog information stays accurate. _(→ `ProductFormDialogComponent` (edit mode) / `ProductDetailComponent` → `ProductService.updateProduct()`)_
- **As a user**, I want to view the full detail page of a product so that I can inspect all its attributes. _(→ `/products/:id` route → `ProductDetailComponent`)_
- **As an administrator**, I want to delete a product from the catalog so that discontinued items are removed. _(→ `ProductsComponent.deleteProduct()` → `ProductService.deleteProduct()`)_
- **As a user**, I want to list all orders (filterable by status) so that I can track outstanding and historical orders. _(→ `/orders` route → `OrdersComponent`)_
- **As a user**, I want to create a new order by selecting a user ID, a shipping address, and one or more product line items with quantities so that a purchase is submitted. _(→ `CreateOrderDialogComponent` → `OrderService.createOrder()`)_
- **As an administrator**, I want to advance an order through its status lifecycle (PENDING → CONFIRMED → SHIPPED → DELIVERED, or cancel it) so that order fulfilment is tracked. _(→ `OrdersComponent.updateStatus()` → `OrderService.updateOrderStatus()`)_
- **As an administrator**, I want to delete an order so that erroneous orders are purged. _(→ `OrdersComponent.deleteOrder()` → `OrderService.deleteOrder()`)_

## Business Rules

- **Login redirect:** If a user is already authenticated (`UserService.isLoggedIn()` returns true), navigating to `/login` redirects immediately to the home page. _(enforced in `LoginComponent.ngOnInit`)_
- **Authentication error handling:** A `401` response from the login endpoint is displayed as "Invalid username or password"; other errors show a generic failure message.
- **Order creation — only active, in-stock products may be selected:** The create-order dialog filters the product list to only products where `active === true` AND `stockQuantity > 0`. _(enforced client-side in `CreateOrderDialogComponent.loadProducts()`)_
- **Order creation — minimum one line item required:** An order must contain at least one item row; the "remove item" action is disabled when only one item row remains.
- **Order creation — quantity minimum:** Each line item's quantity must be ≥ 1. _(Validators.min(1))_
- **Order creation — userId must be a positive integer:** `userId` and `productId` fields are validated against the pattern `^[0-9]+$`. _(inferred)_
- **Order status transitions are constrained to a defined flow:**
  - `PENDING` → `CONFIRMED` or `CANCELLED`
  - `CONFIRMED` → `SHIPPED` or `CANCELLED`
  - `SHIPPED` → `DELIVERED`
  - `DELIVERED` → _(terminal, no further transitions)_
  - `CANCELLED` → _(terminal, no further transitions)_
  _(enforced client-side in `OrdersComponent.getNextStatuses()`)_
- **Product fields — required:** `name`, `price` (≥ 0), `stockQuantity` (≥ 0), and `category` are required to create or save a product. `description` and `imageUrl` are optional.
- **Product active flag defaults to `true`** when creating a new product. _(inferred from form initialisation)_
- **User registration — username:** 3–50 characters, required.
- **User registration — email:** Must be a valid email format, required.
- **User registration — password:** Minimum 6 characters, required.
- **User roles available at registration:** `USER`, `ADMIN`, `MANAGER`; defaults to `USER`. _(inferred)_
- **Delete confirmations are required** before deleting a user, product, or order (browser `confirm()` dialog must be accepted).
- **Wildcard routes redirect to home:** Any unrecognised URL is redirected to `/`. _(enforced in `app.routes.ts`)_
- **Product search filtering uses debounce (300 ms)** to avoid excessive backend calls while typing. _(inferred — filtering appears to be client-side against already-loaded data)_
