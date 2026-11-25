# The release workflow

Contents

1. Shape of the workflow
2. The complete workflow file
3. Why each job is ordered as it is
4. Replacing a published release's notes
5. Things that must not change

## 1. Shape of the workflow

One workflow, started by hand with a version, in four jobs:

```text
gate ──► build (linux-x64, win-x64, osx-arm64)  ──► release (tag, GitHub release, assets)
  └────► publish-nuget (pack, push *.nupkg)      ──┘
```

- `gate` refuses the run unless the changelog section exists and has notes. Nothing else runs
  if it fails.
- `build` makes one self-contained single-file binary per platform, on a runner of that platform.
- `publish-nuget` packs and pushes the packages to the GitHub Packages feed.
- `release` runs only when the other three succeed. It creates the tag at the commit being built,
  then the GitHub release with the notes and the binaries.

## 2. The complete workflow file

Save as `.github/workflows/release.yml`. The examples use tags such as `@v4` for readability.
Replace every `@<full-sha> # v4` with the full commit SHA of the same release that your
organisation has reviewed, as the reference workflows do.

The workflow expects `scripts/changelog-gate.sh` in the repository (see `changelog-gate.md`).
It expects `global.json` at the root and the application at `src/my-tool/my-tool.csproj`.
Rename `my-tool` and `PKG` to match.

```yaml
name: release

on:
  workflow_dispatch:
    inputs:
      version:
        description: SemVer version without the v, e.g. 1.4.0 or 1.4.0-preview.1
        required: true
        type: string

permissions:
  contents: read

jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<full-sha> # v4
      - name: Refuse unless CHANGELOG.md has notes for this version
        env:
          VERSION: ${{ inputs.version }}
        run: |
          set -Eeuo pipefail
          mkdir -p notes
          bash scripts/changelog-gate.sh "$VERSION" CHANGELOG.md notes/release-notes.md
      - uses: actions/upload-artifact@<full-sha> # v4
        with:
          name: release-notes
          path: notes/release-notes.md

  build:
    needs: gate
    strategy:
      fail-fast: true
      matrix:
        include:
          - rid: linux-x64
            os: ubuntu-latest
          - rid: win-x64
            os: windows-latest
          - rid: osx-arm64
            os: macos-latest
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@<full-sha> # v4
        with:
          fetch-depth: 0
      - uses: actions/setup-dotnet@<full-sha> # v4
        with:
          global-json-file: global.json
      - name: Publish self-contained single file
        shell: bash
        env:
          VERSION: ${{ inputs.version }}
          RID: ${{ matrix.rid }}
        run: |
          set -Eeuo pipefail
          dotnet publish src/my-tool/my-tool.csproj -c Release -r "$RID" \
            --self-contained true -p:PublishSingleFile=true -p:Version="$VERSION" \
            -o "dist/$RID"
      - name: Archive (Unix)
        if: runner.os != 'Windows'
        shell: bash
        env:
          VERSION: ${{ inputs.version }}
          RID: ${{ matrix.rid }}
        run: |
          set -Eeuo pipefail
          tar -C "dist/$RID" -czf "dist/my-tool-$VERSION-$RID.tar.gz" .
      - name: Archive (Windows)
        if: runner.os == 'Windows'
        shell: pwsh
        env:
          VERSION: ${{ inputs.version }}
          RID: ${{ matrix.rid }}
        run: |
          $ErrorActionPreference = 'Stop'
          Compress-Archive -Path "dist/$env:RID/*" -DestinationPath "dist/my-tool-$env:VERSION-$env:RID.zip"
      - uses: actions/upload-artifact@<full-sha> # v4
        with:
          name: binaries-${{ matrix.rid }}
          path: dist/my-tool-*
          if-no-files-found: error

  publish-nuget:
    needs: gate
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    env:
      GITHUB_PACKAGES_USER: ${{ github.actor }}
      GITHUB_PACKAGES_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@<full-sha> # v4
        with:
          fetch-depth: 0
      - uses: actions/setup-dotnet@<full-sha> # v4
        with:
          global-json-file: global.json
      - name: Pack
        env:
          VERSION: ${{ inputs.version }}
        run: |
          set -Eeuo pipefail
          rm -rf artifacts/nupkg
          dotnet pack src/PKG/PKG.csproj -c Release -p:PackageVersion="$VERSION" -o artifacts/nupkg
      - name: Push (*.nupkg only; GitHub Packages rejects .snupkg)
        run: |
          set -Eeuo pipefail
          dotnet nuget push "artifacts/nupkg/*.nupkg" \
            --api-key "$GITHUB_PACKAGES_TOKEN" \
            --source "https://nuget.pkg.github.com/OWNER/index.json" \
            --skip-duplicate

  release:
    needs: [gate, build, publish-nuget]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    env:
      GH_TOKEN: ${{ github.token }}
      VERSION: ${{ inputs.version }}
    steps:
      - uses: actions/download-artifact@<full-sha> # v4
        with:
          pattern: "*"
          path: release
          merge-multiple: true
      - name: Create tag and GitHub release
        run: |
          set -Eeuo pipefail
          flags=()
          if [[ "$VERSION" == *-* ]]; then flags+=(--prerelease); fi
          gh release create "v$VERSION" \
            --target "$GITHUB_SHA" \
            --title "my-tool $VERSION" \
            --notes-file release/release-notes.md \
            "${flags[@]}" \
            release/my-tool-*
```

The `release` job has no checkout. It needs none: the notes and binaries arrive as artifacts,
and `--target "$GITHUB_SHA"` names the commit for the tag.

## 3. Why each job is ordered as it is

- **The gate is first and the only input to the rest.** A failing gate prevents any build,
  push or tag. This is the enforcement point for the changelog rule, so it must not be
  skippable by reordering jobs.
- **Build before release, push before release.** If a binary fails or the NuGet push fails,
  there is no tag and no release to clean up. A re-run starts clean.
- **`publish-nuget` is separate from `build`** so the package push does not depend on runner
  minutes for the Windows and macOS builds.
- **`merge-multiple: true` with `pattern: "*"`** collects the notes file and every binary into
  one folder. The notes file is named exactly `release-notes.md`, which the release step reads.
- **`gh release create` fails if the tag exists.** That protects against overwriting a release.
  A tag that already exists means the version was used; bump it.
- **Prerelease flag from the version string.** Any version with a hyphen is marked prerelease, so
  it does not become "Latest".

## 4. Replacing a published release's notes

If the changelog section was corrected after the release was made, rewrite only the notes. Do not
re-run the whole workflow, which would fail on the existing tag:

```bash
gh release edit "vX.Y.Z" --notes-file notes/release-notes.md
```

Regenerate the notes file first with `changelog-gate.sh` so that the text matches the changelog.
Do not change the assets or the title in the same step.

## 5. Things that must not change

- Permissions: `contents: write` is needed only in `release`, and `packages: write` only in
  `publish-nuget`. Do not widen them at workflow level.
- `fetch-depth: 0` wherever MinVer runs (`build` and `publish-nuget`). Tags are not fetched otherwise.
- `--skip-duplicate` is a silent skip, not an overwrite. A re-push of the same version is a no-op.
- The `gate` job's `needs` on every other job. Removing it removes the changelog rule.
