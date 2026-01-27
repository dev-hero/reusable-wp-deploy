# Reusable WordPress Deploy Workflow

A centralized, reusable GitHub Actions workflow for deploying WordPress themes, plugins, and mu-plugins to managed hosting environments.

## Features

- **Multi-host support**: Kinsta, WP Engine, Pressable, and generic WordPress hosts
- **Project type detection**: Automatically deploys to correct directory (themes/plugins/mu-plugins)
- **PHP linting**: PHPCS with configurable WordPress Coding Standards
- **Environment-based deployments**: Development, staging, and production
- **Automatic cache clearing**: Host-specific cache purging after deploy
- **Slack notifications**: Configurable deployment alerts with lint & deploy status
- **Flexible configuration**: Per-project settings via `wp-deploy-config.yml`

## Quick Start

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
| `SSH_PORT` | SSH port (default: 22) | `22` |
| `WP_CONTENT_PATH` | Path to wp-content directory | `/www/mysite/public/wp-content` |

#### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
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

## Workflow Inputs

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

## Versioning

This workflow follows semantic versioning:

| Reference | Description | Use Case |
|-----------|-------------|----------|
| `@v1` | Latest v1.x.x | Recommended for most projects |
| `@v1.2.0` | Exact version | When you need guaranteed stability |
| `@main` | Latest development | Testing new features |

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

## Troubleshooting

### SSH Connection Failed

- Verify the public key is added to your hosting provider
- Check `SSH_HOST`, `SSH_USER`, and `SSH_PORT` are correct
- Ensure the private key in `SSH_PRIVATE_KEY` includes the full key with headers

### PHPCS Errors

- Add a `phpcs.xml` to your project to customize rules
- Use `phpcs-standard: 'WordPress-Core'` for less strict linting
- Use `skip-lint: true` to bypass linting temporarily (not recommended for production)

### Files Not Deploying

- Check if the file matches an exclude pattern
- Review the rsync output in the workflow logs

### Cache Not Clearing

- Verify `HOST_TYPE` variable or `project.host` in config matches your host
- Some hosts require WP-CLI to be available on the server

## Contributing

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## License

MIT License - See [LICENSE](LICENSE) for details.
