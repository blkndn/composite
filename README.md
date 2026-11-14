# composite

Shared CI building blocks of the Bulkynaden libraries (`blkndn/composite`):

| File | Kind | Used by |
|---|---|---|
| `.github/workflows/library-release.yml` | reusable workflow | each library's release caller (`package.yml`) |
| `.github/workflows/bump-library.yml` | reusable workflow | each library consumer's `bump-library.yml` |
| `get-and-set-new-version/` | composite action (legacy) | spring-hierarchy, until it moves to `library-release.yml` |

## Referencing a reusable workflow

Pin the caller to the full commit SHA of a composite release tag, never to a branch or a moving tag, so that every change to how libraries are released is reviewed in the library that adopts it:

```sh
gh api repos/blkndn/composite/git/ref/tags/v1.0.10 --jq .object.sha
```

The `uses:` line then reads `blkndn/composite/.github/workflows/library-release.yml@<that SHA>`, followed by one space, `#`, one space and the tag (`v1.0.10`): the only comment the workspace allows in YAML, which Dependabot keeps up to date. Dependabot's `github-actions` ecosystem proposes new composite pins; they are not auto-merged, because they change how libraries are published.

## `library-release.yml`

Releases a library from its change fragments (ADR-0017). It runs on every push to the library's `main`:

1. **`prepare`** checks out the tip of `main` and decides:
   - no fragment in `.changes/unreleased` with a releasable kind (`Breaking`, `Added`, `Deprecated`, `Fixed`, `Security`, `Dependencies`): nothing is published, and the run summary says `Nothing to publish: N commits since vX.Y.Z` (or `since the seed vX.Y.Z (no release tag yet)` before the first tagged release). `Internal` fragments alone never release, but they stay in the notes of the next release;
   - a fragment with an unknown kind, or a library without a seed version in `.changes`, fails the run;
   - otherwise `changie next auto` gives the version. The job batches the fragments (`changie batch`, `changie merge`), fails if the registry already holds that version (a wrong seed), builds and tests with `-PreleaseVersion=X.Y.Z`, commits `Release vX.Y.Z` (`CHANGELOG.md` and `.changes/`), tags `vX.Y.Z` and pushes both atomically. A failed build pushes nothing. The run summary states the version and the notes.
