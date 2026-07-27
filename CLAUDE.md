# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This working directory (`winflexbison_migration/`) contains the project plus supporting material:

- `winflexbison/` — the actual project: a Windows port of Flex and Bison
  (`lexxmark/winflexbison` on GitHub). This is where almost all work happens.
- `upstream/` — pristine upstream **baseline mirrors**, each checked out at the exact tag matching the
  currently vendored version, used for diffing when porting/upgrading: `upstream/flex` (@ `v2.6.4`),
  `upstream/bison` (@ `v3.8.2`, from `akimd/bison`, carries the test suite), `upstream/m4` (@ `v1.4.19`),
  `upstream/gnulib` (at the commit pinned by bison's/m4's submodule). Not built, not committed to the
  project. See `docs/specs/02-baseline-mirrors/spec.md`.
- `docs/specs/` — the **upgrade playbook**: a repeatable process for adopting new upstream
  flex/bison/m4 releases (version inventory, baselines, test adoption, the Windows port-change
  catalog, upgrade runbook, validation gate). Start at `docs/specs/README.md`.

Unless told otherwise, assume any task refers to `winflexbison/`. When diffing against
upstream, use `upstream/<component>` as the baseline — not a fresh `master` checkout, which is newer
than the vendored version.

## Build

Requires Visual Studio 2017+ and CMake. Only MSVC (and clang-cl) toolchains are officially
supported (the root `CMakeLists.txt` warns if `MSVC` is not set).

Quick build via provided batch scripts (from `winflexbison/`), each configures with
the matching VS generator, builds Release, and produces a `cpack` zip package:

```
buildVS2017.bat
buildVS2019.bat
buildVS2022.bat
```

Equivalent manual invocation:

```
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
cmake --build build --target package   # produces build/win_flex_bison*.zip
```

CMake presets (`CMakePresets.json`) are also available for Ninja-based builds:
`x64-Release`, `x64-RelWithDebInfo`, `x64-Debug` (binaries land under `CMakeBuild/`).

Useful CMake options:
- `USE_STATIC_RUNTIME=ON` — link the static CRT (`/MT`) instead of the dynamic CRT (`/MD`); used
  by CI.
- `CPACK_PACKAGE_VERSION` — overrides the package version string embedded in the zip name.

Built executables always land in `bin/Release/win_flex.exe` and `bin/Release/win_bison.exe`
(hardcoded in the root `CMakeLists.txt` via `CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE`), regardless
of which build tree (`CMakeBuildVS2022/`, `CMakeBuild/build/<preset>/`, etc.) produced them; only
intermediate build files and the packaged zip live under the build tree itself.

`bin/Release/` must stay free of anything that isn't shipped: CPack does
`install(DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE}/" DESTINATION "./")`, so every file
sitting there goes into the release zip — including artifacts left over from earlier builds.
Test executables are therefore redirected to `<build tree>/tests/bin/` by `tests/CMakeLists.txt`;
any new target that isn't part of the product needs the same treatment.

CI mirrors these paths: AppVeyor (`.appveyor.yml`) builds VS2022 (x64) and VS2026 (x64+Win32)
with MSVC, and GitHub Actions (`.github/workflows/os_windows.yaml`) builds with `clang-cl` via
Ninja. AppVeyor additionally runs the CTest gate, but **only in the VS2022/x64/Release cell** —
every other job echoes a `[ctest] skipped` line — and packages in `after_test` so a failing test
produces no zip. GitHub Actions only configures/builds/packages.

Win32 coverage comes solely from the VS2026 rows; adding a Win32 exclusion for VS2026 would
silently stop 32-bit builds and artifacts.

## Testing

The project has a CTest suite (wired via `enable_testing()` + `add_subdirectory(tests)` for
top-level builds) plus an opt-in WSL-driven bison autotest. See
`docs/specs/03-test-adoption/spec.md` for the full design.

