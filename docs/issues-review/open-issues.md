# Open Issues Backlog

**Purpose:** A snapshot of all currently-open, upstream-facing issues on
[`lexxmark/winflexbison`](https://github.com/lexxmark/winflexbison), collected in one place for
future triage and fixing.

**Methodology:** Source is the GitHub API (`GET /repos/lexxmark/winflexbison/issues`), filtered to
`state=open` with pull requests excluded. Generated 2026-07-26.

**Total open issues:** 20

Issues are grouped by theme where a natural grouping emerged; each entry shows the issue number,
title, link, a short paraphrase of the ask/report, and its labels (if any).

---

## Build, Packaging & Distribution Tooling

| # | Title | Summary | Labels |
|---|-------|---------|--------|
| [6](https://github.com/lexxmark/winflexbison/issues/6) | Make flex-lib and bison-lib projects | Requests splitting the flex/bison sources into separate library projects (rather than building straight into the two executables), presumably to make the parser/lexer internals reusable as libraries. | enhancement, Bison, Flex, low-priority |
| [16](https://github.com/lexxmark/winflexbison/issues/16) | restructuring the repo | A long discussion (split off from #14) proposing a repo layout overhaul: drop the committed `bin/` binaries, merge the separate bison2/bison3 branches into one tree with side-by-side folders, move `custom_build_rules`/`FlexLexer.h` out of `bin/Release`, and add a `make_dist.bat` that assembles a distributable output directory for a chosen VS/bison version. |  |
| [43](https://github.com/lexxmark/winflexbison/issues/43) | chocolatey package and include paths | A user integrating the Chocolatey package with CMake's `find_package(FLEX)`/`find_package(BISON)` reports that `FLEX_INCLUDE_DIR` isn't populated, so `FlexLexer.h` can't be found even though the `win_flex.exe` shim resolves correctly — the header isn't installed alongside the executable. | |
| [69](https://github.com/lexxmark/winflexbison/issues/69) | Trouble compiling a *.l file using CMake | A user following the C++ example and CMake's `FLEX_TARGET` macro gets a "FlexLexer.h: No such file or directory" compile error, and can't figure out how to make CMake/the compiler locate the header. | help wanted |
| [78](https://github.com/lexxmark/winflexbison/issues/78) | Visual Studio dependencies and the provided CustomBuildRules | Reports that Visual Studio doesn't reliably re-run bison when the `.y` source changes (only when the `.tab.*` outputs are missing), traced to the generated output files defaulting to "excluded from build"; suggests the custom build rules or their docs should set that property automatically or document the workaround. | |
| [93](https://github.com/lexxmark/winflexbison/issues/93) | Chocolatey for 2.5.25 (Bison 3.8) | Asks whether the Chocolatey package (still on 2.5.24 / pre-3.8 Bison) is intentionally lagging behind, or if a Bison-3.8-based release is planned for Chocolatey. | |
| [96](https://github.com/lexxmark/winflexbison/issues/96) | Typo in repo metadata | Points out a typo — "winflexbision" instead of "winflexbison" — in the GitHub repo's description/summary metadata. | |
| [97](https://github.com/lexxmark/winflexbison/issues/97) | Add support install win_bison and win_flex as symbolic link by using winget | When installed via winget, `win_bison.exe`/`win_flex.exe` are placed as symbolic links in a WinGet Links folder; win_bison then fails to locate its `data/m4sugar/m4sugar.m4` support files because it resolves paths relative to the symlink location rather than the real install directory. Asks for path resolution that works correctly through symlinks. | |

## Testing

| # | Title | Summary | Labels |
|---|-------|---------|--------|
| [17](https://github.com/lexxmark/winflexbison/issues/17) | re-add bison + flex tests | Asks for the upstream bison/flex test suites to be integrated and runnable again, noting bison's autotest suite could be wired up fairly directly while flex's automake-based self-tests would need more adaptation; questions why a project this critical (parser/lexer core) ships with no tests. Note: substantial progress on this has since landed in-repo (`tests/flex`, `tests/bison`, `tests/bison-autotest`, `tests/winflexbison` per current CLAUDE.md), though the upstream issue itself remains open. | enhancement, help wanted, Bison, Flex |

## Flex Behavior / Codegen

| # | Title | Summary | Labels |
|---|-------|---------|--------|
| [32](https://github.com/lexxmark/winflexbison/issues/32) | Encoding of lexical file and it generates flex.cpp file | A user writing UTF-8 (Chinese GB2312) characters in comments in their `.l` file gets an MSVC warning/compile failure because the generated `.cpp`'s encoding doesn't match the current code page; asks how to configure Visual Studio or win_flex so UTF-8 source content round-trips correctly. | Flex |
| [70](https://github.com/lexxmark/winflexbison/issues/70) | %option c++ is not compatible with %option noyywrap - linker error: multiple definition | Combining `%option c++` with `%option noyywrap` produces a linker error from a duplicate symbol definition; linked to the same bug reported upstream at westes/flex#472. | Flex, upstream |
| [73](https://github.com/lexxmark/winflexbison/issues/73) | Loss of data conversion in Flex scanner | The generated `yyFlexLexer::LexerInput` returns `yyin.gcount()` (a `std::streamsize`) as an `int`, triggering an MSVC C4244 truncation warning and a genuine risk of data loss on very large reads; suggests widening the return type. | Flex, upstream |

## Bison Behavior / Codegen

| # | Title | Summary | Labels |
|---|-------|---------|--------|
| [34](https://github.com/lexxmark/winflexbison/issues/34) | Reentrant flex/bison call yylex convention | A user combining `%option reentrant bison-bridge` (flex) with `%define api.pure` and `%lex-param`/`YYLEX_PARAM` (bison) finds the generated `YYLEX` macro's argument list doesn't match the reentrant `yylex` signature flex actually generates, and asks how to correctly wire reentrant flex scanners to pure bison parsers. | question |
| [47](https://github.com/lexxmark/winflexbison/issues/47) | win_bison: add (optional) textstyle dependency (colored output) | Requests an optional build of win_bison with the `textstyle` library so error/diagnostic output can be shown in color, similar to upstream releases that support it. | enhancement, Bison |
| [90](https://github.com/lexxmark/winflexbison/issues/90) | Using api.prefix{ ... } breaks yyerror() | With a custom `api.prefix`, the generated `#define yyerror <prefix>error` collides with a same-named token already emitted in the enum in the `.tab.hpp` header (Windows-specific behavior difference from Linux); this causes conflicts, and when multiple such parsers are linked together it leads to duplicate-symbol linker errors. An example project reproducing the bug is attached to the issue. | |
| [95](https://github.com/lexxmark/winflexbison/issues/95) | YYPTRDIFF_T not 64bit compatible? | On Win64 builds, `YYPTRDIFF_T` is hardcoded to `long` (32-bit on Windows) instead of varying by pointer size, causing an MSVC C4244 truncation warning when computing stack sizes as `__int64` differences; suggests conditionally defining it as `long long`/`LLONG_MAX` under `_WIN64`. | |
| [100](https://github.com/lexxmark/winflexbison/issues/100) | win_bison random failure extern_stdin:40: ERROR: end of file in string | Reports an intermittent win_bison parse failure ("end of file in string") seen in CI/build logs, cross-linked to a similar report filed against Mesa's GitLab; appears to be a flaky/non-deterministic bison failure rather than a consistent grammar bug. | |

## Documentation, Process & Meta

| # | Title | Summary | Labels |
|---|-------|---------|--------|
| [72](https://github.com/lexxmark/winflexbison/issues/72) | Install guide lines? | A new user asks for clearer installation guidance: where to put the executables and skeleton/data files if not using the default location, whether there's a required directory structure, and how to run this port side-by-side with a separate GnuWin32 Flex/Bison install. | |
| [89](https://github.com/lexxmark/winflexbison/issues/89) | D language support fixes reported upstream | Notes that small fixes to the `lalr1.d` skeleton and `d.m4` were submitted upstream to akimd/bison (issues #84 and #88) and asks that they be pulled into the next winflexbison release once accepted. | |
| [106](https://github.com/lexxmark/winflexbison/issues/106) | [meta] Upstreaming | A tracking/meta issue for the long-term goal of upstreaming winflexbison's Windows-support changes into Flex and Bison directly (making this fork unnecessary), listing current vendored versions (Flex 2.6.4, Bison 3.8.2, m4 1.4.19) and links to relevant upstream issues/PRs to follow. | |

---

*Generated from a one-time GitHub API pull; re-run the same query against
`lexxmark/winflexbison` issues (state=open, no pull requests) to refresh this list.*
