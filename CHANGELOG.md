# Changelog

All notable changes to this project are documented in this file.

<!--
Please ADD ALL Changes to the UNRELEASED SECTION and not a specific release
-->

## [Unreleased]
### Security
### Added
- Added .ai-instructions and ai/local/index.md from cs-template standard
### Fixed
- update now checks that the checkout is root-owned before pulling, with an actionable error, instead of silently failing at git's dubious-ownership check every timer tick
### Changed
### Removed
### Deployment Changes
- Migrated the deployment from root-run docker compose (with a watchtower container polling the docker socket) to rootless podman compose run by an unprivileged systemd service, for least-privilege container operation
<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->
## [0.0.0] - Project created