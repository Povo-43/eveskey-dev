# EvexAccount local build

This guide explains how to run Misskey locally on Windows 11 with Docker Desktop only, so you can test the EvexAccount authentication flow without installing Node.js on the host.

## What this setup runs

- Misskey web, backend, and migrations in Docker
- PostgreSQL in Docker
- Redis in Docker

## Prerequisites

- Windows 11
- Docker Desktop running
- Git installed
- PowerShell

## 1. Prepare the local config files

From the repository root, create local copies of the example config files:

```powershell
Copy-Item .\.config\docker_example.env .\.config\docker.env
Copy-Item .\.config\example.yml .\.config\default.yml
```

Edit [`.config/default.yml`](C:/Users/anill/Desktop/eveskey/.config/default.yml):

- Set `url` to the URL you will actually open in the browser.
- For local-only testing, `http://localhost:3000` is fine if your EvexAccount registration allows it.
- If EvexAccount requires HTTPS, set `url` to the public HTTPS origin that will reach this container.
- Set `db.host` to `db`.
- Set `redis.host` to `redis`.
- Leave the database port at `5432` and the Redis port at `6379`.

Edit [`.config/docker.env`](C:/Users/anill/Desktop/eveskey/.config/docker.env) and add the EvexAccount values that the backend reads from `process.env`:

```powershell
EVEXACCOUNT_ISSUER=https://account.evex.land
EVEXACCOUNT_CLIENT_ID=<your-client-id>
EVEXACCOUNT_CLIENT_SECRET=<your-client-secret>
```

If you use a custom issuer, also add these optional overrides:

```powershell
EVEXACCOUNT_AUTHORIZATION_ENDPOINT=https://...
EVEXACCOUNT_TOKEN_ENDPOINT=https://...
EVEXACCOUNT_USERINFO_ENDPOINT=https://...
```

## 2. Start the full stack

Use the Docker compose example that runs Misskey itself in Docker:

```powershell
docker compose -f compose_example.yml up -d --build
```

This starts:

- Misskey on `localhost:3000`
- PostgreSQL on the internal `db` service
- Redis on the internal `redis` service

The first start can take a while because the image is built from the repository and the database is migrated during container startup.

To watch the application boot:

```powershell
docker compose -f compose_example.yml logs -f web
```

## 3. Open Misskey

When the containers are ready, open:

- `http://localhost:3000`

If you are testing the full EvexAccount redirect path and the issuer needs a public HTTPS callback, expose the service through an HTTPS endpoint that points to the container and set `url` to that public origin before restarting the stack.

## 4. EvexAccount callback notes

- The local callback route is `/callback`.
- The EvexAccount app registration must allow that redirect URI.
- The backend reads `EVEXACCOUNT_ISSUER`, `EVEXACCOUNT_CLIENT_ID`, `EVEXACCOUNT_CLIENT_SECRET`, and the optional endpoint overrides from the container environment.
- If you change `.config/default.yml` or `.config/docker.env`, restart the stack with `docker compose -f compose_example.yml up -d --build`.

## Stop the stack

```powershell
docker compose -f compose_example.yml down
```

To remove the data as well, delete the `db`, `redis`, and `files` folders in the repository root after stopping the containers.

## Notes

- The Dockerfile already runs `migrateandstart`, so you do not need to run `pnpm` on the host.
- Because `compose_example.yml` now passes `.config/docker.env` into the web container, the EvexAccount settings you add there are available during startup.
