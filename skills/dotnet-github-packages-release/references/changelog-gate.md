# Changelog gate and release notes

Contents

1. The file layout (Keep a Changelog)
2. Releasing the Unreleased notes
3. The gate script
4. Testing the gate before trusting it
5. Using the gate in the workflow and for notes

## 1. The file layout (Keep a Changelog)

`CHANGELOG.md` at the repository root follows [Keep a Changelog](https://keepachangelog.com).
Every release has its own level-two heading in square brackets, with the date after a spaced
hyphen. The newest release sits at the top, below an empty `Unreleased` section.

```markdown
# Changelog

## [Unreleased]

## [1.4.0] - 2026-11-19

### Added
- Export to CSV.

### Fixed
- Empty input no longer crashes the parser.

## [1.3.2] - 2026-10-02

### Fixed
- Retry on a 503 from the upstream service.

[1.4.0]: https://github.com/OWNER/REPO/releases/tag/v1.4.0
[1.3.2]: https://github.com/OWNER/REPO/releases/tag/v1.3.2
```

Rules the gate relies on:

- A release heading is `## [X.Y.Z]` or `## [X.Y.Z] - YYYY-MM-DD`. The version is inside the
  brackets, and it matches the tag without the `v`.
- The notes are the lines under the heading, up to the next `## ` heading or the link references.
- A section with only blank lines counts as missing. A release with no notes is not a release.

## 2. Releasing the Unreleased notes

Move the notes from `Unreleased` to the new version. Keep the empty `Unreleased` heading above them.
Because the notes follow the heading they sit under, the move is a single new heading:

```bash
VERSION=X.Y.Z
DATE=$(date +%F)
tmp=$(mktemp "${TMPDIR:-/tmp}/changelog.XXXXXX")
awk -v v="$VERSION" -v d="$DATE" '
  !done && $0 == "## [Unreleased]" { print; print ""; print "## [" v "] - " d; done = 1; next }
  { print }
' CHANGELOG.md > "$tmp" && mv "$tmp" CHANGELOG.md
git diff -- CHANGELOG.md
```

Then add the link reference at the bottom by hand, following the existing pattern:

```markdown
[X.Y.Z]: https://github.com/OWNER/REPO/releases/tag/vX.Y.Z
```

Read the diff. The only change should be one new heading and one new link line.

## 3. The gate script

Save this outside the repository, for example as `$(mktemp -d)/changelog-gate.sh`, or keep
a copy in the repository's `scripts/` folder if the team wants it versioned. Set `GATE` to the
path.

```bash
#!/usr/bin/env bash
# Refuse a release unless CHANGELOG.md has exactly one non-empty section for VERSION.
# Usage: changelog-gate.sh VERSION CHANGELOG [NOTES_FILE]
# Exit: 0 section present and non-empty; 1 refused (missing, duplicate or empty); 2 usage error.
set -Eeuo pipefail

version=${1:-}
file=${2:-}
notes_file=${3:-}

if [[ -z "$version" || -z "$file" ]]; then
  echo "usage: changelog-gate.sh VERSION CHANGELOG [NOTES_FILE]" >&2
  exit 2
fi
if [[ ! "$version" =~ ^[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?$ ]]; then
  echo "gate: not a version: $version" >&2
  exit 2
fi
if [[ ! -f "$file" ]]; then
  echo "gate: no such file: $file" >&2
  exit 2
fi

# A heading matches only the exact version: "## [1.4.0]" must not match "## [1.4.0-preview.1]".
# index() compares literal text, so the dots in the version are not regular expressions.
heading="## [$version]"
count=$(awk -v h="$heading" '
  index($0, h) == 1 && (length($0) == length(h) || substr($0, length(h) + 1, 3) == " - ") { n++ }
  END { print n + 0 }
' "$file")

if ((count == 0)); then
  echo "gate: refused: no section [$version] in $file" >&2
  exit 1
fi
if ((count > 1)); then
  echo "gate: refused: $count sections for [$version] in $file" >&2
  exit 1
fi

body=$(awk -v h="$heading" '
  index($0, h) == 1 && (length($0) == length(h) || substr($0, length(h) + 1, 3) == " - ") { on = 1; next }
  on && /^## / { exit }
  on && /^\[[^]]+\]: / { exit }
  on { print }
' "$file")
body=$(printf '%s\n' "$body" | sed '/./,$!d')

if [[ ! "$body" =~ [^[:space:]] ]]; then
  echo "gate: refused: section [$version] in $file has no notes" >&2
  exit 1
fi

if [[ -n "$notes_file" ]]; then
  printf '%s\n' "$body" > "$notes_file"
fi
echo "gate: ok: [$version] has $(printf '%s\n' "$body" | wc -l) lines of notes"
```

Why it is written this way:

- `set -Eeuo pipefail` makes a failed `awk` stop the script rather than report a clean section.
- The count runs before extraction, so a duplicate heading fails loudly instead of being
  read twice.
- The emptiness test is a bash regex on the captured body, not `grep -q` in a pipeline, so no
  `SIGPIPE` status can turn a clean result into a failure.
- The version check runs first, so an empty argument cannot match every heading.

## 4. Testing the gate before trusting it

Write the test cases first, run them against no script or a stub to see them fail, then write
the gate. Run the test in a throwaway directory outside the repository, so the fixtures cannot
touch real files.

```bash
fixtures=$(mktemp -d)
cat > "$fixtures/ok.md" <<'EOF'
# Changelog

## [Unreleased]

## [1.4.0] - 2026-11-19

### Added
- Export to CSV.
EOF
printf '# Changelog\n\n## [Unreleased]\n\n## [1.4.0] - 2026-11-19\n\n## [1.3.0] - 2026-01-01\n- Old.\n' > "$fixtures/empty.md"
printf '# Changelog\n\n## [Unreleased]\n\n## [1.4.0] - 2026-11-19\n- A.\n\n## [1.4.0] - 2026-11-20\n- B.\n' > "$fixtures/dupe.md"
printf '# Changelog\n\n## [Unreleased]\n\n## [1.4.0-preview.1] - 2026-11-19\n- Pre.\n' > "$fixtures/pre.md"
printf '# Changelog\n\n## [Unreleased]\n- Not yet released.\n' > "$fixtures/none.md"

check() { # check EXPECTED VERSION FILE
  local got=0
  bash "$GATE" "$2" "$3" > /dev/null 2>&1 || got=$?
  if [[ "$got" == "$1" ]]; then echo "pass: $2 $(basename "$3") -> $got"; else echo "FAIL: $2 $(basename "$3") expected $1 got $got"; fi
}
check 0 1.4.0 "$fixtures/ok.md"
check 1 1.4.0 "$fixtures/empty.md"
check 1 1.4.0 "$fixtures/dupe.md"
check 1 1.4.0 "$fixtures/pre.md"
check 1 1.4.0 "$fixtures/none.md"
check 0 1.4.0-preview.1 "$fixtures/pre.md"
check 2 "" "$fixtures/ok.md"
check 2 1.4.0 "$fixtures/missing.md"
```

Read every line of output. The `empty.md` case is the one that matters most, because a bare
heading looks like a release to a human scanning the file. The `pre.md` case with `1.4.0` must
refuse: a prefix match would release a version with no notes of its own.

Each case should fail for its stated reason. Check the message as well as the code by running a
case without `> /dev/null` once.

## 5. Using the gate in the workflow and for notes

Run the gate and extract the notes in one call, then pass the notes file to the release step:

```bash
bash "$GATE" "$VERSION" CHANGELOG.md "$RUNNER_TEMP/release-notes.md"
```

`RUNNER_TEMP` is set on GitHub-hosted and self-hosted runners. For a local run, use `mktemp`.
Pass the notes to `gh release create --notes-file`, never as `--notes "$(...)"`, so that
markdown with quotes or backticks reaches GitHub intact.
