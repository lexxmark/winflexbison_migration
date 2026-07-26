# Closed Issues — Candidate Regression Test Cases

## Purpose

This document reviews every closed, non-PR GitHub issue on `lexxmark/winflexbison` and, where
the issue describes an actual functional/behavioral bug or feature request (codegen output, CLI
options, compile errors, runtime behavior, cross-platform quirks), proposes a concrete, plausible
regression test case in the idiom of this repo's existing suites (`tests/flex/`, `tests/bison/`,
`tests/winflexbison/` — a `.l`/`.y` fixture or CLI/CMake invocation, built with win_flex/win_bison
and MSVC, checked by exit code or diff). Issues that are purely process/meta (docs, packaging,
CI setup, repo housekeeping, version-bump tracking issues, etc.) are marked not applicable. These
are candidates for future test-suite hardening, not implemented tests.

## Methodology

- Source: GitHub API, `lexxmark/winflexbison`, issues with `state=closed`, pull requests excluded.
- Generated: 2026-07-26.
- 43 closed issues reviewed: 19 with proposed test cases, 24 marked not applicable.

## Implementation status (2026-07-26)

Of the 19 issues with a proposed test case, 12 were implemented as real CTest tests, 2 turned out
to already be covered by existing tests, and 5 were deferred (see each issue's section below for
the actual test name or the deferral rationale):

- **Implemented (12):** #3, #4, #7, #8, #10, #29, #39, #44, #48, #62, #64, #67
- **Already covered by existing tests, no new work needed (2):** #35, #86
- **Deferred, with rationale (5):** #21, #61, #77, #98, #103
- **Not applicable, process/meta (24):** unchanged, see individual sections below.

Implementing the #62 test (`bison.update_flag`) surfaced a genuine, still-live bug beyond what the
original proposal anticipated: `win_bison --update` failed with "cannot backup: Permission denied"
on the very first run whenever the deprecated construct also triggered a caret-style diagnostic,
because `location.c`'s cached source-file handle wasn't released before `fixits_run()`'s `rename()`
call. Fixed in `bison/src/main.c` (moved the `caret_free()` call earlier), in a separate commit from
the test-suite changes.

---

## #3 — register keyword used in the generated lexer

https://github.com/lexxmark/winflexbison/issues/3

The generated scanner used the C++11-deprecated `register` keyword (e.g. in
`yy_size_t number_to_move`), causing warnings/errors under strict C++11+ compilers.

**Proposed test case:**
- Fixture: any `.l` file compiled with `win_flex --wincompat` (e.g. reuse `tests/flex/cases/basic_nr.l`).
- Build the generated `.c`/`.cpp` under MSVC with `/std:c++17` (or `/Za` strict mode) and warnings-as-errors.
- Grep the generated output for the literal token `register` — expect zero matches.
- Pass: compiles cleanly with no `register`-related warnings/errors.

**Implemented as:** `flex.cxx_no_register_keyword` (`tests/flex/CMakeLists.txt`), via a new generic
content-check runner (`tests/flex/run_content_check.cmake`) applied to the already-generated
`cxx_basic.cc`.

## #4 — --header-file option in the latest win_flex

https://github.com/lexxmark/winflexbison/issues/4

`win_flex -o f-l.c --header-file=f-l.h in.l` stopped generating the header file in one release,
though it worked in an earlier one.

**Proposed test case:**
- Invocation: `win_flex --header-file=out.h -o out.c tests/flex/cases/basic_nr.l`.
- Pass/fail: assert both `out.c` and `out.h` exist after the run (exit code 0), and `out.h`
  contains expected prototype declarations (e.g. `yylex` decl).
- Similar to the existing `--header-file` usage already exercised in
  `tests/winflexbison/CMakeLists.txt` (`flex_parallel` test) — extend with an explicit
  file-existence assertion.

**Implemented as:** `flex.header_file_flag` (`tests/flex/CMakeLists.txt`), via a new generic
file-existence runner (`tests/flex/run_files_exist.cmake`).

## #5 — Migrate to CMake

https://github.com/lexxmark/winflexbison/issues/5

**Not applicable:** build-system migration/infrastructure request, not a behavioral bug (and long since done — the project is CMake-based today).

## #7 — Copy the first part of user declarations not into y.tab.h?

