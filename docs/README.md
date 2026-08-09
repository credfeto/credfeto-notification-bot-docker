# Documentation

Operational and architecture notes for this deployment should be added here.

## Rootless podman migration, tried and reverted

Between July and August 2026 this deployment was migrated from root-run
`docker compose` (with a `watchtower` container polling the docker socket)
to rootless `podman compose` run by a dedicated unprivileged system user.
It was verified working end-to-end on the real host, but rootless podman
proved unreliable in production use there and the deployment was reverted
back to the docker-based setup this file now describes as current.

The full migration writeup (systemd unit design, the four real-host issues
hit and fixed, the firewalld/journald gotchas, the trade-offs made) is not
reproduced here since it no longer describes current behaviour, but is
preserved in git history — see PRs #14, #15, #18, #19 and issue #20 (the
revert) — and the lessons that remain relevant to this docker-based setup
are kept in `ai/local/deployment-troubleshooting.instructions.md` (marked
historical where podman-specific).
