# brainmaze-sphinx

Shared tooling for the [BrainMaze](https://github.com/bnelair) package family.

Currently this repo hosts the **reusable GitHub Actions workflows** used by every
BrainMaze package so testing and releasing are defined once and consumed everywhere.
The shared Sphinx documentation theme will be added here in a later phase.

## Reusable workflows

### `test.yml`

Runs the package's own `pytest` suite on one job per OS (Ubuntu, macOS, Windows).

Consume it from a package repo (`.github/workflows/ci.yml`):


```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    uses: bnelair/brainmaze-sphinx/.github/workflows/test.yml@main
    with:
      package-import-name: brainmaze_utils   # optional, only for logging
```

Requirements on the consuming package:
- a `pyproject.toml` with a `test` optional-dependency extra that includes `pytest`,
- an importable test suite discoverable by `python -m pytest`.

### `release.yml`

`pyproject.toml` is the **single source of truth** for the version. This workflow
bumps `[project].version`, commits it to `main`, tags `vX.Y.Z`, builds, publishes to
PyPI, and creates a GitHub Release.

Consume it from a package repo (`.github/workflows/release.yml`):

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      bump:
        description: "Version bump"
        type: choice
        options: [patch, minor, major]
        default: patch
jobs:
  release:
    uses: bnelair/brainmaze-sphinx/.github/workflows/release.yml@main
    with:
      bump: ${{ inputs.bump }}
      pypi-auth: trusted          # or: token
    secrets: inherit              # needed for the API-token publish path
    permissions:
      contents: write
      id-token: write
```

Trigger a release from the package repo's **Actions → Release → Run workflow**, and
pick `patch` / `minor` / `major`.

#### PyPI authentication

- `pypi-auth: trusted` (default) uses [PyPI Trusted Publishing](https://docs.pypi.org/trusted-publishers/)
  via OIDC — no stored secret. Configure a trusted publisher on each PyPI project
  once: repo `bnelair/<package>`, workflow `release.yml`.
- `pypi-auth: token` uses the `PYPI_Token_General` secret (passed through via
  `secrets: inherit`) with the standard PyPI upload action.

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
