# docker-webdav2

A lightweight WebDAV server running on Apache httpd in Docker. Fork of [BytemarkHosting/docker-webdav](https://github.com/BytemarkHosting/docker-webdav) with security hardening, modern TLS, CI/CD, and operational improvements.

## Supported tags

* `2.4`, `latest` — built from [`2.4/Dockerfile`](2.4/Dockerfile)

Base image: `httpd:2.4.62-alpine3.20` (pinned for reproducibility)

## What's New in This Fork

| Improvement | Description |
|---|---|
| **Pinned base image** | `httpd:2.4.62-alpine3.20` instead of floating `httpd:alpine` |
| **Modern TLS** | TLS 1.2+ only, Mozilla Intermediate cipher suite, HSTS + security headers |
| **Healthcheck** | Built-in `HEALTHCHECK` via curl — Docker detects hung Apache |
| **OCI labels** | `org.opencontainers.image.*` metadata on the image |
| **Password safety** | `htpasswd` reads password from stdin, not command line (no process list leak) |
| **Optimized startup** | `chown -R` only runs when ownership is wrong — fast on large data dirs |
| **`.dockerignore`** | `.git/`, docs, and unrelated files excluded from build context |
| **docker-compose.yml** | Ready-to-use compose file included in the repo |
| **GitHub Actions CI** | Automated build + smoke test (auth check, WebDAV PROPFIND, healthcheck) on every push + weekly rebuild |
| **Robust shell quoting** | All env vars properly quoted in entrypoint script |

## Usage

### Basic WebDAV server (HTTP)

```bash
docker run --restart always -v /srv/dav:/var/lib/dav \
    -e AUTH_TYPE=Digest -e USERNAME=alice -e PASSWORD=secret1234 \
    --publish 80:80 -d hoelee/webdav2
```

> When using unencrypted HTTP, use `Digest` authentication (instead of `Basic`) to avoid sending plaintext passwords.

### Via Docker Compose

```yaml
services:
  webdav:
    build:
      context: ./2.4
      dockerfile: Dockerfile
    image: hoelee/webdav2:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      AUTH_TYPE: Digest
      USERNAME: alice
      PASSWORD: secret1234
      SERVER_NAMES: dav.example.com
      # SSL_CERT: selfsigned  # uncomment for self-signed SSL
    volumes:
      - dav-data:/var/lib/dav
      # - ./cert.pem:/cert.pem:ro        # bind mount your own cert
      # - ./privkey.pem:/privkey.pem:ro
      # - ./user.passwd:/user.passwd:ro   # multi-user auth file

volumes:
  dav-data:
```

Save as `docker-compose.yml` and run:

```bash
docker compose up -d
```

### Secure WebDAV with SSL

**Option A — Reverse proxy (recommended):** Use Traefik, Caddy, or nginx to terminate TLS with a Let's Encrypt certificate. The container serves plain HTTP behind the proxy.

**Option B — Self-signed certificate:**

```bash
docker run --restart always -v /srv/dav:/var/lib/dav \
    -e AUTH_TYPE=Basic -e USERNAME=test -e PASSWORD=test \
    -e SSL_CERT=selfsigned --publish 443:443 -d hoelee/webdav2
```

**Option C — Bring your own certificate:** Bind mount `/cert.pem` and `/privkey.pem`:

```bash
docker run --restart always -v /srv/dav:/var/lib/dav \
    -v /path/to/cert.pem:/cert.pem:ro \
    -v /path/to/privkey.pem:/privkey.pem:ro \
    -e AUTH_TYPE=Basic -e USERNAME=test -e PASSWORD=test \
    --publish 443:443 -d hoelee/webdav2
```

### Authenticate multiple users

Specifying `USERNAME` and `PASSWORD` only supports a single user. For multiple logins, bind mount your own `/user.passwd`:

**Basic auth:**
```bash
touch user.passwd
htpasswd -B user.passwd alice
htpasswd -B user.passwd bob
```

**Digest auth:**
```bash
touch user.passwd
htdigest user.passwd WebDAV alice
htdigest user.passwd WebDAV bob
```

Then: `-v /path/to/user.passwd:/user.passwd:ro`

> **Note:** If you specify a custom `REALM`, use that name with `htdigest` instead of `WebDAV`.

## Environment Variables

All environment variables are optional. You should at least specify `USERNAME` and `PASSWORD` (or bind mount `/user.passwd`) — otherwise nobody can access the server.

| Variable | Default | Description |
|---|---|---|
| `SERVER_NAMES` | `localhost` | Comma-separated domains. First = ServerName, rest = ServerAlias |
| `LOCATION` | `/` | URL path for WebDAV (e.g. `/webdav` → `example.com/webdav`) |
| `AUTH_TYPE` | `Basic` | `Basic` (best for HTTPS) or `Digest` (best for HTTP) |
| `REALM` | `WebDAV` | AuthName shown to clients on login prompt |
| `USERNAME` | — | Single-user username (ignored if `/user.passwd` is bind mounted) |
| `PASSWORD` | — | Single-user password (ignored if `/user.passwd` is bind mounted) |
| `ANONYMOUS_METHODS` | — | Comma-separated HTTP methods allowed without auth (e.g. `GET,OPTIONS,PROPFIND`). Set to `ALL` to disable auth |
| `SSL_CERT` | — | Set to `selfsigned` to generate a self-signed certificate |

## Healthcheck

The image includes a built-in health check:

```bash
docker inspect --format='{{.State.Health.Status}}' <container_name>
```

Status will be `healthy`, `unhealthy`, or `starting`. The check runs every 30s with a 5s timeout.

## Volumes

| Path | Purpose |
|---|---|
| `/var/lib/dav` | WebDAV data directory + lock database |
| `/user.passwd` | Optional: bind mount your own multi-user auth file |
| `/cert.pem` | Optional: bind mount your SSL certificate |
| `/privkey.pem` | Optional: bind mount your SSL private key |

## CI/CD

GitHub Actions workflow in `.github/workflows/ci.yml`:
- **On push/PR:** Builds the image, starts a container, runs smoke tests (HTTP response, WebDAV PROPFIND with auth, auth rejection)
- **Weekly schedule:** Rebuilds to catch base image security updates
- Build cache via GitHub Actions cache (`type=gha`)

## License

MIT — see [LICENSE](LICENSE). Original work © Bytemark Hosting.
