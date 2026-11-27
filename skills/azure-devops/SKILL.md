---
name: azure-devops
description: Works with the Azure DevOps REST API and the az devops CLI across Boards (work items, WIQL queries, sprints), Repos (Git, pull requests, branch policies), Pipelines (builds, releases, agents, variables), Test Plans, Artifacts feeds, and organisation security. Use when writing code or scripts that call Azure DevOps, automating work items or pull requests, debugging a 401, 403 or 429 from dev.azure.com, or when asked to "create a work item from this", "query open bugs in Azure DevOps" or "trigger the pipeline from a script". Do NOT use for writing pipeline YAML with no API involved, for GitHub Actions or the GitHub API (use github-api), or for Azure resource management through ARM or the Azure CLI outside the devops extension.
license: Apache-2.0
metadata:
  version: "2.0.0"
  tags: "azure-devops, api, pipelines, boards, repos, automation"
---

# Azure DevOps API

## Credentials

Never write a PAT or token into code, a script or an example. Read it from an environment variable
such as `$AZURE_DEVOPS_PAT`, or from a secrets manager. A token pasted into a reply ends up in shell
history, logs and commits, and it grants whatever scopes it was issued with.

## Basics

Every call takes an `api-version`. Use `7.1` unless the endpoint is only in preview, in which case
the version carries a suffix such as `7.1-preview.1` and can change without notice.

```
https://dev.azure.com/{organization}/{project}/_apis/{area}/{resource}?api-version=7.1
```

Some services live on their own host:

| Service | Host |
|---|---|
| Most services | `dev.azure.com/{organization}` |
| Classic releases | `vsrm.dev.azure.com/{organization}` |
| Package feeds | `feeds.dev.azure.com/{organization}` |
| Users, groups, identities | `vssps.dev.azure.com/{organization}` |
| Analytics (OData) | `analytics.dev.azure.com/{organization}` |

A `404` from the right path on the wrong host is a common first failure.

## Authentication

Microsoft now steers new integrations to Microsoft Entra ID tokens: a managed identity or service
principal for automation, and OAuth through Entra for apps acting as a user. PATs still work and are
the quickest route for a personal script. Issue them with the narrowest scopes and a short expiry.

A PAT goes in Basic auth with an empty user name:

```bash
curl -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/{organization}/_apis/projects?api-version=7.1"
```

An Entra token goes in as a bearer token. In a pipeline, `$(System.AccessToken)` is already one,
scoped to the build identity.

## Which reference to load

| Task | Reference |
|---|---|
| Work items, WIQL, boards, sprints, backlogs | `resources/boards-work-tracking.md` |
| Git repos, commits, branches, pull requests, policies | `resources/repos-version-control.md` |
| Builds, releases, deployments, agents, variables | `resources/pipelines-cicd.md` |
| Test plans, suites, cases, runs, results | `resources/test-plans-quality.md` |
| Feeds for NuGet, npm, PyPI and Maven | `resources/artifacts-packages.md` |
| Projects, teams, users, groups, ACLs, security namespaces | `resources/organization-security.md` |
| Service hooks, extensions, wiki, search, audit | `resources/advanced-integrations.md` |

## Patterns that recur

**Paging.** Most list endpoints return a `continuationToken`, either in the body or in the
`x-ms-continuationtoken` header. Pass it back as `continuationToken` until it stops appearing. A
few older endpoints use `$top` and `$skip` instead. Code that reads only the first page looks
correct on a small project and silently drops data on a big one.

**Work item updates use JSON Patch**, sent with `Content-Type: application/json-patch+json`.
Sending plain JSON gets a `400`.

```json
[
  {"op": "add", "path": "/fields/System.Title", "value": "New title"},
  {"op": "replace", "path": "/fields/System.State", "value": "Active"},
  {"op": "add", "path": "/relations/-", "value": {
    "rel": "System.LinkTypes.Hierarchy-Reverse",
    "url": "https://dev.azure.com/{org}/_apis/wit/workItems/123"
  }}
]
```

**Queries use WIQL.** A WIQL query returns IDs only. Fetch the fields in a second call to
`wit/workitemsbatch`, at most 200 IDs at a time.

```sql
SELECT [System.Id], [System.Title] FROM WorkItems
WHERE [System.WorkItemType] = 'Bug' AND [System.State] = 'Active'
ORDER BY [System.ChangedDate] DESC
```

## Rate limits

Azure DevOps counts throughput units (TSTUs), not requests. Each call is weighted by what it
costs the service, and a user or pipeline gets 200 TSTUs in a sliding five-minute window. Read
`X-RateLimit-Remaining` and `X-RateLimit-Delay` on each response and slow down before you hit a
`429`. When one arrives, wait for `Retry-After`. Batch endpoints and caching do more than any retry
policy.

## Errors

The body usually carries a `TF` code and a `typeKey`, and they say more than the status code.

```json
{
  "message": "TF401019: The Git repository does not exist or you do not have permissions.",
  "typeKey": "GitRepositoryNotFoundException"
}
```

A `401` is a bad or expired credential. A `403`, or a `404` on something you know exists, is
usually a missing PAT scope or project permission: Azure DevOps hides resources you cannot read.

## CLI and SDKs

```bash
az extension add --name azure-devops
az devops configure --defaults organization=https://dev.azure.com/{organization} project={project}
```

The CLI reads `AZURE_DEVOPS_EXT_PAT` for non-interactive use. `az devops invoke` calls any REST
area the CLI has no command for.

| Language | Package |
|---|---|
| .NET | `Microsoft.TeamFoundationServer.Client` |
| Node.js | `azure-devops-node-api` |
| Python | `azure-devops` |

## References

- REST API: <https://learn.microsoft.com/rest/api/azure/devops/>
- Authentication guidance: <https://learn.microsoft.com/azure/devops/integrate/get-started/authentication/authentication-guidance>
- Rate limits: <https://learn.microsoft.com/azure/devops/integrate/concepts/rate-limits>
- CLI: <https://learn.microsoft.com/cli/azure/devops>
