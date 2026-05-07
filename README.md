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

## Required local files

- `notification/appsettings-local.json`
- `dispatcher/appsettings-local.json`
- `proxy/credentials.json` (copy from `proxy/credentials.example.json`)

## Update flow

Run the `update` script to pull the repo and redeploy containers.

During `update`:

- `certs/dispatcher.pfx` is created automatically if it does not exist.
- The certificate is self-signed and generated with `openssl`.
- The compose mount for `dispatcher-bot` maps:
  - `./certs/dispatcher.pfx:/usr/src/app/server.pfx:ro`

If you want to regenerate the certificate, delete `certs/dispatcher.pfx` and run `./update` again.
