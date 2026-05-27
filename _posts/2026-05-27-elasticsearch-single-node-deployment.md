---
layout: post
title: "Deploying Elasticsearch Securely on a Single Cloud VM"
date: 2026-05-27
description: "How I set up Elasticsearch and Kibana on an IBM Cloud RHEL 9 VM with Nginx TLS termination, Let's Encrypt certs, and XPack security."
---

I needed a simple Elasticsearch instance for a project — not a multi-node production cluster, just a single node I could query from anywhere with proper security. This post documents how I set it up on an IBM Cloud RHEL 9 VM using Podman, Nginx, and Let's Encrypt.

The full config is on [GitHub](https://github.com/nicolenair/elasticsearch-single-node-deployment).

## Architecture

```
Client → Cloudflare DNS → Nginx (port 443, TLS) → Elasticsearch :9200
                                                 → Kibana        :5601
```

Three containers run via Docker Compose (Podman-compatible): Elasticsearch, Kibana, and Nginx. Elasticsearch and Kibana are bound exclusively to `127.0.0.1` — they're never directly reachable from the internet. Nginx is the only thing listening on a public port (443), and it terminates TLS before proxying to either service.

## Key decisions

### Nginx handles TLS, not Elasticsearch

Elasticsearch 8.x ships with XPack security enabled by default, which includes its own SSL support. I disabled ES-level SSL (`xpack.security.http.ssl.enabled=false`) and let Nginx handle it instead.

Kibana talks to Elasticsearch over plain HTTP on the internal Docker network, which is fine since that traffic never leaves the VM.

```yaml
# docker-compose.yml
elasticsearch:
  environment:
    - discovery.type=single-node
    - xpack.security.enabled=true
    - xpack.security.http.ssl.enabled=false
  ports:
    - "127.0.0.1:9200:9200"
```

`discovery.type=single-node` tells Elasticsearch not to wait for other nodes to form a cluster — without this it hangs on startup.

### XPack for authentication, not Nginx basic auth

Even though Nginx is the gateway, I kept XPack security enabled for user authentication. This means Elasticsearch manages its own users and passwords, and Kibana authenticates against ES directly. The alternative — using Nginx basic auth — would protect the HTTP layer but leave Elasticsearch itself unauthenticated on the internal network.

Setting up credentials requires running `elasticsearch-setup-passwords interactive` inside the container after first boot, which initializes passwords for the built-in accounts (`elastic`, `kibana_system`, etc.).

### DNS challenge for Let's Encrypt

Certbot's standard HTTP-01 challenge requires port 80 to be open. I skipped that and used the DNS challenge instead, which validates domain ownership via a TXT record. This means port 80 never needs to be open, and the same certificate covers both subdomains (`es.yourdomain.com` and `kibana.yourdomain.com`).

One Cloudflare gotcha: DNS proxying (the orange cloud) interferes with Let's Encrypt validation. The DNS records need to be "DNS only" (grey cloud) during certificate issuance.

### SELinux and rootless Podman

RHEL 9 uses Podman instead of Docker and has SELinux enforcing by default — both of which require a bit of extra setup that often trips people up.

For rootless port binding (port 443 without root), the kernel parameter `net.ipv4.ip_unprivileged_port_start` needs to be set to 443 or lower:

```bash
sysctl -w net.ipv4.ip_unprivileged_port_start=443
```

For SELinux, volume mounts need the correct context label so containers can read host files (like the Let's Encrypt certs). The `:z` or `:Z` flag on the volume mount handles this, or you can set it explicitly with `chcon`.

User lingering also needs to be enabled so that the systemd user session (and containers) persist after you log out:

```bash
loginctl enable-linger $USER
```

## Nginx config

Two virtual hosts, one per subdomain. The Kibana block needs the forwarded headers so Kibana can construct correct redirect URLs:

```nginx
server {
  listen 443 ssl;
  server_name kibana.yourdomain.com;

  ssl_certificate     /etc/letsencrypt/live/es.yourdomain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/es.yourdomain.com/privkey.pem;

  location / {
    proxy_pass http://kibana:5601;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

The Elasticsearch block is simpler — no forwarded headers needed since it's an API, not a web app rendering links.

## What I'd change for production

This setup works well for a personal instance or small team. For anything more serious I'd add:

- **Rate limiting in Nginx** — Elasticsearch's API has no built-in rate limiting
- **Certificate auto-renewal** — a cron job or systemd timer running `certbot renew` and reloading Nginx
- **Snapshot repository** — periodic snapshots to object storage so data survives if the VM goes away
- **Resource limits** — Elasticsearch is memory-hungry; setting `ES_JAVA_OPTS` and container memory limits prevents it from taking down the VM under load
