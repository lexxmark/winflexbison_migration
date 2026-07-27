# 06 — Validation Gate

**Task:** Define the acceptance gate an upgrade must pass before merge.

## 1. Build green — all configurations

Build must succeed for every supported toolchain/config. At minimum:

```
# from winflexbison/
buildVS2022.bat            # VS2022 generator, Release + package
# and via presets:
cmake --preset x64-Release   && cmake --build --preset x64-Release
cmake --preset x64-Debug     && cmake --build --preset x64-Debug
```

CI parity (must also pass): AppVeyor builds VS2022 (x64) and VS2026 (x64/Win32) with MSVC and
`USE_STATIC_RUNTIME=ON`, and runs the **CTest gate in the VS2022/x64/Release cell**; GitHub
Actions builds `clang-cl` via Ninja (`.github/workflows/os_windows.yaml`) without tests. A local
green build is necessary but not sufficient — the `__extension__=` define is
MSVC-only-not-clang, so clang-cl can break where MSVC passes.

Note that CI's cell is *not* the same configuration as `runtests.bat`: AppVeyor passes
`USE_STATIC_RUNTIME=ON`, which registers the extra `winflexbison.static_runtime_no_vcruntime`
test (132 tests rather than 131). To reproduce the CI gate exactly, configure locally with
`-DUSE_STATIC_RUNTIME=ON` and run ctest against that build tree.

Artifacts to confirm exist after a Release build:
- `winflexbison/bin/Release/win_flex.exe`, `win_bison.exe` (+ `data/`)
- the CPack zip `win_flex_bison-*.zip`

Smoke test the binaries:
```
bin/Release/win_flex.exe  --version     # expect the NEW flex version
bin/Release/win_bison.exe --version     # expect the NEW bison version
```

## 2. Warning baseline

The build is **not** warning-free today; treat the current set as a baseline so *new* warnings
stand out. Known-benign, expected warnings on MSVC:
- **C4267** `size_t`→`int`/narrower conversions — widespread across `bison/src` (e.g.
  `state-item.c`, `state.c`, `tables.c`, `symtab.c`) and gnulib. Benign on LLP64 for these code
  paths.
- **C4307 / C4308 / C4018** signed/unsigned overflow & mismatch in `bison/src/strversion.c`,
  `symtab.c`.
- **C4090** `const` qualifier drops in `flex/src/filter.c`.

Verified baseline reference: on the current tree these are the only warnings, and the build
produced `win_flex 2.6.4`, `win_bison 3.8.2`, `win_flex_bison-dev-2.5.25.zip`. After an upgrade,
**diff the warning list** against this set; investigate anything new (a new C4013 "undefined
function" or C4020 often signals a missing gnulib shim or a POSIX call needing a port hunk).

## 3. Tests green

- **Flex** — the CTest suite ([03](../03-test-adoption/spec.md)) must pass:
  ```
  ctest --test-dir build -C Release --output-on-failure
  ```
  Pass = each scanner exits 0 / prints `TEST RETURNING OK`; expected skips (`77`) are allowed
  (e.g. `pthread`, bison-dependent tests when `win_bison` is not wired). No unexpected failures.
- **Bison** — run the adopted `.at` subset / `testsuite` per the chosen bison strategy in
  [03](../03-test-adoption/spec.md). Record which groups ran vs. were skipped.

**Tests must be green before merge**, run against the newly built binaries (not a stale `bin/`).

## 4. Sign-off checklist

- [ ] New baselines added under `upstream/` with tag→SHA recorded (incl. both gnulib SHAs)
      ([01](../01-version-inventory/spec.md), [02](../02-baseline-mirrors/spec.md)).
- [ ] Port catalog ([04](../04-port-change-catalog/spec.md)) re-verified against regenerated diffs;
      any new port hunks added to the catalog.
- [ ] All hand-maintained patches (categories 3–6) replayed and present in the new tree.
- [ ] Committed generated files regenerated from the new upstream.
- [ ] `changelog.md` + version stamps updated; `CMakeLists.txt` version bumped if releasing.
- [ ] Build green: VS2022 Release+Debug locally; CI (AppVeyor MSVC matrix + clang-cl) green —
      including AppVeyor's CTest run in the VS2022/x64/Release cell.
- [ ] Warning list diffed against the baseline (§2); no unexplained new warnings.
- [ ] `--version` smoke test shows the new versions.
- [ ] Flex CTest suite green; bison test subset green; skips documented.
- [ ] Test set refreshed from the new baseline.
