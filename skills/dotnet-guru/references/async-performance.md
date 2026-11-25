# Async and Performance Reference

Optimize for correctness first, then measurable performance.

## Async Correctness

- Keep async flows async end-to-end.
- Avoid sync-over-async (`.Result`, `.Wait()`, blocking `Task` joins).
- Return `Task`/`Task<T>` (or `ValueTask` when justified by profiling).
- Reserve `async void` for event handlers only.

## Cancellation

- Accept `CancellationToken` on async APIs that can be cancelled.
- Pass tokens through to downstream calls (`HttpClient`, EF Core, streams).
- Treat cancellation as non-error in logs unless domain requires escalation.

## ASP.NET Core Request Path Rules

- Avoid `Task.Run` for I/O work in request handlers.
- Do not block request threads with sync I/O.
- Do not capture `HttpContext` for background operations.
- For long-running operations, offload to queue/hosted service.

## Allocation Awareness

- Optimize only hot paths proven by profiling.
- Reduce temporary allocations in loops and serializers.
- Use pooling (`ArrayPool<T>`) for large transient buffers.
- Consider `Span<T>`/`Memory<T>` where they clearly reduce copying.

## Caching and Throughput

- Cache expensive, frequently reused data with clear invalidation rules.
- Prefer bounded caches and monitor hit/miss metrics.
- Use response caching/output caching where semantics allow.

## HTTP and Networking

- Use `IHttpClientFactory` for outbound HTTP clients.
- Set sensible per-client timeout/retry policies.
- Prefer resilient handlers over ad-hoc retry loops.
- Avoid creating/disposing `HttpClient` per request.

## Data Access Performance

- Project only required columns (`Select`).
- Limit result size and paginate (`Take`, keyset pagination where applicable).
- Use `AsNoTracking` for read paths.
- Avoid lazy-loading patterns that trigger N+1 queries.
- Benchmark and inspect query plans for hot endpoints.

## Measuring Performance

- Use realistic datasets and production-like conditions.
- Measure p95/p99 latency, throughput, and allocation rate.
- Add targeted microbenchmarks for critical primitives (`BenchmarkDotNet`).
- Re-check correctness and readability after optimization.
