# Closed Pull Requests — Merged & Rejected

**Purpose:** A reference of every closed pull request on
[`lexxmark/winflexbison`](https://github.com/lexxmark/winflexbison) — what actually shipped, and
what was proposed but not merged — cross-referenced against
[`closed-issues-test-cases.md`](closed-issues-test-cases.md) where a PR's title/body names an
issue documented there.

**Methodology:** Source is the GitHub API (`GET /repos/lexxmark/winflexbison/pulls`), filtered to
`state=closed`. Generated 2026-07-26.

**Totals:** 38 closed PRs reviewed — **35 merged**, **3 closed without merging**.

---

## Merged (35)

Grouped by rough theme; within each group, sorted by PR number.

### Build, CI & Toolchain Support

| # | Title | Summary | Cross-ref |
|---|-------|---------|-----------|
| [1](https://github.com/lexxmark/winflexbison/pull/1) | add Visual Studio file 2013 support | Adds VS2013 project files to the solution-based build. | |
| [23](https://github.com/lexxmark/winflexbison/pull/23) | Declare Visual Studio 2013 as min requirement | Documents/enforces VS2013 as the minimum supported toolset, following discussion in #21. | relates to #21 |
| [25](https://github.com/lexxmark/winflexbison/pull/25) | Add AppVeyor CI | Sets up AppVeyor to build a matrix of VS2013/2015/2017 x {x64, Win32} x {Release, Debug}, since the maintainer couldn't test all VS versions locally. | |
| [28](https://github.com/lexxmark/winflexbison/pull/28) | AppVeyor: Make artifacts names more specific | Renames AppVeyor build artifacts so they're distinguishable per build-matrix leg. | |
| [50](https://github.com/lexxmark/winflexbison/pull/50) | AppVeyor: Test with VS 2019 as well | Extends the AppVeyor build matrix to include VS2019. | |
| [51](https://github.com/lexxmark/winflexbison/pull/51) | Delete VS2013 files | Removes the VS2013 project files, following the bump of the minimum supported toolset to VS2015. | |
| [63](https://github.com/lexxmark/winflexbison/pull/63) | Fix build with VS2019 | Fixes a `dup_safer` redefinition/linkage conflict (`C2375`) between the vendored gnulib `unistd-safer.h` and the Windows SDK's `corecrt_io.h` by ensuring `corecrt_io.h` is included first. | |
| [83](https://github.com/lexxmark/winflexbison/pull/83) | CI: Test Visual Studio 2022 64-bit builds | Adds VS2022 x64 to the AppVeyor build matrix. | |
| [85](https://github.com/lexxmark/winflexbison/pull/85) | minor adjustments for increased mingw64 compat | Small portability tweaks toward MinGW-w64 support (author notes it's "far away from complete"), referencing the still-open MinGW PR #59. | relates to open PR #59 |
| [104](https://github.com/lexxmark/winflexbison/pull/104) | Fix clang-cl compilation | Scopes the MSVC-only `__extension__` compile definition away from clang-cl, which also reports as MSVC-compatible but needs the real GNU-extension meaning for intrinsics headers like `<mmintrin.h>`. | fixes #103 |
| [105](https://github.com/lexxmark/winflexbison/pull/105) | Add CI for clang-cl | Adds a CI job building with clang-cl/Ninja, now that #104 makes that combination work. | follow-up to #104 |

### CMake Project Structure

| # | Title | Summary | Cross-ref |
|---|-------|---------|-----------|
| [52](https://github.com/lexxmark/winflexbison/pull/52) | Add option to use Win STATIC runtime | Adds the `USE_STATIC_RUNTIME` CMake option to link `/MT` instead of `/MD`, addressing the extra `vcruntime140.dll` dependency of CMake-built executables. | relates to #39 |
| [53](https://github.com/lexxmark/winflexbison/pull/53) | CMake: allow using win_bison in-build | Copies the `data/` directory into the build tree so consuming projects can invoke `win_bison` directly from the build directory without installing it. | |
| [54](https://github.com/lexxmark/winflexbison/pull/54) | CMake: make winflexbison easier to use as a subdirectory | When added via `add_subdirectory`, skips output-directory overrides, install/CPack rules, and the global `USE_STATIC_RUNTIME` option; also renames `common_lib` to `winflexbison_common`. | |
| [55](https://github.com/lexxmark/winflexbison/pull/55) | CMake: add CMakeSettings.json | Adds VS's `CMakeSettings.json` defining Release/RelWithDebInfo/Debug configurations. | |
| [79](https://github.com/lexxmark/winflexbison/pull/79) | [cmake] Improve support using CMake | Adds `libfl`/`liby` targets, supports consuming the project as a CMake target dependency, and fixes a `C_STANDARD` build issue with newer MSVC. | |
| [101](https://github.com/lexxmark/winflexbison/pull/101) | Update cmake to fix warnings | Bumps `cmake_minimum_required` to 3.10 (3.5 breaks on CMake 4.0; below 3.10 is deprecated) and explicitly sets policy `CMP0177` to `OLD` to silence an install-path-normalization warning. | relates to #98 |
| [102](https://github.com/lexxmark/winflexbison/pull/102) | Fix CI, include all licenses, simplify CMake | A combined cleanup: fixes an AppVeyor MSBuild-mode error, bundles all license files in CPack packages (completing what #84 started), marks the projects C-only to skip unneeded C++ compiler checks, and touches up the README. | fixes #94 (superseding #84); touches #93 |
| [107](https://github.com/lexxmark/winflexbison/pull/107) | CMake: Migrate VS-only and deprecated CMakeSettings to CMakePresets | Replaces the VS-specific `CMakeSettings.json` with the cross-IDE `CMakePresets.json` standard. | |

### Packaging & Distribution

| # | Title | Summary | Cross-ref |
|---|-------|---------|-----------|
| [26](https://github.com/lexxmark/winflexbison/pull/26) | Add CPack configuration / Build AppVeyor Artifacts | Wires up CPack so AppVeyor produces downloadable zip packages. | |
| [27](https://github.com/lexxmark/winflexbison/pull/27) | Fix CPack packages | Removes redundant files from the package, includes `custom_build_rules/`, and fixes the debug package. | |
| [37](https://github.com/lexxmark/winflexbison/pull/37) | updated nuspec+chocolatey, integrated winbison (v2.x) | Updates the Chocolatey package spec and folds in bison 2.x support, switching to GitHub-hosted release downloads instead of SourceForge. | relates to #16 |
| [76](https://github.com/lexxmark/winflexbison/pull/76) | License clarifications | Clarifies/fixes licensing documentation. | fixes #75 |
| [84](https://github.com/lexxmark/winflexbison/pull/84) | CPack: Add license files to distribution | Adds `COPYING`/`COPYING.DOC` to the CPack zip. | partial fix for #94 (completed by #102) |

### Flex/Bison Core Fixes & Codegen

| # | Title | Summary | Cross-ref |
|---|-------|---------|-----------|
| [9](https://github.com/lexxmark/winflexbison/pull/9) | Fixed 'possible data loss' warning inside Flex yyuserAction() | Widens a truncating cast in generated-scanner code to silence an MSVC data-loss warning. | fixes part of #8 |
| [57](https://github.com/lexxmark/winflexbison/pull/57) | use explicitly the ANSI variant of GetModuleFileName | Switches an app-path lookup to explicitly call `GetModuleFileNameA`, since the code assumed ANSI strings but would silently miscompile under `-DUNICODE`. | |
| [80](https://github.com/lexxmark/winflexbison/pull/80) | change file format from utf-8 to utf-8 BOM | Adds a UTF-8 BOM to a vendored source file containing a non-ASCII bullet character, fixing an MSVC `C4819` warning/build error under VS2019. | |
| [82](https://github.com/lexxmark/winflexbison/pull/82) | upgrade win_bison to version 3.8.2 | Vendors Bison 3.8.2 into the port (the version this repo currently ships). | addresses #74 |
| [91](https://github.com/lexxmark/winflexbison/pull/91) | Fix runtime error upon multiple flex processes | Fixes the temp-file race between concurrent `win_flex` invocations (PID-less `_tmpnam`-derived names colliding under heavy parallel builds). | fixes #86 |

### Documentation & Build Rules

| # | Title | Summary | Cross-ref |
|---|-------|---------|-----------|
| [19](https://github.com/lexxmark/winflexbison/pull/19) | VS custom build rules - Import from SF Wiki and Update | Imports the custom-build-rules documentation from the old SourceForge wiki into the repo and updates it. | fixes #15 |
| [36](https://github.com/lexxmark/winflexbison/pull/36) | update to documentation for custom build rules | Refreshes the custom-build-rules docs now that full docs live in-repo as Markdown. | |
| [38](https://github.com/lexxmark/winflexbison/pull/38) | custom_build_rules: fixed flex manual URL | Fixes a dead link to the Flex manual (SourceForge page removed) to point at the current `westes.github.io` docs. | |
| [40](https://github.com/lexxmark/winflexbison/pull/40) | adjusted README and documentation for custom build rules | Further README/doc adjustments for the custom build rules. | |
| [42](https://github.com/lexxmark/winflexbison/pull/42) | custom_build_rules bison: adjust outputfile to include defines file | Adds the `--defines` header as a declared output file in the bison custom build rule, so MSBuild tracks it correctly. | |

### Misc

| # | Title | Summary | Cross-ref |
|---|-------|---------|-----------|
| [65](https://github.com/lexxmark/winflexbison/pull/65) | Development->master | Merges the long-running `development` branch into `master`, folding in accumulated changes (including a binary-mode question later resolved in discussion). | |

---

## Closed without merging (3)

### [#2 — Add checksum for Chocolatey download](https://github.com/lexxmark/winflexbison/pull/2)

Opened 2016-08-16, closed 2016-08-25 (9 days), 3 comments, not merged.

Proposed adding a checksum to the Chocolatey package download to comply with a Chocolatey policy
change requiring checksums on all packages. **Why it likely wasn't merged as-is:** unclear from
available data — could have been handled differently (e.g. checksum added directly by the
maintainer, or the specific checksum/approach in the PR was rejected pending a different fix). The
underlying need (Chocolatey compliance) was almost certainly addressed some other way, since the
project's Chocolatey package still ships today.

### [#66 — Fixed some redefinition questions caused by the skel.c file and some code / Fixed output executable filenames as bison.exe and flex.exe](https://github.com/lexxmark/winflexbison/pull/66)

Opened 2020-10-27, closed 2020-11-03 (7 days), 5 comments, not merged.

Proposed two changes bundled together: fixing redefinition warnings/errors related to
`skel.c`-generated code, and renaming the output executables to plain `bison.exe`/`flex.exe`
(rather than `win_bison.exe`/`win_flex.exe`). **Why it likely wasn't merged:** the executable
rename is a significant, deliberate naming choice for this project (distinguishing the port from
a system/upstream bison/flex on the same machine) — a maintainer would plausibly reject that part
outright, and bundling it with the unrelated skel.c fix likely made the whole PR hard to accept as
one unit. The skel.c-related fix may have been picked up separately or superseded; unclear from
available data.

### [#87 — updated](https://github.com/lexxmark/winflexbison/pull/87)

Opened 2022-04-05, closed 2026-03-03 (**~3.9 years open**), 3 comments, not merged, no body/title
detail beyond "updated".

Effectively no description is available (title is just "updated" and the body is empty), so the
actual content of the change is unclear from available data. Given it sat open for nearly four
years before being closed, it was most likely abandoned/gone stale (possibly superseded by the
flurry of CMake/CI/licensing PRs that landed around the same November 2025–March 2026 window —
#101, #102, #104, #105, #107 — rather than actively rejected on its merits).

---

*Generated from a one-time GitHub API pull; re-run the same query against
`lexxmark/winflexbison` pull requests (state=closed) to refresh this list.*
