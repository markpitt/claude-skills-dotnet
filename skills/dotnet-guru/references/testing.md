# Testing Reference

Use layered tests that are fast by default and realistic where needed.

## Test Pyramid

- Unit tests: pure logic, deterministic, no infrastructure.
- Integration tests: HTTP + DI + persistence behavior.
- End-to-end tests: minimal high-value happy/sad paths.

## Unit Testing

- Prefer `xUnit` with clear Arrange/Act/Assert sections.
- Use `Theory` + data rows for scenario matrices.
- Avoid logic in tests (loops/conditionals) when data-driven tests suffice.
- Keep tests isolated and repeatable.

## Integration Testing

- Use `WebApplicationFactory` for ASP.NET Core.
- Use ephemeral dependencies (e.g., Testcontainers) for DB/message broker realism.
- Seed minimal test data per scenario.
- Validate auth, error contracts, and serialisation behavior.

## Async and Cancellation Tests

- Test cancellation propagation explicitly.
- Test timeout behaviour and retry boundaries.
- Assert no deadlocks from sync-over-async patterns.

## Contract and Serialization Tests

- Snapshot/contract tests for public API payloads.
- Validate backward compatibility of response shapes.
- Test JSON naming, enum handling, and null-value behavior.

## Performance Validation

- Use BenchmarkDotNet for critical code paths.
- Track allocation regression for hot APIs.
- Keep benchmark baselines versioned when performance is a product requirement.

## Coverage and Quality

- Coverage is a signal, not a goal by itself.
- Prioritise behavior-critical branches and failure modes.
- Keep flaky tests out of main pipeline.
