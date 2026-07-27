# winflexbison_migration

A working area for maintaining and upgrading [**winflexbison**][wfb] — the Windows port of GNU
Flex and Bison — not the port itself.

The port lives in the `winflexbison/` submodule and is developed at
[lexxmark/winflexbison][wfb]. **If you just want `win_flex.exe` / `win_bison.exe`, go there** —
this repository adds nothing for consumers of the tools.

What it adds is the machinery for a specific recurring job: adopting a new upstream Flex, Bison,
or M4 release into the port without losing the Windows patch set. That means keeping pristine
upstream checkouts to diff against, and a written playbook for the upgrade.

## Layout

```
winflexbison_migration/
├─ winflexbison/   the port itself (submodule -> lexxmark/winflexbison)
├─ upstream/       pristine upstream mirrors, pinned to the vendored versions (submodules)
│  ├─ flex/        westes/flex
│  ├─ bison/       akimd/bison        (carries the autotest suite)
│  ├─ m4/          savannah/m4
│  └─ gnulib/      savannah/gnulib
└─ docs/specs/     the upgrade playbook
```

The Windows patch set is not tracked as a patch series — it is recovered by diffing the vendored
sources against the matching `upstream/` baseline. That is why the baselines are pinned to the
*vendored* versions rather than tracking upstream `master`, and why they are submodules: the
repository records a URL and a commit per baseline, never the clones' contents.

## Getting it

The submodules are the whole point, so clone recursively:

```sh
git clone --recurse-submodules <url> winflexbison_migration
```

Already cloned without them? Everything will look empty until:

```sh
git submodule update --init
```

## Pinned baselines

Each mirror sits at the exact tag matching the version currently vendored in the port.

| Component | Upstream | Ref | Commit |
|---|---|---|---|
| `upstream/flex` | github.com/westes/flex | `v2.6.4` | `ab49343` |
| `upstream/bison` | github.com/akimd/bison | `v3.8.2` | `9beba19` |
| `upstream/m4` | git.savannah.gnu.org/git/m4 | `v1.4.19` | `445afe0` |
| `upstream/gnulib` | git.savannah.gnu.org/git/gnulib | bison-side pin | `7818455` |
| `upstream/gnulib` | git.savannah.gnu.org/git/gnulib | m4-side pin | `3639c57` |

`akimd/bison` is used rather than the release tarball because it carries the `tests/` autotest
suite. gnulib has two relevant commits — the ones bison and m4 pin as their own submodule — and
the checkout defaults to the bison-side pin.

## Where to start

**[`docs/specs/README.md`](docs/specs/README.md)** — the playbook index. The six specs cover
version inventory, the baseline mirrors, test adoption, the Windows port-change catalog, the
upgrade runbook, and the validation gate.

For building and testing the port, see `winflexbison/README.md` and the test suites under
`winflexbison/tests/`.

[wfb]: https://github.com/lexxmark/winflexbison
