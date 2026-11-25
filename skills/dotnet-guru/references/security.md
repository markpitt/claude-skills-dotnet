# Security Reference

Apply secure defaults and fail safely.

## Input and Boundary Validation

- Validate all external input (HTTP, queue, file, config).
- Enforce size limits on payloads and uploads.
- Fail fast on invalid contracts.

## Secrets and Configuration

- Never hardcode secrets in source code.
- Use environment/secret providers or platform key vaults.
- Rotate credentials and prefer managed identity where available.

## Data Access Security

- Parameterize SQL commands always.
- Avoid string interpolation in SQL/command text.
- Restrict DB permissions to least privilege.

## Serialization Security

- Prefer `System.Text.Json` defaults for modern safety and performance.
- Avoid polymorphic deserialization of untrusted payloads unless strictly controlled.
- Keep serialization options explicit for externally exposed contracts.

## AuthN/AuthZ

- Enforce authentication and authorization at endpoint boundaries.
- Use policy-based authorization for non-trivial rules.
- Avoid ad-hoc role checks scattered across handlers.

## HTTP Security

- Enforce HTTPS and secure headers where applicable.
- Set strict timeout budgets for outbound calls.
- Validate TLS and certificate behavior in production environments.

## Logging and Privacy

- Never log secrets, tokens, or sensitive PII.
- Structure logs for auditability without overexposure.
- Redact sensitive fields at source.

## Dependency and Supply Chain

- Keep packages patched and monitor vulnerabilities.
- Minimize dependencies in critical paths.
- Pin and review transitive dependencies for production services.

## Threat-Aware Practices

- Treat deserialization, file parsing, and dynamic code paths as high-risk.
- Validate assumptions with security tests for critical endpoints.
- Add rate-limiting, abuse detection, and circuit breakers for public APIs.
