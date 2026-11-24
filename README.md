# Bring Me Food: Docker Compose deployment

One `docker-compose.yml` and one `.env` run every backend and frontend service from the container registry behind [Traefik](https://doc.traefik.io/traefik/). What actually runs is chosen by **one line in `.env`**: `COMPOSE_PROFILES`.

## The switches

```sh
COMPOSE_PROFILES=https-letsencrypt,infra
```

**TLS strategy: pick exactly one**

| Profile | Use it when | What you provide |
| --- | --- | --- |
| `https-letsencrypt` | Public server with real DNS | `ACME_EMAIL`, DNS records, ports 80 and 443 open |
| `https-custom` | Local development with trusted HTTPS (mkcert), or a certificate from your provider | `tls/cert.pem` and `tls/key.pem` covering every host |
| `http` | Local development without certificates, or TLS handled upstream | Nothing. Use `http://` and `ws://` in every URL |

**What to run: add any**

| Profile | Starts | Leave it out when |
| --- | --- | --- |
| `backend` | All backend services: HTTP APIs, WebSocket/MCP, gRPC and consumers | You only need the frontends or the infrastructure |
| `web` | The five frontend containers (nginx) | You run the frontends with `pnpm` |

**Infrastructure: add any**

| Profile | Starts | Leave it out when |
| --- | --- | --- |
| `infra` | All three of the following | You use Supabase, Redis Cloud, a stage environment, etc. |
| `postgres` | Local PostgreSQL only | The database is elsewhere |
| `redis` | Local Redis-compatible broker (Valkey) only | Redis is elsewhere |
| `opensearch` | Local OpenSearch only | Search is elsewhere |

**Other optional parts: add any**

| Profile | Adds | Skip it when |
| --- | --- | --- |
| `observability` | Local Grafana, Loki, Tempo and Prometheus with an OTLP endpoint | You send telemetry elsewhere, or nowhere |
| `backup` | Scheduled `pg_dump` of `POSTGRES_HOST` into `./backups` | You back up another way |

Common setups:

| Situation | `COMPOSE_PROFILES` |
| --- | --- |
| Production VM, everything local | `https-letsencrypt,backend,web,infra,backup` |
| Production VM, Supabase and cloud Redis | `https-letsencrypt,backend,web` |
| Laptop, everything local, trusted HTTPS | `https-custom,backend,web,infra` |
| Laptop, Supabase, no certificates | `http,backend,web` |
| Laptop, backend and local Redis, Postgres and OpenSearch on stage, frontends with pnpm | `http,backend,redis` (see [Running part of the stack](#running-part-of-the-stack)) |

If you forget the TLS profile, no proxy starts and nothing answers on ports 80 and 443. If you forget `backend` or `web`, those services don't start, and the infrastructure profiles start only what they name. Check with `docker compose ps`.

> **Upgrading an existing installation:** `backend` and `web` are new. Services of those tiers used to start whatever the profiles, so add `backend,web` to `COMPOSE_PROFILES` in your `.env` before the next `docker compose up -d`.

## What runs

| Tier | Services | Profile | Reachable from outside? |
| --- | --- | --- | --- |
| Backend HTTP | `auth`, `admin`, `restaurants`, `client`, `delivery`, `images`, `subscription`, `billing`, `rating` | `backend` | yes, on `API_HOST`, by path |
| Backend WebSocket / MCP | `client-ws`, `restaurants-ws`, `delivery-ws`, `restaurants-mcp`, `images-mcp` | `backend` | yes, on `API_HOST`, by path |
| Backend gRPC | `auth-grpc`, `client-grpc`, `restaurants-grpc`, `subscription-grpc`, `billing-grpc`, `notification-grpc`, `rating-grpc` | `backend` | no, internal mTLS only |
| Backend consumers | `address-`, `order-`, `restaurants-`, `menu-`, `auth-`, `notification-`, `subscription-`, `billing-`, `rating-consumer` | `backend` | no |
| Frontend | `auth-web`, `restaurants-web`, `admin-web`, `client-web`, `delivery-web` | `web` | yes, one host each |
| Proxy | one of `traefik-letsencrypt`, `traefik-custom`, `traefik-http` | the TLS profile | yes |
| Infra (optional) | `postgres`, `redis`, `opensearch` | `infra`, or `postgres` / `redis` / `opensearch` | no (Postgres on `127.0.0.1` only) |
| Optional extras | `otel-lgtm`, `postgres-backup` | `observability`, `backup` | no (Grafana on `127.0.0.1` only) |

Images are `${REGISTRY}/<service>:${BACKEND_VERSION}` for the backend and `${REGISTRY}/<name>-web:${FRONTEND_VERSION}` for the frontend, the same names `make build push` produces.

**Not included:** KurrentDB, LiveKit, the database migrator, and the external SaaS services (Stripe, SMTP, Firebase).

## Quick start

```sh
cp .env.example .env     # set COMPOSE_PROFILES, then fill in every CHANGE_ME; chmod 600 .env
docker compose config -q # validates; fails fast on missing required variables
docker compose pull
docker compose up -d
docker compose ps
```

Prerequisites:

- Docker Engine with Docker Compose v2 (`docker compose version`)
- Pull access to the registry (below), mTLS certificates in `./certs`, Firebase credentials in `./credentials`
- Database migrations applied before the apps start (apps are restarted automatically until the database answers)
- For `infra`: on Linux, `sudo sysctl -w vm.max_map_count=262144` (OpenSearch exits without it; persist it in `/etc/sysctl.d/`)

### Registry access

```sh
# an operator's own gcloud login:
gcloud auth configure-docker <REGION>-docker.pkg.dev

# or a service account key (recommended per customer: one key you can revoke):
cat key.json | docker login -u _json_key --password-stdin <REGION>-docker.pkg.dev
```

The account needs the *Artifact Registry Reader* role.

### Service-to-service mTLS certificates

The gRPC services read `./certs/{ca.crt,server.crt,server.key,client.crt,client.key}`. Generate them with `make gen-certs` in the backend repo and copy the `certs/` folder here. Use a **new set per installation**. The containers must be able to read the files (for example `chmod 644 certs/*`). Clients connect with `GRPC_SERVER_NAME=localhost`, which matches the SAN that `gen-certs` creates. These are separate from the browser-facing certificates below.

### Image storage credentials

Put the Firebase service-account file in `./credentials/` (mounted read-only into `images` and `images-mcp`). `FIREBASE_CREDENTIALS_PATH` is the path **inside** the container, e.g. `/credentials/firebase.json`.

## TLS strategies in detail

### `https-letsencrypt`

Create DNS records for `API_HOST` and the five app hosts (or one wildcard `*.<DOMAIN>`) pointing at the server, open ports 80 and 443, and set `ACME_EMAIL`. Certificates are issued on first request to each of the six hosts. While testing, set `ACME_CA_SERVER` to the staging URL in `.env.example` to avoid rate limits, then remove it, delete the `traefik-letsencrypt` volume and restart.

### `https-custom`

Place `tls/cert.pem` and `tls/key.pem` next to `docker-compose.yml`. One certificate must cover **every** host: `API_HOST` and the five app hosts. For local development, [mkcert](https://github.com/FiloSottile/mkcert) creates a certificate your browser trusts:

```sh
mkcert -install                       # once per machine: trusts the local CA
mkcert -cert-file tls/cert.pem -key-file tls/key.pem \
  localhost 127.0.0.1 ::1 \
  api.localhost auth.localhost restaurants.localhost \
  admin.localhost client.localhost delivery.localhost
```

List the hostnames from your `.env`. Firefox needs the `nss` tools installed for `mkcert -install` to register the CA. Never use the mkcert certificate or CA outside your machine. For a provider certificate (for example a Cloudflare origin certificate), put the PEM files in `tls/` under the same names.

### `http`

No certificates and no redirect. Browsers such as Brave and Chrome may still try to upgrade `http://` to `https://` on their own (see Troubleshooting); `https-custom` avoids that. Use `http://` and `ws://` in `.env` (`AUTH_URL`, `ALLOWED_ORIGINS`, every `VITE_*`). If login cookies are marked `Secure`, browsers may refuse them over plain HTTP; use `https-custom` instead.

For local development, Chrome, Firefox and `curl` resolve `*.localhost` to your machine by themselves. If your browser or OS doesn't, add the hostnames to `/etc/hosts`.

## Databases and other services

Use the local profiles (`infra` for all three, or `postgres`, `redis`, `opensearch` one by one) or external services, per service. You can mix them, for example local Redis with Postgres and OpenSearch on stage. Section 4 of `.env.example` shows both.

**Local (`infra`, or the single profile)**: set `POSTGRES_HOST=postgres`, `POSTGRES_HOST_READ=postgres`, `REDIS_ADDR=redis:6379`, `OPENSEARCH_URL=http://opensearch:9200`.

| Service | Image (override with) | Notes |
| --- | --- | --- |
| `postgres` | `postgres:17-alpine` (`POSTGRES_IMAGE`) | Serves reads and writes. Data checksums on, `max_connections` 300. On `127.0.0.1:5432`. |
| `redis` | `valkey/valkey:8-alpine` (`REDIS_IMAGE`) | Redis-compatible. It is the message broker, so AOF persistence is on and eviction is off. |
| `opensearch` | `opensearchproject/opensearch:2.19.1` (`OPENSEARCH_IMAGE`) | Single node, security plugin disabled, internal only. Heap `OPENSEARCH_HEAP`. |
| `otel-lgtm` | `grafana/otel-lgtm` (`LGTM_IMAGE`) | Set `OTEL_COLLECTOR_URL=otel-lgtm:4317`. Grafana on `127.0.0.1:3000`. Pin the image tag. |

The Postgres major version defines the data format: change it with a dump and restore, not by bumping the tag (Postgres 18 also expects a different data path). The standard image has only the contrib extensions; if your migrations need others, use an image that has them.

**External (Supabase and others)**: set the hosts and ports from the provider. The optional `POSTGRES_DB_*` and `POSTGRES_SCHEMA_PREFIX` variables are passed to every service that connects to the database; leave them empty when not needed. Total database connections are roughly `DB_MAX_OPEN_CONNS` times the number of containers, so scaling replicas multiplies them: keep that under your provider's limit.

**Starting parts separately**

```sh
docker compose --profile infra up -d postgres redis opensearch   # infra only
docker compose up -d                                              # the rest, once infra is healthy
docker compose --profile '*' down                                 # stop everything, whatever profiles are active
```

More ways to choose what runs are in [Running part of the stack](#running-part-of-the-stack).

`docker compose down` only removes services of the active profiles, hence the `'*'` form. Data lives in named volumes (`postgres-data`, `redis-data`, `opensearch-data`, `lgtm-data`); `down` keeps them and `down -v` deletes them.

**Backups.** With the `backup` profile, dumps go to `./backups` on the same machine, so also copy that folder off the host (rclone, rsync, object storage). Restore:

```sh
docker compose exec -T postgres \
  pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists < backups/<file>.dump
```

## Running part of the stack

Each tier has its own profile (`backend`, `web`, and `infra` or one of `postgres`, `redis`, `opensearch`), so one file covers everything local, backend only, or a few local pieces with the rest on another environment such as stage.

### Choosing what runs

```sh
# persistent: one line in .env
COMPOSE_PROFILES=http,backend,redis

# one command only (a value in your shell wins over .env)
COMPOSE_PROFILES=http,backend,postgres docker compose up -d

# by service name: naming a service starts it, whatever the profiles
docker compose up -d traefik-http redis auth auth-grpc

# what would the current selection start?
docker compose config --services
```

- Changing the selection does not stop what is already running. Run `docker compose --profile '*' down`, then start again.
- When you name services, only those start. Include the proxy (`traefik-http`, `traefik-custom` or `traefik-letsencrypt`) and any infrastructure they need. There is no `depends_on`.
- Compose reads the whole file whatever the profiles, so every variable in the `x-required` list must be set. Without `web`, use `FRONTEND_VERSION=unused`.

### Backend in containers, infrastructure elsewhere (for example stage)

Start only what you want locally and point the rest at the other environment. Example: backend and Redis here, Postgres and OpenSearch on stage.

```sh
COMPOSE_PROFILES=http,backend,redis

REDIS_ADDR=redis:6379
REDIS_PASSWORD=<local password>
REDIS_TLS=false

POSTGRES_HOST=<stage write host>
POSTGRES_HOST_READ=<stage read host>
POSTGRES_DB_PORT=<stage port>
OPENSEARCH_URL=<stage url>
```

`POSTGRES_USER`, `POSTGRES_PWD`, `POSTGRES_DB` and the optional `POSTGRES_DB_*` tuning also come from stage. Before you point local containers at a shared environment:

- **Keys:** `PERSONAL_INFORMATION_ENCRYPTION_KEY`, `JWT_KEY`, `JWE_KEY` and the other secrets must be stage's values, otherwise stage data cannot be decrypted and stage tokens are not accepted.
- **Third parties:** use test Stripe keys (`sk_test_...`) and a sandbox SMTP server. Local consumers act on whatever you trigger.
- **Version:** set `BACKEND_VERSION` to a version that matches the schema stage has been migrated to. Nothing here runs migrations.
- **Redis state:** the local broker is empty and separate from stage's. Anything held only in stage's Redis (streams, caches) does not exist locally.

### Frontends with pnpm

Leave `web` out and run each app's dev server instead. Traefik keeps serving the API and each app's `/<app>/login`, `/logout`, `/refresh` and `/auth` paths on port 80. On port 80 everything else under an app host answers `404`, because no web container runs: open the dev server's URL instead.

Keep the same `*.localhost` hostnames and only add a port. Cookies are same-site across ports, so login behaves as before, but CORS compares full origins, so the dev origins must be allowed. Pick one port per app, for example:

| App | Dev server URL |
| --- | --- |
| auth | `http://auth.localhost:5170` |
| restaurants | `http://restaurants.localhost:5171` |
| admin | `http://admin.localhost:5172` |
| client | `http://client.localhost:5173` |
| delivery | `http://delivery.localhost:5174` |

In the compose `.env`, the backend redirects the browser to `AUTH_URL` and the `*_WEB_APP_URL` values, so they point at the dev servers. The `*_AUTH_URL` callbacks stay on Traefik (port 80):

```sh
AUTH_URL=http://auth.localhost:5170/
RESTAURANTS_WEB_APP_URL=http://restaurants.localhost:5171/
ADMIN_WEB_APP_URL=http://admin.localhost:5172/
CLIENT_WEB_APP_URL=http://client.localhost:5173/
DELIVERY_WEB_APP_URL=http://delivery.localhost:5174/

# same delimiter as your current value
ALLOWED_ORIGINS=http://auth.localhost:5170 http://restaurants.localhost:5171 http://admin.localhost:5172 http://client.localhost:5173 http://delivery.localhost:5174
```

Then, in each frontend repo:

```sh
# .env.local: the same VITE_* values as in the compose .env (they point at Traefik, not at the dev server)
pnpm dev --port 5173
```

- Check `REDIRECT_INFO_MAP`: if it contains app URLs, they need the dev ports too.
- If Vite answers "Blocked request. This host is not allowed", add the hostname to `server.allowedHosts`.
- If login cookies are marked `Secure` and the browser drops them over plain HTTP, switch to `https-custom` and serve Vite with the same mkcert certificate.

## Routing

Routing reproduces your two nginx configs exactly and is declared as labels on each service.

**API host (`API_HOST`, was nginx port 8001):** one host, routed by path.

| Path prefix | Service |
| --- | --- |
| `/api/v1/auth`, `/swagger/auth` | `auth` |
| `/restaurants`, `/api/v1/restaurants`, `/swagger/restaurants` | `restaurants` |
| `/api/v1/addresses`, `/api/v1/orders`, `/client`, `/swagger/client` | `client` |
| `/delivery`, `/api/v1/delivery`, `/swagger/delivery` | `delivery` |
| `/api/v1/images`, `/swagger/images` | `images` |
| `/api/v1/plans`, `/api/v1/subscriptions`, `/swagger/subscriptions`, `/webhooks/stripe/subscriptions` | `subscription` |
| `/api/v1/merchants`, `/api/v1/payments`, `/swagger/billing`, `/webhooks/stripe/billing` | `billing` |
| `/api/v1/ratings`, `/swagger/rating` | `rating` |
| `/api/v1/events`, `/admin`, `/swagger/admin` | `admin` |
| `/ws/client`, `/ws/restaurants`, `/ws/delivery` | `client-ws`, `restaurants-ws`, `delivery-ws` |
| `/mcp/restaurants`, `/mcp/images` | `restaurants-mcp`, `images-mcp` (prefix rewritten to `/mcp`) |

**App hosts (was nginx port 8000):** `AUTH_HOST`, `RESTAURANTS_HOST`, `ADMIN_HOST`, `CLIENT_HOST`, `DELIVERY_HOST` each serve their web app. On the restaurants, admin, client and delivery hosts, `/<app>/login`, `/<app>/logout`, `/<app>/refresh` and `/<app>/auth` go to the backend instead, so the browser stays on one origin. Everything else is the web app.

The API host must share a registrable domain with the app hosts (`api.example.com` and `restaurants.example.com`): the apps rely on same-site cookies, as they did with `localhost` in dev. Stripe webhooks are public on the API host by design. gRPC services and consumers have no labels and are never exposed.

To add a service, give it `traefik.enable`, a `traefik.http.services.<name>.loadbalancer.server.port` label and a router rule. To use another hostname for an app, change its `*_HOST` in `.env`.

## Scaling

Set replicas per tier in `.env` (`HTTP_REPLICAS`, `WS_REPLICAS`, `GRPC_REPLICAS`, `CONSUMER_REPLICAS`, `WEB_REPLICAS`), or override one service (this wins over `.env`):

```sh
docker compose up -d --scale auth=3 --scale client-grpc=2
```

Traefik discovers new replicas through the Docker socket and load-balances across them. There are no `container_name` entries or host-port mappings, so scaling cannot conflict. Before you scale, check:

- **gRPC:** callers resolve `<service>-grpc` through Docker DNS, which returns every replica's IP. Whether calls spread across replicas depends on the gRPC client's resolver and balancing policy (`pick_first` uses only one). Verify under load.
- **Consumers:** they use Redis consumer groups, so extra replicas share the work only if each instance registers a unique consumer name (for example from the hostname). Check the consumer code first.
- **WebSockets:** no sticky sessions are configured. That is fine if connections depend only on Redis pub/sub; add Traefik sticky labels if not.
- **Database connections** multiply with replicas (see above).
- **One host only:** Compose and Traefik's Docker provider are single-host. For several hosts, move to Swarm, Kubernetes or Cloud Run.

## Upgrades and rollbacks

```sh
# set BACKEND_VERSION / FRONTEND_VERSION in .env, then:
docker compose pull && docker compose up -d
```

To roll back, set the previous tags and repeat. Run database migrations separately and before upgrading. No healthchecks are defined for the application containers, so Traefik routes as soon as a container starts; expect a brief blip per service.

## Differences from the original compose files

- **One file, selected by profiles**, instead of separate infra and local-override files. The files `docker-compose.infra.yml`, `docker-compose.local.yml` and `traefik/` no longer exist. If you ran the old infra file, stop it first (`docker compose -f docker-compose.infra.yml down`) and note that its volumes were named `bmf-infra_*`; the new ones are `bmf_*`.
- **Profiles per tier:** `backend` and `web` select the backend and frontend services, and `postgres`, `redis` and `opensearch` start one piece of local infrastructure each (`infra` is still all three). Add `backend,web` to `COMPOSE_PROFILES` in existing installations.
- **gRPC addresses** are Docker DNS names (`auth-grpc:50050`) instead of `host.docker.internal:<port>`.
- **`admin` mounts the mTLS secrets.** The original prod file set the certificate paths but never mounted them.
- **`LOG_LEVEL`** is a variable everywhere (`auth` and `auth-grpc` had `TRACE` hard-coded). **`THROTTLING_*`** are variables with the same defaults (50 / 10).
- **Database settings** include the extra variables for Supabase (`DB_PORT`, `SCHEMA_PREFIX`, `DB_QUERY_EXEC_MODE`, pool limits).
- **Frontend services are named `*-web`**, matching the registry images and avoiding clashes with backend services of the same name.
- **Removed:** `depends_on`, `migrator`, KurrentDB, `build:` sections and host `ports:`.
- **nginx is replaced by Traefik** with the same routes. Differences: TLS (per profile), nginx's `gzip` is not replicated (add Traefik's `compress` middleware if you want it), and Traefik sets `X-Forwarded-For`, `X-Forwarded-Proto` and `X-Real-Ip` itself. If Traefik sits behind another proxy or CDN, configure `forwardedHeaders.trustedIPs`, otherwise per-IP throttling sees the proxy's address.

## Security notes

- Traefik mounts the Docker socket read-only, which still gives it broad visibility into the host. Put it behind a socket proxy for stricter setups.
- Never commit `.env`, `certs/`, `credentials/`, `backups/` or `tls/*.pem` (the included `.gitignore` covers them).
- Generate JWT, encryption and signing keys per installation. Do not reuse them across customers.
- `VITE_*` values are visible to anyone who loads the frontend. Restrict `VITE_GOOGLE_MAPS_API_KEY` by domain.
- Never publish PostgreSQL, Redis or OpenSearch to the internet. The compose file binds Postgres and Grafana to `127.0.0.1` only.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| Browser forces `https://` although you use the `http` profile | Not the server: `curl -I http://<host>` shows no redirect. The browser remembers an earlier permanent redirect or HSTS, or its HTTPS-upgrade feature is on. Open the site in a private window to confirm, then: in Brave set `brave://settings/shields` > "Upgrade connections to HTTPS" to Disabled (or turn it off per site in the Shields panel); clear the site's cached files; and delete the hosts at `brave://net-internals/#hsts` (Chrome: `chrome://net-internals/#hsts`). Or use `https-custom` and avoid the problem. |
| Nothing answers on ports 80 and 443 | No TLS profile in `COMPOSE_PROFILES`. Add one of `https-letsencrypt`, `https-custom`, `http`. |
| Only the proxy (or only infra) starts, no applications | `backend` and/or `web` missing from `COMPOSE_PROFILES`. Check with `docker compose config --services`. |
| `404 page not found` on `http://<app>.localhost/` while running the frontend with pnpm | Expected: no web container serves port 80. Open the dev server URL (`http://<app>.localhost:<port>`). |
| After login you land on `http://<app>.localhost/` (404) instead of the dev server | `*_WEB_APP_URL` (or `AUTH_URL`) still points at port 80. Set them to the dev server URLs. |
| CORS error from a pnpm dev server | The dev origin, with its port, is missing from `ALLOWED_ORIGINS`. |
| Browser shows `TRAEFIK DEFAULT CERT` on localhost | You are on `https-letsencrypt`, which cannot issue certificates for localhost. Use `https-custom` (mkcert) or `http`. |
| Same message on a server | Certificate not issued yet. Check `docker compose logs traefik-letsencrypt`, DNS for that host, and port 443. |
| `https-custom`: certificate name mismatch | The certificate does not list that hostname. Regenerate it with every host. |
| `required variable ... is missing a value` | Set it in `.env` (sections 1 and 4 of `.env.example`). |
| `pull access denied` / `unauthorized` | Registry login missing or the account lacks the Reader role. |
| `404 page not found` from Traefik | Wrong host or path. Check the host against the `*_HOST` values and the path against the routing table. |
| Login/refresh works in dev but not here | `API_HOST` and the app hosts are on different registrable domains, so cookies are cross-site. |
| Services can't reach `postgres` / `redis` / `opensearch` | Neither `infra` nor the single profile (`postgres`, `redis`, `opensearch`) is in `COMPOSE_PROFILES`, or the host variables still point elsewhere. |
| `opensearch` restarts: `vm.max_map_count ... is too low` | Set `vm.max_map_count=262144` on the host. |
| gRPC `certificate` / `permission denied` errors | Certificates in `./certs` unreadable in the container, or not generated from the same CA. |
| Frontend ignores `VITE_*` changes | The image reads them at build time rather than container start. Rebuild with `BUILD_ARGS`, or confirm the image's entrypoint injects runtime variables. |
| Consumers reprocess old events | `CONSUMERS_VERSION` changed. Keep it stable between upgrades. |