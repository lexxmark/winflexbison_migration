# Upstream Version Check — 2026-07-26

A point-in-time comparison of the vendored versions (recorded in [spec.md](spec.md)) against the
latest upstream releases, done to decide whether an upgrade pass is warranted right now. Unlike
`spec.md` (a reusable procedure), this file is a dated snapshot — re-run the checks below and add a
new dated file rather than editing this one, if the question comes up again later.

## Result summary

| Component | Vendored | Latest upstream | Verdict |
|---|---|---|---|
| Flex | 2.6.4 | 2.6.4 (unchanged since 2017-05-06) | **Up to date** — no release to adopt |
| GNU Bison | 3.8.2 | 3.8.2 (unchanged since akimd/bison + ftp.gnu.org agree) | **Up to date** — no release to adopt |
| GNU M4 | 1.4.19 | **1.4.21** (2026-02-06) | **Behind by two releases** — upgrade recommended |

## Flex

- Vendored: `v2.6.4` / `ab49343b08c933e32de8de78132649f9560a3727` (matches `spec.md`).
- Checked `github.com/westes/flex` tags + `GET /repos/westes/flex/releases/latest` — `v2.6.4` is
  still both the newest tag and the only GitHub Release. Flex has no `ftp.gnu.org` presence (it's
  not GNU-hosted), so GitHub is authoritative here.
- `master` is **589 commits ahead** of `v2.6.4` with nothing tagged since — real upstream activity
  exists but hasn't been cut into a release. Nothing to adopt via the normal
  tag-based baseline process; would require tracking a moving `master` snapshot instead, which is
  out of scope unless a specific unreleased fix is needed.

## GNU Bison

- Vendored: `v3.8.2` / `9beba1919cad5dd08b0cac277c27896808719e4b` (matches `spec.md`).
- Checked `github.com/akimd/bison` tags (Akim Demaille's mirror — Bison's actual maintainer) *and*
  cross-verified against `ftp.gnu.org/gnu/bison/`: both agree `3.8.2` is the newest release.
- `master` is **45 commits ahead** of `v3.8.2`, same situation as Flex — active unreleased
  development, no new tag to adopt.

## GNU M4 — behind, upgrade recommended

- Vendored: `v1.4.19` / `445afe00b62d8a7bee109faf3b96edf0c97b7a85` (matches `spec.md`).
- `ftp.gnu.org/gnu/m4/` lists two newer releases:

  | Version | Released | Tag SHA (`upstream/m4`) | gnulib submodule SHA |
  |---|---|---|---|
  | 1.4.19 *(vendored)* | 2021-05-28 | `445afe00b62d8a7bee109faf3b96edf0c97b7a85` | `3639c57a970191e0bf7a9789bd1341786d0255a1` |
  | 1.4.20 | 2025-05-10 | `3b138ea2f54bb622dae2ee2a6aca2c4f67b392da` | `9fc42e5f5711e501f80559539a78aed2b7c842ac` |
  | 1.4.21 *(latest)* | 2026-02-06 | `d192fbeeae14a5f75cb1a3e76a3ff78f67e39d40` | `c6842ec25c6d5b1596da4eb6b68603f9578b23e9` |

  (Tags/SHAs recovered by `git -C upstream/m4 fetch --tags` then `git rev-parse`/`git ls-tree <tag>
  gnulib` — the same recipe as `spec.md`'s "how to determine versions for any tree" section.)

### Why it's worth adopting

From the upstream `NEWS` (`git -C upstream/m4 diff v1.4.19..v1.4.21 -- NEWS`):

- **Two fixes for regressions introduced in 1.4.19 itself** (the version currently vendored):
  - `debugmode(t)`-style trace output could read invalid memory when tracing a series of pushed
    macros popped during argument collection.
  - The `format` builtin picked up unwanted locale-dependent parsing/output of floating-point
    numbers as a side effect of adding message translations.
- **Windows-relevant fix** (1.4.20): loading a frozen file on "non-Unix platforms where binary
  files differ from text" now correctly uses binary mode — directly applicable to this port.
- 1.4.21 adds an `eval` fix (reject `0x` as invalid rather than silently treating it as zero), a
  `defn`/`pushdef` warning-consistency fix, and a glibc 2.43/C23 portability fix (not
  Windows-relevant, but shows upstream tracking newer compiler standards generally).

### Why it's not a trivial version bump

`git -C upstream/m4 diff --stat v1.4.19..v1.4.21 -- . ':!gnulib' ':!doc' ':!po'` shows 60 files changed
(+3289/-7727 lines). Most of the deletions are `gl/build-aux/*` being split out to a separate
`gl-mod` submodule (bootstrap tooling, not runtime code), but the real interpreter sources changed
substantially too — and per `docs/specs/04-port-change-catalog/spec.md`, **all of these are vendored
and Windows-patched** under `common/m4/` (not just diffed for reference):

`builtin.c`, `debug.c`, `eval.c` (~826 lines changed), `format.c`, `freeze.c`, `input.c`, `m4.c`,
`macro.c`, `output.c`, `path.c`, `symtab.c`, plus the `common/m4/lib` gnulib snapshot (gnulib SHA
also moves, per the table above).

Known Windows patches that touch these files and would need replaying (per the port-change
catalog): the `main`→`main_m4` entry-point rename in `m4.c`, and the `b4_cat`/`m4_syscmd` emulation
in `builtin.c`. A full patch inventory should be regenerated from the new baseline per
[04-port-change-catalog](../04-port-change-catalog/spec.md) before starting the upgrade itself.

## Recommendation

- **Flex, Bison:** no action — already at the latest tagged upstream release.
- **GNU M4:** upgrade to **1.4.21**, following the existing playbook: add the `v1.4.21` baseline
  under `upstream/m4` ([02](../02-baseline-mirrors/spec.md)), regenerate the current port-change catalog
  from the *old* (1.4.19) baseline to confirm it's current
  ([04](../04-port-change-catalog/spec.md)), then replay per
  [05-upgrade-procedure](../05-upgrade-procedure/spec.md) and validate per
  [06-validation](../06-validation/spec.md). Not started as part of this check.
