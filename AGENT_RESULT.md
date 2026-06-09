# Agent Result: helm/helm#31439

## Root Cause

When `helm package` creates a chart archive, the `writeToTar` function in both
`pkg/chart/v2/util/save.go` and `internal/chart/v3/util/save.go` used
`time.Now()` as the fallback modification time when a file had a zero `ModTime`.
This meant charts assembled without explicit file timestamps (e.g., from
programmatic use of the API) would get a different timestamp on every build,
making the resulting `.tgz` non-reproducible.

The `SOURCE_DATE_EPOCH` environment variable is the standard mechanism in the
reproducible-builds ecosystem for specifying a fixed Unix timestamp to use
during a build. Helm had no support for it.

## Change Made

Added `sourceDateEpoch() time.Time` helper function to both:

- `helm.sh/helm/v4/pkg/chart/v2/util.writeToTar` (via `sourceDateEpoch`)
- `helm.sh/helm/v4/internal/chart/v3/util.writeToTar` (via `sourceDateEpoch`)

When `h.ModTime.IsZero()`, the code now calls `sourceDateEpoch()` instead of
`time.Now()` directly. `sourceDateEpoch()` reads the `SOURCE_DATE_EPOCH`
environment variable, parses it as a Unix integer timestamp, and returns that
time if valid. If `SOURCE_DATE_EPOCH` is not set or is not a valid integer, it
falls back to `time.Now()`.

## Testing

Added `TestSourceDateEpoch` to both test files:

- `pkg/chart/v2/util/save_test.go`
- `internal/chart/v3/util/save_test.go`

Each test sets `SOURCE_DATE_EPOCH=1630000000` via `t.Setenv`, creates a chart
with zero-value `ModTime` fields, saves it, and verifies every tar header has
`ModTime` equal to `time.Unix(1630000000, 0).UTC()`.

All relevant tests pass:
- `TestSourceDateEpoch` - PASS
- `TestRepeatableSave` - PASS
- `TestSavePreservesTimestamps` - PASS

## Lint

The linter could not run in this environment due to a Go version mismatch
(the available `golangci-lint` binary was built with Go 1.25, while the module
requires Go 1.26). The only change to both files was adding `"strconv"` to the
imports (in alphabetical order) and adding the `sourceDateEpoch()` function.
The code follows existing style - no new linting issues were introduced.