2. **`publish`** builds the tag and runs `./gradlew publish -PreleaseVersion=X.Y.Z`, unless the registry already holds that version.
3. **`github-release`** creates the GitHub Release with the batch's notes, unless it exists.
4. **`sync-branches`** merges the release commit into `dev` and `integration` through the merges API.
5. **`notify-consumers`** sends `repository_dispatch` (`library-released`) to each consumer with the App token. The payload carries `group`, `artifact`, `version` (without `v`) and `repository` (the library's repository name).

Pushes and merges made with `GITHUB_TOKEN` start no workflows, so the release commit does not trigger another release. One release runs at a time per repository (concurrency group `library-release-<owner>/<repo>`, never cancelled); a caller must not declare the same group.

### Inputs

| Input | Type | Default | Meaning |
|---|---|---|---|
| `group` | string | required | Maven group id, `es.bulkynaden` or a subgroup |
| `artifact` | string | required | Maven artifact id |
| `consumers` | string | `''` | comma- or newline-separated repositories of the same owner to notify |
| `sync-branches` | string | `dev integration` | space-separated branches that receive the release commit |
| `resume-tag` | string | `''` | existing release tag to complete instead of batching (see recovery) |
| `dry-run` | boolean | `false` | show the next version and its notes without releasing |
| `java-version` | string | `25` | JDK version of the build |
| `java-distribution` | string | `corretto` | JDK distribution of the build |
| `changie-version` | string | `v1.26.0` | changie release to install |
| `app-client-id` | string | `''` | client ID of the release GitHub App; required when `consumers` is set |

### Secrets

Pass each one explicitly; never `secrets: inherit`.

| Secret | Meaning |
|---|---|
| `packages-read-token` | token that reads the other Bulkynaden packages while building; `GITHUB_TOKEN` when absent |
| `app-private-key` | private key of the release GitHub App; used only to mint a token for the consumers, with `contents: write` |

Output: `version`, the released version, empty when nothing was released (also in a dry run).

### What the library needs

- `.changie.yaml` with the kinds of ADR-0017 (`Breaking` major; `Added`, `Deprecated` minor; `Fixed`, `Security`, `Dependencies` patch; `Internal` none), `.changes/header.tpl.md` and `.changes/unreleased/.gitkeep`.
- A seed `.changes/v<latest published>.md`, so that changie continues from the published version.
- `build.gradle` takes its version from `-PreleaseVersion` (`0.0.0-SNAPSHOT` otherwise) and publishes with the `GITHUB_TOKEN` environment variable.
- The organisation variable `RELEASE_APP_CLIENT_ID` and secret `RELEASE_APP_PRIVATE_KEY`, and the App installed on every consumer with Contents and Pull requests write.

### Caller

spring-common's `.github/workflows/package.yml`:

```yaml
name: Release

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      resume-tag:
        description: Existing release tag to complete, such as v2.0.0
        required: false
        type: string
        default: ''
      dry-run:
        description: Show the next version and its notes without releasing
        required: false
        type: boolean
        default: false

permissions: {}

jobs:
  release:
    uses: blkndn/composite/.github/workflows/library-release.yml@<full SHA of the composite tag>
    permissions:
      contents: write
      packages: write
    with:
      group: es.bulkynaden.spring
      artifact: common
      consumers: spring.events,educontrol
      resume-tag: ${{ inputs.resume-tag || '' }}
      dry-run: ${{ inputs.dry-run == true }}
      app-client-id: ${{ vars.RELEASE_APP_CLIENT_ID }}
    secrets:
      packages-read-token: ${{ secrets.MY_GITHUB_PAT }}
      app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

- common: `group: es.bulkynaden`, `artifact: common`, `consumers: spring.common,educontrol`, no `packages-read-token`.
- spring-events: `group: es.bulkynaden.spring`, `artifact: events`, `consumers: educontrol`, with `packages-read-token`.

### Dry run

Run the caller by hand (**Actions → Release → Run workflow**) with `dry-run` ticked. The run summary shows the version the next release would get and its batched notes, and whether the registry already holds that version. Nothing is built, committed, tagged, pushed, published, released, merged or sent: every job after `prepare` is skipped. `dry-run` together with `resume-tag` is an error.

### Recovery

- **The build or the tests fail in `prepare`:** nothing was pushed. Fix it through the normal flow; the next push to `main` releases.
- **A job after `prepare` fails** (`publish`, `github-release`, `sync-branches`, `notify-consumers`): use **Re-run failed jobs**. GitHub keeps `prepare`'s outputs, and every later job is idempotent: `publish` skips a version the registry already holds, the GitHub Release is not created twice, a merge with nothing to merge does nothing, and a repeated dispatch finds the bump already open.
- **The run was re-run with "Re-run all jobs", or the re-run window has passed:** `prepare` finds no fragments and does nothing. Run the caller by hand with `resume-tag: vX.Y.Z`. It checks that the value looks like `vX.Y.Z`, that the tag exists and is on `main`, and that its commit is named `Release vX.Y.Z`, and then runs the same idempotent jobs for that tag.
- **A half-uploaded package version:** GitHub Packages rejects a second upload of a version (HTTP 409). Delete that version in the package's settings (**Packages → the package → Manage versions**), then re-run the failed jobs or use `resume-tag`.
- **`sync-branches` reports a conflict:** a released fragment or `CHANGELOG.md` was edited on `dev` or `integration`. Merge `origin/main` into a `chore/sync-release-vX.Y.Z` task branch from that branch and close it like any task.
- **The seed guard fails** (the registry already holds the computed version): the seed in `.changes` is behind the registry. Correct the seed through the normal flow.

## `bump-library.yml`

Runs in a library consumer when it receives `library-released`. It:

1. **validates the payload** before using or printing it: `group` must be `es.bulkynaden` or a subgroup, `artifact` a lower-case artifact id, `version` exactly `X.Y.Z`, `repository` a repository name, and `group:artifact` one of the caller's `accepted` coordinates. Anything else fails the run;
2. stops if a pull request for the bump branch already exists, and opens one if the branch exists without it;
3. creates the branch from `base-branch` and replaces every quoted literal `'<group>:<artifact>:<version>'` in the tracked `*.gradle` and `*.gradle.kts` files. The match starts at the opening quote, so `es.bulkynaden:common` never touches `es.bulkynaden.spring:common` and vice versa. No declaration fails the run; a declared version newer than the release (a downgrade) or equal to it ends the run with nothing to do;
4. in a library (`library: true`), adds a `Dependencies` change fragment;
5. commits as the App, pushes and opens the pull request into `base-branch`. On a major jump the body asks to adapt the code and, in a library, to change the fragment's kind to `Breaking` if the dependency is part of the repository's API;
6. closes every open pull request of an older bump of the same library, with a comment that names the new one. Their branches are kept.

| Coordinate | Branch |
|---|---|
| `es.bulkynaden:common` | `chore/bump-common-<version>` |
| `es.bulkynaden.spring:common` | `chore/bump-spring-common-<version>` |
| `es.bulkynaden.spring:events` | `chore/bump-spring-events-<version>` |

`repository_dispatch` only starts workflows on the consumer's default branch (`main`). Until the consumer's caller has reached `main`, a dispatch is accepted and nothing runs; the fallback is the manual consumer upgrade.

### Inputs and secrets

| Input | Type | Default | Meaning |
|---|---|---|---|
| `group` | string | required | from the payload |
| `artifact` | string | required | from the payload |
| `version` | string | required | from the payload |
| `source-repository` | string | `''` | from the payload; links the release notes |
| `accepted` | string | required | newline-separated `group:artifact` coordinates this repository upgrades |
| `library` | boolean | `false` | whether this repository is a library and needs a fragment |
| `base-branch` | string | `integration` | branch the bump starts from and the pull request targets |
| `app-client-id` | string | required | client ID of the release GitHub App |

| Secret | Meaning |
|---|---|
| `app-private-key` | private key of the release GitHub App; used only to mint a token for this repository with `contents: write` and `pull-requests: write` |

### Caller

Educontrol's `.github/workflows/bump-library.yml`:

```yaml
name: Bump library

on:
  repository_dispatch:
    types:
      - library-released

permissions: {}

jobs:
  bump:
    uses: blkndn/composite/.github/workflows/bump-library.yml@<full SHA of the composite tag>
    with:
      group: ${{ github.event.client_payload.group }}
      artifact: ${{ github.event.client_payload.artifact }}
      version: ${{ github.event.client_payload.version }}
      source-repository: ${{ github.event.client_payload.repository }}
      accepted: |
        es.bulkynaden:common
        es.bulkynaden.spring:common
        es.bulkynaden.spring:events
      app-client-id: ${{ vars.RELEASE_APP_CLIENT_ID }}
    secrets:
      app-private-key: ${{ secrets.RELEASE_APP_PRIVATE_KEY }}
```

- spring-common: `accepted: es.bulkynaden:common` and `library: true`.
- spring-events: `accepted: es.bulkynaden.spring:common` and `library: true`.

## This repository's CI

Changes follow the workspace flow: task branch → `integration` → `dev` → `main`.

- `merge-to-main.yml`: on a push to `dev`, lints every workflow with actionlint 1.7.12 (its Docker image, pinned by digest, which bundles shellcheck), then merges `dev` into `main` with the personal token, so that the push starts `release.yml`.
- `release.yml`: on a push to `main`, creates the next patch release (`gh release create` on the pushed commit, with generated notes) after the highest `vX.Y.Z` tag.
- `auto-merge.yml`: enables auto-merge on Dependabot's minor and patch updates of the workflows into `integration`.
- `dependabot.yml`: daily `github-actions` updates into `integration`.

Every third-party action is pinned to a full commit SHA, followed by its version tag.

## `get-and-set-new-version` (legacy)

Fetches the latest version of a package from GitHub Packages, compares it with the current version and decides the new one: the next patch, unless the current version is ahead in major or minor. Libraries on `library-release.yml` no longer use it; it stays while spring-hierarchy does.

### Inputs

- `github_token` (**required**): token that reads the package versions from GitHub Packages.
- `package_url` (**required**): API URL of the package versions, such as `https://api.github.com/orgs/blkndn/packages/maven/es.bulkynaden.spring.exceptions`.
- `current_version` (**required**): the version in the build file, such as `1.0.0`.

### Usage

```yaml
- name: Get the latest published version and decide the new one
  uses: blkndn/composite/get-and-set-new-version@<full SHA of the composite tag>
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    package_url: 'https://api.github.com/orgs/blkndn/packages/maven/es.bulkynaden.spring.exceptions'
    current_version: ${{ env.CURRENT_VERSION }}
```
