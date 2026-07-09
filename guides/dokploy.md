# Deploying Huly on Dokploy (Compose stack)

This guide runs Huly as a **Dokploy Compose service**, behind Dokploy's built-in
Traefik. Dokploy handles the domain + SSL; Huly's internal nginx does the app
path-routing (`/_accounts`, `/_transactor`, `/_collaborator`, ...).

Use **`compose.dokploy.yml`** — it's identical to the upstream `compose.yml`
except the nginx **host-port publish is removed** (Traefik owns :80/:443 on the
host and routes to the nginx service internally).

## 1. Generate secrets (do NOT commit them)

```bash
openssl rand -hex 32   # SECRET (Huly)
openssl rand -hex 32   # CockroachDB password
openssl rand -hex 32   # Redpanda password
```

## 2. Create the Compose service in Dokploy

- **Create → Compose**, Provider = your fork of this repo, Compose path = `compose.dokploy.yml`.
- **Environment** (paste, fill your generated secrets):

```
DOCKER_NAME=huly
HULY_VERSION=v0.7.426
DESKTOP_CHANNEL=0.7.426
HOST_ADDRESS=huly.example.com
SECURE=true
HTTP_PORT=80
HTTP_BIND=
TITLE=Huly
DEFAULT_LANGUAGE=en
LAST_NAME_FIRST=true
CR_DATABASE=defaultdb
CR_USERNAME=selfhost
CR_USER_PASSWORD=<CR_SECRET>
CR_DB_URL=postgres://selfhost:<CR_SECRET>@cockroach:26257/defaultdb
REDPANDA_ADMIN_USER=superadmin
REDPANDA_ADMIN_PWD=<RP_SECRET>
SECRET=<HULY_SECRET>
VOLUME_ELASTIC_PATH=
VOLUME_FILES_PATH=
VOLUME_CR_DATA_PATH=
VOLUME_CR_CERTS_PATH=
VOLUME_REDPANDA_PATH=
```

Notes:
- `SECURE=true` makes Huly generate `https://`/`wss://` URLs — correct, because
  TLS is terminated at Traefik/Cloudflare. The internal nginx still listens on
  plain `:80` (`.huly.nginx` = `listen 80; server_name _;`), which is exactly
  what you want behind a reverse proxy.
- `HTTP_PORT`/`HTTP_BIND` are unused here (host port removed) but harmless.

## 3. Domain

In the Compose service → **Domains**: host = `huly.example.com`,
service = **`nginx`**, container port = **80**, HTTPS on. Dokploy writes the
Traefik route + issues the certificate.

## 4. Reverse proxy / tunnel

Point your domain at the server. If you use Cloudflare Tunnel, add a public
hostname `huly.example.com → http://localhost:80` (Traefik) + a DNS record.

## 5. Deploy

Hit **Deploy**. First boot pulls ~13 images and initialises CockroachDB /
Elastic — give it a few minutes. Then open the domain and create the first
workspace.

## Updates

`git pull` upstream, bump `HULY_VERSION` (read `MIGRATION.md` first for breaking
changes), redeploy.
