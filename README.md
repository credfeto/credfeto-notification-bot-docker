# credfeto-notification-bot-docker

Podman compose deployment for the notification bot, running as an
unprivileged rootless systemd service.

## Documentation

- Additional notes live in `docs/`.

## Architecture

- The containers are run by `credfeto-notification-bot.service`, a system
  systemd unit that starts at boot (`WantedBy=multi-user.target`) as the
  unprivileged `notification-bot` user via rootless podman. It has no
  standing privileges beyond read access to this checkout and write access
  to the `dispatcher-data` volume's backing directory. Its `ExecStart` is
  the `run-compose` script (`podman compose pull && podman compose up -d`).
- `credfeto-notification-bot-update.timer` fires every 5 minutes and runs
  the root-run `update` script, which pulls this repo, validates/refreshes
  local config, and always restarts `credfeto-notification-bot.service`
  (which re-runs `run-compose`, i.e. pull + idempotent `up -d`, so unchanged
  containers aren't disrupted) — this also picks up new `:latest` image
  pushes on every tick. Every podman invocation goes through that one
  service unit rather than being run directly from `update`, so it always
  runs under `KillMode=none` and isn't at risk of being killed when the
  timer-triggered job's own cgroup is reaped.
- `install` performs one-time, root-only setup: creates the `notification-bot`
  system user with a subuid/subgid range, grants it read access to this
  checkout and ownership of the data directory, registers the external
  `dispatcher-data` podman volume, opens the required firewall port, and
  installs/enables the systemd units.
- `reset` tears down the running containers and re-runs `install`.

## One-time host setup

This checkout must live at `/opt/credfeto-notification-bot-docker` — the
systemd units hardcode that path. Clone (or move) it there, then:

```bash
./install
```

`install` requires `podman` (with the `podman compose` provider) already
installed; it will not install it for you.

## Operations

Use these scripts from the repository root:

```bash
./install
./update
./reset
```

## Required local files

- `notification/appsettings-local.json`
- `dispatcher/appsettings-local.json`
- `proxy/credentials.json` (copy from `proxy/credentials.example.json`, then
  `chmod 644` — rootless podman's UID remapping means `github-api-proxy`'s
  containerized process needs `other`-read to see it; see `docs/README.md`)
- `.env`

## Update flow

The `credfeto-notification-bot-update.timer` runs `update` every 5 minutes;
it can also be run manually.

During `update`:

- `certs/dispatcher.pfx` is created automatically if it does not exist.
- If `certs/dispatcher.pfx` exists as an empty directory, `update` removes it
  and recreates the file.
- If `certs/dispatcher.pfx` exists as a non-empty directory, `update` fails
  with an explicit error.
- The certificate is self-signed and generated with `openssl`.
- The compose mount for `dispatcher-bot` maps:
  - `./certs/dispatcher.pfx:/usr/src/app/server.pfx:ro`
- `credfeto-notification-bot.service` is restarted on every run. The log
  line distinguishes a restart triggered by a git/config change
  (`notification/appsettings-local.json`, `dispatcher/appsettings-local.json`,
  `proxy/credentials.json`, `.env`, `certs/dispatcher.pfx`) from a routine
  tick, but the action taken is the same either way.

If you want to regenerate the certificate, delete `certs/dispatcher.pfx` and
run `./update` again.
