# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

<!--
Please ADD ALL Changes to the UNRELEASED SECTION and not a specific release
-->

## [Unreleased]
### Security
### Added
- Added .ai-instructions and ai/local/index.md from cs-template standard
### Fixed
- update now checks that the checkout is root-owned before pulling, with an actionable error, instead of silently failing at git's dubious-ownership check every timer tick
- Removed NoNewPrivileges=yes from credfeto-notification-bot.service, which blocked rootless podman's newuidmap/newgidmap from creating a user namespace on every reboot, causing the containers to fail to start
- Removed PrivateTmp=yes from credfeto-notification-bot.service - rootless podman's long-lived pause process could end up bound to a private /tmp mount torn down by a later restart, breaking image pulls with no automatic recovery
### Changed
### Deprecated
### Removed
### Deployment Changes
- Migrated the deployment from root-run docker compose (with a watchtower container polling the docker socket) to rootless podman compose run by an unprivileged systemd service, for least-privilege container operation
<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->
## [0.0.0] - Project created