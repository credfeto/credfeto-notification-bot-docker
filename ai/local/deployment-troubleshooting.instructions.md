# Deployment Troubleshooting Instructions

[Back to Local AI Memory Index](index.md)

## `/opt/credfeto-notification-bot-docker` Must Stay Root-Owned (MANDATORY)

- `update` always re-execs itself as root (see the `exec sudo "$0" "$@"` guard near the top) and then runs `git pull --ff-only` in the checkout. Git refuses to operate in a repo whose top-level directory it doesn't own ("detected dubious ownership"), so if that directory's owner is ever changed away from `root` (e.g. `chown -R notification-bot:notification-bot` applied to the whole checkout), every future `update` run dies at the `git pull` line - before it ever reaches the line that restarts `credfeto-notification-bot.service`.
- This fails **silently** from the operator's point of view: the container service itself doesn't error again, it just never gets retried. A transient, unrelated failure (e.g. a one-off `newuidmap` race right after a host reboot) then looks like a permanently broken service, when the actual blocker is upstream in the update timer.
- Symptom: `sudo systemctl status credfeto-notification-bot-update.service` shows `fatal: detected dubious ownership in repository at '/opt/credfeto-notification-bot-docker'`, and `credfeto-notification-bot.service` sits `failed` indefinitely with no further restart attempts in the journal after the initial failure.
- `update` (as of the fix that added this note) checks the top-level directory's owner UID explicitly before pulling and dies with an actionable message if it isn't root - no need to pattern-match git's error text by hand anymore.
- Fix if it happens anyway: `sudo chown root:notification-bot /opt/credfeto-notification-bot-docker` (top-level directory only), then `sudo systemctl restart credfeto-notification-bot.service` to recover immediately rather than waiting for the next timer tick.
- Mixed ownership **deeper** in the checkout (some files `root`, some `notification-bot`) is normal and not itself a bug: `install`/`update` only `chgrp` individual config files and use `chmod -R g+rX` for group-read, they never assert a single owner across the whole tree. Do not "fix" that by recursively chowning the whole checkout to `notification-bot` - that is what breaks the top-level directory's ownership and triggers this failure mode.

## Getting Container Logs (MANDATORY)

- `podman compose logs -f` / `podman logs <name>`, run as `notification-bot` (e.g. via `runuser -u notification-bot`), come back **empty** with no error - do not conclude from this that a container is silent or misbehaving.
- Cause: the containers use the `journald` log driver (`podman inspect <name> --format '{{.HostConfig.LogConfig.Type}}'`). Rootless podman's journald *reader* cannot see those entries even though conmon successfully wrote them - they land in the **system** journal, which `notification-bot` has no read access to.
- Fix: read the system journal directly instead, as root:

  ```bash
  # live tail
  sudo journalctl CONTAINER_NAME=notification-bot -f

  # last N lines, no pager
  sudo journalctl CONTAINER_NAME=notification-bot -n 50 --no-pager
  ```

  Swap the container name for `dispatcher-bot` / `github-api-proxy` as needed.
- Separately: `runuser -u <user> -- ...` does **not** change the working directory - it inherits the caller's cwd verbatim. Running it from a directory `notification-bot` can't read (e.g. an admin's own home directory) breaks `podman`/`podman-compose` outright with `cannot chdir to <dir>: Permission denied`, unrelated to the journald issue above. Always `cd /opt/credfeto-notification-bot-docker` (or pass `-H`, but note that repoints `HOME` too) before a manual `runuser -u notification-bot -- podman ...` invocation.

## Source

Real-host incidents on `notifications.lan`, 2026-07-18:

- A manual `chown -R notification-bot:notification-bot /opt/credfeto-notification-bot-docker` (done in response to seeing mixed ownership across the tree) silently wedged the update timer for hours. Diagnosed via `sudo journalctl -u credfeto-notification-bot-update.service` and `sudo journalctl -u credfeto-notification-bot.service`.
- `podman compose logs -f` returning nothing when run manually as `notification-bot`, diagnosed via `podman inspect` (journald driver) and confirmed working via `sudo journalctl CONTAINER_NAME=...`.
