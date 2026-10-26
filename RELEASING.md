# Releasing the BrainMaze packages

This guide covers every package in the family: how a release works, how to cut one, how to
recover when a release fails part-way, the one-time setup each repository needs, and the
rules that keep the packages compatible with each other. Each package repository has a short
`RELEASING.md` that points here.

It lives in `brainmaze-sphinx` because this public repo hosts the shared workflows it
describes. (The `brainmaze` meta-package repo is private, so a guide there would be a dead
link for everyone outside the organisation.)

## The family

Runtime dependencies as declared in each package's `pyproject.toml` on `main`, and the
latest release on PyPI (checked 2026-10-26):

| PyPI / repo | import name | latest release | runtime dependencies | docs |
|---|---|---|---|---|
| [`brainmaze-utils`](https://github.com/bnelair/brainmaze-utils) | `brainmaze_utils` | 3.0.0 | numpy>=1.24, scipy>=1.10, pandas>=2.0, python-dateutil, tqdm | <https://bnelair.github.io/brainmaze-utils/> |
| [`brainmaze-eeg`](https://github.com/bnelair/brainmaze-eeg) | `brainmaze_eeg` | 3.1.0 | brainmaze_utils>=3.0.0, numpy>=1.24, pandas>=2.2.0, pytz, python-dateutil, scikit-learn>=1.6.1, scipy>=1.10, tqdm. Optional extra `models` = brainmaze-eeg-models>=0.1.0 (on `main`, not yet in a release) | <https://bnelair.github.io/brainmaze-eeg/> |
| [`brainmaze-eeg-models`](https://github.com/bnelair/brainmaze-eeg-models) | `brainmaze_eeg_models` | 0.1.0 | numpy>=1.24, scipy>=1.10, onnxruntime>=1.19, brainmaze-utils>=3.0.0. Ships the trained models as `.onnx` files; no PyTorch (only its optional `export` extra, used to regenerate the ONNX files, needs torch, onnx and brainmaze-torch==0.2.0) | <https://bnelair.github.io/brainmaze-eeg-models/> |
| [`brainmaze-zmq`](https://github.com/bnelair/brainmaze-zmq) | `brainmaze_zmq` | 0.0.1 (old flow) | pyzmq, pythonping, tornado>=6.5 | <https://bnelair.github.io/brainmaze-zmq/> |
| [`brainmaze`](https://github.com/bnelair/brainmaze) (private repo) | `brainmaze` | never released from the repo (PyPI has only `0.0.0a0`) | the family packages, unpinned (meta-package); see below | — |
| [`brainmaze-torch`](https://github.com/bnelair/brainmaze-torch) | `brainmaze_torch` | **0.3.0, final. Retired; repo archived.** | brainmaze-eeg-models>=0.1.0 only: 0.3.0 is a compatibility shim over brainmaze-eeg-models (same API, no PyTorch). Use brainmaze-eeg-models instead. | <https://bnelair.github.io/brainmaze-torch/> (frozen) |

The meta-package on `main` still depends on brainmaze-utils, brainmaze-eeg, brainmaze-torch
and brainmaze-zmq. bnelair/brainmaze#4 (open; private repo)
replaces brainmaze-torch with brainmaze-eeg-models there; brainmaze-torch then stays only in
the meta repo's compatibility test matrix, where the 0.3.0 shim's test suite checks
brainmaze-eeg-models' API, because the archived repo cannot run its own CI any more.

History (brainmaze-torch): 0.2.0 depended on numpy>=1.24, scipy>=1.10 and torch>=2.3;
0.1.1 also pulled in brainmaze-utils, torchvision, torchaudio, pandas, scikit-learn,
matplotlib, pyyaml, tqdm, pytz, python-dateutil, setuptools, sphinx and sphinx_book_theme.
Its models were exported to ONNX and continue in brainmaze-eeg-models.

Shared CI and release logic lives in this repo
([`bnelair/brainmaze-sphinx`](https://github.com/bnelair/brainmaze-sphinx)) as reusable
workflows. In each package repo, `ci.yml`, `docs.yml` and `prepare-release.yml` are thin
callers of those workflows. Two caller workflows contain real logic of their own:

- `release.yml` calls the shared `release.yml` for guard + test + build, then runs its own
  **`publish` job** (download `dist`, upload to PyPI, push the tag, create the GitHub Release);
- `version-guard.yml` is **entirely local** (no shared workflow).

## How a release works

```
 maintainer                     GitHub Actions                                PyPI / GitHub
 ──────────                     ──────────────                                ─────────────
 Actions → "Prepare release"
   (patch | minor | major)  ──► bumps [project].version in pyproject.toml
                                on branch release/bump-X.Y.Z (cut from main)
                                and opens PR "Release vX.Y.Z" as github-actions[bot]
 review + squash-merge  ──────► the push to main triggers "Release" (release.yml)
                                automatically (there is no manual "Run workflow" for it):
                                 1. guard: is tag vX.Y.Z new?  no → stop (ordinary merge)
                                 2. run the test suite (test.yml matrix)
                                 3. build sdist + wheel  → artifact "dist"
                                    and check its contents (package only, no
                                    tests/demos/docs/data; check-dist.yml)
                                 4. publish job (top level)                    ──► PyPI X.Y.Z
                                 5. push tag vX.Y.Z, create GitHub Release   ──► Release + notes
```

Key properties:

- **`pyproject.toml` is the single source of truth for the version** (static
  `[project].version`, `X.Y.Z`). At runtime packages read it from installed metadata
  (`importlib.metadata.version(...)`). No `_version.py`, `setuptools_scm` or `setup.py`.
- **Nothing pushes to a protected branch.** The bump arrives as a PR and goes through review
  (how much review is actually enforced depends on each repo's ruleset; see
  [Branch protection and review](#branch-protection-and-review)).
- **The version line is owned by the release automation.** The *Version guard* check flags
  any PR that changes `[project].version`, unless it is the bot's `release/bump-*` PR into
  `main` (and then it checks that nothing else changed). It is **advisory**: no ruleset
  requires it today (see [Version guard is advisory](#version-guard-is-advisory)). Keeping
  the version line out of feature work is what keeps `dev → main` merges free of version
  conflicts under squash merges.
- **The tag and GitHub Release are created only after a successful PyPI upload.** A run that
  fails part-way must be finished on **that** run (or by hand), never by waiting for the next
  push to `main`; see [Recovering from a failed release](#recovering-from-a-failed-release).
- **Publishing happens in the package's own top-level `release.yml`** because PyPI Trusted
  Publishing cannot publish from a reusable workflow (the OIDC identity has to be the
  package repo's `release.yml`); token-publishing repos use the same layout. The workflow
  is least-privilege: the reusable guard, test and build jobs get a read-only token, and
  only `publish` gets `contents: write` (plus `id-token: write` where Trusted Publishing is
  used). Release runs of one repo are serialised (`concurrency`), so two quick
  pushes cannot both publish/tag the same version.
- **Docs** (`docs.yml`) rebuild on every push to `main` (site root) and `dev` (`/dev/`),
  independent of releases.

### How each package publishes

| package | PyPI authentication |
|---|---|
| brainmaze-utils, brainmaze-eeg, brainmaze (meta; publishing disabled for now) | **Trusted Publishing** (OIDC): PyPI publisher = owner `bnelair`, repo `<repo>`, workflow `release.yml`, no environment. The `publish` job has `id-token: write`. |
| brainmaze-zmq, brainmaze-eeg-models | **API token**, the organisation secret `PYPI_Token_General` (maintainer decision), passed to `pypa/gh-action-pypi-publish` as `password:` in the `publish` job, which first fails loudly if the secret is empty or not shared with the repo. No `id-token: write`. Everything else is the same flow. brainmaze-eeg-models' first upload (0.1.0, run [37357090471](https://github.com/bnelair/brainmaze-eeg-models/actions/runs/37357090471)) created the PyPI project with this token. |
| brainmaze-torch (archived) | History: published 0.2.0 and the final 0.3.0 with the organisation token, as above. The archived repo cannot run workflows. |

## Cutting a release (step by step)

1. Make sure `main` contains what you want to release and CI on `main` is green.
2. In the package repo: **Actions → Prepare release → Run workflow**, choose the bump:
   - `patch`: bug fixes only, no API change;
   - `minor`: new functionality, backward compatible (e.g. a new module, a new optional
     parameter);
   - `major`: breaking change (renamed/removed API, changed defaults that change results).
     **Changed numerical results count as breaking for this family.** Call them out in
     the release notes.
3. Review the **"Release vX.Y.Z"** PR. **Check that its diff is exactly the one
   `version = "X.Y.Z"` line in `pyproject.toml`.** That is the reviewer's job: CI and the
   Version guard do **not** run on this PR by themselves, because PRs opened with
   `GITHUB_TOKEN` don't trigger workflows. (If anyone pushes a commit to the bump branch,
   the push does trigger them, and the guard then fails unless the only change is still the
   version.) The release run tests again before building.
4. Squash-merge it. The merge triggers **Release** automatically; watch it under
   *Actions → Release*: test → build → publish → tag + release.
5. Check the result on PyPI and in GitHub Releases, then edit the generated notes if the
   change affects results (filters, detectors, defaults).

### Recovering from a failed release

**Never wait for "the next push to `main`" to retry.** The guard only checks whether the
version is tagged, so the next push would test, build, publish and tag *that* commit, not the
reviewed bump. This has happened: brainmaze-utils v2.0.0's release run 29460150640 on the
bump commit `13fd520` (#24) failed, and v2.0.0 was then published and tagged by run
29461829249 on `7b93ffa` (#25). Tag `v2.0.0` points at #25, not at "Release v2.0.0".

**Never run *Prepare release* again after a failed release**: it would bump to the next
version and leave X.Y.Z unreleased for good.

Find the failed *Release* run for the bump commit and go by the step that failed:

| failed step | state | what to do |
|---|---|---|
| guard / test / build (flaky runner, network) | nothing published or tagged | **Re-run failed jobs** on that same run. It rebuilds the same commit. |
| test / build (real bug) | nothing published or tagged | Fix it in a normal PR. Merging the fix is a push to `main` and releases the version from the fix commit, so review that PR as the release. |
| publish: token check or upload to PyPI (e.g. `invalid-publisher`, missing secret) | nothing published or tagged | Fix the cause (PyPI publisher settings, token secret), then **Re-run failed jobs** on the same run. The `dist` artifact is kept with the run. |
| publish: tag push, after a successful upload | PyPI has X.Y.Z, no tag | Do **not** re-run: PyPI files are immutable, so the upload step fails. Tag the commit that run built (the run's head SHA) by hand: `git tag vX.Y.Z <sha> && git push origin vX.Y.Z`, then `gh release create vX.Y.Z --verify-tag --title vX.Y.Z --generate-notes`. |
| publish: `gh release create`, after the tag push | PyPI + tag, no GitHub Release | `gh release create vX.Y.Z --verify-tag --title vX.Y.Z --generate-notes`. Re-running would fail at the upload, and later runs skip because the tag exists. |

### Releasing several packages that depend on each other

Release in dependency order and bump the lower bound in the dependents:

1. `brainmaze-utils`
2. `brainmaze-eeg` and `brainmaze-eeg-models` (both depend on `brainmaze-utils>=3.0.0`): in a
   normal PR, raise the brainmaze-utils lower bound in `dependencies` if the package needs the
   new release, merge, then release. Their required dependencies don't include each other,
   so their order doesn't matter, with one exception: brainmaze-eeg's optional `models`
   extra requires brainmaze-eeg-models (>=0.1.0 today); if brainmaze-eeg needs a newer one,
   release brainmaze-eeg-models first. `brainmaze-zmq` doesn't depend on brainmaze-utils
   and can be released any time.
3. `brainmaze`, the meta-package: raise lower bounds if needed, then release.

brainmaze-torch is retired (0.3.0 final, repo archived) and is never released again. Its
0.3.0 shim requires `brainmaze-eeg-models>=0.1.0` with no upper bound, so a breaking
brainmaze-eeg-models release can break installed brainmaze-torch 0.3.0; the meta repo's
compatibility run (see [Compatibility policy](#compatibility-policy)) catches that.

A dependent PR that needs an unreleased utils feature will fail CI until utils is on PyPI,
because CI installs dependencies from PyPI. Merge and release utils first.

## Branch protection and review

The release flow assumes `main` is protected (PR + review). What is configured today
(queried with `gh api repos/bnelair/<repo>/rulesets` on 2026-10-26): every ruleset is
active, blocks deletion and force-push, and lets org admins and the repo admin role bypass
it. No public repo uses classic branch protection.

| repo | ruleset on `main` | approvals | merge methods | notes |
|---|---|---|---|---|
| brainmaze-utils | 4278916 (`main`) | 1 | squash | also requires code-owner review, which has **no effect**: there is no CODEOWNERS file |
| brainmaze-eeg | 4286093 (`main` + `dev`) | 1 | squash | |
| brainmaze-eeg-models | 24383099 (default branch) | 1 | squash | |
| brainmaze-zmq | 2715191 (default branch) | 1 | squash | also requests a Copilot code review (not on every push, not on drafts). Tightened from 0 approvals / all merge methods in October 2026. |
| brainmaze-sphinx | 24320645 (default branch) | 1 | squash | added October 2026. Every caller uses `@main`, so a merge here changes every package's CI and release. |
| brainmaze | **none possible**: private repo on a plan without rulesets (the API answers 403) | — | — | `main` is unprotected; its release workflow is gated off (see that repo's `RELEASING.md`) |
| brainmaze-torch (archived) | 4896155 (`main` + `dev`) | 1 | squash | still configured, but moot: an archived repo is read-only |

No repository has **required status checks** on `main`: a red CI run (including the
artifact-content job `test / dist-contents / check`) does not block a merge by itself.
Reviewers must not merge a red PR. The artifact check is a hard gate only at release time
(`release.yml` fails before `publish`); see [`check-dist.yml`](README.md#check-distyml).

No repository has a `CODEOWNERS` file, so "code owner" review means nothing in practice:
any collaborator with write access can approve.

**"Allow GitHub Actions to create and approve pull requests" trade-off.** *Prepare release*
needs this setting to open its PR with `GITHUB_TOKEN`. The same switch lets *any* workflow
approve PRs: a collaborator can add a workflow to their own branch that runs
`gh pr review --approve` with `GITHUB_TOKEN`, and that satisfies a 1-approval rule.
Mitigations (maintainer decision, not configured today): a CODEOWNERS file listing humans
plus *require review from code owners* (the bot is not a code owner), or opening the bump PR
with a GitHub App token and turning the setting off.

### Version guard is advisory

The *Version guard* stays advisory (maintainer decision): no ruleset requires it (or CI), so
a red guard does not block a merge, and reviewers must not merge a PR it flags. It cannot
simply be made a required check: *Prepare release* opens the bump PR with `GITHUB_TOKEN`,
and PRs opened with `GITHUB_TOKEN` trigger no workflows, so a required check would never
report on a bump PR and would block every release. Making it enforceable would need the bump
PR to be opened with a GitHub App or fine-grained token (so workflows run on it), and then
requiring `guard` and CI.

## One-time setup per package repository

- [ ] Caller workflows in `.github/workflows/`, copied from
      [brainmaze-eeg](https://github.com/bnelair/brainmaze-eeg/tree/main/.github/workflows):
      `ci.yml`, `docs.yml`, `prepare-release.yml`, `release.yml`, `version-guard.yml`
      (change only `package-import-name` in `ci.yml`, plus the asset and `versions`
      inputs where needed). A repo that publishes with the organisation token copies
      `release.yml` from
      [brainmaze-eeg-models](https://github.com/bnelair/brainmaze-eeg-models/tree/main/.github/workflows)
      instead.
- [ ] **PyPI publishing**, one of (see [How each package publishes](#how-each-package-publishes)):
      Trusted Publishing: on pypi.org → project → *Publishing* → add a GitHub publisher:
      owner `bnelair`, repository `<repo>`, workflow `release.yml`, no environment (for a
      project that doesn't exist on PyPI yet, add a *pending* publisher); or the org token:
      make sure the organisation secret `PYPI_Token_General` is shared with the repo.
- [ ] **Organisation/repo setting**: *Settings → Actions → General → Allow GitHub Actions
      to create and approve pull requests* (needed by Prepare release). **Risk:** with it
      on, a workflow on a collaborator's own branch can approve that collaborator's PR,
      which satisfies a 1-approval rule (see the trade-off above). Kept on by maintainer
      decision; reviewers should look at who approved a PR before merging it.
- [ ] **Ruleset** on `main`: require a PR with 1 approval, squash merge only, block
      deletion and force-push. (Not met everywhere yet; see the table above.)
- [ ] **Pages**: *Settings → Pages → Deploy from a branch → `gh-pages` / root*.
- [ ] `pyproject.toml`: static `version = "X.Y.Z"` in `[project]`, `requires-python = ">=3.10"`,
      a `test` extra with `pytest`, and runtime dependencies that the code actually imports
      (no docs or test tools in `dependencies`).
- [ ] **Release artifacts contain only the package** (maintainer rule: no data, tests,
      demos or docs in the wheel or the sdist). In `pyproject.toml`:
      `[tool.setuptools.packages.find] include = ["<pkg>", "<pkg>.*"]` (exclude
      `<pkg>.tests*` if the tests live inside the package), `include-package-data = false`,
      and `[tool.setuptools.package-data]` only for files needed at runtime; a `MANIFEST.in`
      that does not pull data files into the sdist (`include-package-data = false` keeps
      setuptools from adding files via git/MANIFEST; prune `tests/`, `demo/`, `docs_src/`,
      `img/`, `.github/` only where the repository has such directories, since a `prune` of
      a missing directory is a no-op). Pass `dist-check: error` to `test.yml` in `ci.yml` and to `release.yml`; list
      every runtime data file the package needs in `required-assets` (e.g. brainmaze-eeg-models'
      four `.onnx` models), so the check fails if one drops out of the wheel or sdist; see
      [`check-dist.yml`](README.md#check-distyml).

### Release-system status (2026-10-26)

| repo | release system | latest release |
|---|---|---|
| brainmaze-utils | current (PR-based, Trusted Publishing) | v3.0.0 |
| brainmaze-eeg | current (Trusted Publishing) | v3.1.0 |
| brainmaze-eeg-models | current (org token), since its first release | v0.1.0 |
| brainmaze-zmq | current (org token), since bnelair/brainmaze-zmq#15 | v0.0.1, from the old flow; nothing released since the migration |
| brainmaze (meta, private repo) | current since bnelair/brainmaze#2, release gated off while private | never released from that repo; PyPI has only `0.0.0a0` |
| brainmaze-torch | **retired, archived**; was on the current flow (org token) since bnelair/brainmaze-torch#9 | v0.3.0, final |

History: before migrating, brainmaze-torch's old `release.yml` called removed inputs and
could not publish; brainmaze-utils v2.0.0 was tagged at #25 (see the recovery section above).

## Contributor rules

- Never edit `[project].version` by hand. Use *Prepare release*.
- Feature work goes through PRs. Each PR must keep CI green (on the version matrix, once the
  package's `ci.yml` passes it; see below).
- New or changed public functions need numpy-style docstrings: units, array orientation,
  NaN behaviour, and what the function deliberately does **not** do. The docs are
  generated from them.
- Changes that alter numerical results need before/after evidence in the PR (see
  brainmaze-eeg #60 for the expected level of detail).

## Compatibility policy

All family packages must install and pass their tests **together** on:

| | oldest supported | newest |
|---|---|---|
| Python | 3.10 (3.11 is tested too) | 3.12 |
| numpy | 1.x (`<2`, e.g. 1.26) | 2.x |
| other dependencies | each package's declared lower bounds | latest |

How it is checked (detection, not a merge gate):

1. **Per package**: the shared `test.yml` accepts the family matrix:
   ```yaml
   versions: '[{"python": "3.10", "numpy": "<2"}, {"python": "3.12", "numpy": ">=2"}]'
   ```
   (numpy<2 has no wheels for Python >= 3.13; keep the old stack on <= 3.12.)
   **Partly migrated:** brainmaze-eeg-models passes this matrix in `ci.yml` and
   `release.yml`. brainmaze-utils, brainmaze-eeg, brainmaze-zmq and the meta-package don't
   pass `versions` yet, so each is tested per package on Python 3.10 with the latest numpy
   only (the cross-package run below still covers numpy 1.x).
2. **Across packages**: the *Compatibility* workflow (`compat.yml`) in the `brainmaze`
   meta-package repo (private) installs all packages **and the meta-package itself**
   together in one environment (on its `main`: brainmaze-utils, -eeg, -torch and -zmq; with
   the open bnelair/brainmaze#4: brainmaze-utils, -eeg, -eeg-models, the brainmaze-torch
   0.3.0 shim and -zmq), from PyPI (latest releases) and from `main` (what will be
   released next), on Python 3.10 + numpy<2, 3.11, and 3.12 + numpy>=2. A further job
   installs `main` at the **lowest versions allowed by the declared lower bounds**
   (`uv --resolution lowest-direct`, wheels only). Each job runs `pip check`, imports
   everything, and runs each package's test suite (with its `[test]` extra) against that
   shared environment. It runs on every push/PR in that repo, weekly, and on demand
   (*Actions → Compatibility*). Nothing in the package repos triggers it or waits for it,
   so a change in one package that breaks another is caught at the latest by the weekly
   run; run it by hand before releasing a package others depend on.
3. **Dependency declarations**: a lower bound wherever the code needs one, matching what
   the package was tested with (`brainmaze_utils>=<first version with the API you use>`).
   No upper caps unless a known incompatibility exists, and then with a comment linking
   the issue. The declared bounds today are in [the family table](#the-family).
4. Code must not use APIs removed in numpy 2 (`np.NaN`, `np.float_`, `np.product`, …).
   The numpy-1 job catches the opposite problem.

## Testing changes to the shared workflows

`test.yml` and `release.yml` call `check-dist.yml@main`, and `release.yml` calls
`test.yml@main`. These nested references always resolve from `main` of this repo, even
when a caller pins a tag or SHA (GitHub has no "same ref" for nested cross-repo calls).
To exercise a change before merging it:

1. Push a temporary branch here (e.g. `tmp/<topic>`) with the nested `@main` refs in
   `test.yml`/`release.yml` changed to `@tmp/<topic>`.
2. Push a temporary branch in a caller repo whose `ci.yml` (and, for the release path, a
   temporary copy of `release.yml` without the publish job, or with a dummy one) calls
   `@tmp/<topic>`. Bump the version to an unused one (e.g. `9.9.9`) so the guard releases.
3. Check the runs, then **delete every temporary branch** (here and in the caller).

Because of this, keep every input change backward compatible (new inputs need defaults).

## Troubleshooting

| symptom | cause / fix |
|---|---|
| *Prepare release* fails: "branch release/bump-X.Y.Z already exists" | A previous bump PR is open or its branch wasn't deleted. Merge or close it, delete the branch, re-run. |
| *Prepare release* fails creating the PR | Enable *Allow GitHub Actions to create and approve pull requests*. |
| Release run failed part-way | See [Recovering from a failed release](#recovering-from-a-failed-release). |
| Release run: publish fails with `invalid-publisher` | The Trusted Publisher on PyPI doesn't match: it must be repo `bnelair/<repo>`, workflow `release.yml`, no environment. Fix it on PyPI, then re-run the failed job (nothing was tagged). |
| Release run: "Secret PYPI_Token_General is empty or not available" (zmq, eeg-models) | The organisation secret is missing or not shared with this repo. Fix it in the org settings, then re-run the failed job (nothing was published or tagged). |
| Release run: the first upload of a **new** PyPI project fails with 403 (token publishing) | The organisation token is scoped to existing projects only. Widen its scope, or add a PyPI pending trusted publisher; see "First release" in [brainmaze-eeg-models' RELEASING.md](https://github.com/bnelair/brainmaze-eeg-models/blob/main/RELEASING.md). (brainmaze-eeg-models 0.1.0 was created with the token without this problem.) |
| Something still needs brainmaze-torch's CI, issues or a new release | The repo is archived (read-only). Its final 0.3.0 is a shim over brainmaze-eeg-models: fix the problem in brainmaze-eeg-models, and move code to `brainmaze_eeg_models`. |
| *Compatibility* (meta repo) fails with `PackageNotFoundError: ... torch` | brainmaze-torch 0.3.0 no longer pulls in PyTorch, but the workflow on the meta repo's `main` still prints `torch`'s version. Fixed by bnelair/brainmaze#4 (open). |
| Release run: guard fails "[project].version ... is not X.Y.Z" | `pyproject.toml` has no static `[project].version` of the form `X.Y.Z`. Fix it in a reviewed PR. |
| Release run did nothing after merging the bump | Tag `vX.Y.Z` already exists (version not actually bumped), or the guard's `git ls-remote` failed. Check the *guard* job log. |
| A normal PR fails *Version guard* | It changes `[project].version`. Revert that line. |
| CI broke in every repo at once | The shared workflows are referenced as `@main`; a change in brainmaze-sphinx affects everyone immediately. Revert it there. |