https://github.com/lexxmark/winflexbison/issues/7

A `YYSTYPE` typedef (e.g. `char*`) declared in the `.y` prologue was copied into `y.tab.c` but not
into the generated `y.tab.h`, so the header re-declared `YYSTYPE` as `int`, causing memory errors
when both files disagreed on the type.

**Proposed test case:**
- Fixture `.y`: define `#define YYSTYPE char*` (or a custom typedef) in the pre-prologue section, with `%defines`.
- Build: `win_bison --defines=out.h -o out.c fixture.y`, compile a small `main.c` that includes `out.h` and uses `YYSTYPE`.
- Pass: compiles without redefinition/type-mismatch warnings; `out.h`'s `YYSTYPE` matches the declared type, not the default `int`.

**Implemented as:** `bison.ystype_header` (`tests/bison/CMakeLists.txt`), fixture
`tests/bison/cases/ystype_header.y` (uses `%code requires` for the shared typedef) +
`ystype_header_main.c` (a separate TU that only sees the generated header).

## #8 — Fixing compile-time warnings

https://github.com/lexxmark/winflexbison/issues/8

Two new warnings appeared in generated code after a version bump: (1) `size_t`→`int` narrowing in
`yyuserAction()`/`yyfill()`, and (2) `yylex` macro redefined between scanner and parser layers.

