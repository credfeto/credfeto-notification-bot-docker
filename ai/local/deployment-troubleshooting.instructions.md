# Deployment Troubleshooting Instructions

[Back to Local AI Memory Index](index.md)

## `dispatcher-data` Volume Is Writable Only Because `dispatcher-bot` Runs As Root

- `docker-compose.yml` mounts the `dispatcher-data` external volume (bind-backed by
  `/data/dispatcher`, created root:root 0755 by `update`) read-write at
  `/usr/src/app/data` on `dispatcher-bot`. This exists so `credfeto/credfeto-dispatcher`'s
  periodic JSON snapshot of its in-memory store (added in
  [credfeto-dispatcher#211](https://github.com/credfeto/credfeto-dispatcher/pull/211))
  survives container recreation instead of being lost with the writable layer.
- This only works because `credfeto-dispatcher`'s `Dockerfile` has no `USER` directive - the
  container process runs as `root`, which can write to a root-owned bind mount with no
  further permission changes needed.
- If that Dockerfile is ever changed to run as a non-root user (the global docker
  instructions recommend this), the mount becomes silently unwritable for that user: no
  container crash, just a logged `SnapshotSaveFailed` on every write attempt and the
  snapshot never persisting. Containers otherwise report healthy, so this needs an explicit
  `docker logs dispatcher-bot | grep SnapshotSaveFailed` check to catch, not just `docker ps`.
- Fix if that happens: `update` needs to `chown <uid>:<gid> /data/dispatcher` to the image's
  internal UID/GID after `mkdir -p`, idempotently, alongside the existing directory/volume
  creation block.

## Missing `:latest` Tag on `docker-registry.markridgwell.com` (informational)

- `docker compose pull` can fail on just one image with `manifest unknown` for the `:latest` tag, even though `docker/build-push-action` reported a successful push of `:latest` for that exact commit in CI. Confirm with `curl -s https://docker-registry.markridgwell.com/v2/<repo>/tags/list` - if `latest` is absent while commit-sha tags are present (and those sha tags are for old commits, not the latest main build), the registry has somehow lost the `latest` tag after a genuinely successful push.
- This is a registry-side issue (`docker-registry.markridgwell.com`, host `192.168.150.250`) outside this repo and outside `notifications.lan` - not something to chase further from here. It is runtime-agnostic: first observed under `podman-compose pull` during the rootless-podman era (see Historical section below), but nothing about the cause is podman-specific, so it can equally recur under plain `docker compose pull`.
- Workaround that resolved it on 2026-08-06: `gh run rerun <run-id> --repo credfeto/credfeto-github-api-proxy` on the last successful main-branch "Docker: Build and Push" run re-pushed `:latest` successfully. If it recurs, re-running the source repo's build-and-push workflow is the quick fix; only investigate the registry itself if re-running stops working.

## Historical: Rootless Podman Era (July-August 2026, reverted in issue #20)

The deployment ran on rootless podman with a dedicated `notification-bot`
system user for roughly a month (PRs #14, #15, #18, #19) before being
reverted back to root-run `docker compose` + `watchtower` (issue #20)
because rootless podman proved unreliable in production. The notes below
describe incidents from that era. None of them apply to the current
docker-based deployment - there is no `notification-bot` user, no rootless
namespace, and containers run as root via `docker compose` directly - but
they are kept in case a podman-based approach is ever revisited.

### `/opt/credfeto-notification-bot-docker` Must Stay Root-Owned

- `update` always re-execs itself as root (see the `exec sudo "$0" "$@"` guard near the top) and then runs `git pull --ff-only` in the checkout. Git refuses to operate in a repo whose top-level directory it doesn't own ("detected dubious ownership"), so if that directory's owner is ever changed away from `root` (e.g. `chown -R notification-bot:notification-bot` applied to the whole checkout), every future `update` run dies at the `git pull` line - before it ever reaches the line that restarts `credfeto-notification-bot.service`.
- This fails **silently** from the operator's point of view: the container service itself doesn't error again, it just never gets retried. A transient, unrelated failure (e.g. a one-off `newuidmap` race right after a host reboot) then looks like a permanently broken service, when the actual blocker is upstream in the update timer.
- Symptom: `sudo systemctl status credfeto-notification-bot-update.service` shows `fatal: detected dubious ownership in repository at '/opt/credfeto-notification-bot-docker'`, and `credfeto-notification-bot.service` sits `failed` indefinitely with no further restart attempts in the journal after the initial failure.
- `update` (as of the fix that added this note) checks the top-level directory's owner UID explicitly before pulling and dies with an actionable message if it isn't root - no need to pattern-match git's error text by hand anymore.
- Fix if it happens anyway: `sudo chown root:notification-bot /opt/credfeto-notification-bot-docker` (top-level directory only), then `sudo systemctl restart credfeto-notification-bot.service` to recover immediately rather than waiting for the next timer tick.
- Mixed ownership **deeper** in the checkout (some files `root`, some `notification-bot`) is normal and not itself a bug: `install`/`update` only `chgrp` individual config files and use `chmod -R g+rX` for group-read, they never assert a single owner across the whole tree. Do not "fix" that by recursively chowning the whole checkout to `notification-bot` - that is what breaks the top-level directory's ownership and triggers this failure mode.

### Getting Container Logs

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
- Under the current docker-based deployment, containers run as root and `docker compose logs -f` / `docker logs <name>` work directly as root with no equivalent reader restriction - this whole entry doesn't apply.

### Stale Rootless-Podman Pause Process After `PrivateTmp` Teardown

- `credfeto-notification-bot.service` used to set `PrivateTmp=yes`. Every start of that unit got its own private `/tmp` and `/var/tmp` bind mounts, torn down (unmounted, source directory deleted) when that particular service instance ended.
- Rootless podman keeps a long-lived "pause" process per UID (`/run/user/<uid>/libpod/tmp/pause.pid`, cgroup `podman-pause-*.scope`, reparented to PID 1) so it doesn't have to rebuild the user namespace on every invocation. If that pause process was created while a `PrivateTmp` mount namespace from one service instance was current, it kept that namespace's view of `/var/tmp` for as long as it lived - including after the owning service instance (and its private tmp) was gone.
- Symptom: `podman pull` / `podman-compose pull` (and hence `credfeto-notification-bot.service`) failed with `creating a temporary directory: mkdir /var/tmp/container_images_storageNNNNNNNN: no such file or directory`, even though `/var/tmp` plainly existed and a plain `mkdir` there (outside podman) succeeded. `sudo runuser -u notification-bot -- podman-compose pull` reproduced it directly; a plain `mkdir` test did not, because the plain shell wasn't joined to the stale namespace.
- This bug class doesn't exist for plain `docker compose` - no per-UID rootless pause process, no user namespace to go stale.

### Source

Real-host incidents on `notifications.lan`, 2026-07-18:

- A manual `chown -R notification-bot:notification-bot /opt/credfeto-notification-bot-docker` (done in response to seeing mixed ownership across the tree) silently wedged the update timer for hours. Diagnosed via `sudo journalctl -u credfeto-notification-bot-update.service` and `sudo journalctl -u credfeto-notification-bot.service`.
- `podman compose logs -f` returning nothing when run manually as `notification-bot`, diagnosed via `podman inspect` (journald driver) and confirmed working via `sudo journalctl CONTAINER_NAME=...`.

2026-08-06: all three containers down after a host reboot. `credfeto-notification-bot.service` failed with `podman-compose pull` exiting 125. Root cause was two independent problems stacked on top of each other: a stale rootless-podman pause process left over from a prior `PrivateTmp` service instance (see above), and a separately missing `:latest` tag for `credfeto/github-api-proxy` on the registry despite a reportedly successful CI push. Diagnosed via `podman unshare cat /proc/self/mountinfo`, `/proc/<pause-pid>/mountinfo`, and `curl .../v2/<repo>/tags/list` against the registry directly.

2026-08-09: rootless podman judged unreliable enough in production that the whole deployment was reverted back to docker + watchtower - see issue #20.
