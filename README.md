# plusminus-workflows

Reusable GitHub Actions workflows shared by the Plusminus projects.

## `maven-central-release.yml`

Builds, tests and publishes Maven artifacts to Maven Central. It is a `workflow_call`
(reusable) workflow — other repositories call it instead of duplicating the release steps.

### Usage

```yaml
jobs:
  release:
    uses: plusminus-software/plusminus-workflows/.github/workflows/maven-central-release.yml@main
    with:
      java-version: "8"        # optional, default "21"
    secrets: inherit
```

### Inputs

| Input | Default | Description |
|---|---|---|
| `java-version` | `"21"` | Temurin JDK version used for the build |
| `module` | `""` | Release only this module (comma-separated). Empty = resolve automatically |
| `create-git-tag` | `true` | Tag the released versions after a successful deploy |

### Secrets

`MAVEN_CENTRAL_USERNAME`, `MAVEN_CENTRAL_PASSWORD`, `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE`
are all required.

### What it does

1. **Resolves the target modules.** An explicit `module` input always wins. Otherwise, on a
   `push` it diffs against the previous commit and releases only the top-level directories that
   changed and contain a `pom.xml`; a change to the root `pom.xml`, `.mvn/` or `.github/`, an
   unavailable base commit, or any non-push trigger falls back to releasing everything.
2. **Plans the release.** Three repository layouts are supported: a *reactor* (root `pom.xml`
   with `<modules>`), a *single* project (root `pom.xml`, no modules) and *standalone* (no root
   `pom.xml`; every top-level directory with a `pom.xml` is its own project, released in
   alphabetical order so dependencies publish before their consumers).
3. **Picks the action per project.** `-SNAPSHOT` versions deploy on any trigger; release
   versions deploy — GPG-signed — only on a manual `workflow_dispatch` run, and the plan fails
   early if that exact version is already on Central.
4. **Builds, deploys, tags** the released versions (unless `create-git-tag: false`) and
   publishes a job summary.

## `claude.yml`

Answers a `@claude` mention in an issue or a pull request comment, and reviews a pull request
when it is opened. Pull requests from forks are not reviewed, because they get no secrets.

### Usage

```yaml
name: Claude

on:
  pull_request:
    types: [opened]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  claude:
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    uses: plusminus-software/plusminus-workflows/.github/workflows/claude.yml@main
    secrets:
      token: ${{ secrets.<name of your secret> }}
```

The `permissions` block is required: a reusable workflow cannot have more permissions than the
job that calls it.

GitHub runs a comment-triggered workflow from the default branch, so the calling workflow does
nothing until it is on `main`.

### Inputs

| Input | Default | Description |
|---|---|---|
| `model` | `claude-opus-5` | Model used for answers and reviews |

### Secrets

`token` is required: a Claude Code OAuth token created with `claude setup-token`. Runs are paid
for by the Claude subscription that produced the token. The calling repository chooses the name
of its own secret and maps it to `token`.

### Requirements

The Claude GitHub app must be installed on the calling repository.
