# 04 — Port Change Catalog

**Task:** Catalog every category of change made to the vendored source because of the Windows/MSVC
port, so those changes can be faithfully **replayed** on a newer upstream. Also define how to
regenerate the authoritative diffs from the baselines.

The changes fall into **7 categories**. Categories 1–2 are bulk but **mechanical** (redo by
re-vendoring / regenerating). Categories 3–6 are the **hand-maintained patch set** — small,
localized, and the real work of an upgrade. Category 7 is the build system.

---

## 1. Bundled dependency vendoring *(mechanical, largest surface)*

Upstream bison/m4 pull gnulib and (for bison) an external m4 via autotools. winflexbison instead
**checks in** the code:

- `common/misc/` — a gnulib subset + libbison support (bitset, obstack, hash, quotearg, argmatch,
  xalloc, timevar, mbswidth, `bitset/`, `glthread/`, …). Maps to **bison-side gnulib**
  (`upstream/gnulib` @ `7818455…`).
- `common/m4/` — a **complete bundled GNU m4** (`m4.c`, `builtin.c`, `eval.c`, …) with its own
  gnulib in `common/m4/lib/` — maps to **m4-side gnulib** (`upstream/gnulib` @ `3639c57…`). Exists
  because bison needs an m4 processor at runtime and it is linked in-process (see category 5).

