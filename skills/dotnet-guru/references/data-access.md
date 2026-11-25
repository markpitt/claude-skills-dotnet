# Data Access Reference (EF Core)

Prefer predictable query shapes, explicit loading, and measurable performance.

## Query Shape

- Project only required fields (`Select`).
- Limit row counts with `Take` and pagination.
- Use keyset pagination for high-scale sequential browsing.
- Avoid loading full graphs by default.

## Tracking Strategy

- Use tracked queries only when mutation is required.
- Use `AsNoTracking` for read-only endpoints.
- Consider `AsNoTrackingWithIdentityResolution` when needed.

## Loading Related Data

- Prefer explicit eager loading when relationship data is known.
- Use filtered includes to narrow child collections.
- Avoid lazy loading in request-heavy paths (N+1 risk).
- Consider split queries to reduce cartesian explosion when appropriate.

## Async Data Access

- Use async APIs consistently (`ToListAsync`, `SingleAsync`, `SaveChangesAsync`).
- Avoid mixing sync and async data access in hot paths.

## Command and Query Performance

- Validate indexes for common filters/sorts.
- Inspect execution plans for slow queries.
- Use compiled queries only when profiling shows benefit.
- Batch writes where practical, while preserving transactional correctness.

## Migrations and Schema Discipline

- Treat migrations as reviewed code artifacts.
- Keep destructive migrations explicit and reversible when possible.
- Validate migration scripts before production rollout.

## Raw SQL Usage

- Use raw SQL sparingly, only when LINQ translation is insufficient.
- Parameterize values always; never concatenate untrusted inputs.
- Keep SQL-localized logic covered by tests.

## Transaction and Consistency

- Keep transactions as short as practical.
- Avoid long-running work inside transaction scope.
- Model idempotency for retried commands in distributed systems.