**Proposed test case:**
- Fixture: a flex `.l` + bison `.y` pair using an intermediate `#define yylex core_yylex`-style wrapper (mirrors the reporter's setup).
- Compile generated scanner+parser together under MSVC `/W4` (or clang-cl `-Wall`).
- Pass: no `C4267`-style narrowing warning from `yyuserAction`/`yyfill`, and no macro-redefinition warning for `yylex`.

**Implemented as:** `flex.core_yylex_wrapper` (`tests/flex/CMakeLists.txt`, via a new
`EXTRA_COMPILE_OPTIONS` parameter on `add_flex_bison_test()`), fixture triad
`tests/flex/cases/core_yylex_wrapper_{parser.y,scanner.l,main.c}` (a reentrant, bison-bridge
scanner + pure parser, the combination where both warnings historically surfaced), compiled with
`/we4267 /we4005` (the two specific warnings named, not a blanket `/W4`/`/WX`, to stay stable
across VS versions and avoid tripping on unrelated warnings elsewhere in generated code).

## #10 — Won't compile under MSVC 2017 preview

https://github.com/lexxmark/winflexbison/issues/10

`location.cc:80` did `-static_cast<unsigned int>(rhs)` (negating after cast) which MSVC 2017
rejected; fix is to negate first, then cast.

**Proposed test case:**
- Fixture: a `%language "c++"` `.y` file using `%locations`, generating `location.hh`/`location.cc`-equivalent code.
- Build with MSVC (any supported version) in C++ mode.
- Pass: compiles with exit code 0 — regression-guards the specific negate/cast ordering pattern in the location arithmetic.

**Implemented as:** `bison.cxx_locations` (`tests/bison/CMakeLists.txt`), via a new
`add_bison_cxx_run_test()` function (the first native C++ bison fixture in the CTest suite; the
prior only C++ bison grammar lived in the separate, opt-in WSL GNU-Autotest harness), fixture
`tests/bison/cases/cxx_locations.y` (`%locations`, exercises `@1` for real) + `cxx_locations_main.cc`.

## #11 — virus scanner complains

https://github.com/lexxmark/winflexbison/issues/11

**Not applicable:** third-party antivirus false-positive on distributed binaries — not a functional bug in flex/bison/win_flex/win_bison behavior.

## #12 — Make github release with CMakeLists.txt included

https://github.com/lexxmark/winflexbison/issues/12

**Not applicable:** release/packaging process request (what's bundled in GitHub Releases), not behavioral.

## #13 — split HISTORY.md from README.md

https://github.com/lexxmark/winflexbison/issues/13

**Not applicable:** documentation restructuring request.

## #14 — update release page to include history of the releases and binaries

https://github.com/lexxmark/winflexbison/issues/14

**Not applicable:** release-page/process/documentation request.

## #15 — include the documentation for creating VS-build rules in source

https://github.com/lexxmark/winflexbison/issues/15

**Not applicable:** documentation relocation/format request (wiki → in-repo docs).

## #20 — Update to bison 3.0.5

https://github.com/lexxmark/winflexbison/issues/20

**Not applicable:** upstream version-bump tracking issue, no specific behavior described.

## #21 — Build fails with: Cannot open include file: 'src/flex-scanner.h'

https://github.com/lexxmark/winflexbison/issues/21

A generated/vendored source (`bison/src/scan-gram.c`) contained a hardcoded absolute/relative
include path that broke builds from a fresh clone/different directory layout.

**Proposed test case:**
- Build step: configure and build the project from a completely clean clone into a fresh, differently-named build directory (not matching the original checkout path), on a machine/path with no pre-existing `src/` relative context.
- Pass: `cmake --build` succeeds with exit code 0 and no "cannot open include file" errors — guards against reintroducing hardcoded paths in vendored generated sources.

**Deferred.** A design review (reading `bison/src/scan-gram.c` directly) found the actual generated
include (`#include "src/flex-scanner.h"`, etc.) is a *relative*, compiler-`-I`-resolved path, not a
hardcoded absolute path or drive-letter/username string — so a content-check test would trivially
pass without testing anything. Genuinely reproducing the regression needs a full nested
`cmake -S -B` + `cmake --build` from a scratch, differently-named directory, which is too expensive
to run on every default `ctest` invocation. Left as a possible future opt-in test mirroring the
`WFB_WSL_AUTOTEST` gate, not implemented in this pass.

## #22 — Bison 3.1 has been released

https://github.com/lexxmark/winflexbison/issues/22

**Not applicable:** upstream version-bump tracking issue.

## #24 — Remove fork status

https://github.com/lexxmark/winflexbison/issues/24

**Not applicable:** GitHub repository metadata/administrative request.

## #29 — Visual Studio 2017 and inttypes.h

https://github.com/lexxmark/winflexbison/issues/29

Generated flex integer-type header (`FLEXINT_H`) checked `__STDC_VERSION__ >= 199901L` to decide
whether to use `<inttypes.h>`; MSVC doesn't set `__STDC_VERSION__` but does have `<stdint.h>`, so
the fallback branch redefined `INT8_MIN` etc. against the SDK's own `<stdint.h>`, causing C4005.

**Proposed test case:**
- Fixture: any `.l` file generating the standard `FLEXINT_H` preamble, compiled with MSVC (`cl.exe`) with `<stdint.h>` already included/available (default Windows SDK).
- Build with warnings-as-errors (`/WX`) including C4005.
- Pass: no macro-redefinition warnings for `INT8_MIN`/`INT16_MIN`/etc. — guards the `_MSC_VER`/`--wincompat` branch of the integer-type preamble.

**Implemented as:** `flex.flexint_h_stdint` (`tests/flex/CMakeLists.txt`) — recompiles the
already-generated `basic_nr.c` a second time with `/we4005`.

## #30 — 2.5.15 is no longer available on SourceForge

https://github.com/lexxmark/winflexbison/issues/30

**Not applicable:** third-party distribution/hosting (Chocolatey/SourceForge availability) issue.

## #31 — Bison 3.3.1 has been released

https://github.com/lexxmark/winflexbison/issues/31

**Not applicable:** upstream version-bump tracking issue.

## #33 — Bison 3.3.2 has been released

https://github.com/lexxmark/winflexbison/issues/33

**Not applicable:** upstream version-bump tracking issue.

## #35 — Bison gives error data/skeletons/yacc.c:1652: undefined macro `b4_symbol(114, has_type)'

https://github.com/lexxmark/winflexbison/issues/35

A grammar with a useless nonterminal (reported as "1 nonterminal useless in grammar") triggered an
`undefined macro b4_symbol` skeleton-expansion error in win_bison, while other bison builds handled
it fine.

**Proposed test case:**
- Fixture `.y`: a grammar deliberately containing one unused/useless nonterminal and one useless rule (matching the `-Wother` warnings quoted in the issue).
- Run `win_bison` on it (default yacc.c skeleton).
- Pass: exit code 0, no `undefined macro b4_symbol` fatal error, and the `-Wother` useless-nonterminal/rule warnings are emitted as warnings (not escalated to a crash).

**Already covered:** the existing golden fixture `tests/bison/golden-cases/useless.y`
(`bison.golden.useless`) already reproduces this exact trigger pattern — a useless nonterminal
*and* a useless rule together, the same `-Wother` warnings quoted in this issue. No new fixture
needed.

## #39 — CMake based build produces executable with more dependencies

https://github.com/lexxmark/winflexbison/issues/39

VS-solution-built `win_bison.exe`/`win_flex.exe` only depended on `kernel32.dll` (+`ws2_32.dll` for
flex), but the CMake-built executables additionally pulled in `vcruntime140.dll` (dynamic CRT).

**Proposed test case:**
- Build via CMake with `USE_STATIC_RUNTIME=ON`.
- Inspect `bin/Release/win_bison.exe` and `win_flex.exe` imports (e.g. via `dumpbin /dependents`).
- Pass: import table does not list `vcruntime140.dll` (or any dynamic-CRT DLL) when the static-runtime option is enabled — regression-guards the CMake/CRT linkage option.

**Implemented as:** `winflexbison.static_runtime_no_vcruntime` (`tests/winflexbison/CMakeLists.txt`),
only registered `if(USE_STATIC_RUNTIME)` (mirroring the `WFB_WSL_AUTOTEST` opt-in precedent), using
`dumpbin.exe` (located via `find_program`, falling back to the MSVC toolset directory since
`dumpbin` isn't on PATH outside a VS dev prompt) via a new
`tests/winflexbison/run_no_vcruntime_check.cmake`. Verified in a separate `-DUSE_STATIC_RUNTIME=ON`
build (132/132 tests passed there, vs. 131/131 in the default build).

## #41 — Can we delete 3.x branches?

https://github.com/lexxmark/winflexbison/issues/41

**Not applicable:** git branch-management/repo housekeeping question.

## #44 — Is it possible to change table construction in bison?

https://github.com/lexxmark/winflexbison/issues/44

Question about whether win_bison supports selecting the LR table-construction algorithm via
`%define lr.type`.

**Proposed test case:**
- Fixture `.y`: a small LALR grammar plus `%define lr.type canonical-lr` (and separately `ielr`).
- Run `win_bison` on each variant.
- Pass: exit code 0 for both variants, and the generated parser tables differ appropriately from the default LALR build (e.g. diff generated `.output`/`.c` against a plain-LALR run) — confirms the `%define lr.type` variable is honored end-to-end in the Windows port.

**Implemented as:** `bison.lr_type_canonical` and `bison.lr_type_ielr` (`tests/bison/CMakeLists.txt`,
via the existing `add_bison_run_test()`), fixtures `tests/bison/cases/lr_type_{canonical,ielr}.y`
(calc.y's grammar body plus the respective `%define lr.type ...`). Scoped to "accepts the option and
still parses correctly" (exit 0), without diffing generated tables against plain LALR — matches the
original proposal's own "stretch goal, not required" framing.

## #45 — Update bison to 3.4.1

https://github.com/lexxmark/winflexbison/issues/45

**Not applicable:** upstream version-bump tracking issue.

## #46 — Bison 3.4.2 is out

https://github.com/lexxmark/winflexbison/issues/46

**Not applicable:** upstream version-bump tracking issue.

## #48 — win_bison can't find data directory unless it's in the current directory

https://github.com/lexxmark/winflexbison/issues/48

Starting with 2.5.19, `pkgdatadir()` in `bison/src/files.c` stopped calling the Windows-specific
`get_local_pkgdatadir()`/`get_app_path()` helper, so win_bison could only find its `data/` skeleton
directory when invoked from the directory containing it — failing with
`data/m4sugar/m4sugar.m4: cannot open` otherwise.

**Proposed test case:**
- Invocation: run `win_bison.exe` (full path, e.g. `bin/Release/win_bison.exe`) with the current working directory set to somewhere else entirely (e.g. a temp dir with no `data/` subfolder), passing a simple `.y` fixture by absolute path.
- Pass: exit code 0, no "cannot open" `data/m4sugar/...` error — confirms `pkgdatadir()` resolves relative to the executable's own location (`get_app_path()`), not the process CWD.

**Implemented as:** `bison.data_dir_from_exe_location` (`tests/bison/CMakeLists.txt`) — a bare
`add_test` running `win_bison` with `WORKING_DIRECTORY` set to the CTest binary dir (unrelated to
`data/`'s location); no compile step or custom runner needed, CTest's default nonzero-exit-is-fail
already suffices.

## #49 — Bison 3.5.0 released

https://github.com/lexxmark/winflexbison/issues/49

**Not applicable:** upstream version-bump tracking issue.

## #56 — Update bison to 3.6.4

https://github.com/lexxmark/winflexbison/issues/56

**Not applicable:** upstream version-bump tracking issue.

## #58 — Bison Windows executable for 3.5.4+

https://github.com/lexxmark/winflexbison/issues/58

**Not applicable:** version-availability/release-timeline request (driven by a CVE in upstream bison-the-program, not the generated parsers).

## #60 — Bison-3.7.2

https://github.com/lexxmark/winflexbison/issues/60

**Not applicable:** upstream version-bump tracking issue (follow-up to #58).

## #61 — v2.5.23 not working on Windows XP

https://github.com/lexxmark/winflexbison/issues/61

`win_bison.exe`/`win_flex.exe` built with a newer toolset produced a PE image XP's loader rejected
("%1 is not a valid Win32 application"), whereas 2.4-era binaries worked on XP.

**Proposed test case:**
- Build step: with the project's minimum-supported VS toolset/CMake target, inspect the built `win_bison.exe`/`win_flex.exe` PE header (`dumpbin /headers`) for the linker/subsystem minimum-OS version fields.
- Pass: the subsystem version matches the project's currently documented minimum supported Windows version (this is now historical — Windows XP is no longer a support target — so this test doubles as a "don't silently raise the minimum-OS floor" guard rather than an XP-compatibility guarantee).

**Deferred:** no documented "minimum supported Windows version" invariant exists anywhere in the
repo to assert against — implementing this properly would mean inventing new project policy, not
just adding a test. Low value now that Windows XP is historical. Not implemented in this pass.

## #62 — Can not fix file via --update

https://github.com/lexxmark/winflexbison/issues/62

`win_bison --update file.y` failed with `cannot backup: Permission denied` — the rename-based
backup-then-update sequence didn't handle Windows file-locking/renaming semantics.

**Proposed test case:**
- Fixture: a `.y` file using deprecated directive syntax that `--update` is meant to rewrite (per bison's `--update` feature).
- Invocation: `win_bison --update file.y` on a freshly-copied, non-read-only file with no other handle open on it.
- Pass: exit code 0, no "cannot backup: Permission denied", and `file.y` is rewritten in place with a `.y~`-style backup created.

**Implemented as:** `bison.update_flag` (`tests/bison/CMakeLists.txt`, new runner
`tests/bison/run_update_test.cmake`), fixture `tests/bison/cases/update_deprecated.y` (uses the
deprecated `%no_lines` directive, rewritten to `%no-lines`).

**This test caught a real, currently-live bug**, not just a historical one: `win_bison --update`
failed with "cannot backup: Permission denied" on the very first run whenever the deprecated
construct also triggered a caret-style diagnostic (which `%no_lines` does). Root cause:
`location.c`'s cached source-file `FILE*` (opened to quote the source line in the warning) was only
released via `complain_free()`, called in `bison/src/main.c` *after* `fixits_run()` already
attempted `rename()` on that same still-open file — Windows blocks renaming a file with an open
handle; the existing `remove(backup)` fix (commit `ca0317d`) only covered the "stale backup already
exists" case, not this one. **Fixed** in `bison/src/main.c` (call `caret_free()` before the fixits
block runs), in a separate commit from the test-suite changes. Verified both via a standalone manual
repro and via the CTest suite (130/131 passing before the fix, `bison.update_flag` the sole
failure; 131/131 passing after).

## #64 — Crash on invalid input

https://github.com/lexxmark/winflexbison/issues/64

A malformed `.y` file (a typo'd variant of the Bison manual's `rpcalc` example) caused win_bison to
crash via a NULL pointer dereference instead of reporting a clean diagnostic.

**Proposed test case:**
- Fixture: the attached malformed rpcalc-derived `.y` (or a reconstructed equivalent with a similar syntax typo).
- Invocation: `win_bison rpcalc_bad.y`.
- Pass: nonzero exit code with a clean diagnostic message on stderr; no crash (no access violation / no unhandled SEH exception) — add as a `tests/bison/cases` negative/error-path test similar to existing golden-diff diagnostics tests.

**Implemented as:** `bison.golden.rpcalc_bad` (auto-registered by the existing golden-diff
mechanism in `tests/bison/CMakeLists.txt`, zero CMake changes needed). The fixture
(`tests/bison/golden-cases/rpcalc_bad.y`) is the **actual file originally attached** to this issue
(downloaded from the GitHub attachment linked in the issue body, byte-for-byte aside from CRLF→LF
normalization) — the Bison manual's rpcalc example with a real typo (a missing opening quote in a
`printf` action). Golden captured via `tests/bison/generate.sh` under WSL against real reference
bison 3.8.2: exits 1 with a clean "missing '\"' at end of line" / "missing '}' at end of file"
diagnostic — confirming the fixture reproduces a genuine parse-error path, not something reference
bison itself chokes on.

## #67 — Visual Studio - win_bison creates tab.h file exceeding compiler limits

https://github.com/lexxmark/winflexbison/issues/67

With ~830 tokens, the C++ skeleton's `YY_ASSERT` line in `symbol_type` constructors became a single
line over 21,740 characters, hitting MSVC's C2026 "string too big" limit.

**Proposed test case:**
- Fixture `.y`: `%language "c++"` grammar with 800+ distinct tokens (generatable via a small script emitting `%token TOK1 TOK2 ... TOK900`).
- Build the generated `.tab.h`/`.tab.cpp` under MSVC.
- Pass: compiles without C2026 ("string too big") — confirms the `YY_ASSERT` line-length fix (rewritten in bison 3.7.4) holds for the vendored skeleton.

**Implemented as:** `bison.many_tokens` (`tests/bison/CMakeLists.txt`, reusing the
`add_bison_cxx_run_test()` function built for #10), fixture `tests/bison/cases/many_tokens.y`
(committed directly, ~900 `%token` declarations, using `api.token.constructor` + `variant` — the
"complete symbols" API that actually generates the `symbol_type` constructors the bug was in) +
`many_tokens_main.cc`.

## #68 — Update to bison 3.7.4

https://github.com/lexxmark/winflexbison/issues/68

**Not applicable:** version-bump tracking issue; the actual behavioral regression it fixes (long `YY_ASSERT` lines) is already covered by the #67 test case proposed above.

## #74 — Update to Bison 3.8.1

https://github.com/lexxmark/winflexbison/issues/74

**Not applicable:** upstream version-bump tracking issue (long-lived umbrella issue quoting the 3.7.5+ NEWS).

## #75 — Missing LICENSE/COPYING file in repository toplevel

https://github.com/lexxmark/winflexbison/issues/75

**Not applicable:** licensing/legal-file presence in the repo, not a behavioral bug.

## #77 — Cannot compile Bison input in C++ mode

https://github.com/lexxmark/winflexbison/issues/77

A user's `%language "c++"` + `%define api.value.type variant` grammar failed to compile with over
100 MSVC errors. Most stem from grammar mistakes in the reported `.y` (e.g. `%type <std::vector<float>>` combined with plain `$$ = {$1}` assignments, undeclared `fval`/`ival` tags), but the build
log also shows a hard tooling gap: `fatal error C1083: Cannot open include file: 'FlexLexer.h'`
when compiling the flex-generated C++ scanner.

**Proposed test case:**
- Fixture: a minimal valid `%language "c++"` `.y` + a `.l` generating a C++ scanner (`win_flex --c++`), following the custom MSBuild build rules (`custom_build_rules/`) exactly as an end user would.
- Build via MSVC using only the include paths the custom build rules/package are documented to set up (no manual extra include paths).
- Pass: the generated scanner `.cpp` finds `FlexLexer.h` without a manual include-path fix — confirms the packaged/build-rule include paths cover the C++ scanner class header out of the box.

**Deferred:** exercises the packaged `custom_build_rules/` MSBuild `.props`/`.targets` (consumed by
end-user Visual Studio projects), a completely different build mechanism outside the CMake/CTest
tree entirely. Would need a new MSBuild-based test harness that doesn't currently exist. Not
implemented in this pass.

## #81 — Could this project build with gcc

https://github.com/lexxmark/winflexbison/issues/81

**Not applicable:** design/scope question (project is intentionally MSVC/clang-cl only per `CMakeLists.txt`'s `FATAL_ERROR` guard); not a bug.

## #86 — Runtime error when running multiple flex processes

https://github.com/lexxmark/winflexbison/issues/86

Concurrent `win_flex` invocations could race on the same `_tmpnam`-derived temp filename before the
file existed, since the name didn't incorporate the process ID — causing "error deleting file"
under heavy parallel build systems (e.g. `sw`/ninja with thousands of `.l` files).

**Proposed test case:**
- This is already exercised by `tests/winflexbison/CMakeLists.txt`'s `winflexbison.flex_parallel` test (`run_parallel.ps1`, 8 jobs × 25 iterations against `basic_nr.l` with `--header-file`), added per the repo's port-specific "parallel resistance" test suite (see commit `08bfad7`).
- Suggested hardening: raise `-Jobs`/`-Iterations` in a stress variant, and add an explicit assertion that no two concurrent processes ever observe the same temp filename mid-run (not just no leaked files at the end) — directly regression-guards the PID-less-tmpnam race described here.

**Already covered:** confirmed `winflexbison.flex_parallel`/`winflexbison.bison_parallel`
(`tests/winflexbison/run_parallel.ps1`) already assert this — correctness (SHA256 match vs.
sequential reference), real concurrency (peak temp-file-pattern count exceeds baseline), and
cleanup. No new test added; the suggested stress-hardening above remains a possible future
enhancement, not implemented in this pass.

## #94 — BSD license and GPL license missing from the binary package

https://github.com/lexxmark/winflexbison/issues/94

**Not applicable:** packaging/distribution completeness (what files ship in the zip/cpack package), not tool behavior.

## #98 — Project is incompatible with modern tools

https://github.com/lexxmark/winflexbison/issues/98

Two build-system issues: (1) the project requires CMake 3.5 (removed) / 3.10 (deprecated) compatibility mode in newer CMake releases; (2) `clang-cl` always defines `__clang__`/`clang`, but the CMakeLists assumed "if clang, use GNU-compatible flags," when it should check `_MSC_VER`/`MSVC` instead.

**Proposed test case:**
- Build 1: configure with a current CMake release (3.20+) and confirm no deprecation warning/error about minimum-required version.
- Build 2: configure and build fully with `clang-cl` (as GitHub Actions' `os_windows.yaml` already does) and confirm the compiler-family branches in `CMakeLists.txt` key off `MSVC`/`_MSC_VER` rather than `CMAKE_CXX_COMPILER_ID STREQUAL "Clang"` alone.
- Pass: both configure+build combinations succeed with exit code 0.

**Deferred:** the CMake min-version part is inherently exercised by every `cmake` invocation already
(bumped per merged PR #101); the clang-cl compiler-family-branching part is already continuously
verified by GitHub Actions' `os_windows.yaml` clang-cl/Ninja job. No clang-cl toolchain is installed
on this machine (checked the VS2022 install tree and `Program Files\LLVM`) to add or verify a
meaningful new local test. Not implemented in this pass.

## #103 — Doesn't build using clang-cl on windows due to empty definition of __extention__

https://github.com/lexxmark/winflexbison/issues/103

Root `CMakeLists.txt` unconditionally does `add_compile_definitions("__extension__")` for MSVC, but
clang-cl also reports as MSVC-compatible and picks up this define — which then breaks `<mmintrin.h>`
(`__extension__ (__m64)(__v2si){...}`) because clang-cl needs `__extension__` to keep its real GNU-extension meaning, not an empty macro.

**Proposed test case:**
- Build: configure and build `common/misc/bitset/table.c` (or any TU that transitively includes `<intrin.h>`/`<mmintrin.h>`) using `clang-cl` per the existing GitHub Actions Ninja/clang-cl workflow.
- Pass: compiles without the `unexpected type name '__m64'`/`'__v2si'` errors — confirms the `__extension__` compile definition is scoped to exclude clang-cl (e.g. gated on `NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang"`, matching the reporter's proposed patch).

**Deferred:** already regression-guarded by CI's clang-cl build job (added specifically after this
fix landed, PR #105); no clang-cl toolchain available locally to add or verify a meaningful new
local test. Not implemented in this pass.
