# 05 — Upgrade Procedure

**Task:** The runbook for adopting a new upstream release of flex, bison, and/or m4 while
faithfully re-applying the Windows port.

## Three approaches

Pick per component based on how much upstream moved:

1. **Replay** — start from the *fresh new upstream* tree and re-apply the documented port changes
   ([04](../04-port-change-catalog/spec.md)) on top, then fix new compile/link breakage.
   - Best when: upstream refactored a lot, so the old vendored files would conflict heavily. You
     get clean new code + a known, small patch set to reapply.
2. **Diff-merge** — take the *current vendored* tree and merge the upstream `old→new` diff into it
   by hand, resolving conflicts against the port changes.
   - Best when: upstream changed little; the port edits stay in place and you only fold in small
     upstream deltas.
3. **Hybrid (recommended default)** — **re-vendor the mechanical categories fresh** (category 1
   gnulib/m4, category 2 generated files) via *replay*, and **diff-merge the hand-maintained
   categories** (3–6) since those files are few and their port hunks are precious.

## Runbook

### Step 0 — Prepare baselines
- Add the new target version(s) under `upstream/` and recover their gnulib submodule SHA(s)
  ([02](../02-baseline-mirrors/spec.md) → "Adding the NEW target version").
- **Regenerate the current port diffs against the OLD baseline**
  ([04](../04-port-change-catalog/spec.md) → "Regenerating the authoritative diffs") and skim them
  to confirm the catalog still matches reality. This is your patch set to preserve.

### Step 1 — Re-vendor the mechanical bulk (category 1 & 2)
- Copy the new upstream sources into the vendored layout:
  - flex `src/` → `winflexbison/flex/src` (keep the port files; see Step 2).
  - bison `src/`, `data/`, `lib/` → `winflexbison/bison/{src,data,lib}`.
  - m4 → `winflexbison/common/m4`; new gnulib → `common/misc` (bison-pin) and `common/m4/lib`
    (m4-pin), preserving the `common/CMakeLists.txt` exclusion list.
- Regenerate the committed generated files (Step 4) — do not hand-copy stale ones.

### Step 2 — Replay / merge the hand-maintained patches (category 3–6)
Apply, in order, to the corresponding new-upstream files, using the OLD-baseline diffs as the
source of truth:
- `common/misc/config.h`, `bison/src/config.h` — carry forward; reconcile with the new gnulib's
  `_GL_*` macro set and the new `config.h.in` feature list.
- `flex/src/filter.c` — reapply the `#if 0` fork/pipe fencing + temp-file/`add_tmp_dir` path
  (highest-risk; upstream code around it shifts).
- `bison/src/output.c` — reapply the `main_m4` in-process path + `pid_tempname` temp files.
- `common/misc/app_path.c`, `relocatable.c`, and the `main`→`main_m4` rename in `common/m4/m4.c`.
- Flex `#ifdef _MSC_VER` sites: `flexdef.h` snprintf, `tables.c` byteswap, `main.c` binary
  mode/`--wincompat`; and the `--wincompat` block in `flex.skl`.

### Step 3 — Build-system flags (category 7)
- Reconcile `CMakeLists.txt` files against any new upstream source files (new `.c` added upstream
  must be picked up by the `file(GLOB …)` — verify), keep all MSVC flags.

### Step 4 — Regenerate committed generated files
- With a bootstrap flex/bison/m4 available, regenerate flex `scan.c`/`parse.c`/`parse.h`/`skel.c`
  and bison `parse-gram.*`/`scan-*.c`. Confirm the flex `--wincompat` output is present in the new
  `scan.c`/`skel.c`.

### Step 5 — Version bookkeeping
- Update the vendored version stamps: flex `main.c` literal, bison `config.h` `VERSION`/
  `PACKAGE_VERSION`, m4 copyright.
- Bump `winflexbison/CMakeLists.txt` `project(... VERSION x.y.z)` **only if cutting a port
  release**.
- Add a `changelog.md` entry recording every component's new version **and** the resolved tag→SHA
  (including both gnulib SHAs) — update [01](../01-version-inventory/spec.md)'s table too.

### Step 6 — Refresh & run tests
- Refresh the vendored test set from the new baseline and (re-)pre-generate flex scanners
  ([03](../03-test-adoption/spec.md)).
- Proceed to the gate: [06 — validation](../06-validation/spec.md). **Tests must be green before
  merge.**

## Fix-loop expectation

After Step 4, expect a first build to surface new MSVC breakage (new gnulib attribute macros,
`size_t`/`int` narrowing, POSIX calls upstream newly introduced). Triage each: if it's a missing
gnulib shim, extend `common/misc/config.h` (category 3); if it's a new POSIX
subprocess/`unistd`/`fork` usage, it needs a new port hunk (extend category 5/6 **and** the catalog
in [04](../04-port-change-catalog/spec.md) so the next upgrade inherits it).
