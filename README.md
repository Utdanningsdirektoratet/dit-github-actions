# DIT GitHub Actions

Shared composite actions for CI/CD at [Utdanningsdirektoratet](https://github.com/Utdanningsdirektoratet) DIT.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/{action}@v1
```

**Git:** [`git/checkout`](#gitcheckout), [`git/commit`](#gitcommit)

**GitHub:** [`github/runtimes`](#githubruntimes), [`github/clone`](#githubclone), [`github/push`](#githubpush), [`github/pr`](#githubpr), [`github/pr-merge`](#githubpr-merge)

**GoUpdate:** [`goupdate/install`](#goupdateinstall), [`goupdate/scan`](#goupdatescan), [`goupdate/outdated`](#goupdateoutdated), [`goupdate/update`](#goupdateupdate)

**JavaScript:** [`js/install`](#jsinstall), [`js/lint`](#jslint), [`js/playwright`](#jsplaywright)

**DotNet:** [`dotnet/install`](#dotnetinstall), [`dotnet/build`](#dotnetbuild)

**Docker:** [`docker/build`](#dockerbuild), [`docker/tag`](#dockertag)

**Kubernetes:** [`kubernetes/deploy`](#kubernetesdeploy), [`kubernetes/rollout`](#kubernetesrollout)

**DXP:** [`dxp/upload`](#dxpupload), [`dxp/deploy`](#dxpdeploy)

## Examples

<details>
<summary><b>Build and lint a Node.js project</b></summary>

```yaml
name: CI
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Utdanningsdirektoratet/dit-github-actions/js/lint@v1
        with:
          node-version: "22"
          lint-command: "lint"
```

</details>

<details>
<summary><b>E2E tests with Playwright</b></summary>

```yaml
name: E2E Tests
on: [pull_request]
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          node-version: "22"
          node-pms: "pnpm"
      - uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
      - uses: Utdanningsdirektoratet/dit-github-actions/js/playwright@v1
        with:
          browsers: "chromium firefox webkit"
      - run: pnpm test:e2e
```

</details>

<details>
<summary><b>Docker build, tag, and deploy to Kubernetes</b></summary>

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        uses: Utdanningsdirektoratet/dit-github-actions/docker/build@v1
        with:
          registry: registry.example.com
          username: ${{ secrets.REG_USER }}
          password: ${{ secrets.REG_PASS }}
          image: myapp
          push-by-digest: "true"

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - id: tag
        uses: Utdanningsdirektoratet/dit-github-actions/docker/tag@v1
        with:
          registry: registry.example.com
          username: ${{ secrets.REG_USER }}
          password: ${{ secrets.REG_PASS }}
          image: myapp
          digest: ${{ needs.build.outputs.digest }}
          tag-latest: latest-production

      - uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/deploy@v1
        with:
          kube-config: ${{ secrets.KUBECONFIG }}
          namespace: production
          deployment: myapp
          image: ${{ steps.tag.outputs.image-ref }}

      - uses: Utdanningsdirektoratet/dit-github-actions/kubernetes/rollout@v1
        with:
          kube-config: ${{ secrets.KUBECONFIG }}
          namespace: production
          deployment: myapp
```

</details>

<details>
<summary><b>Automated dependency updates with GoUpdate</b></summary>

```yaml
name: Dependency Updates
on:
  schedule:
    - cron: "0 6 * * 1"

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: Utdanningsdirektoratet/dit-github-actions/goupdate/install@v1

      - uses: Utdanningsdirektoratet/dit-github-actions/git/checkout@v1
        id: branch
        with:
          branch: goupdate/auto-update-minor
          source-branch: main

      - id: update
        uses: Utdanningsdirektoratet/dit-github-actions/goupdate/update@v1
        with:
          update-type: minor

      - if: steps.update.outputs.has-changes == 'true'
        uses: Utdanningsdirektoratet/dit-github-actions/git/commit@v1
        with:
          message: "GoUpdate: minor update ({date})"

      - if: steps.update.outputs.has-changes == 'true'
        uses: Utdanningsdirektoratet/dit-github-actions/github/push@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          app-private-key: ${{ secrets.APP_KEY }}
          branch: ${{ steps.branch.outputs.branch-name }}

      - if: steps.update.outputs.has-changes == 'true'
        id: pr
        uses: Utdanningsdirektoratet/dit-github-actions/github/pr@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          app-private-key: ${{ secrets.APP_KEY }}
          title: "GoUpdate: minor update ({date})"
          base: main
          head: ${{ steps.branch.outputs.branch-name }}

      - if: steps.update.outputs.has-changes == 'true'
        uses: Utdanningsdirektoratet/dit-github-actions/github/pr-merge@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          app-private-key: ${{ secrets.APP_KEY }}
          pr-number: ${{ steps.pr.outputs.pr-number }}
          wait-for-checks: "600"
```

</details>

<details>
<summary><b>DXP deploy</b></summary>

```yaml
name: DXP Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      package-name: ${{ steps.package.outputs.package-name }}
    steps:
      - uses: actions/checkout@v4
      - uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
        with:
          dotnet-version: '8.0.x'

      - uses: Utdanningsdirektoratet/dit-github-actions/dotnet/install@v1

      - id: package
        uses: Utdanningsdirektoratet/dit-github-actions/dotnet/build@v1
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
          direct-deploy: 'true'
```

</details>

<details>
<summary><b>Clone another repo and use its contents</b></summary>

```yaml
name: Sync Config
on: [workflow_dispatch]

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: Utdanningsdirektoratet/dit-github-actions/github/clone@v1
        with:
          repository: Utdanningsdirektoratet/shared-config
          path: ./config
          ref: main
          app-id: ${{ secrets.APP_ID }}
          app-private-key: ${{ secrets.APP_KEY }}
      - run: cat ./config/settings.json
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
    branch: "updates/{date}"        # resolves → updates/2026-02-22
    source-branch: main
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `branch` | yes | | Branch name (`{date}` template supported) | `goupdate/auto-update-minor` |
| `source-branch` | yes | | Source branch to create from or ff-merge | `main` |
| `auto-ff` | | `true` | Fast-forward merge from source on existing branches | `true` |
| `create-branch` | | `true` | Create branch from source if missing | `true` |

| Output | Description | Example |
|--------|-------------|---------|
| `branch-name` | Resolved branch name | `updates/2026-02-22` |
| `branch-exists` | Whether branch already existed | `true` |
| `diverged` | Whether branch diverged from source | `false` |

---

#### `git/commit`

Stage and commit all changes.

**Template variables** — `message` supports: `{date}` (YYYY-MM-DD), `{branch}` (current branch).

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/git/commit@v1
  with:
    message: "GoUpdate: minor update ({date})"  # resolves → GoUpdate: minor update (2026-02-22)
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `message` | | `GoUpdate: Update` | Commit message (`{date}`, `{branch}` templates) | `"chore: update deps ({date})"` |
| `git-user-name` | | `github-actions[bot]` | Commit author name | `my-bot` |
| `git-user-email` | | `github-actions[bot]@users.noreply.github.com` | Commit author email | `bot@example.com` |

| Output | Description | Example |
|--------|-------------|---------|
| `has-changes` | Whether any changes were committed | `true` |

---

### GitHub

#### `github/runtimes`

Setup language runtimes. All inputs opt-in — only specified runtimes are installed.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    node-version: "22"
    node-pms: "pnpm"
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `node-version` | | `""` | Node.js version | `"22"` |
| `node-pms` | | `""` | Package managers (comma-separated) | `"pnpm"`, `"npm,yarn"` |
| `php-version` | | `""` | PHP version | `"8.3"` |
| `python-version` | | `""` | Python version | `"3.12"` |
| `go-version` | | `""` | Go version | `"1.22"` |
| `dotnet-version` | | `""` | .NET version | `"8.0.x"` |

---

#### `github/clone`

Clone a repository using GitHub App token or fallback token.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/clone@v1
  with:
    repository: Utdanningsdirektoratet/shared-config
    path: ./config
    ref: main
    app-id: ${{ secrets.APP_ID }}
    app-private-key: ${{ secrets.APP_KEY }}
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `repository` | yes | | Repository to clone (`owner/repo`) | `Utdanningsdirektoratet/my-repo` |
| `path` | yes | | Directory to clone into | `./config` |
| `ref` | | `""` | Branch, tag, or SHA to checkout | `main`, `v1.0.0` |
| `depth` | | `1` | Clone depth (`0` = full history) | `1` |
| `app-id` | | `""` | GitHub App ID | `${{ secrets.APP_ID }}` |
| `app-private-key` | | `""` | GitHub App private key | `${{ secrets.APP_KEY }}` |
| `github-token` | | `""` | Fallback token (if no App credentials) | `${{ secrets.GH_TOKEN }}` |

| Output | Description | Example |
|--------|-------------|---------|
| `sha` | SHA of the cloned commit | `abc1234def5678` |

> Provide `app-id` + `app-private-key` (GitHub App), `github-token`, or neither (public repos only).

---

#### `github/push`

Push commits with optional GitHub App token to trigger downstream workflows.

> [!IMPORTANT]
> Credentials are cleaned up in an `if: always()` step, even on failure.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/push@v1
  with:
    app-id: ${{ secrets.APP_ID }}
    app-private-key: ${{ secrets.APP_KEY }}
    branch: feature/my-branch
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `app-id` | | `""` | GitHub App ID (triggers workflows) | `${{ secrets.APP_ID }}` |
| `app-private-key` | | `""` | GitHub App private key | `${{ secrets.APP_KEY }}` |
| `github-token` | | `""` | Fallback token (no workflow trigger) | `${{ secrets.GH_TOKEN }}` |
| `branch` | yes | | Branch to push | `goupdate/auto-update-minor` |

| Output | Description | Example |
|--------|-------------|---------|
| `pushed` | Whether changes were pushed | `true` |
| `sha` | SHA of the pushed commit | `abc1234def5678` |

> Use `app-id` + `app-private-key` to trigger workflows. `github-token` pushes silently.

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

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `app-id` | | `""` | GitHub App ID | `${{ secrets.APP_ID }}` |
| `app-private-key` | | `""` | GitHub App private key | `${{ secrets.APP_KEY }}` |
| `github-token` | | `""` | Fallback token | `${{ secrets.GH_TOKEN }}` |
| `title` | yes | | PR title (`{date}`, `{base}`, `{head}` templates) | `"Update deps ({date})"` |
| `body` | | `""` | PR body (`{date}`, `{base}`, `{head}` templates) | `"Merging {head} into {base}"` |
| `base` | yes | | Base branch | `main` |
| `head` | yes | | Head branch | `goupdate/auto-update-minor` |

| Output | Description | Example |
|--------|-------------|---------|
| `pr-number` | PR number | `42` |
| `pr-url` | PR URL | `https://github.com/org/repo/pull/42` |
| `created` | Whether a new PR was created | `true` |

---

#### `github/pr-merge`

Wait for checks and merge a pull request. Polls until checks pass (or timeout), then merges and deletes the branch.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/pr-merge@v1
  with:
    app-id: ${{ secrets.APP_ID }}
    app-private-key: ${{ secrets.APP_KEY }}
    pr-number: ${{ steps.pr.outputs.pr-number }}
    wait-for-checks: "600"
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `app-id` | | `""` | GitHub App ID | `${{ secrets.APP_ID }}` |
| `app-private-key` | | `""` | GitHub App private key | `${{ secrets.APP_KEY }}` |
| `github-token` | | `""` | Fallback token | `${{ secrets.GH_TOKEN }}` |
| `pr-number` | yes | | PR number to merge | `42` |
| `merge-method` | | `squash` | Merge strategy | `squash`, `merge`, `rebase` |
| `wait-for-checks` | | `300` | Timeout in seconds (`0` = skip) | `600` |
| `check-interval` | | `15` | Poll interval in seconds | `15` |
| `notify` | | `""` | Users/teams to ping on failure | `@myteam` |
| `skip-merge-comment` | | `""` | If set, skip merge and post this as a PR comment | `"Skipping auto-merge: manual review required"` |

| Output | Description | Example |
|--------|-------------|---------|
| `merged` | Whether PR was merged | `true` |
| `checks-passed` | Whether checks passed | `true` |

---

### GoUpdate

#### `goupdate/install`

Install [goupdate](https://github.com/ajxudir/goupdate) binary from GitHub releases.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/goupdate/install@v1
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `version` | | `latest` | Version or tag | `v1.2.0` |
| `repo` | | `ajxudir/goupdate` | GitHub repo | `ajxudir/goupdate` |
| `github-token` | | `""` | Avoids API rate limits | `${{ secrets.GITHUB_TOKEN }}` |

| Output | Description | Example |
|--------|-------------|---------|
| `version` | Installed version | `1.2.0` |

---

#### `goupdate/scan`

Detect package managers and lock files in the repository.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/goupdate/scan@v1
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `working-directory` | | `.` | Directory to scan | `./packages/app` |

| Output | Description | Example |
|--------|-------------|---------|
| `package-managers` | Detected package managers (comma-separated) | `npm,pip` |
| `rules` | Detected rules (comma-separated) | `package-lock.json,requirements.txt` |

---

#### `goupdate/outdated`

Check for outdated dependencies with major/minor/patch breakdown.

```yaml
- id: outdated
  uses: Utdanningsdirektoratet/dit-github-actions/goupdate/outdated@v1
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `working-directory` | | `.` | Directory to check | `.` |

| Output | Description | Example |
|--------|-------------|---------|
| `total` | Total dependencies | `120` |
| `total-outdated` | Total outdated | `8` |
| `major` | Major updates available | `2` |
| `minor` | Minor updates available | `3` |
| `patch` | Patch updates available | `3` |

---

#### `goupdate/update`

Apply dependency updates. Does **not** commit or push — use [`git/commit`](#gitcommit) and [`github/push`](#githubpush) afterward.

```yaml
- id: update
  uses: Utdanningsdirektoratet/dit-github-actions/goupdate/update@v1
  with:
    update-type: minor
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `update-type` | | `minor` | Update level | `major`, `minor`, `patch` |

| Output | Description | Example |
|--------|-------------|---------|
| `has-changes` | Whether any files changed | `true` |
| `partial-failure` | Some updates failed but others succeeded | `false` |

> Uses `--continue-on-fail`: if some updates fail but others succeed, `partial-failure=true` lets the workflow create a PR without auto-merging.

---

### JavaScript

#### `js/install`

Install and cache Node.js dependencies. Auto-detects package manager from lock files: `pnpm-lock.yaml` > `yarn.lock` > `package-lock.json`.

> [!WARNING]
> Does **not** install Node.js. Use [`github/runtimes`](#githubruntimes) first.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    node-version: "22"
    node-pms: "pnpm"
- uses: Utdanningsdirektoratet/dit-github-actions/js/install@v1
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `working-directory` | | `.` | Directory with `package.json` | `./frontend` |
| `package-manager` | | auto | Force specific PM | `pnpm`, `yarn`, `npm` |

| Output | Description | Example |
|--------|-------------|---------|
| `package-manager` | Detected/forced package manager | `pnpm` |
| `cache-hit` | Whether cache was restored | `true` |

---

#### `js/lint`

Run lint checks. Sets up Node.js, installs dependencies, and runs the lint command automatically.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/js/lint@v1
  with:
    node-version: "22"
    lint-command: "lint"
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `node-version` | | `22` | Node.js version | `"20"` |
| `working-directory` | | `.` | Directory to run lint in | `./frontend` |
| `lint-command` | | `lint` | npm script to run | `lint`, `lint:ci` |

| Output | Description | Example |
|--------|-------------|---------|
| `package-manager` | Detected package manager | `pnpm` |

---

#### `js/playwright`

Install Playwright browsers with caching. Run tests separately (e.g. `pnpm test:e2e`).

> [!WARNING]
> Requires Node.js and deps installed first. Use [`github/runtimes`](#githubruntimes) + [`js/install`](#jsinstall) before this action.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/js/playwright@v1
  with:
    browsers: "chromium firefox webkit"
- run: pnpm test:e2e
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `working-directory` | | `.` | Directory with Playwright config | `./e2e` |
| `browsers` | | `""` | Browsers to install (empty = all) | `"chromium"`, `"chromium firefox webkit"` |

| Output | Description | Example |
|--------|-------------|---------|
| `playwright-version` | Installed Playwright version | `1.42.0` |
| `cache-hit` | Whether browser cache was restored | `true` |

---

### DotNet

#### `dotnet/install`

Restore NuGet packages with caching. Caches `~/.nuget/packages` keyed by `.csproj` and `nuget.config` hashes.

> [!WARNING]
> Does **not** install .NET SDK. Use [`github/runtimes`](#githubruntimes) first.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/github/runtimes@v1
  with:
    dotnet-version: "8.0.x"
- uses: Utdanningsdirektoratet/dit-github-actions/dotnet/install@v1
  with:
    nuget-config: nuget.config
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `working-directory` | | `.` | Directory with solution/project | `./src` |
| `nuget-config` | | `""` | Path to `nuget.config` | `nuget.config` |

| Output | Description | Example |
|--------|-------------|---------|
| `cache-hit` | Whether NuGet cache was restored | `true` |

---

#### `dotnet/build`

Publish .NET project and create an Optimizely DXP deployment package (`.nupkg`).

Package naming: `{app-name}.cms.app.{YYYYMMDD}.{run_number}.nupkg`

```yaml
- id: package
  uses: Utdanningsdirektoratet/dit-github-actions/dotnet/build@v1
  with:
    app-name: myapp
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `app-name` | yes | | Application name for package | `myapp` |
| `configuration` | | `Release` | Build configuration | `Release`, `Debug` |
| `version` | | auto | Package version (default: `YYYYMMDD.{run_number}`) | `1.0.0` |

| Output | Description | Example |
|--------|-------------|---------|
| `package-name` | Generated file name | `myapp.cms.app.20260222.42.nupkg` |
| `package-path` | Full path to the file | `/tmp/myapp.cms.app.20260222.42.nupkg` |

---

### Docker

#### `docker/build`

Build and push a Docker image with registry-based layer caching.

> [!TIP]
> Use `push-by-digest: "true"` when the same build should be tagged differently per environment, then follow up with [`docker/tag`](#dockertag).

```yaml
- id: build
  uses: Utdanningsdirektoratet/dit-github-actions/docker/build@v1
  with:
    registry: registry.example.com
    username: ${{ secrets.REG_USER }}
    password: ${{ secrets.REG_PASS }}
    image: myapp
    push-by-digest: "true"
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `registry` | yes | | Registry host | `registry.example.com` |
| `username` | yes | | Registry username | `${{ secrets.REG_USER }}` |
| `password` | yes | | Registry password/token | `${{ secrets.REG_PASS }}` |
| `image` | yes | | Image name (no registry prefix) | `myapp` |
| `tag` | | `""` | Version tag | `v1.2.3` |
| `tag-latest` | | `latest` | Rolling environment tag | `latest-production` |
| `push-by-digest` | | `false` | Push by digest only (no tag) | `"true"` |
| `context` | | `.` | Build context path | `./docker` |
| `dockerfile` | | auto | Dockerfile path (default: `Dockerfile` in context dir) | `./Dockerfile.prod` |
| `platforms` | | `linux/amd64` | Target platforms | `linux/amd64,linux/arm64` |
| `build-args` | | `""` | Build arguments (multiline `KEY=VALUE`) | `NODE_ENV=production` |

| Output | Description | Example |
|--------|-------------|---------|
| `digest` | Image digest | `sha256:abc123...` |
| `image-ref` | Full image reference | `registry.example.com/myapp:v1.2.3` |

---

#### `docker/tag`

Tag an existing image by digest. Creates version tag, short SHA tag, and rolling latest in parallel.

```yaml
- id: tag
  uses: Utdanningsdirektoratet/dit-github-actions/docker/tag@v1
  with:
    registry: registry.example.com
    username: ${{ secrets.REG_USER }}
    password: ${{ secrets.REG_PASS }}
    image: myapp
    digest: ${{ needs.build.outputs.digest }}
    tag-latest: latest-production
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `registry` | yes | | Registry host | `registry.example.com` |
| `username` | yes | | Registry username | `${{ secrets.REG_USER }}` |
| `password` | yes | | Registry password | `${{ secrets.REG_PASS }}` |
| `image` | yes | | Image name | `myapp` |
| `digest` | yes | | Digest to tag | `sha256:abc123...` |
| `tag` | | auto | Version tag (auto: `YYMMDD-HHMMSS_<sha>`) | `v1.2.3` |
| `tag-latest` | | `latest` | Rolling environment tag | `latest-production` |

| Output | Description | Example |
|--------|-------------|---------|
| `tag` | Applied version tag | `260222-143000_abc1234` |
| `image-ref` | Full image reference | `registry.example.com/myapp:260222-143000_abc1234` |

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

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `kube-config` | yes | | Base64-encoded kubeconfig | `${{ secrets.KUBECONFIG }}` |
| `namespace` | yes | | Target namespace | `production` |
| `deployment` | yes | | Deployment name | `myapp` |
| `image` | yes | | Full image reference | `registry.example.com/myapp:v1.2.3` |

| Output | Description | Example |
|--------|-------------|---------|
| `status` | Deployment result | `success`, `failed` |

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

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `kube-config` | yes | | Base64-encoded kubeconfig | `${{ secrets.KUBECONFIG }}` |
| `namespace` | yes | | Target namespace | `production` |
| `deployment` | yes | | Deployment name | `myapp` |
| `timeout` | | `300` | Timeout in seconds | `600` |
| `rollback-on-failure` | | `true` | Auto-rollback on failure | `true` |

| Output | Description | Example |
|--------|-------------|---------|
| `status` | Rollout result | `success`, `rolled-back`, `failed` |

---

### DXP

#### `dxp/upload`

Upload a deployment package to Optimizely DXP blob storage.

> [!IMPORTANT]
> Credentials are masked with `::add-mask::` and passed via `env:` blocks — never inline.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/dxp/upload@v1
  with:
    package-path: ${{ steps.package.outputs.package-path }}
    project-id: ${{ secrets.DXP_PROJECT_ID }}
    client-key: ${{ secrets.DXP_CLIENT_KEY }}
    client-secret: ${{ secrets.DXP_CLIENT_SECRET }}
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `package-path` | yes | | Path to `.nupkg` package | `${{ steps.package.outputs.package-path }}` |
| `project-id` | yes | | DXP Project ID | `${{ secrets.DXP_PROJECT_ID }}` |
| `client-key` | yes | | DXP API Client Key | `${{ secrets.DXP_CLIENT_KEY }}` |
| `client-secret` | yes | | DXP API Client Secret | `${{ secrets.DXP_CLIENT_SECRET }}` |

| Output | Description | Example |
|--------|-------------|---------|
| `package-name` | Uploaded package file name | `myapp.cms.app.20260222.42.nupkg` |

---

#### `dxp/deploy`

Deploy, complete, or reset an Optimizely DXP deployment. Automatically resets stuck deployments before starting a new one.

> [!CAUTION]
> `package-name` is required when `action` is `deploy`.

```yaml
- uses: Utdanningsdirektoratet/dit-github-actions/dxp/deploy@v1
  with:
    action: deploy
    target-environment: Integration
    package-name: ${{ steps.package.outputs.package-name }}
    project-id: ${{ secrets.DXP_PROJECT_ID }}
    client-key: ${{ secrets.DXP_CLIENT_KEY }}
    client-secret: ${{ secrets.DXP_CLIENT_SECRET }}
    direct-deploy: 'true'
```

| Input | Required | Default | Description | Example |
|-------|:--------:|---------|-------------|---------|
| `action` | yes | | Action to perform | `deploy`, `complete`, `reset` |
| `target-environment` | yes | | Target DXP environment | `Integration`, `Preproduction`, `Production` |
| `project-id` | yes | | DXP Project ID | `${{ secrets.DXP_PROJECT_ID }}` |
| `client-key` | yes | | DXP API Client Key | `${{ secrets.DXP_CLIENT_KEY }}` |
| `client-secret` | yes | | DXP API Client Secret | `${{ secrets.DXP_CLIENT_SECRET }}` |
| `package-name` | | `""` | Package name (required for `deploy`) | `myapp.cms.app.20260222.42.nupkg` |
| `direct-deploy` | | `false` | Skip slot verification (Integration) | `"true"` |

| Output | Description | Example |
|--------|-------------|---------|
| `deployment-id` | DXP deployment ID | `d-abc123` |
| `status` | Deployment status | `InProgress`, `AwaitingVerification`, `completed`, `reset`, `skipped` |

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
<summary>GitHub App vs GITHUB_TOKEN</summary>

Use a GitHub App when pushes need to **trigger workflows**, PRs need to **merge through branch protection**, or you need a **distinct identity** for automated commits. `GITHUB_TOKEN` suffices for everything else.

All `github/*` actions auto-detect auth method: provide `app-id` + `app-private-key` for GitHub App, or `github-token` for default token.

</details>

## Versioning

| Ref | Use |
|-----|-----|
| `@v1` | **Recommended** — latest stable |
| `@v1.2.3` | Pinned for reproducibility |
| `@main` | Latest commit |

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

> [!NOTE]
> `v1` is a moving target (latest stable). Specific versions (`v1.0.1`) are immutable.

</details>
