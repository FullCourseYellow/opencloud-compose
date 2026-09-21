# Full Course Yellow OpenCloud Deployment

FCY-specific deployment layer for the upstream
[`opencloud-eu/opencloud-compose`](https://github.com/opencloud-eu/opencloud-compose)
repository. Upstream files remain unchanged so updates can be merged with a
small conflict surface.

## Components

- OpenCloud 8.0.1 rolling, pinned for compatibility with the current Yjs module
- Euro Office document editing through OpenCloud's WOPI collaboration service
- Yjs collaborative editing
- Separate FCY Keycloak issuers with just-in-time account provisioning
- Private OpenLDAP storage for provisioned OpenCloud users and groups
- Velocitas Imperium light and dark Web UI themes
- Caddy reverse proxy and HarborDNS integration on the shared `caddy_net`
- Local PosixFS storage by default, with optional S3-compatible storage

Euro Office is an early-stage, community-supported OpenCloud integration. Test
document creation, editing, and save-back behavior before using it for critical
data.

## Environments

Development and production run simultaneously on the same Docker host as
independent Komodo Stacks. Their Git revisions, Compose resources, identity
providers, secrets, and persistent data remain separate.

| Setting | Development | Production |
| --- | --- | --- |
| Git branch | `develop` | `prod` |
| Komodo Stack | `opencloud-dev` | `opencloud-prod` |
| Compose project | `opencloud-dev` | `opencloud-prod` |
| OpenCloud | `https://dev-cloud.fullcourseyellow.com` | `https://cloud.fullcourseyellow.com` |
| Euro Office | `https://dev-office.fullcourseyellow.com` | `https://office.fullcourseyellow.com` |
| Keycloak | `https://dev-id.fullcourseyellow.com` | `https://id.fullcourseyellow.com` |
| Deployment trigger | Manual | Manual after promotion |

The `main` branch mirrors upstream, `develop` is the development integration
branch, and `prod` contains only changes promoted after development testing.
Do not commit environment secrets or persistent data to any branch.

## Prerequisites

- Docker Engine and Docker Compose v2
- Komodo with a Periphery agent connected to the Docker host
- The FCY Caddy deployment attached to the external `caddy_net` network
- HarborDNS available to process the service labels, or equivalent DNS records
- Administrative access to both FCY Keycloak environments

Create the shared external network once if it does not already exist:

```bash
docker network create caddy_net
```

Only OpenCloud and Euro Office join `caddy_net`. Each Stack also receives its
own project-scoped `opencloud-net`, keeping LDAP and Yjs private and isolated.

## Keycloak

Configure the four public OIDC clients and OpenCloud role claim independently
in both Keycloak environments before starting either Stack. See
[keycloak.md](keycloak.md) for the exact realm, web, desktop, Android, and iOS
settings.

The issuers are:

```text
Development: https://dev-id.fullcourseyellow.com/realms/fullcourseyellow
Production:  https://id.fullcourseyellow.com/realms/fullcourseyellow
```

Do not change an issuer after users have been provisioned. The issuer and OIDC
`sub` claim form each user's external identity.

## Web Theme

The FCY Compose overlay mounts the Velocitas Imperium theme from
`fcy/theme/velocitas-imperium` into OpenCloud. The Web UI follows the user's
saved light or dark preference and otherwise uses the operating-system color
scheme.

The theme uses the VI logos and the same warm light and charcoal dark palettes
as the Keycloak theme. The racing hero appears on OpenCloud's plain shell,
including password-protected public links, logout, and access-denied pages.
Authentication itself redirects to the external Keycloak issuer, so Keycloak
continues to own the interactive login screen.

Theme files are part of the deployment and do not live in the persistent
OpenCloud data volume. Restart the OpenCloud service after changing them.

## Komodo Stacks

Create two Git-backed Stack resources using repository
`FullCourseYellow/opencloud-compose`. Configure these fields explicitly:

| Komodo field | `opencloud-dev` | `opencloud-prod` |
| --- | --- | --- |
| Branch | `develop` | `prod` |
| Project name | `opencloud-dev` | `opencloud-prod` |
| Auto update | Off | Off |
| Webhook | Not configured | Not configured |

Use the same Compose files, in this order, for both Stacks:

```text
docker-compose.yml
yjs/yjs.yml
weboffice/euro-office.yml
idm/external-idp.yml
fcy/compose.caddy.yaml
```

If S3 is selected before the first deployment, insert
`storage/decomposeds3.yml` before `fcy/compose.caddy.yaml`.

Komodo writes each Stack's Environment configuration to an env file and passes
it to Docker Compose. Start from [`.env.example`](.env.example), but omit
`COMPOSE_FILE` and `COMPOSE_PATH_SEPARATOR` because the Stack file list above
already selects the modules. Then apply the endpoint values below.

| Variable | Development | Production |
| --- | --- | --- |
| `OC_DOMAIN` | `dev-cloud.fullcourseyellow.com` | `cloud.fullcourseyellow.com` |
| `EURO_OFFICE_DOMAIN` | `dev-office.fullcourseyellow.com` | `office.fullcourseyellow.com` |
| `IDP_DOMAIN` | `dev-id.fullcourseyellow.com` | `id.fullcourseyellow.com` |
| `IDP_ISSUER_URL` | `https://dev-id.fullcourseyellow.com/realms/fullcourseyellow` | `https://id.fullcourseyellow.com/realms/fullcourseyellow` |
| `IDP_ACCOUNT_URL` | `https://dev-id.fullcourseyellow.com/realms/fullcourseyellow/account` | `https://id.fullcourseyellow.com/realms/fullcourseyellow/account` |

Create distinct Komodo secrets for each environment and interpolate them into
the corresponding Stack Environment:

```env
LDAP_BIND_PASSWORD=[[OPENCLOUD_DEV_LDAP_BIND_PASSWORD]]
EURO_OFFICE_JWT_SECRET=[[OPENCLOUD_DEV_EURO_OFFICE_JWT_SECRET]]
```

Use production-specific secret keys in `opencloud-prod`. Generate suitable
values with:

```bash
openssl rand -hex 32
```

The FCY Compose layer requires all public domains, IDP URLs, and secrets. A
missing value therefore fails Compose interpolation before a deployment can
modify containers.

Keep `OC_DOCKER_TAG`, `YJS_DOCKER_TAG`, and `EURO_OFFICE_DOCKER_TAG` identical
between the two Stack configurations during promotion. `latest` is a moving
Euro Office tag; use a tested versioned tag or `latest@sha256:<digest>` for
production so the deployed image is the one tested in development.

## Local Validation

The checked-in example describes development. Copy it to the repository root
only for local Compose operations:

```bash
cp fcy/.env.example .env
docker compose config --quiet
```

Replace the example secret placeholders before starting containers. The root
`.env` file is ignored by Git.

## Storage

Choose the storage backend before users begin storing data. OpenCloud does not
provide an automatic migration path between PosixFS and S3.

### Named Volumes

Leaving `OC_CONFIG_DIR`, `OC_DATA_DIR`, `LDAP_CERTS_DIR`, and `LDAP_DATA_DIR`
empty uses Docker named volumes. The explicit Compose project names isolate the
resources:

```text
opencloud-dev_opencloud-config
opencloud-dev_opencloud-data
opencloud-dev_ldap-certs
opencloud-dev_ldap-data

opencloud-prod_opencloud-config
opencloud-prod_opencloud-data
opencloud-prod_ldap-certs
opencloud-prod_ldap-data
```

### Bind Mounts

Use different absolute host paths and ensure they are writable by the UID/GID
configured in `OC_CONTAINER_UID_GID`, which defaults to `1000:1000`:

```env
# Development
OC_CONFIG_DIR=/srv/opencloud/dev/config
OC_DATA_DIR=/srv/opencloud/dev/data
LDAP_CERTS_DIR=/srv/opencloud/dev/ldap-certs
LDAP_DATA_DIR=/srv/opencloud/dev/ldap-data

# Production uses /srv/opencloud/prod/... in its own Stack Environment.
```

### S3

S3 is an alternative primary storage backend, not an additional backup target.
Use separate buckets and credentials for development and production. Configure
all `DECOMPOSEDS3_*` values in each Stack Environment.

S3 deployments still require separate OpenCloud system-data and LDAP volumes
and backups in addition to bucket backups.

## Architecture

```text
Internet -> Caddy (caddy_net) -> opencloud-dev  -> dev LDAP and Yjs
                              -> dev Euro Office
                              -> opencloud-prod -> prod LDAP and Yjs
                              -> prod Euro Office

opencloud-dev  -> development Keycloak issuer (public HTTPS)
opencloud-prod -> production Keycloak issuer (public HTTPS)
```

Caddy and HarborDNS distinguish the two Stacks by their unique hostnames. The
Compose project names distinguish their containers, private networks, and named
volumes. Deploying code never promotes or copies persistent data.

## Promotion Workflow

1. Make changes on `develop`, optionally through a feature branch and pull
   request.
2. Manually deploy `opencloud-dev` in Komodo.
3. Complete the validation checklist below.
4. Open and merge a pull request from `develop` into protected branch `prod`.
5. Manually deploy `opencloud-prod` in Komodo.
6. Merge production-only hotfixes back into `develop`.

Protect `prod` against direct pushes and require pull requests. Keep automatic
Stack updates disabled so neither environment changes outside this workflow.

Use this checklist before promotion:

- OIDC login and first-login account provisioning
- Light and dark theme switching on desktop and mobile
- Password-protected public-link rendering with the themed hero
- One login for every mapped OpenCloud role
- WebFinger desktop and mobile discovery
- File upload, download, sharing, and restart persistence
- Document creation, editing, and save-back through Euro Office
- Concurrent editing through Yjs
- Logs for OpenCloud, Euro Office, LDAP, and Yjs

## Operations

Inspect a Stack from Komodo or use its explicit project name on the host:

```bash
docker compose --project-name opencloud-dev ps
docker compose --project-name opencloud-dev logs -f opencloud euro-office ldap-server yjs
```

Back up OpenCloud and LDAP state separately for each environment. Before an
upgrade, back up production state and account for storage schema migrations;
reverting a Git revision does not reverse a persistent-data migration.

## Branch Management

The `develop` and `prod` branches were initialized from the same tested FCY
deployment baseline. Configure and test both Komodo Stacks before changing the
GitHub default branch to `develop`. The legacy `fcy-deployment` branch can be
retired after the default branch, Komodo, and any other integrations no longer
reference it.

## Upstream Updates

Update the upstream mirror, then bring it into development. Production receives
the update only through the normal promotion workflow.

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
git switch develop
git merge main
```

After resolving changes, validate both storage variants:

```bash
docker compose --env-file fcy/.env.example --project-name opencloud-dev config --quiet
COMPOSE_FILE=docker-compose.yml:yjs/yjs.yml:weboffice/euro-office.yml:idm/external-idp.yml:storage/decomposeds3.yml:fcy/compose.caddy.yaml docker compose --env-file fcy/.env.example --project-name opencloud-dev config --quiet
```
