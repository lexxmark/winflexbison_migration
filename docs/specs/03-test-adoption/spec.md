# 03 — Test Adoption

**Task:** Analyze the upstream flex and bison test suites and define how to bring them into
winflexbison and run them on Windows, **plus define a native winflexbison test set we author
ourselves** for the port-specific behavior upstream doesn't cover, so an upgrade can be proven
green before merge.

## Current state

**winflexbison has no vendored tests.** `winflexbison/tests/` is empty and there is no CTest
wiring anywhere (no `enable_testing()`/`add_test()` in any `CMakeLists.txt`, no
`add_subdirectory(tests)`). Test adoption is therefore a **greenfield import** from the `orig/`
baselines — there is nothing pre-existing to preserve or refresh.

Source material lives in the baselines ([02](../02-baseline-mirrors/spec.md)):
- `orig/flex/tests` — the full upstream flex suite (~39 `.l` inputs + `.rules`/`.txt`/`.direct`
  fixtures and the automake/m4 harness).
- `orig/bison/tests` — 29 `.at` autotest files + `testsuite.at`.

## How the upstream FLEX suite works (why it ports easily)

The key property: **pass/fail is the scanner's exit code, not golden-output diffing.**

- Automake drives it via `TESTS`/`check_PROGRAMS` in `tests/Makefile.am`, with per-extension
  `LOG_COMPILER` wrappers (`testwrapper.sh`, `testwrapper-direct.sh`, `options.cn`).
- A scanner is generated from `.l`/`.ll`/`.lll` by the just-built flex (`.ll` ⇒ `-+` C++;
  `.lll` ⇒ C output compiled as C++), compiled, then run with the matching `.txt` piped in.
- **The pass/fail logic is baked into the test scanner's rules**: the catch-all rule calls
  `exit(1)` (or the `main` returns non-zero) on unexpected input, and a clean run returns 0 (most
  also print `TEST RETURNING OK.`). Exit `77` = skipped. Upstream keys pass/fail on the **exit
  code**, not golden-output diffing.
- **`tableopts` tests** are the one generated family in v2.6.4: `tableopts.sh` emits a
  `tableopts.am` fragment enumerating one scanner per table-option (`-Ca -Ce -Cf …`) × backend
  (`nr`/`r`) × mode (`opt`/`ser`/`ver`), all from the single template `tableopts.l4`. Generating
  the *list* needs `sh`; generating each *scanner* is a normal `win_flex` invocation.

> **Note (verified against `orig/flex` @ v2.6.4):** this baseline does **not** use the
> `.rules`/`ruleset.sh`/`testmaker.m4` machinery found on newer flex `master`. Re-verify the exact
> mechanism against the target baseline before adopting a newer flex — the file conventions below
> are v2.6.4's.

File conventions (v2.6.4): `.l` flex input, `.ll` C++ (`-+`) input, `.lll` C output compiled as
C++; `.txt` runtime input; `.l4` m4/flex template shared by the `reject`/`table`/`tableopts`
families (generated with `--unsafe-no-m4-sect3-escape`); `.direct` runs from srcdir; `.cn`
option-conformance script; `.tables` serialized DFA for `.ser`/`.ver`.

## Windows design — FLEX: generate scanners at build time, run under CTest

