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
- **The pass/fail logic is baked into the generated scanner by m4**: the catch-all rule emits
  `fprintf(stderr,"TEST FAILED…");exit(1);`, and clean EOF prints `TEST RETURNING OK.` and returns
  0. Exit `77` = skipped.
- **Ruleset tests** (`*.rules`) are backend-independent rulesets expanded — per backend
  (`nr`/`r`/`c99`) and table option — into concrete `<stem>_<backend>.l` + `.txt` by
  `ruleset.sh` → `testmaker.sh` → `testmaker.m4`. **This generation requires `m4` + `sh`.**

File conventions: `.l`/`.ll`/`.lll` flex input; `.txt` runtime input; `.rules` backend-independent
ruleset (input after a `###` marker); `.direct` runs from srcdir; `.cn` option-conformance script;
`.tables` serialized DFA for `.ser`/`.ver`.

## Windows design — FLEX: pre-generate & commit, run under CTest

**Chosen approach: no `sh`/`m4` at test time.** Do the ruleset generation **once** on a machine
that has `sh` + `m4`, commit the resulting concrete `.l`/`.txt` (and, where practical, the
flex-generated `.c`), so the on-Windows test run needs only **MSVC + CTest**.

Rationale vs. the alternative (invoke `ruleset.sh`/`testmaker`/m4 live at build time): live
generation tracks upstream with zero committed artifacts but forces every builder/CI to have
`sh`+`m4` on PATH. Pre-generating trades a set of committed generated files for a dependency-free,
deterministic `ctest` run — the better fit for an MSVC-centric project.

### Components to build (future execution)

1. **Import** the upstream suite from `orig/flex/tests` into `winflexbison/tests/flex`, keeping the
   upstream sources (`.l`, `.rules`, `.txt`, `.direct`, the `*.sh`/`testmaker.m4` machinery)
   separate from anything generated.

2. **One-time generator step** (run under Git-Bash with `m4`):
   - Invoke the imported `ruleset.sh` + `testmaker.sh` + `testmaker.m4` to emit every
     `<stem>_<backend>[_<tableopt>].l` and its `.txt`.
   - Commit those under a dedicated generated subdir (e.g. `tests/flex/generated/`) so a refresh
     can wipe and regenerate cleanly.
   - Capture the exact command in a reproducible script (e.g. `tests/flex/pregenerate.sh`) for the
     next upgrade.

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

## Windows design — BISON: source & adapt the autotest suite

Bison tests are sourced from `orig/bison/tests` (29 `.at` files + `testsuite.at`).

- Bison uses **GNU Autotest**: `.at` files are m4 macros compiled by `autom4te` into a single
  `testsuite` shell script, run as `./testsuite`. This is heavier and more POSIX-coupled than
  flex's exit-code model (it expects `sh`, `m4`, `diff`, a working `bison`, and per-language
  toolchains for C/C++/Java/D tests), so it cannot be pre-flattened the way flex is.
- Windows adaptation options to evaluate when this is executed:
  1. Run the generated `testsuite` under Git-Bash/MSYS against the built `win_bison.exe` — closest
     to upstream; requires the POSIX toolset at test time (a deliberate exception to the flex
     "MSVC-only at test time" stance, because bison autotest is impractical to flatten).
  2. Select a **toolchain-light subset** of `.at` groups (e.g. `input.at`, `output.at`,
     `reduce.at`, `conflicts.at`) and drive them via CTest, skipping Java/D/`c++` groups.
- **This pass documents the sourcing and options only**; choosing and implementing the bison
  harness is future work.

## Windows design — our own tests (author, don't import)

The upstream flex/bison suites prove the **generators still behave like upstream**, but they say
nothing about the parts winflexbison actually adds and maintains. Those are exactly the parts most
likely to regress across an upgrade or an MSVC change, and upstream has no test for them because
upstream doesn't have them. We therefore also need a **native winflexbison test set that we
author and own**, separate from the imported suites (e.g. under `tests/winflexbison/`).

Areas to cover (future execution — this pass only records the intent):
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
- Flex: re-import `orig/flex/tests` for the new version into `tests/flex` (preserving the Windows
  CMake/launcher/`pregenerate.sh` files), then re-run `pregenerate.sh`.
- Bison: re-import the chosen `orig/bison/tests` `.at` subset for the new version.
- Always import against the **new** baseline tag, never a branch tip
  ([02](../02-baseline-mirrors/spec.md)).
