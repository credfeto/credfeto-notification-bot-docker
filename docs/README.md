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
  `couldn't determine address of session bus`; see "Known gaps" for detail.
  It has no `ExecStop` and sets `KillMode=none`, so that `systemctl
  restart` doesn't send a cgroup-wide kill to the containers on the stop
  half before `run-compose` (`podman compose pull && podman compose up -d`)
  runs again on the start half — `up -d` only recreates containers whose
  image or config actually changed. **This particular part is still
  unverified** — see "Known gaps" below.
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
  `ReadWritePaths` for the podman storage dir and `/data/dispatcher`), and
  `PrivateTmp=yes` — confirmed on the real host not to block podman (image
  pull and container creation both succeeded under it). `ProtectHome=yes`
  was tried too but deliberately dropped: it also masks `/run/user/*`,
  which `XDG_RUNTIME_DIR` now depends on.

### Known gaps / follow-ups for whoever runs this on the real host

- **[RESOLVED on first real-host `install` run] Container networking failed
  without a D-Bus session bus.** `./install` ran successfully through user
  creation, subuid/subgid, permissions, and podman volume registration, but
  `credfeto-notification-bot.service` itself failed to start. `journalctl
  -xeu credfeto-notification-bot.service` showed image pull and container
  *creation* succeeding, then: `failed to move the rootless netns pasta
  process to the systemd user.slice: dbus: couldn't determine address of
  session bus`, followed by `aardvark-dns failed to start` and the
  container failing to start. Root cause: rootless podman's default network
  backend (pasta + netavark/aardvark-dns) needs a D-Bus session bus, which
  only exists via `systemd --user` for that account — and nothing
  provisioned one for a system-unit-only, no-login `notification-bot`.
  Fixed by having `install` run `loginctl enable-linger notification-bot`
  (starts that user's manager, and its bus, at boot with no login required)
  and pointing `XDG_RUNTIME_DIR` at the real `/run/user/<uid>` it manages,
  replacing the bespoke `RuntimeDirectory=notification-bot` from the first
  version of this unit, which had no bus socket behind it. Also had to drop
  `ProtectHome=yes` from the unit, since it independently masks
  `/run/user/*` too.
- **[RESOLVED, found on the real host] The systemd unit's
  `Environment=XDG_RUNTIME_DIR=/run/user/%U` resolved to the wrong UID.**
  After the linger fix above, the unit still failed:
  `Failed to obtain podman configuration: lstat /run/user/0: no such file or
  directory` — `%U` expanded to `0` (the service manager's UID), not
  `notification-bot`'s. Fixed by dropping the `Environment=` line entirely
  and having `run-compose` compute `XDG_RUNTIME_DIR=/run/user/$(id -u)`
  itself at runtime, which is correct regardless of that specifier's
  behaviour on a given systemd version. `install`/`reset`'s own `runuser`
  calls were never affected — they already computed the UID via
  `id -u notification-bot` in shell, not via a systemd specifier.
- **[RESOLVED, found on the real host] Containers failed to start with
  `Permission denied: OCI permission denied` creating a
  `libpod-<id>.scope`.** After the two fixes above, images pulled and
  containers were *created*, but `podman compose up -d` failed for all
  three with `runc create failed: ... unable to start unit
  "libpod-<id>.scope" ... Permission denied: OCI permission denied`. With
  lingering enabled, podman defaults to `--cgroup-manager=systemd` and asks
  `notification-bot`'s own `systemd --user` instance (over its D-Bus
  session bus) to create a transient scope unit per container — but that
  request comes from `credfeto-notification-bot.service`, a *system* unit
  reaching into that user session from outside it, and gets rejected.
  Fixed by having `install` write `cgroup_manager = "cgroupfs"` to
  `~notification-bot/.config/containers/containers.conf`, so podman never
  needs to talk to the user session's systemd for cgroups at all — trading
  away per-container `systemd-cgtop`/`systemctl` visibility for reliability
  in this specific system-unit-without-a-real-login-session topology.
- **[RESOLVED, found on the real host] A pre-migration
  `credfeto-notification-bot.timer` kept firing against the new
  `credfeto-notification-bot.service`.** The old design had a timer/service
  pair of that same name doing a completely different job (the root-run
  update). Systemd timers default to targeting the identically-named
  `.service` when no `Unit=` override is given, so upgrading in place left
  the old timer firing the new container-runner service every 5 minutes.
  `install` now detects and removes `/etc/systemd/system/credfeto-notification-bot.timer`
  if present, before installing the new units.
- **All three of the above were only found by running `install` on the
  real host and reading `journalctl -xeu credfeto-notification-bot.service`
  after each attempt** — confirm a fresh `install` run now gets all three
  containers running (`podman compose ps` as `notification-bot`, or
  `systemctl status credfeto-notification-bot.service` for a clean
  "active").
- **`systemctl restart credfeto-notification-bot.service` may still kill
  the containers — still unverified.** Rootless podman is daemonless:
  `conmon`/container processes normally live inside the launching unit's
  cgroup, and systemd's default `KillMode=control-group` kills the whole
  cgroup on the stop half of a restart — the absence of an `ExecStop` line
  doesn't prevent that, only `KillMode=none` (set on this unit) does. The
  first real-host run never got as far as a restart cycle (it failed on
  first start, see above), so this is still just plausible-but-unverified.
  **Before trusting `update`'s behaviour** (it restarts this unit on every
  5-minute tick), watch `podman ps`/`podman compose ps` across a manual
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
- **[Confirmed on the real host] `ProtectSystem=strict`/`PrivateTmp=yes`
  don't block podman.** The first `install` run got as far as `podman`
  pulling an image and creating a container under these directives before
  failing on the networking issue above — so unlike `ProtectHome=yes`
  (dropped, see above), these two are proven compatible, not just
  plausible.
- **[Confirmed on the real host] `HOME` handling in the `runuser` calls
  works.** `install` and `reset` invoke podman as `notification-bot` via
  `runuser -u notification-bot -- env HOME=/var/lib/notification-bot
  XDG_RUNTIME_DIR=... ...`, explicitly setting `HOME` because `runuser`
  without `--login` otherwise leaves the caller's `HOME` (`/root`, since
  these scripts self-elevate) in place. The real-host `install` run's
  podman volume registration step (which goes through this exact path)
  succeeded, confirming the override works as intended.
- **[Confirmed on the real host] Registry access, pulling, and rootless
  container creation all work**, up to the networking issue above.
  `docker-registry.markridgwell.com` pulled all three images and podman
  created at least one container successfully before the D-Bus/pasta
  failure stopped the rest. subuid/subgid and storage permissions are not
  implicated by any of the errors seen.
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
