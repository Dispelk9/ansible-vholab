# Manual deploy runbook: NetBox + Diode server + NetBox Diode plugin + Orb Agent

This is a literal, step-by-step CLI mirror of what `netbox_deploy.yml` and
`netbox_discovery_deploy.yml` do, for when you want to run/debug steps by
hand on the host instead of through Ansible. **Prefer running the actual
playbooks for real deploys** - this doc exists so manual debugging on
vhohetzner1 can follow the same sequence the automation does, not as a
parallel way to actually manage the stack (drift between this file and the
roles is only caught when someone remembers to update both).

All commands assume you're root (or sudo) on **vhohetzner1**.

- Part 1: [NetBox](#part-1-netbox) (`netbox_deploy.yml` / `roles/netbox`)
- Part 2: [Diode server](#part-2-diode-server) (`roles/diode_server`)
- Part 3: [NetBox Diode plugin](#part-3-netbox-diode-plugin) (`roles/netbox_diode_plugin`)
- Part 4: [Orb Agent](#part-4-orb-agent) (`roles/orb_agent`)
- [Self-healing NetBox by hand](#self-healing-netbox-by-hand)

## Secrets you need before starting

These live in `host_vars/vhohetzner1/vault.yml` (ansible-vault encrypted) in
the repo. Decrypt and read them yourself - **never paste real secret values
into a chat/AI session or commit them anywhere unencrypted**:

```
ansible-vault view host_vars/vhohetzner1/vault.yml
```

You'll need these seven values from that file, referred to by these names
throughout this doc:

| Placeholder | Vault key |
|---|---|
| `<DIODE_REDIS_PASSWORD>` | `diode_redis_password` |
| `<DIODE_POSTGRES_PASSWORD>` | `diode_postgres_password` |
| `<HYDRA_POSTGRES_PASSWORD>` | `hydra_postgres_password` |
| `<HYDRA_SECRETS_SYSTEM_0>` | `hydra_secrets_system_0` |
| `<DIODE_TO_NETBOX_CLIENT_SECRET>` | `diode_to_netbox_client_secret` |
| `<NETBOX_TO_DIODE_CLIENT_SECRET>` | `netbox_to_diode_client_secret` |
| `<ORB_DIODE_CLIENT_SECRET>` | `orb_diode_client_secret` |

Wherever a command below shows one of these placeholders, substitute the
real value from the vault.

---

## Part 1: NetBox

Mirrors `netbox_deploy.yml` / `roles/netbox`.

### 1.1 Base directory + clone/update netbox-docker

```bash
mkdir -p /opt/netbox
chown root:root /opt/netbox
chmod 0755 /opt/netbox

if [ -d /opt/netbox/netbox-docker/.git ]; then
  # Re-running on an existing install. env/*.env (rewritten with real
  # secrets by 1.3 below, every run) and configuration/plugins.py
  # (customized in place by Part 3, the Diode plugin) are all tracked files
  # in netbox-docker's own git history - a plain `git pull` refuses on
  # them ("Local modifications exist"). git stash here is the WRONG fix:
  # it conflicts with itself, since 1.3 rewrites env/netbox.env again right
  # after the pull, and popping the stash later then tries to merge that
  # fresh rewrite against the stash's older version of the same file
  # ("local changes would be overwritten by merge"). skip-worktree instead
  # tells git to never compare/touch these specific files during pull at
  # all - idempotent to re-run every time.
  cd /opt/netbox/netbox-docker
  for f in env/netbox.env env/postgres.env env/redis.env env/redis-cache.env configuration/plugins.py; do
    git update-index --skip-worktree "$f"
  done
  git pull origin release
else
  git clone -b release https://github.com/netbox-community/netbox-docker.git /opt/netbox/netbox-docker
fi
```

### 1.2 Bind NetBox locally + healthchecks

```bash
cat > /opt/netbox/netbox-docker/docker-compose.override.yml << 'EOF'
services:
  netbox:
    ports:
      - "127.0.0.1:8083:8080"
    healthcheck:
      interval: 10s
      timeout: 10s
      retries: 30
      start_period: 60s
  netbox-worker:
    healthcheck:
      interval: 10s
      timeout: 10s
      retries: 30
      start_period: 60s
EOF
```

### 1.3 Generate/set secrets in the env files

netbox-docker ships `env/postgres.env`, `env/netbox.env`, `env/redis.env`,
`env/redis-cache.env` with placeholder values. Only generate new secrets if
they aren't already set (so re-running this doesn't rotate credentials on a
live install):

```bash
cd /opt/netbox/netbox-docker

# Check what's already set - if these grep hits show real values (not
# empty/placeholder), skip regenerating and reuse them instead.
grep '^POSTGRES_PASSWORD=' env/postgres.env
grep -E '^(DB_PASSWORD|SECRET_KEY|API_TOKEN_PEPPER_1|REDIS_PASSWORD|REDIS_CACHE_PASSWORD)=' env/netbox.env
grep '^REDIS_PASSWORD=' env/redis.env
grep '^REDIS_PASSWORD=' env/redis-cache.env

# If this is a genuinely first-time setup, generate fresh values:
DBPASS=$(openssl rand -base64 24)
REDISPASS=$(openssl rand -base64 24)
REDISCACHEPASS=$(openssl rand -base64 24)
SECRETKEY=$(openssl rand -base64 48)
PEPPER=$(openssl rand -base64 48)

sed -i "s/^POSTGRES_PASSWORD=.*/POSTGRES_PASSWORD=${DBPASS}/" env/postgres.env

sed -i \
  -e "s/^DB_PASSWORD=.*/DB_PASSWORD=${DBPASS}/" \
  -e "s/^SECRET_KEY=.*/SECRET_KEY=${SECRETKEY}/" \
  -e "s/^API_TOKEN_PEPPER_1=.*/API_TOKEN_PEPPER_1=${PEPPER}/" \
  -e "s/^REDIS_PASSWORD=.*/REDIS_PASSWORD=${REDISPASS}/" \
  -e "s/^REDIS_CACHE_PASSWORD=.*/REDIS_CACHE_PASSWORD=${REDISCACHEPASS}/" \
  env/netbox.env
# ALLOWED_HOSTS needs the public FQDN AND the bare "netbox" service name -
# anything reaching NetBox internally as http://netbox:8080/... (e.g.
# Diode's reconciler calling back into the plugin API) sends "netbox" as
# its Host header, and Django rejects anything not in this list with a
# bare, unstyled 400 (DisallowedHost) before URL routing even runs:
sed -i "s/^ALLOWED_HOSTS=.*/ALLOWED_HOSTS=netbox.dispelk9.de localhost 127.0.0.1 netbox/" env/netbox.env

sed -i "s/^REDIS_PASSWORD=.*/REDIS_PASSWORD=${REDISPASS}/" env/redis.env
sed -i "s/^REDIS_PASSWORD=.*/REDIS_PASSWORD=${REDISCACHEPASS}/" env/redis-cache.env
```

### 1.4 Start NetBox

**Explicit `-f` flags, not bare `docker compose`** - once Part 3 (the Diode
plugin) has ever run, `.env`'s `COMPOSE_FILE` points at a third file
(`docker-compose.diode.yml`) that swaps `netbox`/`netbox-worker` onto a
custom image built locally and never pushed anywhere. A bare `docker
compose pull` here picks that up from `.env` and fails outright trying to
pull it from a registry ("pull access denied for netbox-diode-custom").
Explicit `-f` flags override `.env`'s `COMPOSE_FILE` entirely, keeping this
part scoped to only the two files it actually owns. (If you run this Part 1
again after Part 3 has already run, NetBox briefly reverts to the plain
image until you re-run Part 3 - do that immediately after.)

```bash
cd /opt/netbox/netbox-docker
docker compose -f docker-compose.yml -f docker-compose.override.yml pull
docker compose -f docker-compose.yml -f docker-compose.override.yml up -d
docker compose ps
```

### 1.5 (Optional) TLS cert via certbot + Cloudflare DNS

Requires `/root/.secrets/cloudflare.ini` (Cloudflare API token) to already
exist.

```bash
certbot certonly --non-interactive --agree-tos -m admin@dispelk9.de \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 60 \
  --cert-name netbox.dispelk9.de \
  --key-type ecdsa \
  -d netbox.dispelk9.de
```

### 1.6 Nginx reverse proxy

```bash
cat > /etc/nginx/sites-available/nginx-netbox.conf << 'EOF'
upstream netbox_local {
    server 127.0.0.1:8083;
}

server {
    listen 80;
    server_name netbox.dispelk9.de;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name netbox.dispelk9.de;

    ssl_certificate     /etc/letsencrypt/live/netbox.dispelk9.de/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/netbox.dispelk9.de/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    location / {
        proxy_pass http://netbox_local;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

ln -sf /etc/nginx/sites-available/nginx-netbox.conf /etc/nginx/sites-enabled/nginx-netbox.conf
nginx -t
systemctl reload nginx
```

### 1.7 (Optional) Create a superuser

```bash
cd /opt/netbox/netbox-docker
docker compose exec -T netbox /opt/netbox/netbox/manage.py createsuperuser
```

---

## Part 2: Diode server

Mirrors `roles/diode_server`. Assumes Part 1 is done (needs
`/opt/netbox` to exist).

### 2.1 Pre-flight: disk space + shared docker network

```bash
df -h /
# Need >= 3GB free on / before pulling images.

docker network inspect netbox_discovery_net >/dev/null 2>&1 \
  || docker network create --driver bridge netbox_discovery_net
```

### 2.2 Tear down any previous deploy

This is **safe and intentional** - nothing in Diode's own postgres/redis is
source-of-truth data (that's NetBox's job), it's just OAuth2 state and
reconciler dedup state, both fully reconstructable. Skipping this step on a
re-run risks Postgres never re-running its init script against a
partially-initialized volume from an earlier failed attempt.

```bash
if [ -f /opt/netbox/discovery/diode/docker-compose.yaml ]; then
  cd /opt/netbox/discovery/diode
  docker compose down --volumes --remove-orphans
fi
```

### 2.3 Fetch the pinned compose file + nginx.conf

Pinned to an exact commit (not the floating `release` branch - see the long
comment in `roles/diode_server/defaults/main.yml` for why). Re-diff against
a newer commit before bumping this ref.

```bash
mkdir -p /opt/netbox/discovery/diode/nginx
mkdir -p /opt/netbox/backups

REF=c017ee191aab5b88327e03987a9c6de3f887e8ad

curl -fsSL -o /opt/netbox/discovery/diode/docker-compose.yaml \
  "https://raw.githubusercontent.com/netboxlabs/diode/${REF}/diode-server/docker/docker-compose.yaml"

curl -fsSL -o /opt/netbox/discovery/diode/nginx/nginx.conf \
  "https://raw.githubusercontent.com/netboxlabs/diode/${REF}/diode-server/docker/nginx/nginx.conf"
```

### 2.4 Detect the nginx front-door service name

```bash
cd /opt/netbox/discovery/diode
grep -B2 'image: nginx' docker-compose.yaml
# Confirmed on this pinned ref: the service is named "ingress-nginx".
# If you re-pin to a newer commit, re-check this - use whatever the actual
# service key is in the rest of this section.
```

### 2.5 Write `.env`

Substitute the `<...>` placeholders with the vault values listed at the top
of this doc. `diode_postgres_bootstrap_user` (`postgres`) must differ from
`diode_postgres_user` (`diode`) - the init script connects as the former
and then `CREATE USER`s the latter, which fails if they're the same name.

```bash
cat > /opt/netbox/discovery/diode/.env << 'EOF'
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_USERNAME=
REDIS_PASSWORD=<DIODE_REDIS_PASSWORD>

POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<DIODE_POSTGRES_PASSWORD>

DIODE_POSTGRES_DB_NAME=diode
DIODE_POSTGRES_USER=diode
DIODE_POSTGRES_PASSWORD=<DIODE_POSTGRES_PASSWORD>

HYDRA_POSTGRES_DB_NAME=hydra
HYDRA_POSTGRES_USER=hydra
HYDRA_POSTGRES_PASSWORD=<HYDRA_POSTGRES_PASSWORD>

GRPC_PORT=8081
# NOT a bare port number - see roles/diode_server/defaults/main.yml's long
# comment on diode_nginx_shadow_port. This is upstream's OWN
# `ports: - ${DIODE_NGINX_PORT}:80` line for ingress-nginx, which Compose
# ADDS to (never replaces with) our own override's port mapping below - so
# both would try to publish the SAME host port if this were a bare number.
# Embedding "127.0.0.1:<shadow-port>" here makes that entry loopback-only
# and on a DIFFERENT port than our real one (8280 + 1 = 8281), so they
# don't collide.
DIODE_NGINX_PORT=127.0.0.1:8281

HYDRA_STRATEGIES_ACCESS_TOKEN=jwt
HYDRA_STRATEGIES_REFRESH_TOKEN=jwt
HYDRA_STRATEGIES_JWT_SCOPE_CLAIM=both
HYDRA_TTL_ACCESS_TOKEN=1h
HYDRA_OIDC_SUBJECT_IDENTIFIERS_SUPPORTED_TYPES=public
HYDRA_URLS_SELF_ISSUER=http://hydra:4444
HYDRA_SECRETS_SYSTEM_0=<HYDRA_SECRETS_SYSTEM_0>

AUTH_HTTP_PORT=8080
OAUTH2_PUBLIC_SERVER_URL=http://hydra:4444
OAUTH2_ADMIN_SERVER_URL=http://hydra:4445
DIODE_AUTH_TOKEN_URL=http://diode-auth:8080/token

DIODE_TO_NETBOX_CLIENT_ID=diode-to-netbox
DIODE_TO_NETBOX_CLIENT_SECRET=<DIODE_TO_NETBOX_CLIENT_SECRET>

NETBOX_DIODE_PLUGIN_API_BASE_URL=http://netbox:8080/api/plugins/diode
NETBOX_DIODE_PLUGIN_SKIP_TLS_VERIFY=true
MIGRATION_ENABLED=true
ENABLE_GRAPH_DB=false

LOGGING_LEVEL=info
SENTRY_DSN=

TELEMETRY_METRICS_EXPORTER=none
TELEMETRY_METRICS_PORT=9090
TELEMETRY_TRACES_EXPORTER=none
TELEMETRY_ENVIRONMENT=production
EOF
chmod 0600 /opt/netbox/discovery/diode/.env
```

### 2.6 Write `oauth2/client/client-credentials.json`

This is how Diode's `diode-auth-bootstrap` container registers the three
OAuth2 clients in Hydra - **not** via env vars. Mode `0644`/dir `0755`
(world-readable) is deliberate: this file is bind-mounted (not a Docker
secret) into a container that runs as a non-root user; a restrictive mode
here means that user can't even traverse into the directory.

```bash
mkdir -p /opt/netbox/discovery/diode/oauth2/client
chmod 0755 /opt/netbox/discovery/diode/oauth2/client

cat > /opt/netbox/discovery/diode/oauth2/client/client-credentials.json << 'EOF'
[
  {
    "client_id": "diode-to-netbox",
    "client_secret": "<DIODE_TO_NETBOX_CLIENT_SECRET>",
    "grant_types": ["client_credentials"],
    "scope": "netbox:read netbox:write"
  },
  {
    "client_id": "netbox-to-diode",
    "client_secret": "<NETBOX_TO_DIODE_CLIENT_SECRET>",
    "grant_types": ["client_credentials"],
    "scope": "diode:read diode:write"
  },
  {
    "client_id": "orb-agent",
    "client_secret": "<ORB_DIODE_CLIENT_SECRET>",
    "grant_types": ["client_credentials"],
    "scope": "diode:ingest"
  }
]
EOF
chmod 0644 /opt/netbox/discovery/diode/oauth2/client/client-credentials.json
```

### 2.7 Write `docker-compose.override.yml`

Re-points the network onto the shared bridge, publishes nginx on the host,
and fixes an upstream bug in `diode-auth-bootstrap`'s command (Dockerfile
bakes the script at `/etc/config/oauth2/bootstrap-clients.sh`, but the
pinned compose file's command looks for it one directory too deep).

```bash
cat > /opt/netbox/discovery/diode/docker-compose.override.yml << 'EOF'
networks:
  default:
    name: "netbox_discovery_net"
    external: true

services:
  "ingress-nginx":
    ports:
      - "127.0.0.1:8280:80"

  diode-auth-bootstrap:
    command: ["/bin/sh", "/etc/config/oauth2/bootstrap-clients.sh"]
EOF
```

### 2.8 Validate, pull, start

```bash
cd /opt/netbox/discovery/diode
docker compose config -q
docker compose pull
docker compose up -d
```

If `up` fails with `port is already allocated` on 8280 even though nothing
is actually listening (`ss -ltnp | grep 8280`), it's very likely a stuck
Docker port-allocator - just retry `docker compose up -d` a couple of times
a few seconds apart before assuming anything else is wrong.

### 2.9 Confirm everything came up correctly

`hydra-migrate` and `diode-auth-bootstrap` are **one-shot jobs** - they're
supposed to show `Exited (0)`, not stay running. Everything else
(`diode-auth`, `diode-ingester`, `diode-reconciler`, `hydra`,
`ingress-nginx`, `postgres`, `redis`) should show `Up`/`running`.

```bash
docker compose ps -a
docker compose logs diode-auth-bootstrap --tail 20   # should show 3x "client ... created"
docker compose logs postgres --tail 20               # should show CREATE ROLE / CREATE DATABASE x2, no errors
nc -zv 127.0.0.1 8280                                 # Diode's client-facing gRPC endpoint
docker compose ps --format json postgres | python3 -c "import sys,json; print(json.load(sys.stdin).get('Health'))"
```

---

## Part 3: NetBox Diode plugin

Mirrors `roles/netbox_diode_plugin`. Assumes Parts 1 and 2 are done (needs
NetBox running and Diode server up, since `diode_plugin_diode_target`
below points at Diode's front door and the reconciler needs to reach
NetBox).

### 3.1 Self-heal check (do this FIRST, every time)

Before assuming NetBox is healthy, check it explicitly. If it's not
running, the most likely cause is a bad `configuration/plugins.py` from an
earlier run of this same section - see
[Self-healing NetBox by hand](#self-healing-netbox-by-hand) below, fix
that, confirm NetBox is `Up`, **then** continue here.

```bash
cd /opt/netbox/netbox-docker
docker compose ps netbox
# Must show State=running before proceeding.
```

### 3.2 Detect NetBox's core version, gate on plugin compatibility

```bash
cd /opt/netbox/netbox-docker
docker compose exec -T netbox /opt/netbox/venv/bin/python3 /opt/netbox/netbox/manage.py shell -c \
  "from django.conf import settings; print(settings.VERSION)"
# e.g. "4.5.3-Docker-4.0.1" - the part before "-Docker" is NetBox core's
# own version. It must be >= 4.4.10 and <= 4.6.99 for plugin 1.12.0 - if
# not, check https://github.com/netboxlabs/diode-netbox-plugin/releases
# for a plugin version whose min/max_version covers your NetBox version.
```

### 3.3 Detect which app services are actually running

```bash
cd /opt/netbox/netbox-docker
docker compose ps --format json | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if line:
        print(json.loads(line)['Service'])
"
# On this host, effective services = netbox, netbox-worker
# (netbox-housekeeping isn't defined/running here - don't start it just to
# match a stock list; only rebuild what's already running).
```

### 3.4 Detect NetBox's current upstream image

```bash
cd /opt/netbox/netbox-docker
docker inspect --format '{{.Config.Image}}' netbox-docker-netbox-1
# e.g. netboxcommunity/netbox:v4.5-4.0.1 - this is the FROM base for the
# custom image below. Don't hardcode it; always detect live, since the
# netbox role tracks netbox-docker's floating "release" branch.
```

### 3.5 Build the custom NetBox + Diode plugin image

```bash
UPSTREAM_IMAGE=$(docker inspect --format '{{.Config.Image}}' netbox-docker-netbox-1)

mkdir -p /opt/netbox/discovery/netbox-diode-build

cat > /opt/netbox/discovery/netbox-diode-build/Dockerfile << EOF
FROM ${UPSTREAM_IMAGE}

COPY plugin_requirements.txt /opt/netbox/plugin_requirements.txt
RUN uv pip install --python /opt/netbox/venv/bin/python --no-cache -r /opt/netbox/plugin_requirements.txt
EOF

cat > /opt/netbox/discovery/netbox-diode-build/plugin_requirements.txt << 'EOF'
netboxlabs-diode-netbox-plugin==1.12.0
EOF

docker build -t netbox-diode-custom:diode-1.12.0 /opt/netbox/discovery/netbox-diode-build
```

> **Note:** `netboxcommunity/netbox:v4.5-4.0.1` ships `uv`, not a `pip`
> binary inside `/opt/netbox/venv/bin/` - confirmed by inspecting the image
> directly. If a future base image changes this, check
> `docker run --rm --entrypoint sh <image> -c "which uv; which pip"`
> before assuming the install command above still works.

### 3.6 Provision the netbox_to_diode Docker secret

```bash
mkdir -p /opt/netbox/discovery/netbox-diode-build/secrets
chmod 0700 /opt/netbox/discovery/netbox-diode-build/secrets

printf '%s' '<NETBOX_TO_DIODE_CLIENT_SECRET>' > /opt/netbox/discovery/netbox-diode-build/secrets/netbox_to_diode
chmod 0600 /opt/netbox/discovery/netbox-diode-build/secrets/netbox_to_diode
```

### 3.7 Configure the plugin in `configuration/plugins.py`

**Do NOT assume `PLUGINS`/`PLUGINS_CONFIG` are already defined earlier in
the file** - that assumption crash-looped a live NetBox instance
(`NameError: name 'PLUGINS' is not defined`) because this deployment's own
`configuration/plugins.py` doesn't define them itself. The block below is
defensive against that.

```bash
cp /opt/netbox/netbox-docker/configuration/plugins.py \
   /opt/netbox/backups/plugins.py.$(date +%Y%m%dT%H%M%S).bak

# Remove any existing managed block first (idempotent re-run):
sed -i '/# BEGIN ANSIBLE MANAGED BLOCK - netbox_diode_plugin/,/# END ANSIBLE MANAGED BLOCK - netbox_diode_plugin/d' \
  /opt/netbox/netbox-docker/configuration/plugins.py

cat >> /opt/netbox/netbox-docker/configuration/plugins.py << 'EOF'
# BEGIN ANSIBLE MANAGED BLOCK - netbox_diode_plugin
try:
    PLUGINS
except NameError:
    PLUGINS = []
try:
    PLUGINS_CONFIG
except NameError:
    PLUGINS_CONFIG = {}

if "netbox_diode_plugin" not in PLUGINS:
    PLUGINS.append("netbox_diode_plugin")

PLUGINS_CONFIG.setdefault("netbox_diode_plugin", {})
PLUGINS_CONFIG["netbox_diode_plugin"].update({
    "diode_target_override": "grpc://ingress-nginx:80/diode",
    "diode_username": "diode",
    "netbox_to_diode_client_id": "netbox-to-diode",
})
# END ANSIBLE MANAGED BLOCK - netbox_diode_plugin
EOF
```

### 3.8 Add the Diode compose overlay + point `COMPOSE_FILE` at it

**Do not** redefine the top-level `default` network here - that used to
override the *entire* netbox-docker project's network, which swept
netbox-docker's own `postgres`/`redis`/`redis-cache` containers onto the
shared discovery network too. Diode's own compose project *also* has
services literally named `postgres` and `redis` - two different containers
answering to the same DNS alias on the same network, causing
`diode-reconciler` to non-deterministically connect to netbox-docker's
database instead of Diode's own and fail auth against a database that has
no idea what the `diode` role is (confirmed live via
`docker network inspect netbox_discovery_net` listing both
`netbox-docker-postgres-1` and `diode-postgres-1`, and by checking that
reconciler's actual connection IP matched neither). Only `netbox`/
`netbox-worker` get multi-homed onto the shared network below, in addition
to (not instead of) their existing `default` network - everything else in
netbox-docker stays exactly where it was.

```bash
cat > /opt/netbox/netbox-docker/docker-compose.diode.yml << 'EOF'
services:
  netbox:
    image: "netbox-diode-custom:diode-1.12.0"
    secrets:
      - netbox_to_diode
    networks:
      - default
      - netbox_discovery
  netbox-worker:
    image: "netbox-diode-custom:diode-1.12.0"
    secrets:
      - netbox_to_diode
    networks:
      - default
      - netbox_discovery

networks:
  netbox_discovery:
    name: "netbox_discovery_net"
    external: true

secrets:
  netbox_to_diode:
    file: "/opt/netbox/discovery/netbox-diode-build/secrets/netbox_to_diode"
EOF

cd /opt/netbox/netbox-docker
docker compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.diode.yml config -q

cp .env /opt/netbox/backups/netbox-docker.env.$(date +%Y%m%dT%H%M%S).bak 2>/dev/null || true

# Point docker compose at all three files by default:
grep -q '^COMPOSE_FILE=' .env 2>/dev/null \
  && sed -i 's|^COMPOSE_FILE=.*|COMPOSE_FILE=docker-compose.yml:docker-compose.override.yml:docker-compose.diode.yml|' .env \
  || echo 'COMPOSE_FILE=docker-compose.yml:docker-compose.override.yml:docker-compose.diode.yml' >> .env
```

### 3.9 Recreate NetBox app services + confirm health

Recreate **every currently-running netbox-docker service**, not just
`netbox`/`netbox-worker` - `postgres`/`redis`/`redis-cache` need to be
recreated too if they were previously attached to the shared discovery
network by an older version of `docker-compose.diode.yml` (see 3.8); this
won't start anything that wasn't already running (e.g. won't start
`netbox-housekeeping` if it's absent), it only reconciles services already
part of this project against the current config:

```bash
cd /opt/netbox/netbox-docker
RUNNING_SERVICES=$(docker compose ps --format json | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if line:
        print(json.loads(line)['Service'])
")
docker compose up -d $RUNNING_SERVICES

# Wait for netbox to report healthy (bounded: 30 retries x 10s = 5min,
# matching netbox_health_retries/netbox_health_delay defaults - if it's
# still not healthy after this, stop and read the logs instead of waiting
# longer):
CID=$(docker compose ps -q netbox)
for i in $(seq 1 30); do
  STATUS=$(docker inspect --format '{{.State.Health.Status}}' "$CID")
  [ "$STATUS" = "healthy" ] && break
  echo "waiting for netbox to become healthy (${i}/30, currently: ${STATUS})..."
  sleep 10
done
[ "$STATUS" = "healthy" ] || { echo "netbox did not become healthy - check: docker compose logs netbox"; exit 1; }

# Confirm it joined the shared discovery network:
docker inspect --format '{{json .NetworkSettings.Networks}}' netbox-docker-netbox-1
# Should include "netbox_discovery_net".
```

### 3.10 Verify the plugin actually loaded

```bash
cd /opt/netbox/netbox-docker
docker compose exec -T netbox /opt/netbox/venv/bin/python3 /opt/netbox/netbox/manage.py shell -c \
  "from django.conf import settings; print(list(settings.PLUGINS))"
# Must include 'netbox_diode_plugin'.
```

---

## Part 4: Orb Agent

Mirrors `roles/orb_agent`. Independent of NetBox/Diode being up (it can be
deployed any time after Part 2, since it reaches Diode via the host-published
port, not the docker network) - but it obviously can't successfully *send*
anything to Diode until Part 2 is healthy.

### 4.1 Refuse to go live with the placeholder scan target

**Do not flip `orb_dry_run` to `false` while `orb_discovery_targets` still
includes the RFC 5737 placeholder `192.0.2.0/24`** - that's a deliberate
non-routable test range, not a guess at your real network. Confirm your
actual, permitted scan range (see `group_vars/vhodockers.yml` -
`10.0.0.0/24`, the Hetzner private network, is what's currently configured
there alongside the placeholder) before disabling dry-run.

### 4.2 Directories + `.env`

```bash
mkdir -p /opt/netbox/discovery/orb/data
chmod 0750 /opt/netbox/discovery/orb /opt/netbox/discovery/orb/data

cat > /opt/netbox/discovery/orb/.env << 'EOF'
DIODE_CLIENT_ID=orb-agent
DIODE_CLIENT_SECRET=<ORB_DIODE_CLIENT_SECRET>
EOF
chmod 0600 /opt/netbox/discovery/orb/.env
```

### 4.3 `agent.yaml`

This example keeps `dry_run: true` (the safe default - writes discovered
inventory as JSON instead of calling Diode). To go live, flip `dry_run` to
`false` and uncomment the `target`/`client_id`/`client_secret` block (see
`roles/orb_agent/templates/agent.yaml.j2` for the exact conditional
structure being mirrored here).

```bash
cat > /opt/netbox/discovery/orb/agent.yaml << 'EOF'
version: "1.0"
orb:
  config_manager:
    active: local
  backends:
    network_discovery:
    common:
      diode:
        dry_run: true
        dry_run_output_dir: "/opt/orb/dry-run"
        agent_name: "vhohetzner1"
  policies:
    network_discovery:
      discovery_1:
        config:
          schedule: "*/15 * * * *"
          defaults:
            description: "Discovered by ansible-vholab"
            tags:
              - net-discovery
              - orb-agent
        scope:
          targets: ["192.0.2.0/24", "10.0.0.0/24"]
          scan_types: ["connect"]
          skip_host: true
          fast_mode: false
EOF
```

For a **live** (non-dry-run) config, replace the `diode:` block with:

```yaml
      diode:
        dry_run: false
        target: "grpc://127.0.0.1:8280/diode"
        client_id: "${DIODE_CLIENT_ID}"
        client_secret: "${DIODE_CLIENT_SECRET}"
        agent_name: "vhohetzner1"
```

### 4.4 `compose.yml`

`network_mode: host` is required for raw network scanning (nmap SYN/etc.);
`NET_RAW`/`NET_ADMIN` are the least-privileged capabilities upstream's own
examples imply.

```bash
cat > /opt/netbox/discovery/orb/compose.yml << 'EOF'
services:
  orb-agent:
    image: "netboxlabs/orb-agent:2.11.0"
    container_name: orb-agent
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_RAW
      - NET_ADMIN
    env_file:
      - .env
    volumes:
      - "/opt/netbox/discovery/orb/agent.yaml:/opt/orb/agent.yaml:ro"
      - "/opt/netbox/discovery/orb/data:/opt/orb"
    command: ["run", "-c", "/opt/orb/agent.yaml"]
EOF
```

### 4.5 Validate, pull, start

```bash
cd /opt/netbox/discovery/orb
docker compose config -q
docker compose pull
docker compose up -d
docker compose ps -q orb-agent   # should print a container ID once it's up
```

### 4.6 Check output (dry-run mode)

```bash
ls -la /opt/netbox/discovery/orb/data/dry-run/
docker compose logs orb-agent --tail 30
```

---

## Self-healing NetBox by hand

If `docker compose ps netbox` (Part 3.1) shows anything other than
`running` - most likely because `configuration/plugins.py` has a bad
managed block from a previous incomplete run - repair it directly rather
than re-cloning/resetting netbox-docker:

```bash
cd /opt/netbox/netbox-docker

# 1. Re-provision the secret (in case it's missing/stale):
mkdir -p /opt/netbox/discovery/netbox-diode-build/secrets
printf '%s' '<NETBOX_TO_DIODE_CLIENT_SECRET>' > /opt/netbox/discovery/netbox-diode-build/secrets/netbox_to_diode
chmod 0600 /opt/netbox/discovery/netbox-diode-build/secrets/netbox_to_diode

# 2. Re-apply configuration/plugins.py - repeat step 3.7 above verbatim.

# 3. Restart:
docker compose up -d netbox netbox-worker

# 4. Confirm:
docker compose logs netbox --tail 30
docker compose ps netbox
```

If it's still crash-looping after that, the cause is something other than
the known `PLUGINS` bug - read the actual traceback
(`docker compose logs netbox`, or set `DB_WAIT_DEBUG=1` in `env/netbox.env`
for a full Python traceback) rather than re-applying this same fix blindly.

**Never run `netbox_deploy.yml`/the git-clone step (Part 1.1) to "fix" a
running install** - `configuration/plugins.py` is a tracked file in the
netbox-docker git repo that Part 3.7 modifies in place; a `git pull`/reset
there will either refuse (dirty working tree) or silently discard that
customization.
