# Tooling Reference

Use a modern, SDK-first toolchain and enforce quality at build time.

## SDK Baseline

- Pin SDK with `global.json` for reproducible local + CI builds.
- Prefer current LTS SDK for new services unless platform constraints require STS.
- Keep SDK updates frequent and deliberate; avoid multi-year lag.

Example `global.json`:

```json
{
  "sdk": {
    "version": "<installed-lts-sdk-version>",
    "rollForward": "latestFeature"
  }
}
```

Use an SDK version already present in your environment (for example from `dotnet --list-sdks`) and keep roll-forward policy explicit.

If the repository already pins another version, follow repository constraints.

## Project-Level Quality Defaults

Use `Directory.Build.props` to enforce shared defaults:

```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnableNETAnalyzers>true</EnableNETAnalyzers>
    <AnalysisLevel>latest</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

Tune warning strictness per repository maturity if needed.

## Analyzer Strategy

- Use built-in .NET analyzers first (reliability, perf, security, style).
- Configure `.editorconfig` severities intentionally.
- Avoid mass suppressions; document every suppression with rationale.
- Keep analyzers in CI to prevent regressions.

Useful command set:

```bash
dotnet build -warnaserror
dotnet format --verify-no-changes
dotnet test --configuration Release
```

## Formatting

- Use `.editorconfig` as single source of style truth.
- Keep formatting deterministic in local and CI (`dotnet format`).
- Prefer auto-fix over style debates.

## Package Management

- Keep dependencies minimal and up-to-date.
- Use central package management (`Directory.Packages.props`) for multi-project repos.
- Prefer stable packages for core runtime paths.
- Watch for transitive vulnerabilities (audit in CI).

## CI Baseline

Typical pipeline stages:

1. Restore (`dotnet restore`)
2. Build with warnings as errors (`dotnet build -warnaserror`)
3. Format verification (`dotnet format --verify-no-changes`)
4. Tests (`dotnet test`)
5. Publish smoke check for service apps (`dotnet publish`)

For AOT/trim-aware apps, include publish checks:

```bash
dotnet publish -c Release -r linux-x64 -p:PublishTrimmed=true
dotnet publish -c Release -r linux-x64 -p:PublishAot=true
```

Only enforce AOT in CI when the app explicitly targets AOT. Native AOT requires a concrete RID (for example `linux-x64`) and compatible dependencies/native toolchain; treat trim/AOT warnings as blockers before enabling broadly.

## Observability Tooling

- Emit structured logs (`ILogger` templates, no string concatenation).
- Add OpenTelemetry traces/metrics for production services.
- Use health checks and readiness/liveness endpoints in hosted services.

## Practical Defaults by App Type

- **Library**: strict analyzers, API compatibility checks, source link, docs.
- **Web API**: analyzers + ASP.NET Core performance checks + integration tests.
- **Worker**: cancellation-aware loops, health telemetry, graceful shutdown tests.
- **CLI/tooling**: fast startup, clear exit codes, deterministic output.
