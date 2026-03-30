# DIT GitHub Actions

Shared composite actions for CI/CD at [Utdanningsdirektoratet](https://github.com/Utdanningsdirektoratet) DIT.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/{action}@v1
```

**Built-in caching everywhere** — dependency stores, Next.js incremental builds, Playwright browsers, Docker layers. Your pipelines are fast by default without any extra configuration.

**Update once, optimize all** — upstream action versions (`actions/checkout`, `actions/cache`, `docker/build-push-action`, etc.) are managed centrally. A single version bump here speeds up every pipeline across the projects.

**Composable by design** — each action does one thing. Chain them together for any workflow: checkout → runtimes → install → build → containerize → tag → deploy → rollout.

---

**Git:** [`git/checkout`](#gitcheckout), [`git/commit`](#gitcommit)

**GitHub:** [`github/checkout`](#githubcheckout), [`github/runtimes`](#githubruntimes), [`github/clone`](#githubclone), [`github/push`](#githubpush), [`github/pr`](#githubpr), [`github/pr-merge`](#githubpr-merge)

**GoUpdate:** [`goupdate/install`](#goupdateinstall), [`goupdate/scan`](#goupdatescan), [`goupdate/outdated`](#goupdateoutdated), [`goupdate/update`](#goupdateupdate)

**JavaScript:** [`js/install`](#jsinstall), [`js/nextjs`](#jsnextjs), [`js/playwright`](#jsplaywright)

**DotNet:** [`dotnet/install`](#dotnetinstall), [`dotnet/build`](#dotnetbuild)

**Docker:** [`docker/build`](#dockerbuild), [`docker/tag`](#dockertag)

**Kubernetes:** [`kubernetes/deploy`](#kubernetesdeploy), [`kubernetes/rollout`](#kubernetesrollout)

**DXP:** [`dxp/upload`](#dxpupload), [`dxp/deploy`](#dxpdeploy)

## Quick Start

Real production workflows from DIT projects. Copy, adjust secrets/vars, ship.

<details>
<summary><b>Next.js — PR validation with build + lint</b> (<a href="https://github.com/Utdanningsdirektoratet/komp-frontend-react"><code>Utdanningsdirektoratet/komp-frontend-react</code></a>)</summary>

Validates every pull request with a full Next.js build and lint check in parallel. The build job also pushes an image to verify the Dockerfile works.

```yaml
name: Pull Request

on:
  pull_request:
    branches: [stage, main]

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build
    runs-on: runners-ditiac-stage
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: "24"
          node-pms: pnpm
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/js/nextjs@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/docker/build@v1
        with:
          registry: ${{ vars.STAGE_REGISTRY_HOST }}
          username: ${{ secrets.STAGE_REGISTRY_USERNAME }}
          password: ${{ secrets.STAGE_REGISTRY_PASSWORD }}
          image: ${{ vars.IMAGE_NAME }}
          tag-latest: latest-stage
          push-by-digest: "true"

  lint:
    name: Lint
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: "24"
          node-pms: pnpm
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - run: pnpm run lint
```

</details>

<details>
<summary><b>Next.js — Deploy to stage</b> (<a href="https://github.com/Utdanningsdirektoratet/komp-frontend-react"><code>Utdanningsdirektoratet/komp-frontend-react</code></a>)</summary>

Full pipeline: build Next.js with caching → push image by digest → tag → deploy to Kubernetes → wait for rollout. Production is the same pattern with different secrets/vars and a `release` trigger.

```yaml
name: Release - Stage

on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Image tag name"
        required: true
  push:
    branches: [stage]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build
    runs-on: runners-ditiac-stage
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: "24"
          node-pms: pnpm
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/js/nextjs@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/docker/build@v1
        id: build
        with:
          registry: ${{ vars.STAGE_REGISTRY_HOST }}
          username: ${{ secrets.STAGE_REGISTRY_USERNAME }}
          password: ${{ secrets.STAGE_REGISTRY_PASSWORD }}
          image: ${{ vars.IMAGE_NAME }}
          tag-latest: latest-stage
          push-by-digest: "true"

  tag:
    name: Tag
    needs: build
    runs-on: runners-ditiac-stage
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
      image-ref: ${{ steps.tag.outputs.image-ref }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/docker/tag@v1
        id: tag
        with:
          registry: ${{ vars.STAGE_REGISTRY_HOST }}
          username: ${{ secrets.STAGE_REGISTRY_USERNAME }}
          password: ${{ secrets.STAGE_REGISTRY_PASSWORD }}
          image: ${{ vars.IMAGE_NAME }}
          digest: ${{ needs.build.outputs.digest }}
          tag: ${{ inputs.tag }}
          tag-latest: latest-stage

  deploy:
    name: Deploy
    needs: tag
    runs-on: runners-ditiac-stage
    environment:
      name: stage
      url: https://kp.ditiac-stage.udir.no/
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/deploy@v1
        with:
          kube-config: ${{ secrets.STAGE_K8S_KUBE_CONFIG }}
          namespace: ${{ vars.K8S_NAMESPACE }}
          deployment: ${{ vars.K8S_DEPLOYMENT }}
          image: ${{ needs.tag.outputs.image-ref }}

      - uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/rollout@v1
        with:
          kube-config: ${{ secrets.STAGE_K8S_KUBE_CONFIG }}
          namespace: ${{ vars.K8S_NAMESPACE }}
          deployment: ${{ vars.K8S_DEPLOYMENT }}
```

</details>

<details>
<summary><b>Docker — Deploy to production</b> (<a href="https://github.com/Utdanningsdirektoratet/kurs-lms-moodle"><code>Utdanningsdirektoratet/kurs-lms-moodle</code></a>)</summary>

Docker-only pipeline (no Next.js build step). Triggered by GitHub release or manual dispatch. Build args pass the version into the image.

```yaml
name: Release - Production

on:
  workflow_dispatch:
    inputs:
      tag:
        description: "Image tag name"
        required: true
  release:
    types: [published]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build
    runs-on: runners-ditiac-prod
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/docker/build@v1
        id: build
        with:
          registry: ${{ vars.PROD_REGISTRY_HOST }}
          username: ${{ secrets.PROD_REGISTRY_USERNAME }}
          password: ${{ secrets.PROD_REGISTRY_PASSWORD }}
          image: ${{ vars.IMAGE_NAME }}
          push-by-digest: "true"
          build-args: |
            BUILDKIT_INLINE_CACHE=1
            VERSION=${{ github.event.release.tag_name || github.event.inputs.tag }}

  tag:
    name: Tag
    needs: build
    runs-on: runners-ditiac-prod
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
      image-ref: ${{ steps.tag.outputs.image-ref }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/docker/tag@v1
        id: tag
        with:
          registry: ${{ vars.PROD_REGISTRY_HOST }}
          username: ${{ secrets.PROD_REGISTRY_USERNAME }}
          password: ${{ secrets.PROD_REGISTRY_PASSWORD }}
          image: ${{ vars.IMAGE_NAME }}
          digest: ${{ needs.build.outputs.digest }}
          tag: ${{ github.event.release.tag_name || github.event.inputs.tag }}

  deploy:
    name: Deploy
    needs: tag
    runs-on: runners-ditiac-prod
    environment:
      name: production
      url: https://kurs.udir.no/
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/deploy@v1
        with:
          kube-config: ${{ secrets.PROD_K8S_KUBE_CONFIG }}
          namespace: ${{ vars.K8S_NAMESPACE }}
          deployment: ${{ vars.K8S_DEPLOYMENT }}
          image: ${{ needs.tag.outputs.image-ref }}

      - uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/rollout@v1
        with:
          kube-config: ${{ secrets.PROD_K8S_KUBE_CONFIG }}
          namespace: ${{ vars.K8S_NAMESPACE }}
          deployment: ${{ vars.K8S_DEPLOYMENT }}
```

</details>

<details>
<summary><b>E2E tests with Playwright</b> (<a href="https://github.com/matematikk-mooc/frontend"><code>matematikk-mooc/frontend</code></a>)</summary>

Clones a separate test repository, installs Playwright with browser caching, then runs E2E tests against the app.

```yaml
name: Pull Request

on:
  pull_request:
    branches: [stage, main]

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: "24"
          node-pms: pnpm
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - run: pnpm run production

  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: "24"
          node-pms: pnpm

      - uses: Utdanningsdirektoratet/dit-github-actions/github/clone@v1
        with:
          repository: matematikk-mooc/frontend-react
          path: .tests_playwright
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
        with:
          working-directory: .tests_playwright
      - uses: Utdanningsdirektoratet/dit-github-actions/js/playwright@v1
        with:
          working-directory: .tests_playwright
          browsers: chromium

      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - name: Run E2E tests
        run: ./e2e.sh
        env:
          APP_ENV: development
          TEST_CANVAS_LOCAL_THEME: "true"
          TEST_CANVAS_CHROMIUM_USERNAME: ${{ vars.TEST_CANVAS_CHROMIUM_USERNAME }}
          TEST_CANVAS_CHROMIUM_PASSWORD: ${{ secrets.TEST_CANVAS_CHROMIUM_PASSWORD }}
```

</details>

<details>
<summary><b>GoUpdate — Automated dependency updates with auto-merge</b> (<a href="https://github.com/matematikk-mooc/frontend"><code>matematikk-mooc/frontend</code></a>)</summary>

Runs weekly on Monday. Scans for outdated packages, applies updates, creates a PR, waits for CI checks, and auto-merges if everything passes. Skips auto-merge on partial failures and notifies the team.

```yaml
name: GoUpdate - Auto Update

on:
  schedule:
    - cron: "0 0 * * 1"
  workflow_dispatch:
    inputs:
      update-type:
        description: "Update type"
        required: true
        type: choice
        default: "minor"
        options:
          - major
          - minor
          - patch
      base-branch:
        description: "Base branch"
        required: true
        type: string
        default: "stage"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    name: Check
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      package-managers: ${{ steps.scan.outputs.package-managers }}
      rules: ${{ steps.scan.outputs.rules }}
      total-outdated: ${{ steps.outdated.outputs.total-outdated }}
      major: ${{ steps.outdated.outputs.major }}
      minor: ${{ steps.outdated.outputs.minor }}
      patch: ${{ steps.outdated.outputs.patch }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
        with:
          ref: ${{ github.event.inputs.base-branch || 'stage' }}
          fetch-depth: 0
      - uses: Utdanningsdirektoratet/dit-github-actions/git/checkout@v1
        with:
          branch: goupdate/auto-update-${{ github.event.inputs.update-type || 'minor' }}
          source-branch: ${{ github.event.inputs.base-branch || 'stage' }}
          create-branch: "false"
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/scan@v1
        id: scan
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: ${{ contains(steps.scan.outputs.package-managers, 'js') && '24' || '' }}
          node-pms: ${{ steps.scan.outputs.rules }}
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/outdated@v1
        id: outdated

  update:
    name: Update
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: check
    outputs:
      has-changes: ${{ steps.update.outputs.has-changes }}
      branch-name: ${{ steps.checkout-branch.outputs.branch-name }}
      partial-failure: ${{ steps.update.outputs.partial-failure }}
      diverged: ${{ steps.checkout-branch.outputs.diverged }}
    if: >-
      ((github.event.inputs.update-type || 'minor') == 'major' && fromJSON(needs.check.outputs['total-outdated']) > 0) ||
      ((github.event.inputs.update-type || 'minor') == 'minor' && (fromJSON(needs.check.outputs.minor) > 0 || fromJSON(needs.check.outputs.patch) > 0)) ||
      ((github.event.inputs.update-type || 'minor') == 'patch' && fromJSON(needs.check.outputs.patch) > 0)
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
        with:
          ref: ${{ github.event.inputs.base-branch || 'stage' }}
          fetch-depth: 0
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: ${{ contains(needs.check.outputs.package-managers, 'js') && '24' || '' }}
          node-pms: ${{ needs.check.outputs.rules }}
      - uses: Utdanningsdirektoratet/dit-github-actions/git/checkout@v1
        id: checkout-branch
        with:
          branch: goupdate/auto-update-${{ github.event.inputs.update-type || 'minor' }}
          source-branch: ${{ github.event.inputs.base-branch || 'stage' }}
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/update@v1
        id: update
        with:
          update-type: ${{ github.event.inputs.update-type || 'minor' }}
      - uses: Utdanningsdirektoratet/dit-github-actions/git/commit@v1
        if: steps.update.outputs.has-changes == 'true'
        with:
          message: "GoUpdate: ${{ github.event.inputs.update-type || 'minor' }} update"
      - uses: Utdanningsdirektoratet/dit-github-actions/github/push@v1
        if: steps.update.outputs.has-changes == 'true'
        with:
          app-id: ${{ secrets.GOUPDATE_APP_ID }}
          app-private-key: ${{ secrets.GOUPDATE_APP_PRIVATE_KEY }}
          branch: ${{ steps.checkout-branch.outputs.branch-name }}

  pr:
    name: Pull Request
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: [check, update]
    if: needs.update.outputs.has-changes == 'true'
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/pr@v1
        id: pr
        with:
          app-id: ${{ secrets.GOUPDATE_APP_ID }}
          app-private-key: ${{ secrets.GOUPDATE_APP_PRIVATE_KEY }}
          title: "GoUpdate: ${{ github.event.inputs.update-type || 'minor' }} ({date})"
          base: ${{ github.event.inputs.base-branch || 'stage' }}
          head: ${{ needs.update.outputs.branch-name }}
      - uses: Utdanningsdirektoratet/dit-github-actions/github/pr-merge@v1
        with:
          app-id: ${{ secrets.GOUPDATE_APP_ID }}
          app-private-key: ${{ secrets.GOUPDATE_APP_PRIVATE_KEY }}
          pr-number: ${{ steps.pr.outputs.pr-number }}
          wait-for-checks: 600
          notify: "@ajxudir"
          skip-merge-comment: ${{ needs.update.outputs.partial-failure == 'true' && 'Not all updates succeeded. Auto-merge skipped.' || '' }}
```

</details>

<details>
<summary><b>GoUpdate — Dependency scan only</b> (<a href="https://github.com/Utdanningsdirektoratet/dub-cms-optimizely"><code>Utdanningsdirektoratet/dub-cms-optimizely</code></a>)</summary>

Lightweight version that only checks for outdated packages without applying updates. Useful as a scheduled audit or dashboard check.

```yaml
name: GoUpdate - Scan

on:
  schedule:
    - cron: "0 0 * * 1"
  workflow_dispatch:
    inputs:
      base-branch:
        description: "Branch to scan"
        required: true
        type: string
        default: "stage"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  check:
    name: Check
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      package-managers: ${{ steps.scan.outputs.package-managers }}
      rules: ${{ steps.scan.outputs.rules }}
      total-outdated: ${{ steps.outdated.outputs.total-outdated }}
      major: ${{ steps.outdated.outputs.major }}
      minor: ${{ steps.outdated.outputs.minor }}
      patch: ${{ steps.outdated.outputs.patch }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
        with:
          ref: ${{ github.event.inputs.base-branch || 'stage' }}
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/scan@v1
        id: scan
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: ${{ contains(steps.scan.outputs.package-managers, 'js') && '24' || '' }}
          node-pms: ${{ steps.scan.outputs.rules }}
          dotnet-version: ${{ contains(steps.scan.outputs.package-managers, 'dotnet') && '8.0.x' || '' }}
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/outdated@v1
        id: outdated
```

</details>

<details>
<summary><b>DXP — Build and deploy</b> (<a href="https://github.com/Utdanningsdirektoratet/dub-cms-optimizely"><code>Utdanningsdirektoratet/dub-cms-optimizely</code></a>)</summary>

.NET project deployed to Optimizely DXP. Builds a NuGet package, uploads to blob storage, and triggers deployment.

```yaml
name: DXP Deploy

on:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      package-name: ${{ steps.package.outputs.package-name }}
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          dotnet-version: "8.0.x"
      - uses: Utdanningsdirektoratet/dit-github-actions/dotnet/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/dotnet/build@v1
        id: package
        with:
          app-name: myapp
      - uses: Utdanningsdirektoratet/dit-github-actions/dxp/upload@v1
        with:
          package-path: ${{ steps.package.outputs.package-path }}
          project-id: ${{ secrets.DXP_PROJECT_ID }}
          client-key: ${{ secrets.DXP_CLIENT_KEY }}
          client-secret: ${{ secrets.DXP_CLIENT_SECRET }}
      - uses: Utdanningsdirektoratet/dit-github-actions/dxp/deploy@v1
        with:
          action: deploy
          target-environment: Integration
          package-name: ${{ steps.package.outputs.package-name }}
          project-id: ${{ secrets.DXP_PROJECT_ID }}
          client-key: ${{ secrets.DXP_CLIENT_KEY }}
          client-secret: ${{ secrets.DXP_CLIENT_SECRET }}
          direct-deploy: "true"
```

</details>

---

## Action Reference

### Git

#### `git/checkout`

Create or checkout a branch from a source branch. If the branch exists, optionally fast-forward merges from source.

**Template variables** — `branch` supports: `{date}` (YYYY-MM-DD).

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/git/checkout@v1
  id: branch
  with:
    branch: "updates/{date}" # resolves → updates/2026-02-22
    source-branch: main
```

<details>
<summary>Inputs & Outputs</summary>

| Input           | Required | Default | Description                                         | Example                      |
| --------------- | :------: | ------- | --------------------------------------------------- | ---------------------------- |
| `branch`        |   yes    |         | Branch name (`{date}` template supported)           | `goupdate/auto-update-minor` |
| `source-branch` |   yes    |         | Source branch to create from or ff-merge            | `main`                       |
| `auto-ff`       |          | `true`  | Fast-forward merge from source on existing branches | `true`                       |
| `create-branch` |          | `true`  | Create branch from source if missing                | `true`                       |

| Output          | Description                         | Example              |
| --------------- | ----------------------------------- | -------------------- |
| `branch-name`   | Resolved branch name                | `updates/2026-02-22` |
| `branch-exists` | Whether branch already existed      | `true`               |
| `diverged`      | Whether branch diverged from source | `false`              |

</details>

---

#### `git/commit`

Stage and commit all changes.

**Template variables** — `message` supports: `{date}` (YYYY-MM-DD), `{branch}` (current branch).

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/git/commit@v1
  with:
    message: "GoUpdate: minor update ({date})"
```

<details>
<summary>Inputs & Outputs</summary>

| Input            | Required | Default                                        | Description                                     | Example                         |
| ---------------- | :------: | ---------------------------------------------- | ----------------------------------------------- | ------------------------------- |
| `message`        |          | `GoUpdate: Update`                             | Commit message (`{date}`, `{branch}` templates) | `"chore: update deps ({date})"` |
| `git-user-name`  |          | `github-actions[bot]`                          | Commit author name                              | `my-bot`                        |
| `git-user-email` |          | `github-actions[bot]@users.noreply.github.com` | Commit author email                             | `bot@example.com`               |

| Output        | Description                        | Example |
| ------------- | ---------------------------------- | ------- |
| `has-changes` | Whether any changes were committed | `true`  |

</details>

---

### GitHub

#### `github/checkout`

Checkout repository. Wraps `actions/checkout` so the upstream version is managed in one place.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
```

<details>
<summary>Inputs</summary>

| Input         | Required | Default               | Description                                        | Example                   |
| ------------- | :------: | --------------------- | -------------------------------------------------- | ------------------------- |
| `ref`         |          | `""`                  | Branch, tag, or SHA to checkout                    | `main`, `v1.0.0`          |
| `fetch-depth` |          | `1`                   | Number of commits to fetch (`0` for full history)  | `0`                       |
| `submodules`  |          | `false`               | Checkout submodules (`true`, `false`, `recursive`) | `recursive`               |
| `token`       |          | `${{ github.token }}` | PAT or GitHub token for private repos              | `${{ secrets.GH_TOKEN }}` |
| `path`        |          | `""`                  | Relative path under `$GITHUB_WORKSPACE`            | `./sub-dir`               |
| `lfs`         |          | `false`               | Download Git LFS files                             | `true`                    |

</details>

---

#### `github/runtimes`

Setup language runtimes. All inputs opt-in — only specified runtimes are installed.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    node-version: "24"
    node-pms: "pnpm"
```

<details>
<summary>Inputs</summary>

| Input            | Required | Default | Description                        | Example                |
| ---------------- | :------: | ------- | ---------------------------------- | ---------------------- |
| `node-version`   |          | `""`    | Node.js version                    | `"24"`                 |
| `node-pms`       |          | `""`    | Package managers (comma-separated) | `"pnpm"`, `"npm,yarn"` |
| `php-version`    |          | `""`    | PHP version                        | `"8.3"`                |
| `python-version` |          | `""`    | Python version                     | `"3.12"`               |
| `go-version`     |          | `""`    | Go version                         | `"1.22"`               |
| `dotnet-version` |          | `""`    | .NET version                       | `"8.0.x"`              |

</details>

---

#### `github/clone`

Clone a repository using GitHub App token or fallback token. For private repos, authenticate with either a GitHub App (`app-id` + `app-private-key`) or a `github-token`. Public repos need no credentials.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/clone@v1
  with:
    repository: Utdanningsdirektoratet/shared-config
    path: ./config
```

<details>
<summary>Inputs & Outputs</summary>

| Input             | Required | Default | Description                            | Example                          |
| ----------------- | :------: | ------- | -------------------------------------- | -------------------------------- |
| `repository`      |   yes    |         | Repository to clone (`owner/repo`)     | `Utdanningsdirektoratet/my-repo` |
| `path`            |   yes    |         | Directory to clone into                | `./config`                       |
| `ref`             |          | `""`    | Branch, tag, or SHA to checkout        | `main`, `v1.0.0`                 |
| `depth`           |          | `1`     | Clone depth (`0` = full history)       | `1`                              |
| `app-id`          |          | `""`    | GitHub App ID                          | `${{ secrets.APP_ID }}`          |
| `app-private-key` |          | `""`    | GitHub App private key                 | `${{ secrets.APP_KEY }}`         |
| `github-token`    |          | `""`    | Fallback token (if no App credentials) | `${{ secrets.GH_TOKEN }}`        |

| Output | Description              | Example          |
| ------ | ------------------------ | ---------------- |
| `sha`  | SHA of the cloned commit | `abc1234def5678` |

</details>

---

#### `github/push`

Push commits with optional GitHub App token to trigger downstream workflows. Use `app-id` + `app-private-key` to trigger workflows. `github-token` pushes silently.

> ❗ Credentials are cleaned up in an `if: always()` step, even on failure.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/push@v1
  with:
    app-id: ${{ secrets.APP_ID }}
    app-private-key: ${{ secrets.APP_KEY }}
    branch: feature/my-branch
```

<details>
<summary>Inputs & Outputs</summary>

| Input             | Required | Default | Description                          | Example                      |
| ----------------- | :------: | ------- | ------------------------------------ | ---------------------------- |
| `app-id`          |          | `""`    | GitHub App ID (triggers workflows)   | `${{ secrets.APP_ID }}`      |
| `app-private-key` |          | `""`    | GitHub App private key               | `${{ secrets.APP_KEY }}`     |
| `github-token`    |          | `""`    | Fallback token (no workflow trigger) | `${{ secrets.GH_TOKEN }}`    |
| `branch`          |   yes    |         | Branch to push                       | `goupdate/auto-update-minor` |

| Output   | Description                 | Example          |
| -------- | --------------------------- | ---------------- |
| `pushed` | Whether changes were pushed | `true`           |
| `sha`    | SHA of the pushed commit    | `abc1234def5678` |

</details>

---

#### `github/pr`

Create or update a pull request. If a PR already exists for the same head/base, its title and body are updated.

**Template variables** — `title` and `body` support: `{date}` (YYYY-MM-DD), `{base}`, `{head}`.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/pr@v1
  id: pr
  with:
    app-id: ${{ secrets.APP_ID }}
    app-private-key: ${{ secrets.APP_KEY }}
    title: "GoUpdate: minor update ({date})"
    base: main
    head: goupdate/auto-update-minor
```

<details>
<summary>Inputs & Outputs</summary>

| Input             | Required | Default | Description                                       | Example                        |
| ----------------- | :------: | ------- | ------------------------------------------------- | ------------------------------ |
| `app-id`          |          | `""`    | GitHub App ID                                     | `${{ secrets.APP_ID }}`        |
| `app-private-key` |          | `""`    | GitHub App private key                            | `${{ secrets.APP_KEY }}`       |
| `github-token`    |          | `""`    | Fallback token                                    | `${{ secrets.GH_TOKEN }}`      |
| `title`           |   yes    |         | PR title (`{date}`, `{base}`, `{head}` templates) | `"Update deps ({date})"`       |
| `body`            |          | `""`    | PR body (`{date}`, `{base}`, `{head}` templates)  | `"Merging {head} into {base}"` |
| `base`            |   yes    |         | Base branch                                       | `main`                         |
| `head`            |   yes    |         | Head branch                                       | `goupdate/auto-update-minor`   |

| Output      | Description                  | Example                               |
| ----------- | ---------------------------- | ------------------------------------- |
| `pr-number` | PR number                    | `42`                                  |
| `pr-url`    | PR URL                       | `https://github.com/org/repo/pull/42` |
| `created`   | Whether a new PR was created | `true`                                |

</details>

---

#### `github/pr-merge`

Wait for checks and merge a pull request. Polls until checks pass (or timeout), then merges and deletes the branch.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/pr-merge@v1
  with:
    app-id: ${{ secrets.APP_ID }}
    app-private-key: ${{ secrets.APP_KEY }}
    pr-number: ${{ steps.pr.outputs.pr-number }}
```

<details>
<summary>Inputs & Outputs</summary>

| Input                | Required | Default  | Description                                      | Example                                         |
| -------------------- | :------: | -------- | ------------------------------------------------ | ----------------------------------------------- |
| `app-id`             |          | `""`     | GitHub App ID                                    | `${{ secrets.APP_ID }}`                         |
| `app-private-key`    |          | `""`     | GitHub App private key                           | `${{ secrets.APP_KEY }}`                        |
| `github-token`       |          | `""`     | Fallback token                                   | `${{ secrets.GH_TOKEN }}`                       |
| `pr-number`          |   yes    |          | PR number to merge                               | `42`                                            |
| `merge-method`       |          | `squash` | Merge strategy                                   | `squash`, `merge`, `rebase`                     |
| `wait-for-checks`    |          | `300`    | Timeout in seconds (`0` = skip)                  | `""`                                            |
| `check-interval`     |          | `15`     | Poll interval in seconds                         | `15`                                            |
| `notify`             |          | `""`     | Users/teams to ping on failure                   | `@myteam`                                       |
| `skip-merge-comment` |          | `""`     | If set, skip merge and post this as a PR comment | `"Skipping auto-merge: manual review required"` |

| Output          | Description           | Example |
| --------------- | --------------------- | ------- |
| `merged`        | Whether PR was merged | `true`  |
| `checks-passed` | Whether checks passed | `true`  |

</details>

---

### GoUpdate

#### `goupdate/install`

Install [goupdate](https://github.com/ajxudir/goupdate) binary from GitHub releases.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/goupdate/install@v1
```

<details>
<summary>Inputs & Outputs</summary>

| Input          | Required | Default            | Description            | Example                       |
| -------------- | :------: | ------------------ | ---------------------- | ----------------------------- |
| `version`      |          | `latest`           | Version or tag         | `v1.2.0`                      |
| `repo`         |          | `ajxudir/goupdate` | GitHub repo            | `ajxudir/goupdate`            |
| `github-token` |          | `""`               | Avoids API rate limits | `${{ secrets.GITHUB_TOKEN }}` |

| Output    | Description       | Example |
| --------- | ----------------- | ------- |
| `version` | Installed version | `1.2.0` |

</details>

---

#### `goupdate/scan`

Detect package managers and lock files in the repository.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/goupdate/scan@v1
```

<details>
<summary>Inputs & Outputs</summary>

| Input               | Required | Default | Description       | Example          |
| ------------------- | :------: | ------- | ----------------- | ---------------- |
| `working-directory` |          | `.`     | Directory to scan | `./packages/app` |

| Output             | Description                                 | Example                              |
| ------------------ | ------------------------------------------- | ------------------------------------ |
| `package-managers` | Detected package managers (comma-separated) | `npm,pip`                            |
| `rules`            | Detected rules (comma-separated)            | `package-lock.json,requirements.txt` |

</details>

---

#### `goupdate/outdated`

Check for outdated dependencies with major/minor/patch breakdown.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/goupdate/outdated@v1
```

<details>
<summary>Inputs & Outputs</summary>

| Input               | Required | Default | Description        | Example |
| ------------------- | :------: | ------- | ------------------ | ------- |
| `working-directory` |          | `.`     | Directory to check | `.`     |

| Output           | Description             | Example |
| ---------------- | ----------------------- | ------- |
| `total`          | Total dependencies      | `120`   |
| `total-outdated` | Total outdated          | `8`     |
| `major`          | Major updates available | `2`     |
| `minor`          | Minor updates available | `3`     |
| `patch`          | Patch updates available | `3`     |

</details>

---

#### `goupdate/update`

Apply dependency updates. Does **not** commit or push — use [`git/commit`](#gitcommit) and [`github/push`](#githubpush) afterward.

> ⚠️ Uses `--continue-on-fail`: if some updates fail but others succeed, `partial-failure=true` lets the workflow create a PR without auto-merging.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/goupdate/update@v1
  with:
    update-type: minor
```

<details>
<summary>Inputs & Outputs</summary>

| Input         | Required | Default | Description  | Example                   |
| ------------- | :------: | ------- | ------------ | ------------------------- |
| `update-type` |          | `minor` | Update level | `major`, `minor`, `patch` |

| Output            | Description                              | Example |
| ----------------- | ---------------------------------------- | ------- |
| `has-changes`     | Whether any files changed                | `true`  |
| `partial-failure` | Some updates failed but others succeeded | `false` |

</details>

---

### JavaScript

#### `js/install`

Install and cache Node.js dependencies. Auto-detects package manager from lock files: `pnpm-lock.yaml` > `yarn.lock` > `package-lock.json`.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    node-version: "24"
    node-pms: "pnpm"
- uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
```

<details>
<summary>Inputs & Outputs</summary>

| Input               | Required | Default | Description                   | Example               |
| ------------------- | :------: | ------- | ----------------------------- | --------------------- |
| `working-directory` |          | `.`     | Directory with `package.json` | `./frontend`          |
| `package-manager`   |          | auto    | Force specific PM             | `pnpm`, `yarn`, `npm` |

| Output            | Description                     | Example |
| ----------------- | ------------------------------- | ------- |
| `package-manager` | Detected/forced package manager | `pnpm`  |
| `cache-hit`       | Whether cache was restored      | `true`  |

</details>

---

#### `js/nextjs`

Build Next.js with `.next/cache` caching for incremental compilation. Auto-detects package manager from lock files for the build command.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/checkout@v1
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    node-version: "24"
    node-pms: pnpm
- uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
- uses: Utdanningsdirektoratet/dit-github-actions/js/nextjs@v1
```

> **Cache strategy:** `.next/cache` is keyed on lockfile hash + commit SHA, with fallback to any previous build from the same dependencies. This enables Next.js incremental compilation — only changed pages and components are rebuilt.

> **Package manager detection:** `pnpm-lock.yaml` → `pnpm run build`, `yarn.lock` → `yarn build`, `package-lock.json` → `npm run build`. Fails if no lockfile is found.

<details>
<summary>Inputs</summary>

| Input               | Required | Default | Description                                  | Example              |
| ------------------- | :------: | ------- | -------------------------------------------- | -------------------- |
| `build-env`         |          | `""`    | Extra env vars for build (`KEY=VALUE` lines) | `APP_VERSION=v1.2.3` |
| `working-directory` |          | `.`     | Directory with `package.json`                | `./frontend`         |

</details>

---

#### `js/playwright`

Install Playwright browsers with caching. Run tests separately (e.g. `pnpm test:e2e`).

> ⚠️ Requires Node.js and deps installed first. Use [`github/runtimes`](#githubruntimes) + [`js/install`](#jsinstall) before this action.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/js/playwright@v1
  with:
    browsers: "chromium firefox webkit"
- run: pnpm test:e2e
```

<details>
<summary>Inputs & Outputs</summary>

| Input               | Required | Default | Description                       | Example                                   |
| ------------------- | :------: | ------- | --------------------------------- | ----------------------------------------- |
| `working-directory` |          | `.`     | Directory with Playwright config  | `./e2e`                                   |
| `browsers`          |          | `""`    | Browsers to install (empty = all) | `"chromium"`, `"chromium firefox webkit"` |

| Output               | Description                        | Example  |
| -------------------- | ---------------------------------- | -------- |
| `playwright-version` | Installed Playwright version       | `1.42.0` |
| `cache-hit`          | Whether browser cache was restored | `true`   |

</details>

---

### DotNet

#### `dotnet/install`

Restore NuGet packages with caching. Caches `~/.nuget/packages` keyed by `.csproj` and `nuget.config` hashes.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    dotnet-version: "8.0.x"
- uses: Utdanningsdirektoratet/dit-github-actions/dotnet/install@v1
```

<details>
<summary>Inputs & Outputs</summary>

| Input               | Required | Default | Description                     | Example        |
| ------------------- | :------: | ------- | ------------------------------- | -------------- |
| `working-directory` |          | `.`     | Directory with solution/project | `./src`        |
| `nuget-config`      |          | `""`    | Path to `nuget.config`          | `nuget.config` |

| Output      | Description                      | Example |
| ----------- | -------------------------------- | ------- |
| `cache-hit` | Whether NuGet cache was restored | `true`  |

</details>

---

#### `dotnet/build`

Publish .NET project and create an Optimizely DXP deployment package (`.nupkg`).

Package naming: `{app-name}.cms.app.{YYYYMMDD}.{run_number}.nupkg`

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/dotnet/build@v1
  id: package
  with:
    app-name: myapp
```

<details>
<summary>Inputs & Outputs</summary>

| Input           | Required | Default   | Description                                        | Example            |
| --------------- | :------: | --------- | -------------------------------------------------- | ------------------ |
| `app-name`      |   yes    |           | Application name for package                       | `myapp`            |
| `configuration` |          | `Release` | Build configuration                                | `Release`, `Debug` |
| `version`       |          | auto      | Package version (default: `YYYYMMDD.{run_number}`) | `1.0.0`            |

| Output         | Description           | Example                                |
| -------------- | --------------------- | -------------------------------------- |
| `package-name` | Generated file name   | `myapp.cms.app.20260222.42.nupkg`      |
| `package-path` | Full path to the file | `/tmp/myapp.cms.app.20260222.42.nupkg` |

</details>

---

### Docker

#### `docker/build`

Build and push a Docker image with registry-based layer caching.

> 💡 Use `push-by-digest: "true"` when the same build should be tagged differently per environment, then follow up with [`docker/tag`](#dockertag).

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/docker/build@v1
  id: build
  with:
    registry: registry.example.com
    username: ${{ secrets.REGISTRY_USERNAME }}
    password: ${{ secrets.REGISTRY_PASSWORD }}
    image: myapp
    push-by-digest: "true"
```

<details>
<summary>Inputs & Outputs</summary>

| Input            | Required | Default       | Description                                            | Example                   |
| ---------------- | :------: | ------------- | ------------------------------------------------------ | ------------------------- |
| `registry`       |   yes    |               | Registry host                                          | `registry.example.com`    |
| `username`       |   yes    |               | Registry username                                      | `${{ secrets.REG_USER }}` |
| `password`       |   yes    |               | Registry password/token                                | `${{ secrets.REG_PASS }}` |
| `image`          |   yes    |               | Image name (no registry prefix)                        | `myapp`                   |
| `tag`            |          | `""`          | Version tag                                            | `v1.2.3`                  |
| `tag-latest`     |          | `latest`      | Rolling environment tag                                | `latest-production`       |
| `push-by-digest` |          | `false`       | Push by digest only (no tag)                           | `"true"`                  |
| `context`        |          | `.`           | Build context path                                     | `./docker`                |
| `dockerfile`     |          | auto          | Dockerfile path (default: `Dockerfile` in context dir) | `./Dockerfile.prod`       |
| `platforms`      |          | `linux/amd64` | Target platforms                                       | `linux/amd64,linux/arm64` |
| `build-args`     |          | `""`          | Build arguments (multiline `KEY=VALUE`)                | `NODE_ENV=production`     |

| Output      | Description          | Example                             |
| ----------- | -------------------- | ----------------------------------- |
| `digest`    | Image digest         | `sha256:abc123...`                  |
| `image-ref` | Full image reference | `registry.example.com/myapp:v1.2.3` |

</details>

---

#### `docker/tag`

Tag an existing image by digest. Creates version tag, short SHA tag, and rolling latest in parallel.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/docker/tag@v1
  id: tag
  with:
    registry: registry.example.com
    username: ${{ secrets.REGISTRY_USERNAME }}
    password: ${{ secrets.REGISTRY_PASSWORD }}
    image: myapp
    digest: ${{ needs.build.outputs.digest }}
```

<details>
<summary>Inputs & Outputs</summary>

| Input        | Required | Default  | Description                               | Example                   |
| ------------ | :------: | -------- | ----------------------------------------- | ------------------------- |
| `registry`   |   yes    |          | Registry host                             | `registry.example.com`    |
| `username`   |   yes    |          | Registry username                         | `${{ secrets.REG_USER }}` |
| `password`   |   yes    |          | Registry password                         | `${{ secrets.REG_PASS }}` |
| `image`      |   yes    |          | Image name                                | `myapp`                   |
| `digest`     |   yes    |          | Digest to tag                             | `sha256:abc123...`        |
| `tag`        |          | auto     | Version tag (auto: `YYMMDD-HHMMSS_<sha>`) | `v1.2.3`                  |
| `tag-latest` |          | `latest` | Rolling environment tag                   | `latest-production`       |

| Output      | Description          | Example                                            |
| ----------- | -------------------- | -------------------------------------------------- |
| `tag`       | Applied version tag  | `260222-143000_abc1234`                            |
| `image-ref` | Full image reference | `registry.example.com/myapp:260222-143000_abc1234` |

</details>

---

### Kubernetes

#### `kubernetes/deploy`

Set image on a Kubernetes deployment. Kubeconfig uses `mktemp` with `if: always()` cleanup.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/deploy@v1
  with:
    kube-config: ${{ secrets.KUBECONFIG }}
    namespace: production
    deployment: myapp
    image: registry.example.com/myapp:v1.2.3
```

<details>
<summary>Inputs & Outputs</summary>

| Input         | Required | Default | Description               | Example                             |
| ------------- | :------: | ------- | ------------------------- | ----------------------------------- |
| `kube-config` |   yes    |         | Base64-encoded kubeconfig | `${{ secrets.KUBECONFIG }}`         |
| `namespace`   |   yes    |         | Target namespace          | `production`                        |
| `deployment`  |   yes    |         | Deployment name           | `myapp`                             |
| `image`       |   yes    |         | Full image reference      | `registry.example.com/myapp:v1.2.3` |

| Output   | Description       | Example             |
| -------- | ----------------- | ------------------- |
| `status` | Deployment result | `success`, `failed` |

</details>

---

#### `kubernetes/rollout`

Wait for rollout with optional auto-rollback on failure.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/rollout@v1
  with:
    kube-config: ${{ secrets.KUBECONFIG }}
    namespace: production
    deployment: myapp
```

<details>
<summary>Inputs & Outputs</summary>

| Input                 | Required | Default | Description               | Example                     |
| --------------------- | :------: | ------- | ------------------------- | --------------------------- |
| `kube-config`         |   yes    |         | Base64-encoded kubeconfig | `${{ secrets.KUBECONFIG }}` |
| `namespace`           |   yes    |         | Target namespace          | `production`                |
| `deployment`          |   yes    |         | Deployment name           | `myapp`                     |
| `timeout`             |          | `300`   | Timeout in seconds        | `600`                       |
| `rollback-on-failure` |          | `true`  | Auto-rollback on failure  | `true`                      |

| Output   | Description    | Example                            |
| -------- | -------------- | ---------------------------------- |
| `status` | Rollout result | `success`, `rolled-back`, `failed` |

</details>

---

### DXP

#### `dxp/upload`

Upload a deployment package to Optimizely DXP blob storage.

> ❗ Credentials are masked with `::add-mask::` and passed via `env:` blocks — never inline.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/dxp/upload@v1
  with:
    package-path: ${{ steps.package.outputs.package-path }}
    project-id: ${{ secrets.DXP_PROJECT_ID }}
    client-key: ${{ secrets.DXP_CLIENT_KEY }}
    client-secret: ${{ secrets.DXP_CLIENT_SECRET }}
```

<details>
<summary>Inputs & Outputs</summary>

| Input           | Required | Default | Description              | Example                                     |
| --------------- | :------: | ------- | ------------------------ | ------------------------------------------- |
| `package-path`  |   yes    |         | Path to `.nupkg` package | `${{ steps.package.outputs.package-path }}` |
| `project-id`    |   yes    |         | DXP Project ID           | `${{ secrets.DXP_PROJECT_ID }}`             |
| `client-key`    |   yes    |         | DXP API Client Key       | `${{ secrets.DXP_CLIENT_KEY }}`             |
| `client-secret` |   yes    |         | DXP API Client Secret    | `${{ secrets.DXP_CLIENT_SECRET }}`          |

| Output         | Description                | Example                           |
| -------------- | -------------------------- | --------------------------------- |
| `package-name` | Uploaded package file name | `myapp.cms.app.20260222.42.nupkg` |

</details>

---

#### `dxp/deploy`

Deploy, complete, or reset an Optimizely DXP deployment. Automatically resets stuck deployments before starting a new one.

> 🔴 `package-name` is required when `action` is `deploy`.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/dxp/deploy@v1
  with:
    action: deploy
    target-environment: Integration
    package-name: ${{ steps.package.outputs.package-name }}
    project-id: ${{ secrets.DXP_PROJECT_ID }}
    client-key: ${{ secrets.DXP_CLIENT_KEY }}
    client-secret: ${{ secrets.DXP_CLIENT_SECRET }}
```

<details>
<summary>Inputs & Outputs</summary>

| Input                | Required | Default | Description                          | Example                                      |
| -------------------- | :------: | ------- | ------------------------------------ | -------------------------------------------- |
| `action`             |   yes    |         | Action to perform                    | `deploy`, `complete`, `reset`                |
| `target-environment` |   yes    |         | Target DXP environment               | `Integration`, `Preproduction`, `Production` |
| `project-id`         |   yes    |         | DXP Project ID                       | `${{ secrets.DXP_PROJECT_ID }}`              |
| `client-key`         |   yes    |         | DXP API Client Key                   | `${{ secrets.DXP_CLIENT_KEY }}`              |
| `client-secret`      |   yes    |         | DXP API Client Secret                | `${{ secrets.DXP_CLIENT_SECRET }}`           |
| `package-name`       |          | `""`    | Package name (required for `deploy`) | `myapp.cms.app.20260222.42.nupkg`            |
| `direct-deploy`      |          | `false` | Skip slot verification (Integration) | `"true"`                                     |

| Output          | Description       | Example                                                               |
| --------------- | ----------------- | --------------------------------------------------------------------- |
| `deployment-id` | DXP deployment ID | `d-abc123`                                                            |
| `status`        | Deployment status | `InProgress`, `AwaitingVerification`, `completed`, `reset`, `skipped` |

</details>

---

## Security

<details>
<summary>Credential handling</summary>

- **Token masking** — `::add-mask::` before any use
- **Credential cleanup** — `if: always()` steps remove git headers
- **Temp files** — Kubeconfigs use `mktemp` + `trap`-based cleanup
- **No inline secrets** — All passed via `env:` blocks

</details>

<details>
<summary>Secrets vs vars</summary>

> ⚠️ **GitHub Actions `vars` are visible in workflow logs and to anyone with read access to the repository.** On public repositories, this means the entire internet. Never store sensitive values (tokens, passwords, connection strings) in `vars` — always use `secrets`.

Safe for `vars`: registry hostnames, image names, namespaces, deployment names, non-sensitive usernames, environment URLs.

Must use `secrets`: passwords, API keys, tokens, kubeconfigs, private keys, client secrets.

</details>

<details>
<summary>GitHub App vs GITHUB_TOKEN</summary>

Use a GitHub App when pushes need to **trigger workflows**, PRs need to **merge through branch protection**, or you need a **distinct identity** for automated commits. `GITHUB_TOKEN` suffices for everything else.

All `github/*` actions auto-detect auth method: provide `app-id` + `app-private-key` for GitHub App, or `github-token` for default token.

</details>

## Versioning

| Ref       | Use                             |
| --------- | ------------------------------- |
| `@v1`     | **Recommended** — latest stable |
| `@v1.2.3` | Pinned for reproducibility      |
| `@main`   | Latest commit                   |

<details>
<summary>Release process</summary>

**Tag specific release:**

```bash
git tag v1.0.1 && git push origin v1.0.1
```

**Update rolling v1 tag:**

```bash
git tag -f v1 && git push -f origin v1
```

> ℹ️ `v1` is a moving target (latest stable). Specific versions (`v1.0.1`) are immutable.

</details>
