# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
