# ASP.NET Core Web API Reference

Build APIs that are explicit, observable, and performant.

## Endpoint Design

- Keep handlers thin; delegate business logic to services.
- Validate input at the boundary.
- Use clear response contracts and status codes.
- Prefer DTOs over exposing EF entities directly.

## Middleware Order

Typical ordering for APIs:

1. Exception handling / problem details
2. HTTPS redirection (if applicable)
3. Routing
4. Authentication
5. Authorization
6. Endpoint mapping

Use repository conventions if already established.

## Error Handling

- Centralise exception-to-response mapping.
- Use Problem Details payloads for machine-readable errors.
- Log exceptions once with context; avoid duplicate noisy logs.

## Validation

- Validate request contracts consistently.
- Return structured validation errors.
- Reject oversized/invalid payloads early.

## Background Work

- Do not run unbounded fire-and-forget work in request handlers.
- Use hosted services/queues for long-running operations.
- Create explicit DI scopes in background workers when resolving scoped services.

## HttpContext Safety

- Use `HttpContext` only during active request processing.
- Do not store `HttpContext` in fields for later use.
- Do not access `HttpContext` across threads in parallel operations.

## Outbound Calls

- Use typed clients via `IHttpClientFactory`.
- Apply resilience policies and timeout budgets.
- Propagate trace context and cancellation tokens.

## Observability

- Use structured logs with templates and named properties.
- Add distributed tracing (OpenTelemetry) for cross-service diagnostics.
- Expose health endpoints for readiness/liveness.
- Track request duration, error rate, and dependency latency.

## API Versioning and Compatibility

- Version breaking changes intentionally.
- Keep backward-compatible default behaviour where feasible.
- Add integration tests for old and new contract paths.
