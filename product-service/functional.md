---
repo: product-service
spec_type: functional
commit: 4b4f2e1f136fd78b1352d3f16ac6e5489a5c8e13
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: e49333137049b1873579da841b8a4813883e9324476a408b17905fe59ee37194
generated_at: 2026-06-30T17:02:40.200968382+02:00
generator: specsync
---

## Business Purpose

The `product-service` provides centralised product catalogue management for the SpecSync platform. It exposes a RESTful API that enables consumers to create, read, update, and delete products, manage inventory stock levels, and query the catalogue by category, name, or price range. The service acts as the authoritative source of truth for product data within the platform.

## Domain Scope (DDD Bounded Context)

- **Bounded context:** Product Catalogue
- **Core aggregate:** `Product` (persisted in the `products` table), with associated value objects `Category` (enum) and monetary `price` (BigDecimal).
- **Owned entities/fields:** `id`, `name`, `description`, `price`, `stockQuantity`, `category`, `imageUrl`, `active`, `createdAt`, `updatedAt`.
- **Neighbouring contexts:** No explicit upstream/downstream integrations are evident from the code (no events published or consumed, no REST client calls to other services). Other services that need product data must call this service's API directly. _Not determinable from code_ whether an order or inventory service consumes this API.

## Use Cases / User Stories

- **As a platform consumer**, I want to list all active products (`GET /api/products`) so that I can browse the available catalogue.
- **As a platform consumer**, I want to retrieve a single product by its ID (`GET /api/products/{id}`) so that I can display its full details.
- **As a platform consumer**, I want to filter products by category (`GET /api/products/category/{category}`) so that I can narrow the catalogue to a relevant segment (ELECTRONICS, CLOTHING, BOOKS, FOOD, SPORTS, HOME, OTHER).
- **As a platform consumer**, I want to search products by name (`GET /api/products/search?q={query}`) so that I can find items using free-text keywords.
- **As a platform consumer**, I want to filter products by price range (`GET /api/products/price-range?min=&max=`) so that I can find products within a budget.
- **As a catalogue manager**, I want to create a new product (`POST /api/products`) so that new items become visible in the catalogue.
- **As a catalogue manager**, I want to update product attributes (`PUT /api/products/{id}`) so that pricing, descriptions, and metadata remain accurate.
- **As a warehouse operator**, I want to add or subtract stock quantities (`PATCH /api/products/{id}/stock`) so that inventory levels reflect actual stock.
- **As a catalogue manager**, I want to soft-delete a product (`DELETE /api/products/{id}`) so that it is hidden from consumers without permanently losing its data.

## Business Rules

- **Name is required** and must be between 2 and 255 characters; blank names are rejected (HTTP 400).
- **Price is required**, must be greater than 0 (minimum 0.01), and may have at most 10 integer digits and 2 decimal places.
- **Stock quantity cannot be negative** on create or direct update (minimum value: 0).
- **Category is required** on creation and must be one of the defined enum values: `ELECTRONICS`, `CLOTHING`, `BOOKS`, `FOOD`, `SPORTS`, `HOME`, `OTHER`; an invalid value returns HTTP 400.
- **Description** is optional and capped at 1 000 characters.
- **Image URL** is optional and capped at 512 characters.
- **Soft-delete:** Deletion sets `active = false` rather than removing the database row. Soft-deleted products are invisible to all read queries (`GET` by ID, list, category, search, price-range) and return HTTP 404.
- **Active flag** can also be toggled explicitly via the `PUT /{id}` update endpoint (by setting `active: false/true` in the request body).
- **Stock SUBTRACT operation** requires that the requested quantity does not exceed current stock; attempting to subtract more than available stock returns HTTP 409 (Conflict).
- **Stock update quantity** must be at least 1; zero or negative values are rejected (HTTP 400).
- **`createdAt` is immutable** after initial persistence (`updatable = false`); `updatedAt` is automatically refreshed on every update via `@PrePersist` / `@PreUpdate`.
- **Search with a null or blank query** falls back to returning all active products (inferred).
- **Price-range `min`** defaults to 0 if null or negative; `max` must be ≥ `min`, otherwise HTTP 400 is returned (inferred from service logic).
- **Update is partial:** Only non-null fields supplied in `PUT /{id}` are applied; unset fields retain their existing values (inferred).
