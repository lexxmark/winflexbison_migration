# Release Plan — win_flex_bison 2.5.26

## Purpose

Everything queued for the next release of `lexxmark/winflexbison`, plus the open decisions that
gate cutting it. Sections A and B are inventory (already done, needs no debate); sections C and D
are the decisions; section E is the mechanical checklist once C and D are settled.

Status of this document: **draft for review**. Nothing here is implemented beyond what is already
on `dev`.

## Snapshot

| | |
|---|---|
| Branch | `dev`, 56 commits ahead of `master` (local and `origin/dev` both at `ea60519`) |
| Version | root `CMakeLists.txt` still says `2.5.25`; changelog heading still `### unreleased` |
| Last release | v2.5.25, tagged 2022-01 |
| Last CI | AppVeyor build 167 (`afb22e2`, 2026-08-17) — green on all 9 jobs, 134/134 ctest in the VS2022/x64/Release cell, autotest 715 run / 12 failed, all 12 on the documented xfail list |
| Open on GitHub | 20 issues, 5 pull requests |
| Generated | 2026-08-18 |

---

## A. Already in the box

The `### unreleased` section of `changelog.md` currently holds 13 entries. Grouped by what they
mean to a user:

### Correctness fixes

- win_bison not finding its `data/` directory when started through a symbolic link, e.g. the
  links winget creates in its `Links` folder (#97, fixed in `afb22e2`)
- random failures when several win_flex/win_bison run concurrently, as ninja/meson builds do —
  same `%TEMP%` names, mutual truncation, surfacing as `ERROR: end of file in string`, a crash,
  or silently wrong output (#86, #100; fix by @jonnysoe in #91)
- garbled or blank source lines in win_bison caret diagnostics
- stray `_m4eof` delimiter lines leaking into generated code and skeleton-emitted diagnostics
- `bison: /dev/null: cannot open` appended to stderr by every run that issued a diagnostic
- `--update`/`--fixit` failing with "cannot backup: Permission denied" when the rewritten
  grammar also produced a caret diagnostic
- win_flex/win_bison temp files are now delete-on-close, so they no longer leak when the tool
  crashes or is killed

### Behavior change — needs its own callout in the release notes

- **win_bison now writes LF, not CRLF**, in generated parsers, headers, `.output` reports and
  `.dot` graphs — restoring the behavior 2.5.16 documented and 2.5.17 silently lost.

This is the one item in the set that can surprise a downstream consumer. It should not be a
bullet buried among twelve others — it belongs at the top of the release notes, stated plainly,
so anyone diffing generated output against a 2.5.25 baseline knows why every line moved.

**The wording matters, and the first draft of it was wrong.** This is not a new policy: it is a
repair. See section F for the verified history and the follow-ups it turned up.

### Build and test infrastructure

- fixed Debug builds with `USE_STATIC_RUNTIME=ON` failing to link, and C++ sources being built
  against the wrong CRT
- build with `/utf-8` so UTF-8 literals survive the MSVC execution charset
- added a CTest suite — the flex 2.6.4 suite, bison compile-run and golden-diagnostic tests, and
  port-specific tests — plus `runtests.bat`; run as a gate on AppVeyor
- added the full bison GNU Autotest (776 groups) as an opt-in harness run under MSYS2

---

## B. Issues the release closes for free

No new work; verify and close as part of the release.

| Issue | Title | Why it closes |
|---|---|---|
| #97 | winget symlink / `data` not found | Fixed in `afb22e2`. Explanation already posted. |
| #100 | `extern_stdin:40: ERROR: end of file in string` | No code change was ever needed — #91 fixed it in `master` in 2023-03 and it has simply never been released (v2.5.25 is from 2022-01). Explanation already posted. |
| #17 | re-add bison + flex tests | This is exactly what the CTest suite and the MSYS2 autotest deliver. Open since 2018; @GitMensch's original ask. Worth calling out in the release notes rather than closing quietly. |
| #6 | Make flex-lib and bison-lib projects | `add_library(fl STATIC …)` and `add_library(y STATIC …)` both exist now (`flex/CMakeLists.txt:26`, `bison/CMakeLists.txt:38`). Check whether they are *installed*, not merely built, before closing. |

Both #97 and #100 are still marked OPEN on GitHub despite having their closing comments posted.

---

## C. Candidates to fix before cutting — decision needed on each

### Recommended in

| Issue | What | Notes |
|---|---|---|
| #95 | `YYPTRDIFF_T` not 64-bit compatible | Verified: `bison/data/skeletons/c.m4:296-311` is byte-identical to upstream 3.8.2. MSVC defines neither `__PTRDIFF_TYPE__` nor `PTRDIFF_MAX` (absent `<stdint.h>`), so it lands on the `long` fallback and every x64 build gets C4244. Fix is an MSVC/`_WIN64` branch in `c.m4`. **Caveat:** a deliberate divergence from an upstream skeleton — interacts with the autotest goldens and with #106. |
| #73 | C4244 in `LexerInput` | `return yyin.gcount()` — `std::streamsize` to `int`. One cast in `FlexLexer.h`. Same family as #95, and cheap. |
| #96 | Typo in repo metadata | Not a code change. A minute. |

### Needs discussion

| Issue | What | Notes |
|---|---|---|
| #70 | `%option c++` + `%option noyywrap` linker error (multiple definition) | Real flex-side bug, self-contained, but touches the same `fl` library split that #6 would be closed on. Effort unknown until reproduced — suggest timeboxing a repro before committing it to this release. |
| #43 / #93 | Packaging: `find_package(FLEX)` cannot locate `FlexLexer.h` because the Chocolatey shim puts the exe elsewhere | A packaging fix, and a release is the natural moment for it. #93 (Chocolatey still on 2.5.25) is release mechanics regardless — see E.7. |

### Recommended out — post-release or upstream

| Issue | What | Why not now |
|---|---|---|
| #90 | `api.prefix{…}` breaks `yyerror()` | Upstream bison semantics, not a Windows port bug. Route to #106. |
| #47 | Colored output via textstyle | A feature, and a new dependency. |
| #72, #78, #69, #32, #34 | Support and documentation threads | Some are probably closeable with an answer rather than a change. Worth a sweep, but not a release blocker. |
| #16 | Restructuring the repo | Meta, open-ended. |

---

## D. Open pull requests — each needs a verdict

An unanswered PR queue is what makes a release look like a drive-by. These should all be answered
before the tag, even where the answer is "no".

| PR | Author | Size | What | Suggested verdict |
|---|---|---|---|---|
| #108 | @Croydon | +1/-1 | README status badge for the default branch | Check first: `a746398` on `dev` already retargeted the badge at the current AppVeyor project, so this may be superseded. Merge or close with the explanation. |
| #99 | @Febbe | +40/-47 | Further CMake cleanup | Needs review against the CMake work already on `dev`; conflicts likely. |
| #92 | @jonnysoe | +123/-1 | VS Code support | A feature. In or out? |
| #71 | @dpasukhi | +5/-5 | Empty lines with flex `-L` under GCC/Clang | Small, probably mergeable. |
| #59 | @KOLANICH | +17/-6 | MinGW-w64 compatibility | The project is MSVC-only by charter (the root `CMakeLists.txt` warns when `MSVC` is unset). Decline with a reason, or park behind an explicit "unsupported toolchain" note. |

---

## E. Release mechanics

Ordered; each step depends on the one before it.

1. Bump `project(winflexbison VERSION 2.5.26 …)` in the root `CMakeLists.txt`.
2. Change `### unreleased` to `### version 2.5.26` in `changelog.md`, and reorder so the LF
   behavior change leads, followed by the correctness fixes, then the build/test work.
3. README pass — version references and badge.
4. Green AppVeyor run on `dev` with the final content (ctest gate plus the autotest job).
5. Merge `dev` into `master`; tag `v2.5.26`.
6. GitHub release with the zips, keeping the `win_flex_bison-latest.zip` link-freeze convention
   described in the 2.5 changelog entry.
7. Chocolatey package refresh (#93) and the winget manifest — the latter is what the reporters on
   #97 are actually installing.
8. Close #97, #100, #17, #6, plus whatever lands from section C.

---

## F. Line endings — full review

Prompted by the recollection that LF/CRLF has caused trouble here before. It has. Everything
below is verified against the git history, the shipped release binaries, and byte counts of real
output — not inferred from the source.

### What actually happened

| When | What |
|---|---|
| 2018-09-20 | `a32e862` (bison) and `31807e4` (flex) opened every generated output `"wb"`. Shipped as the 2.5.16 changelog line: *"write output flex/bison files in binary mode 'wb' that means use '\n' EOL not '\r\n'"*. |
| 2018-10-02 | `a3a2d74` additionally forced `stdout`/`stderr` to binary. |
| 2020-10-26 | `be28ee7` put that behind `WINFLEXBISON_BINARY_OUTPUT=Y`. PR #65 asked outright, *"Should we wrap the binary mode in an environment variable or can it go in as-is?"* — it was wrapped as a precaution. No bug report is attached to it. |
| 2019-02 (v2.5.17) | **The bison 3.3.1 re-vendor silently dropped the bison `"wb"` patches.** `scan-skel.c` — the file that writes the generated parser and header — went back to `"w"` and stayed there through v2.5.25. `print.c` (`.output`) and `print-graph.c` (`.dot`) reverted too. `print-xml.c` happened to survive. |
| flex | Never re-vendored after 2.6.4, so its 2018 `"wb"` patches are still intact. |

So LF output shipped in exactly one release — **2.5.16** — and every release from 2.5.17 to
2.5.25 has written CRLF parsers.

### Measured, not assumed

Same grammar through the shipped `win_flex_bison-2.5.25.zip` binary and the current `dev` build,
counting CRLF and LF bytes:

| Output | 2.5.25 (shipped) | dev |
|---|---|---|
| `.tab.c` parser | CRLF (1236) | LF (1236) |
| `.tab.h` header | CRLF (75) | LF (75) |
| `.output` report | CRLF (51) | LF (51) |
| `.dot` graph | CRLF (21) | LF (21) |
| `.xml` report | **LF (94)** | LF (94) |
| win_flex `.c` / `.h` | LF | LF (byte-identical) |

Two things fall out. The claim in the changelog is correct — the change is real and it is
bison-only, flex output does not move a byte. And 2.5.25 was *internally inconsistent*: four
output kinds CRLF, the XML report LF, because that one `"wb"` survived the re-vendor by luck.

### Why the fix is built the way it is

`f42c460` deliberately centralizes the binary mode in `xfopen()` (writes only; reads stay text so
CRLF *input* grammars still get their CR stripped) rather than re-patching each call site. That is
the right call precisely because the per-call-site version is what got lost in 2019, silently, and
stayed lost for nine releases.

### Follow-ups this review turned up

1. **Reword the changelog entry and release note.** "now writes LF instead of CRLF" reads as a new
   policy. It is a repair of a regression that has been shipping since 2.5.17. Say so — and say
   that `--xml` output is unchanged, since it was already LF.
2. **Decide what `--update` / `--fixit` should do to the user's own grammar.** `fixits.c` reads the
   backup in text mode and writes through `xfopen(input, "w")`, which is now forced binary. Verified:
   a CRLF `.y` comes back **entirely LF**, not just on the lines that changed; the `.y~` backup keeps
   CRLF. This is the user's source file, not generated output, so the argument for forcing LF is much
   weaker there. Note this is not a regression anyone can have hit — `--update` was broken on Windows
   before this release (reproduced with the 2.5.25 binary: *"cannot backup: Permission denied"*) — but
   now that it works, it silently reformats the whole file. Suggested: exempt the fixits output.
3. **~~Add a gating test for the EOL guarantee.~~ DONE (2026-08-18).** Nothing in the CTest gate
   checked it: the golden-diff harness normalizes CRLF to LF before comparing
   (`tests/bison/run_bison_test.cmake`), the compile-run tests pass under either convention, and the
   only thing that would catch it — the MSYS2 autotest — is `allow_failures` in `.appveyor.yml`,
   i.e. deliberately non-gating. Added `winflexbison.bison_output_is_lf` and
   `winflexbison.flex_output_is_lf` (`tests/winflexbison/run_eol_test.cmake`): every output kind
   bison can write (`.tab.c`, `.tab.h`, `.output`, `.dot`, `.xml`) plus win_flex's `.c`/`.h` must
   contain no CR byte at all. Pure CMake, no MSYS2, runs in every cell, cannot skip.

   Verified in both directions, which is the part that matters: it passes on `dev`, and it **fails
   on the shipped 2.5.25 binary** — `g.tab.c contains 1327 CR byte(s)` — so it demonstrably catches
   the regression it exists for.

   A second pass feeds win_bison a grammar stored with CRLF (`tests/winflexbison/cases/crlf_input.y`,
   held CRLF by a local `.gitattributes` against the root's `* text=auto`) to guard the other half of
   the fix: reads stay in text mode, so a Windows-authored grammar must still work without leaking
   CRs into the parser. The test checks the fixture really is CRLF before using it, so a normalizing
   checkout fails loudly instead of quietly repeating the first pass.

   One trap worth recording: the CRLF fixture is committed rather than generated, because CMake's
   `file(WRITE)` translates LF to CRLF on Windows — writing `"\r\n"` from a script yields `"\r\r\n"`,
   and bison then correctly copies those stray lone CRs into the parser as ordinary text. That first
   draft of the test reported a tool bug that did not exist.
4. **Record the trap in `docs/specs/04-port-change-catalog/`** so the next bison upgrade does not
   drop the `xfopen` patch the way 2.5.17 dropped its predecessors.
5. **Clarify `WINFLEXBISON_BINARY_OUTPUT`.** It now governs only `stdout`/`stderr`; generated files
   are unconditionally LF regardless of it. Either document that scope or reconsider the variable.

---

## The two decisions that gate everything else

1. **How much of section C goes in?** Current recommendation: #95, #73 and #96 in; #70 timeboxed
   to a repro before it is promised; #43 folded into the packaging step.
2. **Are the section D pull requests resolved before or after the tag?** Recommendation: before.
3. **Does `--update` keep the user's CRLF?** (Section F.2.) This one is a release blocker in the
   sense that it changes what a shipped feature does to a file the user owns, and it is much
   cheaper to settle now than to change again in 2.5.27. Recommendation: exempt the fixits output
   from the binary forcing, and add F.3's gating EOL test in the same change.
