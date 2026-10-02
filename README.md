# brainmaze-sphinx

Shared tooling for the [BrainMaze](https://github.com/bnelair) package family.

Currently this repo hosts the **reusable GitHub Actions workflows** used by every
BrainMaze package so testing and releasing are defined once and consumed everywhere.
The shared Sphinx documentation theme will be added here in a later phase.

## Reusable workflows

### `test.yml`

Runs the package's own `pytest` suite on Ubuntu, macOS and Windows, for each
Python/numpy combination in `versions`.

Consume it from a package repo (`.github/workflows/ci.yml`):

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    uses: bnelair/brainmaze-sphinx/.github/workflows/test.yml@main
    with:
      package-import-name: brainmaze_utils   # optional, only for logging
      # Family default: oldest supported stack + newest stack.
      versions: '[{"python": "3.10", "numpy": "<2"}, {"python": "3.12", "numpy": ">=2"}]'
```

| input | default | meaning |
|---|---|---|
| `versions` | `""` | Non-empty JSON list of `{"python": "3.x", "numpy": "<pip specifier>"}`, values as JSON strings (`"3.10"`, not `3.10`). `"numpy": ""` = latest; a missing `"python"` falls back to `python-version`. numpy is installed binary-only, so pair `"<2"` with Python <= 3.12. Empty = one job per OS with `python-version` and latest numpy (old behaviour and check names). |
| `python-version` | `"3.10"` | Python when `versions` is empty or an entry has no `"python"`. |
| `package-import-name` | `""` | Prints `<pkg>.__version__` after install. |
| `dist-check` | `"warn"` | Artifact-content check (job `dist-contents`, see [`check-dist.yml`](#check-distyml)): `"error"` fails the run on an unwanted file, `"warn"` only annotates, `"off"` skips it. |
| `allowed-assets` | `""` | Whitespace-separated globs of non-code files the package may ship, relative to the artifact root, e.g. `brainmaze_torch/seizure_detection/_models/*.pt` (see [glob syntax](#check-distyml)). |
| `required-assets` | `""` | Globs (same syntax) that must each match at least one file in the wheel **and** the sdist; they are allowed too. A missing one fails the check even with `dist-check: warn`. |
| `dist-packages` | `""` | Top-level packages allowed in the artifacts; empty = the distribution name with `-` → `_`. |

Requirements on the consuming package:
- a `pyproject.toml` with a `test` optional-dependency extra that includes `pytest`,
- an importable test suite discoverable by `python -m pytest`,
- numpy-version support declared in `dependencies` that matches the matrix (every
  BrainMaze package must work on numpy 1.x **and** 2.x; see the family
  [compatibility policy](RELEASING.md#compatibility-policy)).

The tests run from the repository checkout (editable install), so the test suite and its
data stay **out** of the wheel and sdist (see [`check-dist.yml`](#check-distyml)).

### `check-dist.yml`

Family rule: the wheel **and** the sdist ship only the package code and the runtime
assets it needs; never tests, demos, docs or data (they are unnecessary size at
deployment). `check-dist.yml` builds (or downloads) both artifacts, prints their size and
largest files, and checks them against an **allowlist**:

- Inside the package (`dist-packages`, default: the distribution name with `-` → `_`)
  only `*.py`, `*.pyi` and `py.typed` are allowed, plus files matching `allowed-assets`
  or `required-assets`. Anything else (`.json`, `.npz`, `.png`, `README.md`, …) is flagged.
- Never allowed, even inside the package: a `tests/`, `test/`, `demo/`, `demos/`, `docs/`,
  `docs_src/`, `examples/` or `.github/` directory (any case), and test files
  (`conftest.py`, `test_*.py`, `*_test.py`).
- Outside the package only the package's **own** metadata is allowed: in the wheel
  `<dist>-<ver>.dist-info/` with its known files (`METADATA`, `WHEEL`, `RECORD`,
  `top_level.txt`, `entry_points.txt`, `licenses/…`); in the sdist `PKG-INFO`,
  `pyproject.toml`, `setup.cfg`, `setup.py`, `MANIFEST.in`, the exact names
  `README`/`LICENSE`/`LICENCE`/`COPYING`/`NOTICE` (optionally `.md`/`.rst`/`.txt`), and
  `<dist>.egg-info/` with its known files. Wheel `.data/` directories, foreign
  `*.dist-info`/`*.egg-info` and files like `requirements.txt` or `CHANGELOG.md` are flagged.
- sdist members that are symlinks, hardlinks or not regular files, and member names that
  are absolute or contain `..`, are flagged.
- Every `required-assets` glob must match at least one file in **each** artifact (wheel
  and sdist). This catches a runtime asset (e.g. brainmaze-torch's trained models) that
  silently dropped out of the wheel, which the tests cannot notice because they import
  the package from the source tree. A missing required asset **always fails** (also in
  `warn` mode): it means the published package would be broken.
  A file that is itself flagged (e.g. in a `tests/` directory or with a control character
  in its name) does not count as the required asset.
- Files admitted by `allowed-assets`/`required-assets` are listed per artifact in the log
  (first 20), so an over-broad glob is visible; a glob whose last segment is only wildcards
  (`**`, `*`, `pkg/**`, `pkg/*`) raises a warning. File names are escaped everywhere they
  are printed, and a name containing a control character (e.g. a newline) is flagged.

Glob syntax (`allowed-assets`, `required-assets`): paths relative to the artifact root,
matched **per path segment**. `*`, `?` and `[…]` never cross `/`
(`pkg/_models/*.pt` does not match `pkg/_models/sub/x.pt`); a segment `**` matches zero
or more directories (`pkg/assets/**/*.json`). Matching is case-sensitive.

It runs automatically: `test.yml` checks a fresh build on every push/PR (job
`dist-contents`, built with the same `python-version` as the release build), and
`release.yml` checks the exact `dist` artifact before the caller's `publish` job may run.
Callers set the inputs on both:

```yaml
# ci.yml
    uses: bnelair/brainmaze-sphinx/.github/workflows/test.yml@main
    with:
      package-import-name: brainmaze_torch
      dist-check: error
      # only if the package needs non-code files at runtime (here: the trained models)
      required-assets: >-
        brainmaze_torch/seizure_detection/_models/modelA_paper.pt
        brainmaze_torch/seizure_detection/_models/modelB_full.pt
# release.yml (the `build` job): the same inputs
    uses: bnelair/brainmaze-sphinx/.github/workflows/release.yml@main
    with:
      dist-check: error
      required-assets: >-
        brainmaze_torch/seizure_detection/_models/modelA_paper.pt
        brainmaze_torch/seizure_detection/_models/modelB_full.pt
```

The default `"warn"` keeps callers that have not cleaned up their packaging green (the
offending files show as warnings); every family package should set `dist-check: error`,
and the default will become `"error"` once all of them do.
To keep artifacts clean, restrict setuptools discovery to the package
(`include = ["<pkg>", "<pkg>.*"]`, exclude `<pkg>.tests*`), set
`include-package-data = false` with an explicit `[tool.setuptools.package-data]` for
required assets (and list them in `required-assets`), and `prune` everything else from
the sdist in `MANIFEST.in`.

**Where the check is enforced.** No family repository has required status checks on
`main` (the rulesets only require one approving review). So a failing `dist-contents`
job on a PR is visible but does **not** block the merge: reviewers must not merge a PR
whose `dist-contents` check is red. The hard gate is the release: with
`dist-check: error` (or a missing required asset) `release.yml` fails before the caller's
`publish` job, so nothing reaches PyPI. Making `test / dist-contents / check` a required
status check in a repo's ruleset is an optional, per-repo maintainer decision.

**Nested workflows always resolve from `main`.** `test.yml` and `release.yml` call
`check-dist.yml@main` (and `release.yml` calls `test.yml@main`). GitHub has no "same ref"
for nested reusable workflows across repositories, so a caller that pins this repo to a
tag or SHA still gets the nested workflows from `main`. Consequences: (a) keep the
inputs of `check-dist.yml` backward compatible (new inputs need defaults); (b) a change to
these workflows can only be exercised before merge on a temporary branch whose nested
`@main` refs point at that branch (see [RELEASING.md](RELEASING.md#testing-changes-to-the-shared-workflows)).

### Release workflows (`prepare-release.yml` + `release.yml`)

The full maintainer guide (one-time setup, step-by-step release, dependency order,
compatibility policy, troubleshooting) is **[RELEASING.md](RELEASING.md)** in this repo.
In short:

1. **Prepare release** (manual, `workflow_dispatch`, pick `patch` / `minor` / `major`):
   the shared `prepare-release.yml` bumps `[project].version` in `pyproject.toml` on a new
   `release/bump-X.Y.Z` branch cut from `main` and opens a **"Release vX.Y.Z"** PR as
   `github-actions[bot]`. It never pushes to `main`.
2. A maintainer reviews it (the diff must be the version line only; CI does not run on a
   bot-opened PR) and **squash-merges** it. The merge starts step 3 automatically.
3. **Release** (`release.yml` in the package repo, runs on every push to `main`):
   - the shared `release.yml` checks whether tag `vX.Y.Z` already exists. If it does
     (an ordinary merge), nothing happens. If not, it runs `test.yml` (same `versions`
     matrix if passed), builds sdist + wheel and uploads them as the `dist` artifact;
   - the package repo's own **top-level** `publish` job downloads `dist`, publishes to
     PyPI with **Trusted Publishing** (OIDC; must be top-level, a reusable workflow
     cannot publish) or, in brainmaze-zmq and brainmaze-torch, the organisation API token
     `PYPI_Token_General`, and only then pushes tag `vX.Y.Z` and creates the GitHub Release
     with generated notes. The caller's jobs are least-privilege (only `publish` gets
     `contents: write`, plus `id-token: write` for Trusted Publishing) and serialised per
     repo. If a run fails
     part-way, re-run it or finish by hand; never wait for the next push, which would
     release a different commit ([recovery](RELEASING.md#recovering-from-a-failed-release)).
4. **Version guard** (`version-guard.yml` in the package repo, entirely local, on PRs to
   `main`/`dev`) flags any PR that changes `[project].version`, unless it is the bot's
   `release/bump-*` PR into `main` that changes nothing else. It is advisory (not a
   required check; see [RELEASING.md](RELEASING.md#version-guard-is-advisory)).

Package-repo callers (copy these verbatim from
[brainmaze-eeg](https://github.com/bnelair/brainmaze-eeg/tree/main/.github/workflows)):
`prepare-release.yml`, `release.yml`, `version-guard.yml`.

Shared-workflow inputs:

| workflow | input | meaning |
|---|---|---|
| `prepare-release.yml` | `bump` (required) | `patch` / `minor` / `major` |
| `release.yml` | `versions` | forwarded to `test.yml` |
| `release.yml` | `python-version` | forwarded to `test.yml` **and** the interpreter used to build the distributions |
| `release.yml` | `dist-check`, `allowed-assets`, `required-assets`, `dist-packages` | artifact-content check of the built `dist` artifact (see [`check-dist.yml`](#check-distyml)); with `dist-check: error` a violation blocks the publish |
| `release.yml` | `bump` | **deprecated, ignored**. Only the input is tolerated: old callers that also pass `pypi-auth` or `secrets: PYPI_Token_General` (no longer declared) fail validation, and the shared workflow no longer publishes. Such callers must migrate to the caller set below. |
| `release.yml` outputs | `released`, `version` | `'true'` when an untagged version was built |

> **Note — callers pin `@main`.** A change merged here affects every package's CI and
> releases immediately. Keep changes backward compatible (new inputs must have
> defaults), or tag this repo and pin callers to a tag.

### `docs.yml`

Builds the package's Sphinx docs and publishes them to the repo's `gh-pages`
branch, hosting two versions on one GitHub Pages site:

- push to `main` -> site **root** (released docs, e.g. `bnelair.github.io/brainmaze-eeg/`)
- push to `dev` -> **`/dev/`** subpath (live preview, e.g. `bnelair.github.io/brainmaze-eeg/dev/`)

Consume it from a package repo (`.github/workflows/docs.yml`):

```yaml
name: Docs
on:
  push:
    branches: [main, dev]
  workflow_dispatch:
permissions:
  contents: write
jobs:
  docs:
    uses: bnelair/brainmaze-sphinx/.github/workflows/docs.yml@main
```

Requirements on the consuming package:
- Sphinx sources at `docs_src/source` (override via the `source-dir` input),
- optional `docs_src/requirements.txt` (installed if present); the package itself
  is installed with `pip install -e .` so autodoc can import it,
- **one-time repo setting:** Settings -> Pages -> Source = *Deploy from a branch* ->
  `gh-pages` / `/ (root)`. The `gh-pages` branch is created by the first run.

Note: `keep_files: true` keeps the sibling version (root vs `/dev`) intact across
deploys; a page removed from the sources lingers until overwritten.

## Versioning contract

Each package declares a **static** version in `pyproject.toml`:

```toml
[project]
version = "1.0.4"
```

and exposes it at runtime from installed metadata:

```python
from importlib.metadata import version, PackageNotFoundError
try:
    __version__ = version("brainmaze-utils")   # distribution name
except PackageNotFoundError:
    __version__ = "0.0.0"
```

No `_version.py`, no `setuptools_scm`, no `setup.py`.
