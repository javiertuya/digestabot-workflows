# digestabot-workflows

Reusable GitHub Actions workflows to keep pinned Docker base images up to date.

## Refresh pinned Docker digests

[`docker-digests.yml`](.github/workflows/docker-digests.yml) refreshes the digest of the base
images pinned as `FROM image:tag@sha256:...` in a set of Dockerfiles.
Dependabot only compares tags, so a base image rebuilt behind the same tag never gets a PR.
The workflow uses [digestabot](https://github.com/chainguard-dev/digestabot) to open a PR
with the new digests and adds two checks that digestabot does not make by itself:

- Fails if an image cannot be resolved (e.g. registry without login).
- Fails if an image was built more than `stale-days` ago (it has not been rebuilt upstream).

When the job fails, it opens an issue (or comments on the open one), assigned to `assignee` if set.

### Usage

The caller is a thin workflow, e.g. run manually. The job must grant the permissions below, because
a reusable workflow cannot raise the permissions of its caller.

```yaml
name: digestabot
on:
  workflow_dispatch:
jobs:
  update-digests:
    permissions:
      contents: write
      pull-requests: write
      issues: write
    uses: javiertuya/digestabot-workflows/.github/workflows/docker-digests.yml@v1.0.0
    with:
      image-name: node
      working-dir: docker
      assignee: your-user
    secrets:
      DIGESTABOT_TOKEN: ${{ secrets.DIGESTABOT_TOKEN }}
```

- Reference an exact release tag: with the `github-actions` ecosystem enabled, Dependabot proposes
  a PR in each caller for every new release, so a change does not reach all repositories at once.
- Secrets are passed explicitly because `secrets: inherit` only works inside one organization.
- Two values are fixed on purpose (not inputs), so that other tools can identify these PRs:
  the branch prefix `digestabot/docker/` and the PR label `digestabot`
  (e.g. [DashGit](https://github.com/javiertuya/dashgit) lists them in its Dependabot view).
- A second run while the previous PR is still open updates the same branch and PR.

### Inputs

| Input | Default | Description |
|---|---|---|
| `image-name` | (required) | Image name, used in the branch, PR title and issue title |
| `working-dir` | `dockerfiles` | Folder with the Dockerfiles |
| `include-files` | `Dockerfile*` | Dockerfile name pattern, basename only (matched with `find -name` inside `working-dir`) |
| `title-prefix` | `Update Docker image digests for` | PR title and commit message are `<title-prefix> <image-name>`; the PR branch is `digestabot/docker/<image-name>` |
| `stale-days` | `30` | Fail if an image was built more than this number of days ago |
| `dhi-login` | `false` | Log in to `dhi.io` before running (needs the DHI secrets) |
| `assignee` | (none) | User assigned and mentioned in the failure issue; if empty, the issue is not assigned |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `DIGESTABOT_TOKEN` | yes | Personal access token, see below |
| `DHI_USERNAME`, `DHI_TOKEN` | only if `dhi-login` | Docker Hub user and access token (read-only is enough), the same credentials used with `docker login dhi.io` |

### Tokens

**`DIGESTABOT_TOKEN`** is a personal access token (PAT) used to push the branch and open the PR.
It is used in `actions/checkout` too, because digestabot pushes with the checkout credentials.
A PAT is needed instead of `GITHUB_TOKEN` because pushes and PRs made with `GITHUB_TOKEN` do not
trigger other workflows, so the CI of the PR would never run. The PR author is the token owner.

| Token type | Required configuration |
|---|---|
| Classic | Scope `repo`. Scope `workflow` is not needed because only Dockerfiles are modified |
| Fine-grained | *Repository access*: the repositories where the workflow runs. *Repository permissions*: `Contents: Read and write` (push the branch), `Pull requests: Read and write` (open the PR and add labels), `Metadata: Read-only` (added automatically) |

Notes:

- A fine-grained token belongs to a single resource owner (a user or an organization), so
  repositories of different owners need a different token each, stored with the same secret name.
  Organizations may require approval of fine-grained tokens, and may restrict classic ones.
- The issue on failure uses the `GITHUB_TOKEN` of the job, so it only needs `issues: write` in the
  caller permissions (shown above).
- With a PAT, the setting *Allow GitHub Actions to create and approve pull requests* is not needed.
