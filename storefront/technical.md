---
repo: storefront
spec_type: technical
commit: aeb8a560fd0d89f7906350b77d76a9b0ac58449c
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 3ac71d18a2e48b5c79e887cae2f954d2a9191b766b1a2147673a85a417de765c
generated_at: 2026-06-30T17:03:39.742940382+02:00
generator: specsync
---

## Tech Stack

| Concern | Detail |
|---|---|
| **Language** | TypeScript ~5.4.0 |
| **Runtime** | Node.js 18+ (build/dev tooling only; output is static HTML/JS/CSS) |
| **Framework** | Angular 17.3.x (standalone components API) |
| **UI Component Library** | Angular Material 17.3.x (`@angular/material`, `@angular/cdk`) |
| **Reactive primitives** | RxJS ~7.8.0 |
| **Build toolchain** | Angular CLI 17.3.x / `@angular-devkit/build-angular` |
| **Polyfills** | `zone.js` ~0.14.0, `tslib` ^2.3.0 |

No server-side runtime is included; this service produces a browser-side single-page application.

## Architecture Patterns

**Single-Page Application (SPA) — Angular feature-module-free, standalone-component architecture.**

Key structural patterns evident in the source:

- **Lazy-loaded route-level components**: All top-level routes (`/`, `/products`, `/products/:id`, `/orders`, `/users`, `/login`) are loaded via `loadComponent()`, minimising the initial bundle.
- **Service layer abstraction**: Dedicated injectable services (`UserService`, `ProductService`, `OrderService`) encapsulate all HTTP communication, keeping components free of transport logic.
- **Reactive forms**: `ReactiveFormsModule` / `FormBuilder` is used for all data-entry dialogs (login, register, create/edit product, create order).
- **Dialog-driven CRUD**: Create and edit operations are presented in `MatDialog` overlay components (`ProductFormDialogComponent`, `CreateOrderDialogComponent`, `RegisterDialogComponent`), keeping list views decoupled from mutation flows.
- **Client-side filtering**: Product search (debounced `FormControl`) and order status filtering are performed in-memory after a full list fetch.
- **JWT-based authentication**: `UserService` manages JWT storage and exposes `isLoggedIn()` / `logout()` helpers consumed by the navbar and login guard logic.

Internal component tree:

```
AppComponent
├── NavbarComponent          (persistent shell)
└── RouterOutlet
    ├── HomeComponent
    ├── ProductsComponent
    │   ├── ProductFormDialogComponent  (dialog)
    │   └── ProductDetailComponent     (route /products/:id)
    ├── OrdersComponent
    │   └── CreateOrderDialogComponent (dialog)
    ├── UsersComponent
    │   └── RegisterDialogComponent    (dialog)
    └── LoginComponent
```

## Database & Data Ownership

This service is a **browser-side frontend**; it owns no server-side datastore and performs no database migrations. All persistent state is managed by the three downstream backend services it calls.

The only client-side state is:

- **JWT token** — stored in browser memory / local storage by `UserService` (exact storage mechanism _not determinable from code_).
- **In-memory list caches** — held transiently in component properties; discarded on navigation.

## Dependencies

### Runtime (shipped to browser)

| Dependency | Purpose |
|---|---|
| `@angular/core`, `common`, `compiler`, `platform-browser`, `platform-browser-dynamic` | Angular framework core |
| `@angular/router` | Client-side routing |
| `@angular/forms` | Template and reactive forms |
| `@angular/animations` | Material animation support |
| `@angular/material` + `@angular/cdk` | UI component library (tables, dialogs, snackbars, toolbars, chips, badges, etc.) |
| `rxjs` ~7.8.0 | Observables, operators (`debounceTime`, `distinctUntilChanged`) |
| `zone.js` ~0.14.0 | Angular change-detection integration |
| `tslib` ^2.3.0 | TypeScript runtime helpers |

### Build / Dev only

| Dependency | Purpose |
|---|---|
| `@angular/cli` | CLI scaffolding and `ng serve` / `ng build` |
| `@angular-devkit/build-angular` | Webpack/esbuild build pipeline |
| `@angular/compiler-cli` | AOT template compilation |
| `typescript` ~5.4.0 | Type checking and transpilation |

### External service dependencies (runtime HTTP calls)

| Service | Base URL (default) | Used for |
|---|---|---|
| **User Service** | `http://localhost:8081` | Registration, login (JWT), user list, user deletion |
| **Product Service** | `http://localhost:8082` | Product CRUD, product list/detail |
| **Order Service** | `http://localhost:8083` | Order list, create order, update order status, delete order |

No message broker, cache, or third-party external API integrations are present.

## Deployment Model

**Build**

```bash
npm run build   # ng build
```

Output is a fully static asset bundle written to `dist/storefront/`. No server process is required to run the compiled application; it can be served by any static file server or CDN.

**Development server**

```bash
npm start       # ng serve --port 4200
```

Serves the app at `http://localhost:4200` with live-reload.

**Container / orchestration**

_Not determinable from code._ No `Dockerfile`, `docker-compose.yml`, Kubernetes manifests, or Helm charts are present in the repository snapshot.

**Ports**

| Mode | Port |
|---|---|
| Development (`ng serve`) | `4200` (TCP) |
| Production | Determined by the static file host; not specified in source |

**Environment configuration**

Angular environment files (`src/environments/`) are not included in the snapshot. Backend base URLs are hardcoded to `localhost:808x` in service classes (or environment files); the exact injection mechanism _not determinable from code._

**Health / readiness endpoints**

None — this is a static SPA; health probes would be the responsibility of the serving infrastructure (e.g., an HTTP 200 check on `/`).
