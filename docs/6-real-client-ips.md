# Real Client IPs

Apps behind Traefik learn the visitor's IP from request headers, because the connection they actually see comes from Traefik. Anything that rate limits, logs, or bans by IP depends on that header being right, and on nobody being able to fake it.

This page covers how Traefik handles it out of the box, the swarm port setting that keeps it working, and what to do when something sits in front of Traefik, with Cloudflare covered in depth.

> [!WARNING]
> You are responsible for your own security. Test your own setup with the [check at the end](#check-it) rather than assuming it works.

## How Traefik Handles It

Out of the box Traefik is safe here, and you don't need to configure anything. For a connection that doesn't come from a trusted IP (and by default nothing is trusted), Traefik:

1. Deletes any `X-Forwarded-*` and `X-Real-Ip` headers the client sent.
2. Sets `X-Real-Ip` and `X-Forwarded-For` to the address the connection actually came from.

So a client can't claim to be someone else by sending its own headers. Your app should read `X-Real-Ip`, or the last entry of `X-Forwarded-For`. Never trust the first entry of `X-Forwarded-For` unless you know exactly who wrote it.

> [!WARNING]
> Never set `--entrypoints.websecure.forwardedHeaders.insecure=true` in production. It trusts forwarded headers from everyone, so any client can claim to be any IP.

## Docker Swarm: Host Mode Ports

In swarm, a normal `"443:443"` port goes through the routing mesh, which replaces the visitor's IP with an internal swarm address. Traefik then correctly reports that internal address, so every visitor looks like the same person. This is a [long-standing swarm limitation](https://github.com/moby/moby/issues/25526).

`docker-swarm.yaml` publishes 80 and 443 in `mode: host` to get around it. The tradeoff is that only the node actually running Traefik listens on 80 and 443, so your DNS needs to point at that node. On a single node that's the same thing. If you have more than one manager, pin Traefik to one node (for example `node.hostname == your-node`) so it doesn't move away from your DNS.

<details>
<summary>Switching an existing swarm deploy to host mode</summary>

Redeploying the stack updates the service, but swarm may not replace the running Traefik task for a port change alone. Check with:

```bash
docker service inspect portaefik_traefik --format '{{json .UpdateStatus}}'
docker service ps portaefik_traefik
```

If `UpdateStatus` is `null` and only the old task is running, force it:

```bash
docker service update --force portaefik_traefik
```

You'll get a few seconds of downtime while the old task frees the ports. Afterwards `sudo ss -ltnp | grep -E ':(80|443)\b'` should show `docker-proxy` on 80 and 443 instead of `dockerd`.

</details>

## Docker Compose

Plain Docker port publishing keeps the visitor's IPv4 address, so `docker-compose.yaml` needs nothing extra.

If your server has IPv6 but the Docker network doesn't, Docker passes IPv6 connections through a proxy process, and they can arrive with the Docker network's gateway address instead of the visitor's. If you serve IPv6, run the check at the end from an IPv6 connection.

## A Load Balancer in Front of Traefik

If you run your own load balancer in front of Traefik, Traefik sees the load balancer as every visitor. Tell Traefik to trust the headers that load balancer sends by uncommenting the `trustedIPs` line in the stack file and listing only the load balancer's IPs:

```yaml
      - "--entrypoints.websecure.forwardedHeaders.trustedIPs=10.0.0.5/32"
```

Traefik then keeps the `X-Forwarded-For` and `X-Real-Ip` that the load balancer sets, so make sure the load balancer overwrites them rather than passing along whatever the client sent. Anything inside the trusted range can claim any IP, so keep the list as narrow as you can.

## Behind Cloudflare

When a site is proxied through Cloudflare (the orange cloud in DNS), every connection to your server comes from a Cloudflare edge server. Traefik correctly sets `X-Real-Ip` to that edge address, so it's useless for telling visitors apart. Cloudflare puts the visitor's real address in its own header, `CF-Connecting-IP`.

Leave `trustedIPs` off behind Cloudflare. Cloudflare strips `X-Real-Ip`, so trusting it in Traefik still leaves `X-Real-Ip` set to the edge address. Have your app read `CF-Connecting-IP` instead, as described below.

### Why Trusting Cloudflare's IPs Isn't Enough

The common advice is to read `CF-Connecting-IP` only when the request comes from one of [Cloudflare's IP ranges](https://www.cloudflare.com/ips/). That stops people hitting your server directly from outside Cloudflare, but it misses one case.

Anyone with a free Cloudflare account can write a Worker that sends requests straight to your server's IP. Those requests come from real Cloudflare addresses, and [Cloudflare's docs](https://developers.cloudflare.com/fundamentals/reference/http-headers/) say that in this case "the Worker script controls the client IP headers: both `CF-Connecting-IP` and `x-real-ip` can be altered by the Worker." So a range check alone lets anyone who finds your server's IP pick whatever visitor IP they like.

Normal visitors going through your domain can't do this. Cloudflare sets `CF-Connecting-IP` itself, and when I tested sending a fake one through Cloudflare, it refused the request with a 403 before it reached the server.

The fix is to make your server accept only connections that come through your own Cloudflare zone, using zone-level Authenticated Origin Pulls.

### Lock the Server With Authenticated Origin Pulls

With [Authenticated Origin Pulls](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/) (AOP), Cloudflare presents a client certificate that only you issued whenever it connects to your server, and Traefik refuses any TLS connection without it. A Worker hitting your IP directly has no certificate, so it never gets past the handshake. It's free on every Cloudflare plan.

Use the zone-level setup with your own certificate. The global AOP certificate is shared by every Cloudflare account, so it only proves a request came from Cloudflare, not from your zone.

> [!WARNING]
> This applies to every site on this Traefik, because the setting goes on Traefik's `default` TLS option. Before you start, make sure:
>
> - Every hostname Traefik serves is proxied through Cloudflare. Any that isn't will stop working.
> - Nothing reaches your server by IP, such as an uptime monitor pointed at the IP. Monitors that use your domain go through Cloudflare and are fine.
> - Cloudflare's SSL/TLS mode is Full (strict).
>
> Let's Encrypt's HTTP challenge runs on port 80, which has no TLS, so certificate renewals aren't affected.

#### 1. Make the Certificates

Run this on the server, outside the repo. It makes a small certificate authority (CA) and a client certificate signed by it. You'll be asked for a passphrase for the CA key.

```bash
mkdir -p ~/cloudflare-aop && cd ~/cloudflare-aop
umask 077

# CA, valid 5 years
openssl genrsa -aes256 -out rootca.key 4096
openssl req -x509 -new -key rootca.key -sha256 -days 1826 \
  -subj "/O=yourdomain/CN=yourdomain.com" -out rootca.crt

# client certificate Cloudflare will present, valid 2 years
openssl req -new -nodes -newkey rsa:4096 -keyout cert.key \
  -subj "/O=yourdomain/CN=cloudflare-aop.yourdomain.com" -out cert.csr
printf 'basicConstraints=CA:FALSE\nkeyUsage=critical,digitalSignature,keyEncipherment\nextendedKeyUsage=clientAuth\n' > cert.v3.ext
openssl x509 -req -in cert.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial \
  -out cert.crt -days 730 -sha256 -extfile ./cert.v3.ext

# check
openssl verify -CAfile rootca.crt -purpose sslclient cert.crt   # cert.crt: OK
openssl x509 -in cert.crt -noout -enddate                        # note this date
```

Save `rootca.key`, its passphrase, and `rootca.crt` in your password manager. You need them to sign the next client certificate when this one expires.

#### 2. Upload It to Cloudflare and Turn It On

1. In the Cloudflare dashboard, open your zone, then SSL/TLS > Origin Server > **Authenticated Origin Pulls**.
2. In the **Zone-level** section, choose **Upload certificate**. Use `cert.crt` as the certificate and `cert.key` as the private key. Don't upload `rootca.crt` or `rootca.key`; Cloudflare only accepts the client certificate.
3. Save the certificate ID it shows. You'll need it to delete this certificate later.
4. Switch the **Zone-level** toggle on. It's a separate toggle from the global one.

Nothing changes for visitors yet. Cloudflare now offers the certificate, but Traefik isn't asking for one.

#### 3. Make Traefik Require It

The repo includes [`traefik/cloudflare-aop.yml`](../traefik/cloudflare-aop.yml), which sets Traefik's `default` TLS option to require a client certificate signed by your CA. TLS options can't be set with labels, which is why it's a file.

**With swarm**, store both files as swarm configs. Configs can't be edited, so the names end in a version number:

```bash
docker config create cloudflare_aop_ca_v1 ~/cloudflare-aop/rootca.crt
docker config create traefik_cloudflare_aop_v1 ~/portaefik/traefik/cloudflare-aop.yml
```

Then uncomment the three `cloudflare authenticated origin pulls` blocks in `docker-swarm.yaml`: the `--providers.file.filename` line in Traefik's `command`, the `configs` block on the Traefik service, and the top-level `configs` section.

**With compose**, copy the CA next to the config file and uncomment the two volume lines and the `--providers.file.filename` line in `docker-compose.yaml`:

```bash
cp ~/cloudflare-aop/rootca.crt ~/portaefik/traefik/cloudflare-aop-ca.pem
```

#### 4. Deploy

For swarm, deploy as usual (see [Setup](./2-setup.md#optionally-use-docker-swarm)), then check that a new Traefik task started:

```bash
docker service ps portaefik_traefik
```

For compose, run `docker compose up -d`.

You'll get a few seconds of downtime while Traefik restarts. Once it's back, every line in Traefik's access log should include `"TLSClientSubject":"CN=cloudflare-aop.yourdomain.com,..."`, which means Cloudflare is presenting the certificate.

#### 5. Verify

Your sites should load as normal through Cloudflare. Then, from another machine, connect straight to the server's IP and skip Cloudflare:

```bash
curl -sS -o /dev/null -w 'status %{http_code}\n' \
  --resolve yourdomain.com:443:<server-ip> https://yourdomain.com/
```

This must fail with a certificate error and `status 000`. Depending on your curl, the error reads "alert certificate required" or `reason(1116)`. If you get a status like 200, Traefik isn't requiring the certificate, so recheck step 3.

#### If It Breaks

If your sites go down after step 4, the usual cause is the Zone-level toggle being off, or the wrong certificate being uploaded. For swarm, roll Traefik back to how it was:

```bash
docker service rollback portaefik_traefik
```

For compose, comment the lines out again and run `docker compose up -d`. Fix the cause, then deploy again.

#### Once It Works

Delete `~/cloudflare-aop` once the CA key, its passphrase, and the CA certificate are in your password manager. Cloudflare holds the client certificate, and Traefik has the CA.

Then put a reminder in your calendar a month before the date from step 1. When the client certificate expires, every request fails. Cloudflare also emails you 30 and 14 days before.

#### Rotating the Certificate

The CA lasts five years, so rotating the client certificate doesn't touch Traefik:

1. Get the CA key, passphrase, and certificate from your password manager into a fresh `~/cloudflare-aop`.
2. Repeat the "client certificate" part of step 1.
3. Upload the new certificate in the Zone-level section. Cloudflare uses the newest one.
4. Once it shows as active, delete the old one by its ID.

When the CA itself gets close to five years, make a new CA, create a `cloudflare_aop_ca_v2` config containing both CA certificates in one file, switch the stack to it, and then rotate the client certificate onto the new CA.

### Reading the IP in Your App

With AOP on, only your Cloudflare zone can reach Traefik, so `CF-Connecting-IP` can be trusted. Still, have your app check where the request came from before using it. That check is what protects you if AOP is ever switched off, or if another container on the `proxy` network calls your app directly:

1. Take the peer address from `X-Real-Ip` (which Traefik sets from the real connection).
2. If the peer is inside [Cloudflare's IP ranges](https://www.cloudflare.com/ips/) and `CF-Connecting-IP` is a valid IP address, the visitor is `CF-Connecting-IP`.
3. Otherwise the visitor is the peer.

Cloudflare's ranges change rarely. The lists are at https://www.cloudflare.com/ips-v4 and https://www.cloudflare.com/ips-v6, and `https://api.cloudflare.com/client/v4/ips` returns both with an `etag` that changes when they do. Keep a copy in your app, and add a scheduled CI job that fails when Cloudflare's lists stop matching yours.

## Rate Limiting by IP

A few things matter once the IP is right:

- **Group IPv6 by /64.** A single home connection usually gets at least a whole /64 block, so limiting by the full IPv6 address lets one person rotate through millions of addresses. Limit IPv6 by its /64 prefix and IPv4 by the full address. Keep the full address in your logs.
- **Limit signed-in users by account, not IP.** Many people can share one IPv4 address (mobile carriers, offices), and one person can have many. Once you know who someone is, key their limits on their user ID.
- **Treat IP limits as a speed bump.** Attackers with many addresses get around them easily. Real protection on sign-in comes from things like passkeys, MFA, and per-account lockouts.

## Check It

Deploy [traefik/whoami](https://github.com/traefik/whoami) on a path of one of your domains, so it needs no new DNS record or certificate:

```yaml
services:
  whoami:
    image: traefik/whoami:v1.10
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.whoami-test.rule=Host(`yourdomain.com`) && PathPrefix(`/__whoami`)"
        - "traefik.http.routers.whoami-test.entrypoints=websecure"
        - "traefik.http.routers.whoami-test.tls.certresolver=letsencrypt"
        - "traefik.http.routers.whoami-test.priority=100"
        - "traefik.http.services.whoami-test.loadbalancer.server.port=80"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

This is the swarm version. For compose, move the labels out from under `deploy:`.

From your phone or a laptop on another network, compare what whoami sees with your real IP, then try faking it:

```bash
curl -s https://www.cloudflare.com/cdn-cgi/trace | grep '^ip='   # your real IP
curl -s https://yourdomain.com/__whoami | grep -iE 'cf-connecting-ip|x-real-ip|x-forwarded-for'
curl -s https://yourdomain.com/__whoami -H 'X-Real-Ip: 1.2.3.4' -H 'X-Forwarded-For: 1.2.3.4' \
  | grep -iE 'cf-connecting-ip|x-real-ip|x-forwarded-for'
```

`1.2.3.4` must never show up. Without Cloudflare, `X-Real-Ip` should be your real IP. Behind Cloudflare, `Cf-Connecting-Ip` should be your real IP, and `X-Real-Ip` a Cloudflare address.

Remove whoami when you're done. While it's up, anyone can use it to see your request headers and the container's internal addresses.
