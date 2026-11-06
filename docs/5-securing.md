# Securing Your Server

After setting up Portaefik, you should harden your server. This guide covers the basics. 

> [!WARNING]
> You are responsible for your own security. This is not exhaustive, research additional hardening for your specific use case.

## Configure UFW Firewall

UFW (Uncomplicated Firewall) is the standard firewall on Ubuntu.

### Install and Enable UFW

```bash
sudo apt install ufw -y
```

### Allow Required Ports

Before enabling the firewall, allow the ports you need:

```bash
# Allow SSH (so you don't lock yourself out)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS for Traefik
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

If you're using Docker Swarm with multiple nodes, you'll also need:

```bash
# Docker Swarm ports (only if using multi-node swarm)
sudo ufw allow 2377/tcp  # Cluster management
sudo ufw allow 7946/tcp  # Node communication
sudo ufw allow 7946/udp  # Node communication
sudo ufw allow 4789/udp  # Overlay network traffic
```

### Enable the Firewall

```bash
sudo ufw enable
```

Verify it's working:

```bash
sudo ufw status
```

## Block SSH When Not Needed

Once Portaefik is running, you can manage everything through Portainer's web UI. Consider using your hosting provider's firewall (or physical firewall if on prem) to block all ports except 80 and 443, and only open SSH when you need terminal access.

## Keep System Updated

Set up automatic security updates:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
```

## Fail2Ban 

If you keep SSH enabled, Fail2Ban helps prevent brute force attacks:

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

The default config protects SSH. Check status with:

```bash
sudo fail2ban-client status sshd
```

## Docker-Specific Security

### Don't Expose Unnecessary Ports

Only expose ports through Traefik labels. Avoid using `ports:` in your compose files when possible - use the internal `proxy` network instead.

**Bad** (exposes port directly):

```yaml
services:
  myapp:
    ports:
      - "3000:3000"
```

**Better** (only accessible through Traefik):

```yaml
services:
  myapp:
    labels:
      - "traefik.enable=true"
      # ... other labels
    networks:
      - proxy
```

## Real Client IPs

Apps behind Traefik see the visitor's IP in the `X-Real-Ip` and `X-Forwarded-For` headers. Anything that rate limits, logs, or bans by IP depends on that being right.

Out of the box Traefik handles this safely. If a request doesn't come from a trusted IP, it throws away any `X-Forwarded-*` or `X-Real-Ip` headers the client sent and sets them itself from the actual connection. So a client can't fake its IP, and you don't need to configure anything.

### Swarm: Host Mode Ports

In swarm, a normal `"443:443"` port goes through the routing mesh, which replaces the client's IP with an internal swarm address. Traefik then correctly reports that internal IP, so every visitor looks like the same person. This is a [long-standing swarm limitation](https://github.com/moby/moby/issues/25526).

`docker-swarm.yaml` publishes 80 and 443 in `mode: host` to get around it. The tradeoff is that only the node actually running Traefik listens on 80 and 443, so your DNS needs to point at that node. On a single node that's the same thing. If you have more than one manager, pin Traefik to one node (for example `node.hostname == your-node`) so it doesn't move away from your DNS.

### Something in Front of Traefik

If traffic goes through a Cloudflare proxy, a tunnel, or a load balancer first, Traefik sees that upstream's IP as the client. To fix it, tell Traefik to trust the headers that upstream sends by uncommenting the `trustedIPs` line in the compose file and listing only that upstream's IP ranges ([Cloudflare publishes theirs](https://www.cloudflare.com/ips/)).

> [!WARNING]
> Never use `--entrypoints.websecure.forwardedHeaders.insecure=true` in production. It trusts headers from anyone, so any client can claim to be any IP.

Once an upstream is trusted, Traefik keeps the `X-Real-Ip` it sends and only fills it in when it's missing. Cloudflare doesn't send `X-Real-Ip` (it uses `CF-Connecting-IP`), so apps reading `X-Real-Ip` will see Cloudflare's IP. Have the app read `CF-Connecting-IP`, or use a Traefik plugin that copies it into `X-Real-Ip`. It also means a fake `X-Real-Ip` that a client sends through the upstream is kept as is, so run the check below through the upstream too.

### Check It

Deploy [traefik/whoami](https://github.com/traefik/whoami) behind Traefik, open it from your phone or another network, and compare `X-Real-Ip` with your actual IP. Then try faking it:

```bash
curl -H "X-Real-Ip: 1.2.3.4" -H "X-Forwarded-For: 1.2.3.4" https://whoami.yourdomain.com
```

If `1.2.3.4` shows up in the response, Traefik is trusting headers it shouldn't.
