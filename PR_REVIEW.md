# Review of PR #339: Add --color flag and visualizer coloring support

## Summary

This PR adds a `--color` flag (`auto`/`always`/`never`) to `pdu`, enabling colorized
output of the tree visualization using `LS_COLORS` environment variable via the `lscolors`
crate. Files, directories, executables, and symlinks each get appropriate ANSI color codes.

The implementation spans 17 commits of iterative refinement and touches ~600 lines across
26 files.

## Architecture

The design is clean and well-layered:

1. **`src/args/color.rs`** — `ColorWhen` enum (clap `ValueEnum`)
2. **`src/ls_colors.rs`** — `LsColors` wrapper that parses `LS_COLORS` env var into
   per-type ANSI prefix strings
3. **`src/visualizer/coloring.rs`** — `Coloring` struct (path-to-color map + `LsColors`),
   `Color` enum, `ColoredTreeHorizontalSlice` display impl
4. **`src/app/sub.rs`** — `build_coloring_map` walks the `DataTree` to populate the
   path-to-color map
5. **`src/visualizer/methods.rs`** — rendering logic branches on coloring presence

## Issues

### 1. `build_coloring_map` does filesystem I/O on already-scanned paths (Medium severity)

**Location**: `src/app/sub.rs:199-209`

`file_color()` calls `path.is_symlink()`, `path.is_dir()`, and `path.metadata()` for every
leaf node. This is redundant syscall overhead since the data tree was already built by walking
the filesystem. Worse, these paths are relative (rooted at `"."`) and the results depend on the
current working directory, which could produce incorrect results if the CWD changes or the paths
are passed as absolute.

**Suggestion**: Carry the file type information through the `DataTree` during the initial scan,
or at minimum, document that this function requires the CWD to match the scan root.

### 2. `HashMap<PathBuf, Color>` is unnecessarily expensive (Low-Medium severity)

**Location**: `src/app/sub.rs:157-161`

Building a `HashMap<PathBuf, Color>` that reconstructs full paths by joining path components is
O(n) allocations. Then during rendering (`methods.rs:96-105`), paths are reconstructed *again*
by collecting ancestor names into a `PathBuf` for every row.

**Suggestion**: Consider using the tree node's identity (e.g., a node index or pointer) as the
map key instead of reconstructing paths twice.

### 3. `Name: Display + AsRef<OsStr>` trait bound tightening (Low severity)

**Location**: `src/visualizer/display.rs:9` and `src/visualizer/methods.rs:28`

The `Visualizer` impl blocks now require `Name: AsRef<OsStr>` in addition to `Display`. This is
fine for the current use case (`OsStringDisplay`), but it's a breaking API change for any
downstream code that uses `Visualizer` with a custom `Name` type. The `Visualizer` struct
definition itself doesn't have this bound — it only appears on the impl blocks, which creates an
inconsistency.

### 4. Missing ANSI reset for column alignment padding (Low severity)

**Location**: `src/visualizer/coloring.rs:88`

The color prefix is applied to the name, and the reset is placed immediately after:
`{skeletal_component}{prefix}{name}{suffix}`. However, `ColoredTreeHorizontalSlice` implements
`Width` by delegating to the inner slice's `Width`, which doesn't account for the ANSI bytes.
When `align_left` pads the name with spaces to fill `tree_width`, those padding spaces appear
*after* the reset, which is correct. But if a terminal has background color in the style, the
padding won't be colored — this is likely fine for typical `LS_COLORS` configs but worth noting.

### 5. `Hash` derive added to `OsStringDisplay` (Low severity)

**Location**: `src/os_string_display.rs:21`

Adding `Hash` is needed for the `HashMap<PathBuf, Color>` approach, but `OsStringDisplay` isn't
used as a key anywhere. The `Hash` derive is on the type but the map keys are `PathBuf`, so this
derive may actually be unnecessary. Verify whether this is still needed after the refactoring
iterations.

### 6. Test `color_always` hardcodes file-type expectations (Low severity)

**Location**: `tests/usual_cli.rs:957-967`

The `color_always` test manually builds a `HashMap` of expected path-to-color mappings. This is
brittle — if `build_coloring_map`'s behavior changes, the test's manually-constructed map won't
catch regressions in the actual mapping logic. Consider testing through the CLI output instead
(which `colorful_equals_colorless` already does well).

### 7. `simple_tree_with_diverse_kinds` is unix-only but not `cfg`-gated (Low severity)

**Location**: `tests/_utils.rs:205`

Uses `std::os::unix::fs::symlink` but only the calling test (`color_always`) is `#[cfg(unix)]`.
If another test on a non-unix platform called this helper, it would fail to compile.

## Positives

- Clean separation of concerns between `LsColors`, `Coloring`, `Color`, and
  `ColoredTreeHorizontalSlice`
- The `--color=auto` TTY detection is correctly placed in `app.rs` and uses
  `stdout().is_terminal()`
- Good test coverage: stripped-equals-plain test, with-and-without-LS_COLORS test, and a full
  expected-output test
- The `AnsiPrefix::suffix()` returning `""` when the prefix is empty is a nice touch — avoids
  spurious resets
- `lscolors` dependency with `nu-ansi-term` feature is a solid choice for LS_COLORS parsing
- The `from_ls_colors_string` constructor enables deterministic testing without `set_var`

## Verdict

The PR is in good shape overall. The most impactful issue is #1 (redundant filesystem I/O in
`build_coloring_map`), which adds unnecessary syscalls and has a correctness risk with relative
paths. Issue #2 (double path reconstruction) is a performance concern but unlikely to matter in
practice. The rest are minor.

Recommendation: Address #1 before merging; the others can be follow-ups.