- **Windows CTest gate — `runtests.bat`** (configure + build Release + `ctest`), **no WSL**:
  - `tests/flex/` — the flex v2.6.4 suite adapted to CTest (~112 exit-code/comparison tests;
    scanners generated with `win_flex --wincompat`, compiled with MSVC, run).
  - `tests/bison/` — a self-contained compile-run parser plus golden-diff diagnostics (win_bison vs
    golden captured from the reference bison; regenerate with `tests/bison/generate.sh` under WSL).
  - `tests/winflexbison/` — **our own** port-specific tests (e.g. parallel-invocation resistance,
    verifying per-process temp files don't collide and don't leak).
- **Full bison GNU Autotest — `tests/bison-autotest/`** (696 groups), a POSIX-shell harness run
  under **WSL** against `win_bison.exe`: `runtests.bat --with-autotest`, or as a ctest test with
  `cmake -DWFB_WSL_AUTOTEST=ON`. Install its WSL deps once with
  `tests/bison-autotest/install-wsl-deps.sh`.

The upstream test sources live in the baselines under `upstream/`; the adapted copies are vendored
under `tests/`.

## Architecture

The CMake project is split into three subdirectories, each its own CMake project rolled into the
top-level build via `add_subdirectory`:

- **`common/`** (`winflexbison_common` static lib) — the portability layer shared by both tools.
  Built as C90. Contains:
  - `misc/` — bitset (`misc/bitset`) and gnulib thread helpers (`misc/glthread`) used by bison.
  - `m4/` — a trimmed-down vendored copy of gnulib (`m4/lib`), providing POSIX/gnulib functions
    (regex, malloc wrappers, etc.) that Windows/MSVC lacks natively. A few files are deliberately
    excluded from the build (`regexec.c`, `regcomp.c`, `regex_internal.c`,
    `dynarray-skeleton.c`) because flex/bison's own vendored regex sources are used instead.
  Both `flex/` and `bison/` link against this library.

- **`flex/`** (`win_flex.exe`) — vendored, Windows-patched copy of Flex's `src/`. Builds the main
  scanner-generator executable, excluding `libmain.c`/`libyywrap.c` from the exe (those instead
  form the separate `fl` static library, mirroring upstream Flex's `libfl`).

- **`bison/`** (`win_bison.exe`) — vendored, Windows-patched copy of Bison's `src/` plus
  `bison/data/` (the skeleton/output templates Bison reads at runtime — copied next to the built
  exe via a post-build step) and `bison/lib/` (built into the `y` static library, mirroring
  upstream Bison's `liby`). Excludes generated scanner/parser/skeleton-compiler sources
  (`scan-code.c`, `scan-gram.c`, `scan-skel.c`) since those would normally be produced by
  flex/bison themselves during upstream's own build.

When porting a fix from upstream Flex or Bison, the counterpart source usually lives at the same
relative path under `flex/src/` or `bison/src/` — diff against the version-matched baseline in
`upstream/flex` or `upstream/bison` (see `docs/specs/04-port-change-catalog/spec.md` for the diff recipes)
to see what changed.

Two other top-level pieces are not part of the compiled product:
- **`custom_build_rules/`** — MSBuild custom-build-rule triplets (`.props`/`.targets`/`.xml`) that
  *consumers* of win_flex_bison install into their own Visual Studio projects to auto-invoke
  win_flex/win_bison on `.l`/`.y` files. See `custom_build_rules/README.md`.
- **`chocolatey/`** — Chocolatey package definition for distributing releases.

## Versioning

The project version is set once, in the `project(winflexbison VERSION x.y.z ...)` call in the
root `CMakeLists.txt`. Bump it there and add a matching entry at the top of `changelog.md` when
cutting a release. Per `README.md`: 2.4.x package versions bundle Bison 2.7, 2.5.x bundle Bison
3.x.

## Commit messages

Keep them short: a concise summary line, and at most a couple of body lines only when the *why*
isn't obvious. Don't restate the diff or write long explanations.
