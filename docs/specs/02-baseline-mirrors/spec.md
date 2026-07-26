# 02 — Baseline Mirrors (`orig/`)

**Task:** Maintain pristine upstream checkouts, pinned to the exact tags matching the vendored
versions, so the Windows patch set can be extracted by diffing vendored-vs-baseline.

## Location & rationale

Baselines live at the **workspace root**, outside the winflexbison git repo:

```
C:\Users\azhon\source\repos\winflexbison_migration\
├─ winflexbison\   ← the project (git repo)
├─ docs\specs\     ← this playbook
└─ orig\           ← baseline mirrors (NOT committed to the project)
   ├─ flex\        westes/flex     @ v2.6.4
   ├─ bison\       akimd/bison     @ v3.8.2   (carries tests/*.at)
   ├─ m4\          savannah/m4     @ v1.4.19
   └─ gnulib\      savannah/gnulib @ bison-pin (default); m4-pin also fetched
```

The workspace root is not a git repository, so these large clones stay out of the project's
history automatically. They are a **working area**, reproducible from the commands below — they do
not need to be backed up or committed.

## Current pinned baselines

| Subfolder | Upstream | Ref | Resolved SHA |
|---|---|---|---|
| `orig/flex` | github.com/westes/flex | `v2.6.4` | `ab49343b08c933e32de8de78132649f9560a3727` |
| `orig/bison` | github.com/akimd/bison | `v3.8.2` | `9beba1919cad5dd08b0cac277c27896808719e4b` |
| `orig/m4` | git.savannah.gnu.org/git/m4 | `v1.4.19` | `445afe00b62d8a7bee109faf3b96edf0c97b7a85` |
| `orig/gnulib` (bison-side) | git.savannah.gnu.org/git/gnulib | — | `7818455627c5e54813ac89924b8b67d0bc869146` |
| `orig/gnulib` (m4-side) | git.savannah.gnu.org/git/gnulib | — | `3639c57a970191e0bf7a9789bd1341786d0255a1` |

`akimd/bison` is the Bison maintainer's mirror and is used because it carries the `tests/`
autotest suite that the canonical release tarball layout keeps (see
[03](../03-test-adoption/spec.md)); the `v3.8.2` tag there resolves to the release commit.

## How these were created (reproducible commands)

```sh
cd C:/Users/azhon/source/repos/winflexbison_migration
mkdir -p orig && cd orig

# flex / bison / m4 — shallow clone at the exact tag
git clone --branch v2.6.4  --depth 1 https://github.com/westes/flex.git       flex
git clone --branch v3.8.2  --depth 1 https://github.com/akimd/bison.git       bison
git clone --branch v1.4.19 --depth 1 https://git.savannah.gnu.org/git/m4.git  m4

# gnulib — recover the two pinned commits from the submodule gitlinks, then
# fetch ONLY those commits (a full gnulib clone is huge and unnecessary)
BISON_GNULIB=$(git -C bison ls-tree v3.8.2  gnulib | awk '{print $3}')   # 7818455...
M4_GNULIB=$(   git -C m4    ls-tree v1.4.19 gnulib | awk '{print $3}')   # 3639c57...
mkdir -p gnulib && cd gnulib
git init -q
git remote add origin https://git.savannah.gnu.org/git/gnulib.git
git fetch --depth 1 origin "$BISON_GNULIB" "$M4_GNULIB"
git checkout -q "$BISON_GNULIB"     # bison-side is the default checkout
```

> Windows note: do **not** try to `mv` an existing live git clone into `orig/` — read-only `.git`
> pack files cause partial-move / access-denied failures. Clone fresh instead.

To diff against the **m4-side** gnulib (for `common/m4/lib`), switch the same clone:
`git -C orig/gnulib checkout 3639c57a970191e0bf7a9789bd1341786d0255a1` (or add a worktree).

## Verifying a baseline is correct

```sh
git -C orig/flex  describe --tags   # v2.6.4
git -C orig/bison describe --tags   # v3.8.2
git -C orig/m4    describe --tags   # v1.4.19
git -C orig/gnulib rev-parse HEAD   # 7818455627c5e54813ac89924b8b67d0bc869146
```

**Spot-check the version, not just the tag.** The pitfall this guards against: a clone left on a
branch tip is the *wrong* tree. For example, a `westes/flex` clone at `master` is ~589 commits past
`v2.6.4` and contains `src/skeletons.c`, `src/c99-flex.skl`, `src/cpp-flex.skl` — files that do not
exist at `v2.6.4`. Confirm the baseline is right:

```sh
# at v2.6.4 these MUST be absent; if present, you are on the wrong ref:
ls orig/flex/src/skeletons.c orig/flex/src/c99-flex.skl 2>/dev/null && echo "WRONG REF"
```

## Adding the NEW target version during an upgrade

When adopting version *N+1*, keep the old baseline (needed to confirm the current patch set) and
add the new one alongside. Two equivalent options:

- **Second clone** — `git clone --branch <newtag> … orig/flex-<newver>` (simplest, isolated).
- **Worktree** — `git -C orig/flex fetch --tags && git -C orig/flex worktree add ../flex-<newver> <newtag>` (shares object store, faster/smaller).

For each new component version, **re-run the gnulib submodule recovery** (`git ls-tree <newtag>
gnulib`) to pin the new gnulib commit, and record every new tag→SHA in
[01](../01-version-inventory/spec.md) and `changelog.md`.
