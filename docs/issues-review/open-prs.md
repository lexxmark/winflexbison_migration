# Open Pull Requests Backlog

**Purpose:** A snapshot of all currently-open pull requests on
[`lexxmark/winflexbison`](https://github.com/lexxmark/winflexbison), collected for triage —
which ones are ready to merge, need a rebase/re-review, or should just be closed.

**Methodology:** Source is the GitHub API (`GET /repos/lexxmark/winflexbison/pulls`), filtered to
`state=open`. Generated 2026-07-26.

**Total open PRs:** 6

Each entry: number, title, link, a plain-English paraphrase of the change, labels (if any), and a
staleness read based on `created_at` vs. today (2026-07-26).

---

## [#59 — Implemented compatibility to MinGW-w64](https://github.com/lexxmark/winflexbison/pull/59)

Opened 2020-09-03 (**~5.9 years old**). 9 comments, no labels, no PR body captured.

Proposes changes to get the project building under MinGW-w64 as an alternative toolchain to
MSVC/clang-cl. The project's `CMakeLists.txt` currently hard-fails if `MSVC` isn't set, so this
would be a scope change (officially supporting a GCC-based toolchain on Windows), which likely
explains the long discussion (9 comments) without a merge. PR #85 later applied some smaller
"mingw64 compat" adjustments referencing this PR, suggesting a full MinGW port was never finished
or accepted piecemeal instead.

**Staleness:** Very stale — nearly 6 years old. Almost certainly needs a full rebase against the
current CMake/source layout, and a decision on whether MinGW support is in scope at all before any
further work continues. Prime candidate for closing (with a redirect to file a fresh, narrower PR)
unless someone actively wants to pick it back up.

---

## [#71 — Warning GCC and Clang : Empty lines : %option '-L'](https://github.com/lexxmark/winflexbison/pull/71)

Opened 2021-01-11 (**~5.5 years old**). 6 comments, labels: `Flex, upstream`.

Fixes two flex-codegen issues: (1) generated scanners triggered `-Wmisleading-indentation`
warnings under GCC 6+/Clang 10+ (fix in `flex/src/skel.l`), and (2) using `-L` (no `#line`
directives) together with a `%top` block produced a flood of blank lines and still emitted a
`#line` directive for `%top`, traced to an incorrect comparison in `flex/src/buf.c` plus a missing
`-L` check in `flex/src/scan.c`.

**Staleness:** Very stale — over 5 years old, labeled `upstream` (meaning the fix may need to be
coordinated with/ported from westes/flex directly rather than merged standalone). Needs a fresh
look: verify the fix still applies cleanly against the current vendored `flex/src/`, check whether
upstream flex fixed this differently since 2021, and either rebase or close.

---

## [#92 — Add support for vscode](https://github.com/lexxmark/winflexbison/pull/92)

Opened 2023-03-11 (**~3.4 years old**). 7 comments, no labels, no PR body captured.

Proposes adding VS Code integration/config (likely `.vscode/` tasks, CMake Tools settings, or
similar) so the project can be built/debugged from VS Code instead of only through the VS
solution/CMake CLI. Exact scope is unclear without the body text, but the comment count suggests
back-and-forth on what should actually be committed (editor-specific config is often contentious).

**Staleness:** Stale — over 3 years old. Needs re-review to see if it's still relevant given the
CMakePresets-based workflow added since (PR #107), which already gives VS Code's CMake Tools
extension a supported entry point — this PR may now be redundant.

---

## [#99 — Cleanup CMake even more](https://github.com/lexxmark/winflexbison/pull/99)

Opened 2025-09-24 (**~10 months old**). 7 comments, no labels.

Originally submitted to fix issue #98 (CMake modernization / clang-cl detection), but per its own
body, most of that work landed via #101 and #104 instead — the author says this PR now "only
contains changes the other commits haven't done," i.e. it's the leftover diff after two other PRs
were merged.

**Staleness:** Self-flagged as partially superseded already. Needs a fresh diff against current
`main` to see what (if anything) is left worth taking, then either finish it or close it as
subsumed by #101/#104.

---

## [#108 — Readme: Show status badge of the check-runs of the default git branch](https://github.com/lexxmark/winflexbison/pull/108)

Opened 2026-03-04 (**~4.7 months old**). 2 comments, no labels.

Small README change: swap the current AppVeyor-only build-status badge for one showing GitHub
Actions check-runs on the default branch, since (per the PR body) AppVeyor no longer actually runs
against the main repository.

**Staleness:** Moderate — a few months old but low-risk/low-effort (docs-only change with only 2
comments). Should be quick to re-review and merge; no obvious blocker.

---

## [#109 — Resolve symlinks when getting the path to the app](https://github.com/lexxmark/winflexbison/pull/109)

Opened 2026-04-08 (**~3.7 months old**). 0 comments, no labels.

Windows-only fix: when win_bison/win_flex are installed via winget, the executables end up as
symlinks in a WinGet "Links" folder; the app-path lookup used to locate `data/m4sugar/m4sugar.m4`
was resolving relative to the *symlink's* location rather than the real install directory. This PR
resolves the symlink to its final target first, before deriving the data-directory path. Per the
author, it only touches Windows-specific code (no upstream bison changes needed), and flex is not
affected (only bison depends on runtime-resolved data files).

**Cross-reference — issue [#97](https://github.com/lexxmark/winflexbison/issues/97)** ("Add
support install win_bison and win_flex as symbolic link by using winget", open-issues.md): this is
the same bug, described from the user side — win_bison failing to find
`data/m4sugar/m4sugar.m4` when installed via winget's symlink mechanism. The PR body itself says
"Maybe fixes #97". Based on the description, **this PR looks like a strong candidate to close
#97** — the mechanism described (symlink resolution before path derivation) directly matches the
reported failure mode. Recommend prioritizing review/merge of this one and then closing #97 once
verified against a real winget install layout.

**Staleness:** Fresh (under 4 months), zero comments — likely just hasn't been reviewed yet rather
than stalled. Good candidate for prompt review given it fixes a real, still-open user-reported bug.

---

*Generated from a one-time GitHub API pull; re-run the same query against
`lexxmark/winflexbison` pull requests (state=open) to refresh this list.*
