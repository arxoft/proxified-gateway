# Proxy

Shared reverse proxy for all apps hosted on this machine. Deploy once — every app
that joins the `nginx_proxy` network with a `VIRTUAL_HOST` env var gets routed
automatically. No changes needed here when adding or removing apps.

Built on [jwilder/nginx-proxy](https://github.com/nginx-proxy/nginx-proxy), which
watches the Docker socket and manages nginx vhost configs in real time.

---

## How it works

```
Internet
   │
   ▼
nginx-proxy :80 / :443
   │
   ├──  VIRTUAL_HOST=vpos.com        →  vpos_web container
   ├──  VIRTUAL_HOST=pma.vpos.com    →  vpos_phpmyadmin container
   └──  VIRTUAL_HOST=example.com     →  someother_web container
```

The proxy listens on the Docker socket. When a container:
- joins the `nginx_proxy` network, **and**
- has a `VIRTUAL_HOST` environment variable set

…the proxy generates an nginx vhost for it instantly. When the container stops,
the vhost is removed. No restarts, no config edits.

---

## Requirements

- Docker + Docker Compose v2
- Ports 80 and 443 free on the host

---

## Deploy

```bash
docker compose up -d
```

That's it. The `nginx_proxy` network is created automatically and persists across
restarts. Run this once per machine — never again unless you're upgrading.

---

## Directory layout

```
proxy/                  ← this repo
├── docker-compose.yml
├── .gitignore
└── README.md

apps/                   ← not part of this repo (.gitignored)
├── vpos/               ← separate git repo
└── other-app/          ← separate git repo
```

Each app lives in `apps/` on disk but is its own independent git repo. This repo
ignores that directory entirely.

---

## Adding an app

No changes needed in this repo. In the app's `docker-compose.yml`:

**1. Declare the network as external:**
```yaml
networks:
  proxy_net:
    name: nginx_proxy
    external: true
```

**2. Join the network and set `VIRTUAL_HOST` on the web-facing service:**
```yaml
services:
  web:
    environment:
      VIRTUAL_HOST: myapp.com
      VIRTUAL_PORT: 80   # always set explicitly
    networks:
      - proxy_net
```

**3. Start the app:**
```bash
docker compose up -d
```

The proxy picks it up within seconds.

---

## Stopping an app

```bash
docker compose down
```

The proxy removes the vhost automatically.

---

## HTTPS / Let's Encrypt

The proxy is pre-wired for [acme-companion](https://github.com/nginx-proxy/acme-companion).
To enable automatic TLS, add the companion service to `docker-compose.yml`:

```yaml
services:
  acme:
    image: nginxproxy/acme-companion:latest
    container_name: nginx_proxy_acme
    restart: unless-stopped
    volumes_from:
      - proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - certs:/etc/nginx/certs:rw
    environment:
      DEFAULT_EMAIL: you@example.com
    networks:
      - proxy_net
```

Then in each app, add alongside `VIRTUAL_HOST`:
```yaml
LETSENCRYPT_HOST: myapp.com
```

---

## Upgrading the proxy image

```bash
docker compose pull
docker compose up -d
```

Zero-downtime if apps are still running — nginx-proxy restarts in seconds.
