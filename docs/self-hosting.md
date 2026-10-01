# Self-Hosting ShadowNote

ShadowNote is designed to be self-hosted: it is a zero-knowledge system, so running your own server is the strongest privacy guarantee — the server only ever relays ciphertext, and now the ciphertext never leaves infrastructure you control either.

This guide covers the recommended Docker deployment. For local development see the [README](../README.md#development).

## Requirements

- Docker Engine 20.10+ (with the compose plugin)
- ~200 MB RAM for the API + Redis, a few MB of disk for the SQLite fallback and Redis AOF
- No GPU, no external database, no account system

## Quick Start

```bash
git clone https://github.com/build-workbench/shadow-note.git
cd shadow-note

# optional: adjust the public origin / port
cp .env.example .env

docker compose up -d --build
```

Open <http://localhost:8080>, create a note chain, and back up the 12-word mnemonic it shows you. **The mnemonic is the only key to your notes — the server cannot recover it, and neither can we.**

What the compose stack starts:

| Container | Image | Role |
|---|---|---|
| `web` | built from `apps/web` | nginx serves the built SPA and reverse-proxies `/socket.io` to the API (same origin, no CORS) |
| `api` | built from `apps/api` | Express + Socket.IO sync server; primary storage Redis, fallback SQLite |
| `redis` | `redis:7-alpine` | primary ciphertext store with AOF persistence |

Data lives in two named volumes (`api-data` for SQLite, `redis-data` for Redis AOF). To start over: `docker compose down -v`.

## Configuration

All variables live in `.env` (see `.env.example`):

| Variable | Default | Description |
|---|---|---|
| `SHADOW_NOTE_ORIGIN` | `http://localhost:8080` | Public site origin, passed to the API as `CORS_ORIGIN`. The backend rejects `*` in production — set this to the exact address users visit (e.g. `https://notes.example.com`). |
| `SHADOW_NOTE_PORT` | `8080` | Host port published for the web container. |
| `SHADOW_NOTE_SOCKET_URL` | `/` | Socket URL baked into the frontend at build time. Keep `/` for same-origin mode; set to the API's public URL only when serving frontend and API from different domains (requires rebuilding the web image). |
| `SHADOW_NOTE_LOG_LEVEL` | `info` | API log level (`error` / `warn` / `info` / `debug`). |

Other API tunables (room TTL, storage adapters, PBKDF2 iterations) are documented in `apps/api/.env.example` and `apps/web/.env.example`; add them to the `api`/`web` services in `docker-compose.yml` if you need to change them.

## TLS termination (recommended)

The compose stack serves plain HTTP on `SHADOW_NOTE_PORT`. Put your existing reverse proxy in front for HTTPS. Because the frontend connects to `/socket.io` on the same origin, the proxy only needs to forward WebSocket upgrades.

### Caddy

```caddy
notes.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

That's it — Caddy provisions the certificate and forwards WebSocket upgrades automatically.

### nginx

```nginx
server {
    listen 443 ssl http2;
    server_name notes.example.com;

    ssl_certificate     /etc/letsencrypt/live/notes.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/notes.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        # long-lived WebSocket sessions
        proxy_read_timeout 86400s;
    }
}
```

Then set `SHADOW_NOTE_ORIGIN=https://notes.example.com` and `docker compose up -d` again.

> **Why same-origin matters**: the frontend derives all crypto keys locally; the only server-side restriction is the API's CORS origin allow-list. Serving the SPA and Socket.IO behind one domain keeps the deployment to a single public endpoint and avoids cross-origin cookie/token handling entirely.

## Split-domain deployment (advanced)

If you must serve the SPA and API from different origins:

1. Build the web image with the API's public address:
   ```bash
   SHADOW_NOTE_SOCKET_URL=https://sync.example.com docker compose up -d --build
   ```
2. Set `CORS_ORIGIN` to the SPA's origin, e.g. `SHADOW_NOTE_ORIGIN=https://notes.example.com`.
3. Expose the API port and put TLS on both origins.

The same-origin setup above is simpler and is what we test; prefer it unless you have a specific reason.

## Upgrades & backups

- **Upgrade**: `git pull && docker compose up -d --build`. Protocol changes are noted in [CHANGELOG.md](../CHANGELOG.md).
- **Backup**: stop writes and copy both volumes:
  ```bash
  docker compose stop api
  docker run --rm -v shadow-note_api-data:/src -v "$PWD":/bak alpine tar czf /bak/api-data.tgz -C /src .
  docker run --rm -v shadow-note_redis-data:/src -v "$PWD":/bak alpine tar czf /bak/redis-data.tgz -C /src .
  docker compose start api
  ```
- **Restore**: recreate the stack, then untar into the corresponding volumes before starting `api`.

What a backup protects: ciphertext rooms and sync metadata. It does **not** include mnemonics — those exist only on user devices.

## Threat model recap

- The server stores and relays AES-256-GCM ciphertext; note contents are never decryptable server-side.
- Room membership is enforced by protocol version + room handshake, not by accounts — there is no user database to leak.
- Compromising the server exposes ciphertext, room identifiers and timestamps, not note contents.
- Full details: [SECURITY.md](../SECURITY.md) and [ARCHITECTURE.md](../ARCHITECTURE.md).
