# Usage - Docker Compose

This guide covers how to expose your services through Traefik using standard Docker Compose.

> [!NOTE]
> If you want features like native secret management, multi-node deployments, or blue-green deployments, see [Usage - Swarm](./4-usage-swarm.md) instead.

## Quick Start

To expose any service through Traefik with automatic SSL, you need to:

1. Add Traefik labels to your container
2. Connect it to the `proxy` network
3. Set the required environment variables

## Simple Example

Here's a full docker-compose.yaml example using Uptime Kuma:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes:
      - uptime-kuma:/app/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
      - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
      - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
      - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
    networks:
      - proxy
    restart: always

volumes:
  uptime-kuma:

networks:
  proxy:
    external: true
```

Then you need to define the env vars `APP_DOMAIN`, `APP_PORT`, and `APP_NAME` like:

```bash
# what you set in the dns record. Do not include http(s)://
APP_DOMAIN=whatsup.yourDomain.com

# must be what your app actually uses
APP_PORT=3001

# any name unique in your traefik config
APP_NAME=uptime
```

Start the service:

```bash
docker compose up -d
```

After a minute or two for SSL certificate generation, your service will be available at `https://whatsup.yourDomain.com`.

## Understanding the Labels

Let's break down what each Traefik label does:

| Label | Purpose |
|-------|---------|
| `traefik.enable=true` | Tells Traefik to route traffic to this container |
| `traefik.http.routers.${APP_NAME}.rule=Host(...)` | Routes traffic based on the domain name |
| `traefik.http.routers.${APP_NAME}.entrypoints=websecure` | Uses the HTTPS entrypoint (port 443) |
| `traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt` | Automatically generates SSL certificates |
| `traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}` | Tells Traefik which port your app listens on |

## The Proxy Network

The `proxy` network is created by the Portaefik setup. All services that need to be exposed through Traefik must be connected to this network.

**Add to your service:**

```yaml
networks:
  - proxy
```

**Declare at root level:**

```yaml
networks:
  proxy:
    external: true
```

It must be `external: true` because the network was created by the Portaefik docker-compose file, not this one.

## Another Example: n8n

[n8n](https://n8n.io/) is a workflow automation tool. It needs some extra environment variables to work properly behind a reverse proxy:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_HOST=${APP_DOMAIN}
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${APP_DOMAIN}/
      - GENERIC_TIMEZONE=${TIMEZONE}
      - TZ=${TIMEZONE}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${APP_NAME}.rule=Host(`${APP_DOMAIN}`)"
      - "traefik.http.routers.${APP_NAME}.entrypoints=websecure"
      - "traefik.http.routers.${APP_NAME}.tls.certresolver=letsencrypt"
      - "traefik.http.services.${APP_NAME}.loadbalancer.server.port=${APP_PORT}"
    networks:
      - proxy
    restart: always

volumes:
  n8n_data:

networks:
  proxy:
    external: true
```

Environment variables:

```bash
APP_DOMAIN=n8n.yourDomain.com
APP_PORT=5678
APP_NAME=n8n
TIMEZONE=America/New_York  # https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
```

## Troubleshooting

### SSL Certificate Not Generating

- Ensure DNS records are pointing to your server's IP address
- Wait for DNS propagation (can take up to 48 hours, but usually minutes)
- Check Traefik logs: `docker logs traefik`

### 502 Bad Gateway

- Verify the `APP_PORT` matches the port your application actually listens on
- Ensure your container is connected to the `proxy` network
- Check if your application is running: `docker logs <container-name>`

### Too Many Redirects (Cloudflare)

If using Cloudflare, set SSL mode to `Full (strict)` in your Cloudflare dashboard under SSL/TLS settings.

## Managing with Portainer

Once Portaefik is set up, you can create and manage all your stacks directly in the Portainer web UI at your configured Portainer domain.

For detailed Portainer usage, see the [official Portainer documentation](https://docs.portainer.io/user/docker).
