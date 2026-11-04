# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed

- **Breaking:** `python_test` is split into `python_test_pip`, `python_test_conda` and `python_test_pixi`, each
  exposing only its installer's inputs (`installer` is gone; pixi takes `environments`, conda takes
  `environment_file`). `python_test.yml` becomes their internal shared core and is no longer a public interface.
- **Breaking:** `container_build`: `latest` now defaults to `true`, so stable versioned builds are tagged `latest`
  unless the caller passes `latest: false`. Builds without a version are never tagged `latest`.
- **Breaking:** `python_test`: the Codecov upload now runs whenever the `codecov_token` secret is passed; the
  `codecov` input is removed.

### Removed

- **Breaking:** `python_test`: `setup_command`; put setup lines at the top of the multi-line `test_command`.
- **Breaking:** `python_test`: `pixi_locked`; pixi always installs with `--locked`.
- **Breaking:** `python_test`: `ruff` and its lint job; add a job using `astral-sh/ruff-action` in the caller.
- **Breaking:** `release_prepare` and `citation_check`: `cffconvert_version`; both now use the cffconvert version
  pinned in this repo, so they always validate the same way.

## [1.0.0] - 2026-10-27

### Added

- `release_prepare`: compute the next version from the latest `v*` tag on the base branch, check it against the
  changelog, bump the changelog and `CITATION.cff` (validated with cffconvert), run a caller-supplied
  `additional_bump_command`, and open a release pull request with a version and changelog summary. Supports draft
  PRs, reviewers and an optional token so the PR triggers CI.
- `release_publish`: after a `release/v<version>` PR is merged, tag the merge commit on the base branch, create the
  GitHub release with changelog notes and optional artifact files, and delete the release branch. Prereleases are
  never marked latest. Supports an approval environment.
- `sync_dev`: merge the release branch into the development branch after a release, falling back to a `sync/*`
  pull request on conflicts or rejected pushes.
- `container_build`: build, smoke-test and push a container image with version, major.minor, `latest`, `sha-` and
  extra tags; prereleases get only their exact version tag.
- `pixi_lock_refresh`: regenerate `pixi.lock` on Dependabot pull requests and push it to the PR branch.
- `changelog_check`: fail pull requests that add no entry under `[Unreleased]`, with a skip label and skipped
  authors.
- `citation_check`: validate `CITATION.cff` with cffconvert and warn when its version differs from the latest tag.
- `python_test`: run tests across a matrix with pip, conda or pixi, with optional ruff, external tools from
  artifacts and Codecov upload.
- End-to-end tests for `release_prepare`, `release_publish`, `sync_dev`, `container_build`, `citation_check` and
  `python_test`, plus actionlint on every push and pull request.
- Release workflows for this repo itself (`release.yml`, `publish.yml`, which also moves the `v<major>` tag callers
  pin to) and a changelog check on its pull requests (`changelog.yml`).
- Dependabot updates for the pinned GitHub Actions.

[Unreleased]: https://github.com/stajichlab/gh_workflows/compare/v1.0.0...HEAD

[1.0.0]: https://github.com/stajichlab/gh_workflows/releases/tag/v1.0.0
