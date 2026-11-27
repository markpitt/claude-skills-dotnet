# Language and API Design Reference

Use modern C# features that improve correctness, maintainability, and intent.

## Core Language Defaults

- Prefer file-scoped namespaces.
- Prefer expression-bodied members only when they remain readable.
- Prefer pattern matching and switch expressions for branching clarity.
- Prefer `required` + `init` for mandatory initialization.
- Prefer records for value-like data contracts.
- Keep domain models explicit; avoid overusing records for mutable entities.

## Nullability

- Keep nullable reference types enabled (`<Nullable>enable</Nullable>`).
- Model optionality explicitly (`string?` vs `string`).
- Avoid null-forgiving (`!`) except where invariants are proven and documented.
- Validate external input at boundaries before entering core logic.

## Constructors and Initialization

- Use primary constructors where they improve clarity.
- Use `required` members for DTOs/config models that must be initialized.
- Use `SetsRequiredMembers` only when constructor guarantees are real.

## Collections and Immutability

- Prefer read-only abstractions in public APIs (`IReadOnlyList<T>`, `IReadOnlyDictionary<TKey,TValue>`).
- Use collection expressions where supported and readable.
- Prefer immutable models at boundaries to reduce hidden state bugs.

## Exceptions and Results

- Throw exceptions for exceptional conditions, not normal control flow.
- Use domain/result types for expected failures in high-volume flows.
- Catch specific exception types and add contextual logging.

## API Surface Design

- Keep methods small and cohesive.
- Accept `CancellationToken` for async I/O operations.
- Return concrete outcome types (`Results<T1,T2>`, typed responses) for APIs.
- Version externally exposed APIs intentionally.

## DI-Friendly Design

- Depend on abstractions where substitution is meaningful.
- Keep service constructor dependencies small; many dependencies signal SRP issues.
- Avoid static mutable state and hidden service locators.

## Source Generators and Compile-Time Features

- Prefer compile-time generation when it improves startup/perf and reduces reflection.
- Examples: System.Text.Json source generation, options validation generation.
- For trim/AOT-sensitive apps, avoid heavy reflection-based magic paths.

## Practical "Prefer" Rules

- Prefer `ArgumentNullException.ThrowIfNull(...)` for guard clauses.
- Prefer `DateTimeOffset` for external timestamps.
- Prefer `TimeProvider` abstraction over direct `DateTime.UtcNow` in testable components.
- Prefer strongly-typed IDs/value objects in complex domains.

## Public Contract Stability

- Breaking changes need explicit versioning strategy.
- Avoid widening nullable contracts accidentally.
- Keep serialisation contracts explicit and test-backed.
