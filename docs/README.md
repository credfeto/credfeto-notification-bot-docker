# Documentation

Operational and architecture notes for this deployment should be added here.

## Rootless podman migration (issue #12)

This deployment was migrated from root-run `docker compose` (with a
`watchtower` container polling the docker socket) to rootless `podman
compose` run by a dedicated unprivileged system user, `notification-bot`.

### Why two systemd units

- `credfeto-notification-bot.service` is the actual container runner. It's
  a boot-time system unit (`WantedBy=multi-user.target`), not a `systemctl
  --user` unit — the ask was explicitly for a system service, not a user
  one. It runs `run-compose` as `User=notification-bot`. It still needs
  `loginctl enable-linger notification-bot` (done in `install`), because
  rootless podman's network backend (pasta/netavark) registers itself with
  a D-Bus session bus at `/run/user/<uid>/bus`, and that bus only exists if
  systemd-logind has started that user's manager — lingering makes it start
  at boot without a login, independent of the container-runner unit itself
  staying a plain system unit. `run-compose` itself computes and exports
  `XDG_RUNTIME_DIR=/run/user/$(id -u)` before calling podman, rather than
  the unit setting `Environment=XDG_RUNTIME_DIR=/run/user/%U` — on the real
  host, `%U` resolved to the *service manager's* UID (0), not
  `User=notification-bot`'s, pointing podman at `/run/user/0` and failing
  with `Failed to obtain podman configuration: lstat /run/user/0: no such
  file or directory`; `id -u` inside the already-`User=`-scoped process is
  reliable regardless of that specifier's behaviour on a given systemd
  version. An even earlier version of this unit used a bespoke
  `RuntimeDirectory=notification-bot` (`/run/notification-bot`) instead —
  that has no bus socket behind it and failed on the real host with
  `netavark: ... aardvark-dns failed to start` /
  `couldn't determine address of session bus`; see "Issues found and fixed
  during real-host testing" below for detail.
  It has no `ExecStop` and sets `KillMode=none`, so that `systemctl
  restart` doesn't send a cgroup-wide kill to the containers on the stop
  half before `run-compose` (`podman compose pull && podman compose up -d`)
  runs again on the start half — `up -d` only recreates containers whose
  image or config actually changed. **Confirmed on the real host**: a
  manual `systemctl restart` left all three containers' IDs and uptimes
  untouched (`podman compose ps` before/after showed the same container IDs
  with uptime still counting from before the restart, not reset).
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
  actually runs (a timer-triggered oneshot job's own cgroup is reaped by
  systemd on exit, which would kill anything it spawned directly).
- `reset` self-elevates the same way, and drops to `notification-bot` (via
  `runuser`) to explicitly stop/remove (`podman compose down`) and prune
  containers — that's fine because it's run interactively, not spawned by
  a systemd oneshot unit that will reap its own cgroup on exit.
- `credfeto-notification-bot.service` itself never runs as root. It adds
  `NoNewPrivileges=yes`, `ProtectSystem=strict` (with explicit
  `ReadWritePaths` for the podman storage dir and `/data/dispatcher`), and
  `PrivateTmp=yes` — confirmed on the real host not to block podman (image
  pull and container creation both succeeded under it). `ProtectHome=yes`
  was tried too but deliberately dropped: it also masks `/run/user/*`,
  which `XDG_RUNTIME_DIR` now depends on.

### Verified end-to-end on the real host

`install` (then `reset`, after fixes) was run repeatedly on the actual
`notification-bot` deployment host until `credfeto-notification-bot.service`
reached `active (exited)` with all three containers `Up ... (healthy)` in
`podman compose ps`, and a manual `systemctl restart
credfeto-notification-bot.service` left all three container IDs and
uptimes untouched (no recreation, no gap). Registry access/pulling, the
`notification-bot` user/subuid/subgid setup, `runuser`'s `HOME=`/
`XDG_RUNTIME_DIR=` overrides, and `ProtectSystem=strict`/`PrivateTmp=yes`
were all exercised along the way with no issues.

