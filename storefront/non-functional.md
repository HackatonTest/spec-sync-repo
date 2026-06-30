---
repo: storefront
spec_type: non_functional
commit: aeb8a560fd0d89f7906350b77d76a9b0ac58449c
model: openai-compatible:claude-sonnet-4-6
prompt_version: v1
input_hash: 3ac71d18a2e48b5c79e887cae2f954d2a9191b766b1a2147673a85a417de765c
generated_at: 2026-06-30T17:03:39.742940382+02:00
generator: specsync
---

## Performance

- The application is an Angular 17 single-page application (SPA) served statically; runtime performance is therefore primarily governed by browser execution and network round-trips to backend services rather than server-side configuration.
- Route-level lazy loading is applied to all feature components (`loadComponent` with dynamic `import()` in `app.routes.ts`), reducing initial bundle size and improving first-paint latency.
- Product search applies a `debounceTime(300)` and `distinctUntilChanged()` operator on the search `FormControl`, throttling filter evaluations to at most one per 300 ms of idle input.
- Client-side filtering for products and users is performed entirely in memory after an initial bulk fetch; no paginated or server-side search API calls are made.
- No explicit HTTP timeout values, retry delays, or cache-control headers are configured in the visible source. Backend HTTP timeouts are delegated to Angular's default `HttpClient` behaviour (browser-level).
- Build output is written to `dist/storefront/` via `ng build`; production optimisation (tree-shaking, minification, differential loading) is handled by `@angular-devkit/build-angular` at build time.
- No service-worker or PWA caching configuration is evident.

_Not determinable from code_: specific latency/throughput SLOs, CDN or HTTP cache headers, bundle size budgets.

## Scalability

- As a compiled static SPA, the application itself is fully stateless; all session state is held in the browser (JWT token stored client-side via `UserService`).
- Horizontal scaling is achieved by serving the static build artefacts (`dist/storefront/`) from any number of web-server or CDN instances without coordination.
- No server-side replica counts, autoscaling rules, or container resource limits are defined within this repository (no Kubernetes manifests, Dockerfile, or CI/CD pipeline files are present in the snapshot).
- Backend service URLs are hardcoded to `localhost` ports (8081, 8082, 8083) in the README, indicating the current configuration targets local development only; environment-specific URL configuration for production scaling is not evident in the source.

_Not determinable from code_: replica counts, autoscaling policies, resource requests/limits.

## Security

- **Authentication**: JWT-based authentication is implemented. `UserService.login()` submits credentials and receives an `AuthResponse` containing a `token`, which is stored client-side. `UserService.isLoggedIn()` guards the login route to redirect already-authenticated users.
- **Authorisation**: Role-aware UI is present (`ADMIN`, `USER`, `MANAGER` roles rendered with distinct colours/chips); however, enforcement of role-based access control is UI-only — no route guards (`CanActivate`) or HTTP interceptors enforcing role checks are visible in the source.
- **Transport security**: No TLS/HTTPS configuration is present in the codebase. Backend URLs in the README use plain `http://`. TLS termination, if any, must be provided externally.
- **Secrets handling**: No API keys or secrets are embedded in source. The JWT token handling is delegated to `UserService`; storage mechanism (localStorage, sessionStorage, memory) is not visible in the sampled files.
- **Input validation**: Angular Reactive Forms validators are applied on all user-facing forms:
  - Username: `required`, `minLength(3)`, `maxLength(50)`
  - Email: `required`, `email`
  - Password: `required`, `minLength(6)`
  - Product price/stock: `required`, `min(0)`
  - Order quantity: `required`, `min(1)`
  - Numeric IDs: `pattern('^[0-9]+$')`
- **HTTP interceptors**: No `HttpInterceptor` attaching `Authorization` headers is visible in the sampled source; it may exist in unsampled files but cannot be confirmed.
- No Content Security Policy (CSP), CORS configuration, or XSS-specific sanitisation beyond Angular's built-in DOM sanitisation is evident.

_Not determinable from code_: JWT storage location, presence of auth interceptor in unsampled files, HTTPS enforcement.

## Observability

- **Logging**: No structured logging library or remote log-shipping is configured. User-facing feedback is provided exclusively through Angular Material `MatSnackBar` notifications (3–4 second auto-dismiss) for success and error states.
- **Metrics**: No metrics instrumentation (e.g., OpenTelemetry, Prometheus client) is present.
- **Tracing**: No distributed tracing instrumentation is present.
- **Health/readiness endpoints**: Not applicable for a static SPA; no server process exposes health endpoints from within this service. The development server (`ng serve --port 4200`) does not expose a health check endpoint.
- **Error visibility**: HTTP errors are caught in `.subscribe({ error })` handlers throughout all components and surfaced to the user via snack-bar messages including the HTTP status code or error message body where available (e.g., 401 distinguished from generic errors in `LoginComponent`).

_Not determinable from code_: production server (nginx/Apache) access logs, external APM integration, alerting rules.

## Reliability

- **Error handling**: All HTTP calls use RxJS `subscribe` with explicit `error` callbacks; failures display a user-visible snack-bar message and reset loading state, preventing the UI from becoming permanently stuck.
- **Retries**: No automatic HTTP retry logic (e.g., RxJS `retry` or `retryWhen`) is implemented. Failed requests require manual user re-initiation.
- **Circuit breakers**: None present.
- **Idempotency**: No idempotency keys or deduplication mechanisms are applied to mutation requests (create order, create product, register user). Duplicate submission on network error is possible.
- **Optimistic UI**: Not used; all mutations reload the full list from the server on success, ensuring consistency at the cost of an additional round-trip.
- **Offline/resilience**: No service-worker, offline fallback, or request queuing is configured; the application has no capability when backend services are unavailable.
- **Data consistency**: Order status transitions are enforced client-side via a hard-coded state machine (`getNextStatuses`), limiting invalid status progressions in the UI, though server-side enforcement is not verifiable from this codebase.
- **Availability target**: _Not determinable from code._
