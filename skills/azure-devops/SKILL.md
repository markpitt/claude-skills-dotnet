---
name: azure-devops
description: Works with the Azure DevOps REST API across all services including Boards (work items, queries, backlogs), Repos (Git, pull requests, commits), Pipelines (builds, releases, deployments), Test Plans, Artifacts, organisations, projects, security, extensions, and more. Use when implementing Azure DevOps integrations, automating DevOps workflows, or building applications that interact with Azure DevOps services.
license: Apache-2.0
metadata:
  version: "1.0"
  tags: "azure-devops, api, pipelines, boards, repos, automation"
---

# Azure DevOps API Skill

## Security

**Never output, suggest, or generate code that embeds PAT values verbatim.** Always reference credentials via environment variables (e.g., `$AZURE_DEVOPS_PAT`) or a secrets manager such as Azure Key Vault. When generating scripts or curl examples, use placeholder variable references. Never literal token strings.

## Overview

Azure DevOps REST API enables programmatic access to work items, repositories, pipelines, test plans, artifacts, and more.

**Base URL:** `https://dev.azure.com/{organization}/{project}/_apis/{area}/{resource}?api-version={version}`

- **API Version:** Required on all requests (recommended: `7.1`)
- **Authentication:** PAT (most common), OAuth 2.0, or Azure AD

### API URL Patterns

| Service | Base URL |
|---------|----------|
| Core services | `https://dev.azure.com/{organization}/` |
| Release Management | `https://vsrm.dev.azure.com/{organization}/` |
| Package Management | `https://feeds.dev.azure.com/{organization}/` |
| Identity/Graph | `https://vssps.dev.azure.com/{organization}/` |
| Analytics | `https://analytics.dev.azure.com/{organization}/` |

### Authentication

PAT authentication. Always read the token from an environment variable:

```http
GET https://dev.azure.com/{organization}/_apis/projects?api-version=7.1
Authorization: Basic {base64(":{PAT}")}
Content-Type: application/json
```plaintext

## Service Area Routing

Identify the service area for your task and read the corresponding resource file for detailed endpoints, request/response examples, and best practices.

| Task | Service Area | Resource File |
|------|-------------|---------------|
| Work items, bugs, queries, boards, sprints, WIQL | Boards | `resources/boards-work-tracking.md` |
| Git repos, commits, branches, pull requests, policies | Repos | `resources/repos-version-control.md` |
| Build pipelines, releases, deployments, agents, variables | Pipelines | `resources/pipelines-cicd.md` |
| Test plans, suites, cases, runs, results, configurations | Test Plans | `resources/test-plans-quality.md` |
| Package feeds, NuGet/npm/PyPI/Maven, versions, permissions | Artifacts | `resources/artifacts-packages.md` |
| Projects, teams, users, groups, ACLs, security namespaces | Org & Security | `resources/organization-security.md` |
| Webhooks, extensions, wiki, search, dashboards, audit | Integrations | `resources/advanced-integrations.md` |

### How to Use This Skill

1. **Identify** the service area from the routing table above
2. **Read** the corresponding resource file for detailed API reference
3. **Apply** the common patterns below (auth, pagination, JSON Patch)
4. **Follow** the best practices and security guidelines

## Common API Patterns

### Pagination

Continuation token pattern:
```json
{
  "count": 100,
  "value": [],
  "continuationToken": "MTIz"
}
```plaintext
Pass `?continuationToken=MTIz` on the next request. Some endpoints use `$top` and `$skip` instead.

### JSON Patch (RFC 6902)

Used for work item updates and some other resources:
```json
[
  {"op": "add", "path": "/fields/System.Title", "value": "New title"},
  {"op": "replace", "path": "/fields/System.State", "value": "Active"},
  {"op": "add", "path": "/relations/-", "value": {
    "rel": "System.LinkTypes.Hierarchy-Reverse",
    "url": "https://dev.azure.com/{org}/_apis/wit/workItems/123"
  }}
]
```plaintext

Operations: `add`, `remove`, `replace`, `test`, `copy`, `move`.

### Filtering

