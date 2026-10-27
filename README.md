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
    uses: stajichlab/gh_workflows/.github/workflows/release_sync_dev.yml@v1
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

## release_sync_dev

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
    uses: stajichlab/gh_workflows/.github/workflows/release_sync_dev.yml@v1
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
- `test_release_sync_dev.yml` runs `release_sync_dev` end to end on three throwaway branch pairs (already up to
  date, clean merge, conflict), checks the results and the conflict PR, then deletes the PR and branches. The
  "push rejected" fallback needs branch protection and is not covered. Same triggers and repo setting as above.
- `test_container_build.yml` builds a tiny image from a throwaway orphan branch for a release and a prerelease
  version, runs the smoke test and checks the tags. It never pushes.
- Dependabot updates the pinned action SHAs monthly.
