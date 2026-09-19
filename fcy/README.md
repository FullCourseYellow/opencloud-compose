# Full Course Yellow OpenCloud Deployment

FCY-specific deployment layer for the upstream
[`opencloud-eu/opencloud-compose`](https://github.com/opencloud-eu/opencloud-compose)
repository. Upstream files remain unchanged so updates can be merged with a
small conflict surface.

## Components

- OpenCloud 8.0.1 rolling, pinned for compatibility with the current Yjs module
- Euro Office document editing through OpenCloud's WOPI collaboration service
- Yjs collaborative editing
- Existing FCY Keycloak realm with just-in-time account provisioning
- Private OpenLDAP storage for provisioned OpenCloud users and groups
- Caddy reverse proxy and HarborDNS integration on `caddy_net`
- Local PosixFS storage by default, with optional S3-compatible storage

Euro Office is an early-stage, community-supported OpenCloud integration. Test
document creation, editing, and save-back behavior before using it for critical
data.

## Prerequisites

- Docker Engine and Docker Compose v2
- The FCY Caddy deployment attached to the external `caddy_net` network
- HarborDNS available to process the service labels, or equivalent DNS records
- Administrative access to the `fullcourseyellow` Keycloak realm

Create the external network once if it does not already exist:

```bash
docker network create caddy_net
```

## Keycloak

Configure the four public OIDC clients and OpenCloud role claim before starting
the stack. See [keycloak.md](keycloak.md) for the exact web, desktop, Android,
and iOS settings.

The issuer is fixed to:

```text
https://dev-id.fullcourseyellow.com/realms/fullcourseyellow
```

Do not change the issuer after users have been provisioned. The issuer and the
OIDC `sub` claim form each user's external identity.

## Configure

Create the root environment file:

```bash
cp fcy/.env.example .env
```

Replace at least these values:

```env
EURO_OFFICE_JWT_SECRET=replace-with-at-least-32-random-characters
LDAP_BIND_PASSWORD=replace-with-a-strong-random-password
```

Generate suitable values with:

```bash
openssl rand -hex 32
```

The default public endpoints are:

| Service | URL |
| --- | --- |
| OpenCloud | `https://dev-cloud.fullcourseyellow.com` |
| Euro Office | `https://dev-office.fullcourseyellow.com` |
| Keycloak | `https://dev-id.fullcourseyellow.com` |

## Deploy With Local Storage

The default Compose selection stores user files on OpenCloud's PosixFS driver
inside the `opencloud-data` volume:

```bash
docker compose config --quiet
docker compose up -d
docker compose ps
```

For bind-mounted storage, set `OC_CONFIG_DIR` and `OC_DATA_DIR` and ensure both
paths are writable by UID/GID `1000:1000`.

## Deploy With S3

S3 is an alternative primary storage backend, not an additional backup target.
Before the first deployment, append the upstream S3 module to `COMPOSE_FILE`:

```env
COMPOSE_FILE=docker-compose.yml:yjs/yjs.yml:weboffice/euro-office.yml:idm/external-idp.yml:storage/decomposeds3.yml:fcy/compose.caddy.yaml
```

Uncomment and configure all `DECOMPOSEDS3_*` variables in `.env`, then validate
and start normally:

```bash
docker compose config --quiet
docker compose up -d
```

OpenCloud does not provide an automatic migration path between PosixFS and S3.
Choose the storage backend before users begin storing data.

## Architecture

```text
Internet -> Caddy (caddy_net) -> OpenCloud (:9200)
                              -> Euro Office (:80)

OpenCloud network:
OpenCloud -> LDAP (:1636)
OpenCloud -> Yjs (:1234)
OpenCloud <-> Euro Office (WOPI)
OpenCloud -> FCY Keycloak (public HTTPS)
```

Only OpenCloud and Euro Office join `caddy_net`. LDAP and Yjs remain private to
the upstream `opencloud-net` network.

## Operations

Inspect service state and logs:

```bash
docker compose ps
docker compose logs -f opencloud euro-office ldap-server yjs
```

Back up both OpenCloud and LDAP state. With the default named volumes, the
important volumes are `opencloud-config`, `opencloud-data`, `ldap-certs`, and
`ldap-data`. S3 deployments still need the OpenCloud system-data and LDAP
volumes backed up in addition to the bucket.

Before upgrades, review OpenCloud release notes and test document editing,
OIDC login, WebFinger discovery, desktop synchronization, and mobile login.

## Upstream Updates

The `main` branch mirrors upstream. FCY changes live on `fcy-deployment`.

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
git switch fcy-deployment
git merge main
```

After resolving any upstream changes, validate both storage variants:

```bash
docker compose --env-file fcy/.env.example config --quiet
COMPOSE_FILE=docker-compose.yml:yjs/yjs.yml:weboffice/euro-office.yml:idm/external-idp.yml:storage/decomposeds3.yml:fcy/compose.caddy.yaml docker compose --env-file fcy/.env.example config --quiet
```
