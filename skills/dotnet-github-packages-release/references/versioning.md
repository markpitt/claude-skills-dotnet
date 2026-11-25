# Versioning: MinVer or an explicit version

Contents

1. Choosing the scheme
2. MinVer from git tags
3. Explicit versions
4. Semantic versions and prerelease suffixes
5. Mapping versions to tags and releases
6. Pitfalls

## 1. Choosing the scheme

| Use | When |
|---|---|
| MinVer | Versions should follow git tags, and untagged builds should get a prerelease version automatically. Fits repositories that release from tags and ship many builds between releases. |
| Explicit `PackageVersion` | Releases are cut from a workflow input, the version is the only source of truth, and the repository has no tag history to rely on. |

Use one scheme per repository. Mixing them lets a tag and a package disagree.

## 2. MinVer from git tags

Add the package to each packable project, or to `Directory.Build.props` for every project:

```xml
<ItemGroup>
  <PackageReference Include="MinVer" PrivateAssets="all" />
</ItemGroup>
```

Set the tag prefix explicitly so the mapping is visible in review:

```xml
<PropertyGroup>
  <MinVerTagPrefix>v</MinVerTagPrefix>
  <MinVerDefaultPreReleaseIdentifiers>preview.0</MinVerDefaultPreReleaseIdentifiers>
</PropertyGroup>
```

Behaviour to expect:

- A commit exactly on `v1.4.0` builds version `1.4.0`.
- A commit after `v1.4.0` builds a prerelease of the next patch, for example `1.4.1-preview.0.3`,
  where `3` is the commit height since the tag.
- With no tag at all, the version is `0.0.0-...`. That is a missing-history or missing-tag problem,
  not a version to publish.

Print what MinVer computed before packing, so the log shows it. Read the build output directly
rather than piping it through a filter that could hide a failed build:

```bash
dotnet build src/PKG/PKG.csproj -c Release -p:MinVerVerbosity=normal
```

The MinVer messages in the log name the version the pack step will use. Check the exact wording
on your MinVer version before scripting against it.

The checkout must carry the tags and full history:

```yaml
- uses: actions/checkout@<full-sha> # v4
  with:
    fetch-depth: 0
```

## 3. Explicit versions

Pass the version to `dotnet pack` and never to the project file:

```bash
dotnet pack src/PKG/PKG.csproj -c Release -p:PackageVersion=X.Y.Z -o artifacts/nupkg
```

`PackageVersion` takes precedence over `Version` and `VersionPrefix` for packing. Setting only
`Version` in the project and a different `PackageVersion` on the command line gives the command-line
value, which is the one that is tested against the changelog gate.

Validate the input before it reaches `dotnet pack`:

```bash
VERSION="$1"
if [[ ! "$VERSION" =~ ^(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)(-[0-9A-Za-z.-]+)?$ ]]; then
  echo "not a SemVer 2.0 version: $VERSION" >&2
  exit 1
fi
```

## 4. Semantic versions and prerelease suffixes

- Stable: `1.4.0`. Prerelease: `1.4.0-preview.1`, `1.4.0-rc.2`. Build metadata (`+sha`) is
  accepted by SemVer but is not useful on a NuGet feed, so do not use it for releases.
- A prerelease sorts before its stable version, so `1.4.0-preview.1` < `1.4.0`.
- Flag GitHub releases with a prerelease suffix as prereleases: `gh release create --prerelease`.
  Only stable versions should appear as "Latest".
- Start a new minor or major line with a fresh prerelease before the stable release if the
  change is large, so consumers can opt in.

## 5. Mapping versions to tags and releases

- Tag: `vX.Y.Z`. The `v` is the convention used by MinVer's default prefix and by the gate.
- Release title: the tag name, or `Name X.Y.Z`. Notes come from the changelog section.
- Never move a tag. If a published tag is wrong, release a new patch version and say so in the
  changelog.

## 6. Pitfalls

- **Silent skip.** `--skip-duplicate` returns success for a version already on the feed. A
  rebuilt-but-same-number package is therefore not published, and the old bytes stay. Bump the
  version.
- **Prerelease mismatch.** A NuGet version with a suffix and a GitHub tag without one disagree.
  The gate should be given the same string the pack step uses.
- **Shallow clone.** A shallow checkout makes MinVer report `0.0.0`, which then pushes a version
  nobody can consume. Check the first build line of the log.
- **Leading zeros** in a numeric identifier, such as `1.04.0`, are not valid SemVer and are
  rejected by the validator above.
