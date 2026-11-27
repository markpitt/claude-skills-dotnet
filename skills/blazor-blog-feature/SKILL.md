---
name: blazor-blog-feature
description: Adds a markdown blog to an existing Blazor WebAssembly app hosted on Azure Static Web Apps, with an Azure Functions (isolated worker) API that reads posts from an Azure File Share. Covers the shared models, YAML frontmatter parsing with YamlDotNet, Markdig rendering, the list and post pages, app settings, local development and testing. Use when asked to "add a blog to this Blazor site", "serve markdown posts from Azure storage" or "build a posts page and API for the Static Web App". Do NOT use for a Blazor Server or Blazor Web App with server rendering (use blazor-expert), for a static site generator, or for general Static Web Apps configuration (use azure-swa).
license: Apache-2.0
metadata:
  version: "3.0.0"
---

# Blog feature for Blazor WebAssembly on Static Web Apps

This builds a small blog with no database. Posts are markdown files with YAML frontmatter in an
Azure File Share. A Functions API lists and returns them, and two Blazor pages render them. It suits
a personal or company site with tens to a few hundred posts. Past that, or with several authors
editing at once, a CMS or a blob container with an index will serve better.

## What gets built

| Layer | Pieces |
|---|---|
| Shared | `BlogPost` and `BlogPostMetadata` models, used by both projects |
| API | `BlogStorageService` over `Azure.Storage.Files.Shares`, frontmatter parsing with `YamlDotNet`, `GetBlogPosts` and `GetBlogPost` HTTP functions |
| Client | `Pages/Blog/Index.razor` and `Pages/Blog/Post.razor`, markdown rendered with `Markdig`, styles |
| Storage | A share holding `blog-posts/{slug}.md` and `blog-posts/images/` |

## Before starting

Check that the solution already has a Blazor WebAssembly client and a Functions API project on the
isolated worker, both on .NET 10 or later, and that it deploys to Static Web Apps. If it has no API
project yet, create one first. This skill adds to an existing app rather than scaffolding one.

## Which reference to load

| Step | Reference |
|---|---|
| Architecture, project layout, shared models | `resources/core-architecture.md` |
| Storage service, frontmatter parsing, functions, DI | `resources/backend-services.md` |
| Razor pages, markdown rendering, styling | `resources/frontend-components.md` |
| App settings, file share, local run, deployment | `resources/azure-configuration.md` |
| Sample posts, test checklist, troubleshooting | `resources/sample-content-testing.md` |

Build in that order: models, then the API, then the pages, then configuration. The pages are hard
to check until the API returns real posts.

## Things that catch people out

- **Azurite does not emulate Azure Files.** It covers the Functions host storage
  (`AzureWebJobsStorage`), but the blog share needs a real storage account, ideally a separate
  development one. The API reads the share from its own setting, `BlogStorageConnection`, so the two
  can point at different places.
- **Browse through the SWA CLI locally**, on `http://localhost:4280`. The Blazor dev server's own
  port skips the route rules, so a deep link that works there can still 404 in Azure.
- **Rendered markdown is HTML you inject into the page.** If anyone other than you writes posts,
  sanitise the Markdig output or disable raw HTML in the pipeline, or a post can carry script.
- **Listing reads every file.** Fine for a few hundred posts. Past that, cache the list in the API or
  write an index file when posts are published.
- **The connection string is a full-access key.** Keep it in app settings, never in
  `staticwebapp.config.json` or the client, and prefer a SAS token scoped to the share or a managed
  identity where the hosting plan allows it.
