<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/assets/banner-dark.svg">
    <img src="docs/assets/banner-light.svg" alt=".NET skills" width="100%">
  </picture>
</h1>

.NET, Blazor, Avalonia, Aspire and the Azure services .NET shops use.

The .NET consultant's shelf. `azure-swa` sits here because `blazor-blog-feature` builds on it, and `azure-devops` because the people who use it are .NET shops. `dotnet-guru` belongs to the language-guru family as well; it lives here so this group has its anchor.

This is one of seven skill groups. [claude-skills](https://github.com/markpitt/claude-skills) lists them all and
holds the shared tooling: validation, packaging, the evaluation harness and the TUI.

## Skills

<!-- BEGIN GENERATED SKILL INDEX -->
| Skill | Bundled files | What it does |
|---|---|---|
| [avalonia](skills/avalonia/SKILL.md) | 6 files | Expert guidance for developing cross-platform desktop applications with Avalonia UI framework. |
| [azure-devops](skills/azure-devops/SKILL.md) | 7 files | Works with the Azure DevOps REST API across all services including Boards (work items, queries, backlogs)… |
| [azure-swa](skills/azure-swa/SKILL.md) | 6 files | Expert guidance for Azure Static Web Apps including architecture, configuration, API integration with Azure… |
| [blazor-blog-feature](skills/blazor-blog-feature/SKILL.md) | 5 files | Adds a complete blog feature to an existing Blazor WebAssembly Static Web App with Azure Functions backend… |
| [blazor-expert](skills/blazor-expert/SKILL.md) | 6 files | Blazor development expertise covering Blazor Server, WebAssembly, and Hybrid apps. |
| [dotnet-aspire](skills/dotnet-aspire/SKILL.md) | 6 files | Adds .NET Aspire cloud-native orchestration to existing .NET solutions. |
| [dotnet-github-packages-release](skills/dotnet-github-packages-release/SKILL.md) | 7 files | Publishes .NET NuGet packages to a GitHub Packages organisation feed, lets the organisation's other… |
| [dotnet-guru](skills/dotnet-guru/SKILL.md) | 9 files | Enforces current C# and .NET 10 best practices when writing, reviewing, or refactoring .NET code. |

8 skills.
<!-- END GENERATED SKILL INDEX -->

## Install

Copy a skill directory into either skills location, then start a new session:

```bash
cp -r skills/<skill-name> ~/.claude/skills/    # personal, all projects
cp -r skills/<skill-name> .claude/skills/      # this project only
```

For Claude Desktop, zip the skill directory's contents and upload the zip under Settings,
Capabilities, Skills:

```bash
cd skills/<skill-name> && zip -r ../../<skill-name>.zip .
```

## Licence

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Each skill directory carries its
own copy of the licence, so a skill copied out on its own stays covered.