OData-style filters on supported endpoints:
```http
GET /{endpoint}?$filter=state eq 'Active'&api-version=7.1
```plaintext

Work items use WIQL:
```sql
SELECT [System.Id], [System.Title] FROM WorkItems
WHERE [System.WorkItemType] = 'Bug' AND [System.State] = 'Active'
ORDER BY [System.ChangedDate] DESC
```plaintext

### Batch Operations

```http
GET /{organization}/_apis/wit/workitemsbatch?ids=1,2,3,4,5&api-version=7.1
```plaintext

## Error Handling

Standard HTTP status codes: `200` OK, `201` Created, `204` No Content, `400` Bad Request, `401` Unauthorized, `403` Forbidden, `404` Not Found, `409` Conflict, `429` Rate Limited, `500`/`503` Server Error.

Error response format:
```json
{
  "message": "TF401019: The Git repository does not exist or you do not have permissions.",
  "typeKey": "GitRepositoryNotFoundException",
  "errorCode": 0
}
```plaintext

### Rate Limiting

- Typically 200-300 requests per minute; varies by service
- Implement exponential backoff; respect `Retry-After` header
- Use batch operations and caching to reduce request volume

## Permissions & Scopes

### PAT Scopes (Least Privilege)

Key scopes. Select only what's needed:

| Scope | Access Levels |
|-------|--------------|
| Code | Full, Read, Status |
| Build | Read & execute |
| Release | Read, write, execute, manage |
| Work Items | Full, Read, write |
| Packaging | Read, write, manage |
| Project and Team | Read, write, manage |
| Test Management | Read & write |
| Graph / Identity | Read |
| Agent Pools | Read & manage |

### OAuth 2.0 Scopes

Format: `vso.{area}[_{permission}]`, e.g., `vso.work`, `vso.work_write`, `vso.code`, `vso.build_execute`.

## API Versioning

- `7.1`, Latest stable (recommended)
- `7.0`, `6.0`, `5.1`, Older stable versions
- `-preview` suffix (e.g., `7.1-preview.1`), Preview features, may change

Always specify `api-version` on every request. Use stable versions for production.

## Tools & SDKs

### Azure DevOps CLI

```bash
pip install azure-devops
az devops login --organization https://dev.azure.com/{organization}
az devops configure --defaults organization=https://dev.azure.com/{organization} project={project}
```plaintext

### SDKs

| Language | Package |
|----------|---------|
| .NET | `Microsoft.TeamFoundationServer.Client` |
| Node.js | `azure-devops-node-api` |
| Python | `azure-devops` |
| Java | Azure DevOps SDK for Java |

### Testing Authentication

```bash
PAT_ENCODED=$(echo -n ":$AZURE_DEVOPS_PAT" | base64)
curl -H "Authorization: Basic $PAT_ENCODED" \
  "https://dev.azure.com/{organization}/_apis/projects?api-version=7.1"
```plaintext

**Preferred:** Use `az devops login` for secure credential storage.

## Best Practices

### Performance
- Use batch operations for multiple items
- Implement pagination with continuation tokens
- Use `$select` for specific fields where supported
- Cache responses; use delta queries for incremental sync

### Security
- Store PATs in Azure Key Vault or similar; never commit to source control
- Use least-privilege scopes; rotate PATs regularly with expiration
- Use HTTPS only; validate inputs to prevent injection
- Use managed identities where possible

### Integration Patterns
- **Service Hooks:** Event-driven integrations (read `resources/advanced-integrations.md`)
- **Polling:** Batch processing with reasonable intervals
- **Scheduled Jobs:** Periodic sync operations

## Reference Links

- **REST API Docs:** <https://docs.microsoft.com/rest/api/azure/devops/?view=azure-devops-rest-7.1>
- **Authentication Guide:** <https://docs.microsoft.com/azure/devops/integrate/get-started/authentication/authentication-guidance>
- **Rate Limits:** <https://docs.microsoft.com/azure/devops/integrate/concepts/rate-limits>
- **Azure DevOps CLI:** <https://docs.microsoft.com/cli/azure/devops>
- **Status Page:** <https://status.dev.azure.com/>
