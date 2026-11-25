# Known Traps

Failure modes that present as something other than what they are. Each one
here has a misleading symptom, which is why it earns a page: the obvious
diagnosis sends you the wrong way.

## Contents

- [Testcontainers and parallel test projects contend](#testcontainers-and-parallel-test-projects-contend)
- [Plugin isolation via AssemblyLoadContext](#plugin-isolation-via-assemblyloadcontext)
- [Rebuilding the DI container per request](#rebuilding-the-di-container-per-request)
- [Rate limiting needs no package on net10.0](#rate-limiting-needs-no-package-on-net100)
- [Choosing the test layer](#choosing-the-test-layer)

## Testcontainers and parallel test projects contend

Several test projects each spinning up their own database container at the
same time produce connection contention and intermittent failures. The symptom
is a test that passes alone and fails in CI, which reads as flakiness.

It is not flakiness, and treating it as flakiness leads to a retry attribute
that hides a real resource problem. The fix is to serialise test-project
execution rather than letting the task runner parallelise it, or to share one
container across projects rather than one per project.

Before concluding either way, prove it. Run the suspect test in an N-iteration
loop on its own, then run the full suite. If failures cluster with concurrent
runs and the isolated loop is clean, it is contention. If failures are randomly
distributed in both, it is a genuinely flaky test and the container is
incidental. A test is not flaky until a repeat loop says so.

## Plugin isolation via AssemblyLoadContext

For host-plus-plugin architectures:

- Set `<EnableDynamicLoading>true</EnableDynamicLoading>` on the plugin project.
- Load with a collectible `AssemblyLoadContext` (`isCollectible: true`) so the
  plugin can be unloaded.
- Defer shared contract assemblies to the default context rather than letting
  the plugin context load its own copy.

That last point is the one that bites. If the plugin context loads its own copy
of a contract assembly, you get two distinct types with the same full name, and
a cast fails with a type appearing not to implement an interface it visibly
does implement. The error message is about type identity and gives no hint that
assembly loading is involved.

Resolve shared contracts from the default context in the load callback, and
keep the contract assembly out of the plugin's output directory.

## Rebuilding the DI container per request

Watch for provider, session or tenant abstractions that construct a service
provider on each operation rather than resolving from the existing one. It
looks harmless in a small class and is easy to miss in review.

A real instance of this rebuilt EF Core, ASP.NET Identity, the OpenIddict core
services and a fresh Data Protection key ring on every admin action. The key
ring matters most: it is expensive to construct and it is stateful, so
rebuilding it is a correctness problem as well as a performance one. Tokens and
cookies protected by one instance are not readable by the next.

Build the container once. Inject what you need, and use a scope where a scope
is what you actually want.

## Rate limiting needs no package on net10.0

`Microsoft.AspNetCore.RateLimiting` is part of the shared framework. Adding a
NuGet reference for it is unnecessary and can pin an older implementation than
the framework already provides. Use `AddRateLimiter` and the built-in limiter
policies directly.

## Choosing the test layer

| Layer | Fake strategy | Infrastructure |
|---|---|---|
| Unit | A mocking library for collaborators | None |
| Component (Blazor) | bUnit test context, mocks for injected services | None |
| Integration | Real implementations | Containerised database |
| End-to-end | None | Real hosts, real database, browser automation, dev tokens via `dotnet user-jwts` |

Two rules decide most cases. Do not fake the thing you are testing. Do not use
a real database for a unit test.

If a test needs a container to prove that a pure function works, the design is
wrong before the test is: the logic is entangled with infrastructure and should
be separated first.

For component tests, write the test before the component rather than
retrofitting one afterwards. Retrofitted component tests tend to assert the
markup that happens to exist rather than the behaviour that was wanted.
