# CompDesk

Family IT support arrives by text message, at dinner, with no detail. A small office is the same thing with more people. CompDesk is a helpdesk: requests become tickets, tickets go to the right department, and nothing gets lost in a chat thread.

[CompDesk](https://github.com/TahaHydra/CompDesk) is a self-hosted helpdesk for small organisations.

1. **Department-scoped routing** — tickets land in the right queue
2. **Four roles** enforced on the server: User, Agent, Department Admin, Super Admin
3. **Templates, custom fields, SLA policies, escalation, tags**
4. **Immutable timeline** on every ticket
5. **Private attachments**, optional ClamAV scanning
6. **SMTP notifications, signed webhooks**, local accounts or Microsoft Entra ID
7. **English and French** interface, custom branding

!!! warning "Public beta"

    CompDesk describes itself as public beta, not production-hardened software. Read the [known limitations](https://github.com/TahaHydra/CompDesk/blob/main/docs/PUBLIC_RELEASE_CHECKLIST.md) before putting anything you can't lose in it.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)

## 1. Get the Compose file from a release

Unlike most apps here, don't write the Compose file yourself. CompDesk's is unusually careful — a one-shot init container, read-only filesystems, dropped capabilities, bounded restarts — and the release asset is pinned to a matching image version.

```bash
sudo mkdir -p /opt/docker/compdesk && cd /opt/docker/compdesk
```

Download `docker-compose.yml` from the latest [release](https://github.com/TahaHydra/CompDesk/releases) into this directory. The one on `main` defaults to an unpublished placeholder version and fails on purpose.

## 2. Port and bind address

Two defaults to change in `.env`:

```bash
cat > .env <<EOF
APP_PORT=3010
APP_BIND_ADDRESS=0.0.0.0
EOF
```

1. **`APP_PORT`** — CompDesk publishes 3000, which [Grafana](grafana.md) and [Karakeep](karakeep.md) already use.
2. **`APP_BIND_ADDRESS`** — the default is `127.0.0.1`, so out of the box it is only reachable from the Docker host itself. `0.0.0.0` opens it to the LAN. Leave the default if a reverse proxy on the same host is the only way in.

## 3. Start it

```bash
docker compose up -d
docker compose ps
```

Three services: `config-init` runs once and exits (that is expected), then `db` (PostgreSQL 16) and `compdesk`. The database password is generated for you and stored in the `config` volume — there is no password to put in `.env`.

## 4. First-run setup

The setup wizard needs a one-time token that is only printed to the logs:

```bash
docker compose logs --tail=50 compdesk
```

Browse to `http://<host-ip>:3010/setup`, paste the token, and create the first Super Admin. **The token expires after 30 minutes** — if you get distracted, restart the container for a new one.

When setup finishes, the same container switches to production on the same port. No second Compose command.

## 5. Departments, then people

Create departments before inviting anyone. Every ticket and every agent is scoped to one, and the queues only make sense once they exist. For a household, "IT" and "Everything else" is plenty.

Then invite agents into their departments and set roles. Department Admins manage their own department; keep Super Admin to yourself.

## 6. Email

Set up SMTP in the admin settings so people are notified when tickets change. A helpdesk nobody gets notifications from is a list nobody reads.

CompDesk hands mail to your relay; whether it reaches the inbox is the relay's problem. Use a proper transactional provider, not your home IP.

## 7. Behind a reverse proxy

Add to `.env`:

```bash
SETUP_PUBLIC_ORIGIN=https://help.example.com
SETUP_TRUST_PROXY=true
```

Then route `help.example.com` to the `compdesk` container's port 3000 — Traefik labels as on any other page, or a proxy host in [NPM](nginx-proxy-manager.md). Without `SETUP_PUBLIC_ORIGIN` the setup wizard rejects requests from the proxied origin.

## Updating

Download the newer release's `docker-compose.yml`, replacing the old one, then:

```bash
cd /opt/docker/compdesk
docker compose pull
docker compose up -d
```

Back up first. It is a beta; schema changes between releases are expected.

## Backup

Everything lives in named volumes: `pgdata`, `config` (which holds the generated database secret), `uploads` and `attachments`.

```bash
cd /opt/docker/compdesk
docker compose exec -T db pg_dump -U compdesk compdesk | gzip > /mnt/user/backups/compdesk-db-$(date +%F).sql.gz
for v in config uploads attachments; do
  docker run --rm -v compdesk_$v:/v:ro -v /mnt/user/backups:/b alpine \
    tar czf /b/compdesk-$v-$(date +%F).tar.gz -C /v .
done
```

**Back up `config` with the database.** It holds the database password; a database dump without it is harder to restore than it needs to be.

## Troubleshooting

**`compdesk` keeps exiting, then stops restarting.** Deliberate — it gives up after five attempts so a broken install is visible. `docker compose logs compdesk` shows the reason; fix it and `docker compose up -d`.

**`config-init` shows "Exited (0)".** Normal. It is a one-shot.

**Image pull fails for version `0.0.0-local`.** You used the Compose file from `main`, not the release asset.

**Setup token rejected.** Expired (30 minutes) or copied with whitespace. Restart and fetch a new one.

**Connection refused from another machine.** `APP_BIND_ADDRESS` is still `127.0.0.1`.

**Setup page errors behind the proxy.** `SETUP_PUBLIC_ORIGIN` / `SETUP_TRUST_PROXY` not set.

**Want to use an existing PostgreSQL server.** There is a `docker-compose.external-db.yml`, but upstream marks it unvalidated. Use the bundled database.

## Where this sits in my lab

<!-- TODO: fill in from your setup -->
