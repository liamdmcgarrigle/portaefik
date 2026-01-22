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