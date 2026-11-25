# Debugging a failed release and running without Actions minutes

Contents

1. Find the failing step
2. Re-run without re-running everything
3. Symptom to cause table
4. Dry-run locally with act
5. When Actions minutes run out
6. Recovering from a partial release

## 1. Find the failing step

```bash
gh run list --workflow release.yml --limit 5
gh run view RUN_ID --json jobs \
  --jq '.jobs[] | {job: .name, conclusion, failed: [.steps[] | select(.conclusion == "failure") | .name]}'
gh run view RUN_ID --log-failed
```

`--log-failed` prints only the steps that failed. If the output is empty, the failure was in a
job that never started, such as a skipped `needs` chain. Check the job list above it.

For a single job's full log, use `gh run view RUN_ID --job JOB_ID --log`. Logs are large. Save the
output to a file and search that file afterwards, so that a filter on the stream cannot hide the
line that matters.

## 2. Re-run without re-running everything

```bash
gh run rerun RUN_ID --failed
```

This re-runs only the failed jobs, with the original inputs. Confirm the version input is the one
you meant before re-running: a re-run keeps the old `version`.

## 3. Symptom to cause table

| Symptom | Likely cause | Fix |
|---|---|---|
| Gate fails: no section for the version | Notes not moved from `Unreleased` | `references/changelog-gate.md` section 2, then a new commit on the release branch |
| Gate fails: section has no notes | Heading present, body empty | Write the notes; do not add placeholder text just to pass the gate |
| Gate fails: duplicate sections | Two headings for the same version | Remove the stale one; keep the notes that belong to the release |
| Push fails with 401 or 403 | `packages: write` missing, or `GITHUB_PACKAGES_TOKEN` empty, or the source name in `nuget.config` does not match | Check the job's `permissions` and `env`, then the `packageSourceCredentials` element name |
| Push reports nothing new, or logs `Skipped` | The version already exists on the feed (`--skip-duplicate`) | Bump the version. Do not republish different bytes under it |
| Every build reports `0.0.0` | Shallow checkout, or no tag in the clone | `fetch-depth: 0` on the checkout |
| `gh release create` says the release or tag exists | An earlier run got past this step | See section 6. Do not overwrite |
| `Resource not accessible by integration` from `gh` | `contents: write` missing on the `release` job | Add it to that job only |
| Binary builds but exits at once on the runner | Missing runtime file, or a trimming problem | Run `--version` on the same RID in the build job; see `cross-platform-builds.md` |
| Restore works locally but fails in Actions | Consumer cannot read the package | `consumer-access.md` section 6 |
| Restore works only with a warm cache | Stale `obj/` or NuGet cache | `dotnet nuget locals all --clear` and `dotnet restore --force` |

## 4. Dry-run locally with act

`act` runs workflow jobs in Docker containers on the local machine. Confirm the flags on the
installed version first, since they change between releases:

```bash
act --help | grep -E 'input|secret-file|dryrun|-j'
```

Validate the workflow file without starting containers:

```bash
act -n workflow_dispatch -W .github/workflows/release.yml --input version=1.4.0
```

Run the gate job for real, which needs no secrets:

```bash
act workflow_dispatch -W .github/workflows/release.yml -j gate --input version=1.4.0
```

Limits to keep in mind:

- Only Linux jobs run under `act` with the default images. The `win-x64` and `osx-arm64` builds
  cannot run locally this way.
- `act` does not have the real `GITHUB_TOKEN` permissions. Do not run the `release` or the
  `publish-nuget` push locally with act and expect it to represent GitHub.
- Secrets go in a file given by `--secret-file`, mode 600, outside the repository, and never on
  the command line.

## 5. When Actions minutes run out

Minutes are a billing limit, not a code fault. Three fallbacks, in the order to try them:

**A. A self-hosted runner for the Linux build and publish jobs.** Register a runner on a machine
you control, with labels, then point the jobs at it:

```yaml
runs-on: [self-hosted, linux, x64]
```

Rules for a runner on a public repository:

- Only trigger the release workflow with `workflow_dispatch` or a tag, by a trusted person. Never
  let pull request workflows reach a self-hosted runner, because fork code would run on your machine.
- Use an ephemeral runner, or wipe the work folder between runs.
- Keep secrets for the run in the environment, not on disk.

Windows and macOS builds need a matching machine of their own. Do not substitute a Linux runner.

**B. A local release script.** Run the same steps from a clean checkout of the release commit. Refuse
to run if the tree is dirty or HEAD is not the pushed `main`:

```bash
set -Eeuo pipefail
VERSION="${1:?usage: release-local.sh VERSION}"
git fetch origin main --tags
[[ -z "$(git status --porcelain)" ]] || { echo "tree is dirty" >&2; exit 1; }
[[ "$(git rev-parse HEAD)" == "$(git rev-parse origin/main)" ]] || { echo "HEAD is not origin/main" >&2; exit 1; }
mkdir -p notes
bash scripts/changelog-gate.sh "$VERSION" CHANGELOG.md notes/release-notes.md
# The push needs write:packages on the gh token: gh auth refresh --scopes write:packages
export GITHUB_PACKAGES_USER="$(gh api user --jq .login)"
export GITHUB_PACKAGES_TOKEN="$(gh auth token)"
# build and pack as in release-workflow.md, then:
dotnet nuget push "artifacts/nupkg/*.nupkg" --api-key "$GITHUB_PACKAGES_TOKEN" \
  --source "https://nuget.pkg.github.com/OWNER/index.json" --skip-duplicate
gh release create "v$VERSION" --target "$(git rev-parse HEAD)" --title "my-tool $VERSION" \
  --notes-file notes/release-notes.md dist/my-tool-"$VERSION"-*
```

The local script builds only what the local machine can build. Say so in the release notes, and
do not attach a platform the script did not build.

**C. Shrink the matrix.** Build only `linux-x64` in CI and attach the other binaries from a
machine that has the platform. This is the cheapest change to a working pipeline.

Whichever fallback is used, the gate still runs first, and the version still passes through the
same tag and changelog rules.

## 6. Recovering from a partial release

A run can fail after the tag or the NuGet push. Work out which of these happened before doing
anything:

```bash
gh release view "vX.Y.Z" --json tagName,isDraft,isPrerelease,assets
git ls-remote --tags origin "vX.Y.Z"
gh api "/orgs/OWNER/packages/nuget/PKG/versions" --jq '.[].name'
```

- **Tag and release exist, assets missing.** Upload the missing assets with
  `gh release upload vX.Y.Z FILES --clobber` (safe if the bytes are the same build), then check.
- **Notes wrong.** `gh release edit vX.Y.Z --notes-file FILE`.
- **NuGet version published, release not made.** Re-run only the `release` job with
  `gh run rerun RUN_ID --failed`. The push skips the duplicate and the release step runs.
- **Nothing published except the changelog commit.** Fix the cause and re-run the whole workflow.
- **The version is wrong or the release must be withdrawn.** Deleting a tag, a release or a
  package version is destructive and visible to consumers. Do it only with the repository owner's
  explicit decision. The usual answer is a new patch version with a changelog note.
