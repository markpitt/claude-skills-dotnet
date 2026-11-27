---
name: dotnet-guru
description: Enforces current C# and .NET 10 best practices when writing, reviewing, or refactoring .NET code. Use when the user asks for C#/.NET help, ASP.NET Core APIs, Entity Framework Core, performance tuning, testing, security hardening, or project setup. Covers analyzers and SDK tooling, nullable/reference safety, async and allocation-aware patterns, DI/options correctness, observability, and anti-pattern elimination. Do NOT use for non-.NET languages or generic architecture discussions without concrete .NET implementation details.
license: Apache-2.0
metadata:
  version: "1.1.0"
---

# Dotnet Guru

Enforce modern, production-grade C# and .NET practices on every task.
Default to the latest stable SDK available in the repository, and prefer current
LTS for new projects. Never suggest legacy patterns when a modern built-in .NET
approach exists.

On 26/11/2026 the current LTS is .NET 10 (supported to November 2028). .NET 11 shipped on
10/11/2026 as a standard-term release with the same end date; use it when the repository
already targets it or the user wants C# 15 features, and say that it is STS.

## Critical

- Treat warnings as quality signals; do not ignore analyzer warnings without reason.
- Prefer BCL/SDK-native solutions before third-party dependencies.
- Keep implementations trim-safe and AOT-aware where practical.
- Preserve readability: simple, explicit, measurable improvements over cleverness.
- For new project/setup guidance, always include concrete quality commands: `dotnet format`, `dotnet test`, and build with warnings-as-errors.
- For DTO/API design guidance, always state nullability explicitly (which properties are nullable vs non-nullable and why).
- Treat "singleton depends on scoped" as an invalid lifetime configuration (captive dependency) and say so clearly.

## Response Guardrails

- When asked to set up a .NET project, include a short "Quality Gate" block with:
  - `dotnet build -warnaserror`
  - `dotnet format --verify-no-changes`
  - `dotnet test`
- When asked about models/DTOs/contracts, include one explicit nullability note (for example, `string` vs `string?`).
- When asked about DI lifetimes, explicitly validate lifetime compatibility and call out captive dependencies.

## When to Use

- Writing new C#/.NET code (libraries, services, APIs, workers, tooling)
- Reviewing or refactoring C#/.NET code for correctness or maintainability
- Designing ASP.NET Core endpoints and middleware
- Optimising EF Core data access and query performance
- Setting up .NET project tooling, testing, and CI
- Debugging async, DI lifetimes, configuration, or performance issues

## When NOT to Use

- Non-.NET language tasks where C# is incidental
- High-level architecture with no concrete .NET implementation
- Ops-only tasks unrelated to .NET code authoring

## Enforcement Checklist

Run this checklist for **every** C#/.NET answer.

### SDK & Tooling
- [ ] Target latest stable SDK in repo (prefer current LTS for greenfield)
- [ ] `Nullable` enabled and warnings treated as errors where feasible
- [ ] `ImplicitUsings` enabled (unless project has explicit reason not to)
- [ ] Roslyn analyzers + code analysis enabled (`EnableNETAnalyzers=true`)
- [ ] `.editorconfig` and analyzer severity aligned with team rules
- [ ] `dotnet format` and `dotnet test` used in CI

### Language & API Design
- [ ] Use file-scoped namespaces and modern concise syntax when it improves clarity
- [ ] Prefer `required`/`init`/records for immutable or DTO-style models
- [ ] Prefer pattern matching and switch expressions over nested conditionals
- [ ] Use collection expressions where supported and readable
- [ ] Public APIs use cancellation tokens for async I/O paths
- [ ] Avoid `dynamic` unless interop truly requires it

### Async & Performance
- [ ] No sync-over-async (`.Result`, `.Wait()`, blocking I/O in async flows)
- [ ] No `Task.Run` inside ASP.NET Core request path for I/O work
- [ ] Use async end-to-end for I/O (`ToListAsync`, `ReadAsync`, `SendAsync`)
- [ ] Consider hot-path allocations (`ArrayPool<T>`, `Span<T>`, projection)
- [ ] Return paged results for large collections
- [ ] Avoid exception-driven control flow on hot paths

### ASP.NET Core
- [ ] Prefer minimal APIs/controllers with explicit validation and result contracts
- [ ] Register middleware in correct order (authn/authz/error handling etc.)
- [ ] Avoid capturing `HttpContext` beyond request lifetime
- [ ] Use `IHttpClientFactory` for outbound HTTP
- [ ] Use health checks, structured logging, and OpenTelemetry-ready instrumentation

