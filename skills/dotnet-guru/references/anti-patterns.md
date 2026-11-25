# Anti-Patterns Reference

Reject these patterns unless there is a documented, exceptional reason.

## Async and Concurrency

- Sync-over-async (`.Result`, `.Wait()`) in app code.
- `async void` outside UI/event handlers.
- Fire-and-forget without supervision, cancellation, or error observation.
- Blocking I/O in ASP.NET Core request paths.

## DI and Lifetime

- Captive dependencies (scoped service injected into singleton).
- Service locator (`IServiceProvider.GetService`) in domain/app logic.
- Calling `BuildServiceProvider` during registration.
- Global static service access patterns.

## ASP.NET Core

- Capturing `HttpContext` for later/background use.
- Parallel access to mutable request context objects.
- Starting long-running jobs directly in request handlers.
- Returning huge unpaged collections.

## EF Core

- Blind lazy loading causing N+1 queries.
- Loading full entities when projection would suffice.
- Missing `AsNoTracking` in read-heavy paths.
- Unbounded queries without pagination/limits.

## Error Handling

- Catch-all `Exception` with swallow-and-continue behavior.
- Exceptions as normal control flow in hot paths.
- Missing contextual logs for actionable failures.

## Security

- Hardcoded secrets and credentials.
- String-concatenated SQL.
- Unsafe deserialization of untrusted data.
- Logging sensitive PII/tokens.

## Testing

- Heavy infrastructure dependencies in unit tests.
- Flaky tests with time/network randomness and no control seams.
- Assertions that overfit implementation details instead of behavior.
- No tests for cancellation/error paths in async APIs.

## Code Quality

- Disabling analyzers globally to silence warnings.
- Large classes with excessive constructor dependencies.
- Over-abstraction that hides simple intent.
- Premature micro-optimizations without profiling evidence.

## Better Alternatives

When a listed anti-pattern appears, replace it with:

- explicit async flows,
- correct DI lifetimes,
- bounded query and payload strategies,
- typed options with validation,
- deterministic tests,
- and measurable performance/security checks.
