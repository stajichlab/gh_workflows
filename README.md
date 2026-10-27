# gh_workflows

Reusable GitHub Actions workflows for stajichlab repositories.

## release_prepare

Computes the next version from the latest `v*` tag, checks it against `CHANGELOG.md`
([Keep a Changelog](https://keepachangelog.com) format) if the repo has one, bumps the changelog, runs the
calling repo's own version bump command, updates `CITATION.cff` if present, commits to a `release/v<version>`
branch and opens a pull request for review. Tagging, building and publishing are left to the caller (for
example, a workflow triggered when the release PR is merged).

**Version source.** Tags are the source of truth: the previous version is the highest semver among `v*` tags reachable from `base_branch`
(`0.0.0` if there are none), and the next one is derived from it with `bump_type`/`preid`. If a changelog exists,
its latest release must equal the latest tag, otherwise the workflow fails; the changelog is then bumped to the
tag-derived version. Without a changelog, the PR lists commit subjects since the last tag instead.

The workflow also fails early if the tag or the `release/v<version>` branch already exists.

The version bump is supplied by the caller through `additional_bump_command`, so each repo decides which files carry the
version. The new version is available as `$VERSION`.

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
      token: ${{ secrets.RELEASE_TOKEN }} # optional, see below
```

| Input | Default | Description |
|---|---|---|
| `bump_type` | (required) | `major`, `premajor`, `minor`, `preminor`, `patch`, `prepatch` or `prerelease` |
| `preid` | `""` | Prerelease identifier |
| `additional_bump_command` | `""` | Shell command that updates version strings; runs with `$VERSION` set |
| `changelog_path` | `CHANGELOG.md` | Changelog location; skipped if missing |
| `base_branch` | `main` | Branch to release from and PR target |
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

```yaml
on:
  pull_request:
    types: [closed]

jobs:
  build:   # repo-specific build and test; upload files with actions/upload-artifact
    if: github.event.pull_request.merged && startsWith(github.head_ref, 'release/v')
    ...

  publish:
    needs: build
    uses: stajichlab/gh_workflows/.github/workflows/release_publish.yml@v1
    permissions:
      contents: write
    with:
      artifact: dist   # optional

  container:   # repo-specific, runs after the tag exists
    needs: publish
    if: needs.publish.outputs.published == 'true'
    ...

  sync-dev:
    needs: publish
    if: needs.publish.outputs.published == 'true'
    uses: stajichlab/gh_workflows/.github/workflows/sync_dev.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

To publish by hand (for example after a failed run), call it from a `workflow_dispatch` workflow with `version`
and optionally `target`. If the tag already exists on the same commit it is reused; on a different commit the
workflow fails.

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

```yaml
on:
  release:
    types: [published]

jobs:
  sync-dev:
    uses: stajichlab/gh_workflows/.github/workflows/sync_dev.yml@v1
    permissions:
      contents: write
      pull-requests: write
```

Or chain it after the publish job with `needs:`. A release created with the default `GITHUB_TOKEN` does not
trigger `release: published` workflows.

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

Typical use is after `release_publish`, building from the new tag:

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

For a manual test image, call it from a `workflow_dispatch` workflow with `ref: ${{ github.ref_name }}`,
`extra_tags: testing` and `push: false` (or `true`).

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

```yaml
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

```yaml
name: Changelog
on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]

jobs:
  changelog:
    uses: stajichlab/gh_workflows/.github/workflows/changelog_check.yml@v1
```

Including `labeled` and `unlabeled` makes adding the skip label re-run the check. Create the `skip-changelog` label
in each repo, and make the check required in branch protection if it should block merging.

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

```yaml
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

`setup_command` and `test_command` run with bash inside the installed environment (the pip Python, the conda
environment, or the pixi environment), so they see its packages and tools.

| Input | Default | Description |
|---|---|---|
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
each new compatible release. Breaking changes get `v2`.

## Development

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
- `test_container_build.yml` builds a tiny image from a throwaway orphan branch for a release and a prerelease
  version, runs the smoke test and checks the tags. It never pushes.
- Dependabot updates the pinned action SHAs monthly.
