# Build Warnings — Review and Cleanup Plan

## Purpose

Get the MSVC build to zero warnings without turning the vendored flex/bison/m4/gnulib sources
into a merge-conflict field at the next upstream upgrade.

Status of this document: **in progress**, config-first order (4 → 5 → 1 → 2 → 3 → 6).
Last worked 2026-08-27. See [Where we stopped](#where-we-stopped) to resume.

| Phase | State |
|---|---|
| 4 — vendored target suppression | **done**, `c3a75a4` |
| — | *(unplanned)* **done**, `c88acd3` — `flex.skl`/`skel.c` drift repair, see [Findings](#findings-made-during-the-work) |
| 5 — test target configuration | 3 of 4 items **done**, `41465d6`; `C4005` held — reopened as a product bug (#29) |
| 1 — scope the C-only defines | **done**, `a4360ac`; carried a CMake minimum bump with it, see [Findings](#findings-made-during-the-work) |
| 2 — port-owned defects | **done**; 5 `const` fixes plus the `lalr.c` format fix |
| 3, 6 | not started |

Current count: **x64 302 → 56**, Win32 182 → 56, ctest 137/137. Both architectures are now equal:
everything left is `C4005` (54, held for #29) and `C4307` (2, Phase 3).

Everything so far was build configuration except Phase 2, which changed six source lines. Only one
is user-visible, and only in `--trace=automaton` output.

## Where the numbers come from

AppVeyor build 21 (`94ea54db`, "bison: make YYPTRDIFF_T pointer-wide on MSVC (#95)"), all 9 jobs
green. The 8 build cells were downloaded and de-duplicated; no local build was run, so these are
exactly what CI emits today.

| Cell | warning lines |
|---|---|
| VS2022 / x64 / Release | 291 |
| VS2022 / x64 / Debug | 291 |
| VS2022 / Win32 / Release | 171 |
| VS2022 / Win32 / Debug | 171 |
| VS2019 cells | same within ±6 |

x64 and Win32 differ by design: `C4267` (size_t → smaller) only fires on LLP64, and `C4018`
(signed/unsigned compare) fires ~3× more on Win32, where `size_t` is 32-bit and therefore
comparable with `int` in more places. **Both must be checked** — a change that silences x64 can
leave Win32 untouched, and vice versa.

Compilation happens at `/W3` (CMake's MSVC default for C; explicitly set for C++ in the root
`CMakeLists.txt`). No `/wd` suppression exists anywhere in the tree today.

## What the build actually emits

VS2022 / x64 / Release, by warning code, with the owning target:

| Code | x64 | Win32 | Meaning | Where it lives |
|---|---:|---:|---|---|
| C4267 | 122 | 0 | `size_t` → `int`/`symbol_number`/… | `win_bison` 79, `winflexbison_common` 42 |
| C4005 | 45 | 45 | macro redefinition (`INT8_MIN`…) | 4 C++ flex **test** targets |
| C4244 | 38 | 17 | `__int64`/`long`/`ptrdiff_t` → narrower | `win_bison` 27, `winflexbison_common` 11 |
| C4018 | 19 | 58 | signed/unsigned mismatch | `winflexbison_common` 17/31, `win_bison` 2/27 |
| D9025 | 16 | 16 | `/Uinline` overriding `/Dinline=__inline` | 8 **test** targets |
| C4311 | 14 | 0 | `void *` → `long` truncation | `flextest_mem_r`, `flextest_mem_nr` |
| C4065 | 13 | 13 | `switch` with `default` but no `case` | `bisontest_many_tokens` (generated) |
| C4308 | 5 | 5 | negative constant → unsigned | `win_bison` 2, `winflexbison_common` 3 |
| C4146 | 5 | 5 | unary minus on unsigned | `winflexbison_common` |
| C4090 | 5 | 5 | different `const` qualifiers | `win_bison` 3, `win_flex` 2 |
| *(bison)* | 4 | 4 | `%pure-parser` deprecated, "fix-its can be applied" | 4 flex test grammars |
| C4477 | 2 | 0 | `printf` format vs argument type | `win_bison` (`lalr.c:152`) |
| C4307 | 2 | 2 | signed integral constant overflow | `win_bison` |
| C4116 | 1 | 1 | unnamed type definition in parentheses | `winflexbison_common` (`obstack.c`) |

By target, x64 Release, counting distinct file:line sites rather than lines:

| Target | sites | Nature |
|---|---:|---|
| `win_bison` | 117 | vendored bison + a handful of port-added lines |
| `winflexbison_common` | 79 | vendored gnulib / GNU m4 — **all** upstream |
| `win_flex` | 2 | **both** port-added lines |
| test targets | 30 + D9025 | ours to configure |

Two facts that shape the whole plan:

1. **~177 of 198 product warnings are `C4267`/`C4244`/`C4018` in vendored code.** These are MSVC's
   equivalents of `-Wconversion` and `-Wsign-compare`, which GCC does *not* enable at `-Wall`.
   Upstream builds clean and has never seen them. Fixing them in place means editing roughly 120
   lines across 60 vendored files — every one of those lines becomes a conflict when we adopt the
   next flex/bison/m4 release. That is precisely the cost this project exists to avoid.
2. **20 of them are not even locatable.** `bison/src/scan-gram.c` and `scan-code.c` are
   pre-generated flex output carrying `#line` directives that point at
   `/Users/akim/src/gnu/bison/src/scan-gram.l` — the maintainer's laptop. MSVC reports the warning
   against that path. There is no file to annotate; only target-level or wrapper-level suppression
   can reach them.

## The rule

Every fix lands in the highest tier it can:

1. **Build system** (`CMakeLists.txt`) — zero upstream diff. Default for anything in vendored code.
2. **Port-owned code** — lines this port added (`pid_tempname` glue, `main_m4`, our test CMake).
   Fix properly; these are ours.
3. **Vendored upstream source** — only where the warning marks a *real* Windows-specific defect,
   and then as a minimal change recorded in `docs/specs/04-port-change-catalog/` and reported
   upstream.

Tier 1 is a deliberate trade: we lose `/W3` coverage over vendored code in exchange for a clean
diff. Phase 6 buys the coverage back as an on-demand audit rather than a permanent wall of noise.

---

## Phase 1 — Stop generating our own noise (D9025, 16 lines)

**Not upstream's fault, ours.** The root `CMakeLists.txt` does

```cmake
add_definitions(-Dinline=__inline)
add_definitions(-Drestrict=__restrict)
```

globally, because MSVC compiles C as C89 and the vendored C sources use both keywords. In C++
those macros are illegal (they collide with `<xkeycheck.h>` and `__declspec(restrict)`), so five
places in `tests/` undo them with `/Uinline /Urestrict` — and `cl` warns about each override, in
every C++ test target, twice.

Fix: make the definitions apply to C only, so nothing has to be undone.

```cmake
add_compile_definitions(
    $<$<COMPILE_LANGUAGE:C>:inline=__inline>
    $<$<COMPILE_LANGUAGE:C>:restrict=__restrict>)
```

then delete `/Uinline /Urestrict` from `tests/bison/CMakeLists.txt:138`,
`tests/flex/CMakeLists.txt:239`, `:308`, `:506` (keeping `/we4244` at `:308`), and update the
comments that explain the workaround.

**Caveat to verify before committing:** `$<COMPILE_LANGUAGE:...>` in `add_compile_definitions`
needs CMake ≥ 3.11 and, for the Visual Studio generator, only works because C and C++ never share
a `.vcxproj` here. If it misbehaves, the fallback is an `INTERFACE` target carrying the two
definitions, linked by the product targets and the C test targets. Either way this must be proven
on a real VS2022 *and* VS2019 configure, not reasoned about.

Removes: 16 lines per cell, and one long-standing wart.

**Outcome.** Done. Removed 18 per cell, not 16: the local tree has two more C++ test targets than
the CI cells the 16 came from. x64 81 → 63, Win32 79 → 61, ctest 137/137.

The caveat was real. Tested with a small throwaway project with three targets:

| Target | Gets the defines? | |
|---|---|---|
| C files only | yes | correct |
| C++ files only | no | correct |
| both .c and .cc | **no, not even the .c file** | **problem** |

Visual Studio uses one flag set per `.vcxproj`, so in a mixed target `COMPILE_LANGUAGE` is CXX and
the C files lose the defines. `set(CMAKE_C_FLAGS ...)` was tried too and fails the same way, so
this is a Visual Studio limit, not a wrong choice of command.

So this only works while no target has both `.c` and `.cc` files. Checked every generated
`.vcxproj`: none does. Keep it that way — a future mixed target would lose `inline`/`restrict` on
its C files with no error.

`add_compile_definitions` needs CMake 3.12, so the floor went 3.10 → **3.16** in all four
`CMakeLists.txt`. The 3.10 was already wrong: line 58 has used `add_compile_definitions` for a
while. The bump has a cost, see the policy finding below.

**VS2019 was not tested locally.** Only VS2022 is installed here; `cmake -G "Visual Studio 16
2019"` says "could not find any instance of Visual Studio". AppVeyor builds four VS2019 cells, so
CI covers it, but this is the one part proven by CI only.

## Phase 2 — Fix the port-owned defects (C4090 ×5, C4477 ×2)

Small, genuinely wrong, and all in code this port added.

**C4090 — `flex/src/filter.c:78` and `bison/src/output.c:817,835`.** `pid_tempname()` returns
`const char *`; the port assigns it to `char *p`. Change the three locals to `const char *`.
No upstream line moves — `pid_tempname` and both call sites are port additions.

**C4090 — `flex/src/filter.c:485` and `bison/src/output.c:842`.** `char const *argv[10]` passed to
`main_m4 (int argc, char *const *argv, …)`. `main_m4`'s signature is upstream m4's `main`
signature, so change the *call*, not the callee: `main_m4 (i - 1, (char *const *) argv, …)`.

**C4477 — `bison/src/lalr.c:152`.** This one is a real bug, not a nit:

```c
fprintf (stderr, "goto_map[%d (%s)] = %ld .. %ld\n",
         i, symbols[ntokens + i]->tag, goto_map[i], goto_map[i+1] - 1);
```

`goto_number` is `size_t` (`lalr.h:80`, identical upstream). On LP64 Linux `%ld` and `size_t` are
both 64-bit, so upstream never noticed; on MSVC x64 `long` is 32-bit, so `%ld` truncates.

Fix: `%zu` for both. Two characters, in vendored code, so it is a Tier-3 change: record it in the
port-change catalog and send it upstream — bison would want this fix on any LLP64 target.

**Correction, measured while doing the work.** An earlier draft of this section said the following
arguments in the call end up misaligned. They do not. The MSVC x64 varargs slot is 8 bytes and
`%ld` consumes the whole slot, so only the value itself is truncated:

```
old %ld: goto_map[1 (expr)] = 5 .. -1                    <- positions fine
new %zu: goto_map[1 (expr)] = 5 .. 18446744073709551615
old %ld big: 7  |  new %zu big: 4294967303               <- the actual bug
```

So it needs a value above 2^32 (over 4 billion gotos) to show. Real, but not reachable in
practice. See catalog 6e for the `-1` → `SIZE_MAX` display change this brings with it.

## Phase 3 — Use upstream's own suppression hook (C4307 ×2, C4308 ×2, some C4018)

`bison/src/system.h:85-94` defines `IGNORE_TYPE_LIMITS_BEGIN` / `_END` — upstream's marker for
"the overflow-check macros below intentionally compare against type limits, do not warn". It has a
GCC arm and an empty `#else`, so on MSVC the regions are unmarked and gnulib's `INT_ADD_WRAPV` /
`INT_MULTIPLY_WRAPV` expansions warn: `strversion.c:39,49,50,61`, `InadequacyList.c:38`,
`symtab.c:1083`.

Add the MSVC arm upstream left open:

```c
# elif defined _MSC_VER
#  define IGNORE_TYPE_LIMITS_BEGIN \
     __pragma (warning (push)) \
     __pragma (warning (disable : 4307 4308 4018 4146))
#  define IGNORE_TYPE_LIMITS_END __pragma (warning (pop))
```

~6 lines, in one file, at the exact spot upstream designed for it, silencing only the regions
upstream itself marked as intentional. This is the highest-value Tier-3 change in the plan and is
worth offering upstream verbatim.

Remaining `C4308`/`C4146` sites (`common/m4/freeze.c:312`, `eval.c:726,812`, `builtin.c:465`,
`m4/lib/malloc/dynarray_resize.c:45`, `misc/reallocarray.c:31`) sit outside any marked region and
fall to Phase 4.

## Phase 4 — Suppress the vendored mass at target level (C4267, C4244, C4018, C4146, C4308, C4116)

Add to the root `CMakeLists.txt`, applied to the three vendored targets only:

```cmake
# Vendored upstream (flex, bison, GNU m4, gnulib) is written for GCC, where none
# of these fire at -Wall. Silencing them per target keeps ~180 warnings out of the
# log without touching a single upstream line -- see docs/build-warnings-plan.md.
#   C4018  signed/unsigned mismatch          C4146  unary minus on unsigned
#   C4244  narrowing conversion              C4267  size_t -> smaller (x64 only)
#   C4308  negative constant -> unsigned     C4116  unnamed type in parentheses
set(WFB_VENDORED_WARNINGS /wd4018 /wd4146 /wd4244 /wd4267 /wd4308 /wd4116)
```

applied via `target_compile_options(<t> PRIVATE ${WFB_VENDORED_WARNINGS})` on
`winflexbison_common`, `win_flex`, `win_bison` (and `fl`, `y` for consistency).

Deliberately **not** applied to `tests/winflexbison/` — that is our own code and stays at full
`/W3`, so new port work still gets checked.

Two alternatives were considered and rejected:

- *Per-file `#pragma warning(disable)`* — reaches ~60 vendored files, i.e. 60 conflict points.
- *`#pragma` in the `scan-*-c.c` wrappers* — attractive for the 20 `/Users/akim/…` warnings, since
  those wrappers are three lines each, but they exist upstream too, and it only covers 20 of 180.

Keep `/wd4090` **out** of the list: after Phase 2 there are no `C4090` left, and leaving it live
means the next port change that drops a `const` gets caught.

## Phase 5 — Configure the test targets (C4005, C4065, C4311, bison deprecations)

All 30 remaining warnings come from test targets, and all four causes are ours to configure.

**C4005 ×54 — not test noise. This is issue #29, closed but never fixed.**

The first draft of this plan called for `/wd4005` on the C++ test targets. That was wrong. In a
generated C++ scanner:

```
line  56:  #if defined (__STDC_VERSION__) && __STDC_VERSION__ >= 199901L   <- false in C++
line  81:  #define INT8_MIN (-128)          ... and 8 more
line 118:  #include <iostream>              -> pulls <stdint.h> -> 9x C4005
```

MSVC never defines `__STDC_VERSION__` in C++ mode, so the non-C99 branch always runs. **Every
downstream user compiling a flex C++ scanner with MSVC gets these 9 warnings** — `flextest_cxx_basic`
is doing nothing unusual, it is the ordinary `win_flex -+` path. The macros reach generated
scanners because `flex.skl:234` does `m4preproc_include(\`flexint.h')`, which bakes the whole
header into `skel.c`.

The existing `flex.flexint_h_stdint` test pins `/we4005`, but only on a **C** scanner, where no
`<stdint.h>` ever follows — so it passes trivially and never covered the C++ path.

Upstream fixed this after 2.6.4: `westes/flex` master splits the typedefs into a new
`src/flexint_shared.h` (with an explicit `_MSC_VER >= 1600` arm) and leaves the limit macros only
in `flexint.h`, which the skeleton no longer includes — `src/Makefile.am` confirms the skeleton now
depends on `flexint_shared.h`. Generated scanners stop defining the macros at all. Note PR #309 is
*closed, not merged*; the change landed separately.

**Decision: backport that structure.** New `flex/src/flexint_shared.h`, `flexint.h` reduced to
limit macros plus an include of it, `flex.skl` including the shared header, `skel.c` regenerated,
plus a C++ regression test that pins `/we4005` the way the C one does, and a changelog entry.
This is a product fix, not a warnings cleanup — it closes #29 and matches what 2.6.5 will ship, so
the next flex upgrade is a no-op here rather than a conflict.

**C4065 ×13.** `many_tokens.cc/.hh`, generated by `win_bison` from our own grammar; the skeleton
emits `switch` blocks with only a `default:`. Add `/wd4065` to `bisontest_many_tokens`.

**C4311 ×14.** `tests/flex/cases/mem_nr.l` and `mem_r.l` print pointers as `(long)ptrs[i].p`. It is
a genuine 64-bit truncation, but only in the test's own diagnostic printf, and both files are
vendored verbatim from `upstream/flex/tests/`. **Decided: `/wd4311` on the two targets, sources
left untouched**, so `diff upstream/flex/tests` stays empty and the build is quiet. Note it in the
test README.

**4 bison deprecation lines.** `tests/flex/cases/bison_{nr,yylloc,yylval}_parser.y` use
`%pure-parser`; `win_bison` warns and then adds "fix-its can be applied. Rerun with `--update`".
Three of the four are vendored from upstream flex's test suite, so do not run `--update` on them —
pass `-Wno-deprecated -Wno-other` to `win_bison` in those test rules instead. (Our own
`core_yylex_wrapper_parser.y` may simply be updated.)

## Phase 6 — Keep the signal, then hold the line

Suppression without a way back is how real 64-bit bugs get buried — #95 was exactly that class of
bug. Two cheap counterweights:

- **An audit switch.** `option(WFB_VENDOR_WARNINGS "Re-enable warnings in vendored code" OFF)`
  that empties `WFB_VENDORED_WARNINGS`. One configure flag regenerates the full 291-line log for
  a periodic read, and it costs nothing when off.
- **`/WX` where we own the code.** Once the tree is clean, `tests/winflexbison/` and any future
  port-owned target can carry `/WX` so port code cannot regress. Do **not** put `/WX` on the
  vendored targets — one new upstream warning would then break the build on upgrade day.

Optional follow-up, not part of this plan: the GitHub Actions clang-cl workflow has been failing at
*configure* since before this work (`Windows-Clang.cmake:187` under CMake 4.4), so there is no
clang-cl warning data at all. Every number above is MSVC-only. `/wd` flags are accepted by
clang-cl, so the plan does not need splitting, but clang-cl's own diagnostics are an unknown until
that job is repaired.

---

## Expected end state

Against the **local** baseline (302 x64 / 182 Win32), which is 11 higher than CI's 291/171 because
commit `77978ed` added the `flextest_cxx_batch` target and every new C++ test target arrives
carrying 9 × `C4005` + 2 × `D9025`:

| Phase | x64 removed | Win32 removed |
|---|---:|---:|
| 4 — vendored target suppression ✅ | 190 | 86 |
| 5a — test config (`C4065`, `C4311`, deprecations) ✅ | 31 | 17 |
| 5b — the `C4005` product fix (#29) | 54 | 54 |
| 1 — scope the C-only defines ✅ | 18 | 18 |
| 2 — port-owned defects ✅ | 7 | 5 |
| 3 — `IGNORE_TYPE_LIMITS` MSVC arm | 2 | 2 |
| **Total** | **302 → 0** | **182 → 0** |

Phase 3 dropped from 4 to 2 because Phase 4's `/wd4308` already covers the two `strversion.c`
sites; only the two `C4307` remain, which is most of the argument for skipping Phase 3 entirely.

Upstream/vendored files touched: `flex/src/flexint.h`, a new `flex/src/flexint_shared.h`,
`flex/src/flex.skl` and the regenerated `skel.c` (the #29 fix, tracking upstream's own shape);
`bison/src/lalr.c` (one `%ld` → `%zu`); optionally one block in `bison/src/system.h`. Everything
else is CMake or port-owned code.

## Verification

1. Configure and build all four VS2022 cells (x64/Win32 × Release/Debug) plus VS2019/x64/Release,
   capture the logs, and assert `grep -c " warning "` is 0. x64 and Win32 must both be checked.
2. `runtests.bat` stays green — Phase 1 changes how *every* test target is compiled, and Phase 2
   changes the temp-file path in both tools, so the CTest gate is the real proof, not the build.
3. Diff `flex/src`, `bison/src`, `common/` against `upstream/` and confirm the only new hunks are
   the two named in "Expected end state".
4. Re-run with `-DWFB_VENDOR_WARNINGS=ON` once and keep that log as the baseline the audit is
   compared against later.

## Open decisions

1. **Phase 3 scope.** Now worth only 2 warnings (`C4307`), since Phase 4's `/wd4308` already covers
   the rest. It is proposed because it is *correct* — it marks intent where upstream marks intent —
   not because it is necessary. Drop it if the preference is zero discretionary upstream edits;
   `/wd4307` on `win_bison` gets the same log for no upstream diff.
2. ~~**`C4311` in the flex mem tests.**~~ Decided: suppress, sources untouched.
3. **Does the #29 fix belong in this work or its own?** It is now the largest item in Phase 5 and
   it is a product change with its own test and changelog entry, not a warnings cleanup. Landing it
   as a separate commit (or separate branch) keeps the warnings work reviewable.
4. **Release gating.** Does this go into 2.5.26, or after it? Phase 4 changes no shipped behavior,
   but the #29 fix changes what every generated scanner contains — that is a release-note item,
   and it argues for 2.5.26 rather than after.

---

## Findings made during the work

Things discovered while implementing that were not in the original plan. Recorded here so they
are not re-derived.

### `flex.skl` and `skel.c` had drifted — `--wincompat` lived only in the generated file

Found while verifying the skeleton regeneration pipeline before using it. Regenerating `skel.c`
from `flex.skl` with upstream's own `mkskel.sh` produced a file **9 lines short**: the
`M4_YY_WIN_COMPAT` block (`<io.h>`, `_isatty`, `_fileno`) appeared once in `skel.c` and zero times
in `flex.skl`. The option is wired up in `main.c:1288`, `options.c:202`, `options.h:129`, but the
skeleton half of the feature was only ever added to the generated file.

So regenerating the skeleton the upstream way **silently deleted `--wincompat`** — the port's
headline flex feature and the flag the whole flex test suite generates with. Nothing caught it,
because `flex/CMakeLists.txt` only compiles `skel.c`; no build step regenerates it.

Repaired in `c88acd3`: the block was ported into `flex.skl`, and the repair is proved by
`skel.c` being **unchanged** — the repaired source now reproduces the committed file byte for byte.

**Regeneration recipe** (needs MSYS2 for `m4` and `sh`; `mkskel.sh` is not vendored, it comes from
the baseline mirror):

```
C:\msys64\usr\bin\bash.exe -lc \
  "cd <repo>/winflexbison/flex/src && \
   sh <repo>/upstream/flex/src/mkskel.sh . m4 2.6.4 > /tmp/skel_regen.c"
cmp /tmp/skel_regen.c winflexbison/flex/src/skel.c   # must be identical before any change
```

Always run that `cmp` **before** editing the skeleton, and again after, so the only delta is the
intended one. `flex.skl` is otherwise byte-identical to upstream 2.6.4 except the one
`(int)yyin.gcount()` line from #73.

### `C4005` is issue #29 — closed, but never actually fixed

See Phase 5. Short version: every downstream user compiling a flex C++ scanner with MSVC gets 9
macro-redefinition warnings, because MSVC never defines `__STDC_VERSION__` in C++ mode. The
existing `flex.flexint_h_stdint` regression test pins `/we4005` but only on a **C** scanner, where
no `<stdint.h>` ever follows — so it passes trivially and never covered the C++ path. Upstream
fixed it after 2.6.4 by moving the typedefs into `src/flexint_shared.h` (with an `_MSC_VER >= 1600`
arm) and keeping the limit macros out of the skeleton.

Groundwork already done, so it does not need redoing:

- the skeleton body never references `INT8_MIN` & co., so dropping them from generated scanners is
  safe (`grep` of `flex.skl` for the limit macros: no hits);
- `westes/flex` PR #309 is **closed, not merged** — the change landed separately; read
  `src/flexint_shared.h` and `src/Makefile.am` on master, not the PR;
- upstream master has since restructured the skeleton heavily (`cpp-flex.skl`, `c99-flex.skl`,
  `go-flex.skl`, `skeletons.c`), so this is a targeted backport of the idea, not a file copy.

### Every new C++ test target costs 11 warnings

The local baseline (302) is 11 higher than CI's 291 purely because commit `77978ed` added the
`flextest_cxx_batch` target, which arrives carrying 9 × `C4005` + 2 × `D9025`. That is the
argument that Phase 1 and the #29 fix are structural rather than cosmetic.

### Raising the CMake minimum silently adopts two MSVC policies

Found while doing Phase 1. `cmake_minimum_required` does more than allow newer commands: it also
turns on every policy up to that version. Going 3.10 → 3.16 turns on **CMP0091** and **CMP0092**,
both added in 3.15. Both change MSVC flags:

| Policy | What NEW does | What that would break here |
|---|---|---|
| `CMP0091` | removes `/MD` from `CMAKE_<LANG>_FLAGS_<CONFIG>` | `USE_STATIC_RUNTIME` works by replacing `/MD` with `/MT` in those flags. With no `/MD` there, it does nothing. CI builds with `USE_STATIC_RUNTIME=ON`, so releases would link the dynamic CRT |
| `CMP0092` | removes `/W3` from `CMAKE_<LANG>_FLAGS` | C++ sets its own `/W3`, but C would drop to `/W1`. Most warnings this document tracks would disappear, and the phase would look better than it was |

Neither one fails the build. Measured by configuring the same tiny project at both floors:

```
FLOOR=3.10   C_FLAGS=[/DWIN32 /D_WINDOWS /W3]   C_FLAGS_RELEASE=[/MD /O2 /Ob2 /DNDEBUG]
FLOOR=3.16   C_FLAGS=[/DWIN32 /D_WINDOWS]       C_FLAGS_RELEASE=[/O2 /Ob2 /DNDEBUG]
```

Both are set to `OLD` before `project()`. That order matters: `project()` is where CMake fills in
these flags, so setting the policies after it does nothing. Checked after the change: `/W3` is
back in `CMAKE_C_FLAGS`, the C targets' `.vcxproj` say `<WarningLevel>Level3</WarningLevel>`, and
`-DUSE_STATIC_RUNTIME=ON` still gives `<RuntimeLibrary>MultiThreaded</RuntimeLibrary>`.

**Separate task, not this one:** move to the NEW behaviour properly — use
`CMAKE_MSVC_RUNTIME_LIBRARY` instead of replacing text, and set `/W3` directly. Also note the
replace loop never listed `CMAKE_C_FLAGS_MINSIZEREL` and `_RELWITHDEBINFO`, so those two configs
ignore `USE_STATIC_RUNTIME`. We do not ship them, so nothing is broken today.

### Phase 3 shrank

Phase 4's `/wd4308` on `win_bison` already covers the two `strversion.c` sites, so Phase 3 is now
worth only the 2 `C4307` warnings. `/wd4307` would get the same log for no upstream diff.

## Where we stopped

**Landed** (branch `dev`):

| Commit | Repo | What |
|---|---|---|
| `c3a75a4` | `winflexbison` | Phase 4 — per-target `/wd` list + `WFB_VENDOR_WARNINGS` switch |
| `c88acd3` | `winflexbison` | `flex.skl` drift repair + 2 changelog entries |
| `08178e8` | parent | this document + submodule bump (points at `c3a75a4`) |
| `41465d6` | `winflexbison` | Phase 5 — the three config items |
| `f94fd68` | parent | this document + submodule bump (points at `41465d6`) |
| *(uncommitted)* | `winflexbison` | Phase 1 + the CMake 3.16 bump — see below |

**Phase 5, three of four items done** (x64 112 → 81, Win32 96 → 79, ctest 137/137):

- `C4065` ×13 — `/wd4065` on `bisontest_many_tokens` in `tests/bison/CMakeLists.txt`.
- `C4311` ×14 — `/wd4311` on `flextest_mem_nr`/`flextest_mem_r`, sources untouched as decided.
- bison deprecations ×4 — `add_flex_bison_test` got a new `BISON_OPTIONS` parameter. The three
  grammars copied from `upstream/flex/tests/` are now generated with `-Wno-deprecated -Wno-other`.
  Our own `core_yylex_wrapper_parser.y` was changed to `%define api.pure` instead. Checked that
  win_bison writes the same bytes either way, so the issue #8 test still tests the same thing.

No changelog entry. Phase 4 got one because `WFB_VENDOR_WARNINGS` is an option users can set;
none of this is visible to users.

**Phase 1 done** (x64 81 → 63, Win32 79 → 61, ctest 137/137), `a4360ac`:

- `D9025` ×18 — the two defines are now C-only via `$<$<COMPILE_LANGUAGE:C>:…>`, and all four
  `/Uinline /Urestrict` workarounds are gone (`/we4244` kept on `flextest_cxx_batch`).
- `cmake_minimum_required` 3.10 → **3.16** in all four `CMakeLists.txt`, with `CMP0091` and
  `CMP0092` set to `OLD` before `project()`. See the two findings above.
- Changelog entry added, because the new CMake minimum does affect users.

**Phase 2 done** (x64 63 → 56, Win32 61 → 56, ctest 137/137), not committed yet:

- `C4090` ×5 — five locals made `const`, and the two `main_m4` calls now cast their `argv`. All
  five lines were added by this port, so they are fixed, not hidden.
- `C4477` ×2 — `lalr.c:152` `%ld` → `%zu`. Vendored code, so it is written up in port-change
  catalog 6e and should go upstream.
- Changelog entry for the trace fix only; the `const` changes are not visible to users.

**Loose ends:**

- The Phase 4 changelog entry went in with `c88acd3`, not with Phase 4 itself.
- VS2019 was never configured locally (not installed). CI is the only check for those four cells.
- `USE_STATIC_RUNTIME` ignores `MinSizeRel`/`RelWithDebInfo`. We do not ship them.
- `lalr.c` trace output now shows `SIZE_MAX` where upstream shows `-1`. See catalog 6e.
- Nothing in the test suite reads `--trace=automaton` output, so the `lalr.c` fix is uncovered.

**Remaining 56 on x64 and 56 on Win32:**

| Code | x64 | Win32 | Owner |
|---|---:|---:|---|
| `C4005` | 54 | 54 | Phase 5 — the #29 product fix, deliberately held |
| `C4307` | 2 | 2 | Phase 3 |

**Next step:** two things are left, and they are very different in size.

- **Phase 3** — 2 warnings, optional. Open decision 1 still stands: add the MSVC arm to
  `IGNORE_TYPE_LIMITS` (correct, but edits vendored code), or just use `/wd4307` on `win_bison`
  for the same result and no upstream diff.
- **#29 / `C4005`** — the other 54, and the only real work left. It is a product fix with its own
  test and changelog entry, not a warnings cleanup. It needs open decisions 3 and 4 answered
  first: does it land on its own branch, and does it go into 2.5.26 or after?

### Measurement loop

Warnings are only re-emitted for translation units that actually recompile, so **every
measurement needs a fresh build tree** — an incremental build silently under-reports.

```
rm -rf CMakeBuildWarn_x64
cmake -B CMakeBuildWarn_x64 -S . -G "Visual Studio 17 2022" -A x64
cmake --build CMakeBuildWarn_x64 --config Release   > build.log 2>&1
grep -c " warning " build.log
ctest --test-dir CMakeBuildWarn_x64 -C Release
```

Then the same with `-A Win32`. Both architectures must be checked: `C4267`, `C4311` and `C4477`
fire only on x64, while `C4018` fires ~3× more on Win32.

`CMakeBuild*/` is gitignored. Note that **the runtime output directory is hardcoded** to
`bin/Release` by a plain `set()` in the root `CMakeLists.txt`, so `-D` cannot redirect it and a
Win32 build leaves 32-bit binaries there — run Win32 first and x64 last so the tree is left 64-bit.
