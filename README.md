# credfeto-notification-bot-docker

Docker compose for notification bot.

## Documentation

- Additional notes live in `docs/`.

## Operations

Use these scripts from the repository root:

```bash
./install
./update
./reset
```

### One-time migration cleanup

If this host previously ran the rootless-podman deployment (see
`docs/README.md`), run `./cleanup-podman-migration --yes` once before
`./install` to remove the `notification-bot` system user, the podman-era
systemd units, and the extra firewall rules it opened.

## Required local files

- `notification/appsettings-local.json`
- `dispatcher/appsettings-local.json`
- `workflow-mcp/appsettings-local.json`
- `proxy/credentials.json` (copy from `proxy/credentials.example.json`)

## Update flow

Run the `update` script to pull the repo and redeploy containers.

During `update`:

- `certs/dispatcher.pfx` is created automatically if it does not exist.
- If `certs/dispatcher.pfx` exists as an empty directory, `update` removes it
  and recreates the file.
- If `certs/dispatcher.pfx` exists as a non-empty directory, `update` fails
  with an explicit error.
- The certificate is self-signed and generated with `openssl`.
- The compose mount for `dispatcher-bot` maps:
  - `./certs/dispatcher.pfx:/usr/src/app/server.pfx:ro`
- `certs/workflow-mcp.pfx` is created automatically in the same way, with the
  same empty/non-empty directory handling.
- The compose mount for `workflow-mcp` maps:
  - `./certs/workflow-mcp.pfx:/usr/src/app/server.pfx:ro`
- `workflow-mcp` serves HTTPS on host port `8091`, mapped to container port
  `8081`. `update` opens TCP port `8091` in the firewall for private networks
  only.

If you want to regenerate a certificate, delete `certs/dispatcher.pfx` or
`certs/workflow-mcp.pfx` and run `./update` again.
