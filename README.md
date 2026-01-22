# Portaefik

The name "Portaefik" is a portmanteau of the two open source projects it uses: [Portainer](https://www.portainer.io/) and [Traefik](https://traefik.io/traefik/).

**Portainer** is a Docker management software for managing compose, swarm, and even Kubernetes with an easy to use web UI.

**Traefik** is a modern reverse proxy and load balancer that is extremely easy to set up and use.

**Combined together into Portaefik**, along with the docs I have written, it's an easy way to get up and running with self-hosting fast.

## Quick Start

> [!NOTE]
> I recommend follow the [documentation](#documentation) instead for detailed setup instructions.

```bash
# Clone the repo
git clone https://github.com/liamdmcgarrigle/portaefik.git
cd portaefik

# Copy and edit the env file
cp .env.example .env
nano .env

# Generate Traefik password hash and append to .env
echo "HASHED_TRAEFIK_PASSWORD='$(openssl passwd -apr1 your-password)'" >> .env

# Start (after DNS records are pointing to your server)
docker compose up -d
```

## Documentation

| Doc | Description |
|-----|-------------|
| [1. Prerequisites](docs/1-prerequisites.md) | Server setup, non-root user, installing Docker |
| [2. Setup](docs/2-setup.md) | Configuration, DNS, starting containers |
| [3. Usage - Compose](docs/3-usage-compose.md) | Exposing services with Docker Compose |
| [4. Usage - Swarm](docs/4-usage-swarm.md) | Docker Swarm setup, secrets management |
| [5. Securing](docs/5-securing.md) | Firewall, SSH hardening, security basics |

## What's Included

- `docker-compose.yaml` - Standard Docker Compose setup
- `docker-swarm.yaml` - Docker Swarm setup with secrets support
- `.env.example` - Template for required environment variables

## Requirements

- A server (VPS) with Docker installed
- A domain name with DNS pointing to your server
- Ports 80 and 443 open