**Chosen approach (revised during phase-1 implementation): generate each test scanner at build
time with `win_flex`; commit no generated `.c`.** Verified that `win_flex.exe` produces a scanner
with **no `m4` on PATH** (winflexbison's flex bundles its m4 stage), so the earlier worry that
drove "pre-generate & commit the `.c`" does not apply to the per-scanner step. A CMake
`add_custom_command` runs `win_flex --wincompat -o <name>.c <name>.l`; `--wincompat` is the
winflexbison-recommended flag that maps `<unistd.h>`→`<io.h>` and `isatty`/`fileno`→`_isatty`/
`_fileno`, so the suite also exercises that Windows code path. The generated `.c` lives in the
build tree only.

The residual `sh` dependency is narrow: only expanding the **`tableopts` list** needs `sh`. When
that family is adopted, pre-generate just the list (or hand-enumerate the ~22 option combos in
CMake) — not the scanners. So the on-Windows run needs only **MSVC + CTest**.

### Status — flex suite complete (112/112)

`winflexbison/tests/` now holds a working CTest harness (`tests/CMakeLists.txt` →
`tests/flex/CMakeLists.txt` + `run_scanner_test.cmake`), wired from the root via
`enable_testing()` + `add_subdirectory(tests)` (top-level builds only). Phase 1 covers the **19
single-file, C, exit-code "simple" tests** (`basic_*`, `array_*`, `mem_*`, `string_*`, `debug_*`,
`prefix_*`, `ccl`, `extended`, `quotes`, `quote_in_comment`, `posix`, `yyextra`, `alloc_extra`) —
all passing under VS2022 x64 Release. The launcher redirects the `.txt` on stdin via
`execute_process(INPUT_FILE …)` and honours `SKIP_RETURN_CODE 77`.

Phase 2 adds the **5 multi-file C tests** (`header_nr`, `header_r`, `top`,
`multiple_scanners_nr`, `multiple_scanners_r`) via an `add_flex_multifile_test()` helper: each
scanner `.l` carries `%option header="<stem>.h"`, so `win_flex` emits `<stem>.c` + `<stem>.h` into
the build dir (run with `WORKING_DIRECTORY` = build dir so the bare header name lands there), and
the generated `.c`(s) link with the hand-written `*_main.c`. **Suite is 24/24 green.**

Phase 3 adds the **5 C++ tests** (`cxx_basic`, `cxx_restart`, `c_cxx_nr`, `c_cxx_r`,
`cxx_multiple_scanners`) via `add_flex_cxx_test()`: `.ll` → `win_flex -+` (a C++ `yyFlexLexer`
scanner), `.lll` → `win_flex` without `-+` (C output compiled as C++), both `--wincompat`.
`enable_language(CXX)` is scoped to the test tree so the C-only product build is untouched. Two
MSVC-specific gotchas when compiling flex output as **C++** (both diagnosed and worked around in
the test CMake, not the product):
  1. **Don't link `winflexbison_common`** — its PUBLIC include of `common/m4/lib` brings gnulib's
     replacement `<stdlib.h>`/`<stdint.h>`, illegal in C++. Use `common/misc` (for `config.h`) and
     `flex/src` (for `<FlexLexer.h>`) directly.
  2. **Undefine the root's global `-Dinline=__inline` / `-Drestrict=__restrict`** per C++ target
     (`/Uinline /Urestrict`) — in C++ they macroize keywords and break `<xkeycheck.h>` and
     `__declspec(restrict)` in the CRT.
**Suite is 29/29 green.**

Phase 4 adds the **4 reject/table tests** (`reject_nr`, `reject_r`, `reject_ver`, `reject_ser`)
via `add_flex_generated_test()`: all generated from one `reject.l4` template with
`--unsafe-no-m4-sect3-escape` (so `win_flex` expands its section-3 m4 macros) plus per-variant
flags/defines. `reject_nr/r` read `reject.txt` on stdin; `reject_ver/ser` use **external
serialized DFA tables** — `win_flex` emits a `.tables` file and the scanner takes `<tables>
<input>` as **argv**, so `run_scanner_test.cmake` gained `ARG1`/`ARG2` positional-arg support.
  - **Windows gap found:** flex's serialized-tables output unconditionally `#include`s
    `<netinet/in.h>` (for `ntohs`/`ntohl`), which MSVC lacks — so `win_flex`'s serialized-tables
    output is **not self-compilable on MSVC** without a substitute. Worked around with a test-only
    shim (`tests/flex/shim/netinet/in.h`) providing the four byte-order helpers via MSVC
    intrinsics; the product skeleton is untouched. Candidate for a real port fix (guard the include
    or ship a Windows `htonl`/`ntohl`), tracked in [04](../04-port-change-catalog/spec.md).
**Suite is 33/33 green.**

Phase 5 finalizes the flex suite:
- **direct (5)** — `include_by_buffer`/`push`/`reentrant`, `rescan_nr`/`r`: run from the cases dir
  with the input as `argv[1]` so the scanner's relative include-file opens resolve
  (`run_scanner_test.cmake` gained `WORKDIR` + `ARG3`).
- **lineno comparison-mode (3)** — `lineno_nr`/`r`/`trailing` via `run_compare_test.cmake`: run the
  scanner twice on the same stdin and require flex's `yylineno` output to equal the reference
  newline count.
- **posixly_correct** — generated with `POSIXLY_CORRECT=1` in the flex env.
- **cxx_yywrap** — C++ scanner whose `main` takes input file(s) as `argv`.
- **bison (3)** — `bison_nr`/`yylloc`/`yylval`: link a `win_bison`-generated parser + a
  `win_flex`-generated scanner + a hand-written `*_main.c`; registered only when the `win_bison`
  target exists. Exercises win_bison and win_flex together.
- **tableopts (66)** — `-Ca … -Caem` × `{nr,r}` × `{opt,ser,ver}`, enumerated directly in CMake
  (no `sh`; reuses `add_flex_generated_test`). `ser`/`ver` feed input on stdin (`STDIN_INPUT`) so
  the reentrant main's default-stdin `yyin` path works. Case-safe labels avoid the `-Cf`/`-CF` and
  `-Caef`/`-CaeF` filename collisions on Windows.

**Intentionally excluded on MSVC:** `pthread.pthread` (needs a POSIX pthreads lib) and `options.cn`
(a `sh` option-conformance script, not a scanner).

**The flex suite is complete: 112/112 green** under VS2022 x64 Release. The one product-level gap
surfaced is the serialized-tables `<netinet/in.h>` include (see Phase 4), worked around with a test
shim and tracked as a candidate port fix in [04](../04-port-change-catalog/spec.md).

### Components to build (future execution)

1. **Import** the upstream sources from `orig/flex/tests` into `winflexbison/tests/flex/cases`
   (`.l`/`.ll`/`.lll`, `.txt`, `.direct`, `.l4` and, for the generated families, the `*.sh`
   machinery) — done for the phase-1 subset.

2. **`tableopts` list expansion (only if that family is adopted)** — the sole `sh`-dependent step.
   Either pre-generate the `tableopts.am` list once under Git-Bash and commit it, or hand-enumerate
   the ~22 `-Ca/-Ce/-Cf/…` × `nr/r` combinations directly in CMake. The per-scanner step is still
   a plain `win_flex` call; there is nothing to pre-generate for the non-`tableopts` tests.

3. **`tests/CMakeLists.txt` + CTest mapping** — `enable_testing()` at the root,
   `add_subdirectory(tests)`, and for each test case:
   - Run `win_flex` (`-+` for `.ll`; treat `.lll` as C compiled as C++) to produce `<name>.c` at
     build time, OR consume the committed pre-generated `.c`.
   - Compile with the CMake-detected C/C++ compiler (**not** a hardcoded MSVC path) adding
     `-I ${flex_src}` so generated scanners find `config.h`/`FlexLexer.h`.
   - `add_test(NAME <name> COMMAND <exe>)` with the `.txt` piped as stdin (a small CTest launcher
     handles `< input` redirection on Windows).
   - Pass criteria: `set_tests_properties(<name> PROPERTIES PASS_REGULAR_EXPRESSION "TEST RETURNING OK" SKIP_RETURN_CODE 77)` (or an exit-code check).

4. **Special/multi-file tests** to handle explicitly:
   - `bison_nr` / `bison_yylloc` / `bison_yylval` — need `win_bison` to generate the parser; wire
     it in or mark skipped (`77`) when unavailable (mirror upstream's `HAVE_BISON` fallback to
     `no_bison_stub.c`).
   - `multiple_scanners_*`, `cxx_multiple_scanners` — link several objects into one exe.
   - `pthread.pthread` — needs a pthreads lib; likely **skip on MSVC**.
   - `options.cn`, `test-yydecl-*.sh` — shell-based option checks; port to CTest or skip.

## Windows design — BISON: scoping the autotest suite

Bison tests are sourced from `orig/bison/tests` (@ `v3.8.2`). Unlike flex's exit-code model, this
is a full **GNU Autotest** suite and is much larger and more POSIX-coupled.

### Inventory (measured against the v2.8.2/3.8.2 baseline)

- `testsuite.at` `m4_include`s **27** `.at` files (plus `local.at` = the macro library, and
  `testsuite.h`). Total ≈ **287 `AT_SETUP` groups**; parameterized files (esp. `calc.at`, and the
  `c++`/`glr` families that loop over variants) expand to **~600+ actual test groups** in the
  generated `testsuite`.
- **Build:** `autom4te --language=autotest testsuite.at -o testsuite` compiles the `.at` m4 into
  one ~MB POSIX shell script. Needs `autom4te` (autoconf) + `m4` + `perl`.
- **Run:** `sh testsuite -C tests` — needs `atconfig` + `atlocal` (normally produced by bison's
  `configure`; they fill in `CC`, `CXX` with the `CXX98…CXX2B` flag sets, `DC` for D,
  `CONF_JAVAC`/`CONF_JAVA` for Java, `EXEEXT`, warning flags, `BISON`), plus `sh`, `diff`, `sed`,
  `grep`. A group whose toolchain var is empty **skips** itself (`BISON_CXX_WORKS=false`, etc.).
- `bison` is invoked as a drop-in — `win_bison.exe` matches the CLI, and needs its `data/`
  skeletons discoverable (next to the exe, or `BISON_PKGDATADIR`).

### Toolchain profile per file (how portable each group is)

Measured by `AT_BISON_CHECK` (run bison, check output — no compiler) vs `AT_COMPILE`/
`AT_PARSER_CHECK` (compile & run a generated parser) vs Java/D:

| Tier | Files | Needs |
|---|---|---|
| **Toolchain-FREE** (run `bison`, diff diagnostics/report/`.output`/sets — no compiler) | `diagnostics`, `report`, `sets`, `counterexample`, `m4`, `existing`, `named-refs`, most of `input` (67 groups, ~diagnostics), most of `conflicts` (44), `reduce`, `skeletons`, `output`, `synclines` | `win_bison` + `sh` + `diff` only |
| **C compiler** (compile+run generated C parser) | `actions`, `regression`, `torture`, `push`, `headers`, `types`, parts of `input`/`conflicts` | + `CC` |
| **C++ compiler** | `c++` (26 runs), `glr-regression` (43 runs, C++), `cxx-type` | + `CXX` (multiple `-std` levels) |
| **Java** | `java` (111), `javapush` (14), `calc` (37 Java variants) | + `javac`/`java` |
| **D** | `d`, one `scanner`/`calc` variant | + `DC` (gdc/ldc) |

The toolchain-free tier is the high-value, high-portability core: it validates win_bison's grammar
analysis, conflict/counterexample reporting, diagnostics, `.output` reports, first/follow sets, and
generated-file naming — i.e. most of what the Windows port can actually break — **without any
compiler**.

### Windows-specific risks (must be validated, not assumed)

1. **Building `testsuite`** needs `autom4te`, absent on stock Windows. Mitigation: pre-generate the
   `testsuite` script once on a POSIX box and commit it (as the flex plan does for generated
   artifacts), regenerating on each bison upgrade.
2. **Output diffs from Windows-isms:** synclines/`#line` directives, error messages, and `.output`
   reports embed **file paths** — backslashes vs `/`, drive letters, and CRLF can make otherwise-
   passing groups mismatch their expected output. `synclines.at`, `headers.at`, `diagnostics.at`
   and any group that greps generated `#line` are the likely offenders; expect a per-group triage
   pass, not a clean 100%.
3. **`atconfig`/`atlocal` must be hand-authored** (no autoconf `configure` runs here) — set
   `BISON=win_bison`, `EXEEXT=.exe`, leave `CXX`/`DC`/`CONF_JAVAC` empty to auto-skip those tiers
   at first, and point `CC` at whatever compiler is chosen (see next).
4. **Which C/C++ compiler drives the compile tiers?** Autotest drives `$CC $CFLAGS` in a shell,
   which fits **MinGW gcc/g++** far more naturally than MSVC `cl` (arg syntax, `-o`, exit codes).
   Using MinGW validates *win_bison's output*, but not *MSVC-compilation of that output*. Testing
   the MSVC path is better served by the CTest-native "our own tests" below.

### Recommended architecture — hybrid (not one or the other)

Flattening bison autotest into CTest the way flex was flattened is **impractical** (the grammars +
expected outputs are embedded in autotest m4; extracting ~287 groups by hand loses fidelity and is
enormous). So:

- **B1 — CTest-native bison smoke/port tests (MSVC, default build).** A small hand-authored set
  under the "our own tests" umbrella: `win_bison --version`, generate a parser from a canned `.y`,
  **compile it with the detected MSVC** and run it, plus a few `--output`/`--defines`/`--graph`/
  `--report` and relocatable-`data/` checks. This is what proves the Windows/MSVC path and it is
  cheap. **Do this first.**
- **B2 — Full upstream autotest, opt-in, under Git-Bash + MinGW.** Commit a pre-generated
  `testsuite` + hand-authored `atconfig`/`atlocal`; run `sh testsuite` against `win_bison.exe` as a
  separate, non-default target (its own CMake option / CI job), gated on `sh`+`gcc` being present.
  Start by running only the **toolchain-free tier** (leave `CC`/`CXX`/`DC`/`javac` unset so the
  compile/Java/D tiers auto-skip), triage the path/CRLF diffs, then enable the C tier with MinGW.

This mirrors the project's split: the MSVC-native CTest run stays dependency-free and is the gate;
the faithful upstream autotest is an optional deeper check.

### Phased plan

1. **Scope & smoke (B1):** author the CTest-native bison smoke/port tests; wire a `bison/` subdir
   under `tests/` alongside `flex/`. Proves win_bison end-to-end on MSVC. *(next actionable step)*
2. **Autotest bring-up (B2a):** pre-generate + commit `testsuite`; hand-author `atconfig`/`atlocal`
   (toolchain-free only); get `sh testsuite` running the free tier under Git-Bash against
   win_bison; record the baseline pass/skip/xfail counts and triage path/CRLF diffs.
3. **C tier (B2b):** add MinGW `CC`; enable `actions`/`regression`/`push`/`headers`/`torture`.
4. **C++ tier (B2c):** add MinGW `CXX`; enable `c++`/`glr-regression`/`cxx-type`.
5. **Java/D (B2d, optional):** only if a `javac`/D toolchain is worth requiring; otherwise leave
   permanently skipped and documented.

### Status — bison harness bootstrapped (B1 + golden-diff proof)

`winflexbison/tests/bison/` now implements the hybrid, with the everyday Windows run staying
CTest-native and WSL-free:

- **compile-run (exit-code, MSVC)** — `cases/calc.y` is a self-contained calculator (integrated
  scanner + `main`); `win_bison` generates the parser, MSVC compiles it, the exe runs with input on
  stdin, pass == exit 0 (`run_parser_test.cmake`). Proves win_bison end-to-end on Windows.
- **golden-diff (Windows-native)** — `win_bison` runs on each `golden-cases/*.y` and its
  stdout/stderr/exit are compared against `golden/` captured from **reference bison 3.8.2**
  (`run_bison_test.cmake`). Golden is regenerated under WSL by `generate.sh` and committed; a
  checkout without golden simply registers no golden tests.
- **Portability tricks that made golden stable:** invoke by bare filename from the cases dir (stable
  `name.y:line` paths); normalize the actual output for Windows-isms — `CRLF→LF`,
  `win_bison[.exe]:→bison:`, `\→/`, and fold locale-dependent fancy quotes (`‘’“”`) to ASCII; and
  run with `-fno-caret` to compare the semantic diagnostic rather than the caret rendering.
- **Fixtures:** `clean` (silent happy path), `sr_conflict` (S/R conflict count), `undef`
  (undefined-symbol error, exit 1), `useless` (useless nonterminal/rule warnings).

**Port finding (candidate fix for [04](../04-port-change-catalog/spec.md)):** win_bison **omits the
echoed source line in caret diagnostics on Windows** — it prints the `N |` prefix and the `^~~~`
markers but a blank source line, even for LF-only input. The golden suite sidesteps this with
`-fno-caret`; the underlying caret source-echo path is worth a dedicated port fix (win_bison cannot
re-read/echo the grammar line on Windows).

Full suite (flex + bison) is **117/117 green** under VS2022 x64 Release.

The WSL side (`generate.sh` + reference bison 3.8.2) is proven. Growing `golden-cases/` (more
`input`/`conflicts`/`output`/`report` grammars) is incremental.

### Status — B2 faithful autotest complete (`tests/bison-autotest/`)

The full upstream suite runs under WSL against `win_bison.exe`, driven by `run.sh`:

- `at/` vendors the version-matched `.at` sources (29 files + `testsuite.h`) + a hand-authored
  `package.m4`; `run.sh` compiles them with `autom4te` into the **776-group** `testsuite` (produced
  on demand, not committed). `install-wsl-deps.sh` installs the WSL prerequisites.
- The `bison` wrapper is a plain `exec` — no post-processing — because it forwards
  `BISON_PROGRAM_NAME=bison` and `WINFLEXBISON_BINARY_OUTPUT=Y` (plus the env the tests set:
  `COLUMNS`, `YYFLAT`, `POSIXLY_CORRECT`, …) to the Windows process via `WSLENV`. `atconfig`/`atlocal`
  are hand-authored; `run.sh` auto-detects `gcc`/`g++` to enable the C/C++ tiers (`GREP`/`EGREP`/
  `PERL`/… defined; `CPPFLAGS=-I` for `testsuite.h`). `@tb@` (a test token = literal TAB) is
  substituted in the generated `testsuite`.

**Result: 696 groups run, 0 unexpected failures, ~80 skipped** (Java + D only, no `javac`/D compiler).
A handful of environment/limitation cases are documented xfails (NTFS-illegal filenames; a couple of
win_bison m4 skeleton-complaint and byte-escaping diffs). Getting here surfaced and fixed **five real
win_bison bugs** (caret binary read, `xfopen` binary output, `b4_cat` `_m4eof` leak, `/utf-8`
glyphs, `--fixit` backup) — all cataloged in [04](../04-port-change-catalog/spec.md).

**Runner split:** the Windows `ctest` gate stays dependency-free; the 696-test autotest is the
deeper, WSL-only engine. It can also be driven from Windows: `runtests.bat --with-autotest`, or as
one aggregate ctest test with `cmake -DWFB_WSL_AUTOTEST=ON` (the `bison.autotest.wsl` test).

## Windows design — our own tests (author, don't import)

The upstream flex/bison suites prove the **generators still behave like upstream**, but they say
nothing about the parts winflexbison actually adds and maintains. Those are exactly the parts most
likely to regress across an upgrade or an MSVC change, and upstream has no test for them because
upstream doesn't have them. We therefore also need a **native winflexbison test set that we
author and own**, separate from the imported suites (e.g. under `tests/winflexbison/`).

**Started (`tests/winflexbison/`):** a self-verifying **parallel-invocation resistance** test
(`run_parallel.ps1`) — N concurrent win_flex/win_bison over the same grammar, sharing `%TEMP%`, must
all produce output identical to a single-call reference, actually overlap (measured peak concurrency,
else fail), and leave no temp files behind. This guards the port's per-process-unique temp names
(`pid_tempname`) and delete-on-close temp files.

Areas still to cover:
- **Port-specific behavior** catalogued in [04](../04-port-change-catalog/spec.md): the
  `app_path`/relocatable data-dir lookup (`win_bison` finding its `data/` skeletons next to the
  exe), `config.h` substitutions, the `filter.c`/`output.c` Windows patches, and the flex
  `#ifdef` guards — each should have a test that fails if the patch is dropped during a replay.
- **CLI/packaging smoke tests**: `win_flex --version` / `win_bison --version` report the expected
  vendored version; a trivial `.l` and `.y` round-trip (generate → compile with the detected MSVC
  → run) succeeds from a clean packaged layout, catching missing skeleton/data files in the zip.
- **Windows path & I/O edge cases** that upstream POSIX tests never hit: paths with spaces or
  backslashes, CRLF inputs, drive-letter/relative output paths, `--outfile` into another dir.
- **Regression pins** for bugs fixed in this port over time, so a future upgrade can't silently
  reintroduce them.

These live alongside the imported suites under the same CTest wiring
([section above](#windows-design--flex-pre-generate--commit-run-under-ctest)) but are **not**
refreshed from `orig/` on upgrade — they are ours to keep and extend. When [04]'s catalog gains a
new port change, add the matching test here.

## Refreshing / re-importing the test set on upgrade

Because nothing is vendored yet, each upgrade **imports fresh from the new baseline** rather than
merging over an old copy:
- Flex: re-import `orig/flex/tests` for the new version into `tests/flex/cases` (preserving the
  Windows `CMakeLists.txt`/`run_scanner_test.cmake`), and re-verify the suite mechanism against the
  new baseline (newer flex uses different generation machinery — see the note above).
- Bison: re-import the chosen `orig/bison/tests` `.at` subset for the new version.
- Always import against the **new** baseline tag, never a branch tip
  ([02](../02-baseline-mirrors/spec.md)).
