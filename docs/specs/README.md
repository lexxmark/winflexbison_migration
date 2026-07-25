# winflexbison Upgrade Playbook

This directory is the **repeatable playbook for adopting new upstream releases** of GNU Flex,
GNU Bison, and GNU M4 into the winflexbison Windows/MSVC port.

## Why this exists

winflexbison does not depend on the upstream projects — it **vendors** (copies in) their C source
and then **patches** it to build under MSVC. Historically the only record of *what* was vendored
was a version number in `changelog.md`, and there was no record of *which Windows patches* were
applied or *how* to redo them. Every upgrade therefore started with archaeology. These specs
replace that with: pinned baselines, a catalog of the port changes, a step-by-step upgrade
procedure, and a test gate that must be green before merge.

## Glossary

- **Vendored tree** — the copy of upstream source that lives inside the project
  (`winflexbison/flex/src`, `winflexbison/bison/src`, `winflexbison/common/m4`,
  `winflexbison/common/misc`). This is what actually ships.
- **Baseline** — a pristine upstream checkout at the **exact tag matching the vendored version**,
  kept under `orig/`. Diffing vendored-vs-baseline yields the Windows patch set.
- **Replay** — re-applying the documented Windows patches on top of a fresh newer upstream tree.
- **Generated file** — a build product upstream produces at build time (e.g. flex `scan.c` from
  `scan.l`) that winflexbison commits directly so the project builds without flex/bison/sh/m4
  present to bootstrap.

## The specs (execute in this order during an upgrade)

| # | Spec | Purpose |
|---|---|---|
| 01 | [version-inventory](01-version-inventory/spec.md) | What is vendored now, with exact tag→SHA provenance, and how to determine versions for any tree. |
| 02 | [baseline-mirrors](02-baseline-mirrors/spec.md) | The `orig/` baseline layout, exact repos/tags/SHAs, and how to add the new target version. |
| 03 | [test-adoption](03-test-adoption/spec.md) | How the upstream flex/bison test suites work and the Windows CTest design that reuses them. |
| 04 | [port-change-catalog](04-port-change-catalog/spec.md) | The canonical catalog of Windows port changes to replay, and how to regenerate the diffs. |
| 05 | [upgrade-procedure](05-upgrade-procedure/spec.md) | The runbook: replay / diff-merge / hybrid, step by step. |
| 06 | [validation](06-validation/spec.md) | The acceptance gate — build green, warning baseline, tests green, sign-off. |

## Upgrade at a glance

```
 pick new upstream version(s)
   │
   ▼
 [02] add new baseline under orig/ (new tag → recover its gnulib submodule SHA)
   │
   ▼
 [04] regenerate current port diffs from the OLD baseline → confirm catalog is current
   │
   ▼
 [05] apply new upstream + replay port changes (or diff-merge / hybrid)
   │   – re-vendor gnulib/m4 fresh (mechanical categories)
   │   – replay the hand-maintained patches (config.h ×2, filter.c, output.c, app_path/relocatable, flex #ifdefs)
   │   – regenerate the committed generated files
   │   – bump version in CMakeLists.txt + config.h + changelog.md
   │
   ▼
 [03] refresh the vendored test set from the new baseline
   │
   ▼
 [06] build all configs green + tests green + warning-baseline check → sign off → merge
```

## Current pinned baselines (as of this playbook's creation)

| Component | Version | Tag | Commit SHA | Upstream |
|---|---|---|---|---|
| Flex | 2.6.4 | `v2.6.4` | `ab49343b08c933e32de8de78132649f9560a3727` | github.com/westes/flex |
| GNU Bison | 3.8.2 | `v3.8.2` | `9beba1919cad5dd08b0cac277c27896808719e4b` | github.com/akimd/bison |
| GNU M4 | 1.4.19 | `v1.4.19` | `445afe00b62d8a7bee109faf3b96edf0c97b7a85` | git.savannah.gnu.org/git/m4 |
| gnulib (bison-side → `common/misc`) | — | — | `7818455627c5e54813ac89924b8b67d0bc869146` | git.savannah.gnu.org/git/gnulib |
| gnulib (m4-side → `common/m4/lib`) | — | — | `3639c57a970191e0bf7a9789bd1341786d0255a1` | git.savannah.gnu.org/git/gnulib |

## Scope note

This playbook was authored together with the initial `orig/` baseline setup. Implementing the
CTest harness, pre-generating the flex test scanners, adapting the bison test suite, and any actual
version upgrade are **future work executed by following these specs** — they were intentionally not
done in the authoring pass.
