# Reusable WordPress Workflows

Shared GitHub Actions workflows for Dev Hero's WordPress themes and plugins:

- [WordPress Deploy](#deploy-features) lints and deploys themes, plugins, and mu-plugins to managed hosting.
- [PR Build Check](#pr-build-check) builds front-end assets on every pull request.
- [Renovate → ClickUp](#renovate--clickup) mirrors Renovate pull requests into ClickUp tasks.

| Workflow | Called from other projects | Purpose |
|----------|---------------------------|---------|
| `wordpress-deploy.yml` | Yes | Lint and deploy a theme or plugin |
| `pr-build.yml` | Yes | Build front-end assets on pull requests |
| `renovate-clickup.yml` | Yes | Sync Renovate pull requests to ClickUp |
| `release.yml` | No | Runs in this repository to cut releases. See [Releasing](#releasing) |

Each shared workflow has the same filename as the caller file a project adds, so a project's `.github/workflows/pr-build.yml` calls this repository's `pr-build.yml`.

## Deploy Features

Reusable workflow: [`.github/workflows/wordpress-deploy.yml`](.github/workflows/wordpress-deploy.yml).

- **Multi-host support**: Kinsta, WP Engine, Pressable, and generic WordPress hosts
- **Project type detection**: Automatically deploys to correct directory (themes/plugins/mu-plugins)
- **PHP linting**: PHPCS with configurable WordPress Coding Standards
- **Environment-based deployments**: Development, staging, and production
- **Automatic cache clearing**: Host-specific cache purging after deploy
- **Slack notifications**: Configurable deployment alerts with lint & deploy status
- **Flexible configuration**: Per-project settings via `wp-deploy-config.yml`

## Deploy Quick Start

### 1. Add the Caller Workflow

Create `.github/workflows/deploy.yml` in your project:

```yaml
name: 'Deploy'

on:
  push:
    branches: [main, staging, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

jobs:
  deploy:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/wordpress-deploy.yml@v1
    with:
      environment: ${{ github.event.inputs.environment || (github.ref_name == 'main' && 'production') || (github.ref_name == 'staging' && 'staging') || 'development' }}
    secrets: inherit
```

### 2. Add Project Configuration

Create `wp-deploy-config.yml` in your project root:

```yaml
project:
  type: theme          # theme | plugin | mu-plugin
  slug: my-theme       # Directory name on server
  host: kinsta         # kinsta | wpengine | pressable | generic

slack:
  channel: deployments
  username: DeployBot
  icon_emoji: ':rocket:'

excludes:
  - 'src/'
  - '*.map'
  - 'package.json'
  - 'package-lock.json'
  - 'webpack.config.js'
```

### 3. Configure Repository Secrets & Variables

Add these to your repository (Settings → Secrets and variables → Actions):

#### Required Secrets

| Secret | Description |
|--------|-------------|
| `SSH_PRIVATE_KEY` | Private key for SSH authentication |

#### Required Variables or Secrets

These can be set as either variables or secrets (variables take precedence):

| Name | Description | Example |
|------|-------------|---------|
| `SSH_HOST` | Server hostname | `ssh.kinsta.cloud` |
| `SSH_USER` | SSH username | `deploy-user` |
| `WP_CONTENT_PATH` | Path to the `wp-content` directory. A path without `wp-content` has `/wp-content` appended | `/www/mysite/public/wp-content` |

#### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `SSH_PORT` | SSH port. Can also be a secret | `22` |
| `HOST_TYPE` | Override host type from config | Value from `wp-deploy-config.yml` |

#### Optional Secrets

| Secret | Description |
|--------|-------------|
| `SLACK_WEBHOOK` | Slack incoming webhook URL for notifications |

### 4. Set Up Environments (Recommended)

For environment-specific deployments, create environments in your repository (Settings → Environments):

- `development`
- `staging`
- `production`

Each environment can have its own secrets/variables and protection rules (e.g., required reviewers for production).

## Configuration Reference

### `wp-deploy-config.yml`

A complete example lives in [`examples/wp-deploy-config.yml`](examples/wp-deploy-config.yml). Branch-to-environment mapping is not set here; it comes from the caller workflow (see [Branch → Environment Mapping](#branch--environment-mapping)).

```yaml
project:
  # Required: Type determines the deployment subdirectory
  type: theme          # theme → /themes/, plugin → /plugins/, mu-plugin → /mu-plugins/
  
  # Required: The directory name for your project on the server
  slug: my-theme
  
  # Optional: Default host type (can be overridden by HOST_TYPE variable)
  host: kinsta         # kinsta | wpengine | pressable | generic

slack:
  # Optional: Slack channel for notifications (default: deployments)
  channel: deployments
  
  # Optional: Bot username (default: DeployBot)
  username: DeployBot
  
  # Optional: Bot emoji (default: :rocket:)
  icon_emoji: ':rocket:'

# Optional: Additional files/patterns to exclude from deployment
excludes:
  - 'src/'
  - 'node_modules/'
  - '*.map'
  - 'package.json'
  - 'package-lock.json'
  - 'webpack.config.js'
  - 'tailwind.config.js'
  - 'postcss.config.js'
  - '.browserslistrc'
  - 'tests/'
  - '*.md'
```

### Default Excludes

These are always excluded (in addition to your custom excludes):

- `.git*`, `.github*`
- `node_modules/`
- `vendor/`
- `wp-deploy-config.yml`
- `phpcs.xml`, `phpcs.xml.dist`
- `composer.json`, `composer.lock`
- `.editorconfig`
- `.zencoder`, `.cursor`, `.vscode`, `.idea`

`vendor/` is always excluded, so a plugin that needs Composer packages at runtime will not have them on the server.

### Files Removed on the Server

rsync runs with `--delete --delete-excluded`. After a deploy, the project's directory on the server matches the repository exactly:

- Files on the server that are not in the repository are deleted.
- Files that match an exclude pattern are deleted from the server too, even if they were uploaded some other way.

Anything a site writes into the theme or plugin directory at runtime, such as generated caches or uploaded files, is removed on the next deploy. Store that data under `wp-content/uploads` instead.

## Deploy Workflow Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `environment` | Target environment | Yes | — |
| `php-version` | PHP version for linting | No | `8.2` |
| `phpcs-standard` | PHPCS coding standard (see below) | No | `WordPress-Core` |
| `skip-lint` | Skip linting step entirely | No | `false` |
| `config-file` | Path to config file | No | `wp-deploy-config.yml` |

### Example with Custom Inputs

```yaml
jobs:
  deploy:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/wordpress-deploy.yml@v1
    with:
      environment: production
      php-version: '8.1'
      phpcs-standard: 'WordPress'
      config-file: 'config/deploy.yml'
    secrets: inherit
```

### Skip Linting

```yaml
jobs:
  deploy:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/wordpress-deploy.yml@v1
    with:
      environment: staging
      skip-lint: true
    secrets: inherit
```

## Linting Standards

The lint job installs PHPCS one of two ways:

- **No `composer.json` in the project:** it installs PHP_CodeSniffer 3 and WordPress Coding Standards 3 on the fly.
- **A `composer.json` in the project:** it runs `composer install` and uses the project's `vendor/bin/phpcs`. The project's `require-dev` must include `squizlabs/php_codesniffer` and `wp-coding-standards/wpcs`, or the lint job fails.

The workflow supports four WordPress coding standards via the `phpcs-standard` input:

| Standard | Description | Strictness |
|----------|-------------|------------|
| `WordPress-Core` | Code style & formatting (default) | Base level |
| `WordPress-Docs` | Documentation requirements only | Docs only |
| `WordPress-Extra` | Best practices & performance warnings | Stricter |
| `WordPress` | All of the above combined | Most strict |

### WordPress-Core (Recommended Default)

Enforces essential coding standards:
- Indentation (tabs, not spaces)
- Spacing around operators, parentheses, brackets
- Yoda conditions (`if ( true === $var )`)
- Naming conventions (snake_case)
- Proper escaping (`esc_html()`, `esc_attr()`, etc.)
- Input sanitization
- Nonce verification

### WordPress-Extra

Adds advisory warnings beyond Core:
- Slow query warnings (`meta_query`, `tax_query`)
- Discouraged functions (`eval()`, `extract()`)
- Direct database queries without caching
- Assignment in conditions
- Direct filesystem calls (prefer `WP_Filesystem`)
- Loose comparisons (`==` vs `===`)

### WordPress-Docs

Documentation requirements:
- File-level docblocks
- Function/method docblocks with `@param`, `@return`
- Class docblocks

### Using a Project phpcs.xml

If your project contains a `phpcs.xml` or `phpcs.xml.dist` file, it will be used **instead of** the `phpcs-standard` input. This allows full customization:

```xml
<?xml version="1.0"?>
<ruleset name="My Project">
    <file>.</file>
    <exclude-pattern>vendor/*</exclude-pattern>
    <exclude-pattern>node_modules/*</exclude-pattern>
    
    <rule ref="WordPress-Core"/>
    
    <!-- Or use full WordPress standard with exclusions -->
    <!-- 
    <rule ref="WordPress">
        <exclude name="WordPress.DB.SlowDBQuery"/>
        <exclude name="WordPress.WP.AlternativeFunctions"/>
    </rule>
    -->
</ruleset>
```

### Exposing Linting Standard as a Dropdown

To let users choose the linting standard when manually triggering deployments:

```yaml
name: 'Deploy'

on:
  push:
    branches: [main, staging, develop]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
      phpcs-standard:
        description: 'Linting standard'
        required: false
        type: choice
        options:
          - WordPress-Core
          - WordPress
          - WordPress-Extra
          - WordPress-Docs
        default: 'WordPress-Core'

jobs:
  deploy:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/wordpress-deploy.yml@v1
    with:
      environment: ${{ github.event.inputs.environment || (github.ref_name == 'main' && 'production') || (github.ref_name == 'staging' && 'staging') || 'development' }}
      phpcs-standard: ${{ github.event.inputs.phpcs-standard || 'WordPress-Core' }}
    secrets: inherit
```

## Host-Specific Setup

### Kinsta

1. Generate an SSH key pair
2. Add public key to MyKinsta → Sites → Your Site → Info → SFTP/SSH
3. Set secrets:
   - `SSH_HOST`: `ssh.kinsta.cloud`
   - `SSH_PORT`: Your site's SSH port (from MyKinsta)
   - `SSH_USER`: Your site's SSH username (from MyKinsta)
   - `WP_CONTENT_PATH`: `/www/{site}_xxx/public/wp-content`

### WP Engine

1. Generate an SSH key pair
2. Add public key to WP Engine portal → Your Install → Git Push
3. Set secrets:
   - `SSH_HOST`: `{install}.ssh.wpengine.net`
   - `SSH_USER`: `{install}`
   - `WP_CONTENT_PATH`: `/sites/{install}/wp-content`

### Pressable

1. Generate an SSH key pair
2. Add public key via Pressable SFTP settings
3. Set secrets accordingly based on Pressable's SSH access details

## Branch → Environment Mapping

By default, the caller workflow maps branches to environments:

| Branch | Environment |
|--------|-------------|
| `main` | `production` |
| `staging` | `staging` |
| `develop` | `development` |

Manual deployments via `workflow_dispatch` allow selecting any environment.

## Slack Notifications

When `SLACK_WEBHOOK` is configured, notifications include:

- Repository name
- Branch name
- Environment
- Lint status (Passed / Failed / Skipped)
- Deploy status (Success / Failed)
- Triggered by (username)

## Deploy Troubleshooting

### SSH Connection Failed

- Verify the public key is added to your hosting provider
- Check `SSH_HOST`, `SSH_USER`, and `SSH_PORT` are correct
- Ensure the private key in `SSH_PRIVATE_KEY` includes the full key with headers

### PHPCS Errors

- If the log says `vendor/bin/phpcs` is not found, the project has a `composer.json` without PHPCS in `require-dev`. See [Linting Standards](#linting-standards)
- Add a `phpcs.xml` to your project to customize rules
- Use `phpcs-standard: 'WordPress-Core'` for less strict linting
- Use `skip-lint: true` to bypass linting temporarily (not recommended for production)

### Files Not Deploying

- Check if the file matches an exclude pattern, including the [default excludes](#default-excludes)
- Review the rsync output in the workflow logs

### Files Disappeared From the Server

- The deploy deletes anything in the project's server directory that is not in the repository. See [Files Removed on the Server](#files-removed-on-the-server)

### Cache Not Clearing

- Verify `HOST_TYPE` variable or `project.host` in config matches your host
- Some hosts require WP-CLI to be available on the server

## PR Build Check

Reusable workflow: [`.github/workflows/pr-build.yml`](.github/workflows/pr-build.yml).

Runs `npm ci` and the project's build on each pull request, then confirms the expected build output exists. A broken build fails the pull request instead of surfacing at deploy time.

A repository without `package.json` passes with a notice, so PHP-only themes and plugins can use the same caller file. A repository with `package.json` but no `package-lock.json` fails, because `npm ci` requires the lockfile.

### Add the caller workflow

Create `.github/workflows/pr-build.yml`:

```yaml
name: 'PR Build Check'

on:
  pull_request:
  workflow_dispatch:

jobs:
  build:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/pr-build.yml@v1
```

A project whose build writes somewhere other than `dist/` sets `output-paths`. For example, to require both the JavaScript and CSS folders:

```yaml
jobs:
  build:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/pr-build.yml@v1
    with:
      output-paths: dist/js dist/css
```

### PR Build Check inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node-version-file` | File that pins the Node version | No | `.nvmrc` |
| `node-version` | Node version used when `node-version-file` does not exist | No | `24` |
| `build-command` | Command that builds the assets | No | `npm run build` |
| `output-paths` | Space-separated paths that must exist after the build. Empty skips the check | No | `dist` |

## Renovate → ClickUp

Reusable workflow: [`.github/workflows/renovate-clickup.yml`](.github/workflows/renovate-clickup.yml).

Mirrors Mend Renovate pull requests into ClickUp, with one task per dependency update. ClickUp IDs stay out of every repository. Each repo is routed with an organization custom property, and the workflow finds or creates a **Dependency Updates** list in that space.

An update often gets several pull requests. Renovate opens each one against the default branch. When one is retargeted to `develop` or `staging` and merged, Renovate opens a new pull request for the same update against the default branch. All of those pull requests share one task, which stays open until the update is merged into the final branch.

| Pull request event | ClickUp result |
|--------------------|----------------|
| `opened` | Create the task. If the update already has one, comment with the new pull request and reopen the task if it was closed |
| `reopened` | Same as `opened` |
| `edited` (title only) | Rename the task |
| `closed`, merged into the final branch | Comment that it was merged, then close the task |
| `closed`, merged into any other branch | Comment that it was merged there, and leave the task open |
| `closed` without merging | Comment that it was closed, then close the task |

The final branch is the repository's default branch unless the `final-branch` input says otherwise.

Pull requests from any author other than `renovate[bot]` are ignored. Edits that do not change the title are ignored. A repository with neither custom property below is skipped.

### 1. Add the caller workflow

Create `.github/workflows/renovate-clickup.yml` in each repository Renovate opens pull requests against:

```yaml
name: Renovate → ClickUp

on:
  pull_request:
    types: [opened, reopened, edited, closed]

jobs:
  sync:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/renovate-clickup.yml@v1
    secrets:
      CLICKUP_API_TOKEN: ${{ secrets.CLICKUP_API_TOKEN }}
```

`secrets: inherit` works too, as long as the caller can see `CLICKUP_API_TOKEN`.

### 2. Route the repository

In the organization: **Settings → Repository → Custom properties**.

| Property | Required | Value |
|----------|----------|-------|
| `clickup_space` | Yes, unless `clickup_list_id` is set | ClickUp space name or space ID |
| `clickup_list_id` | No | List ID. When set, the workflow uses this list and does not look up the space |

Set `clickup_space` on each repository. The name match is case-insensitive. The token's workspaces are searched until a space matches.

The workflow then looks for a list named `Dependency Updates` in that space and creates it when it is missing. A list created by the workflow keeps ClickUp's default icon and sets that icon's color to orange. Override the list name with the `list-name` input.

If `clickup_space` is set but no space matches, the job fails. If neither property is set, the job exits successfully and logs a notice.

### 3. Add the ClickUp token

Store a ClickUp personal API token (`pk_...`) as the Actions secret `CLICKUP_API_TOKEN`. An organization secret lets every caller use the same token.

The token needs access to the destination workspace, including permission to create lists and tasks.

### What gets created

Tasks are named `[owner/repo] PR title`. Later events find the task by that name, so every pull request with the same title in the same repository shares it. Renovate puts the target version in the title, so a newer version of the same dependency gets its own task. When Renovate changes the title of an open pull request, the task is renamed with it.

Tasks created before this naming are named `[owner/repo#123] PR title`. Those are still found by their title and are renamed to the new format on their next event.

The description comes from the first pull request: its link, title, branch, and labels. The default tag is `renovate`. Each later pull request and every merge or close is added as a comment that names the pull request and its target branch. Closing moves the task to the list's closed or done status. Reopening moves it back to the list's open status.

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `list-name` | List to find or create in the repository's space | No | `Dependency Updates` |
| `bot-login` | Pull request author to track | No | `renovate[bot]` |
| `tags` | Comma-separated ClickUp tags for new tasks | No | `renovate` |
| `priority` | ClickUp priority: `1` urgent, `2` high, `3` normal, `4` low. Empty sets none | No | empty |
| `final-branch` | Branch whose merge closes the task. Empty uses the repository's default branch | No | empty |

### Example with custom inputs

```yaml
jobs:
  sync:
    uses: dev-hero/reusable-wp-deploy/.github/workflows/renovate-clickup.yml@v1
    with:
      list-name: Renovate
      tags: renovate, dependencies
      priority: '3'
    secrets:
      CLICKUP_API_TOKEN: ${{ secrets.CLICKUP_API_TOKEN }}
```

### Renovate → ClickUp troubleshooting

- **Nothing was created.** Confirm the pull request author is `renovate[bot]` (or whatever `bot-login` is set to), and that the repository has `clickup_space` or `clickup_list_id`.
- **Job failed looking up the space.** The token cannot see a space with that name or ID. Check the property value and the token's workspace access.
- **Task did not reopen or close.** The destination list needs a status of type open, and a status of type closed or done. The close comment is still posted when a status is missing.
- **Task stayed open after a merge.** The pull request was merged into a branch other than the final branch. Check the task's comments for the target branch, and set `final-branch` if updates finish somewhere other than the default branch.
- **Two tasks for one update.** The pull requests had different titles, for example after Renovate moved an open pull request to a newer version and an older pull request had already closed. Tasks are matched by title, so close the older one by hand.

## Versioning

Every workflow in this repository is released together under one set of semantic version tags:

| Reference | Description | Use Case |
|-----------|-------------|----------|
| `@v1` | Latest v1.x.x | Recommended for most projects |
| `@vX.Y.Z` | Exact version | When you need guaranteed stability |
| `@main` | Latest development | Testing new features |

### Releasing

Releases are automated with [release-please](https://github.com/googleapis/release-please-action) through `release.yml`:

1. Merge changes to `main` with a [Conventional Commits](https://www.conventionalcommits.org/) message. When squash merging, the squashed commit takes the commit message for a single-commit pull request and the pull request title otherwise, so give both the prefix.
2. release-please opens or updates a release pull request. It bumps `version.txt` and `.release-please-manifest.json` and adds the new entry to `CHANGELOG.md`.
3. Merging the release pull request creates the `vX.Y.Z` tag and a GitHub Release with the same notes.
4. `release.yml` then moves the major tag (`v1`) to the new release. Projects on `@v1` pick it up on their next run. Projects pinned to an exact tag stay on it until their caller changes.

The prefix decides the version bump and the changelog section:

| Prefix | Bump | Changelog section |
|--------|------|-------------------|
| `feat:` | Minor | Added |
| `fix:` | Patch | Fixed |
| `perf:`, `refactor:`, `revert:` | Patch | Changed |
| `docs:` | Patch | Documentation |
| `chore:`, `ci:`, `build:`, `test:`, `style:` | None | Hidden |
| `feat!:` or a `BREAKING CHANGE:` footer | Major, and callers on `@v1` must move to the new major tag | Breaking changes |

Commits without a prefix are left out of the changelog and do not trigger a release.

Renovate updates to the actions these workflows use are titled `fix(deps): ...`, so each one ships as a patch release.

Nothing references `release.yml`. It has no `workflow_call` trigger and only runs in this repository. Pushing a version tag by hand still works: the tag trigger moves `v1` to it.

### Renamed workflows

`v1.0.5` shipped the ClickUp workflow as `reusable-renovate-clickup.yml`. From `v1.0.6` it is `renovate-clickup.yml`. A caller on `@v1` still pointing at the old name fails once `v1` moves to `v1.0.6` or later, so update the `uses:` line in each project:

```yaml
uses: dev-hero/reusable-wp-deploy/.github/workflows/renovate-clickup.yml@v1
```

## Contributing

1. Create a branch from `main`.
2. Make the change, and update this README in the same pull request when a workflow's inputs, secrets, or behavior change. Releases do not touch the README.
3. Open a pull request with a Conventional Commits title, such as `feat: add php-lint input to pr-build`. See [Releasing](#releasing) for how the prefix sets the version.
4. Squash and merge. Do not edit `CHANGELOG.md` or `version.txt` by hand; release-please updates both.

## License

MIT License - See [LICENSE](LICENSE) for details.
