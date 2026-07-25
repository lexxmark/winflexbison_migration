# 01 — Version Inventory & Provenance

**Task:** Determine and record the exact upstream version of every vendored component, with
commit-level provenance, and define how to determine versions for any tree.

## Current inventory (verified)

| Component | Version | Upstream repo | Tag | Commit SHA | Vendored under |
|---|---|---|---|---|---|
| Flex | **2.6.4** | github.com/westes/flex | `v2.6.4` | `ab49343b08c933e32de8de78132649f9560a3727` | `winflexbison/flex/src` |
| GNU Bison | **3.8.2** | github.com/akimd/bison | `v3.8.2` | `9beba1919cad5dd08b0cac277c27896808719e4b` | `winflexbison/bison/{src,lib,data}` |
| GNU M4 | **1.4.19** | git.savannah.gnu.org/git/m4 | `v1.4.19` | `445afe00b62d8a7bee109faf3b96edf0c97b7a85` | `winflexbison/common/m4` |
| gnulib (bison-side) | *snapshot* | git.savannah.gnu.org/git/gnulib | — | `7818455627c5e54813ac89924b8b67d0bc869146` | `winflexbison/common/misc` |
| gnulib (m4-side) | *snapshot* | git.savannah.gnu.org/git/gnulib | — | `3639c57a970191e0bf7a9789bd1341786d0255a1` | `winflexbison/common/m4/lib` |

Notes:
- **WinFlexBison's own version is 2.5.25** (`winflexbison/CMakeLists.txt:3`
  `project(winflexbison VERSION 2.5.25 ...)`). This is the *port's* release number, unrelated to
  any vendored component's version.
- Bison bundles a heavy gnulib subset that upstream would normally compile into `bison/lib`; in
  winflexbison that subset lives in `common/misc`, so **bison-side gnulib maps to `common/misc`**.
- M4 has its own gnulib copy; **m4-side gnulib maps to `common/m4/lib`**. The two gnulib snapshots
  differ (see the two SHAs above), which is why both are pinned separately.

## Where each version is stamped in the vendored tree

Use these when you need to read the version out of the *vendored* source (e.g. to confirm a state
mid-upgrade):

| Component | Evidence in vendored tree |
|---|---|
| Flex | `flex/src/main.c:44` — `static char flex_version[] = "2.6.4";` (hardcoded, replaces autoconf `FLEX_VERSION`). Also `flex/src/skel.c:49-51` — `YY_FLEX_{MAJOR,MINOR,SUBMINOR}_VERSION 2/6/4`. |
| Bison | `bison/src/config.h:4` `#define VERSION "3.8.2"`; `:10` `PACKAGE_VERSION`. Also `bison/src/parse-gram.c:52` `#define YYBISON_VERSION "3.8.2"`. |
| GNU M4 | **No numeric literal** — `common/m4/m4.h:27` defines only `PACKAGE_STRING "M4"` and `common/m4/m4.c:620-621` has the version print commented out. Version is inferred from the copyright range `common/m4/m4.c:3` (`… 2020-2021 …` ⇒ 1.4.19, May 2021) and `changelog.md:9`. |
| gnulib | No version number exists (gnulib is an unversioned moving snapshot). In the vendored tree the only signal is copyright years (max **2021** across `common/`, zero 2022 hits). The exact commit is **not** recoverable from the vendored files — it must come from the release submodule pin (below). |

## How to determine versions for ANY tree (the reusable procedure)

1. **Flex / Bison** — read the macros above. Flex's is a hand-edited literal; Bison's is in
   `config.h`.
2. **M4** — read the copyright year range in `m4.c` / `m4.h` and cross-check `changelog.md`. There
   is no numeric define to trust.
3. **gnulib exact commit (the important trick)** — gnulib ships with releases as a **git
   submodule**, so the exact snapshot a release used is recorded by that release's tree, not by any
   version string. Recover it from a baseline clone (see [02](../02-baseline-mirrors/spec.md)):
   ```sh
   # bison-side gnulib (authoritative for common/misc):
   git -C orig/bison ls-tree v3.8.2 gnulib      # → 160000 commit <SHA> gnulib
   # m4-side gnulib (authoritative for common/m4/lib):
   git -C orig/m4   ls-tree v1.4.19 gnulib      # → 160000 commit <SHA> gnulib
   ```
   The `<SHA>` on the `160000 commit` line is the pinned gnulib commit.

## Rule going forward

**Record commit SHAs, not just version numbers.** Every future upgrade must append a row to the
inventory table above (and to `changelog.md`) capturing component → version → tag → resolved SHA,
including both gnulib SHAs recovered via the submodule trick. Version numbers alone have already
proven insufficient (they could not pin gnulib at all).