### Issues found and fixed during real-host testing

Four distinct failures surfaced only by actually running `install` on the
target host and reading `journalctl -xeu credfeto-notification-bot.service`
after each attempt — none were visible from `podman compose config` or
testing against a public image elsewhere. In the order hit:

1. **No D-Bus session bus for `notification-bot`.** Rootless podman's
   network backend (pasta + netavark/aardvark-dns) needs one, and nothing
   provisions one for a system-unit-only, no-login account. Fixed by having
   `install` run `loginctl enable-linger notification-bot`, and pointing
   `XDG_RUNTIME_DIR` at the real `/run/user/<uid>` that provisions —
   replacing an earlier bespoke `RuntimeDirectory=notification-bot`
   (`/run/notification-bot`) which had no bus socket behind it. Also
   required dropping `ProtectHome=yes` from the unit, since it independently
   masks `/run/user/*` too.
2. **The unit's `Environment=XDG_RUNTIME_DIR=/run/user/%U` resolved to UID
   0**, not `notification-bot`'s — `%U` is not reliably "UID of this unit's
   `User=`" on every systemd version. Fixed by dropping that line and having
   `run-compose` compute `XDG_RUNTIME_DIR=/run/user/$(id -u)` itself at
   runtime, which reflects whoever the process is actually running as
   regardless of specifier behaviour. `install`/`reset`'s own `runuser`
   calls were never affected — they already computed the UID via `id -u` in
   shell, not a systemd specifier.
3. **`Permission denied: OCI permission denied` creating a
   `libpod-<id>.scope`.** With lingering enabled, podman defaults to
   `--cgroup-manager=systemd` and asks `notification-bot`'s own `systemd
   --user` instance (over its D-Bus session bus) to create a transient
   scope per container — but that request comes from
   `credfeto-notification-bot.service`, a *system* unit reaching into that
   user session from outside it, and is rejected. Fixed by having `install`
   write `cgroup_manager = "cgroupfs"` to
   `~notification-bot/.config/containers/containers.conf`, so podman never
   needs that cross-boundary request — trading away per-container
   `systemd-cgtop`/`systemctl` visibility for reliability in this specific
   topology.
4. **A pre-migration `credfeto-notification-bot.timer` kept firing against
   the new `credfeto-notification-bot.service`.** The old design had a
   timer/service pair of that same name doing a different job (the root-run
   update); systemd timers default to targeting the identically-named
   `.service` with no `Unit=` override, so upgrading in place left the old
   timer firing the new container-runner service every 5 minutes. `install`
   now detects and removes that stale timer file before installing the new
   units.

Separately, `reset` originally called `podman compose rm -f`, which this
podman-compose version (1.6.0) doesn't support (`invalid choice: 'rm'`,
no such subcommand) — fixed to use `podman compose down` instead, which
covers stop+remove in one step and is actually supported.

### Remaining caveats

- **Subuid/subgid range.** `install` allocates `200000-265535` to
  `notification-bot` if it doesn't already have a range in `/etc/subuid` /
  `/etc/subgid`. If that range collides with another user on the host,
  `useradd`/`usermod` will error — pick a different range by hand and rerun.
- **Container hardening not applied blind.** `security_opt:
  no-new-privileges:true` was added to all three services. `read_only:
  true` and `cap_drop: [ALL]` were deliberately *not* added, since verifying
  them would have meant more rounds of real-host trial and error and either
  could silently break the app (e.g. if it writes temp files). Worth doing
  as a follow-up now the base deployment is confirmed working.
- **`dispatcher-data` volume stays unmounted.** The compose file declares
  `dispatcher-data` as an external volume but no service actually mounts
  it — that was already true before this migration and is left as-is.
- **Watchtower removed.** Automatic redeploy on a bare `:latest` image push
  (independent of this repo changing) is now handled by `update`
  unconditionally restarting `credfeto-notification-bot.service` every
  5-minute tick — `podman compose up -d` only recreates containers whose
  image actually changed — rather than a container polling the
  docker/podman socket every 60s.
