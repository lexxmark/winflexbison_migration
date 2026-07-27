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

Because only the tree at each pinned commit matters, the baselines clone shallow by default.

## Getting it

```sh
git clone <url> winflexbison_migration
cd winflexbison_migration
git submodule update --init            # top level only -- see the warning below
```

To build the port and nothing else, one submodule is enough:

```sh
git submodule update --init winflexbison
```

The `upstream/` mirrors clone shallow (`shallow = true` in `.gitmodules`) — only the tree at each
pinned commit, which is all a diff needs. All four together take about 20 seconds and ~70 MB.
If you need tags or history in one of them — `git describe`, `ls-tree <tag>`, or diffing two
upstream releases — undo it for that mirror only:

```sh
git -C upstream/m4 fetch --unshallow --tags
```

> **Do not use `git clone --recurse-submodules` here.** It is equivalent to
> `git submodule update --init --recursive`, and `upstream/bison` carries submodules of its own
> (`gnulib`, `submodules/autoconf`). Recursing pulls those too — far more than this workspace
> needs, and on Windows the nested `.git/modules/…` paths can blow past the 260-character limit
> and fail the clone outright. Plain `--init`, as above, stops at the top level.

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
