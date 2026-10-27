# gh_workflows

Reusable GitHub Actions workflows for stajichlab repositories.

## Workflows in this repo

**Shared** (call these from other repos with
`uses: stajichlab/gh_workflows/.github/workflows/<file>@v1`):

| Workflow | Purpose |
|---|---|
| [`release_prepare`](#release_prepare) | Bump the version and open a release pull request |
| [`release_publish`](#release_publish) | Tag a merged release PR and create the GitHub release |
| [`sync_dev`](#sync_dev) | Merge the release into the development branch |
| [`container_build`](#container_build) | Build, smoke-test and push a container image |
| [`pixi_lock_refresh`](#pixi_lock_refresh) | Refresh `pixi.lock` on Dependabot PRs |
| [`changelog_check`](#changelog_check) | Require a changelog entry on PRs |
| [`citation_check`](#citation_check) | Validate `CITATION.cff` |
| [`python_test`](#python_test) | Run Python tests with pip, conda or pixi |

> [!NOTE]
> **For this repo only, not for callers:** `release.yml`, `publish.yml`, `changelog.yml`, `lint.yml` and the
> `test_*.yml` workflows. They have no `workflow_call` trigger, so other repos cannot call them. They release, check
> and test `gh_workflows` itself; see [Releasing this repo](#releasing-this-repo) and [Development](#development).

## release_prepare

Computes the next version from the latest `v*` tag, checks it against `CHANGELOG.md`
([Keep a Changelog](https://keepachangelog.com) format) if the repo has one, bumps the changelog, runs the
calling repo's own version bump command, updates and validates `CITATION.cff` if present (warning if it is missing), commits to a `release/v<version>`
branch and opens a pull request for review. Tagging, building and publishing are left to the caller (for
example, a workflow triggered when the release PR is merged).

**Version source.** Tags are the source of truth: the previous version is the highest semver among `v*` tags reachable from `base_branch`
(`0.0.0` if there are none), and the next one is derived from it with `bump_type`/`preid`. If a changelog exists,
its latest release must equal the latest tag, otherwise the workflow fails; the changelog is then bumped to the
tag-derived version. Without a changelog, the PR lists commit subjects since the last tag instead.

The workflow also fails early if the tag or the `release/v<version>` branch already exists.

The version bump is supplied by the caller through `additional_bump_command`, so each repo decides which files carry the
version. The new version is available as `$VERSION`.

### Use it when

- You want releases to go through a reviewed pull request instead of tagging by hand.
- The version should come from git tags, with `CHANGELOG.md` and `CITATION.cff` kept in step automatically.
- Each repo stores its version somewhere different (`pyproject.toml`, `__init__.py`, an R `DESCRIPTION` file, ...)
  and needs its own small bump command.
- You publish release candidates (`1.3.0-rc.0`) before a final release, or patch an older release line.

### Examples

**Minimal: manual release with a bump-type menu.** Run it from **Actions → Release**; it opens a
`Release v<version>` PR.

```yaml
# .github/workflows/release.yml
name: Release
on:
  workflow_dispatch:
    inputs:
      bump_type:
        description: Version bump type
        type: choice
        options: [patch, minor, major, prepatch, preminor, premajor, prerelease]
      preid:
        description: Prerelease identifier (e.g. rc)
        type: string

jobs:
  prepare:
    uses: stajichlab/gh_workflows/.github/workflows/release_prepare.yml@v1
    permissions:
      contents: write
      pull-requests: write
    with:
      bump_type: ${{ inputs.bump_type }}
      preid: ${{ inputs.preid }}
```

**Python package with a fallback version in `pyproject.toml`** (as in Phyling), with reviewers requested and a
token so the release PR runs CI:

```yaml
jobs:
  prepare:
    uses: stajichlab/gh_workflows/.github/workflows/release_prepare.yml@v1
    permissions:
      contents: write
      pull-requests: write
    with:
      bump_type: ${{ inputs.bump_type }}
      preid: ${{ inputs.preid }}
      reviewers: some-user,stajichlab/some-team
      additional_bump_command: |
        sed -i "s/^default-version = .*/default-version = \"$VERSION\"/" pyproject.toml
        grep -q "^default-version = \"$VERSION\"" pyproject.toml
    secrets:
      token: ${{ secrets.RELEASE_TOKEN }}
```

**Version in several files** (a Python `__init__.py` and an R `DESCRIPTION`). Ending with `grep -q` checks makes
the run fail if a file didn't contain the expected line:

```yaml
    with:
      bump_type: ${{ inputs.bump_type }}
      additional_bump_command: |
        sed -i "s/^__version__ = .*/__version__ = \"$VERSION\"/" src/mytool/__init__.py
        sed -i "s/^Version: .*/Version: $VERSION/" DESCRIPTION
        grep -q "^__version__ = \"$VERSION\"" src/mytool/__init__.py
        grep -q "^Version: $VERSION$" DESCRIPTION
```

**Patch an older release line** from a maintenance branch. Only tags on that branch count, so `patch` after
`v1.4.2` gives `1.4.3` even if `main` is already at `2.x`:

```yaml
    with:
      bump_type: patch
      base_branch: v1.x
```

**Release candidates.** `preminor` with `preid: rc` turns `1.2.0` into `1.3.0-rc.0`. Later runs with
`prerelease` give `1.3.0-rc.1`, and a final `patch` run gives `1.3.0`.

### Reference

| Input | Default | Description |
|---|---|---|
| `bump_type` | (required) | `major`, `premajor`, `minor`, `preminor`, `patch`, `prepatch` or `prerelease` |
| `preid` | `""` | Prerelease identifier |
| `additional_bump_command` | `""` | Shell command that updates version strings; runs with `$VERSION` set |
| `changelog_path` | `CHANGELOG.md` | Changelog location; skipped if missing |
| `base_branch` | `main` | Branch to release from and PR target |
| `cffconvert_version` | `2.0.0` | cffconvert version used to validate `CITATION.cff` after the bump |
| `draft` | `false` | Open the PR as a draft |
| `reviewers` | `""` | Comma-separated users or `org/team` slugs to request review from |

| Secret | Description |
|---|---|
| `token` | Optional GitHub App token or PAT (contents and pull requests: write). PRs opened with the default `GITHUB_TOKEN` don't trigger the repo's CI; pass this to get CI checks on the release PR. Requesting team reviewers may also need it. |

Outputs: `version`, `previous_version`, `release_notes`, `prerelease` (`'true'`/`'false'`), `pr_url`.

Calling repos must enable **Settings → Actions → General → Allow GitHub Actions to create and approve pull
requests**.

If this repo is private, allow access from other org repos under
**Settings → Actions → General → Access**.

If a step fails after the release branch was pushed, the branch is deleted so the run can simply be retried.

Only one `release_prepare` run per repository proceeds at a time. A second run waits; if a third is started while
one is waiting, GitHub cancels the waiting one.

## release_publish

The other half of `release_prepare`. When a `release/v<version>` pull request is merged, it tags the merge commit
on the base branch as `v<version>` (where the next `release_prepare` run will find it), creates the GitHub release
and deletes the release branch. Release notes come from the changelog entry for that version, or are generated
by GitHub if there is no changelog. Versions with a prerelease suffix (`1.3.0-rc.0`) are marked as prereleases and
never as the latest release. Files from an artifact built earlier in the calling run can be attached.

It does nothing (and outputs `published: 'false'`) unless the triggering PR was merged from a `release/v*` branch,
so it is safe to call on every closed PR.

### Use it when

- You use `release_prepare` and need the merged release PR turned into a tag and a GitHub release.
- You want built files (an sdist, wheels, binaries) attached to the GitHub release.
- Publishing should wait for a maintainer's approval.
- A publish run failed partway and needs to be re-run by hand.

### Examples

**Minimal: tag and release on merge.** Safe to run on every closed PR; it only publishes merged `release/v*` PRs.

```yaml
# .github/workflows/publish.yml
name: Publish
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  publish:
    uses: stajichlab/gh_workflows/.github/workflows/release_publish.yml@v1
    permissions:
      contents: write
```

**Full pipeline: build, publish with files attached, then container and dev sync.** Repo-specific jobs (`build`)
stay in the repo; shared ones are chained with `needs:` so they run after the tag exists.

```yaml
name: Publish
on:
  pull_request:
    types: [closed]
    branches: [main]

jobs:
  build:
    if: github.event.pull_request.merged && startsWith(github.head_ref, 'release/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - run: pipx run build --sdist
      - uses: actions/upload-artifact@v7
        with:
          name: dist
          path: dist/

  publish:
    needs: build
    uses: stajichlab/gh_workflows/.github/workflows/release_publish.yml@v1
    permissions:
      contents: write
    with:
      artifact: dist

  container:
    needs: publish
    if: needs.publish.outputs.published == 'true'
    uses: stajichlab/gh_workflows/.github/workflows/container_build.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      ref: ${{ needs.publish.outputs.tag }}
      latest: true   # ignored for prereleases

  sync-dev:
    needs: publish
    if: needs.publish.outputs.published == 'true'
    uses: stajichlab/gh_workflows/.github/workflows/sync_dev.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

**Require approval before publishing.** Create a `release` environment with required reviewers under
**Settings → Environments**, then:

```yaml
  publish:
    uses: stajichlab/gh_workflows/.github/workflows/release_publish.yml@v1
    permissions:
      contents: write
    with:
      environment: release
```

**Re-publish by hand** after a failed run. Start it from `main` so the commit it runs on is the release commit (or
pass `target`); the tag is reused if it already exists on that commit:

```yaml
# .github/workflows/publish-manual.yml
name: Publish (manual)
on:
  workflow_dispatch:
    inputs:
      version:
        description: Version to publish (e.g. 1.3.0)
        required: true

jobs:
  publish:
    uses: stajichlab/gh_workflows/.github/workflows/release_publish.yml@v1
    permissions:
      contents: write
    with:
      version: ${{ inputs.version }}
      delete_branch: false
```

### Reference

| Input | Default | Description |
|---|---|---|
| `version` | from `release/v<version>` | Version to publish |
| `target` | PR merge commit (`github.sha` if `version` is given) | Commit to tag |
| `changelog_path` | `CHANGELOG.md` | Changelog location; GitHub-generated notes if missing |
| `artifact` | `""` | Artifact from the calling run whose files are attached |
| `environment` | `""` | Deployment environment for the publish job, e.g. to require approval |
| `delete_branch` | `true` | Delete `release/v<version>` afterwards |

| Secret | Description |
|---|---|
| `token` | Optional GitHub App token or PAT. Tags and releases created with `GITHUB_TOKEN` don't trigger other workflows (`push: tags`, `release: published`); pass this if anything relies on those triggers. |

Outputs: `published`, `version`, `tag`, `prerelease`, `release_url`.

**Requiring approval.** Pass `environment: release` (any name) and, in the calling repo, configure that environment
under **Settings → Environments** with required reviewers. The publish job then waits for approval before
tagging. The workflow only names the environment; the approval rule lives in each repo's settings, and GitHub
creates an unconfigured environment (no approval) if it doesn't exist yet. A `token` secret stored in that
environment is used in place of one passed by the caller, so it is only available to approved runs.

## sync_dev

Brings the development branch (`dev`) up to date with the release branch (`main`) after a release. It merges
`main` into `dev` and pushes; if `dev` already contains `main` it does nothing. If the merge conflicts or the push
is rejected (for example by branch protection on `dev`), it pushes `main`'s commit to a
`sync/main-to-dev-<sha>` branch and opens a pull request into `dev` instead, so conflicts are resolved on that
branch and never on `main`. Re-running while that PR is open reuses it.

### Use it when

- Your repo develops on `dev` (or `develop`) and releases from `main`, and `dev` should pick up the release commit
  (version bump, changelog) right after each release.
- Fixes made on `main` during a release must not be lost on `dev`.
- `dev` is protected, so updates must arrive as a pull request.

### Examples

**After publishing (recommended).** Chain it after `release_publish` so it runs as soon as the release exists:

```yaml
jobs:
  publish:
    uses: stajichlab/gh_workflows/.github/workflows/release_publish.yml@v1
    permissions:
      contents: write

  sync-dev:
    needs: publish
    if: needs.publish.outputs.published == 'true'
    uses: stajichlab/gh_workflows/.github/workflows/sync_dev.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

**Different branch names, with reviewers for the fallback PR:**

```yaml
  sync-develop:
    uses: stajichlab/gh_workflows/.github/workflows/sync_dev.yml@v1
    permissions:
      contents: write
      pull-requests: write
    with:
      source_branch: master
      target_branch: develop
      reviewers: some-user
```

**On a published release, or by hand.** `release: published` only fires if the release was created with a token
other than `GITHUB_TOKEN`:

```yaml
# .github/workflows/sync-dev.yml
name: Sync dev
on:
  release:
    types: [published]
  workflow_dispatch:

jobs:
  sync-dev:
    uses: stajichlab/gh_workflows/.github/workflows/sync_dev.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

### Reference

| Input | Default | Description |
|---|---|---|
| `source_branch` | `main` | Branch that received the release |
| `target_branch` | `dev` | Branch to bring up to date |
| `reviewers` | `""` | Reviewers to request if a PR is opened |

| Secret | Description |
|---|---|
| `token` | Optional GitHub App token or PAT. Needed if pushes to `dev` should trigger CI, or if `dev` is protected and the token is allowed to bypass that protection. |

Outputs: `result` (`up-to-date`, `merged` or `pull-request`), `pr_url`.

## container_build

Builds a container image from a branch, tag or commit, runs an optional smoke test against the locally loaded
image, then pushes it (to GHCR by default). Tags:

| Tag | When |
|---|---|
| `1.2.3` | a version is given, or `ref` looks like `v1.2.3` |
| `1.2` | same, but not for prereleases |
| `latest` | `latest: true`, but not for prereleases |
| `sha-<short sha>` | always |
| extra tags | from `extra_tags` |

### Use it when

- Your tool ships a container image and every release should publish a matching, versioned image.
- You want a test image from a branch without making a release.
- PRs that touch the `Dockerfile` should prove the image still builds and runs, without pushing anything.
- You need multi-architecture images or a registry other than GHCR.

### Examples

**Release image, after `release_publish`.** Tags `1.2.3`, `1.2`, `latest` and `sha-...`; a prerelease gets only its
exact version:

```yaml
  container:
    needs: publish
    if: needs.publish.outputs.published == 'true'
    uses: stajichlab/gh_workflows/.github/workflows/container_build.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      ref: ${{ needs.publish.outputs.tag }}
      latest: true
      smoke_test_args: --help
```

**Test image from any branch, by hand:**

```yaml
# .github/workflows/container-test.yml
name: Container (test image)
on:
  workflow_dispatch:
    inputs:
      tag:
        description: Image tag to publish
        default: testing

jobs:
  container:
    uses: stajichlab/gh_workflows/.github/workflows/container_build.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      ref: ${{ github.ref_name }}
      extra_tags: ${{ inputs.tag }}
      smoke_test_args: --help
```

**Build check on pull requests** (builds and smoke-tests, never pushes):

```yaml
# .github/workflows/container-check.yml
name: Container check
on:
  pull_request:
    paths: [Dockerfile, .dockerignore, pyproject.toml]

jobs:
  container:
    uses: stajichlab/gh_workflows/.github/workflows/container_build.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      push: false
      smoke_test_args: --version
```

**Multi-architecture image on Docker Hub, with a build argument:**

```yaml
  container:
    needs: publish
    if: needs.publish.outputs.published == 'true'
    uses: stajichlab/gh_workflows/.github/workflows/container_build.yml@v1
    permissions:
      contents: read
      packages: write
    with:
      ref: ${{ needs.publish.outputs.tag }}
      image: docker.io/stajichlab/mytool
      platforms: linux/amd64,linux/arm64
      build_args: |
        CONDA_ENV=environment.yml
      latest: true
    secrets:
      registry_username: ${{ secrets.DOCKERHUB_USERNAME }}
      registry_password: ${{ secrets.DOCKERHUB_TOKEN }}
```

### Reference

| Input | Default | Description |
|---|---|---|
| `ref` | triggering commit | Branch, tag or commit to build |
| `version` | from `ref` if it is `v<semver>` | Version tag |
| `image` | `ghcr.io/<owner>/<repo>` (lowercased) | Image name without tag |
| `context` | `.` | Build context |
| `dockerfile` | `Dockerfile` | Dockerfile path |
| `build_args` | `""` | Newline-separated `NAME=value` |
| `platforms` | `linux/amd64` | Platforms to push; QEMU is set up for anything else |
| `smoke_test_args` | `""` | Arguments for `docker run --rm <image>`; skipped if empty |
| `latest` | `false` | Tag as `latest` (ignored for prereleases) |
| `extra_tags` | `""` | Newline-separated extra tags |
| `push` | `true` | Push; if `false`, only build and smoke-test |

| Secret | Description |
|---|---|
| `registry_username`, `registry_password` | For registries other than GHCR (e.g. Docker Hub). GHCR uses `GITHUB_TOKEN`. |

Outputs: `image`, `tags`, `digest`.

The checkout includes full history and tags, so version tools such as versioningit or setuptools-scm work during
the build (the `.git` directory must not be excluded by `.dockerignore` for that). The first push of a new GHCR
image creates it as private; change its visibility in the package settings.

## pixi_lock_refresh

Dependabot bumps dependencies in `pyproject.toml` / `pixi.toml` but does not update `pixi.lock`. This regenerates
the lock file on the pull request branch and pushes it, so the PR stays installable with `pixi install --locked`.
It does nothing if the lock file is already current, and by default only acts on Dependabot PRs from the same
repository.

### Use it when

- Your repo uses pixi with a committed `pixi.lock`, and Dependabot updates dependencies in `pyproject.toml` or
  `pixi.toml`.
- CI installs with `--locked` (as `python_test` does by default), so Dependabot PRs fail until the lock file is
  regenerated.

### Examples

**Basic: refresh on Dependabot PRs.**

```yaml
# .github/workflows/pixi-lock.yml
name: Refresh pixi.lock
on:
  pull_request:
    paths: [pyproject.toml, pixi.toml]

jobs:
  refresh:
    uses: stajichlab/gh_workflows/.github/workflows/pixi_lock_refresh.yml@v1
    permissions:
      contents: write   # Dependabot runs get a read-only token unless the caller grants write
```

**Manifest in a subdirectory, pixi version pinned, and a token so CI re-runs on the lock-file commit** (store
`PIXI_LOCK_TOKEN` under **Settings → Secrets and variables → Dependabot**):

```yaml
name: Refresh pixi.lock
on:
  pull_request:
    paths: [analysis/pixi.toml]

jobs:
  refresh:
    uses: stajichlab/gh_workflows/.github/workflows/pixi_lock_refresh.yml@v1
    permissions:
      contents: write
    with:
      working_directory: analysis
      pixi_version: v0.50.0
    secrets:
      token: ${{ secrets.PIXI_LOCK_TOKEN }}
```

**Any same-repo PR, not just Dependabot** (for contributors who edit dependencies without re-locking):

```yaml
    with:
      dependabot_only: false
```

### Reference

| Input | Default | Description |
|---|---|---|
| `working_directory` | `.` | Directory with the manifest and `pixi.lock` |
| `pixi_version` | latest | pixi version to install |
| `dependabot_only` | `true` | Only act on Dependabot PRs |

| Secret | Description |
|---|---|
| `token` | Optional GitHub App token or PAT. Workflows triggered by Dependabot can only read **Dependabot secrets**, so store it there. |

Outputs: `updated`.

**Things to know.**
- A push made with `GITHUB_TOKEN` does not trigger workflows, so the PR's CI does not rerun on the lock-file
  commit. If CI checks are required for merging, pass `token`, or the PR waits on checks that never run for its
  latest commit.
- Once someone else has pushed to a Dependabot branch, Dependabot stops rebasing it automatically. Comment
  `@dependabot recreate` to start over (the lock file is refreshed again on the new PR).

## changelog_check

Fails a pull request that doesn't add an entry under `[Unreleased]` in `CHANGELOG.md`, so `release_prepare`
never finds that section empty. It checks that the PR adds non-blank lines to the changelog and that `[Unreleased]`
has content (which also catches a malformed changelog). Skipped for PRs labelled `skip-changelog`, PRs opened by
Dependabot, and `release/v*` PRs.

### Use it when

- Your repo keeps a Keep a Changelog `CHANGELOG.md` and releases with `release_prepare`, which fails if
  `[Unreleased]` is empty.
- You want contributors to describe user-visible changes in the PR that makes them, not at release time.

### Examples

**Basic:**

```yaml
# .github/workflows/changelog.yml
name: Changelog
on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]

jobs:
  changelog:
    uses: stajichlab/gh_workflows/.github/workflows/changelog_check.yml@v1
```

**Different file and label, more bots skipped, and only for PRs into `dev`:**

```yaml
name: Changelog
on:
  pull_request:
    branches: [dev]
    types: [opened, synchronize, reopened, labeled, unlabeled]

jobs:
  changelog:
    uses: stajichlab/gh_workflows/.github/workflows/changelog_check.yml@v1
    with:
      changelog_path: docs/CHANGELOG.md
      skip_label: no-changelog
      skip_actors: "dependabot[bot] renovate[bot] pre-commit-ci[bot]"
```

Including `labeled` and `unlabeled` makes adding the skip label re-run the check. Create the skip label in each
repo, and make the check required in branch protection if it should block merging.

### Reference

| Input | Default | Description |
|---|---|---|
| `changelog_path` | `CHANGELOG.md` | Changelog location |
| `skip_label` | `skip-changelog` | Label that skips the check |
| `skip_actors` | `dependabot[bot]` | Space-separated PR authors to skip |

## citation_check

Validates `CITATION.cff` against the Citation File Format schema with
[cffconvert](https://github.com/citation-file-format/cffconvert), and warns (without failing) if its `version`
differs from the latest `v*` tag on the base branch, or from the PR's version on a `release/v*` PR. Does nothing if
the file doesn't exist, unless `required: true`.

### Use it when

- Your repo has a `CITATION.cff` (for GitHub's "Cite this repository" button or Zenodo archiving) and edits to it
  should be checked before they merge.
- You want an early warning that the citation version has drifted from the released version.

`release_prepare` also validates the file at release time; this check catches problems in the PR that causes them.

### Examples

**Basic: check when the file changes.**

```yaml
# .github/workflows/citation.yml
name: Citation
on:
  pull_request:
    paths: [CITATION.cff]
  push:
    branches: [main]
    paths: [CITATION.cff]

jobs:
  citation:
    uses: stajichlab/gh_workflows/.github/workflows/citation_check.yml@v1
```

**Require the file**, for repos that must always be citable. It runs on every PR, so deleting the file is caught
too:

```yaml
name: Citation
on:
  pull_request:

jobs:
  citation:
    uses: stajichlab/gh_workflows/.github/workflows/citation_check.yml@v1
    with:
      required: true
```

### Reference

| Input | Default | Description |
|---|---|---|
| `ref` | triggering commit | Branch, tag or commit to check |
| `citation_path` | `CITATION.cff` | Citation file location |
| `required` | `false` | Fail if the file is missing |
| `cffconvert_version` | `2.0.0` | cffconvert version used to validate |

## python_test

Runs a Python project's tests across a matrix, installing it with pip, conda (micromamba) or pixi. One workflow
replaces separate per-installer CI workflows; call it once per installer. Optionally lints with ruff, puts
externally built tools from artifacts on `PATH`, and uploads coverage to Codecov.

### Use it when

- Your repo is a Python package and needs tests on several Python versions.
- You support more than one way to install (pip, conda/bioconda, pixi) and want to test each without keeping
  separate CI files in step.
- Tests need external command-line tools (aligners, tree builders) built or downloaded earlier in the run.

### Examples

**Simple pip package** (installs `-e .[dev]`, runs `pytest` on Python 3.10–3.14):

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    uses: stajichlab/gh_workflows/.github/workflows/python_test.yml@v1
```

**Linux and macOS, two Python versions, with ruff:**

```yaml
jobs:
  test:
    uses: stajichlab/gh_workflows/.github/workflows/python_test.yml@v1
    with:
      runs_on: '["ubuntu-latest", "macos-latest"]'
      python_versions: '["3.11", "3.13"]'
      ruff: true
```

**All three installers, external tools and coverage** (Phyling's setup):

```yaml
name: CI
on:
  push:
    branches: [main, dev]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-tools:   # repo-specific, uploads artifacts named tool-*
    uses: ./.github/workflows/build-tools.yml

  pip:
    needs: build-tools
    uses: stajichlab/gh_workflows/.github/workflows/python_test.yml@v1
    with:
      ruff: true
      tools_artifact_pattern: tool-*
      test_command: pytest --cov --cov-report=xml
      codecov: true
    secrets:
      codecov_token: ${{ secrets.CODECOV_TOKEN }}

  conda:
    uses: stajichlab/gh_workflows/.github/workflows/python_test.yml@v1
    with:
      installer: conda
      conda_environment_file: dev_environment.yml
      python_versions: '[""]'   # use the Python pinned in the file

  pixi:
    uses: stajichlab/gh_workflows/.github/workflows/python_test.yml@v1
    with:
      installer: pixi
      pixi_environments: '["dev"]'
      setup_command: command -v astral && command -v fasttree && command -v iqtree
```

**Several pixi environments** (for example one per Python version, defined in `pixi.toml`), running a pixi task:

```yaml
jobs:
  test:
    uses: stajichlab/gh_workflows/.github/workflows/python_test.yml@v1
    with:
      installer: pixi
      pixi_environments: '["py310", "py313"]'
      test_command: pixi run test
```

### Reference

`setup_command` and `test_command` run with bash inside the installed environment (the pip Python, the conda
environment, or the pixi environment), so they see its packages and tools.

| Input | Default | Description |
|---|---|---|
| `ref` | triggering commit | Branch, tag or commit to test |
| `installer` | `pip` | `pip`, `conda` or `pixi` |
| `python_versions` | `["3.10", …, "3.14"]` | JSON list (pip, conda). For conda, `[""]` uses the file's Python |
| `pixi_environments` | `["default"]` | JSON list of pixi environments (pixi) |
| `runs_on` | `["ubuntu-latest"]` | JSON list of runner images |
| `pip_install` | `-e .[dev]` | Arguments to `pip install` (pip) |
| `conda_environment_file` | `environment.yml` | File that installs the project and test dependencies (conda) |
| `pixi_locked` | `true` | Fail if `pixi.lock` is out of date (pixi) |
| `tools_artifact_pattern` | `""` | Artifacts from the calling run whose files go on `PATH` |
| `setup_command` | `""` | Command run in the environment before the tests |
| `test_command` | `pytest` | Command that runs the tests |
| `codecov` | `false` | Upload `coverage.xml` to Codecov, flagged by installer and version |
| `ruff` | `false` | Also run a ruff lint job |

| Secret | Description |
|---|---|
| `codecov_token` | Codecov upload token |

## Versioning

Pin callers to a major tag (`@v1`). Within `v1`, inputs and outputs stay compatible; the `v1` tag is moved to
each new compatible release automatically. Breaking changes get `v2`. Every release is listed in
[CHANGELOG.md](CHANGELOG.md).

## Releasing this repo

This repo releases itself with its own workflows (the repo-only callers below use `./.github/workflows/...`, so
they always run the current code):

1. Add an entry under `[Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with every PR. `changelog.yml` enforces this
   (label a PR `skip-changelog` if it needs none).
2. Run **Actions → Release** (`release.yml`) with a bump type: `patch` for fixes, `minor` for new workflows or
   inputs, `major` for breaking changes. It opens a `Release v<version>` PR.
3. Review and merge the PR. `publish.yml` then tags `v<version>` and creates the GitHub release (via
   `release_publish`), and a second job moves `v<major>` (e.g. `v1`) to it, so callers pinned to `@v1` get the
   release. Prereleases, and a patch to an older minor version, leave `v<major>` where it is.

The first release is `major` from no tags, giving `v1.0.0` and `v1`.

## Development

- `release.yml`, `publish.yml` and `changelog.yml` run this repo's own releases and changelog check (see
  [Releasing this repo](#releasing-this-repo)).
- `lint.yml` runs [actionlint](https://github.com/rhysd/actionlint) on every push to `main` and every PR.
- `test_release_prepare.yml` runs `release_prepare` end to end against a throwaway orphan branch with a `v0.1.0`
  tag, checks the release branch and the draft PR, then deletes the PR, branches and tag. It runs on pushes to
  `main` that touch either workflow, or manually. It needs **Allow GitHub Actions to create and approve pull
  requests** enabled in this repo.
- `test_release_publish.yml` publishes a test prerelease (`v0.0.0-test.<run id>`, never marked latest) from a
  throwaway orphan branch with an attached artifact, checks the tag, notes, asset and branch deletion, then deletes
  the release, tag and branches. The merged-PR trigger path is not covered.
- `test_sync_dev.yml` runs `sync_dev` end to end on three throwaway branch pairs (already up to
  date, clean merge, conflict), checks the results and the conflict PR, then deletes the PR and branches. The
  "push rejected" fallback needs branch protection and is not covered. Same triggers and repo setting as above.
- `test_citation_check.yml` runs `citation_check` on throwaway orphan branches with a valid `CITATION.cff` matching
  its tag (`required: true`) and with no file, both of which must pass, and checks that the pinned cffconvert
  rejects an invalid file. The invalid case calls cffconvert directly, since a failing reusable-workflow call would
  fail the whole test run.
- `test_python_test.yml` builds a tiny package on a throwaway orphan branch (with a conda environment file and a
  freshly generated `pixi.lock`) and runs `python_test` against it with pip (two Python versions, ruff, and a fake
  tool from an artifact on `PATH`), conda and pixi. The test commands check that they run inside the right
  environment and Python version.
- `test_container_build.yml` builds a tiny image from a throwaway orphan branch for a release and a prerelease
  version, runs the smoke test and checks the tags. It never pushes.
- Dependabot updates the pinned action SHAs monthly.
