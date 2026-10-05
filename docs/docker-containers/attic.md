# Attic

Insurance asks what was in the house. The warranty claim needs the receipt. The spare HDMI cable is in a box, somewhere. Attic is a home inventory: every thing you own, where it lives, what it cost, and the paperwork that goes with it.

[Attic](https://getattic.dev/) is a self-hosted home inventory application.

1. **Nested locations** — house → room → shelf → box
2. **Categories with custom attributes**, plus condition tracking
3. **Warranties, receipts, manuals and photos** attached to each item
4. **Shared collections** for a household
5. **Metadata imports** from Google Books, TMDB, BoardGameGeek and IGDB — scan in a bookshelf or a game collection instead of typing it
6. **Local accounts or OIDC/SSO**, REST API, local or S3 storage

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - `openssl` for secrets

## 1. Create the directory

```bash
sudo mkdir -p /opt/docker/attic && cd /opt/docker/attic
mkdir -p ./data/{uploads,postgres}
```

## 2. The `.env` file

```bash
cat > .env <<EOF
POSTGRES_USER=attic
POSTGRES_PASSWORD=$(openssl rand -hex 24)
POSTGRES_DB=attic
ATTIC_SESSION_SECRET=$(openssl rand -base64 48)
ATTIC_BASE_URL=http://192.168.1.20:8095
ATTIC_ADMIN_EMAIL=you@example.com
ATTIC_ADMIN_PASSWORD=change-me
EOF
chmod 600 .env
```

**Set `ATTIC_ADMIN_EMAIL` and `ATTIC_ADMIN_PASSWORD` before the first start.** If they are missing, Attic creates the admin as `admin` / `admin`.

## 3. The Compose file

Upstream's compose also starts Keycloak. Leave it out unless you want SSO — local accounts work without it:

```yaml
services:
  attic:
    image: ghcr.io/lmmendes/attic:latest
    container_name: attic
    restart: unless-stopped
    # Starts as root to fix ownership, then drops to ATTIC_PUID/PGID
    user: "0:0"
    ports:
      - 8095:8080
    environment:
      ATTIC_PORT: 8080
      ATTIC_DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@attic-db:5432/${POSTGRES_DB}?sslmode=disable
      ATTIC_BASE_URL: ${ATTIC_BASE_URL}
      ATTIC_CORS_ORIGINS: ${ATTIC_BASE_URL}
      ATTIC_SESSION_SECRET: ${ATTIC_SESSION_SECRET}
      ATTIC_ADMIN_EMAIL: ${ATTIC_ADMIN_EMAIL}
      ATTIC_ADMIN_PASSWORD: ${ATTIC_ADMIN_PASSWORD}
      ATTIC_LOCAL_STORAGE_PATH: /data/uploads
      ATTIC_PUID: 1000
      ATTIC_PGID: 1000
    volumes:
      - ./data/uploads:/data/uploads
    depends_on:
      attic-db:
        condition: service_healthy

  attic-db:
    image: postgres:16-alpine
    container_name: attic-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
```

Notes:

1. **Port 8095 on the host.** Attic listens on 8080 inside, which [qBittorrent](qbittorrent.md) already has on the host.
2. **`ATTIC_BASE_URL` and `ATTIC_CORS_ORIGINS`** must match the address in your browser. The CORS default is `http://localhost:8080`, which is wrong for everything except local development.
3. **Migrations run automatically** on startup. No manual step.
4. **Attachments** go to `./data/uploads` unless you configure S3.

```bash
docker compose up -d
docker compose logs -f attic
```

## 4. Log in

Browse to `http://<host-ip>:8095` and log in with the admin email and password from `.env`. Change the password, then remove `ATTIC_ADMIN_PASSWORD` from `.env` — it has done its job.

## 5. Build the location tree first

Locations are the skeleton everything hangs off. Create rooms, then the shelves and boxes inside them, *before* adding items. Moving 300 items between locations later is tedious; creating a shelf is not.

## 6. Import plugins

Optional API keys turn typing into scanning:

| Plugin | Variable | Note |
|---|---|---|
| Google Books | `ATTIC_GOOGLE_BOOKS_API_KEY` | Works without a key, but shares an anonymous quota |
| TMDB | `ATTIC_TMDB_API_KEY` | Films and TV |
| BoardGameGeek | `ATTIC_BGG_API_KEY` | |
| IGDB | `ATTIC_IGDB_CLIENT_ID` + `ATTIC_IGDB_CLIENT_SECRET` | Needs a Twitch developer application |

Add to `.env`, pass through in the Compose `environment`, then `docker compose up -d`.

## 7. Put it behind Traefik

```yaml
    environment:
      ATTIC_BASE_URL: https://attic.example.com
      ATTIC_CORS_ORIGINS: https://attic.example.com
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.attic.rule=Host(`attic.example.com`)"
      - "traefik.http.routers.attic.entrypoints=https"
      - "traefik.http.routers.attic.tls.certresolver=letsencrypt"
      - "traefik.http.services.attic.loadbalancer.server.port=8080"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Keep `default` in the list so Attic can still reach `attic-db`. Change the base URL and CORS origin at the same time.

An inventory with serial numbers, prices and receipts is a burglar's shopping list. LAN-only or behind [wg-easy](wg-easy.md) is the sensible default.

## Updating

```bash
cd /opt/docker/attic
docker compose pull
docker compose up -d
```

Migrations run on start, so back up first. Pin a version tag once you have real data in it.

## Backup

Two things: the database and the uploads.

```bash
cd /opt/docker/attic
docker compose exec -T attic-db pg_dump -U attic attic | gzip > /mnt/user/backups/attic-db-$(date +%F).sql.gz
sudo tar czf /mnt/user/backups/attic-uploads-$(date +%F).tar.gz data/uploads .env
```

Keep a copy somewhere other than the house. An inventory for insurance claims that burns down with the house is not much use.

## Troubleshooting

**Login works, then everything fails with CORS errors.** `ATTIC_CORS_ORIGINS` does not match the address in your browser.

**Logged in as `admin` / `admin`.** `ATTIC_ADMIN_*` was not set on first start. Change the password now.

**Forgot the admin password.**

```bash
docker compose exec attic /app/attic --reset-password --email you@example.com --new-password new-password
```

**Attic exits on start with a database error.** Postgres not ready or the password in `.env` changed after the database was created. The password is only applied when `./data/postgres` is first initialised.

**Uploads fail with permission denied.** `ATTIC_PUID`/`ATTIC_PGID` do not match the owner of `./data/uploads`.

## Where this sits in my lab

<!-- TODO: fill in from your setup -->
