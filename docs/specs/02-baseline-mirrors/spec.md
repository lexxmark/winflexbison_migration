# 02 — Baseline Mirrors (`upstream/`)

**Task:** Maintain pristine upstream checkouts, pinned to the exact tags matching the vendored
versions, so the Windows patch set can be extracted by diffing vendored-vs-baseline.

## Location & rationale

Baselines live at the **workspace root**, outside the winflexbison git repo:

```
winflexbison_migration\   ← workspace repo (this checkout)
├─ winflexbison\   ← the project (submodule)
├─ docs\specs\     ← this playbook
└─ upstream\       ← baseline mirrors (submodules; no content in the project's history)
   ├─ flex\        westes/flex     @ v2.6.4
   ├─ bison\       akimd/bison     @ v3.8.2   (carries tests/*.at)
   ├─ m4\          savannah/m4     @ v1.4.19
   └─ gnulib\      savannah/gnulib @ bison-pin (default); m4-pin also fetched
```

The workspace root is itself a git repository, and each baseline is registered there as a
**submodule** (see `.gitmodules`) — so the workspace records only a gitlink per baseline (a URL
plus the pinned commit), never the clones' contents. The pins below are therefore reproducible
from a fresh `git clone --recurse-submodules` of the workspace, and the working trees remain a
**working area** that does not need to be backed up.

## Current pinned baselines

| Subfolder | Upstream | Ref | Resolved SHA |
|---|---|---|---|
| `upstream/flex` | github.com/westes/flex | `v2.6.4` | `ab49343b08c933e32de8de78132649f9560a3727` |
| `upstream/bison` | github.com/akimd/bison | `v3.8.2` | `9beba1919cad5dd08b0cac277c27896808719e4b` |
| `upstream/m4` | git.savannah.gnu.org/git/m4 | `v1.4.19` | `445afe00b62d8a7bee109faf3b96edf0c97b7a85` |
| `upstream/gnulib` (bison-side) | git.savannah.gnu.org/git/gnulib | — | `7818455627c5e54813ac89924b8b67d0bc869146` |
| `upstream/gnulib` (m4-side) | git.savannah.gnu.org/git/gnulib | — | `3639c57a970191e0bf7a9789bd1341786d0255a1` |

`akimd/bison` is the Bison maintainer's mirror and is used because it carries the `tests/`
autotest suite that the canonical release tarball layout keeps (see
[03](../03-test-adoption/spec.md)); the `v3.8.2` tag there resolves to the release commit.

## How these were created (reproducible commands)

```sh
cd <workspace-root>          # the winflexbison_migration checkout
mkdir -p upstream && cd upstream

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

> Windows note: do **not** try to `mv` an existing live git clone into `upstream/` — read-only `.git`
> pack files cause partial-move / access-denied failures. Clone fresh instead.

To diff against the **m4-side** gnulib (for `common/m4/lib`), switch the same clone:
`git -C upstream/gnulib checkout 3639c57a970191e0bf7a9789bd1341786d0255a1` (or add a worktree).

## Verifying a baseline is correct

```sh
git -C upstream/flex  describe --tags   # v2.6.4
git -C upstream/bison describe --tags   # v3.8.2
git -C upstream/m4    describe --tags   # v1.4.19
git -C upstream/gnulib rev-parse HEAD   # 7818455627c5e54813ac89924b8b67d0bc869146
```

> **These need tags, which a default clone does not have.** The mirrors are `shallow = true` in
> `.gitmodules`, so `submodule update --init` fetches the pinned commit alone — `describe --tags`
> then fails with *"No names found"* even though the checkout is correct. `rev-parse HEAD` still
> works and is sufficient to confirm the pin. To use the tag-based commands (here, and the
> `ls-tree <tag>` recipe in [01](../01-version-inventory/spec.md), and any release-to-release
> `diff`), deepen just the mirror you need:
>
> ```sh
> git -C upstream/m4 fetch --unshallow --tags     # ~4s for m4
> ```

**Spot-check the version, not just the tag.** The pitfall this guards against: a clone left on a
branch tip is the *wrong* tree. For example, a `westes/flex` clone at `master` is ~589 commits past
`v2.6.4` and contains `src/skeletons.c`, `src/c99-flex.skl`, `src/cpp-flex.skl` — files that do not
exist at `v2.6.4`. Confirm the baseline is right:

```sh
# at v2.6.4 these MUST be absent; if present, you are on the wrong ref:
ls upstream/flex/src/skeletons.c upstream/flex/src/c99-flex.skl 2>/dev/null && echo "WRONG REF"
```

## Adding the NEW target version during an upgrade

When adopting version *N+1*, keep the old baseline (needed to confirm the current patch set) and
add the new one alongside. Two equivalent options:

- **Second clone** — `git clone --branch <newtag> … upstream/flex-<newver>` (simplest, isolated).
- **Worktree** — `git -C upstream/flex fetch --tags && git -C upstream/flex worktree add ../flex-<newver> <newtag>` (shares object store, faster/smaller).

For each new component version, **re-run the gnulib submodule recovery** (`git ls-tree <newtag>
gnulib`) to pin the new gnulib commit, and record every new tag→SHA in
[01](../01-version-inventory/spec.md) and `changelog.md`.
