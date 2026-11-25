---
name: dotnet-github-packages-release
description: Publishes .NET NuGet packages to a GitHub Packages organisation feed, lets the organisation's other repositories restore them, and cuts versioned releases with a changelog gate, a tag, GitHub release notes and cross-platform binaries. Use when writing a nuget.config for https://nuget.pkg.github.com, packing and pushing with GITHUB_TOKEN in Actions, choosing between MinVer and an explicit version, giving a consumer repository read access to private packages, running gh release create, or debugging a failed release workflow. Do NOT use for C# language or framework advice such as nullable reference types, async code or ASP.NET design (use dotnet-guru), or for pushing to nuget.org with an API key.
license: Apache-2.0
metadata:
  version: "1.0.0"
  tags: "dotnet, nuget, github-packages, github-actions, release, changelog, minver, gh-cli"
---

# .NET packages and releases on GitHub

Publish packable .NET projects to a GitHub Packages NuGet feed owned by an organisation,
let that organisation's other repositories restore them, and release each version as a tag,
a GitHub release with notes taken from the changelog, and attached self-contained binaries.

Placeholders used throughout: `OWNER` is the organisation or user that owns the feed,
`REPO` is the repository that publishes, `CONSUMER` is a repository that restores,
`PKG` is a package id, `X.Y.Z` is a version and `my-tool` is an example application name.

## Critical rules

1. **A version ships only with a changelog section.** `CHANGELOG.md` must hold a non-empty
   `## [X.Y.Z]` section before the tag is made. The gate in `references/changelog-gate.md`
   refuses otherwise, locally and in the workflow. Missing changelog sections were a recurring
   miss; never bypass the gate by tagging by hand.
2. **Never put a token in a committed file, and keep it off command lines on shared machines.**
   A process argument is visible to `ps` and lands in shell history. `dotnet nuget push
   --api-key` necessarily takes the value as an argument, so use it only on an ephemeral CI
   runner. Elsewhere pass tokens through environment variables or stdin.
3. **`GITHUB_TOKEN` reads only its own repository's packages.** A consumer in the same
   organisation needs either a per-package access grant or a read token in a secret. Per-package
   grants do not scale; see `references/consumer-access.md`.
4. **GitHub Packages rejects `.snupkg` files.** Push `*.nupkg` only.
5. **`dotnet nuget push --skip-duplicate` silently skips a version that already exists.** It
   does not replace it. Never publish different bytes under an existing version; bump the version.
6. **MinVer needs full git history.** Use `actions/checkout` with `fetch-depth: 0`, or every
   build reports `0.0.0`.
7. **Pin third-party actions to a full commit SHA.** The tags in the examples are for reading.

## Situation to reference

| Situation | Load |
|---|---|
| Writing `nuget.config`, package metadata, or a local pack and push | `references/feed-and-nuget-config.md` |
| Choosing MinVer or an explicit version, prerelease suffixes, tags | `references/versioning.md` |
| Enforcing a changelog section, extracting release notes, bumping `[Unreleased]` | `references/changelog-gate.md` |
| Writing the publish and release workflow, `gh release create`, asset upload | `references/release-workflow.md` |
| Self-contained single-file builds for linux-x64, win-x64, osx-arm64 | `references/cross-platform-builds.md` |
| A consumer repository cannot restore a private package | `references/consumer-access.md` |
| A release workflow failed, a local dry run, or Actions minutes ran out | `references/troubleshooting-and-fallbacks.md` |

## Procedure 1: add a publishable package to a repository

1. Confirm the feed URL. It is `https://nuget.pkg.github.com/OWNER/index.json`, with the owner
   in lower case.
2. Add `nuget.config` at the repository root as set out in
   `references/feed-and-nuget-config.md` section 1. Credentials come from environment variables
   only.
3. In each packable project, set `PackageId`, `Description`, `PackageLicenseExpression`,
   `RepositoryUrl` and `PackageReadmeFile` (section 3 of the same reference).
4. Pack locally with an explicit version and inspect the result:
   ```bash
   dotnet pack src/PKG/PKG.csproj -c Release -p:PackageVersion=X.Y.Z -o artifacts/nupkg
   unzip -l artifacts/nupkg/PKG.X.Y.Z.nupkg
   ```
   The listing must show the `lib/` assemblies and the readme. Nothing else needs to be there.
5. Add the publish job from `references/release-workflow.md` with `permissions: packages: write`.

## Procedure 2: cut a release

1. Confirm the branch being released has a clean tree: `git status --short` prints nothing.
2. Rename `## [Unreleased]` to the new version and date, and add a fresh empty `[Unreleased]`
   (`references/changelog-gate.md` section 2). Review the diff.
3. Run the gate. It must exit 0 before anything else happens:
   ```bash
   bash "$GATE" X.Y.Z CHANGELOG.md; echo "gate exit $?"
   ```
   `$GATE` is the script saved from `references/changelog-gate.md`, kept outside the repository.
4. Commit the changelog change on its own (`chore(release): X.Y.Z changelog`). Push the branch
   and merge it through the repository's normal pull request flow.
5. Start the release workflow on the merged commit:
   ```bash
   gh workflow run release.yml --ref main -f version=X.Y.Z
   gh run list --workflow release.yml --limit 3
   ```
6. When the run is green, verify each output. Do not report the release as done before this:
   ```bash
   gh release view vX.Y.Z --json tagName,isPrerelease,assets --jq '{tagName, isPrerelease, assets: [.assets[].name]}'
   gh api "/orgs/OWNER/packages/nuget/PKG/versions" --jq '.[].name'
   ```
   The version must appear in the package list and the assets must match the platforms built.

## Procedure 3: consume packages from another repository

1. Choose the access model in `references/consumer-access.md`. For an organisation with several
   consumers, the read-token secret is the usual answer.
2. Add the same `nuget.config` as the publisher, with the `github-OWNER` source and the
   credentials taken from `GITHUB_PACKAGES_USER` and `GITHUB_PACKAGES_TOKEN`.
3. Store the token as a secret on the consumer, reading it from a file or stdin:
   ```bash
   gh secret set GITHUB_PACKAGES_TOKEN --repo OWNER/CONSUMER < "$TOKEN_FILE"
   gh variable set GITHUB_PACKAGES_USER --repo OWNER/CONSUMER --body "LOGIN"
   ```
   `LOGIN` is the account that owns the token. `TOKEN_FILE` is mode 600 and is deleted afterwards.
4. Map the secret into the restore step, not the whole job:
   ```yaml
   - run: dotnet restore
     env:
       GITHUB_PACKAGES_USER: ${{ vars.GITHUB_PACKAGES_USER }}
       GITHUB_PACKAGES_TOKEN: ${{ secrets.GITHUB_PACKAGES_TOKEN }}
   ```
5. Prove the source works before relying on it: `dotnet nuget list source` shows the feed as
   enabled, and `dotnet restore --force` succeeds with the variables set.

## Validation checklist

- `nuget.config` has no literal token, and the `packageSourceCredentials` values are `%VAR%` references.
- `dotnet pack` output name matches the `PackageId` and the chosen version.
- The gate script exits 0 for a present section and non-zero for a missing or empty one.
  Test it against fixtures before trusting it (see `references/changelog-gate.md` section 4).
- The release job has `contents: write` for `gh release create` and `packages: write` for the push.
- Every version pushed to the feed matches a tag and a changelog section.

## Scope

This skill covers NuGet packages on GitHub Packages and GitHub releases produced by Actions or
by a local script. It does not cover nuget.org publishing, container registries or application
hosting. C# code quality belongs to `dotnet-guru`.
