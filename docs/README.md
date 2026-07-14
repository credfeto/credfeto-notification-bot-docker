# Documentation

Operational and architecture notes for this deployment should be added here.

## Rootless podman migration (issue #12)

This deployment was migrated from root-run `docker compose` (with a
`watchtower` container polling the docker socket) to rootless `podman
compose` run by a dedicated unprivileged system user, `notification-bot`.

### Why two systemd units

- `credfeto-notification-bot.service` is the actual container runner. It's
  a boot-time system unit (`WantedBy=multi-user.target`), not a `systemctl
  --user` unit — the latter would need `loginctl enable-linger` to survive
  without an active login session, and the ask was explicitly for a system
  service. It runs `run-compose` as `User=notification-bot`, with
  `RuntimeDirectory=notification-bot` providing `XDG_RUNTIME_DIR` so
  rootless podman has somewhere to keep its runtime state. It has no
  `ExecStop` and sets `KillMode=none`, so that `systemctl restart` doesn't
  send a cgroup-wide kill to the containers on the stop half before
  `run-compose` (`podman compose pull && podman compose up -d`) runs again
  on the start half — `up -d` only recreates containers whose image or
  config actually changed. **This is unverified** — see "Known gaps" below.
- `credfeto-notification-bot-update.timer`/`.service` (renamed from the
  previous `credfeto-notification-bot.timer`/`.service`, which did this job
  under the old docker-based design) still runs as root every 5 minutes,
  because it needs to write to the repo checkout, regenerate the TLS cert,
  and fix up config file ownership — all root-only operations, since the
  checkout at `/opt/credfeto-notification-bot-docker` is `root:notification-bot`
  with group-read-only permissions.

### Privilege boundary

- `install` is the only script that needs broad root privileges on an
  ongoing basis (user/volume/firewall/systemd setup). It's idempotent and
  safe to re-run. It also drops to `notification-bot` (via `runuser`) for
  the one-off `podman volume create`/`inspect` calls.
- `update` self-elevates via `sudo` if not already root, then does all its
  work as root (git pull, cert generation, config permissions). It never
  invokes podman itself — it always ends by restarting
  `credfeto-notification-bot.service`, which is the only place podman
  actually runs (see "Known gaps" for why).
- `reset` self-elevates the same way, and drops to `notification-bot` (via
  `runuser`) to explicitly stop/remove/prune containers — that's fine
  because it's run interactively, not spawned by a systemd oneshot unit
  that will reap its own cgroup on exit (see "Known gaps").
- `credfeto-notification-bot.service` itself never runs as root. It adds
  `NoNewPrivileges=yes`, `ProtectSystem=strict` (with explicit
  `ReadWritePaths` for the podman storage dir and `/data/dispatcher`),
  `ProtectHome=yes`, and `PrivateTmp=yes` — **unverified**, see "Known gaps".

### Known gaps / follow-ups for whoever runs this on the real host

These three are the top priority to verify — everything else in this
section is lower-severity:

- **`systemctl restart credfeto-notification-bot.service` may still kill
  the containers.** Rootless podman is daemonless: `conmon`/container
  processes normally live inside the launching unit's cgroup, and systemd's
  default `KillMode=control-group` kills the whole cgroup on the stop half
  of a restart — the absence of an `ExecStop` line doesn't prevent that,
  only `KillMode=none` (now set) does. This wasn't verified against a real
  rootless podman + systemd setup — no host with a properly-provisioned
  rootless user was available while writing this. **Before trusting
  `update`'s behaviour** (it restarts this unit on every 5-minute tick),
  watch `podman ps`/`podman compose ps` across a manual
  `systemctl restart credfeto-notification-bot.service` and confirm the
  containers are recreated, not bounced with a gap, and that they survive
  at all. If they still get killed, look at `podman generate systemd` or
  Quadlet (`.container` unit files) instead of `KillMode=none` — and note
  that testing this by running `update`/`run-compose` manually from an
  interactive shell is **not** equivalent to the real timer path: a login
  session's cgroup persists across the command exiting, but the timer
  invokes `update` as a `Type=oneshot` unit whose cgroup systemd reaps the
  moment it exits, which is exactly the scenario `KillMode=none` (on the
  *container* unit, which `update` now always defers to rather than ever
  running podman directly) exists to avoid. Verify with
  `systemctl start credfeto-notification-bot-update.service`, not a manual
  `sudo ./update`.
- **The systemd sandboxing directives are just as unverified as the
  container hardening deliberately left out below.** `ProtectSystem=strict`,
  `ProtectHome=yes`, and `PrivateTmp=yes` together are a classic way for a
  rootless podman unit to fail outright (can't write its storage/lock
  files) — they're plausible-but-unverified, not a proven-safe default. After
  `install` enables the unit, check `systemctl status
  credfeto-notification-bot.service`. If it fails to start, strip these
  three down to `NoNewPrivileges=yes` only, confirm the unit starts, then
  re-add one at a time (with `ReadWritePaths` adjusted as needed) rather
  than assuming all three together are fine.
- **`HOME` for the `notification-bot` user in `runuser` calls.** `install`
  and `reset` invoke podman as `notification-bot` via
  `runuser -u notification-bot -- env HOME=/var/lib/notification-bot
  XDG_RUNTIME_DIR=/run/notification-bot ...`, explicitly setting `HOME`
  because `runuser` without `--login` otherwise leaves the caller's `HOME`
  (`/root`, since these scripts self-elevate) in place, which would point
  rootless podman's storage at the wrong (root-owned, unwritable) path.
  `credfeto-notification-bot.service` doesn't have this problem —
  `User=notification-bot` makes systemd set `HOME` correctly on its own.
  Confirm on the real host that `runuser -u notification-bot --
  printenv HOME` actually prints `/var/lib/notification-bot`; if `runuser`
  behaves differently there, the explicit `HOME=` override should still
  win, but this hasn't been exercised end-to-end.
- **Not tested against the real registry or the real host.** `podman
  compose config` was used to confirm `docker-compose.yml` parses correctly
  under podman, and a public test image was pulled to confirm rootless
  podman itself works, but the actual application images live on
  `docker-registry.markridgwell.com`, which wasn't reachable from the
  environment this was developed in, and the full privileged `install` flow
  (creating a system user, subuid/subgid allocation, firewalld, systemd)
  was not exercised end-to-end. Run `./install` then `./run-compose`
  manually and check `podman compose ps` / `journalctl -u
  credfeto-notification-bot.service` on the real host before trusting the
  timer.
- **Subuid/subgid range.** `install` allocates `200000-265535` to
  `notification-bot` if it doesn't already have a range in `/etc/subuid` /
  `/etc/subgid`. If that range collides with another user on the host,
  `useradd`/`usermod` will error — pick a different range by hand and rerun.
- **Container hardening not applied blind.** `security_opt:
  no-new-privileges:true` was added to all three services. `read_only:
  true` and `cap_drop: [ALL]` were deliberately *not* added, since these
  images weren't available to test against and either could silently break
  the app (e.g. if it writes temp files) with no way to catch that here.
  Worth doing once you can verify each container still works with them on.
- **`dispatcher-data` volume stays unmounted.** The compose file declares
  `dispatcher-data` as an external volume but no service actually mounts
  it — that was already true before this migration and is left as-is.
- **Watchtower removed.** Automatic redeploy on a bare `:latest` image push
  (independent of this repo changing) is now handled by `update`
  unconditionally restarting `credfeto-notification-bot.service` every
  5-minute tick — `podman compose up -d` only recreates containers whose
  image actually changed — rather than a container polling the
  docker/podman socket every 60s.
