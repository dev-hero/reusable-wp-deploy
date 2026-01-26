# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

### Changed

### Deprecated

### Removed

### Fixed

### Security

---

## [1.0.0] - 2025-01-26

### Added

- Initial release of the reusable WordPress deploy workflow
- Multi-job workflow with lint, deploy, and notify stages
- PHP CodeSniffer linting with WordPress Coding Standards
- SSH-based deployment via rsync
- Support for multiple project types:
  - Themes (`themes/`)
  - Plugins (`plugins/`)
  - MU-Plugins (`mu-plugins/`)
- Multi-host cache clearing support:
  - Kinsta (`wp kinsta cache purge`)
  - WP Engine (`wp engine cache flush`)
  - Pressable (`wp pressable cache purge`)
  - Generic (`wp cache flush`)
- Per-project configuration via `wp-deploy-config.yml`
- Configurable exclude patterns for rsync
- Slack notifications for deployment status
- Environment-based deployments (development, staging, production)
- Branch-to-environment mapping (main→production, staging→staging, develop→development)
- Manual deployment trigger via `workflow_dispatch`
- Configurable PHP version for linting
- Support for both repository variables and secrets for SSH configuration

---

## Version History

| Version | Date | Description |
|---------|------|-------------|
| [1.0.0] | 2025-01-26 | Initial release |

---

## Upgrade Guides

### Migrating from Inline Workflow to v1.0.0

If you're migrating from a standalone workflow in your project:

1. **Remove** your existing `.github/workflows/deploy.yml` (or similar)

2. **Create** a new minimal caller workflow:

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

3. **Ensure** your `wp-deploy-config.yml` exists and is properly configured

4. **Verify** all required secrets are set in your repository

---

## Release Process

For maintainers:

1. Update this CHANGELOG with all changes under `[Unreleased]`
2. Change `[Unreleased]` to the new version number and date
3. Add a new empty `[Unreleased]` section at the top
4. Commit: `git commit -am "Release vX.Y.Z"`
5. Tag: `git tag -a vX.Y.Z -m "Release vX.Y.Z"`
6. Push: `git push origin main --tags`
7. The release workflow will automatically:
   - Create a GitHub Release with auto-generated notes
   - Update the major version tag (e.g., `v1` → `vX.Y.Z`)

---

[Unreleased]: https://github.com/dev-hero/reusable-wp-deploy/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/dev-hero/reusable-wp-deploy/releases/tag/v1.0.0