# Cross-platform self-contained builds

Contents

1. Runtime identifiers and runners
2. The publish command and what each flag does
3. Smoke test on the runner
4. Archives and checksums
5. Native AOT and trimming
6. Cost and runner choice

## 1. Runtime identifiers and runners

| Runtime identifier (RID) | Build on | Archive | Notes |
|---|---|---|---|
| `linux-x64` | `ubuntu-latest` or an Ubuntu LTS image | `.tar.gz` | Runs on most glibc distributions. Alpine needs `linux-musl-x64`, a separate build. |
| `win-x64` | `windows-latest` | `.zip` | Built on Windows so that the zip carries `.exe` and the Windows-specific host files. |
| `osx-arm64` | `macos-latest` (Apple silicon at the time of writing) or a pinned `macos-14`+ image | `.tar.gz` | Build on macOS so the native host matches. Check the runner label's documentation before relying on the image name. |

Add `linux-arm64` or `win-arm64` only when a consumer asks for them. Each new RID adds a matrix row,
runner minutes and a support obligation.

## 2. The publish command and what each flag does

```bash
dotnet publish src/my-tool/my-tool.csproj -c Release -r linux-x64 \
  --self-contained true -p:PublishSingleFile=true -p:Version="$VERSION" \
  -o "dist/linux-x64"
```

- `-r <rid>` selects the target platform. Without it the output is platform-neutral.
- `--self-contained true` bundles the runtime, so the consumer needs no .NET install. Pass it
  explicitly so the intent does not depend on SDK defaults.
- `PublishSingleFile=true` produces one executable. Native libraries that the runtime must load
  from disk need `IncludeNativeLibrariesForSelfExtract=true` as well.
- `-p:Version` stamps the assembly version. It does not change the NuGet package version.
- `-o` gives each RID its own folder, so archives do not mix platforms.

Useful additions in the project file, set only if tested:

```xml
<PropertyGroup>
  <InvariantGlobalization>true</InvariantGlobalization>
  <EnableCompressionInSingleFile>true</EnableCompressionInSingleFile>
  <SelfContained>true</SelfContained>
</PropertyGroup>
```

`InvariantGlobalization` changes culture behaviour. Turn it on only if the application does not
need locale-specific formatting or collation.

## 3. Smoke test on the runner

A binary that builds but cannot start is the most common silent failure. Run it inside the build
job, after publish and before archiving:

```bash
# Unix runners
"dist/$RID/my-tool" --version
```

```pwsh
# Windows runner
& "dist/$env:RID/my-tool.exe" --version
```

A non-zero exit fails the step. Use a command that exits quickly and does not touch the network
or the user's files.

## 4. Archives and checksums

Archive each RID separately, with the executable at the top level of the archive:

```bash
tar -C "dist/$RID" -czf "dist/my-tool-$VERSION-$RID.tar.gz" .
```

Create a checksum file in the release job, after all archives are downloaded:

```bash
cd release
sha256sum my-tool-"$VERSION"-* > SHA256SUMS
cat SHA256SUMS
```

Attach `SHA256SUMS` as a release asset. Verification on the consumer side is
`sha256sum --check SHA256SUMS`.

## 5. Native AOT and trimming

- `PublishAot=true` with the same RID switches to a native compile. It needs the platform's
  native toolchain on the runner, so it follows the same one-runner-per-platform rule.
- Trimming (`PublishTrimmed=true`) removes unused code. It is safe only where the code avoids
  reflection-heavy patterns. Run the smoke test and the application's own tests on the trimmed
  output before shipping it.
- A trimmed or AOT build that passes the smoke test can still fail on a code path the smoke test
  does not reach. State in the release notes which path was exercised, if the change touches it.

## 6. Cost and runner choice

- Linux runners are the cheapest. Windows and macOS minutes are billed at a multiplier against
  the Linux rate for private repositories, so check the current multipliers on the billing page
  before adding a matrix row.
- Building macOS on Linux is not a substitute: the native host and signing steps need macOS. The
  workflow builds each RID on its own runner for this reason.
- Code signing and notarisation are not covered here. A macOS build that is not notarised will
  trigger Gatekeeper warnings for consumers. Add those steps as a separate, reviewed change.