### DI & Configuration
- [ ] Service lifetimes are correct (no scoped-in-singleton captive dependencies)
- [ ] Avoid service locator anti-pattern (`GetService` in app logic)
- [ ] Use Options pattern for config sections (typed options)
- [ ] Validate options (`ValidateDataAnnotations`/`ValidateOnStart` as needed)
- [ ] Avoid `BuildServiceProvider` during registrations

### Data Access (EF Core)
- [ ] Query only required columns (`Select`) and limit result size (`Take`/pagination)
- [ ] Use `AsNoTracking` for read-only queries
- [ ] Avoid N+1; prefer explicit eager loading and filtered includes
- [ ] Prefer async EF APIs consistently
- [ ] Use indexes/migrations intentionally and review query plans for hot queries

### Serialisation & Security
- [ ] Prefer `System.Text.Json` as default serialiser
- [ ] Use source-generated JSON context for trim/AOT/perf-sensitive workloads
- [ ] Validate and bound request payload sizes
- [ ] No secrets in code; use config providers/secret stores
- [ ] Parameterise SQL and avoid unsafe deserialisation patterns

### Testing
- [ ] Unit tests are deterministic and isolated (`xUnit` v3; an assertion library is optional)
- [ ] Integration tests use realistic hosts (`WebApplicationFactory`, Testcontainers)
- [ ] Use `Theory` for data-driven tests instead of logic-heavy test bodies
- [ ] Include cancellation/error-path tests for async methods
- [ ] Add performance-focused benchmarks for critical paths (`BenchmarkDotNet`)

### Anti-Patterns (instant fail)
- [ ] No `async void` (except event handlers)
- [ ] No `.Result`/`.Wait()` in application code
- [ ] No fire-and-forget tasks without explicit supervision strategy
- [ ] No singleton services that capture scoped dependencies
- [ ] No EF lazy-loading reliance in high-throughput request paths
- [ ] No broad `catch (Exception)` without meaningful handling and rethrow policy

## Quick Reference: Old → Modern

| Avoid | Prefer |
|---|---|
| `.Result` / `.Wait()` | `await` all the way |
| `new HttpClient()` per call | `IHttpClientFactory` |
| Stringly-typed config reads | Options pattern + validation |
| `Newtonsoft.Json` by default | `System.Text.Json` (source-gen where useful) |
| Loading full entities always | Projection with `Select(...)` |
| Tracked reads by default | `AsNoTracking()` for read-only |
| Service locator access | Constructor injection |
| Manually created scopes everywhere | Proper scoped lifetimes + hosted services |

## New Project Bootstrap

Use this baseline for a modern API/service:

```text
my-dotnet-app/
├── src/
│   └── MyApp/
│       ├── Program.cs
│       ├── appsettings.json
│       └── Features/
├── tests/
│   ├── MyApp.UnitTests/
│   └── MyApp.IntegrationTests/
├── Directory.Build.props
├── .editorconfig
└── global.json
```

Recommended setup commands. FluentAssertions needs a paid licence for commercial use from v8
onwards, so do not add it by default: use xUnit's own `Assert`, or Shouldly if the team wants
fluent assertions.

```bash
dotnet new webapi -n MyApp
cd MyApp
dotnet new install xunit.v3.templates
dotnet new xunit3 -n MyApp.UnitTests -o ../tests/MyApp.UnitTests
dotnet new xunit3 -n MyApp.IntegrationTests -o ../tests/MyApp.IntegrationTests
dotnet add ../tests/MyApp.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing
dotnet add ../tests/MyApp.IntegrationTests package Testcontainers
```

## Validation Guidelines

Before final output:

1. Confirm code targets project SDK constraints.
2. Re-check nullability, lifetimes, and cancellation behaviour.
3. Re-check query shape, serialisation defaults, and secret handling.
4. Provide the simplest safe implementation that passes analyzers/tests.
5. If tradeoffs exist, state them explicitly with measurable impact.

## References

Load these files for depth as needed:

- `references/tooling.md`: SDK pinning, analyzers, formatting, CI
- `references/language-design.md`: modern C# language and API design patterns
- `references/async-performance.md`: async correctness and performance patterns
- `references/webapi.md`: ASP.NET Core endpoint, middleware, observability
- `references/data-access.md`: EF Core querying, performance, migrations
- `references/testing.md`: unit/integration testing and benchmark practices
- `references/security.md`: secure coding and hardening for .NET services
- `references/anti-patterns.md`: common .NET anti-patterns and remediations
- `references/known-traps.md`: failure modes whose symptom points the wrong way