**Replay:** re-vendor from the new baselines rather than merging line-by-line. Preserve the two
build exclusions (they mirror upstream's own build):
- `common/CMakeLists.txt` excludes `m4/lib/regexec.c`, `regcomp.c`, `regex_internal.c`,
  `malloc/dynarray-skeleton.c` — these are `#include`-d into other TUs (gnulib amalgamation);
  compiling them standalone double-defines symbols.

## 2. Generated files committed *(mechanical)*

Upstream produces these at build time; winflexbison commits them so no flex/bison/sh/m4 is needed
to bootstrap:

| Tool | Committed generated file | Generated from |
|---|---|---|
| Flex | `flex/src/scan.c` | `scan.l` |
| Flex | `flex/src/parse.c`, `parse.h` | `parse.y` |
| Flex | `flex/src/skel.c` | `flex.skl` via `mkskel.sh` |
| Bison | `bison/src/parse-gram.c`, `parse-gram.h` | `parse-gram.y` |
| Bison | `bison/src/scan-gram.c`, `scan-code.c`, `scan-skel.c` | the `.l` files |

**Replay:** regenerate from the new upstream (with a bootstrap flex/bison/m4), then re-apply any
in-generated-file port touches (e.g. the flex `--wincompat` emission lands inside `scan.c`/`skel.c`
string tables — those follow automatically once category 4 is in the skeleton source).

**Bison build note (do not "fix"):** `bison/CMakeLists.txt` excludes `scan-code.c`, `scan-gram.c`,
`scan-skel.c` from direct compilation because each is `#include`-d by a thin wrapper TU
(`scan-code-c.c`, `scan-gram-c.c`, `scan-skel-c.c`). This is intentional and mirrors upstream.

## 3. Hand-written MSVC `config.h` ×2 *(hand-maintained)*

Replace autoconf-generated config. **These are the primary porting artifacts.**

- `common/misc/config.h` — the big one. `#include <io.h>`; defines `STDIN/STDOUT/STDERR_FILENO`,
  `ssize_t`; stubs the entire gnulib `_GL_ATTRIBUTE_*` family, `_Static_assert`, `_Noreturn`,
  `_GL_INLINE`/`_GL_EXTERN_INLINE`, `S_ISDIR`, etc. (the keyword/attribute shims gnulib normally
  derives from autoconf).
- `bison/src/config.h` — `#pragma once`; version/package strings, `ssize_t`→`ptrdiff_t`,
  `RENAME_OPEN_FILE_WORKS`, an `fopen`→`_fsopen(…,_SH_DENYNO)` redirect (Windows share semantics),
  prototypes for `_stpcpy`, `strverscmp`, `obstack_printf` (provided by `common`).

**Replay:** carry these forward mostly as-is; reconcile against the new gnulib's `_GL_*` macro set
(new gnulib may add/rename attribute macros that need new stubs) and the new bison's `config.h.in`
feature list.

## 4. `unistd.h` avoidance (not shimming) *(hand-maintained)*

There is **no `unistd.h` shim file**. Two mechanisms instead:
- Build-time: `common/misc/config.h` pulls `<io.h>` and defines the `*_FILENO` macros.
- Emit-time: flex's `--wincompat` option makes **generated scanners** emit `<io.h>` +
  `#define isatty _isatty` / `#define fileno _fileno` instead of `<unistd.h>`. Documented in
  `winflexbison/UNISTD_ERROR.readme`. The option is wired in `flex/src/main.c` (help text at
  `main.c:1976`) and its output lives in `flex/src/flex.skl` (→ `scan.c`/`skel.c`).

**Replay:** ensure the `--wincompat` block survives in the new `flex.skl`; regenerate `scan.c`/
`skel.c`.

## 5. In-process subprocess replacement *(hand-maintained — deepest logic rewrites)*

POSIX `fork`/`pipe`/`exec` pipelines don't exist on MSVC; both are replaced with temp-file /
in-process equivalents.

- **flex filter chain** — `flex/src/filter.c`. Upstream's `pipe()`/`fork()`/`execvp()` body is
  wrapped in `#if 0 … #endif` (lines **257–327**; a second `#if 0` block at **107–161**) and
  replaced with a temp-file implementation using `<io.h>`/`<process.h>` and
  `add_tmp_dir()` (line **43**, honoring `FLEX_TMP_DIR`).
- **bison → m4** — `bison/src/output.c`. Upstream opens a bidirectional pipe to an external `m4`;
  winflexbison declares `main_m4(...)` (line **724**), writes the m4 program to a temp file
  (`pid_tempname("~m4_in_")`, line **817**), calls the **bundled** m4 in-process (line **840**),
  reads back a temp output file. `#include <spawn-pipe.h>` is commented out (line **30**).
- **exe/data location** — `common/misc/app_path.c` (`get_app_path()` via `GetModuleFileNameA`) and
  `relocatable.c` (`GetModuleFileName`) let bison find its `data/` skeletons next to the exe. The
  m4 entry point is renamed `main`→`main_m4` in `common/m4/m4.c` (line ~406) so it links into
  `win_bison.exe`.

**Replay:** these are the highest-risk merges — diff the new upstream `filter.c` / `output.c`
carefully; the surrounding upstream code often changes around the `#if 0` regions.

## 6. Scattered `#ifdef _MSC_VER` blocks *(hand-maintained — small)*

Very few, **flex only** (bison proper has **zero** inline ifdefs):
- `flex/src/flexdef.h:44-46` — `#if _MSC_VER < 1900` `#define snprintf _snprintf` (pre-VS2015).
- `flex/src/tables.c:39-41` — `htonl`/`htons` via `_byteswap_ulong`/`_byteswap_ushort`.
- `flex/src/main.c:181-182` — `_setmode(_fileno(stdout/stderr), _O_BINARY)` for binary stdio; plus
  `<io.h>`/`<process.h>` includes and the `--wincompat` help text (`main.c:1976`).

**Replay:** these are self-contained; re-apply verbatim to the same functions in the new flex.

### 6a. Bison line-ending / binary I/O *(hand-maintained — upgrade-fragile)*

Bison has its own Windows binary-mode handling, **not** `#ifdef`-guarded, and history shows it is
easily lost when bison is re-vendored. Keep these on every upgrade:

- `bison/src/main.c` — stdout/stderr set to `_O_BINARY` **only when `WINFLEXBISON_BINARY_OUTPUT=Y`**
  (env-var gated, commit `be28ee7`; default stays text so console CRLF is unchanged).
- `bison/src/files.c` `xfopen` — **force binary for write/append modes** so generated output
  (parser `.c`/`.h`, `.output` report, `.dot` graph, skeleton output) is LF like upstream. Reads
  stay text so CRLF **input** grammars have their `\r` stripped (`scan-gram.c` reads via `xfopen
  "r"`). Centralized here (not per-call-site) *specifically* because the original per-file `"wb"`
  patches (`a32e862`, in `scan-skel.c`/`print.c`/`print-graph.c`) were **silently lost** in the
  3.8.2 re-vendor — a central chokepoint survives future re-vendors. On POSIX `text == binary`.
- `bison/src/location.c` `caret_set_file` — open the quoted source file **binary** (`"rb"`), not
  text: the caret code smashes `\r\n` itself and relies on true `ftell`/`fseek` byte offsets; text
  mode desyncs them, producing garbled/empty caret source-line echoes.
- `common/m4/builtin.c` `m4_syscmd` — the port emulates bison's `b4_cat` (`syscmd([cat <<'_m4eof'
  …])`) since there is no shell; it must **strip the here-doc delimiter lines**, else `_m4eof`
  leaks into generated code and diagnostics. (This trimming had regressed in the m4 1.4.19 upgrade.)
- `bison/src/files.c` `xfopen` — map **`"/dev/null"` → `"NUL"`**. The MSVC CRT has no `/dev/null`,
  and `scan-skel.c` redirects skeleton output there whenever complaints have been issued
  (`xfopen (complaint_status ? "/dev/null" : *out_namep, "w")`). Without the mapping *every*
  diagnostic-producing run dies with `bison: /dev/null: cannot open` appended to its stderr —
  24 of the first 25 autotest failures traced to this one line. `compute_output_file_names` in the
  same file already performed the substitution for `--output`, so this was a gap, not a decision.

**Replay:** after re-vendoring bison, grep the new tree for `xfopen`, `_setmode`,
`caret_set_file`/`fopen (caret`, `cat <<`, and `/dev/null`, and re-apply. The `tests/bison-autotest`
suite catches regressions of all of these (diagnostic caret tests, `%define`/skeleton error tests,
and any test comparing a generated file).

**Class of defect worth naming: `/dev/null` is not a writable file on this port.** It has bitten
twice — the `xfopen` entry above, and `tests/bison-autotest/run.sh`'s C++-standard probe, which
linked to `-o /dev/null` and therefore reported every `-std=` flag as unsupported, silently skipping
59 `glr2.cc` test groups. Both failed quietly rather than loudly. When a Windows-side tool "does
nothing" for no visible reason, check this first.

### 6b. Temp-file handling *(hand-maintained)*

Both tools replace the Unix `fork`+`pipe` filter/m4 machinery with real temp files in `%TEMP%`
(there is no `fork`; MSVC has no `fmemopen`, and `tmpfile()` writes to the drive root and fails
without admin). Keep:

- `common/misc/pid_tempname.c` — temp names include `_getpid()` so **concurrent** win_flex/win_bison
  invocations get unique names (the documented race the port fixes). Honors `FLEX_TMP_DIR`/`TEMP`.
- **Delete-on-close** (MSVC `"D"` fopen flag) for the intermediate temp files, so they are removed
  when the handle closes — including on crash/kill — with no manual tracking:
  `flex/src/filter.c` `mkstempFILE`, `flex/src/main.c` `flex_temp_out_main` (`freopen … "w+D"`),
  `bison/src/output.c` `m4_in`/`m4_out` (`"wb+D"`). Each is used only via its `FILE*` handle, never
  reopened by name, so delete-on-close is safe. The old explicit `unlinktemp()`/`_unlink` paths were
  dropped (they only ran on clean exit and leaked otherwise).
- `bison/src/fixits.c` — `--fixit` `remove(backup)` before `rename(input, backup)`: MSVC `rename`
  fails if the target exists (POSIX overwrites), so a re-run reported "cannot backup".

**Replay:** small, self-contained; grep the new trees for `mkstempFILE`, `pid_tempname`, `"wb+"`
around `m4_in`/`m4_out`, and `rename` in `fixits.c`. `tests/winflexbison` (parallel resistance)
guards the concurrency + no-leak behavior.

### 6c. Bison skeleton divergences (`bison/data/`) *(hand-maintained — upgrade-fragile)*

`bison/data/` is data, not code: it is copied next to the exe and read at run time, so upgrades
tend to replace it wholesale and a patch here disappears without a compiler ever noticing. Until
2.5.26 the tree was byte-identical to upstream apart from `m4sugar/`. Now one patch lives here:

- `bison/data/skeletons/c.m4`, `b4_sizes_types_define` — an `_MSC_VER` branch in the `YYPTRDIFF_T`
  width ladder (#95). Upstream's ladder asks for `__PTRDIFF_TYPE__` (GCC/Clang), then `PTRDIFF_MAX`
  (`<stdint.h>`, which the skeleton includes only when `__STDC_VERSION__ >= 199901`), then falls
  back to `long`. MSVC publishes neither, and reports `__STDC_VERSION__` only under `/std:c11` or
  later, so every generated parser landed on `long` — 32 bits on 64-bit Windows, where `ptrdiff_t`
  is 64 — and every x64 build warned C4244 on `YYPTRDIFF_T yysize = yyssp - yyss + 1`. The branch
  takes `ptrdiff_t` from `<stddef.h>` and picks the maximum by `_WIN64`.

  **Do not "simplify" this by including `<stdint.h>` for MSVC instead** — the obvious fix, and it
  breaks the flex+bison combination: a generated flex scanner defines `INT8_MIN`, `INT32_MAX` and
  friends itself, and MSVC's `<stdint.h>` redefines them unguarded, so C4005 kills any translation
  unit holding both. `tests/flex` `core_yylex_wrapper` catches it; that is how it was found.

**Replay:** after re-vendoring `bison/data/`, grep the new `c.m4` for `_MSC_VER` — one hit in
`b4_sizes_types_define` — and re-apply. `bison.ptrdiff_width` in the CTest suite fails (loudly, on
x64) if it is lost. This is also an upstreaming candidate (#106): upstream master still has the
unguarded ladder.

**Build-dependency note:** a skeleton edit does not rebuild `win_bison`, so nothing in the build
graph used to connect it to the parsers the tests generate — an incremental build after a skeleton
change quietly recompiled the previous run's output. `tests/CMakeLists.txt` now globs
`bison/data/*` into `WFB_BISON_SKELETONS` and every `win_bison` custom command lists it in
`DEPENDS`. A newly *added* skeleton file still needs a cmake re-run.

## 7. Build-system flags *(mechanical)*

In the CMake tree (see [../../../winflexbison/CMakeLists.txt](../../../winflexbison/CMakeLists.txt)):
- Root: `-D_CRT_SECURE_NO_WARNINGS`, `-Dinline=__inline` (ucrt C-mode bug), `-Drestrict=__restrict`
  (VS2017), `__extension__=` (MSVC-only, not clang-cl), `-D_DEBUG` (Debug),
  `/utf-8` (both source **and** execution charset UTF-8 — the execution charset matters so bison's
  UTF-8 glyph literals, e.g. the item dot in `.output`/graphs, stay UTF-8 rather than being
  transcoded to the system codepage), optional `/MD`→`/MT` (`USE_STATIC_RUNTIME`).
- Subdirs: `-D_LIB` (common), `-D_CONSOLE` (both exes), link `kernel32.lib user32.lib`,
  `C_STANDARD 90` for common.

Additional keyword/attribute coping lives in `common/misc/config.h` (category 3), not the build
system.

---

## Regenerating the authoritative diffs

The diffs ARE the catalog's ground truth. Regenerate them from the baselines
([02](../02-baseline-mirrors/spec.md)) and store the artifacts in this folder
(`04-port-change-catalog/diffs/`) so an upgrade can see exactly what must survive.

```sh
# run from the workspace checkout, but NOT from inside a submodule --
# --show-toplevel would then resolve to the submodule instead of the workspace
ROOT=$(git rev-parse --show-toplevel)
OUT=$ROOT/docs/specs/04-port-change-catalog/diffs
mkdir -p "$OUT"

# Flex: vendored vs baseline (exclude committed generated files for the hand-patch view)
diff -ru "$ROOT/upstream/flex/src" "$ROOT/winflexbison/flex/src" \
     -x scan.c -x parse.c -x parse.h -x skel.c > "$OUT/flex-src.patch"

# Bison src: vendored vs baseline (exclude generated + the injected config.h)
diff -ru "$ROOT/upstream/bison/src" "$ROOT/winflexbison/bison/src" \
     -x 'parse-gram.*' -x 'scan-*.c' > "$OUT/bison-src.patch"

# Bison data: skeletons and m4sugar the exe reads at run time (category 6c)
diff -ru "$ROOT/upstream/bison/data" "$ROOT/winflexbison/bison/data" > "$OUT/bison-data.patch"

# M4: vendored vs baseline
diff -ru "$ROOT/upstream/m4/src" "$ROOT/winflexbison/common/m4" > "$OUT/m4-src.patch"

# gnulib (bison-side → common/misc): checkout the bison-pin first
git -C "$ROOT/upstream/gnulib" checkout 7818455627c5e54813ac89924b8b67d0bc869146
diff -ru "$ROOT/upstream/gnulib/lib" "$ROOT/winflexbison/common/misc" > "$OUT/gnulib-misc.patch"

# gnulib (m4-side → common/m4/lib): switch to the m4-pin
git -C "$ROOT/upstream/gnulib" checkout 3639c57a970191e0bf7a9789bd1341786d0255a1
diff -ru "$ROOT/upstream/gnulib/lib" "$ROOT/winflexbison/common/m4/lib" > "$OUT/gnulib-m4lib.patch"
```

> Path caveats: the vendored trees are re-organized relative to upstream (e.g. bison's `lib/`
> gnulib is split into `common/misc`; m4's gnulib is under `common/m4/lib`). The `diff` roots above
> are approximate — expect "only in" noise from the reorg and from vendored files that were dropped.
> The **signal** is the per-file hunks in the hand-maintained files (categories 3–6): `filter.c`,
> `output.c`, `flexdef.h`, `tables.c`, `main.c`, the two `config.h`, `app_path.c`, `relocatable.c`,
> `m4.c`. Focus the review there.

Regenerating these diffs (against the OLD baseline) at the start of an upgrade confirms the catalog
above still matches reality before you touch anything.
