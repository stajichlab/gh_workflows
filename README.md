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

## Versioning

Pin callers to a major tag (`@v1`). Within `v1`, inputs and outputs stay compatible; the `v1` tag is moved to
each new compatible release. Breaking changes get `v2`.

## Development

- `lint.yml` runs [actionlint](https://github.com/rhysd/actionlint) on every push to `main` and every PR.
- `test_release_prepare.yml` runs `release_prepare` end to end against a throwaway orphan branch with a `v0.1.0`
  tag, checks the release branch and the draft PR, then deletes the PR, branches and tag. It runs on pushes to
  `main` that touch either workflow, or manually. It needs **Allow GitHub Actions to create and approve pull
  requests** enabled in this repo.
- Dependabot updates the pinned action SHAs monthly.
