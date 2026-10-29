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
- Added the workflow-mcp service, served over HTTPS on host port 8091 from the ghcr.io/credfeto/workflow-mcp:latest image and configured from workflow-mcp/appsettings-local.json
- update generates a self-signed TLS certificate for workflow-mcp, owned by the container's non-root user so the service can read it, and opens port 8091 in the firewall for private networks only
- workflow-mcp runs with a read-only root filesystem, all Linux capabilities dropped and no-new-privileges set, so a compromised container has less scope to cause harm
### Fixed
- update now checks that the checkout is root-owned before pulling, with an actionable error, instead of silently failing at git's dubious-ownership check every timer tick
- Removed NoNewPrivileges=yes from credfeto-notification-bot.service, which blocked rootless podman's newuidmap/newgidmap from creating a user namespace on every reboot, causing the containers to fail to start
- Removed PrivateTmp=yes from credfeto-notification-bot.service - rootless podman's long-lived pause process could end up bound to a private /tmp mount torn down by a later restart, breaking image pulls with no automatic recovery
- Corrected the invalid :r mount mode to :ro on both appsettings-local.json mounts, which made docker-compose.yml unparseable by non-Docker compose tooling
- Renamed the tracked notifications folder to notification, so the compose and script paths match the folder name
- github-api-proxy always publishes host port 3000, matching the port update always opens in the firewall, so a PORT override can no longer leave the proxy unreachable on the opened port
- install and update remove an empty directory that Docker created in place of a missing bind-mounted config file or TLS certificate, stop with an actionable message when it is not empty, and tell the operator to force-recreate the containers once the file exists
### Changed
### Deprecated
### Removed
### Deployment Changes
- Migrated the deployment from root-run docker compose (with a watchtower container polling the docker socket) to rootless podman compose run by an unprivileged systemd service, for least-privilege container operation
- Reverted the deployment from rootless podman back to root-run docker compose + watchtower after rootless podman proved unreliable in production
- Mounted the dispatcher-data volume at /usr/src/app/data on dispatcher-bot so the in-memory store's JSON snapshot survives container recreation

<!--
Releases that have at least been deployed to staging, BUT NOT necessarily released to live.  Changes should be moved from [Unreleased] into here as they are merged into the appropriate release branch
-->

## [0.0.0] - Project created