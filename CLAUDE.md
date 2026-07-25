# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This working directory contains the project plus supporting material:

- `winflexbison/winflexbison/` — the actual project: a Windows port of Flex and Bison
  (`lexxmark/winflexbison` on GitHub). This is where almost all work happens.
- `orig/` — pristine upstream **baseline mirrors**, each checked out at the exact tag matching the
  currently vendored version, used for diffing when porting/upgrading: `orig/flex` (@ `v2.6.4`),
  `orig/bison` (@ `v3.8.2`, from `akimd/bison`, carries the test suite), `orig/m4` (@ `v1.4.19`),
  `orig/gnulib` (at the commit pinned by bison's/m4's submodule). Not built, not committed to the
  project. See `docs/specs/02-baseline-mirrors/spec.md`.
- `docs/specs/` — the **upgrade playbook**: a repeatable process for adopting new upstream
  flex/bison/m4 releases (version inventory, baselines, test adoption, the Windows port-change
  catalog, upgrade runbook, validation gate). Start at `docs/specs/README.md`.

Unless told otherwise, assume any task refers to `winflexbison/winflexbison/`. When diffing against
upstream, use `orig/<component>` as the baseline — not a fresh `master` checkout, which is newer
than the vendored version.

## Build

Requires Visual Studio 2017+ and CMake. Only MSVC (and clang-cl) toolchains are officially
supported (the root `CMakeLists.txt` warns if `MSVC` is not set).

Quick build via provided batch scripts (from `winflexbison/winflexbison/`), each configures with
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

CI mirrors these paths: AppVeyor (`.appveyor.yml`) builds VS2017/2019/2022 x64+Win32 with MSVC,
and GitHub Actions (`.github/workflows/os_windows.yaml`) builds with `clang-cl` via Ninja. Both
just run the CMake configure/build/package sequence above.

## Testing

There is no automated test suite wired into the build — `CMakeLists.txt` does not call
`enable_testing()`/`add_test()`, so `ctest` has nothing to run, and no tests are vendored in the
project. Verifying a change to `win_flex` or `win_bison` today means manually generating output
from a `.l`/`.y` file with the built executable, then compiling/running the generated C to confirm
correctness. The upstream test suites (flex's exit-code tests, bison's autotest `.at` files) live
in the baselines under `orig/`; `docs/specs/03-test-adoption/spec.md` designs how to import them
and run them under CTest.

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
`orig/flex` or `orig/bison` (see `docs/specs/04-port-change-catalog/spec.md` for the diff recipes)
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
