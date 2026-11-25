# The feed, nuget.config and package metadata

Contents

1. nuget.config with the GitHub feed
2. Local credentials without argv exposure
3. Package metadata in the project file
4. Pack and push commands
5. Checks after a push

## 1. nuget.config with the GitHub feed

Place this at the repository root. It keeps nuget.org for public packages and adds the
organisation feed. Credentials are `%VAR%` references, which NuGet expands at restore or push
time, so no secret is written to the file.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github-OWNER" value="https://nuget.pkg.github.com/OWNER/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github-OWNER>
      <add key="Username" value="%GITHUB_PACKAGES_USER%" />
      <add key="ClearTextPassword" value="%GITHUB_PACKAGES_TOKEN%" />
    </github-OWNER>
  </packageSourceCredentials>
</configuration>
```

Notes:

- The source key (`github-OWNER`) must match the element name under
  `packageSourceCredentials`. A mismatch gives a 401 that looks like a token problem.
- `<clear />` stops inherited sources from the user-level NuGet config leaking into the build.
  Remove it only if the repository deliberately uses another feed.
- In the publishing repository, `GITHUB_PACKAGES_TOKEN` is the workflow's `GITHUB_TOKEN`
  (see `release-workflow.md`). `GITHUB_PACKAGES_USER` can be `${{ github.actor }}` there.
- Verify the `%VAR%` expansion once with a real restore. A run that prints nothing is not proof.

## 2. Local credentials without argv exposure

Do not run `dotnet nuget update source ... --password "$(gh auth token)"`. The token then
appears in the process list. Export it into the environment instead, so NuGet reads it from
`%GITHUB_PACKAGES_TOKEN%`:

```bash
gh auth refresh --hostname github.com --scopes read:packages   # one-off, interactive
export GITHUB_PACKAGES_USER="$(gh api user --jq .login)"
export GITHUB_PACKAGES_TOKEN="$(gh auth token)"
dotnet restore
```

Only the restore reads the variables. Unset them afterwards with `unset GITHUB_PACKAGES_TOKEN`.
For a consumer on a shared machine, use a dedicated token rather than the interactive `gh`
token (see `consumer-access.md`).

If a source must be registered by hand, add it with the variables still unexpanded:

```bash
dotnet nuget add source "https://nuget.pkg.github.com/OWNER/index.json" \
  --name github-OWNER --username '%GITHUB_PACKAGES_USER%' --password '%GITHUB_PACKAGES_TOKEN%' \
  --store-password-in-clear-text
```

Single quotes keep the literal `%...%` text, so the stored file stays free of the token.

## 3. Package metadata in the project file

Put shared metadata in `Directory.Build.props` so each `.csproj` stays short.

```xml
<Project>
  <PropertyGroup>
    <Authors>OWNER</Authors>
    <Company>OWNER</Company>
    <RepositoryUrl>https://github.com/OWNER/REPO</RepositoryUrl>
    <RepositoryType>git</RepositoryType>
    <PackageLicenseExpression>Apache-2.0</PackageLicenseExpression>
    <PackageReadmeFile>README.md</PackageReadmeFile>
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <EmbedUntrackedSources>true</EmbedUntrackedSources>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  </PropertyGroup>
</Project>
```

Per project:

```xml
<PropertyGroup>
  <PackageId>PKG</PackageId>
  <Description>One line that says what the package does.</Description>
  <IsPackable>true</IsPackable>
</PropertyGroup>
<ItemGroup>
  <None Include="../../README.md" Pack="true" PackagePath="\" />
</ItemGroup>
```

`IncludeSymbols` and `snupkg` produce a symbol package that GitHub Packages refuses. Keep them
for local or nuget.org packing if wanted, and push only `*.nupkg`. Alternatively set
`<IncludeSymbols>false</IncludeSymbols>` in the publishing repository.

Set `PackageLicenseExpression` to the licence the repository actually carries, and do not
copy this example's value without checking.

## 4. Pack and push commands

Pack every packable project in one pass, with one explicit version:

```bash
VERSION=X.Y.Z
rm -rf artifacts/nupkg
for p in PKG PKG.Extras; do
  dotnet pack "src/$p/$p.csproj" -c Release -p:PackageVersion="$VERSION" -o artifacts/nupkg
done
ls artifacts/nupkg
```

Push only the `.nupkg` files. The glob is quoted so the shell does not expand it early:

```bash
dotnet nuget push "artifacts/nupkg/*.nupkg" \
  --api-key "$GITHUB_PACKAGES_TOKEN" \
  --source "https://nuget.pkg.github.com/OWNER/index.json" \
  --skip-duplicate
```

`--api-key` is the one place a token reaches a process argument list: the shell expands the
variable before `dotnet` starts. Accept that only on an ephemeral CI runner, where the reference
workflows do the same with `secrets.GITHUB_TOKEN`. Never run this push with a personal token on
a shared machine; use the CI path instead. Pass the value with `env:` on the step, not by
writing it into a file. GitHub masks secret values in logs, but it does not mask a value you
echo after transforming it, so do not print it.

## 5. Checks after a push

```bash
gh api "/orgs/OWNER/packages/nuget/PKG/versions" --jq '.[] | {name, created_at}'
```

The new version must be listed. If the list is empty, the push went elsewhere or was skipped
as a duplicate; check the push step's log for `Skipped` lines.
